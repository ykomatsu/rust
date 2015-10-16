% BorrowとAsRef

[`Borrow`][borrow]トレイトと[`AsRef`][asref]トレイトは非常に似ていますが、別物です。
これはそれら2つのトレイトが意味するものについて簡単に記憶を整理するものです。

[borrow]: ../std/borrow/trait.Borrow.html
[asref]: ../std/convert/trait.AsRef.html

# Borrow

`Borrow`トレイトは、あなたがデータ構造を書いていて、所有される型とボローされる型を何らかの目的で同意語として使いたいときに使われます。

例えば、[`HashMap`][hashmap]は`Borrow`を使う[`get`メソッド][get]を持っています。

```rust,ignore
fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where K: Borrow<Q>,
          Q: Hash + Eq
```

[hashmap]: ../std/collections/struct.HashMap.html
[get]: ../std/collections/struct.HashMap.html#method.get

このシグニチャーは結構複雑です。
ここで私たちの関心があるのは`K`パラメーターです。
それは`HashMap`そのもののパラメーターとして言及されています。

```rust,ignore
struct HashMap<K, V, S = RandomState> {
```

`K`パラメーターは`HashMap`の使う _キー_ の型です。
そこで、`get()`のシグニチャーをもう一度見ると分かりますが、私たちはキーが`Borrow<Q>`を実装しているときに`get()`を使えるのです。
そのことから、私たちは`String`をキーとして使う`HashMap`を作ることができ、検索のときには`&str`を使うこともできます。

```rust
use std::collections::HashMap;

let mut map = HashMap::new();
map.insert("Foo".to_string(), 42);

assert_eq!(map.get("Foo"), Some(&42));
```

これは、標準ライブラリーに`impl Borrow<str> for String`があるからです。

あなたが所有される型、又はボローされる型を受け取りたいとき、多くの型にとっては`&T`で十分です。
しかし、`Borrow`が効果的な領域の1つは、1つより多い種類のボローされる値があるときです。
これは参照やスライスのときに特にそうです。あなたは`&T`と`&mut T`のどちらか、又は両方を持つことができます。
もし私たちがこれらの型の両方を受け取りたいのであれば、`Borrow`はちょうどよいでしょう。

```rust
use std::borrow::Borrow;
use std::fmt::Display;

fn foo<T: Borrow<i32> + Display>(a: T) {
    println!("a is borrowed: {}", a);
}

let mut i = 5;

foo(&i);
foo(&mut i);
```

これは`a is borrowed: 5`と2回出力します。

# AsRef

`AsRef`トレイトは変換トレイトです。
それはジェネリックなコードで、ある値を参照に変換するために使われます。
こういう感じです。

```rust
let s = "Hello".to_string();

fn foo<T: AsRef<str>>(s: T) {
    let slice = s.as_ref();
}
```

# どちらを使うべきか

私たちはそれらがどう似ているのかを知りました。それらは両方とも、ある型の所有されるバージョンとボローされるバージョンを扱います。
しかし、それらには少し異なる点もあります。

あなたが異なる種類のボローイングを抽象化したいとき、又はあなたがハッシュ化や比較のような、所有される値とボローされる値とを同じ方法で扱うデータ構造を構築しているときには`Borrow`を選びましょう。

あなたが直接に何かを参照に変換したくて、ジェネリックなコードを書いているときには`AsRef`を選びましょう。
