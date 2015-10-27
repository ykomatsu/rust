% ミュータビリティー

ミュータビリティー、何かを変更する能力はRustでは他の言語の中でとは少し違って動作します。
ミュータビリティーの最初の面はそれが非デフォルトの状態だということです。

```rust,ignore
let x = 5;
x = 6; // error!
```

私たちは`mut`キーワードを使ってミュータビリティーを導入することができます。

```rust
let mut x = 5;

x = 6; // no problem!
```

これはミュータブルな[変数束縛][vb]です。
束縛がミュータブルなとき、それはあなたが束縛の指示するものを変更することができることを意味します。
そのため、前の例では`x`の値を変更しているというよりはむしろ束縛を`i32`から他に変更しています。

[vb]: variable-bindings.html

もしあなたが束縛の指示するものを変更したいのであれば、あなたは[ミュータブルな参照][mr]を使う必要があるでしょう。

```rust
let mut x = 5;
let y = &mut x;
```

[mr]: references-and-borrowing.html

`y`はミュータブルな参照へのイミュータブルな束縛です。それはあなたが`y`を他の何かに束縛すること（`y = &mut z`）はできないが、`y`に束縛されているものを変更すること（`*y = 5`）はできるということを意味します。
微妙な区別です。

もちろん、あなたが両方を必要とするのであれば、こうします。

```rust
let mut x = 5;
let mut y = &mut x;
```

今`y`はもう1つの値に束縛されています。そして、それの参照している値は変更することができます。

`mut`が[パターン][pattern]の一部であることに気付くことは重要です。そして、あなたはこのようなことをすることができます。

```rust
let (mut x, y) = (5, 6);

fn foo(mut x: i32) {
# }
```

[pattern]: patterns.html

# 内的ミュータビリティー対外的ミュータビリティー

しかし、私たちが何かが「イミュータブル」だとRustで言ったとき、それはそれを変更することができないということを意味しません。私たちは何かが「外的ミュータビリティー」を持つことを示します。
例えば、[`Arc<T>`][arc]を考えましょう。

```rust
use std::sync::Arc;

let x = Arc::new(5);
let y = x.clone();
```

[arc]: ../std/sync/struct.Arc.html

私たちが`clone()`を呼び出すとき、`Arc<T>`は参照カウントの更新を必要とします。
しかし、私たちはここでは`mut`を全く使っていないので、`x`はイミュータブルな束縛です。私たちは`&mut 5`など受け取りませんでした。
では、何を与えられたのでしょうか。

これを理解するために、私たちはRustの指針となる哲学のコア、メモリー安全性、そしてRustがそれを保証するために用いるメカニズム、[所有権][ownership]システム、そして特に[ボローイング][borrowing]に立ち戻る必要があります。

> あなたは次の2種類のボローのどちらか1つを持つかもしれませんが、同時に両方を持つことはありません。
>
> * リソースへの1つ以上の参照（`&T`）
> * ただ1つのミュータブルな参照（`&mut T`）

[ownership]: ownership.html
[borrowing]: references-and-borrowing.html#borrowing

そして、それが「イミュータビリティー」の本当の定義です。2つのポインターを持つことは安全でしょうか。
`Arc<T>`の場合は安全です。変更は全体として構造自体の中に含まれます。
それはユーザーに直面しません。
この理由から、それは`&T`を`clone()`で手渡します。
しかし、もしそれが`&mut T`を手渡したならば、それは問題になるでしょう。

[`std::cell`][stdcell]モジュール内のもののような他の型は逆のもの、内的ミュータビリティーを持ちます。
例えばこうです。

```rust
use std::cell::RefCell;

let x = RefCell::new(42);

let y = x.borrow_mut();
```

[stdcell]: ../std/cell/index.html

RefCellはそれの中身への`&mut`参照を`borrow_mut()`メソッドで手渡します。
それは危険ではないのでしょうか。
私たちが次のことをしたらどうなるでしょうか。

```rust,ignore
use std::cell::RefCell;

let x = RefCell::new(42);

let y = x.borrow_mut();
let z = x.borrow_mut();
# (y, z);
```

これは実際、実行時にパニックするでしょう。
これが`RefCell`のしていることです。つまり、それはRustのボローイングルールを実行時に実施し、それらが違反されれば`panic!`します。
これによって私たちはRustのミュータビリティールールのもう1つの面を乗り越えることが可能になります。
まずそれについて話しましょう。

## フィールドレベルのミュータビリティー

ミュータビリティーはボロー（`&mut`）と束縛（`let mut`）の両方の特性です。
これは例えば、あなたがいくつかのフィールドがミュータブルでいくつかのフィールドがイミュータブルな[`struct`][struct]を持つことができないということを意味します。

```rust,ignore
struct Point {
    x: i32,
    mut y: i32, // nope
}
```

構造体のミュータビリティーはその束縛の中にあります。

```rust,ignore
struct Point {
    x: i32,
    y: i32,
}

let mut a = Point { x: 5, y: 6 };

a.x = 10;

let b = Point { x: 5, y: 6};

b.x = 10; // error: cannot assign to immutable field `b.x`
```

[struct]: structs.html

しかし、[`Cell<T>`][cell]を使うことであなたはフィールドレベルのミュータビリティーをエミュレートすることができます。

```rust
use std::cell::Cell;

struct Point {
    x: i32,
    y: Cell<i32>,
}

let point = Point { x: 5, y: Cell::new(6) };

point.y.set(7);

println!("y: {:?}", point.y);
```

[cell]: ../std/cell/struct.Cell.html

これは`y: Cell { value: 7 }`をプリントするでしょう。
私たちは`y`の更新に成功しました。
