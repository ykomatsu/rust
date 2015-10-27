% 参照とボローイング

このガイドはRustの所有権システムの3つの発表の1つです。
これはRustの最も独特で注目されている機能です。そして、Rust開発者はそれについて高度に精通しておくべきです。
所有権はどのようにRustがその最大の目標、メモリー安全性を得るのかということです。
そこにはいくつかの別個の概念があり、各概念が独自の章を持ちます。

* 鍵となる概念、[所有権][ownership]
* あなたが今読んでいる、ボローイング
* ボローイングのもう一歩進んだ概念、[生存期間][lifetimes]

それらの3つの章は関連していて、それらには順序があります。
あなたは所有権システムを完全に理解するために、3つ全てを必要とするでしょう。

[ownership]: ownership.html
[lifetimes]: lifetimes.html

# メタ

私たちが詳細に入る前に、所有権システムについての2つの重要な注意があります。

Rustは安全性とスピートに焦点を合わせます。
Rustはそれらの目標をたくさんの「0コストの抽象化」を通じて成し遂げます。それは、Rustでは抽象化を機能させるためのコストをできる限り小さくすることを意味します。
所有権システムは0コストの抽象化の主な例です。
私たちがこのガイドの中で話すであろう解析の全ては _コンパイル時に行われます_ 。
あなたはそれらのどの機能に対しても実行時のコストを全く支払いません。

しかし、このシステムはあるコストを持ちます。それは学習曲線です。
多くの新しいRustのユーザーは私たちが「ボローチェッカーとの戦い」と好んで呼ぶものを経験します。そこではRustコンパイラーが開発者が正しいと考えるプログラムをコンパイルすることを拒絶します。
所有権がどのように機能するのかについてのプログラマーのメンタルモデルがRustの実装する実際のルールにマッチしないため、これはしばしば起きます。
しかし、よいニュースがあります。より経験豊富なRustの開発者は次のことを報告します。一度彼らが所有権システムのルールとともにしばらく仕事をすれば、彼らがボローチェッカーと戦うことは少なくなっていくということです。

それを念頭に置いて、ボローイングについて学びましょう。

# ボローイング

[所有権][ownership]セクションの最後に、私たちはこのように見える厄介な関数を持ちました。

```rust
fn foo(v1: Vec<i32>, v2: Vec<i32>) -> (Vec<i32>, Vec<i32>, i32) {
    // do stuff with v1 and v2

    // hand back ownership, and the result of our function
    (v1, v2, 42)
}

let v1 = vec![1, 2, 3];
let v2 = vec![1, 2, 3];

let (v1, v2, answer) = foo(v1, v2);
```

しかし、これは慣習的なRustではありません。なぜなら、それはボローイングの利点を使わないからです。
これが最初のステップです。

```rust
fn foo(v1: &Vec<i32>, v2: &Vec<i32>) -> i32 {
    // do stuff with v1 and v2

    // return the answer
    42
}

let v1 = vec![1, 2, 3];
let v2 = vec![1, 2, 3];

let answer = foo(&v1, &v2);

// we can use v1 and v2 here!
```

私たちの引数として`Vec<i32>`を使う代わりに、私たちは参照、つまり`&vec<i32>`を使います。
そして、`v1`と`v2`を直接渡す代わりに、私たちは`&v1`と`&v2`を渡します。
私たちは`&T`型を「参照」と呼びます。それは、リソースを所有するのではなく、所有権をボローします。
何かをボローした束縛はそれがスコープから外れるときにリソースを割当解除しません。
これは`foo()`の呼出しの後に私たちは私たちの元の束縛を再び使うことができることを意味します。

参照は束縛とちょうど同じようにイミュータブルです。
これは`foo()`の中ではベクターは全く変更できないことを意味します。

```rust,ignore
fn foo(v: &Vec<i32>) {
     v.push(5);
}

let v = vec![];

foo(&v);
```

このエラーが出ます。

```text
error: cannot borrow immutable borrowed content `*v` as mutable
v.push(5);
^
```

