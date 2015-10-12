% テスト

> プログラムのテストは、バグの存在を示すためには非常に効率的な方法ですが、バグの不存在を示すためには絶望的に不十分です。
>
> Edsger W. Dijkstra, "The Humble Programmer" (1972)

Rustのコードをテストする方法について話しましょう。
私たちはRustのコードをテストする正しい方法について議論するつもりはありません。
テストを書くための正しい方法、誤った方法に関する流派はたくさんあります。
それらの方法は全て、同じ基本的なツールを使うので、私たちはそれらを使うための文法をお見せしましょう。

# `test`属性

Rustでの一番簡単なテストは、`test`属性の付いた関数です。
`adder`という名前の新しいプロジェクトをCargoで作りましょう。

```bash
$ cargo new adder
$ cd adder
```

あなたが新しいプロジェクトを作ると、Cargoは自動的に簡単なテストを生成します。
これが`src/lib.rs`の内容です。

```rust
#[test]
fn it_works() {
}
```

`#[test]`に注意しましょう。
この属性は、この関数がテスト関数であるということを示します。
今のところ、その関数には本文がありません。
成功させるためにはそれで十分です！
私たちは`cargo test`でテストを実行することができます。

```bash
$ cargo test
   Compiling adder v0.0.1 (file:///home/you/projects/adder)
     Running target/adder-91b3e234d4ed382a

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Cargoは私たちのテストをコンパイルし、実行しました。
ここでは2種類の結果が出力されています。1つは私たちが書いたテストについてのもの、もう1つはドキュメンテーションテストについてのものです。
それらについては後で話しましょう。
とりあえず、この行を見ましょう。

```text
test it_works ... ok
```
`it_works`に注意しましょう。
これは私たちの関数の名前から来ています。

```rust
fn it_works() {
# }
```

サマリーも出力されています。

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

なぜ私たちの何も書いていないテストがこのように成功するのでしょうか。
`panic!`しないテストは全て成功で、`panic!`するテストは全て失敗なのです。
私たちのテストを失敗させましょう。

```rust
#[test]
fn it_works() {
    assert!(false);
}
```

`assert!`はRustが提供するマクロで、1つの引数を取ります。引数が`true`であれば何も起きません。
引数が`false`であれば`panic!`します。
私たちのテストをもう一度実行しましょう。

```bash
$ cargo test
   Compiling adder v0.0.1 (file:///home/you/projects/adder)
     Running target/adder-91b3e234d4ed382a

running 1 test
test it_works ... FAILED

failures:

---- it_works stdout ----
        thread 'it_works' panicked at 'assertion failed: false', /home/steve/tmp/adder/src/lib.rs:3



failures:
    it_works

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured

thread '<main>' panicked at 'Some tests failed', /home/steve/src/rust/src/libtest/lib.rs:247
```

Rustは私たちのテストが失敗したことを示しています。

```text
test it_works ... FAILED
```

そして、それはサマリーにも反映されます。

```text
test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured
```

ステータスコードも非0になっています。
私たちはOS XやLinuxでは`$?`を使うことができます。

```bash
$ echo $?
101
```

Windowsでは、あなたが`cmd`を使っていればこうです。

```bash
> echo %ERRORLEVEL%
```

そして、あなたがPowerShellを使っていればこうです。

```bash
> echo $LASTEXITCODE # the code itself
> echo $? # a boolean, fail or succeed
```

これはあなたが`cargo test`を他のツールと統合したいときに便利です。

私たちはもう1つの属性、`should_panic`を使ってテストの失敗を反転させることができます。

```rust
#[test]
#[should_panic]
fn it_works() {
    assert!(false);
}
```

今度は、このテストが`panic!`すれば成功で、完走すれば失敗です。
試しましょう。

```bash
$ cargo test
   Compiling adder v0.0.1 (file:///home/you/projects/adder)
     Running target/adder-91b3e234d4ed382a

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Rustはもう1つのマクロ、`assert_eq!`を提供しています。これは2つの引数の等価性を調べます。

```rust
#[test]
#[should_panic]
fn it_works() {
    assert_eq!("Hello", "world");
}
```

このテストは成功でしょうか、失敗でしょうか。
`should_panic`属性があるので、これは成功です。

```bash
$ cargo test
   Compiling adder v0.0.1 (file:///home/you/projects/adder)
     Running target/adder-91b3e234d4ed382a

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

`should_panic`テストは壊れやすいものです。なぜなら、テストが予想外の理由で失敗したのではないということを保証することが難しいからです。
これを何とかするために、`should_panic`属性にはオプションで`expected`パラメーターを付けることができます。
テストハーネスが、失敗したときのメッセージに与えられたテキストが含まれていることを確かめてくれるでしょう。
前述の例のもっと安全なバージョンはこうなるでしょう。

```rust
#[test]
#[should_panic(expected = "assertion failed")]
fn it_works() {
    assert_eq!("Hello", "world");
}
```

基本はそれだけです！
「リアルな」テストを書いてみましょう。

```rust,ignore
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[test]
fn it_works() {
    assert_eq!(4, add_two(2));
}
```

これは非常に一般的な`assert_eq!`の使い方です。いくつかの関数に結果の分かっている引数を渡して呼び出し、期待した結果と比較します。

# `ignore`属性

ときどき、特定のテストの実行に非常に時間が掛かることがあります。
そのようなテストは、`ignore`属性を使ってデフォルトでは無効にすることができます。

```rust
#[test]
fn it_works() {
    assert_eq!(4, add_two(2));
}

#[test]
#[ignore]
fn expensive_test() {
    // code that takes an hour to run
}
```

私たちはテストを実行し、`it_works`が実行されることを確認しますが、今度は`expensive_test`は実行されません。

```bash
$ cargo test
   Compiling adder v0.0.1 (file:///home/you/projects/adder)
     Running target/adder-91b3e234d4ed382a

running 2 tests
test expensive_test ... ignored
test it_works ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

その高コストなテストは`cargo test -- --ignored`を使って明示的に実行することができます。

```bash
$ cargo test -- --ignored
     Running target/adder-91b3e234d4ed382a

running 1 test
test expensive_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

`--ignored`属性はテストバイナリーの引数であって、Cargoのものではありません。
コマンドが`cargo test -- --ignored`となっているのはそういうことです。

# `tests`モジュール

今までの私たちの例での方法は、慣用的ではありません。`tests`モジュールがないからです。
私たちの例の慣用的な書き方はこのようになります。

```rust,ignore
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::add_two;

    #[test]
    fn it_works() {
        assert_eq!(4, add_two(2));
    }
}
```

ここでは、いくつかの変更点があります。
まず、`cfg`属性の付いた`mod tests`を導入しました。
このモジュールを使うと、私たちは全てのテストをグループ化することができます。また、必要であれば、ヘルパー関数を定義し、それをクレートの一部に含まれないようにすることもできます。
この`cfg`属性によって、私たちがテストを実行しようとしているときにだけテストコードがコンパイルされるようになります。
これは、コンパイル時間を節約し、私たちのテストが通常のビルドからは完全に無関係になることを保証してくれます。

2つ目の変更点は、`use`宣言です。
私たちは内部モジュールの中にいるので、テスト関数をスコープの中に持ち込む必要があります。
あなたのモジュールが大きい場合、これは面倒になり得るので、ここがグロブの一般的な使い所です。
私たちの`src/lib.rs`をグロブを使うように変更しましょう。

```rust,ignore
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(4, add_two(2));
    }
}
```

`use`行が変わったことに注意しましょう。
さて、私たちのテストを実行します。

```bash
$ cargo test
    Updating registry `https://github.com/rust-lang/crates.io-index`
   Compiling adder v0.0.1 (file:///home/you/projects/adder)
     Running target/adder-91b3e234d4ed382a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

動きます！

現在の慣習では、`tests`モジュールは「ユニット」テストを入れるために使うことになっています。
単一の小さな機能の単位をテストするだけのものは全て、ここに入れる意味があります。
しかし、「結合」テストはどうでしょうか。
そのために、私たちには`tests`ディレクトリーがあります。

# `tests`ディレクトリー

結合テストを書くために、`tests`ディレクトリーを作りましょう。そして、その中に次の内容の`tests/lib.rs`ファイルを置きます。

```rust,ignore
extern crate adder;

#[test]
fn it_works() {
    assert_eq!(4, adder::add_two(2));
}
```

これは私たちの前のテストと似ていますが、少し違います。
今回、私たちは`extern crate adder`を先頭に書いています。
これは、`tests`ディレクトリーの中のテストが全く別のクレートであるため、私たちはライブラリーをインポートしなければならないからです。
これは、なぜ`tests`が結合テストを書くのに適切な場所なのかという理由でもあります。そこにあるテストは、そのライブラリーを他のプログラムと同じようなやり方で使うからです。

テストを実行しましょう。

```bash
$ cargo test
   Compiling adder v0.0.1 (file:///home/you/projects/adder)
     Running target/adder-91b3e234d4ed382a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

     Running target/lib-c18e7d3494509e74

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

今度は3つのセクションが出力されました。私たちの新しいテストが実行され、前に書いたテストも同様に実行されます。

`tests`ディレクトリーについてはそれだけです。
`tests`モジュールはここでは必要ありません。全てのものがテストのためのものだからです。

最後に、3つ目のセクションを確認しましょう。ドキュメンテーションテストです。

# ドキュメンテーションテスト

例の付いたドキュメントほどよいものはありません。
実際に動かない例ほど悪いものはありません。ドキュメントを書いた後にコードが変更されているからです。
この状況を終わらせるために、Rustはあなたのドキュメント内の例の自動実行をサポートします（ **注意** 。これはライブラリークレートの中でのみ動作し、バイナリークレートの中では動作しません）。
これが例を付けた具体的な`src/lib.rs`です。

```rust,ignore
//! The `adder` crate provides functions that add numbers to other numbers.
//!
//! # Examples
//!
//! ```
//! assert_eq!(4, adder::add_two(2));
//! ```

/// This function adds two to its argument.
///
/// # Examples
///
/// ```
/// use adder::add_two;
///
/// assert_eq!(4, add_two(2));
/// ```
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(4, add_two(2));
    }
}
```

モジュールレベルのドキュメントには`//!`を付け、関数レベルのドキュメントには`///`を付けていることに注意しましょう。
RustのドキュメントはMarkdown形式のコメントをサポートしていて、3連バッククオートはコードブロックを表します。
`# Examples`セクションを含めるのが慣習で、そのとおり、例が後に続きます。

テストをもう一度実行しましょう。

```bash
$ cargo test
   Compiling adder v0.0.1 (file:///home/steve/tmp/adder)
     Running target/adder-91b3e234d4ed382a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

     Running target/lib-c18e7d3494509e74

running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 2 tests
test add_two_0 ... ok
test _0 ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

今回、私たちは、全ての種類のテストを実行しています！
ドキュメンテーションテストの名前に注意しましょう。`_0`はモジュールテストのために生成された名前で、`add_two_0`は関数テストのために生成された名前です。
あなたが例を追加するにつれて、それらの名前は`add_two_1`というような形で数値が増えていきます。

私たちはまだドキュメンテーションテストの書き方の詳細について、全てをカバーしてはいません。
もっと知りたい場合は、[ドキュメントの章](documentation.html)を見てください。
