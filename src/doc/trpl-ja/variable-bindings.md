% 変数束縛

「Hello World」以外の事実上すべてのRustのプログラムは *変数束縛* を使います。
それらはある値を名前に束縛し、それを後で使えるようにします。
`let`はこのように、束縛を導入するために使われます。

```rust
fn main() {
    let x = 5;
}
```

`fn main() {`をそれぞれの例に付けるのはやや冗長なので、私たちは今後、それを省略します。
もしあなたが実際に入力しているのであれば、`main()`関数を省略するのではなく、それを付けるのを忘れないようにしましょう。

# パターン

多くの言語では、変数束縛は *変数* と呼ばれます。しかし、Rustの変数束縛にはいくつか秘密兵器があります。
例えば、`let`式の左辺は単なる変数名ではなく、「[パターン][pattern]」です。
これは、私たちにはこのようなことができるということです。

```rust
let (x, y) = (1, 2);
```

この式が評価された後、`x`は1になり、`y`は2になります。
パターンは本当に強力で、本書には[それらのための独立したセクション][pattern]があります。
今のところ、私たちにはそれらの機能は必要ないので、私たちは先に進むためにこれを記憶の片隅にとどめておくだけにします。

[pattern]: patterns.html

# 型注釈

Rustは静的型付言語です。これは、私たちが型を事前に指定し、それらがコンパイル時にチェックされるということを意味します。
それでは、なぜ私たちの最初の例はコンパイルできたのでしょうか。
ええ、Rustには「型推論」と呼ばれるものがあるのです。
もしあるものの型が何であるかをそれが理解できるのであれば、Rustは実際に型を入力することをあなたに要求しません。

しかし、私たちは望むのであれば型を付けることもできます。
コロン（`:`）の後に入力しましょう。

```rust
let x: i32 = 5;
```

もし私があなたに、これを教室のみんなに向かって大きな声で読みなさいと言ったならば、あなたは「`x`は`i32`型の束縛で、その値は`5`です。」と言うでしょう。

この場合、私たちは`x`を表現するために32ビット符号付整数を選んでいます。
Rustにはたくさんの異なった整数のプリミティブ型があります。
それらが`i`から始まれば符号付き、`u`から始まれば符号なしです。
利用可能な整数のサイズは8、16、32、64ビットです。

後の例では、私たちはコメントに型注釈を付けることがあるかもしれません。
この例はこのようになります。

```rust
fn main() {
    let x = 5; // x: i32
}
```

この注釈とあなたが`let`で使った文法の間の類似性に注意しましょう。
それらの種類のコメントを付けることはRustでは慣習的ではありません。しかし、私たちはRustが推論した型が何であるのかをあなたが理解することを助けるためにときどきそれらを付けます。

# ミュータビリティー

デフォルトで束縛は *イミュータブル* です。
このコードはコンパイルできません。

```rust,ignore
let x = 5;
x = 10;
```

これは次のエラーを出します。

```text
error: re-assignment of immutable variable `x`
     x = 10;
     ^~~~~~~
```

もしあなたが束縛をミュータブルにしたいのであれば、あなたは`mut`を使うことができます。

```rust
let mut x = 5; // mut x: i32
x = 10;
```

束縛がデフォルトでイミュータブルであることの理由は1つではありません。しかし、私たちはRustの重要な焦点を通じてそれについて考えることができます。安全性です。
もしあなたが`mut`と言うのを忘れれば、コンパイラーはそれを検出し、変更することを意図していないものを変更していることをあなたに知らせます。
もし束縛がデフォルトでミュータブルだったら、コンパイラーはこれをあなたに知らせることができません。
もしあなたが変更を意図して *いなかった* ならば、解決法は非常に簡単です。`mut`を付けましょう。

可能であればミュータブルな状態を避けるということには他にもよい理由がありますが、それらはこのガイドの範囲外になります。
一般的に、あなたはしばしば明示的な変更を避けることができ、Rustではそれが望ましいです。
それはときどきこのように言われます。変更があなたの必要とするものならば、それは禁じられてはいません。

# 束縛の初期化

Rustの変数束縛には他の言事は異なるもう1つの側面があります。束縛は、あなたがそれらの使用を許される前に値で初期化される必要があるということです。

それを試しましょう。
あなたの`src/main.rs`ファイルをこのような感じに変更しましょう。

```rust
fn main() {
    let x: i32;

    println!("Hello world!");
}
```

あなたはそれをビルドするためにコマンドラインで`cargo build`を使うことができます。
警告が出ますが、それはまだ「Hello, world!」を出力します。

```text
   Compiling hello_world v0.0.1 (file:///home/you/projects/hello_world)
src/main.rs:2:9: 2:10 warning: unused variable: `x`, #[warn(unused_variable)]
   on by default
src/main.rs:2     let x: i32;
                      ^
```

