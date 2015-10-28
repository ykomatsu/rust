% ドキュメント

ドキュメントはどんなソフトウェアプロジェクトにとっても重要な部分であり、Rustにおいてはファーストクラスです。
あなたがプロジェクトのドキュメントを作成するために、Rustが提供するツールについて話しましょう。

## `rustdoc`について

Rustの配布物には`rustdoc`というドキュメントを生成するツールが含まれています。
`rustdoc`は`cargo doc`によってCargoでも使われます。

ドキュメントは2通りの方法で生成することができます。ソースコードから、そして単体のMarkdownファイルからです。

## ソースコードのドキュメントの作成

Rustのプロジェクトでドキュメントを書く1つ目の方法は、ソースコードに注釈を付けることで行います。
あなたはこの目的のためにドキュメンテーションコメントを使うことができます。

```rust,ignore
/// Constructs a new `Rc<T>`.
///
/// # Examples
///
/// ```
/// use std::rc::Rc;
///
/// let five = Rc::new(5);
/// ```
pub fn new(value: T) -> Rc<T> {
    // implementation goes here
}
```

このコードは[このような][rc-new]見た目のドキュメントを生成します。
実装についてはそこにある普通のコメントのとおり、省略しています。

この注釈について注意すべき1つ目のことは、`//`の代わりに`///`が使われていることです。
3連スラッシュはドキュメンテーションコメントを示します。

ドキュメンテーションコメントはMarkdownで書きます。

Rustはそれらのコメントを把握し、ドキュメントを生成するときにそれらを使います。
このことは列挙型のようなもののドキュメントを作成するときに重要です。

```rust
/// The `Option` type. See [the module level documentation](index.html) for more.
enum Option<T> {
    /// No value
    None,
    /// Some value `T`
    Some(T),
}
```

上記の例は動きますが、これは動きません。

```rust,ignore
/// The `Option` type. See [the module level documentation](index.html) for more.
enum Option<T> {
    None, /// No value
    Some(T), /// Some value `T`
}
```

エラーが出ます。

```text
hello.rs:4:1: 4:2 error: expected ident, found `}`
hello.rs:4 }
           ^
```

