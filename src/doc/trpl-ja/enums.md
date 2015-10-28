% 列挙型

Rustでの`enum`はいくつかの採り得る値の中の1つにできるデータを表現する型です。

```rust
enum Message {
    Quit,
    ChangeColor(i32, i32, i32),
    Move { x: i32, y: i32 },
    Write(String),
}
```

各バリアントはオプションとしてそれに関連付けられる値を持つことができます。
バリアントを定義する構文は構造体を定義するために使われる構文に似ています。つまり、あなたはデータを持たない（ユニット的構造体のような）バリアント、名前付きデータを持つバリアント、無名データを持つ（タプル構造体のような）バリアントを持つことができます。
しかし、各部分に分かれている構造体の定義とは違って、`enum`は1つの型です。
列挙型の値はどのバリアントにもマッチすることができます。
この理由から、列挙型はときどき「直和型」と呼ばれます。列挙型の採り得る値のセットは各バリアントの採り得る値のセットの直和です。

私たちは`::`構文を各バリアントの名前を使うために使います。それらは`enum`自体の名前のスコープを持ちます。
これによって次のものの両方が動作することが可能になります。

```rust
# enum Message {
#     Move { x: i32, y: i32 },
# }
let x: Message = Message::Move { x: 3, y: 4 };

enum BoardGameTurn {
    Move { squares: i32 },
    Pass,
}

let y: BoardGameTurn = BoardGameTurn::Move { squares: 1 };
```

両方のバリアントは`Move`と名付けられますが、それらは列挙型の名前のスコープを持つので、それらは両方とも衝突することなく使うことができます。

列挙型の値はそれがどのバリアントなのか、加えてそのバリアントに関連付けられる任意のデータについての情報を含みます。
データがそれが何の型なのかを示す「タグ」を含むため、これはときどき「タグ付き共用体」と呼ばれます。
コンパイラーはこの情報をあなたが列挙型の中のデータに安全にアクセスすることを強制するために使います。
例えば、あなたは値が採り得るバリアントの1つであるかのようにそれを単純に分配束縛しようとすることができません。

```rust,ignore
fn process_color_change(msg: Message) {
    let Message::ChangeColor(r, g, b) = msg; // compile-time error
}
```

それらの作業をサポートしないことはむしろ制限のように見えるかもしれません。しかし、それは乗り越えることができる制限です。
それには2つの方法があります。私たち自身で相等性を実装すること、又はあなたが次のセクションで学ぶであろうパターンマッチをバリアントに対して[`match`][match]式を使って行うことです。
私たちはRustについて相等性を実装するために必要なほど十分にはまだ知りません。しかし、私たちは[`traits`][traits]セクションで知るでしょう。

[match]: match.html
[if-let]: if-let.html
[traits]: traits.html

# 関数としてのコンストラクター

列挙型のコンストラクターは関数のようにも使うことができます。
例えばこのようにです。

```rust
# enum Message {
# Write(String),
# }
let m = Message::Write("Hello, world".to_string());
```

これは次のものと同じです。

```rust
# enum Message {
# Write(String),
# }
fn foo(x: String) -> Message {
    Message::Write(x)
}

let x = foo("Hello, world".to_string());
```

これは私たちにすぐに便利なものではありませんが、私たちが[`closures`][closures]を手に入れるとき、私たちは他の関数の引数として関数を渡す方法について話すでしょう。
例えば、[`iterators`][iterators]を使って、私たちは`String`のベクターを`Message::Write`のベクターに変換するためにこれをすることができます。

```rust
# enum Message {
# Write(String),
# }

let v = vec!["Hello".to_string(), "World".to_string()];

let v1: Vec<Message> = v.into_iter().map(Message::Write).collect();
```

[closures]: closures.html
[iterators]: iterators.html
