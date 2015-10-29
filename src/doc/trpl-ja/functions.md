% 関数

Rustのプログラムは全て、少なくとも1つの関数、`main`関数を持っています。

```rust
fn main() {
}
```

これは評価可能な関数定義の最も単純なものです。
前に私たちが言ったように、`fn`は「これは関数です」ということを示します。この関数には引数がないので、名前と丸括弧が続きます。そして、その本文を表す波括弧が続きます。
これが`foo`という名前の関数です。

```rust
fn foo() {
}
```

それでは、引数を取る場合はどうでしょうか。
これが数値を出力する関数です。

```rust
fn print_number(x: i32) {
    println!("x is: {}", x);
}
```

これが`print_number`を使う完全なプログラムです。

```rust
fn main() {
    print_number(5);
}

fn print_number(x: i32) {
    println!("x is: {}", x);
}
```

見てのとおり、関数の引数は`let`宣言と非常によく似た動きをします。
あなたは引数の名前にコロンに続けて型を追加します。

これが2つの数値を足してそれらを出力する完全なプログラムです。

```rust
fn main() {
    print_sum(5, 6);
}

fn print_sum(x: i32, y: i32) {
    println!("sum is: {}", x + y);
}
```

あなたは関数を呼び出すときも、それを宣言したときと同様に、引数をコンマで区切ります。

`let`と異なり、あなたは関数の引数の型を宣言 _しなければなりません_ 。
これは動きません。

```rust,ignore
fn print_sum(x, y) {
    println!("sum is: {}", x + y);
}
```

このエラーが出ます。

```text
expected one of `!`, `:`, or `@`, found `)`
fn print_number(x, y) {
```

これはよく考えられた設計上の決断です。
プログラムによる完全な推論は可能ですが、Haskellのようにそれを行っている言語では、しばしばあなたの型を明示的にドキュメント化することがベストプラクティスであるとして提案されます。
私たちは関数に型の宣言を強制する一方で、関数の本文での推論を認めることが完全な推論と推論なしとの間のすばらしいスイートスポットであるということで意見が一致したのです。

戻り値についてはどうでしょうか。
これが整数に1を加える関数です。

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}
```

Rustの関数は明示的に1つの値を返します。そして、あなたはダッシュ（`-`）の後ろに大なりの記号（`>`）を続けた「矢印」の後にその型を宣言します。
関数の最後の行が何を戻すのかを決定します。
あなたはここにセミコロンがないことに気が付くでしょう。
もし私たちがそれを追加すると、こうなります。

```rust,ignore
fn add_one(x: i32) -> i32 {
    x + 1;
}
```

エラーが出ます。

```text
error: not all control paths return a value
fn add_one(x: i32) -> i32 {
     x + 1;
}

help: consider removing this semicolon:
     x + 1;
          ^
```

これはRustについて2つの興味深いことを明らかにします。それが式ベースの言語であること、そしてセミコロンが他の「波括弧とセミコロン」ベースの言語でのセミコロンとは違っているということです。
これら2つのことは関連します。

## 式対文

Rustは主に式ベースの言語です。
文には2種類しかなく、その他の全ては式です。

ではその違いは何でしょうか。
式は値を戻しますが、文は戻しません。
それがここで私たちが「not all control paths return a value」で終わった理由です。文`x + 1;`は値を戻さないからです。
Rustには2種類の文があります。「宣言文」と「式文」です。
その他の全ては式です。
まずは宣言文について話しましょう。

いくつかの言語では、変数束縛を文としてだけではなく、式として書くことができます。
Rubyではこうなります。

```ruby
x = y = 5
```

しかし、Rustでは束縛を導入するための`let`の使用は式ではありません。
次の例はコンパイルエラーを起こします。

```ignore
let x = (let y = 5); // expected identifier, found keyword `let`
```

ここでコンパイラーは次のことを私たちに教えています。式の先頭を検出することが期待されていたところ、`let`は式ではなく文の先頭にしかなれないということです。

次のことに注意しましょう。既に束縛されている変数への割当ては、その値が特に役に立つものではなかったとしてもやはり式です。割当てが割り当てられる値（例えば、前の例では`5`）を評価する他の言語とは異なり、Rustでは割当ての値は空のタプル`()`です。なぜなら、割り当てられる値には[単一の所有者](ownership.html)しかおらず、他のどんな値を戻したとしても予想外の出来事になってしまうからです。

```rust
let mut y = 5;

let x = (y = 6);  // x has the value `()`, not `6`
```

Rustでの2種類目の文は *式文* です。
これの目的は式を文に変換することです。
実際にはRustの文法は文の後には他の文が続くことが期待されています。
これはあなたがそれぞれの式を区切るためにセミコロンを使うということを意味します。
これはRustが全ての行末にセミコロンを使うことをあなたに要求する他の言語のほとんどとよく似ていること、そしてあなたの見るRustのコードのほとんど全ての行末で、あなたはセミコロンを見るということを意味します。

私たちが「ほとんど」と言ったこの例外は何でしょうか。
あなたがこの例で既に見ています。

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}
```