値の挿入はベクターを変更し、そして私たちはそうすることを許されていません。

# &mut参照

参照には2種類目、`&mut T`があります。
「ミュータブルな参照」はあなたにあなたがボローしているリソースの変更を許します。
例はこうなります。

```rust
let mut x = 5;
{
    let y = &mut x;
    *y += 1;
}
println!("{}", x);
```

これは`6`をプリントするでしょう。
私たちは`y`を`x`へのミュータブルな参照にして、それから`y`の指示先に1を足します。
あなたは`x`も`mut`とマークしなければならないことに気付くでしょう。もしそれをそうしなかったならば、私たちはイミュータブルな値へのミュータブルなボローを使うことはできません。

あなたは私たちがアスタリスク（`*`）を`y`の前に追加して、それを`*y`にしたことにも気付くでしょう。これは、`y`が`&mut`参照だからです。
あなたは参照の内容にアクセスするためにもそれらを使う必要があるでしょう。

そうしなければ、`&mut`参照はただの参照です。
しかし、2つの間には、そしてそれらがどのように反応するかには大きな違いがあります。
あなたは前の例で何かが怪しいと言うことができます。なぜなら、私たちは`{`と`}`を使って追加のスコープを必要とするからです。
もし私たちがそれらを削除すれば、エラーが出ます。

```text
error: cannot borrow `x` as immutable because it is also borrowed as mutable
    println!("{}", x);
                   ^
note: previous borrow of `x` occurs here; the mutable borrow prevents
subsequent moves, borrows, or modification of `x` until the borrow ends
        let y = &mut x;
                     ^
note: previous borrow ends here
fn main() {

}
^
```

結論から言うと、そこにはルールがあります。

# ルール

これがRustでのボローイングについてのルールです。

最初に、ボローは全て所有者のスコープより長く存続してはなりません。
次に、あなたは次の2種類のボローのどちらか1つを持つかもしれませんが、両方を同時に持つことはありません。

* リソースに対する1つ以上の参照（`&T`）
* ただ1つのミュータブルな参照（`&mut T`）

あなたはこれがデータ競合の定義と非常に似ていることに気付くかもしれません。全く同じではありませんが。

> 「データ競合」は2つ以上のポインターがメモリーの同じ場所に同時にアクセスするとき、少なくともそれらの1つが書込みを行っていて、作業が同期されていないところで「データ競合」は起きます。

参照を使うとき、あなたは好きなだけ参照を持つかもしれません。しかし、それらのどれも書込みは行いません。
もしあなたが書込みを行うのであれば、あなたは同じメモリーへの2つ以上のポインターを必要とし、同時に1つだけ`&mut`を持つことができます。
これがどのようにRustがデータ競合をコンパイル時に回避するのかということです。もし私たちがルールを破れば、そのときはエラーが出るでしょう。

これを念頭に置いて、もう一度私たちの例を考えましょう。

## スコープの考え方

これがコードです。

```rust,ignore
let mut x = 5;
let y = &mut x;

*y += 1;

println!("{}", x);
```

このコードはこのエラーを出します。

```text
error: cannot borrow `x` as immutable because it is also borrowed as mutable
    println!("{}", x);
                   ^
```

これは私たちがルールに違反しているからです。つまり、私たちは`x`を指示する`&mut T`を持つので、`&T`を作ることは許されていないということです。
どちらか1つです。
メモはこの問題についての考え方のヒントを示します。

```text
note: previous borrow ends here
fn main() {

}
^
```

言い換えれば、ミュータブルなボローは私たちの例の残りの間ずっと保持されます。
私たちが欲しいものは、私たちが`println!`を呼び出し、イミュータブルなボローを作ろうとする _前に_ 終わるミュータブルなボローです。
Rustではボローイングはボローが有効なスコープと結び付けられます。
そして私たちのスコープはこのように見えます。
s
```rust,ignore
let mut x = 5;

let y = &mut x;    // -+ &mut borrow of x starts here
                   //  |
*y += 1;           //  |
                   //  |
println!("{}", x); // -+ - try to borrow x here
                   // -+ &mut borrow of x ends here
```

