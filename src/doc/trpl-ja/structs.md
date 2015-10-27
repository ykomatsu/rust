% 構造体

`struct`は複雑なデータ型を作るための方法です。
例えば、もし私たちが2D空間内の座標に関わる計算を行っていたならば、私たちは`x`の値と`y`の値の両方を必要としたでしょう。

```rust
let origin_x = 0;
let origin_y = 0;
```

`struct`によって私たちはそれらの2つを1つの統合されたデータ型に結合することが可能になります。

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let origin = Point { x: 0, y: 0 }; // origin: Point

    println!("The origin is at ({}, {})", origin.x, origin.y);
}
```

ここでは多くのことが行われているので、それを分解しましょう。
私たちは`struct`を`struct`キーワードと名前で宣言します。
慣習によって、`struct`は大文字で始まりキャメルケースです。つまり、`Point_In_Space`ではなく`PointInSpace`です。

いつものとおり、私たちは私たちの`struct`のインスタンスを`let`を通じて作ることができます。しかし、私たちは`key: value`スタイルの構文を各フィールドをセットするために使います。
順番は元の宣言内と同じである必要はありません。

最後に、フィールドは名前を持つので、私たちはフィールドにドット記法を通じてアクセスすることができます。つまり、`origin.x`のようにです。

`struct`内の値はRustでの他の束縛と同じようにデフォルトでイミュータブルです。
それらをミュータブルにするためには`mut`を使いましょう。

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let mut point = Point { x: 0, y: 0 };

    point.x = 5;

    println!("The point is at ({}, {})", point.x, point.y);
}
```

これは`The point is at (5, 0)`をプリントするでしょう。

Rustはフィールドレベルのミュータビリティーを言語レベルではサポートしません。そのため、あなたはこのようなものを書くことができません。

```rust,ignore
struct Point {
    mut x: i32,
    y: i32,
}
```

ミュータビリティーは束縛の特性で、構造体自体の特性ではありません。
もしあなたがフィールドレベルのミュータビリティーを使っていたのであれば、これは最初は変に見えるかもしれません。しかし、それはかなり物事を単純にします。
それによってあなたは何かを短い時間だけミュータブルにすることさえ可能になります。

```rust,ignore
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let mut point = Point { x: 0, y: 0 };

    point.x = 5;

    let point = point; // this new binding can’t change now

    point.y = 6; // this causes an error
}
```

# 更新構文

`struct`はあなたがいくつかの値のために他の`struct`のコピーが欲しいことを示すために`..`を含むことができます。
例えばこうです。

```rust
struct Point3d {
    x: i32,
    y: i32,
    z: i32,
}

let mut point = Point3d { x: 0, y: 0, z: 0 };
point = Point3d { y: 1, .. point };
```

これは`point`に新しい`y`を与えますが、古い`x`の値と`z`の値を保持します。
それは同じ`struct`である必要はありません。あなたは新しい構造体を作るときにこの構文を使うことができ、それはあなたの明示しない値をコピーするでしょう。

```rust
# struct Point3d {
#     x: i32,
#     y: i32,
#     z: i32,
# }
let origin = Point3d { x: 0, y: 0, z: 0 };
let point = Point3d { z: 1, x: 2, .. origin };
```

# タプル構造体

Rustはもう1つのデータ型を持ちます。それは[タプル][tuple]と`struct`の間のハイブリッドのようなもので、「タプル構造体」と呼ばれます。
タプル構造体は名前を持ちますが、それらのフィールドは持ちません。

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);
```

[tuple]: primitive-types.html#tuples

それら2つは等しくはないでしょう。それらが同じ値を持っているとしても。

```rust
# struct Color(i32, i32, i32);
# struct Point(i32, i32, i32);
let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

`struct`を使うことはタプル構造体を使うことよりもほとんど常によいです。
私たちは`Color`と`Point`を代わりにこのように書くでしょう。

```rust
struct Color {
    red: i32,
    blue: i32,
    green: i32,
}

struct Point {
    x: i32,
    y: i32,
    z: i32,
}
```

今度は、私たちは位置ではなく実際の名前を持ちます。
よい名前は重要です。そして`struct`では私たちは実際の名前を持ちます。

しかし、タプル構造体が非常に便利な場合が1つ *だけ* あります。それは、1つだけ要素を持つタプル構造体です。
私たちはこれを「ニュータイプ」パターンと呼びます。なぜなら、それによってあなたは新しい型を作ることが可能になるからです。それはそれの含む値とは別個の型であり、それ独自のセマンティックな意味を表現します。

```rust
struct Inches(i32);

let length = Inches(10);

let Inches(integer_length) = length;
println!("length is {} inches", integer_length);
```

ここであなたが見たとおり、ちょうど通常のタプルのように、あなたは分配束縛`let`を通じて内側の整数型を展開することができます。
この場合、`let Inches(integer_length)`は`10`を`integer_length`に割り当てます。

# ユニット的構造体

あなたは全くメンバーを持たない`struct`を定義することができます。

```rust
struct Electron;

let x = Electron;
```

そのような`struct`は「ユニット的」と呼ばれます。なぜなら、ときどき「ユニット」と呼ばれる空タプル`()`にそれが似ているからです。
タプル構造体のように、それは新しい型を定義します。

これがそれ自体で便利なことはめったにありません（ときどきそれはマーカー型として役に立ちますが）。しかし、他の機能と組み合わせることによって、それは便利になることがあります。
例えば、ライブラリーはあなたにイベント処理のためにある[トレイト][trait]を実装する構造体を作ることを求めるかもしれません。
もしあなたが構造体に保存する必要のあるデータを全く持たないのであれば、あなたは単純にユニット的`struct`を作ることができます。

[trait]: traits.html