Rustは私たちに、その変数束縛が全く使われていないことを警告します。しかし、私たちはそれを全く使っていないのですから、怪我がなければファウルではありません。
しかし、もし私たちが実際にこの`x`を使おうとするならば、話は変わります。
そうしましょう。
あなたのプログラムをこのような感じに変えましょう。

```rust,ignore
fn main() {
    let x: i32;

    println!("The value of x is: {}", x);
}
```

そして、ビルドしてみましょう。
エラーが出ます。

```bash
$ cargo build
   Compiling hello_world v0.0.1 (file:///home/you/projects/hello_world)
src/main.rs:4:39: 4:40 error: use of possibly uninitialized variable: `x`
src/main.rs:4     println!("The value of x is: {}", x);
                                                    ^
note: in expansion of format_args!
<std macros>:2:23: 2:77 note: expansion site
<std macros>:1:1: 3:2 note: in expansion of println!
src/main.rs:4:5: 4:42 note: expansion site
error: aborting due to previous error
Could not compile `hello_world`.
```

Rustは私たちに初期化されていない値を使わせてくれません。
次に、私たちが`println!`に追加したものについて話しましょう。

もしあなたが出力する文字列に2つの波括弧（`{}`です。マスタッシュと呼ぶ人もいます……）を入れたならば、Rustはこれをある種類の値の挿入を要求していると解釈します。
*文字列挿入* はコンピューターサイエンス用語で「文字列の中に突っ込む」という意味です。
私たちはコンマを追加し、それから私たちが`x`を挿入する値にしたいということを示すために`x`を追加します。
コンマはあなたが1つより多い引数を関数やマクロに渡すときに、それらを区切るために使われます。

あなたが単に波括弧だけを使った場合、Rustはその値の型をチェックし、意味のある方法でそれを表示しようとします。
もしあなたがもっと詳しい方法で書式を指定したいのであれば、[たくさんのオプションがあります][format]。
とりあえず、私たちはデフォルトのままにしておきます。整数は出力するのがそんなにややこしくはないからです。

[format]: ../std/fmt/index.html

# スコープとシャドーイング

束縛に戻りましょう。
変数束縛にはスコープがあります。それらは定義されたときのブロックの中でしか生存できません。
ブロックは`{`と`}`で囲まれた文の集まりです。
関数定義もブロックです！　
次の例では、私たちは別のブロックで生存する`x`と`y`の2つの変数束縛を定義します。
`x`は`fn main() {}`ブロックの内側からアクセスできる一方、`y`はもう1つ内側のブロックの内側からしかアクセスできません。

```rust,ignore
fn main() {
    let x: i32 = 17;
    {
        let y: i32 = 3;
        println!("The value of x is {} and value of y is {}", x, y);
    }
    println!("The value of x is {} and value of y is {}", x, y); // This won't work
}
```

1つ目の`println!`は「The value of x is 17 and the value of y is 3」と出力するのですが、この例は正常にコンパイルできません。なぜなら、2つ目の`println!`がもうスコープの中にいない`y`の値にアクセスできないからです。
実際にはこのエラーが出ます。

```bash
$ cargo build
   Compiling hello v0.1.0 (file:///home/you/projects/hello_world)
main.rs:7:62: 7:63 error: unresolved name `y`. Did you mean `x`? [E0425]
main.rs:7     println!("The value of x is {} and value of y is {}", x, y); // This won't work
                                                                       ^
note: in expansion of format_args!
<std macros>:2:25: 2:56 note: expansion site
<std macros>:1:1: 2:62 note: in expansion of print!
<std macros>:3:1: 3:54 note: expansion site
<std macros>:1:1: 3:58 note: in expansion of println!
main.rs:7:5: 7:65 note: expansion site
main.rs:7:62: 7:63 help: run `rustc --explain E0425` to see a detailed explanation
error: aborting due to previous error
Could not compile `hello`.

To learn more, run the command again with --verbose.
```

加えて、変数束縛はシャドーすることができます。
これは、そのときスコープの中にある他の束縛と同じ名前の後に作られた変数束縛を意味し、前の束縛を上書きします。

```rust
let x: i32 = 8;
{
    println!("{}", x); // Prints "8"
    let x = 12;
    println!("{}", x); // Prints "12"
}
println!("{}", x); // Prints "8"
let x =  42;
println!("{}", x); // Prints "42"
```

シャドーイングとミュータブルな束縛とは同じコインの裏表のように見えるかもしれません。しかし、それらは常に置き換えて使うことができるわけではない別個の2つの概念です。
一例として、シャドーイングによって私たちは名前を別の型の値に再束縛することができます。
それは束縛のミュータビリティーを変更することも可能です。

```rust
let mut x: i32 = 1;
x = 7;
let x = x; // x is now immutable and is bound to 7

let y = 4;
let y = "I can also be bound to text!"; // y is now of a different type
```
