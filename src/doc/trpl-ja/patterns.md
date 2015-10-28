% パターン

パターンはRustでは非常に一般的です。
私たちはそれらを[変数束縛][bindings]、[マッチ文][match]、そして他の場所でも使います。
パターンにできること全てのための弾丸ツアーに出発しましょう！

[bindings]: variable-bindings.html
[match]: match.html

簡単な復習です。あなたはリテラルに対して直接にマッチすることができ、`_`は「任意の」場合のように振る舞います。

```rust
let x = 1;

match x {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

これは`one`をプリントします。

そこにはパターンの1つの落とし穴があります。新しい束縛を導入する全てのものと同様に、それらはシャドーイングを導入するのです。
例えばこうです。

```rust
let x = 'x';
let c = 'c';

match c {
    x => println!("x: {} c: {}", x, c),
}

println!("x: {}", x)
```

これは次のものをプリントします。

```text
x: c c: c
x: x
```

言い換えると、`x =>`はパターンにマッチし、`x`という名前の新しい束縛を導入するということです。そしてその束縛はマッチ肢のスコープ内にあります。
私たちは既に`x`という名前の束縛を持つので、この新しい`x`はそれを隠蔽します。

# 複数のパターン

あなたは`|`を使って複数のパターンにマッチすることができます。

```rust
let x = 1;