この[残念なエラー](https://github.com/rust-lang/rust/issues/22547)は正しいのです。ドキュメンテーションコメントはそれらの後のものに適用されるところ、その最後のコメントの後には何もないからです。

[rc-new]: https://doc.rust-lang.org/nightly/std/rc/struct.Rc.html#method.new

### ドキュメンテーションコメントの記述

とりあえず、このコメントの各部分を詳細にカバーしましょう。

```rust
/// Constructs a new `Rc<T>`.
# fn foo() {}
```

ドキュメンテーションコメントの最初の行は、その機能の短いサマリーにすべきです。
一文で。
基本だけを。
高レベルから。

```rust
///
/// Other details about constructing `Rc<T>`s, maybe describing complicated
/// semantics, maybe additional options, all kinds of stuff.
///
# fn foo() {}
```

私たちの例にはサマリーしかありませんが、もしもっと言うべきことがあれば、私たちは新しい段落にもっと多くの説明を追加することができます。

#### 特別なセクション

次は特別なセクションです。
それらには`#`が付いていて、ヘッダーであることを示しています。
一般的には、4種類のヘッダーが使われます。
今のところそれらは特別な構文ではなく、単なる慣習です。

```rust
/// # Panics
# fn foo() {}
```

Rustにおいて、関数の回復不可能な誤用（つまり、プログラミングエラー）は普通、パニックによって表現されます。パニックは、少なくとも現在のスレッド全体の息の根を止めてしまいます。
もしあなたの関数にこのような、パニックによって検出されたり強制されたりするような些細とは言えない取決めがあるときには、ドキュメントを作成することは非常に重要です。

```rust
/// # Failures
# fn foo() {}
```

もしあなたの関数やメソッドが`Result<T, E>`を戻すのであれば、それが`Err(E)`を戻したときの状況を説明すべきです。失敗は型システムによってコード化されますが、それでもまだそうすることはよいことだからです。

```rust
/// # Safety
# fn foo() {}
```

もしあなたの関数が`unsafe`であれば、あなたは呼出元が動作を続けるためにはどの不正について責任を持つべきなのかを説明すべきです。

```rust
/// # Examples
///
/// ```
/// use std::rc::Rc;
///
/// let five = Rc::new(5);
/// ```
# fn foo() {}
```

4つ目は`Examples`です。
あなたの関数やメソッドの使い方の例を1つ以上含めてください。
それらの例はコードブロック注釈内に入れます。コードブロック注釈についてはすぐ後で話しますが、それらは1つ以上のセクションを持つことができます。

```rust
/// # Examples
///
/// Simple `&str` patterns:
///
/// ```
/// let v: Vec<&str> = "Mary had a little lamb".split(' ').collect();
/// assert_eq!(v, vec!["Mary", "had", "a", "little", "lamb"]);
/// ```
///
/// More complex patterns with a lambda:
///
/// ```
/// let v: Vec<&str> = "abc1def2ghi".split(|c: char| c.is_numeric()).collect();
/// assert_eq!(v, vec!["abc", "def", "ghi"]);
/// ```
# fn foo() {}
```

それらのコードブロックの詳細について議論しましょう。

#### コードブロック注釈

コメント内にRustのコードを書くためには、3連バッククオートを使います。

```rust
/// ```
/// println!("Hello, world");
/// ```
# fn foo() {}
```

もしあなたがRustのコードではないものを書きたいのであれば、注釈を追加することができます。

```rust
/// ```c
/// printf("Hello, world\n");
/// ```
# fn foo() {}
```

これは、あなたが書いている言語が何であるかに応じてハイライトされます。
もしあなたが単なるプレーンテキストを書いているのであれば、`text`を選択してください。

ここでは正しい注釈を選ぶことが重要です。なぜなら、`rustdoc`はそれを興味深い方法で使うからです。それらが消費期限切れにならないように、ライブラリークレート内で実際にあなたの例をテストするために使うのです。
もしあなたの例の中にCのコードが含まれているのに、あなたが注釈を付けるのを忘れてしまい、`rustdoc`がそれをRustのコードだと考えてしまえば、`rustdoc`はドキュメントを生成しようとするときに文句を言うでしょう。

## テストとしてのドキュメント

私たちのドキュメントの例について議論しましょう。

```rust
/// ```
/// println!("Hello, world");
/// ```
# fn foo() {}
```

あなたは`fn main()`とかがここでは不要だということに気が付くでしょう。
`rustdoc`は自動的に`main()`ラッパーをあなたのコードの周りの正しい場所に追加します。
例えば、こうです。

```rust
/// ```
/// use std::rc::Rc;
///
/// let five = Rc::new(5);
/// ```
# fn foo() {}
```

これが、テストのときには結局こうなります。

```rust
fn main() {
    use std::rc::Rc;
    let five = Rc::new(5);
}
```

これが`rustdoc`が例の前処理に使うアルゴリズムの全てです。

1. 前の方にある全ての`#![foo]`属性は、そのままクレートの属性として置いておく
2. `unused_variables`、`unused_assignments`、`unused_mut`、`unused_attributes`、`dead_code`を含むいくつかの一般的な`allow`属性を追加する。
   小さな例はしばしばこれらのチェックに引っ掛かる
3. もしその例が`extern crate`を含んでいなければ、`extern crate <mycrate>;`を挿入する
4. 最後に、もし例が`fn main`を含んでいなければ、テキストの残りの部分を`fn main() { your_code }`で囲む

しかし、これでは不十分なことがときどきあります。
例えば、私たちが話してきた全ての`///`の付いたコード例はどうだったでしょうか。
生のテキストはこうなっています。

```text
/// Some documentation.
# fn foo() {}
```

それは出力とは違って見えます。

```rust
/// Some documentation.
# fn foo() {}
```

そうです。正解です。あなたは`# `で始まる行を追加することで、コードをコンパイルするときには使われるけれども、出力はされないというようにすることができます。
あなたはこれを都合のよいように使うことができます。
この場合、私はあなたにドキュメンテーションコメントそのものを見せたいので、ドキュメンテーションコメントを何らかの関数に適用する必要があります。そのため、私には、その後に小さい関数定義を追加する必要があります。
同時に、それは単にコンパイラーを満足させるためだけのものなので、それを隠すことで、例がすっきりするのです。
あなたは、長い例を詳細に説明する一方、テスト可能性を維持するためにこのテクニックを使うことができます。
例えば、このコードです。

```rust
let x = 5;
let y = 6;
println!("{}", x + y);
```

これがレンダリングされた説明です。

-------------------------------------------------------------------------------

First, we set `x` to five:

```rust
let x = 5;
# let y = 6;
# println!("{}", x + y);
```

Next, we set `y` to six:

```rust
# let x = 5;
let y = 6;
# println!("{}", x + y);
```

Finally, we print the sum of `x` and `y`:

```rust
# let x = 5;
# let y = 6;
println!("{}", x + y);
```

-------------------------------------------------------------------------------

これが同じ説明の生のテキストです。

-------------------------------------------------------------------------------

> First, we set `x` to five:
>
> ```text
> let x = 5;
> # let y = 6;
> # println!("{}", x + y);
> ```
>
> Next, we set `y` to six:
>
> ```text
> # let x = 5;
> let y = 6;
> # println!("{}", x + y);
> ```
>
> Finally, we print the sum of `x` and `y`:
>
> ```text
> # let x = 5;
> # let y = 6;
> println!("{}", x + y);
> ```

-------------------------------------------------------------------------------

例の全体を繰り返すことで、あなたは、例がちゃんとコンパイルされることを保証する一方、説明に関係する部分だけを見せることができます。

### マクロのドキュメントの作成

これはマクロのドキュメントの例です。

```rust
/// Panic with a given message unless an expression evaluates to true.
///
/// # Examples
///
/// ```
/// # #[macro_use] extern crate foo;
/// # fn main() {
/// panic_unless!(1 + 1 == 2, “Math is broken.”);
/// # }
/// ```
///
/// ```should_panic
/// # #[macro_use] extern crate foo;
/// # fn main() {
/// panic_unless!(true == false, “I’m broken.”);
/// # }
/// ```
#[macro_export]
macro_rules! panic_unless {
    ($condition:expr, $($rest:expr),+) => ({ if ! $condition { panic!($($rest),+); } });
}
# fn main() {}
```

あなたは3つのことに気が付くでしょう。`#[macro_use]`属性を追加するために、私たちは自分で`extern crate`行を追加しなければなりません。
2つ目に、私たちは`main()`も自分で追加する必要があります。
最後に、それらの2つが出力されないようにコメントアウトするという`#`の賢い使い方です。

### ドキュメンテーションテストの実行

テストを実行するには、どちらかを使います。

```bash
$ rustdoc --test path/to/my/crate/root.rs
# or
$ cargo test
```

正解です。`cargo test`は組み込まれたドキュメントもテストします。
**しかし、`cargo test`がテストするのはライブラリークレートだけで、バイナリークレートはテストしません。**
これは`rustdoc`の動き方によるものです。それはテストするためにライブラリーをリンクしますが、バイナリーには何もリンクするものがないからです。

`rustdoc`があなたのコードをテストするときに正しく動作するのを助けるために便利な注釈があと少しあります。

```rust
/// ```ignore
/// fn foo() {
/// ```
# fn foo() {}
```

`ignore`ディレクティブはRustにあなたのコードを無視するよう指示します。
これはあまりに汎用的なので、必要になることはほとんどありません。
もしそれがコードではなければ、代わりに`text`の注釈を付けること、又は問題となる部分だけが表示された、動作する例を作るために`#`を使うことを検討してください。

```rust
/// ```should_panic
/// assert!(false);
/// ```
# fn foo() {}
```

`should_panic`は、そのコードは正しくコンパイルされるべきではあるが、実際にテストとして成功する必要まではないということを`rustdoc`に教えます。

```rust
/// ```no_run
/// loop {
///     println!("Hello, world");
/// }
/// ```
# fn foo() {}
```

`no_run`属性はあなたのコードをコンパイルしますが、実行はしません。
これは「これはネットワークサービスを開始する方法です」というような例や、コンパイルされることは保証したいけれども、無限ループになってしまうような例にとって重要です！

### モジュールのドキュメントの作成

Rustには別の種類のドキュメンテーションコメント、`//!`があります。
このコメントは次に続く要素のドキュメントではなく、それを囲っている要素のドキュメントです。
言い換えると、こうです。

```rust
mod foo {
    //! This is documentation for the `foo` module.
    //!
    //! # Examples

    // ...
}
```

あなたが`//!`を頻繁に見る場所はここです。モジュールドキュメントです。
もしあなたが`foo.rs`内にモジュールを持っていれば、しばしばそのコードを開いてこれを見るでしょう。

```rust
//! A module for using `foo`s.
//!
//! The `foo` module contains a lot of useful functionality blah blah blah
```

### ドキュメンテーションコメントのスタイル

ドキュメントのスタイルや書式についての全ての慣習を知るには[RFC 505][rfc505]をチェックしてください。

[rfc505]: https://github.com/rust-lang/rfcs/blob/master/text/0505-api-comment-conventions.md

## その他のドキュメント

ここにある振舞いは全て、非Rustのソースコードファイルでも働きます。
コメントはMarkdownで書かれるので、しばしば`.md`ファイルになります。

あなたがドキュメントをMarkdownファイルに書くとき、ドキュメントにコメントのプレフィックスを付ける必要はありません。
例えば、こうする必要はありません。

```rust
/// # Examples
///
/// ```
/// use std::rc::Rc;
///
/// let five = Rc::new(5);
/// ```
# fn foo() {}
```

これは、単にこうします。

~~~markdown
# Examples

```
use std::rc::Rc;

let five = Rc::new(5);
```
~~~

Markdownファイルの中ではこうします。
ただし、1つだけ新しいものがあります。Markdownファイルではこのように題名を付けなければなりません。

```markdown
% The title

This is the example documentation.
```

この`%`行はそのファイルの一番先頭の行に書く必要があります。

## `doc`属性

もっと深いレベルで言えは、ドキュメンテーションコメントはドキュメント属性の糖衣構文です。

```rust
/// this
# fn foo() {}

#[doc="this"]
# fn bar() {}
```

これらは同じもので、次のものも同じものです。

```rust
//! this

#![doc="/// this"]
```

この属性がドキュメントを書くために使われているのを見ることはそんなにないでしょう。しかし、これは何らかのオプションを変更したり、マクロを書いたりするときに便利です。

### 再エクスポート

`rustdoc`はパブリックな再エクスポートがなされた場合に、両方の場所にドキュメントを表示します。

```ignore
extern crate foo;

pub use foo::bar;
```

これは`bar`のドキュメントをあなたのクレートのドキュメントの中に生成するのと同様に、`foo`クレートのドキュメントの中にも生成します。
同じドキュメントが両方の場所で使われます。

この振舞いは`no_inline`で抑制することができます。

```ignore
extern crate foo;

#[doc(no_inline)]
pub use foo::bar;
```

## ドキュメントの不存在

ときどき、プロジェクト内の公開されている全てのものについて、ドキュメントが作成されていることを確認したいことがあります。これは特にあなたがライブラリーについて作業をしているときにあります。
Rustでは、要素にドキュメントがないときに警告やエラーを生成することができます。
警告を生成するために、あなたは`warn`を使います。

```rust
#![warn(missing_docs)]
```

そしてエラーを生成するとき、あなたは`deny`を使います。

```rust,ignore
#![deny(missing_docs)]
```

何かを明示的にドキュメント化されていないままにするため、それらの警告やエラーを無効にしたい場合があります。
これは`allow`を使えば可能です。

```rust
#[allow(missing_docs)]
struct Undocumented;
```

ドキュメントから要素を完全に見えなくしたいこともあるかもしれません。

```rust
#[doc(hidden)]
struct Hidden;
```

### HTMLの制御

あなたは`rustdoc`の生成するHTMLのいくつかの外見を、`#![doc]`属性を通じて制御することができます。

```rust
#![doc(html_logo_url = "https://www.rust-lang.org/logos/rust-logo-128x128-blk-v2.png",
       html_favicon_url = "https://www.rust-lang.org/favicon.ico",
       html_root_url = "https://doc.rust-lang.org/")]
```

これは、複数の異なったオプション、つまりロゴ、お気に入りアイコン、ルートのURLをセットします。

## 生成オプション

`rustdoc`はさらなるカスタマイズのために、その他にもコマンドラインのオプションをいくつか持っています。

- `--html-in-header FILE`: FILEの内容を`<head>...</head>`セクションの末尾に加える
- `--html-before-content FILE`: FILEの内容を`<body>`の直後、レンダリングされた内容（検索バーを含む）の直前に加える
- `--html-after-content FILE`: FILEの内容を全てのレンダリングされた内容の後に加える

## セキュリティー上の注意

ドキュメンテーションコメント内のMarkdownは最終的なウェブページの中に無修正で挿入されます。
リテラルのHTMLには注意してください。

```rust
/// <script>alert(document.cookie)</script>
# fn foo() {}
```