スコープは衝突します。`y`がスコープにある間d、私たちは`&x`を作ることができません。

そして、私たちが波括弧を追加するときはこうなります。

```rust
let mut x = 5;

{                   
    let y = &mut x; // -+ &mut borrow starts here
    *y += 1;        //  |
}                   // -+ ... and ends here

println!("{}", x);  // <- try to borrow x here
```

問題ありません。
私たちのミュータブルなボローは私たちがイミュータブルなボローを作る前にスコープから外れます。
しかしスコープはボローがどれくらい存続するのかを考えるための鍵です。

## ボローイングが回避する問題

なぜこのような厳格なルールを持つのでしょうか。
ええ、私たちが言及したように、それらのルールはデータ競合を回避します。
データ競合はどのような種類の問題を起こすのでしょうか。
これが一部です。

### イテレーターの不正

一例は「イテレーターの不正」です。それはあなたがあなたが繰返しを行っているコレクションを変更しようとするときに起こります。
Rustのボローチェッカーはこれの発生を回避します。

```rust
let mut v = vec![1, 2, 3];

for i in &v {
    println!("{}", i);
}
```

これは1から3までをプリントアウトします。
私たちはベクターを繰り返すので、私たちは要素への参照だけ与えられます。
そして、`v`はそれ自体イミュータブルとしてボローされ、それは私たちが繰返しを行っている間はそれを変更できないことを意味します。

```rust,ignore
let mut v = vec![1, 2, 3];

for i in &v {
    println!("{}", i);
    v.push(34);
}
```

これがエラーです。

```text
error: cannot borrow `v` as mutable because it is also borrowed as immutable
    v.push(34);
    ^
note: previous borrow of `v` occurs here; the immutable borrow prevents
subsequent moves or mutable borrows of `v` until the borrow ends
for i in &v {
          ^
note: previous borrow ends here
for i in &v {
    println!(“{}”, i);
    v.push(34);
}
^
```

`v`はループによってボローされるので、私たちはそれを変更できません。

### 解放後の使用

参照はそれらの指示するリソースよりも長く生存することはできません。
Rustはこれが真であることを保証するために、あなたの参照のスコープをチェックするでしょう。

もしRustがこの所有物をチェックしなければ、私たちは不正な参照をうっかり使ってしまうことがあります。
例えばこうです。

```rust,ignore
let y: &i32;
{ 
    let x = 5;
    y = &x;
}

println!("{}", y);
```

このエラーが出ます。

```text
error: `x` does not live long enough
    y = &x;
         ^
note: reference must be valid for the block suffix following statement 0 at
2:16...
let y: &i32;
{ 
    let x = 5;
    y = &x;
}

note: ...but borrowed value is only valid for the block suffix following
statement 0 at 4:18
    let x = 5;
    y = &x;
}
```

言い換えれば、`y`は`x`が存在するスコープの中でだけ有効です。
`x`がなくなるとすぐに、それを指示することは不正になります。
そのように、エラーはボローが「十分長く生存していない」ことを示します。なぜなら、それが正しい期間正しくないからです。

参照がそれの参照する変数より前に宣言されたとき、同じ問題が起こります。
これは同じスコープにあるリソースはそれらの宣言された順番と逆に解放されるからです。

```rust,ignore
let y: &i32;
let x = 5;
y = &x;

println!("{}", y);
```

このエラーが出ます。

```text
error: `x` does not live long enough
y = &x;
     ^
note: reference must be valid for the block suffix following statement 0 at
2:16...
    let y: &i32;
    let x = 5;
    y = &x;

    println!("{}", y);
}

note: ...but borrowed value is only valid for the block suffix following
statement 1 at 3:14
    let x = 5;
    y = &x;

    println!("{}", y);
}
```

前の例では、`y`は`x`より前に宣言されています。それは、`y`が`x`より長く生存することを意味し、それは許されません。
