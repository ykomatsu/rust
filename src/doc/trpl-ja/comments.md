% コメント

私たちはいくつかの関数を持つので、コメントについて学ぶことはよい考えです。
コメントはあなたのコードについての何かを説明する助けになるように、あなたが他のプログラマーに残すメモです。
コンパイラーはそれらをほとんど無視します。

Rustはあなたが気にすべき2種類のコメント、 *行コメント* と *ドキュメンテーションコメント* を持ちます。

```rust
// Line comments are anything after ‘//’ and extend to the end of the line.

let x = 5; // this is also a line comment.

// If you have a long explanation for something, you can put line comments next
// to each other. Put a space between the // and your comment so that it’s
// more readable.
```

他の種類のコメントがドキュメンテーションコメントです。
ドキュメンテーションコメントは`//`の代わりに`///`を使い、その中でMarkdown記法をサポートします。

```rust
/// Adds one to the number given.
///
/// # Examples
///
/// ```
/// let five = 5;
///
/// assert_eq!(6, add_one(5));
/// # fn add_one(x: i32) -> i32 {
/// #     x + 1
/// # }
/// ```
fn add_one(x: i32) -> i32 {
    x + 1
}
```

もう1つのスタイルのドキュメンテーションコメント、`//!`があります。これは、その後に続く要素ではなく、含んでいる要素（例えばクレート、モジュール、関数）にコメントを付けます。
一般的にはクレートルート（lib.rs）やモジュールルート（mod.rs）の中で使われます。

```
//! # The Rust Standard Library
//!
//! The Rust Standard Library provides the essential runtime
//! functionality for building portable Rust software.
```

ドキュメンテーションコメントを書いているとき、いくつかの使い方の例を提供することは非常に非常に有用です。
あなたは私たちがここで新しいマクロ、`assert_eq!`を使っていることに気付くでしょう。
これは2つの値を比較し、もしそれらが互いに等しくなければ`panic!`します。
これはドキュメントの中で非常に便利です。
もう1つのマクロ、`assert!`は、それに渡された値が`false`であれば`panic!`します。

あなたはそれらのドキュメンテーションコメントからHTMLドキュメントを生成するため、そしてコード例をテストとして実行するためにも[`rustdoc`](documentation.html)ツールを使うことができます！
