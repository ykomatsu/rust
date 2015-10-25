% エラー処理

ほとんどのプログラミング言語と同じように、Rustではプログラマーは特別な方法でエラーを処理することを奨励されています。
一般的に言って、エラー処理は例外と戻り値という大きな2つのカテゴリーに分けられます。
Rustは戻り値を選択しています。

この章で、私たちはRustでのエラー処理の方法の総合的な扱い方を提供するつもりです。
それよりも、私たちはエラー処理を一度に1つだけ導入することを試みます。これはどのように全てを組み合わせるかということのしっかりした役に立つ知識を身に付けてあなたが終えるようにするためです。

単純な方法で行うと、Rustでのエラー処理は冗長で面倒になることがあります。
この章はそれらのつまづきやすい障害物を調査し、エラー処理を簡潔かつ人間工学的にするための標準ライブラリーの使い方を説明するでしょう。

# 目次

この章は非常に長いです。その原因の大部分は、私たちが非常に初歩的な直和型とコンビネーターから始めて、Rustの行うエラー処理の方法に徐々に興味を持たせようと試みていることにあります。
そのため、他の表現力の高い型システムで経験を積んだプログラマーは飛ばし読みをしたくなるかもしれません。

* [基本](#the-basics)
* [解説アンラップ](#unwrapping-explained)
* [`Option`型](#the-option-type)
    * [`Option<T>`値の組合せ](#composing-optiont-values)
* [`Result`型](#the-result-type)
    * [整数のパース](#parsing-integers)
    * [`Result`の型エイリアスのイディオム](#the-result-type-alias-idiom)
* [間奏：アンラップは悪ではない](#a-brief-interlude-unwrapping-isnt-evil)
* [複数のエラー型の使用](#working-with-multiple-error-types)
    * [`Option`と`Result`の組合せ](#composing-option-and-result)
    * [コンビネーターの限界](#the-limits-of-combinators)
    * [途中でのリターン](#early-returns)
    * [`try!`マクロ](#the-try-macro)
    * [独自のエラー型の定義](#defining-your-own-error-type)
* [エラー処理に使われる標準ライブラリー内のトレイト](#standard-library-traits-used-for-error-handling)
    * [`Error`トレイト](#the-error-trait)
    * [`From`トレイト](#the-from-trait)
    * [実際の`try!`マクロ](#the-real-try-macro)
    * [カスタムエラー型の組合せ](#composing-custom-error-types)
    * [ライブラリー開発者への助言](#advice-for-library-writers)
* [事例研究：人口データを読み込むプログラム](#case-study-a-program-to-read-population-data)
    * [初期セットアップ](#initial-setup)
    * [引数のパース](#argument-parsing)
    * [ロジックの記述](#writing-the-logic)
    * [`Box<Error>`を使ったエラー処理](#error-handling-with-boxerror)
    * [標準入力からの読込み](#reading-from-stdin)
    * [カスタム型を使ったエラー処理](#error-handling-with-a-custom-type)
    * [機能追加](#adding-functionality)
* [短編小説](#the-short-story)

# 基本

あなたはエラー処理を、計算が成功したかそうでないかを判断するために *場合分け* を使うようなものだと考えることができます。
あなたが思っているとおり、人間工学的なエラー処理の鍵は、コードを組合せ可能な状態に保ったまま、プログラマーが明示的に行わなくてはならない場合分けの数を減らすことです。

コードを組合せ可能な状態に保つことは重要です。なぜなら、その要求なしでは、予期しないものに遭遇したときに[`panic`](../std/macro.panic!.html)する可能性があるからです。
（`panic`は現在の仕事を巻き戻し、ほとんどの場合にプログラム全体が停止します。）
これが例です。

```rust,should_panic
// Guess a number between 1 and 10.
// If it matches the number we had in mind, return true. Else, return false.
fn guess(n: i32) -> bool {
    if n < 1 || n > 10 {
        panic!("Invalid number: {}", n);
    }
    n == 5
}

fn main() {
    guess(11);
}
```

もしあなたがこのコードを実行しようとすれば、プログラムはこのようなメッセージとともにクラッシュするでしょう。

```text
thread '<main>' panicked at 'Invalid number: 11', src/bin/panic-simple.rs:5
```

これはもう少しひねりのないもう1つの例です。
整数を引数として受け取るプログラムがそれを2倍してプリントします。

<span id="code-unwrap-double"></span>

```rust,should_panic
use std::env;

fn main() {
    let mut argv = env::args();
    let arg: String = argv.nth(1).unwrap(); // error 1
    let n: i32 = arg.parse().unwrap(); // error 2
    println!("{}", 2 * n);
}
```

もしあなたがこのプログラムに引数0を与えたり（エラー1）、1つ目の引数が整数ではなかったりすると（エラー2）、プログラムは1つ目の例とちょうど同じようにパニックするでしょう。

あなたはこのスタイルのエラー処理を、陶磁器屋で走っている雄牛に似たものだと考えることができます。
雄牛は行きたいところへ行くことができますが、プロセス内の全てを粉砕してしまうでしょう。

## 解説アンラップ

前の例では、私たちは2つのエラーの条件のうちの1つに到達した場合にプログラムが単にパニックすることを要求しようとしました。しかし、プログラムは最初の例のように`panic`の明示的な呼出しを含んでいません。
これはパニックが`unwrap`の呼出しの中に組み込まれているからです。

Rustでは、何かを「アンラップ」するということは、言ってみれば「私に計算結果を与えてください。もしエラーが起きたのであれば、単にパニックしてプログラムを中止してください。」ということです。
アンラップするコードは非常に単純なので、私たちはそれを見せたほうがよいでしょう。しかし、それをするために、私たちにはまず`Option`型と`Result`型を調査する必要があるでしょう。
それらの方は両方とも、それらの中で定義される`unwrap`と呼ばれるメソッドを持ちます。

### `Option`型

`Option`型は[標準ライブラリー内で定義されます][5]。

```rust
enum Option<T> {
    None,
    Some(T),
}
```

`Option`型は *不存在の可能性* を表現するためのRustの型システムの使い方です。
不存在の可能性を型システムでコード化することは重要な概念です。なぜなら、それによってコンパイラーがプログラマーにその不存在を処理することを強制させるからです。
文字列から文字を見付けようとする例を見ましょう。

<span id="code-option-ex-string-find"></span>

```rust
// Searches `haystack` for the Unicode character `needle`. If one is found, the
// byte offset of the character is returned. Otherwise, `None` is returned.
fn find(haystack: &str, needle: char) -> Option<usize> {
    for (offset, c) in haystack.char_indices() {
        if c == needle {
            return Some(offset);
        }
    }
    None
}
```

この関数がマッチする文字を見付けたとき、単に`offset`を戻すのではないことに注意しましょう。
代わりにそれは`Some(offset)`を戻します。
`Some`はバリアント、つまり`Option`型の *値コンストラクター* です。
あなたはそれを`fn<T>(value: T) -> Option<T>`型の関数として考えることができます。
同様に、`None`も値コンストラクターですが、引数を持たない点が異なります。
あなたは`None`を`fn<T>() -> Option<T>`型の関数として考えることができます。

これは空騒ぎのように見えるかもしれませんが、これは話の半分でしかありません。
もう半分は、私たちの書いた`find`関数の *使い方* です。
ファイル名の中から拡張子を見付けるためにそれを使ってみましょう。

```rust
# fn find(_: &str, _: char) -> Option<usize> { None }
fn main() {
    let file_name = "foobar.rs";
    match find(file_name, '.') {
        None => println!("No file extension found."),
        Some(i) => println!("File extension: {}", &file_name[i+1..]),
    }
}
```

このコードは`find`関数から戻された`Option<usize>`に対して *場合分け* を行うために[パターンマッチ][1]を使います。
事実上、場合分けは`Option<T>`の中に保存された値を得るための唯一の方法です。
これはあなたがプログラマーとして、`Option<T>`が`Some(t)`ではなく`None`であった場合を処理しなければならないことを意味します。

しかし待ってください。私たちが[前に](#code-unwrap-double)使った`unwrap`はどうでしょうか。
そこには場合分けはありませんでした！　
代わりに、場合分けはあなたのために`unwrap`メソッドの中に入れられていました。
あなたは望むのであればそれをあなた自身で定義することができます。

<span id="code-option-def-unwrap"></span>

```rust
enum Option<T> {
    None,
    Some(T),
}

impl<T> Option<T> {
    fn unwrap(self) -> T {
        match self {
            Option::Some(val) => val,
            Option::None =>
              panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}
```

`unwrap`メソッドは *場合分けを抽象化します* 。
これはまさに`unwrap`の使い方を人間工学的にするものです。
残念ながら、その`panic!`は`unwrap`が組合せ可能ではないということを意味します。それは陶磁器屋の雄牛です。

### `Option<T>`値の組合せ

[先程の例](#code-option-ex-string-find)では、私たちはファイル名から拡張子を発見するための`find`の使い方を見ました。
もちろん、全てのファイル名が`.`をそれらの中に持っているわけではないので、ファイル名が拡張子を持っていない可能性もあります。
言い換えれば、コンパイラーは私たちに拡張子が存在しない可能性に対処することを強制するということです。
この例の場合、私たちはそのように言うメッセージを単にプリントアウトします。

ファイル名の拡張子を得ることはかなり一般的な作業なので、それを関数の中に入れることには意味があります。

```rust
# fn find(_: &str, _: char) -> Option<usize> { None }
// Returns the extension of the given file name, where the extension is defined
// as all characters proceeding the first `.`.
// If `file_name` has no `.`, then `None` is returned.
fn extension_explicit(file_name: &str) -> Option<&str> {
    match find(file_name, '.') {
        None => None,
        Some(i) => Some(&file_name[i+1..]),
    }
}
```
（専門家からのヒント：このコードは使わないこと。代わりに標準ライブラリー内の[`extension`](../std/path/struct.Path.html#method.extension)を使うこと。）

コードは単純なままですが、`find`の型が私たちに不存在の可能性を考慮することを強制することは注意すべき重要なことです。
それはコンパイラーがファイル名が拡張子を持たない場合についてうっかり忘れさせないつもりであることを意味するので、これはよいことです。
一方、私たちが`extension_explicit`で行ったような明示的な場合分けを毎回行うことは少し面倒になることがあります。

実際に、`extension_explicit`での場合分けは非常に一般的なパターンに従っています。 オプションが`None`でない限り、関数を`Option<T>`の中の値に *マップします* 。そうでない場合は、単に`None`を戻します。

Rustはパラメトリックポリモーフィズムを持っています。そのため、このパターンを抽象化するコンビネーターを定義することは非常に簡単です。

<span id="code-option-map"></span>

```rust
fn map<F, T, A>(option: Option<T>, f: F) -> Option<A> where F: FnOnce(T) -> A {
    match option {
        None => None,
        Some(value) => Some(f(value)),
    }
}
```

実際、`map`は標準ライブラリー内の`Option<T>`の[メソッドとして定義されます][2]。

私たちの新しいコンビネーターで武装して、私たちは場合分けをなくすために`extension_explicit`メソッドを書き直すことができます。

```rust
# fn find(_: &str, _: char) -> Option<usize> { None }
// Returns the extension of the given file name, where the extension is defined
// as all characters proceeding the first `.`.
// If `file_name` has no `.`, then `None` is returned.
fn extension(file_name: &str) -> Option<&str> {
    find(file_name, '.').map(|i| &file_name[i+1..])
}
```

私たちが非常に一般的だと考えるもう1つのパターンは、`Option`値が`None`であるときにデフォルト値を割り当てることです。
例えば、ファイルの拡張子がなかったとしても、あなたのプログラムはそれを`rs`とみなすかもしれません。
あなたが想像するように、これに対する場合分けはファイルの拡張子に特有のものではありません。それはどんな`Option<T>`でも使うことができます。

```rust
fn unwrap_or<T>(option: Option<T>, default: T) -> T {
    match option {
        None => default,
        Some(value) => value,
    }
}
```

ここでのトリックはデフォルト値が`Option<T>`の中にあるかもしれない値と同じ型を持たなければならないということです。
この場合、これの使い方は非常に簡単です。

```rust
# fn find(haystack: &str, needle: char) -> Option<usize> {
#     for (offset, c) in haystack.char_indices() {
#         if c == needle {
#             return Some(offset);
#         }
#     }
#     None
# }
#
# fn extension(file_name: &str) -> Option<&str> {
#     find(file_name, '.').map(|i| &file_name[i+1..])
# }
fn main() {
    assert_eq!(extension("foobar.csv").unwrap_or("rs"), "csv");
    assert_eq!(extension("foobar").unwrap_or("rs"), "rs");
}
```

（`unwrap_or`は標準ライブラリー内で`Option<T>`の[メソッドとして定義されるので][3]、私たちは前に定義した自立している関数の代わりにここではそれを使うということに注意しましょう。
もっと一般的な[`unwrap_or_else`][4]メソッドをチェックすることを忘れないでください。）

私たちが特別な注意を払う価値があると考えるもう1つのコンビネーター、`and_then`があります。
それは *不存在の可能性* を含む別個の計算を組み合わせやすくします。
例えば、このセクションの多くのコードは与えられたファイル名から拡張子を見付けることについてのものです。
これを行うために、あなたはまず、ファイル *パス* から一般的な方法で抽出されたファイル名を必要とします。
ほとんどのファイルパスはファイル名を持っていますが、それらの *全て* が持っているわけではありません。
例えば、`.`、`..`、`/`などがそうです。

そのため、私たちは与えられたファイル *パス* から拡張子を見付けることへの挑戦を課されます。
明示的な場合分けから始めましょう。

```rust
# fn extension(file_name: &str) -> Option<&str> { None }
fn file_path_ext_explicit(file_path: &str) -> Option<&str> {
    match file_name(file_path) {
        None => None,
        Some(name) => match extension(name) {
            None => None,
            Some(ext) => Some(ext),
        }
    }
}

fn file_name(file_path: &str) -> Option<&str> {
  // implementation elided
  unimplemented!()
}
```

あなたは場合分けを減らすために、私たちが単に`map`コンビネーターを使うことができると考えたかもしれません。しかし、その型が完全には適合していません。
すなわち、`map`は中身の値を持ったものに対してだけ何かを行う関数を受け取ります。
そのため、その関数の結果は *常に* [`Some`で再ラップされます](#code-option-map)。
実際に、私たちは`map`のような何かを必要としていますが、それは呼出元にもう1つの`Option`を戻すことを許します。
その一般的な実装は`map`よりももっと単純です。

```rust
fn and_then<F, T, A>(option: Option<T>, f: F) -> Option<A>
        where F: FnOnce(T) -> Option<A> {
    match option {
        None => None,
        Some(value) => f(value),
    }
}
```

今度は、私たちは`file_path_ext`関数を明示的な場合分けなしで書き直すことができます。

```rust
# fn extension(file_name: &str) -> Option<&str> { None }
# fn file_name(file_path: &str) -> Option<&str> { None }
fn file_path_ext(file_path: &str) -> Option<&str> {
    file_name(file_path).and_then(extension)
}
```

`Option`型は他にも[標準ライブラリー内で定義される][5]多くのコンビネーターを持ちます。
このリストに目を通して、使えるものをあなた自身の使いやすいようにまとめることはよい考えです。それらはあなたのために場合分けを減らしてくれることがしばしばあります。
多くのコンビネーターは（同じようなセマンティクスで）、私たちが次に話す`Result`のために定義されているので、それらのコンビネーターをあなた自身の使いやすいようにまとめることは役に立つでしょう。

コンビネーターは`Option`のような型の使い方を人間工学的にします。なぜなら、それらは明示的な場合分けを減らすからです。
それらは組合せ可能でもあります。なぜなら、それらは呼出元が自分たちの方法で不存在の可能性を処理することを許すからです。
`unwrap`のようなメソッドは選択肢を奪います。なぜなら、それらは`Option<T>`が`None`のときにパニックするからです。

## `Result`型

`Result`型も[標準ライブラリー内で定義されます][6]。

<span id="code-result-def"></span>

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Result`型は`Option`型のもっとリッチなバージョンです。
`Option`のように *不存在* の可能性を表現する代わりに、`Result`は *エラー* の可能性を表現します。
通常、 *エラー* はある計算の結果がなぜ失敗したのかを説明するために使われます。
これは`Option`のもっと一般的な形式です。
あらゆる意味で実際の`Option<T>`とセマンティックに等しい次の型エイリアスを考えましょう。

```rust
type Option<T> = Result<T, ()>;
```

これは`Result`の2つ目の型パラメーターを常に`()`（「ユニット」又は「空タプル」と発音します）に固定します。
`()`に存在する値は`()`ただ1つです。
（そう、この型レベルの用語と値レベルの用語とは同じ記法を使っています！）

`Result`型は計算の結果としてあり得る2つのうちの1つを表現する方法です。
慣習では、結果の1つが期待されるもの、つまり「`OK`」を意味して、他方の結果は期待されていないもの、つまり「`Err`」を意味します。

`Option`とちょうど同じように、`Result`型も標準ライブラリー内で[定義される`unwrap`メソッド][7]を持ちます。
それを定義してみましょう。

```rust
# enum Result<T, E> { Ok(T), Err(E) }
impl<T, E: ::std::fmt::Debug> Result<T, E> {
    fn unwrap(self) -> T {
        match self {
            Result::Ok(val) => val,
            Result::Err(err) =>
              panic!("called `Result::unwrap()` on an `Err` value: {:?}", err),
        }
    }
}
```

これは私たちの[`Option::unwrap`の定義](#code-option-def-unwrap)と事実上同じですが、`panic!`メッセージの中にエラー値を含む点が異なります。
これはデバッグを簡単にしますが、（私たちのエラー型を表現する）`E`型パラメーターに[`Debug`][8]制約を付けることを私たちに求めます。
型の大多数は`Debug`制約を満たすべきであるため、これは実際にはそうなっている傾向があります。
（型の`Debug`とは、単にその型の値の人間が読めるような説明をプリントするための合理的な方法を意味します。）

はい、例に進みましょう。

### 整数のパース

Rustの標準ライブラリーは文字列から整数への変換を非常に単純にします。
実際には、それは次のような感じのものを書きたくなってしまうくらい簡単です。

```rust
fn double_number(number_str: &str) -> i32 {
    2 * number_str.parse::<i32>().unwrap()
}

fn main() {
    let n: i32 = double_number("10");
    assert_eq!(n, 20);
}
```

この時点で、あなたは`unwrap`の呼出しに疑いを持つべきです。
例えば、もし文字列を数値としてパースしなければ、パニックが起きるでしょう。

```text
thread '<main>' panicked at 'called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }', /home/rustbuild/src/rust-buildbot/slave/beta-dist-rustc-linux/build/src/libcore/result.rs:729
```

これはかなり目障りですし、もしこれがあなたの使っているライブラリーの中で起これば、あなたは当然にいらいらするでしょう。
代わりに、私たちは私たちの関数の中でエラーの処理を試み、何をすべきかを呼出元に判断させるべきです。
これは`double_number`の戻り値の型を変えることを意味します。
しかし、何にでしょうか。
ええ、それには標準ライブラリー内の[`parse`メソッド`][9]のシグネチャーを見る必要があります。

```rust,ignore
impl str {
    fn parse<F: FromStr>(&self) -> Result<F, F::Err>;
}
```

うーん。
私たちは少なくとも`Result`を使う必要があるということを知りました。
確かに、これが`Option`を戻すことは可能です。
結局、文字列は数値としてパースできるか、できないかのどちらかですよね。
それは確かに採るべき合理的な方法ですが、実装は内部的に、 *なぜ* 文字列が整数としてパースできなかったのかということを区別します。
（それが空文字列なのか、数字として無効だったのか、大きすぎたのか、小さすぎたのかということです。）
そのため、`Result`を使うことには意味があります。私たちは単に「不存在」ということ以上の情報を提供したいからです。
私たちは *なぜ* パースが失敗したのかを知らせたいのです。
あなたが`Option`と`Result`のどちらかを選ばなければならなくなったとき、あなたは理由を示したこの行をエミュレートしようとすべきです。
もしあなたが詳細なエラーの情報を提供できるのであれば、そのときはおそらくそうすべきです。
（私たちはこの後でもっと見ることになります。）

はい、しかし私たちは戻り値の型をどう書いたらよいのでしょうか。
前に定義した`parse`メソッドは標準ライブラリー内で定義されている異なる数値型の全てに対して汎用的です。
私たちも関数を汎用的にしたい（そしておそらくはそうすべき）のですが、当面は明示性を大事にしましょう。
私たちは`i32`だけについて注目します。そのため、私たちは[その`FromStr`の実装を見付け](../std/primitive.i32.html)（あなたのブラウザーで「FromStr」を`CTRL-F`してください）、その[関連型][10]、`Err`を見る必要があります。
私たちは具体的なエラー型を見付けることができるようにこうしました。
この場合、それは[`std::num::ParseIntError`](../std/num/struct.ParseIntError.html)です。
最終的に、私たちは私たちの関数を書き直すことができます。

```rust
use std::num::ParseIntError;

fn double_number(number_str: &str) -> Result<i32, ParseIntError> {
    match number_str.parse::<i32>() {
        Ok(n) => Ok(2 * n),
        Err(err) => Err(err),
    }
}

fn main() {
    match double_number("10") {
        Ok(n) => assert_eq!(n, 20),
        Err(err) => println!("Error: {:?}", err),
    }
}
```
これは少しよくなりましたが、私たちは今度はたくさんコードを書いてしまいました！　
この場合分けは再び私たちを悩ませます。

これを助けるのがコンビネーターです。
ちょうど`Option`と同じように、`Result`もたくさんのコンビネーターをメソッドとして持っています。
`Result`と`Option`との間にはよく使われるコンビネーターの広い共通部分があります。
特に、`map`はその共通部分の一部です。

```rust
use std::num::ParseIntError;

fn double_number(number_str: &str) -> Result<i32, ParseIntError> {
    number_str.parse::<i32>().map(|n| 2 * n)
}

fn main() {
    match double_number("10") {
        Ok(n) => assert_eq!(n, 20),
        Err(err) => println!("Error: {:?}", err),
    }
}
```

常連さんはみんな`Result`にいます。そこには[`unwrap_or`](../std/result/enum.Result.html#method.unwrap_or)や[`and_then`](../std/result/enum.Result.html#method.and_then)も含まれます。
加えて、`Result`には2つ目の型パラメーターがあるので、（`map`の代わりとして）[`map_err`](../std/result/enum.Result.html#method.map_err)や（`and_then`の代わりとして）[`or_else`](../std/result/enum.Result.html#method.or_else)
のようなエラー型にだけ作用するコンビネーターもあります。

### `Result`の型エイリアスのイディオム

標準ライブラリー内で、あなたはよく`Result<i32>`というような型を見るかもしれません。
しかし待ってください、2つの型パラメーターを持つために[私たちは`Result`を定義しました](#code-result-def)。
私たちはどうすればどちらか1つだけを取り去ることができるのでしょうか。
その鍵は、型パラメーターのうちの1つを特定の型に *固定する* `Result`の型エイリアスを定義することです。
普通、固定される型はエラー型の方です。
例えば、整数をパースする私たちの前の例はこんな感じに書き直すことができます。

```rust
use std::num::ParseIntError;
use std::result;

type Result<T> = result::Result<T, ParseIntError>;

fn double_number(number_str: &str) -> Result<i32> {
    unimplemented!();
}
```

私たちはなぜこんなことをするのでしょうか。
ええ、もし私たちが`ParseIntError`を戻すことがある関数をたくさん持っていたら、私たちが毎回毎回それを書かなくてもよいように、いつも`ParseIntError`を使うエイリアスを定義することはもっと便利です。

標準ライブラリー内でこのイディオムが使われているところを最も見られるのは、[`io::Result`](../std/io/type.Result.html)です。
典型的には、`std::result`の普通の定義の代わりに`io::Result<T>`を書くことによって、あなたが`io`モジュールの型エイリアスを使っているということがはっきりします。
（このイディオムは[`fmt::Result`](../std/fmt/type.Result.html)でも使われます。）

## 間奏：アンラップは悪ではない

もしあなたが話を理解してくれていれば、あなたは私があなたのプログラムを`panic`させ、停止させてしまう`unwrap`のようなメソッドを呼び出すことに対してやや厳しい対応をしていたことに気付いたかもしれません。
*一般的に言えば* 、それはよい助言です。

しかし、`unwrap`は賢く使うこともできます。
`unwrap`の使用を厳密に正当化するものは、ややグレーな領域であり、合理的な人々は同意できません。
私はその問題についての私の *意見* のいくつかを要約します。

* **例の中や使い捨てで雑なコードの中。** あなたが例や使い捨てのプログラムを書いているときには、エラー処理が単に重要ではないことがあります。
   そのような状況では`unwrap`は非常に魅力的なので、便利さに打ち勝つのは困難なことがあります。
* **パニックがプログラムのバグを示すとき。** あなたのコードの維持している不変性が（例えば、空スタックからのポップのような）あるケースの発生を回避すべきとき、そこでのパニックは許容されます。
   これは、それがあなたのプログラムのバグを明らかにするからです。
   `assert!`の失敗、あるいは配列へのあなたのインデックスが境界を超えたことによって起こるように、これを明示的にすることができます。

これはおそらく網羅的なリストではありません。
さらに、`Option`を使うときにはその[`expect`](../std/option/enum.Option.html#method.expect)メソッドを使うほうがよいこともしばしばあります。
`expect`は`unwrap`とちょうど同じことをしますが、あなたが`expect`に与えたメッセージを出力する点が異なります。
これにより、それは「called unwrap on a `None` value.」の代わりにあなたのメッセージを表示するので、結果として起こるパニックは少しだけ処理しやすくなります。

私の助言を要約するとこうなります。よい判断をしましょう。
私の文書に「決してXをしてはいけません」とか「Yは危険なものだと考えられています」という言葉が出てこないことには理由があります。
全てのものはトレードオフであり、あなたの利用場面で何を受け入れるかは、プログラマーとしてのあなたに委ねられています。
私の目的はあなたがトレードオフをできる限り正確に評価できるように手助けすることだけです。

これで私たちはRustでのエラー処理の基本をカバーし、アンラップについて解説したので、標準ライブラリーををもっと調査し始めましょう。

# 複数のエラー型の利用

ここまで、私たちは`Option<T>`か`Result<T, SomeError>`のどちらかが全てであるエラー処理を見てきました。
しかし、あなたが`Option`と`Result`の両方を使う場合には何が起きるのでしょうか。
又は、もしあなたが`Result<T, Error1>`と`Result<T, Error2>`を使う場合にはどうでしょうか。
*異なるエラー型の組合せ* の処理は私たちの直面する次の課題です。それはこの章の残りの部分を通じた主要なテーマになります。

## `Option`と`Result`の組合せ

これまでに、私は`Option`のために定義されたコンビネーターと`Result`のために定義されたコンビネーターについて話しました。
私たちはこれらのコンビネーターを明示的な場合分けなしで別の計算の結果を組み合わせるために使うことができます。

もちろん、現実のコードでは、物事は常に純粋であるとは限りません。
あなたが`Option`型と`Result`型を混ぜたものを使うこともときどきあるでしょう。
私たちは明示的な場合分けに頼らなければならないのでしょうか。それとも、コンビネータを使い続けることができるのでしょうか。

とりあえず、この章の最初の例の1つを見直しましょう。

```rust,should_panic
use std::env;

fn main() {
    let mut argv = env::args();
    let arg: String = argv.nth(1).unwrap(); // error 1
    let n: i32 = arg.parse().unwrap(); // error 2
    println!("{}", 2 * n);
}
```

`Option`、`Result`、そしてそれらの様々なコンビネーターについて私たちが新しく得た知識によって、適切にエラーを処理し、エラーがなければプログラムがパニックしないように私たちはこれを書き直すべきです。

ここで扱いにくい点は、`arg.parse()`が`Result`を戻すのに、`argv.nth(1)`は`Option`を戻すということです。
これらは直接組み合わせることができません。
`Option`と`Result`の両方に直面したとき、その解決法は普通、`Option`を`Result`に変換することです。
この場合、（`env::args()`で）コマンドラインのパラメーターがないということはユーザーがプログラムを正しく実行しなかったということを意味します。
私たちはエラーを説明するために単に`String`を使うことができます。
やってみましょう。

<span id="code-error-double-string"></span>

```rust
use std::env;

fn double_arg(mut argv: env::Args) -> Result<i32, String> {
    argv.nth(1)
        .ok_or("Please give at least one argument".to_owned())
        .and_then(|arg| arg.parse::<i32>().map_err(|err| err.to_string()))
}

fn main() {
    match double_arg(env::args()) {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```
この例にはいくつか新しいものがあります。
1つ目は[`Option::ok_or`](../std/option/enum.Option.html#method.ok_or)の使用です。
これは`Option`を`Result`に変換する方法の1つです。
その変換のために、あなたは`Option`が`None`だった場合に使うエラーを指定しなければなりません。
私たちが見てきた他のコンビネーターと同様に、その定義は非常に単純です。

```rust
fn ok_or<T, E>(option: Option<T>, err: E) -> Result<T, E> {
    match option {
        Some(val) => Ok(val),
        None => Err(err),
    }
}
```

もう1つの新しいコンビネーターは[`Result::map_err`](../std/result/enum.Result.html#method.map_err)です。
これはちょうど`Result::map`と同じですが、`Result`値の *エラー* 部分に関数をマップする点が異なります。
もし`Result`が`Ok(...)`値であれば、それは何も変更せずに戻されます。

（`and_then`を使うために）エラー型を同じままにしておく必要があるので、私たちはここで`map_err`を使います。
私たちは（`argv.nth(1)`の）`Option<String>`を`Result<String, String>`に変換することにしたので、`arg.parse()`の`ParseIntError`も`String`に変換しなければなりません。

## コンビネーターの限界

IOや入力のパースは非常に一般的な仕事です。そして、それは私が個人的にRustで何度も行っていることです。
そのため、私たちはエラー処理の例を示すためにIOや様々なパースのルーチンを使います（そして使い続けるでしょう）。

単純なものから始めましょう。
私たちはファイルの内容を全て読み込み、その内容を数値に変換するために、ファイルを開かなければなりません。
そうして、私たちはそれに`2`を掛けて結果をプリントします。

私はあなたに`unwrap`を使わないよう納得させようとしましたが、最初に`unwrap`を使ってコードを書くのは便利なことがあります。
それによってあなたはエラー処理の代わりにあなたの問題に焦点を合わせることができるようになります。そして、それは適切なエラー処理を行う必要のある場所を明らかにします。
私たちがコードから手掛かりを得ることができるように、そこから始めましょう。それから、よりよいエラー処理ができるようにそれをリファクタリングしましょう。

```rust,should_panic
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> i32 {
    let mut file = File::open(file_path).unwrap(); // error 1
    let mut contents = String::new();
    file.read_to_string(&mut contents).unwrap(); // error 2
    let n: i32 = contents.trim().parse().unwrap(); // error 3
    2 * n
}

fn main() {
    let doubled = file_double("foobar");
    println!("{}", doubled);
}
```

（注意。[`std::fs::File::open`で使われている同じ境界](../std/fs/struct.File.html#method.open)があるので、`AsRef<Path>`が使われます。これによって全ての種類の文字列をファイルパスとして使うことが人間工学的にできるようになります。）

ここで起き得るエラーには3つのものがあります。

1. ファイルを開くときの問題
2. ファイルからデータを読み込むときの問題
3. データを数値としてパースするときの問題

最初の2つの問題は[`std::io::Error`](../std/io/struct.Error.html)型を使って説明されます。
[`std::fs::File::open`](../std/fs/struct.File.html#method.open)と[`std::io::Read::read_to_string`](../std/io/trait.Read.html#method.read_to_string)の戻り値のために、私たちはこれを知っています。
（それらは両方とも[`Result`の型エイリアスのイディオム](#the-result-type-alias-idiom)を使うということに注意しましょう。もしあなたが`Result`型について理解できていれば、[型エイリアスについても理解できるでしょう](../std/io/type.Result.html)。同時に、基礎にある`io::Error`型についても理解できるでしょう。）
3つ目の問題は[`std::num::ParseIntError`](../std/num/struct.ParseIntError.html)によって説明されます。
`io::Error`型は標準ライブラリーを通じて特に *広く使われています* 。
あなたはそれを何度も何度も見るでしょう。

`file_double`関数のリファクタリングの工程を始めましょう。
この関数をそのプログラムの他の部品と組み合せられるようにするには、もし前に挙げたようなエラーの状況に遭遇したとしても、それはパニックすべきでは *ありません* 。
事実上、これはもしどれかの作業が失敗したとしても、関数が *エラーを戻す* べきではないということを意味します。
私たちの問題は`file_double`の戻り値の型が`i32`であることです。私たちはそれからエラーを報告するための便利な方法を得ることはできません。
そのため、私たちは戻り値の型を`i32`から何か別のものに変えることから始めなければなりません。

私たちが最初に決める必要のあることは次のことです。私たちは`Option`と`Result`のどちらを使うべきでしょうか。
私たちは確かに`Option`を非常に簡単に使うことができました。
もし3つのエラーのうちのどれかが起きたならば、私たちは単に`None`を戻すことができます。
これは動きますし、 *それはパニックするよりはましですが* 、私たちはもっとうまくできます。
代わりに、私たちは発生したエラーについての詳細を渡すべきです。
私たちは *エラーの可能性* を表現したいので、`Result<i32, E>`を使うべきです。
しかし、`E`は何にすべきでしょうか。
2つの *異なる* エラーの型が起こり得るので、私たちはそれを共通の型に変換する必要があります。
そのような型の1つは`String`です。
それが私たちのコードにどう影響するかを見ましょう。

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, String> {
    File::open(file_path)
         .map_err(|err| err.to_string())
         .and_then(|mut file| {
              let mut contents = String::new();
              file.read_to_string(&mut contents)
                  .map_err(|err| err.to_string())
                  .map(|_| contents)
         })
         .and_then(|contents| {
              contents.trim().parse::<i32>()
                      .map_err(|err| err.to_string())
         })
         .map(|n| 2 * n)
}

fn main() {
    match file_double("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

このコードはちょっと難しそうに見えます。
このようなコードを書くことが簡単になるまでにはたくさんの実践を要することがあります。
私たちがそれを書いた方法は *型に従うこと* によっています。
私たちは`file_double`の戻り値の型を`Result<i32, String>`に変えてすぐに、正しいコンビネーターを探し始めなければなりません。
この場合、私たちは`and_then`、`map`、`map_err`という3つの異なるコンビネーターを使っているだけです。

`and_then`は複数の計算がそれぞれエラーを戻すことがあるときに、それらを連結するために使われます。
ファイルを開いた後、さらに2つの計算が失敗する可能性があります。ファイルを読み込むときと内容を数値としてパースするときです。
それに対応して、`and_then`が2回呼び出されます。

`map`は関数を`Result`の`Ok(...)`値に適用するために使われます。
例えば、一番最後の`map`の呼出しは`Ok(...)`値（それは`i32`です）を2倍します。
もしその点よりも前にエラーが起きていれば、`map`の定義の仕方から、この作業は飛ばされます。

`map_err`がこれをちゃんと動くようにするためのトリックです。
`map_err`は`map`とちょうど同じものですが、`Result`の`Err(...)`値に関数を適用する点が異なります。
この場合、私たちは全てのエラーを1つの型、`String`に変換したいわけです。
`io::Error`と`num::ParseIntError`は両方とも`ToString`を実装しているので、私たちはそれらを変換するために`to_string()`メソッドを呼び出すことができます。

これだけ言っても、このコードはまだ難しいです。
コンビネーターの使い方をマスターすることは重要ですが、それらには限界があります。
別のアプローチを試してみましょう。途中でのリターンです。

## 途中でのリターン

私は前のセクションからコードを持ってきて、それを *途中でのリターン* を使って書き直したいと思います。
途中でのリターンによってあなたは関数から途中で抜け出せるようになります。
私たちは`file_double`で他のクロージャーの中から途中で戻ることはできないので、明示的な場合分けに戻る必要があります。

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, String> {
    let mut file = match File::open(file_path) {
        Ok(file) => file,
        Err(err) => return Err(err.to_string()),
    };
    let mut contents = String::new();
    if let Err(err) = file.read_to_string(&mut contents) {
        return Err(err.to_string());
    }
    let n: i32 = match contents.trim().parse() {
        Ok(n) => n,
        Err(err) => return Err(err.to_string()),
    };
    Ok(2 * n)
}

fn main() {
    match file_double("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

合理的な人々はこのコードがコンビネーターを使ったコードよりもよいのかどうかについては同意しない可能性がありますが、あなたがコンビネーターによるアプローチに慣れていなければ、私からはこのコードの方が読みやすいように見えます。
それは`match`と`if let`による明示的な場合分けを使います。
もしエラーが起きれば、それは単に関数の実行を中止し、エラーを（文字列に変換して）戻します。

しかし、これは後戻りなのでしょうか。
前に私たちは、人間工学的なエラー処理の鍵は明示的な場合分けを減らすことだと言いました。しかし、私たちはここでは明示的な場合分けに戻りました。
明示的な場合分けを減らすには *複数* の方法があるということが分かります。
コンビネーターが唯一の方法ではないのです。

## `try!`マクロ

Rustでのエラー処理の礎石は`try!`マクロです。
`try!`マクロはコンビネーターのように場合分けを抽象化しますが、コンビネーターとは異なり、 *制御フロー* も抽象化します。
すなわち、それは前に見た *途中でのリターン* パターンを抽象化することができるのです。

これが`try!`マクロの定義を単純にしたものです。

<span id="code-try-def-simple"></span>

```rust
macro_rules! try {
    ($e:expr) => (match $e {
        Ok(val) => val,
        Err(err) => return Err(err),
    });
}
```

（[実際の定義](../std/macro.try!.html)はもう少し洗練されています。私たちは後で見るでしょう。）

`try!`マクロを使うことによって、私たちの最後の例を単純にすることが非常に簡単になります。
それは私たちのために場合分けと途中でのリターンを行っているので、私たちは読みやすい引き締まったコードを得ることができます。

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, String> {
    let mut file = try!(File::open(file_path).map_err(|e| e.to_string()));
    let mut contents = String::new();
    try!(file.read_to_string(&mut contents).map_err(|e| e.to_string()));
    let n = try!(contents.trim().parse::<i32>().map_err(|e| e.to_string()));
    Ok(2 * n)
}

fn main() {
    match file_double("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {}", err),
    }
}
```

[私たちの`try!`の定義](#code-try-def-simple)はまだ、`map_err`の呼出しを必要とします。
これは、エラー型を`String`に変換する必要がまだあるからです。
私たちがもうすぐ`map_err`の呼出しをなくす方法を学ぶということはよいニュースです！　私たちが`map_err`をなくすことができるようになる前に、私たちは標準ライブラリーのいくつかの重要なトレイトについてもう少し学ぶ必要があるということは悪いニュースです。

## 独自のエラー型の定義

私たちが標準ライブラリー内のエラートレイトのいくつかに立ち入る前に、前の例で私たちのエラー型として`String`を使うことを止めることによって私はこのセクションをまとめたいと思います。

エラーを文字列に変換することや、その場であなた独自のエラーを文字列で作ることは簡単なので、私たちが前の例で行ったように`String`を使うことは便利です。
しかし、`String`をあなたのエラーとして使うことにはいくつかのマイナス面があります。

1つ目のマイナス面は、エラーメッセージがあなたのコードを散らかしてしまいがちであるということです。
エラーメッセージを別の場所で定義することも可能ですが、あなたが異常なくらい訓練されていない限り、エラーメッセージをあなたのコードに組み込みたくなるでしょう。
実際、私たちはまさにこれを[前の例](#code-error-double-string)で行いました。

2つ目のマイナス面はもっと重要で、それは`String`によって *失われるものがある* ことです。
これはもし全てのエラーを文字列に変換したとすると、私たちが呼出元に渡すエラーは全く不透明なものになるということです。
`String`のエラーに呼出元ができる合理的なことは、単にそれをユーザーに見せることだけです。
確かに、エラー型を決定するするために文字列を調査するというのは堅牢ではありません。
（明らかに、このマイナス面は例えばアプリケーションなどとは対照的に、ライブラリーでより重要です。）

例えば、`io:Error`型は[`io:ErrorKind`](../std/io/enum.ErrorKind.html)を組み込みます。そしてそれはIO作業の間に起きた不具合を表現する *構造化されたデータ* です。
あなたはそのエラーに応じて異なる反応をしたいかもしれないので、これは重要です。
（例えば、`NotFound`エラーはエラーコードとともに終了し、エラーをユーザーに見せることを意味するかもしれない一方で、`BrokenPipe`エラーはあなたのプログラムをきれいに終了させることを意味するかもしれません。）
`io:ErrorKind`では呼出元がエラー型を場合分けで検査します。そしてその場合分けは`String`の中のエラーの詳細を解明しようとするよりも明らかに優れています。

ファイルから整数を読み込む私たちの前の例でエラー型として`String`を使う代わりに、私たちは *構造化されたデータ* でエラーを表現する私たち独自のエラー型を定義することができます。
私たちは呼出元が詳細を調査したいと思っているときのために、元となるエラーから情報を落とさないよう努力します。

*複数の可能性の中の1つ* を表現する理想的な方法は、`enum`を使って私たち独自の直和型を定義することです。
この例の場合、エラーは`io::Error`か`num::ParseIntError`かのどちらかなので、自然な定義が浮かびます。

```rust
use std::io;
use std::num;

// We derive `Debug` because all types should probably derive `Debug`.
// This gives us a reasonable human readable description of `CliError` values.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}
```

私たちのコードを微調整することは非常に簡単です。
エラーを文字列に変換する代わりに、私たちは対応する値コンストラクターを使ってそれらを私たちの`CliError`型に変換します。

```rust
# #[derive(Debug)]
# enum CliError { Io(::std::io::Error), Parse(::std::num::ParseIntError) }
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, CliError> {
    let mut file = try!(File::open(file_path).map_err(CliError::Io));
    let mut contents = String::new();
    try!(file.read_to_string(&mut contents).map_err(CliError::Io));
    let n: i32 = try!(contents.trim().parse().map_err(CliError::Parse));
    Ok(2 * n)
}

fn main() {
    match file_double("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {:?}", err),
    }
}
```

ここでの変更点は、（エラーを文字列に変換する）`map_err(|e| e.to_string())`を`map_err(CliError::Io)`、又は`map_err(CliError::Parse)`に切り替えたことだけです。
*呼出元* はユーザーへの報告の詳細さを決定することができます。
要するに、`CliError`のようなカスタム`enum`エラー型は呼出元に、エラーを説明する *構造化されたデータ* に加えて以前と同様の利便性を全て提供する一方で、エラー型として`String`を使うことは呼出元から選択肢を奪うのです。

`String`エラー型はいざというときには何とかしてくれるでしょうが、経験則としてはあなた独自のエラー型を定義することです。あなたがアプリケーションを書いているときには特にそうです。
もしあなたがライブラリーを書いているのであれば、呼出元から不必要に選択肢を奪わないように、あなた独自のエラー型を定義することがぜひとも選択されるべきです。

# エラー処理に使われる標準ライブラリー内のトレイト

標準ライブラリーは2つの統合されたトレイトをエラー処理のために定義します。[`std::error::Error`](../std/error/trait.Error.html)と[`std::convert::From`](../std/convert/trait.From.html)です。
`Error`はエラーを一般的に説明するために特に設計されている一方、`From`トレイトは2つの異なる型の間で値を変換するためのより一般的なルールを提供します。

## `Error`トレイト

`Error`トレイトは[標準ライブラリー内で定義されます](../std/error/trait.Error.html)。

```rust
use std::fmt::{Debug, Display};

trait Error: Debug + Display {
  /// A short description of the error.
  fn description(&self) -> &str;

  /// The lower level cause of this error, if any.
  fn cause(&self) -> Option<&Error> { None }
}
```

このトレイトは非常に汎用的です。なぜなら、それはエラーを表現する *全ての* 型について実装されることが意図されているからです。
これは私たちが後で見るように、組合せ可能なコードを書くために便利であることを証明します。
その他に、そのトレイトによって私たちは少なくとも次のことができるようになります。

* エラーの`Debug`表現を得る
* エラーのユーザーに見せる`Display`表現を得る
* （`description`メソッドを使って）エラーの短い説明を得る
* もし存在すれば、（`cause`メソッドを使って）エラーの因果関係の連鎖を調査する

前の2つは`Error`が`Debug`と`Display`の両方の実装を要求することの結果です。
後の2つは`Error`で定義される2つのメソッドから来ています。
`Error`の力は全てのエラー型が`Error`を実装するという事実に由来します。それは
エラーを[トレイトオブジェクト](../book/trait-objects.html)として存在定量化することができるということを意味します。
これは`Box<Error>`や`&Error`としても現れます。
実際、`cause`メソッドは`&Error`を戻します。これはそれ自身がトレイトオブジェクトです。
私たちは後で、`Error`トレイトのユーティリティーをトレイトオブジェクトとして再訪するでしょう。

とりあえず、`Error`トレイトを実装している例を見せることで十分です。
私たちが[前のセクション](#defining-your-own-error-type)で定義したエラー型を使いましょう。

```rust
use std::io;
use std::num;

// We derive `Debug` because all types should probably derive `Debug`.
// This gives us a reasonable human readable description of `CliError` values.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}
```

この特別なエラー型は2つのエラー型が生じ得ることを表現します。IOを扱うときのエラー、又は文字列を数値に変換するときのエラーです。
エラーは新しいバリアントを追加することであなたの欲しいだけたくさんのエラー型として表現することができます。

`Error`を実装することは非常に単純です。
それはほとんど明示的な場合分けになっていきます。

```rust,ignore
use std::error;
use std::fmt;

impl fmt::Display for CliError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            // Both underlying errors already impl `Display`, so we defer to
            // their implementations.
            CliError::Io(ref err) => write!(f, "IO error: {}", err),
            CliError::Parse(ref err) => write!(f, "Parse error: {}", err),
        }
    }
}

impl error::Error for CliError {
    fn description(&self) -> &str {
        // Both underlying errors already impl `Error`, so we defer to their
        // implementations.
        match *self {
            CliError::Io(ref err) => err.description(),
            // Normally we can just write `err.description()`, but the error
            // type has a concrete method called `description`, which conflicts
            // with the trait method. For now, we must explicitly call
            // `description` through the `Error` trait.
            CliError::Parse(ref err) => error::Error::description(err),
        }
    }

    fn cause(&self) -> Option<&error::Error> {
        match *self {
            // N.B. Both of these implicitly cast `err` from their concrete
            // types (either `&io::Error` or `&num::ParseIntError`)
            // to a trait object `&Error`. This works because both error types
            // implement `Error`.
            CliError::Io(ref err) => Some(err),
            CliError::Parse(ref err) => Some(err),
        }
    }
}
```

私たちはこれが非常に典型的な`Error`の実装だということに気付きます。あなたの異なったエラー型にマッチさせ、`description`と`cause`のために定義された約束を満たします。

## `From`トレイト

`std::convert::From`トレイトは[標準ライブラリー内で定義されます](../std/convert/trait.From.html)。

<span id="code-from-def"></span>

```rust
trait From<T> {
    fn from(T) -> Self;
}
```

すばらしく単純でしょう。
`From`は私たちに特定の型`T` *から* 何か別の型への 変換について話すための汎用的な方法を与えるので非常に便利です（この場合、「何か別の型」は実装の対象、又は`Self`です）。
`From`の要点は[標準ライブラリーによって提供される実装のセット](../std/convert/trait.From.html)です。

これが`From`の動き方を示したいくつかの単純な例です。

```rust
let string: String = From::from("foo");
let bytes: Vec<u8> = From::from("foo");
let cow: ::std::borrow::Cow<str> = From::from("foo");
```

はい、`From`は文字列から文字列への変換に便利です。
しかしエラーについてはどうでしょうか。
決定的な実装が1つあることが分かります。

```rust,ignore
impl<'a, E: Error + 'a> From<E> for Box<Error + 'a>
```

この実装は、`Error`を実装する *任意の* 型のために私たちはそれをトレイトオブジェクト`Box<Error>`に変換することができるということを示しています。
これはそこまで驚くことには思えないかもしれませんが、汎用的な文脈では便利です。

私たちが前に扱った2つのエラーを覚えていますか。
特に`io:Error`と`num::ParseIntError`です。
両方とも`Error`を実装するので、それらは`From`と使うことができます。

```rust
use std::error::Error;
use std::fs;
use std::io;
use std::num;

// We have to jump through some hoops to actually get error values.
let io_err: io::Error = io::Error::last_os_error();
let parse_err: num::ParseIntError = "not a number".parse::<i32>().unwrap_err();

// OK, here are the conversions.
let err1: Box<Error> = From::from(io_err);
let err2: Box<Error> = From::from(parse_err);
```

ここでは理解すべき本当に重要なパターンがあります。
`err1`と`err2`は両方とも *同じ型* を持ちます。
これは、それらが存在定量化された型、つまりトレイトオブジェクトだからです。
特に、それらの基礎にある型はコンパイラーの知識から *消去され* 、それは`err1`と`err2`を本当に全く同じものと考えます。
加えて、私たちは`err1`と`err2`をまさに同じ関数呼出し、`From::from`を使って構築しました。
これは、`From::from`がその引数とその戻り値を両方ともオーバーロードされたからです。

このパターンは私たちが前に抱えていた問題を解決するので重要です。これは私たちに同じ型のエラーを同じ関数を使って変換するための信頼できる方法を与えてくれます。

昔の友人、`try!`マクロを再訪するときです。

## 実際の`try!`マクロ

前に私たちは`try!`のこの定義を見せました。

```rust
macro_rules! try {
    ($e:expr) => (match $e {
        Ok(val) => val,
        Err(err) => return Err(err),
    });
}
```

これはその実際の定義ではありません。
その実際の定義は[標準ライブラリー内に](../std/macro.try!.html)あります。

<span id="code-try-def"></span>

```rust
macro_rules! try {
    ($e:expr) => (match $e {
        Ok(val) => val,
        Err(err) => return Err(::std::convert::From::from(err)),
    });
}
```

そこには小さくても強力な変更があります。エラー値は`From::from`を通して渡されます。
これによって`try!`マクロはずっと強力になります。なぜなら、それはあなたに無償で自動的な型変換を与えるからです。

私たちのより強力な`try!`マクロで武装して、私たちが前に書いたファイルを読み込み、その内容を整数に変換するコードを見ましょう。

```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, String> {
    let mut file = try!(File::open(file_path).map_err(|e| e.to_string()));
    let mut contents = String::new();
    try!(file.read_to_string(&mut contents).map_err(|e| e.to_string()));
    let n = try!(contents.trim().parse::<i32>().map_err(|e| e.to_string()));
    Ok(2 * n)
}
```

前に私たちは私たちが`map_err`の呼出しをなくすことができると約束しました。
実際、私たちがしなければならなかったことの全ては、`From`が使えるような型を選ぶことでした。
私たちが前のセクションで見たように、`From`は任意のエラー型を`Box<Error>`に変換する実装を持ちます。

```rust
use std::error::Error;
use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, Box<Error>> {
    let mut file = try!(File::open(file_path));
    let mut contents = String::new();
    try!(file.read_to_string(&mut contents));
    let n = try!(contents.trim().parse::<i32>());
    Ok(2 * n)
}
```

私たちは理想的なエラー処理にかなり近付いています。
私たちのコードにはエラー処理に起因する非常に小さなオーバーヘッドしかありません。なぜなら、`try!`マクロは3つのことを同時にカプセル化するからです。

1. 場合分け
2. 制御フロー
3. エラー型の変換

3つのこと全てが組み合せられるとき、私たちはコンビネーターによる`unwrap`の呼出しや場合分けから無傷のコードを得ます。

そこにはもう1つ小さなものが残っています。`Box<Error>`型は *不透明* です。
もし私たちが`Box<Error>`を呼出元に戻したとすると、呼出元は（簡単には）基礎となるエラー型を調査することができません。
呼出元は[`description`](../std/error/trait.Error.html#tymethod.description)や[`cause`](../std/error/trait.Error.html#method.cause)を呼び出すことができるので、その状況は`String`よりは確かにましです。しかし、`Box<Error>`が不透明であるという制限は残ります。
（注意。これは完全には正しくありません。なぜなら、Rustは実行時のリフレクションを持っているからです。それは[この章の範囲外の](https://crates.io/crates/error)いくつかのシナリオでは便利です。）

私たちのカスタム`CliError`型を再訪し、全てを結び付けるときが来ました。

## カスタムエラー型の組合せ

直前のセクションで、私たちは実際の`try!`マクロと、それがエラー値の`From::from`を呼び出して、私たちのためにどのように自動的な型変換を行うのかを見ました。
特に、私たちはエラーを`Box<Error>`に変換しました。それは動きますが、型は呼出元にとって不透明です。

これを直すために、私たちは私たちが既に慣れているものと同じ治療法、カスタムエラー型を使います。
再び、これはファイルの内容を読み込みそれを整数に変換するコードです。

```rust
use std::fs::File;
use std::io::{self, Read};
use std::num;
use std::path::Path;

// We derive `Debug` because all types should probably derive `Debug`.
// This gives us a reasonable human readable description of `CliError` values.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}

fn file_double_verbose<P: AsRef<Path>>(file_path: P) -> Result<i32, CliError> {
    let mut file = try!(File::open(file_path).map_err(CliError::Io));
    let mut contents = String::new();
    try!(file.read_to_string(&mut contents).map_err(CliError::Io));
    let n: i32 = try!(contents.trim().parse().map_err(CliError::Parse));
    Ok(2 * n)
}
```

私たちはまだ`map_err`を呼び出しているということに注意しましょう。
なぜでしょうか。
ええ、[`try!`](#code-try-def)と[`From`](#code-from-def)の定義を思い出しましょう。
問題は私たちが`io:Error`や`num:ParseIntError`のようなエラー型から私たち独自のカスタム`CliError`への変換を許す`From`の実装がないということです。
もちろん、これを直すのは簡単です！　
私たちは`CliError`を定義したので、それに`From`を実装することができます。

```rust
# #[derive(Debug)]
# enum CliError { Io(io::Error), Parse(num::ParseIntError) }
use std::io;
use std::num;

impl From<io::Error> for CliError {
    fn from(err: io::Error) -> CliError {
        CliError::Io(err)
    }
}

impl From<num::ParseIntError> for CliError {
    fn from(err: num::ParseIntError) -> CliError {
        CliError::Parse(err)
    }
}
```

それらの実装全てが行っていることは`From`に`CliError`を他のエラー型から作る方法を教えることです。
この例の場合、構築は対応する値コンストラクターの実行と同程度に単純です。
実際、それは *一般的に* 簡単です。

私たちはついに`file_double`を書き直すことができます。

```rust
# use std::io;
# use std::num;
# enum CliError { Io(::std::io::Error), Parse(::std::num::ParseIntError) }
# impl From<io::Error> for CliError {
#     fn from(err: io::Error) -> CliError { CliError::Io(err) }
# }
# impl From<num::ParseIntError> for CliError {
#     fn from(err: num::ParseIntError) -> CliError { CliError::Parse(err) }
# }

use std::fs::File;
use std::io::Read;
use std::path::Path;

fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, CliError> {
    let mut file = try!(File::open(file_path));
    let mut contents = String::new();
    try!(file.read_to_string(&mut contents));
    let n: i32 = try!(contents.trim().parse());
    Ok(2 * n)
}
```

私たちがここで行ったたった1つのことは`map_err`をなくすことでした。
それらはもう必要ありません。なぜなら、`try!`マクロがエラー型の`From::from`を実行するからです。
私たちは`From`の実装を現れ得る全てのエラー型に対して提供しているので、これは動きます。

もし私たちが私たちの`file_double`関数を、例えば文字列を浮動小数点数に変換するような別の作業を行うように変更すれば、私たちは私たちのエラー型に新しいバリアントを追加する必要があります。

```rust
use std::io;
use std::num;

enum CliError {
    Io(io::Error),
    ParseInt(num::ParseIntError),
    ParseFloat(num::ParseFloatError),
}
```

そして新しい`From`の実装を追加しましょう。

```rust
# enum CliError {
#     Io(::std::io::Error),
#     ParseInt(num::ParseIntError),
#     ParseFloat(num::ParseFloatError),
# }

use std::num;

impl From<num::ParseFloatError> for CliError {
    fn from(err: num::ParseFloatError) -> CliError {
        CliError::ParseFloat(err)
    }
}
```

これで終わりです！

## ライブラリー開発者への助言

もしあなたのライブラリーがカスタムエラーを報告しなければならないとすれば、あなたはおそらくあなた独自のエラー型を定義すべきでしょう。
その表現を（[`ErrorKind`](../std/io/enum.ErrorKind.html)のように）公開するか、又は（[`ParseIntError`](../std/num/struct.ParseIntError.html)のように）隠しておくかはあなたに委ねられます。
あなたがそれをどうするかにかかわらず、少なくともエラーについて単なる`String`による表現より多くの情報を提供することは一般的によい習慣です。
しかし、確かにこれは利用場面によって様々でしょう。

控えめに言っても、あなたはおそらく[`Error`](../std/error/trait.Error.html)トレイトを実装すべきです。
これはあなたのライブラリーのユーザーに[エラーを組み合わせる](#the-real-try-macro)ための最小限度の柔軟性を与えるでしょう。
`Error`トレイトを実装することはユーザーにエラーの文字列表現を得る能力が保証されることも意味します（なぜなら、それは`fmt::Debug`と`fmt::Display`の両方の実装を要求するからです）。

その上、あなたのエラー型に`From`の実装を提供することは便利なことがあります。
それはあなた（ライブラリー開発者）とあなたのユーザーに[もっと詳細なエラーを組み合わせる](#composing-custom-error-types)ことを許します。
例えば、[`csv::Error`](http://burntsushi.net/rustdoc/csv/enum.Error.html)は`From`の実装を`io::Error`と`byteorder::Error`の両方に対して提供します。

最終的に、あなたの好みによっては、あなたは[`Result`の型エイリアス](#the-result-type-alias-idiom)も定義したいと思うかもしれません。これはあなたのライブラリーがエラー型を1つだけ定義する場合に特にそうです。
これは標準ライブラリー内で[`io::Result`](../std/io/type.Result.html)と[`fmt::Result`](../std/fmt/type.Result.html)に対して使われています。

# 事例研究：人口データを読み込むプログラム

この章は長く、あなたのバックグラウンドによっては一層中身の濃いものになったかもしれません。
練習問題を追う十分なコード例がある一方で、それのほとんどは教育的見地から設計されたものでした。
そこで、私たちは何か新しいことをしてみようと思います。それは事例研究です。

このために、私たちはあなたが世界の人口データを問い合わせることができるコマンドラインプログラムを作ろうと思います。
目的は単純です。あなたがそれに場所を与えると、それはあなたに人口を教えます。
単純さにもかかわらず、間違えることのあるものがたくさんあります！

私たちが使うデータは[データサイエンスツールキット][11]によっています。
私はこの練習のためにそれからいくつかのデータを用意しています。
あなたは[世界の人口データ][12]（gzip圧縮時41MB、展開時145MB）、又は[USの人口データ][13]だけ（gzip圧縮時2.2MB、展開時7.2MB）を取得することができます。

今までのところ、私たちはコードをRustの標準ライブラリーに限定しています。
しかしこのような実際の仕事のためには、私たちは少なくともCSVデータをパースするため、プログラムの引数をパースするため、何かをRustの型に自動的にデコードするための何かを使いたいと思うでしょう。

そのために、私たちは[`csv`](https://crates.io/crates/csv)クレートと[`rustc-serialize`](https://crates.io/crates/rustc-serialize)クレートを使います。

## 初期セットアップ

私たちはCargoでプロジェクトをセットアップすることにたくさんの時間を費やそうとは思っていません。なぜなら、それは既に[Cargoの章](../book/hello-cargo.html)と[Cargoのドキュメント][14]で十分にカバーされているからです。

スクラッチから始めるために、`cargo new --bin city-pop`を実行し、あなたの`Cargo.toml`が次のようなものであることを確認しましょう。

```text
[package]
name = "city-pop"
version = "0.1.0"
authors = ["Andrew Gallant <jamslam@gmail.com>"]

[[bin]]
name = "city-pop"

[dependencies]
csv = "0.*"
rustc-serialize = "0.*"
getopts = "0.*"
```

あなたは既に実行できるようになっているべきです。

```text
cargo build --release
./target/release/city-pop
# Outputs: Hello, world!
```

## 引数のパース

引数のパースを片付けましょう。
私たちはGetoptsの詳細に立ち入るつもりはありませんが、それを説明した[よいドキュメント][15]があります。
手短に言えば、Getoptsは引数のパーサーとヘルプメッセージをオプションのベクターから生成します（実際にはベクターは構造体と一連のメソッドの後ろに隠されています）。
一度パースが終わると、私たちはプログラムの引数をRustの構造体にデコードすることができます。
そこから私たちはフラグについての情報を得ることができます。例えば、それらが渡されたのかどうか、それらがどんな引数を持っているのかということです。
これは適切な`extern crate`文とGetoptsの基本的な引数のセットアップを付けた私たちのプログラムです。

```rust,ignore
extern crate getopts;
extern crate rustc_serialize;

use getopts::Options;
use std::env;

fn print_usage(program: &str, opts: Options) {
    println!("{}", opts.usage(&format!("Usage: {} [options] <data-path> <city>", program)));
}

fn main() {
    let args: Vec<String> = env::args().collect();
    let program = args[0].clone();

    let mut opts = Options::new();
    opts.optflag("h", "help", "Show this usage message.");

    let matches = match opts.parse(&args[1..]) {
        Ok(m)  => { m }
	Err(e) => { panic!(e.to_string()) }
    };
    if matches.opt_present("h") {
        print_usage(&program, opts);
	return;
    }
    let data_path = args[1].clone();
    let city = args[2].clone();

	// Do stuff with information
}
```

まず、私たちは私たちのプログラムに渡された引数のベクターを得ます。
それから、1つ目は私たちのプログラム名だと知っているので、私たちはそれを保存します。
一度それが終わると、私たちは私たちの引数のフラグ、この場合は単純なヘルプメッセージのフラグをセットアップします。
一度私たちがセットアップされた引数のフラグを持つと、私たちは引数のベクター（インデックス0はプログラム名なのでインデックス1から始まります）をパースするために`Options.parse`を使います。
もしこれが成功したならば、私たちはパースされたオブジェクトにマッチを割り当て、成功しなければ、パニックします。
一度それを通過すると、私たちはユーザーがヘルプフラグを渡したかどうかをテストし、もし渡していれば、使い方のメッセージをプリントします。
ヘルプメッセージのオプションはGetoptsによって組み立てられるので、私たちがしなければならないことの全ては、私たちの欲しいものが使い方のメッセージをプリントすることだということをそれに教えることです。
もしユーザーがヘルプフラグを渡していなかったならば、私たちは適切な変数をそれらの対応する引数に割り当てます。

## ロジックの記述

私たちはどのようにコードを書くかということにおいてみんな違っています。しかし、エラー処理は普通は私たちが考えたいと思う最後のものです。
これはよい設計のためにはあまりよい習慣ではありません。しかし、それは迅速なプロトタイプのためには便利なことがあります。
この場合、Rustは私たちにエラー処理について明示することを強制するので、私たちのプログラムのどの部分がエラーを起こす可能性があるのかも明らかになるでしょう。
なぜでしょうか。
Rustは私たちに`unwrap`を呼び出させるからです！　
これは私たちにどのようなエラー処理の方法が必要かというよい鳥瞰図を与えることがあります。

この事例研究では、ロジックは非常に単純です。
私たちがする必要のあることの全ては、私たちに与えられたCSVデータをパースしてマッチする行のフィールドをプリントすることです。
それをしましょう。
（`extern crate csv;`があなたのファイルの一番上に追加されていることを確かめましょう。）

```rust,ignore
// This struct represents the data in each row of the CSV file.
// Type based decoding absolves us of a lot of the nitty gritty error
// handling, like parsing strings as integers or floats.
#[derive(Debug, RustcDecodable)]
struct Row {
    country: String,
    city: String,
    accent_city: String,
    region: String,

    // Not every row has data for the population, latitude or longitude!
    // So we express them as `Option` types, which admits the possibility of
    // absence. The CSV parser will fill in the correct value for us.
    population: Option<u64>,
    latitude: Option<f64>,
    longitude: Option<f64>,
}

fn print_usage(program: &str, opts: Options) {
    println!("{}", opts.usage(&format!("Usage: {} [options] <data-path> <city>", program)));
}

fn main() {
    let args: Vec<String> = env::args().collect();
    let program = args[0].clone();

    let mut opts = Options::new();
    opts.optflag("h", "help", "Show this usage message.");

    let matches = match opts.parse(&args[1..]) {
        Ok(m)  => { m }
		Err(e) => { panic!(e.to_string()) }
    };

    if matches.opt_present("h") {
        print_usage(&program, opts);
		return;
	}

	let data_file = args[1].clone();
	let data_path = Path::new(&data_file);
	let city = args[2].clone();

	let file = fs::File::open(data_path).unwrap();
	let mut rdr = csv::Reader::from_reader(file);

	for row in rdr.decode::<Row>() {
		let row = row.unwrap();

		if row.city == city {
			println!("{}, {}: {:?}",
				row.city, row.country,
				row.population.expect("population count"));
		}
	}
}
```

エラーを概説しましょう。
私たちは明らかなことから始めることができます。`unwrap`が呼ばれる場所は3つあります。

1. [`fs::File::open`](../std/fs/struct.File.html#method.open)は[`io::Error`](../std/io/struct.Error.html)を戻す可能性がある
2. [`csv::Reader::decode`](http://burntsushi.net/rustdoc/csv/struct.Reader.html#method.decode)は1回で1件のレコードを戻し、[レコードのデコードでは](http://burntsushi.net/rustdoc/csv/struct.DecodedRecords.html)（`Iterator`の実装の`Item`関連型を見ましょう）[`csv::Error`](http://burntsushi.net/rustdoc/csv/enum.Error.html)が発生する可能性がある
3. もし`row.population`が`None`であれば、`expect`の呼出しはパニックする

それらの他にあるでしょうか。
私たちがマッチする都市を見付けられなければどうなるでしょうか。
`grep`のようなツールはエラーコードを戻すので、私たちもおそらく同じようにすべきです。
私たちは私たちの問題特有のロジックエラー、IOエラーとCSVパースエラーを持っています。
私たちはそれらのエラーを処理する2つの異なる方法を調査しようと思います。

私たちは`Box<Error>`から始めたいと思います。
後で、私たちはどうすれば私たち独自のエラー型の定義を便利にすることができるかを考えます。

## `Box<Error>`を使ったエラー処理

`Box<Error>`は *ただ動くだけ* なので、よいものです。
あなたはあなた独自のエラー型を定義する必要がなく、`From`の実装を必要としません。
マイナス面は、`Box<Error>`はトレイトオブジェクトなので、 *型を消す* ことです。それは、コンパイラーがその基礎となる型をについてもう推論することができなくなることを意味します。

[前に](#the-limits-of-combinators)私たちは私たちの関数の型を`T`から`Result<T, OurErrorType>`に変えることで私たちのコードをリファクタリングし始めました。
この場合、`OurErrorType`は単なる`Box<Error>`です。
しかし`T`は何でしょうか。
そして、私たちは`main`に戻り値の型を追加することができるのでしょうか。

2番目の質問に対する答えはノーです。私たちはできません。
それは私たちに新しい関数を書く必要があるということを意味します。
しかし`T`は何でしょうか。
私たちにできる最も単純なことは、`Vec<Row>`としてマッチする`Row`値のリストを戻すことです。
（もっとよいコードはイテレーターを戻すでしょう。しかし、それは読者への練習問題として置いておきます。）

私たちのコードをそれ独自の関数にリファクタリングしましょう。ただし、`unwrap`の呼出しはそのままにしておきましょう。
単純にその行を無視することで人口のカウントを失う可能性を扱うことを私たちは選ぶということに注意しましょう。

```rust,ignore
struct Row {
    // unchanged
}

struct PopulationCount {
    city: String,
    country: String,
    // This is no longer an `Option` because values of this type are only
    // constructed if they have a population count.
    count: u64,
}

fn print_usage(program: &str, opts: Options) {
    println!("{}", opts.usage(&format!("Usage: {} [options] <data-path> <city>", program)));
}

fn search<P: AsRef<Path>>(file_path: P, city: &str) -> Vec<PopulationCount> {
    let mut found = vec![];
    let file = fs::File::open(file_path).unwrap();
    let mut rdr = csv::Reader::from_reader(file);
    for row in rdr.decode::<Row>() {
        let row = row.unwrap();
        match row.population {
            None => { } // skip it
            Some(count) => if row.city == city {
                found.push(PopulationCount {
                    city: row.city,
                    country: row.country,
                    count: count,
                });
            },
        }
    }
    found
}

fn main() {
	let args: Vec<String> = env::args().collect();
	let program = args[0].clone();

	let mut opts = Options::new();
	opts.optflag("h", "help", "Show this usage message.");

	let matches = match opts.parse(&args[1..]) {
		Ok(m)  => { m }
		Err(e) => { panic!(e.to_string()) }
	};
	if matches.opt_present("h") {
		print_usage(&program, opts);
		return;
	}

	let data_file = args[1].clone();
	let data_path = Path::new(&data_file);
	let city = args[2].clone();
	for pop in search(&data_path, &city) {
		println!("{}, {}: {:?}", pop.city, pop.country, pop.count);
	}
}

```

私たちは`expect`（これは`unwrap`のよりよいバリアントです）の使用をなくす一方で、まだ検索結果の不存在を処理するべきです。

これを適切なエラー処理に変換するために、私たちは次のことをする必要があります。

1. `search`の戻り値の型を`Result<Vec<PopulationCount>, Box<Error>>`に変更する
2. プログラムをパニックさせる代わりにエラーを呼出元に戻すため、[`try!`マクロ](#code-try-def)を使う
3. `main`内でエラーを処理する

それをしましょう。

```rust,ignore
fn search<P: AsRef<Path>>
         (file_path: P, city: &str)
         -> Result<Vec<PopulationCount>, Box<Error+Send+Sync>> {
    let mut found = vec![];
    let file = try!(fs::File::open(file_path));
    let mut rdr = csv::Reader::from_reader(file);
    for row in rdr.decode::<Row>() {
        let row = try!(row);
        match row.population {
            None => { } // skip it
            Some(count) => if row.city == city {
                found.push(PopulationCount {
                    city: row.city,
                    country: row.country,
                    count: count,
                });
            },
        }
    }
    if found.is_empty() {
        Err(From::from("No matching cities with a population were found."))
    } else {
        Ok(found)
    }
}
```

`x.unwrap()`の代わりに私たちは今度は`try!(x)`を持ちます。
私たちの関数は`Result<T, E>`を戻すので、エラーが発生した場合に`try!`マクロは関数の途中でリターンするでしょう。

このコードには1つ大きな注意すべき点があります。私たちは`Box<Error>`の代わりに`Box<Error + Send + Sync>`を使ったということです。
私たちは普通の文字列をエラー型に変換できたので、これをしました。
私たちは[対応する`From`の実装](../std/convert/trait.From.html)を使うことができるように、それらの余分なバウンドを必要とします。

```rust,ignore
// We are making use of this impl in the code above, since we call `From::from`
// on a `&'static str`.
impl<'a, 'b> From<&'b str> for Box<Error + Send + Sync + 'a>

// But this is also useful when you need to allocate a new string for an
// error message, usually with `format!`.
impl From<String> for Box<Error + Send + Sync>
```

私たちはどのように`Box<Error>`によって適切なエラー処理を行うかを見たので、私たち独自のカスタムエラー型による異なったアプローチに挑戦しましょう。
しかしまず、エラー処理は少しお休みして`stdin`からの読込みのサポートを追加しましょう。

## 標準入力からの読込み

私たちのプログラムでは、私たちは1つのファイルを入力として受け取り、データを1パスで処理します。
これは私たちが入力を標準入力から受け取ることができるようにすべきだということをおそらく意味します。
しかしもしかしたら、私たちは今の形式も好きかもしれません。では、両方使えるようにしましょう！

標準入力のサポートの追加は実際には非常に簡単です。
私たちがしなければならないことはたった3つです。

1. 人口データを標準入力から読み込むので、1つの引数、都市のみでも受け取ることができるように、プログラムの引数を微調整する
2. もしそれが標準入力から渡されなければ、オプション`-f`がファイルを受け取ることができるように、プログラムを変更する
3. *オプションの* ファイルパスを受け取るように`search`関数を変更する。`None`のときには、それは標準入力から読み込むことを理解すべきである

まず、これが新しい使い方です。

```rust,ignore
fn print_usage(program: &str, opts: Options) {
	println!("{}", opts.usage(&format!("Usage: {} [options] <city>", program)));
}
```

次の部分はちょっとだけ難しくなります。

```rust,ignore
...
let mut opts = Options::new();
opts.optopt("f", "file", "Choose an input file, instead of using STDIN.", "NAME");
opts.optflag("h", "help", "Show this usage message.");
...
let file = matches.opt_str("f");
let data_file = file.as_ref().map(Path::new);

let city = if !matches.free.is_empty() {
	matches.free[0].clone()
} else {
	print_usage(&program, opts);
	return;
};

for pop in search(&data_file, &city) {
	println!("{}, {}: {:?}", pop.city, pop.country, pop.count);
}
...
```

コードのこの部分では、私たちは（`Option<String>`型を持つ）`file`を受け取り、それを`search`が使うことのできる型、この場合は`&Option<AsRef<Path>>`に変換します。
これを行うため、私たちはファイルの参照を受け取り、それに`Path::new`をマップします。
この場合、`as_ref()`は`Option<String>`を`Option<&str>`に変換し、そこから私たちはオプションの内容に対して`Path::new`を実行し、新しい値のオプションを戻すことができます。
一度私たちがそれを持つと、`city`引数を得て`search`を実行することは単純なことです。

`search`の変更はやや難しいです。
`csv`クレートは[`io::Read`を実装している任意の型](http://burntsushi.net/rustdoc/csv/struct.Reader.html#method.from_reader)からパーサーを組み立てることができます。
しかし、どのようにして私たちは両方の型に対して同じコードを使うことができるのでしょうか。
実際には、私たちがこれに対して行うことのできる方法にはいくつかあります。
方法の1つは、`io::Read`を満たす型パラメーター`R`について汎用的であるような`search`を書くことです。
もう1つの方法は単にトレイトオブジェクトを使うことです。

```rust,ignore
fn search<P: AsRef<Path>>
         (file_path: &Option<P>, city: &str)
         -> Result<Vec<PopulationCount>, Box<Error+Send+Sync>> {
    let mut found = vec![];
    let input: Box<io::Read> = match *file_path {
        None => Box::new(io::stdin()),
        Some(ref file_path) => Box::new(try!(fs::File::open(file_path))),
    };
    let mut rdr = csv::Reader::from_reader(input);
    // The rest remains unchanged!
}
```

## カスタム型を使ったエラー処理

前に私たちは[カスタムエラー型を使ってエラーを組み合わせる](#composing-custom-error-types)方法を学びました。
私たちは私たちのエラー型を`enum`として定義したり、`Error`や`From`を実装したりすることによってこれを行いました。

私たちは3つの異なるエラー（IO、CSVパース、不存在）を持つので、3つのバリアントを持つ`enum`を定義しましょう。

```rust,ignore
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Csv(csv::Error),
    NotFound,
}
```

そして今度は`Display`と`Error`に対する実装を取り上げます。

```rust,ignore
impl fmt::Display for CliError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            CliError::Io(ref err) => err.fmt(f),
            CliError::Csv(ref err) => err.fmt(f),
            CliError::NotFound => write!(f, "No matching cities with a \
                                             population were found."),
        }
    }
}

impl Error for CliError {
    fn description(&self) -> &str {
        match *self {
            CliError::Io(ref err) => err.description(),
            CliError::Csv(ref err) => err.description(),
            CliError::NotFound => "not found",
        }
    }
}
```

私たちが私たちの`CliError`型を私たちの`search`関数で使うことができるまで、私たちはいくつかの`From`の実装を提供する必要がありました。
私たちはどの実装を提供するかをどうやって知るのでしょうか。
ええ、私たちは`io::Error`と`CSV::Error`の両方を`CliError`に変換する必要があります。
そこには外部のエラーしかないので、私たちはとりあえず2つの`From`の実装を必要とします。

```rust,ignore
impl From<io::Error> for CliError {
    fn from(err: io::Error) -> CliError {
        CliError::Io(err)
    }
}

impl From<csv::Error> for CliError {
    fn from(err: csv::Error) -> CliError {
        CliError::Csv(err)
    }
}
```

どのように[`try!`が定義されるのか](#code-try-def)ということから、この`From`の実装は重要です。
特に、もしエラーが発生すると`From::from`がエラーに呼び出され、この場合これはそれを私たち独自のエラー型`CliError`に変換します。

`From`の実装が終わると、私たちは2つの微調整を私たちの`search`関数に行う必要があるだけです。それは戻り値の型と「見付かりませんでした」エラーです。
これが全てです。

```rust,ignore
fn search<P: AsRef<Path>>
         (file_path: &Option<P>, city: &str)
         -> Result<Vec<PopulationCount>, CliError> {
    let mut found = vec![];
    let input: Box<io::Read> = match *file_path {
        None => Box::new(io::stdin()),
        Some(ref file_path) => Box::new(try!(fs::File::open(file_path))),
    };
    let mut rdr = csv::Reader::from_reader(input);
    for row in rdr.decode::<Row>() {
        let row = try!(row);
        match row.population {
            None => { } // skip it
            Some(count) => if row.city == city {
                found.push(PopulationCount {
                    city: row.city,
                    country: row.country,
                    count: count,
                });
            },
        }
    }
    if found.is_empty() {
        Err(CliError::NotFound)
    } else {
        Ok(found)
    }
}
```

他に必要な変更はありません。

## 機能追加

汎用的なコードを書くことはすばらしいです。なぜなら、物事の一般化はクールで、それは後で便利になる可能性があるからです。
しかし、ときどき努力する価値のないことがあります。
私たちが前のステップで行ったことを見ましょう。

1. 新しいエラー型の定義
2. `Error`、`Display`の実装、そして2つの`From`の実装の追加

ここでの大きなマイナス面は、私たちのプログラムがそれほどよくはならなかったということです。
エラーを`enum`で表現することはかなりのオーバーヘッドです。このような短いプログラムでは特にそうです。

私たちがここで行ったようなカスタムエラー型の使用の便利な面の *1つ* は `main`関数が違う方法でエラーを処理することを選択できるということです。
前は`Box<Error>`を使いましたが、それは多くの選択肢を持ってはいませんでした。それはメッセージをプリントするだけです。
私たちはまだここではそうしていますが、私たちが望むもの、例えば`--quiet`フラグを追加したいときにはどうでしょうか。
`--quiet`フラグは全ての冗長な出力を沈黙させるべきです。

今は、もしプログラムがマッチを見付けられなければ、それはそのように言うメッセージを出力するでしょう。
これは少し使いにくい可能性があります。あなたがプログラムをシェルスクリプトで使うことを意図するのであれば特にそうです。

フラグを追加することから始めましょう。
前のように私たちは使い方の文字列を微調整し、オプション変数にフラグを追加する必要があります。
一度私たちがそれを終えれば、Getoptsが残りを実行します。

```rust,ignore
...
let mut opts = Options::new();
opts.optopt("f", "file", "Choose an input file, instead of using STDIN.", "NAME");
opts.optflag("h", "help", "Show this usage message.");
opts.optflag("q", "quiet", "Silences errors and warnings.");
...
```

今度は、私たちは「沈黙させる」機能を追加する必要があるだけです。
これは私たちに`main`の中の場合分けの微調整を要求します。

```rust,ignore
match search(&args.arg_data_path, &args.arg_city) {
    Err(CliError::NotFound) if args.flag_quiet => process::exit(1),
    Err(err) => fatal!("{}", err),
    Ok(pops) => for pop in pops {
        println!("{}, {}: {:?}", pop.city, pop.country, pop.count);
    }
}
```

確かに、もしIOエラーがあったり、データのパースに失敗したりした場合には、私たちは沈黙したいとは思いません。
そのため、私たちはエラー型が`NotFound`で、 *かつ* `--quiet`が有効かどうかをチェックする場合分けを使います。
もし検索が失敗すれば、（`grep`の慣習に従って）終了コードとともに終了します。

もし私たちが`Box<Error>`に固執していれば、`--quiet`機能の実装はかなり難しかったでしょう。

これが私たちの事例研究の大体の概要です。
ここから、あなたは世界に飛び出し、あなた独自のプログラムやライブラリーを適切なエラー処理とともに書く準備ができていることでしょう。

# 短編小説

この章は長いですが、Rustでのエラー処理の短い要約を持つことは便利です。
いくつかのよい「経験則」があります。
それらは命令では全く *ありません* 。
それらの経験則の1つ1つを破るためのよい理由がおそらくあります！

* もしあなたがエラー処理が過度な負担になるであろう短いコード例を書いていれば、`unwrap`(それが[`Result::unwrap`](../std/result/enum.Result.html#method.unwrap)であろうと[`Option::unwrap`](../std/option/enum.Option.html#method.unwrap)であろうと、又は望めるならば[`Option::expect`](../std/option/enum.Option.html#method.expect)）を使うのがおそらくちょうどよい。
   あなたのコードの利用者は適切なエラー処理の使い方を知るべきである。
   （もし彼らが知らなければ、それらをここに送ること！）
* もしあなたが使い捨てで雑なプログラムを書いているのであれば、あなたが`unwrap`を使ったとしても恥じ入らないこと。
   注意：もしそれが誰か他の人の手で終了するのであれば、彼らが貧相なエラーメッセージにいらいらしたとしても驚かないこと！　
* もしあなたが使い捨てで雑なプログラムを書いていて、パニックすることについて恥じ入るのであれば、あなたのエラー型に`String`か`Box<Error + Send + Sync>`を使うこと（`Box<Error + Send + Sync>`型は`From`の実装で使うことができるから）
* そうでなければ、[`try!`](../std/macro.try!.html)マクロをより人間工学的にするために、プログラムでは適切な[`From`](../std/convert/trait.From.html)と[`Error`](../std/error/trait.Error.html)の実装とともにあなた独自のエラー型を定義すること
* もしあなたがライブラリーを書いていて、あなたのコードがエラーを発生させる可能性があるのであれば、あなた独自のエラー型を定義し、[`std::error::Error`](../std/error/trait.Error.html)トレイトを実装すること。
   適切な場所では、あなたのライブラリーのコードと呼出元のコードの両方を書きやすくするために[`From`](../std/convert/trait.From.html)を実装すること。
   （Rustの一貫したルールのために、呼出元はあなたのエラー型に対して`From`を実装することはできないであろうから、あなたのライブラリーがそれを行うべきである）
* [`Option`](../std/option/enum.Option.html)と[`Result`](../std/result/enum.Result.html)を定義したコンビネーターを学ぶこと。
   それらを使うことはときどき退屈である可能性があるが、私は個人的に`try!`とコンビネーターを非常に魅力的なものにするたくさんの組合せを見付けている。
  `and_then`、`map`、`unwrap_or`は私のお気に入りである

[1]: ../book/patterns.html
[2]: ../std/option/enum.Option.html#method.map
[3]: ../std/option/enum.Option.html#method.unwrap_or
[4]: ../std/option/enum.Option.html#method.unwrap_or_else
[5]: ../std/option/enum.Option.html
[6]: ../std/result/
[7]: ../std/result/enum.Result.html#method.unwrap
[8]: ../std/fmt/trait.Debug.html
[9]: ../std/primitive.str.html#method.parse
[10]: ../book/associated-types.html
[11]: https://github.com/petewarden/dstkdata
[12]: http://burntsushi.net/stuff/worldcitiespop.csv.gz
[13]: http://burntsushi.net/stuff/uscitiespop.csv.gz
[14]: http://doc.crates.io/guide.html
[15]: http://doc.rust-lang.org/getopts/getopts/index.html