match x {
    1 | 2 => println!("one or two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

これは`one or two`をプリントします。

# 分配束縛

もしあなたが[`struct`][struct]のような合成データ型を持つのならば、あなたはそれをパターン内で分配束縛することができます。

```rust
struct Point {
    x: i32,
    y: i32,
}

let origin = Point { x: 0, y: 0 };

match origin {
    Point { x, y } => println!("({},{})", x, y),
}
```

[struct]: structs.html

私たちは値に別の名前を与えるために`:`を使うことができます。

```rust
struct Point {
    x: i32,
    y: i32,
}

let origin = Point { x: 0, y: 0 };

match origin {
    Point { x: x1, y: y1 } => println!("({},{})", x1, y1),
}
```

もし私たちが値の中のいくつかにしか興味がないのであれば、私たちはそれら全てに名前を付ける必要はありません。

```rust
struct Point {
    x: i32,
    y: i32,
}

let origin = Point { x: 0, y: 0 };

match origin {
    Point { x, .. } => println!("x is {}", x),
}
```

これは`x is 0`をプリントします。

あなたはこの種のマッチを最初のメンバーだけでなく任意のメンバーに対して行うことができます。

```rust
struct Point {
    x: i32,
    y: i32,
}

let origin = Point { x: 0, y: 0 };

match origin {
    Point { y, .. } => println!("y is {}", y),
}
```

これは`y is 0`をプリントします。

この「分配束縛」の挙動は[タプル][tuples]や[列挙型][enum]のような全ての合成データ型に対して有効です。

[tuples]: primitive-types.html#tuples
[enums]: enums.html

# 束縛の無視

あなたは型と値を無視するために`_`をパターン内で使うことができます。
例えばこれは`Result<T, E>`に`match`します。

```rust
# let some_value: Result<i32, &'static str> = Err("There was an error");
match some_value {
    Ok(value) => println!("got a value: {}", value),
    Err(_) => println!("an error occurred"),
}
```

最初の肢では私たちは`Ok`バリアント内の値を`value`に束縛します。
しかし、`Err`肢では私たちは具体的なエラーを無視し、単純に一般的なエラーメッセージをプリントするために`_`を使います。

`_`は束縛を作る任意のパターンで有効です。
これは大きな構造の一部を無視するために便利なことがあります。

```rust
fn coordinate() -> (i32, i32, i32) {
    // generate and return some sort of triple tuple
# (1, 2, 3)
}

let (x, _, z) = coordinate();
```

ここでは私たちはタプルの最初の要素と最後の要素を`x`と`z`に束縛しますが、間の要素は無視します。

同様に、あなたは複数の値を無視するために`..`をパターン内でを使うことができます。

```rust
enum OptionalTuple {
    Value(i32, i32, i32),
    Missing,
}

let x = OptionalTuple::Value(5, -2, 3);

match x {
    OptionalTuple::Value(..) => println!("Got a tuple!"),
    OptionalTuple::Missing => println!("No such luck."),
}
```

これは`Got a tuple!`をプリントします。

# 参照とミュータブルな参照

もしあなたが[参照][ref]を得たいならば、`ref`キーワードを使いましょう。

```rust
let x = 5;

match x {
    ref r => println!("Got a reference to {}", r),
}
```

これは`Got a reference to 5`をプリントします。

[ref]: references-and-borrowing.html

ここでは`match`内の`r`は`&i32`を持ちます。
言い換えると、`ref`キーワードはパターンの中で使うための参照を _作る_ ということです。
もしあなたがミュータブルな参照を必要ならば、`ref mut`が同じ方法で動作するでしょう。

```rust
let mut x = 5;

match x {
    ref mut mr => println!("Got a mutable reference to {}", mr),
}
```

# レンジ

あなたは値のレンジを`...`を使ってマッチすることができます。

```rust
let x = 1;

match x {
    1 ... 5 => println!("one through five"),
    _ => println!("anything"),
}
```

これは`one through five`をプリントします。

レンジは大体整数と`char`で使われます。

```rust
let x = '💅';

match x {
    'a' ... 'j' => println!("early letter"),
    'k' ... 'z' => println!("late letter"),
    _ => println!("something else"),
}
```

これは`something else`をプリントします。

# 束縛

あなたは値を`@`という名前に束縛することができます。

```rust
let x = 1;

match x {
    e @ 1 ... 5 => println!("got a range element {}", e),
    _ => println!("anything"),
}
```

これは`got a range element 1`をプリントします。
これはあなたがデータ構造の一部に対して複雑なマッチを行いたいときに便利です。

```rust
#[derive(Debug)]
struct Person {
    name: Option<String>,
}

let name = "Steve".to_string();
let mut x: Option<Person> = Some(Person { name: Some(name) });
match x {
    Some(Person { name: ref a @ Some(_), .. }) => println!("{:?}", a),
    _ => {}
}
```

これは`Some("Steve")`をプリントします。私たちは内側の`name`を`a`に束縛しました。

もしあなたが`@`を`|`とともに使うならば、あなたは名前がパターンの各部分で束縛されていることを確かめる必要があります。

```rust
let x = 5;

match x {
    e @ 1 ... 5 | e @ 8 ... 10 => println!("got a range element {}", e),
    _ => println!("anything"),
}
```

# ガード

あなたは`if`を使って「マッチガード」を導入することができます。

```rust
enum OptionalInt {
    Value(i32),
    Missing,
}

let x = OptionalInt::Value(5);

match x {
    OptionalInt::Value(i) if i > 5 => println!("Got an int bigger than five!"),
    OptionalInt::Value(..) => println!("Got an int!"),
    OptionalInt::Missing => println!("No such luck."),
}
```

これは`Got an int!`をプリントします。

もしあなたが`if`を複数のパターンで使っているならば、`if`は両方の部分に適用されます。

```rust
let x = 4;
let y = false;

match x {
    4 | 5 if y => println!("yes"),
    _ => println!("no"),
}
```

`if`は`5`にだけではなく`4 | 5`の全体に適用されるので、これは`no`をプリントします。
言い換えると、`if`の優先順位はこのように振る舞うということです。

```text
(4 | 5) if y => ...
```

こうではありません。

```text
4 | (5 if y) => ...
```

# マッチの混合

ふう！　何かにマッチするためのたくさんの異なる方法がありました。そしてあなたがしようとしていることに応じて、それらは全て混合してマッチさせることができます。

```rust,ignore
match x {
    Foo { x: Some(ref name), y: None } => ...
}
```

パターンは非常に強力です。
それらを上手に使いましょう。