私たちの関数は`i32`を戻そうとしていますが、セミコロンを付ければ、それは代わりに`()`を戻します。
Rustはこの挙動がおそらく私たちの求めているものではないということを理解するので、私たちが前に見たエラーの中で、セミコロンを削除することを提案するのです。

## 早期のリターン

しかし、早期のリターンについてはどうでしょうか。
Rustはそのためのキーワード`return`を持っています。

```rust
fn foo(x: i32) -> i32 {
    return x;

    // we never run this code!
    x + 1
}
```

`return`を関数の最後の行で使っても動きますが、それはよろしくないスタイルだと考えられています。

```rust
fn foo(x: i32) -> i32 {
    return x + 1;
}
```

あなたがこれまで式ベースの言語を使ったことがなければ、`return`のない前の定義の方がちょっと変に見えるかもしれません。しかし、それは時間とともに直観的に感じられるようになります。

## ダイバージング関数

Rustはリターンしない関数、「ダイバージング関数」のための特別な構文をいくつか持っています。

```rust
fn diverges() -> ! {
    panic!("This function never returns!");
}
```

`panic!`は私たちが既に見ている`println!`と同様にマクロです。
`println!`とは違って、`panic!`は実行中の現在のスレッドを与えられたメッセージとともにクラッシュさせます。
この関数はクラッシュを引き起こすので、決してリターンしません。そのため、それは「ダイバージ」と読む、「`!`」型を持つのです。

もしあなたが`diverges()`を呼び出すメイン関数を追加してそれを実行するならば、次のようなものが出力されるでしょう。

```text
thread ‘<main>’ panicked at ‘This function never returns!’, hello.rs:2
```

もしあなたがもっと情報を得たいと思うのであれば、`RUST_BACKTRACE`環境変数をセットすることでバックトレースを得ることができます。

```text
$ RUST_BACKTRACE=1 ./diverges
thread '<main>' panicked at 'This function never returns!', hello.rs:2
stack backtrace:
   1:     0x7f402773a829 - sys::backtrace::write::h0942de78b6c02817K8r
   2:     0x7f402773d7fc - panicking::on_panic::h3f23f9d0b5f4c91bu9w
   3:     0x7f402773960e - rt::unwind::begin_unwind_inner::h2844b8c5e81e79558Bw
   4:     0x7f4027738893 - rt::unwind::begin_unwind::h4375279447423903650
   5:     0x7f4027738809 - diverges::h2266b4c4b850236beaa
   6:     0x7f40277389e5 - main::h19bb1149c2f00ecfBaa
   7:     0x7f402773f514 - rt::unwind::try::try_fn::h13186883479104382231
   8:     0x7f402773d1d8 - __rust_try
   9:     0x7f402773f201 - rt::lang_start::ha172a3ce74bb453aK5w
  10:     0x7f4027738a19 - main
  11:     0x7f402694ab44 - __libc_start_main
  12:     0x7f40277386c8 - <unknown>
  13:                0x0 - <unknown>
```

`RUST_BACKTRACE`はCargoの`run`コマンドでも使うことができます。

```text
$ RUST_BACKTRACE=1 cargo run
     Running `target/debug/diverges`
thread '<main>' panicked at 'This function never returns!', hello.rs:2
stack backtrace:
   1:     0x7f402773a829 - sys::backtrace::write::h0942de78b6c02817K8r
   2:     0x7f402773d7fc - panicking::on_panic::h3f23f9d0b5f4c91bu9w
   3:     0x7f402773960e - rt::unwind::begin_unwind_inner::h2844b8c5e81e79558Bw
   4:     0x7f4027738893 - rt::unwind::begin_unwind::h4375279447423903650
   5:     0x7f4027738809 - diverges::h2266b4c4b850236beaa
   6:     0x7f40277389e5 - main::h19bb1149c2f00ecfBaa
   7:     0x7f402773f514 - rt::unwind::try::try_fn::h13186883479104382231
   8:     0x7f402773d1d8 - __rust_try
   9:     0x7f402773f201 - rt::lang_start::ha172a3ce74bb453aK5w
  10:     0x7f4027738a19 - main
  11:     0x7f402694ab44 - __libc_start_main
  12:     0x7f40277386c8 - <unknown>
  13:                0x0 - <unknown>
```

ダイバージング関数は任意の型として使うことができます。

```should_panic
# fn diverges() -> ! {
#    panic!("This function never returns!");
# }
let x: i32 = diverges();
let x: String = diverges();
```

## 関数ポインター

私たちは関数を指示する変数束縛を作ることもできます。

```rust
let f: fn(i32) -> i32;
```

`f`は`i32`を引数として受け取り、`i32`を戻す関数を指示する変数束縛です。
例えばこうです。

```rust
fn plus_one(i: i32) -> i32 {
    i + 1
}

// without type inference
let f: fn(i32) -> i32 = plus_one;

// with type inference
let f = plus_one;
```

それから、私たちはその関数を呼び出すために`f`を使うことができます。

```rust
# fn plus_one(i: i32) -> i32 { i + 1 }
# let f = plus_one;
let six = f(5);
```
