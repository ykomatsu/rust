% ループ

Rustは現在、いくつかの種類の繰返しを行うために3つのアプローチを提供します。
それらは、`loop`、`while`、`for`です。
各アプローチは使い方の独自のセットを持ちます。

## ループ

無限`loop`はRustでの最も単純なループの形式です。
キーワード`loop`を使って、Rustは終了文に到達するまで無限にループする方法を提供します。
Rustの無限ループはこのように見えます。

```rust,ignore
loop {
    println!("Loop forever!");
}
```

## while

Rustは`while`ループも持ちます。
それはこのように見えます。

```rust
let mut x = 5; // mut x: i32
let mut done = false; // mut done: bool

while !done {
    x += x - 3;

    println!("{}", x);

    if x % 5 == 0 {
        done = true;
    }
}
```

`while`ループはあなたが何回のループを必要とするかが分からないときに正しい選択です。

もしあなたが無限ループを必要とするのであれば、あなたはこう書きたくなるかもしれません。

```rust,ignore
while true {
```

しかし、`loop`の方がこの場合を処理するにはより適しています。

```rust,ignore
loop {
```

Rustの制御フロー分析はこの構造を`while true`とは異なる方法で扱います。なぜなら、私たちはそれが常にループするであろうことを知っているからです。
一般的に、私たちがコンパイラーに多くの情報を与えれば与えるほど、それはより安全性の高い、よりよいコード生成を行うことができます。そのため、あなたが無限にループするつもりであれば、あなたは常に`loop`を選ぶべきです。

## for

`for`ループは特定の回数でループするために使われます。
しかし、Rustの`for`ループは他のシステム言語とは少し違った方法で動きます。
Rustの`for`ループはこの「Cスタイル」の`for`ループのようには見えません。

```c
for (x = 0; x < 10; x++) {
    printf( "%d\n", x );
}
```

代わりに、それはこのように見えます。

```rust
for x in 0..10 {
    println!("{}", x); // x: i32
}
```

もう少し抽象的な用語ではこうです。

```ignore
for var in expression {
    code
}
```

expressionは[イテレーター][iterator]です。
イテレーターは一連の要素を返します。
各要素はループの繰返しの1回です。
その値は名前`var`に束縛され、それはループの本文で有効になります。
一度本文が終了すると、次の値がイテレーターから取得されます。そして、私たちは再びループします。
値がなくなったとき、`for`ループは終了します。

[iterator]: iterators.html

私たちの例では、`0..10`は最初と最後の位置を受け取り、それらの値に対するイテレーターを与える式です。
上限は含まれまないので、私たちのループは`0`から`9`までをプリントし、`10`はプリントしないでしょう。

Rustは「Cスタイル」の`for`ループをあえて持ちません。
ループの各要素を手動で制御することは、経験豊富なCの開発者にとっても複雑でエラーを起こしがちです。

### 列挙

あなたが既に何回ループしたかを記録する必要があるとき、あなたは`.enumerate()`関数を使うことができます。

#### レンジの場合

```rust
for (i,j) in (5..10).enumerate() {
    println!("i = {} and j = {}", i, j);
}
```

出力はこうなります。

```text
i = 0 and j = 5
i = 1 and j = 6
i = 2 and j = 7
i = 3 and j = 8
i = 4 and j = 9
```

レンジの周りに丸括弧を追加することを忘れないでください。

#### イテレーターの場合

```rust
# let lines = "hello\nworld".lines();
for (linenumber, line) in lines.enumerate() {
    println!("{}: {}", linenumber, line);
}
```

出力はこうなります。

```text
0: Content of line one
1: Content of line two
2: Content of line three
3: Content of line four
```

## 繰返しの途中での終了

私たちが前に見たその`while`ループを見ましょう。

```rust
let mut x = 5;
let mut done = false;

while !done {
    x += x - 3;

    println!("{}", x);

    if x % 5 == 0 {
        done = true;
    }
}
```

いつ私たちがループを抜け出すべきかを知るために、私たちは専用の`mut`のブーリアンの変数束縛を保持しなくてはなりませんでした。
Rustは繰返しを変更するときに私たちを手助けする2つのキーワード、`break`と`continue`を持ちます。

この場合、私たちはループを`break`を使ってよりよい方法で書くことができます。

```rust
let mut x = 5;

loop {
    x += x - 3;

    println!("{}", x);

    if x % 5 == 0 { break; }
}
```

私たちは`loop`で無限にループし、途中で抜け出すために`break`を使います。
明示的な`return`文を使うこともループを途中で終了することができます。

`continue`も同様ですが、ループを終了する代わりに、次の繰返しに進みます。
これは奇数だけをプリントするでしょう。

```rust
for x in 0..10 {
    if x % 2 == 0 { continue; }

    println!("{}", x);
}
```

## ループラベル

あなたはネストしたループを持ち、あなたの`break`文又は`continue`文がどのループに対するものなのかを特定する必要がある状況にも遭遇するかもしれません。
ほとんどの他の言語のように、デフォルトでは`break`又は`continue`は最も内側のループに適用します。
あなたが外側のループから`break`又は`continue`したい状況では、あなたはどのループに`break`と`continue`が適用されるのかを特定するためにラベルを使うことができます。
これは`x`と`y`の両方が奇数であるときだけプリントするでしょう。

```rust
'outer: for x in 0..10 {
    'inner: for y in 0..10 {
        if x % 2 == 0 { continue 'outer; } // continues the loop over x
        if y % 2 == 0 { continue 'inner; } // continues the loop over y
        println!("x: {}, y: {}", x, y);
    }
}
```
