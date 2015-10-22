% コンパイラープラグイン

# 導入

`rustc`はコンパイラープラグインを読み込むことができます。コンパイラープラグインは新しい構文拡張やリントチェックなどでコンパイラーの挙動を拡張する、ユーザーによって提供されたライブラリーです。

プラグインは`rustc`に拡張を登録する指定された *レジストラー* 関数を持つ動的ライブラリークレートです。
他のクレートはクレート属性`#![plugin(...)]`を使ってそれらの拡張を読み込むことができます。
プラグインの定義と読込みの方法についてもっと知るためには[`rustc::plugin`](../rustc/plugin/index.html)ドキュメントを見ましょう。

もしあれば、`#![plugin(foo(... args ...))]`として渡された引数はrustc自体によって解釈されません。
それらは`Registry`の[`args`メソッド](../rustc/plugin/registry/struct.Registry.html#method.args)を通じてプラグインに提供されます。

大部分の場合では、プラグインは`#![plugin]`を通じて *のみ* 使われるべきで、`extern crate`要素によって使われるべきではありません。
プラグインのリンクはあなたのクレートによってlibsyntaxとlibrustcの全てを引き込みます。
これは一般的にあなたが他のプラグインをビルドしているのでない限り望まれていません。
`plugin_as_library`リントはそれらのガイドラインをチェックします。

普通の慣習ではコンパイラープラグインをそれら独自のクレートに入れ、全ての`macro_rules!`マクロやライブラリーの利用者によって使われることが予定されている通常のRustのコードから分離されます。

# 構文拡張

プラグインはRustの構文を様々な方法で拡張することができます。
構文拡張の一種に手続マクロがあります。
それらは[通常のマクロ](macros.html)と同じ方法で実行されます。しかし、展開は[構文木](../syntax/ast/index.html)をコンパイル時に操作する任意のRustのコードによって行われます。

ローマ数字の整数リテラルを実装するプラグイン[`roman_numerals.rs`](https://github.com/rust-lang/rust/tree/master/src/test/auxiliary/roman_numerals.rs)を書きましょう。

```ignore
#![crate_type="dylib"]
#![feature(plugin_registrar, rustc_private)]

extern crate syntax;
extern crate rustc;

use syntax::codemap::Span;
use syntax::parse::token;
use syntax::ast::{TokenTree, TtToken};
use syntax::ext::base::{ExtCtxt, MacResult, DummyResult, MacEager};
use syntax::ext::build::AstBuilder;  // trait for expr_usize
use rustc::plugin::Registry;

fn expand_rn(cx: &mut ExtCtxt, sp: Span, args: &[TokenTree])
        -> Box<MacResult + 'static> {

    static NUMERALS: &'static [(&'static str, u32)] = &[
        ("M", 1000), ("CM", 900), ("D", 500), ("CD", 400),
        ("C",  100), ("XC",  90), ("L",  50), ("XL",  40),
        ("X",   10), ("IX",   9), ("V",   5), ("IV",   4),
        ("I",    1)];

    let text = match args {
        [TtToken(_, token::Ident(s, _))] => s.to_string(),
        _ => {
            cx.span_err(sp, "argument should be a single identifier");
            return DummyResult::any(sp);
        }
    };

    let mut text = &*text;
    let mut total = 0;
    while !text.is_empty() {
        match NUMERALS.iter().find(|&&(rn, _)| text.starts_with(rn)) {
            Some(&(rn, val)) => {
                total += val;
                text = &text[rn.len()..];
            }
            None => {
                cx.span_err(sp, "invalid Roman numeral");
                return DummyResult::any(sp);
            }
        }
    }

    MacEager::expr(cx.expr_u32(sp, total))
}

#[plugin_registrar]
pub fn plugin_registrar(reg: &mut Registry) {
    reg.register_macro("rn", expand_rn);
}
```

それから私たちは`rn!()`を他のマクロと同じように使うことができます。

```ignore
#![feature(plugin)]
#![plugin(roman_numerals)]

fn main() {
    assert_eq!(rn!(MMXV), 2015);
}
```

単純な`fn(&str) -> u32`に対する利点は次のとおりです。

* コンパイル時に（様々な複雑さの）変換が完了する
* 入力の検査もコンパイル時に行われる
* それはパターンの中での使用を許すように拡張することができる。パターンは任意のデータ型に対して新しいリテラルの構文を定義する方法を効果的に与える

手続マクロに加えて、あなたは新しい[`derive`]的な属性と他の種類の拡張を定義することができます。
[`Registry::register_syntax_extension`](../rustc/plugin/registry/struct.Registry.html#method.register_syntax_extension)
と[`SyntaxExtension`
enum](https://doc.rust-lang.org/syntax/ext/base/enum.SyntaxExtension.html)を見ましょう。
関連するより多くのマクロの例を知りたければ、[`regex_macros`](https://github.com/rust-lang/regex/blob/master/regex_macros/src/lib.rs)を見ましょう。

## ヒントと小技

いくつかの[マクロのデバッグの小技](macros.html#debugging-macro-code)が適用されます。

あなたはトークン木を式のような高レベルの構文要素に変換するために[`syntax::parse`](../syntax/parse/index.html)を使うことができます。

```ignore
fn expand_foo(cx: &mut ExtCtxt, sp: Span, args: &[TokenTree])
        -> Box<MacResult+'static> {

    let mut parser = cx.new_parser_from_tts(args);

    let expr: P<Expr> = parser.parse_expr();
```

[`libsyntax`パーサーのコード](https://github.com/rust-lang/rust/blob/master/src/libsyntax/parse/parser.rs)を十分に調べることはあなたにパースインフラストラクチャーがどのように動作するのかという感覚を与えるでしょう。

よりよいエラー報告のために、あなたがパースした全ての[`Span`](../syntax/codemap/struct.Span.html)を保持しましょう。
あなたは[`Spanned`](../syntax/codemap/struct.Spanned.html)をあなたのカスタムデータ構造でラップすることができます。

[`ExtCtxt::span_fatal`](../syntax/ext/base/struct.ExtCtxt.html#method.span_fatal)の呼出しはすぐにコンパイラーを停止するでしょう。
コンパイラーが続行でき、さらなるエラーを見付けられるように、代わりに[`ExtCtxt::span_err`](../syntax/ext/base/struct.ExtCtxt.html#method.span_err)を呼び出し、[`DummyResult`](../syntax/ext/base/struct.DummyResult.html)を戻す方がよいです。

デバッグのために構文の断片をプリントするために、あなたは[`span_note`](../syntax/ext/base/struct.ExtCtxt.html#method.span_note)を[`syntax::print::pprust::*_to_string`](https://doc.rust-lang.org/syntax/print/pprust/index.html#functions)と一緒に使うことができます。

前の例は[`AstBuilder::expr_usize`](../syntax/ext/build/trait.AstBuilder.html#tymethod.expr_usize)を使って整数リテラルを生成しました。
`AstBuilder`トレイトの代替品として、`libsyntax`は[準クオートマクロ](../syntax/ext/quote/index.html)のセットを提供します。
それらはドキュメント化されておらず、エッジは荒削りです。
しかし、実装は準クオートを通常のプラグインライブラリーとして改善するためのよいスタート地点かもしれません。

# リントプラグイン

プラグインはコードスタイル、安全性などの追加のチェックで[Rustのリントインフラストラクチャー](../reference.html#lint-check-attributes)を拡張することができます。
あなたは例全体のために[`src/test/auxiliary/lint_plugin_test.rs`](https://github.com/rust-lang/rust/blob/master/src/test/auxiliary/lint_plugin_test.rs)を見ることができます。その核をここに再現します。

```ignore
declare_lint!(TEST_LINT, Warn,
              "Warn about items named 'lintme'");

struct Pass;

impl LintPass for Pass {
    fn get_lints(&self) -> LintArray {
        lint_array!(TEST_LINT)
    }

    fn check_item(&mut self, cx: &Context, it: &ast::Item) {
        if it.ident.name == "lintme" {
            cx.span_lint(TEST_LINT, it.span, "item is named 'lintme'");
        }
    }
}

#[plugin_registrar]
pub fn plugin_registrar(reg: &mut Registry) {
    reg.register_lint_pass(box Pass as LintPassObject);
}
```

それからこのようなコードがあります。

```ignore
#![plugin(lint_plugin_test)]

fn lintme() { }
```

このようなコードは次のようなコンパイラーの警告を生成するでしょう。

```txt
foo.rs:4:1: 4:16 warning: item is named 'lintme', #[warn(test_lint)] on by default
foo.rs:4 fn lintme() { }
         ^~~~~~~~~~~~~~~
```

リントプラグインの部品は次のとおりです。

* 静的な[`Lint`](../rustc/lint/struct.Lint.html)構造体を定義する1つ以上の`declare_lint!`の実行
* リントパス（ここでは存在しない）によって必要とされる全ての状態を保持する構造体
* 各構文要素のチェック方法を定義する[`LintPass`](../rustc/lint/trait.LintPass.html)の実装。
   1つの`LintPass`はいくつかの異なる`Lint`のために`span_lint`を呼び出すかもしれないが、`get_lints`メソッドを通じてそれらを全て登録すべきである

リントパスは構文横断ですが、それらは型情報が利用可能になるコンパイルの後ろの段階で実行します。
`rustc`の[組込みチェック](https://github.com/rust-lang/rust/blob/master/src/librustc/lint/builtin.rs)はほとんど同じインフラストラクチャーをリントプラグインとして使い、型情報へのアクセス方法の例を提供します。

プラグインによって定義されたリントは普通の[属性とコンパイラーフラグ](../reference.html#lint-check-attributes)、例えばに`#[allow(test_lint)]`や`-A test-lint`などによって制御されます。
それらの識別子は適切な大文字小文字と句読点の変換の行われた`declare_lint!`の最初の引数に由来します。

あなたは`foo.rs`によって読み込まれたプラグインによって提供されているものを含む、`rustc`が知っているリントの一覧を見るために`rustc -W help foo.rs`を実行できます。
