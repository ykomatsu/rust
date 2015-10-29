% メソッド構文

関数はすばらしいですが、あなたがあるデータに対してそれらの集まりを呼び出したいとき、それは扱いにくくなることがあります。
次のコードを考えましょう。

```rust,ignore
baz(bar(foo));
```

私たちはこれを左から右へと読み、そして私たちは「baz bar foo」と理解します。
しかし、これは関数が呼び出される順番ではありません。それは内側から外側へという順番、「foo bar baz」です。
私たちが代わりにこのようにすることができれば、よくならないでしょうか。

```rust,ignore
foo.bar().baz();
```

幸運なことに、先程の質問からあなたが推測したとおり、あなたはできます！　
Rustはこの「メソッド呼出構文」を`impl`キーワードを通じて使う能力を提供します。

# メソッド呼出し

これはそれがどのように動くのかということです。

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}

fn main() {
    let c = Circle { x: 0.0, y: 0.0, radius: 2.0 };
    println!("{}", c.area());
}
```

これは`12.566371`をプリントするでしょう。

私たちは円を表現する`struct`を作りました。
それから、私たちは`impl`ブロックを書き、その内側でメソッド、`area`を定義します。

メソッドは特別な最初の引数を受け取ります。そしてそれには3つの種類があります。その種類とは`self`、`&self`、`&mut self`です。
あなたはこの最初の引数を`foo.bar()`の中の`foo`として考えることができます。
3つのバリアントは`foo`がなることのできる3つの種類のものと対応します。つまり、それがスタック上のただの値のときは`seif`、それが参照のときは`&self`、それがミュータブルな参照のときは`&mut self`に対応するということです。
私たちは`&self`を`area`の引数として受け取ったので、それを他の引数とちょうど同じように使うことができます。
私たちはそれが`Circle`であることを知っているので、私たちが他の`struct`について行うのとちょうど同じように`radius`にアクセスすることができます。

あなたが所有権の移転よりもボローイングを、ミュータブルな参照よりもイミュータブルな参照を選ぶべきであるのと同様に、私たちは`&self`の使用をデフォルトとすべきです。
これが3つの種類全ての例です。

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn reference(&self) {
       println!("taking self by reference!");
    }

    fn mutable_reference(&mut self) {
       println!("taking self by mutable reference!");
    }

    fn takes_ownership(self) {
       println!("taking ownership of self!");
    }
}
```

# メソッド呼出しの連鎖

さて、今私たちは`foo.bar()`のようなメソッドの呼出方法を知りました。
しかし、私たちの元の例、`foo.bar().baz()`についてはどうでしょうか。
これは「メソッド連鎖」と呼ばれます。
例を見ましょう。

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }

    fn grow(&self, increment: f64) -> Circle {
        Circle { x: self.x, y: self.y, radius: self.radius + increment }
    }
}

fn main() {
    let c = Circle { x: 0.0, y: 0.0, radius: 2.0 };
    println!("{}", c.area());

    let d = c.grow(2.0).area();
    println!("{}", d);
}
```

戻り値の型をチェックしましょう。

```rust
# struct Circle;
# impl Circle {
fn grow(&self, increment: f64) -> Circle {
# Circle } }
```

私たちは単に私たちが`Circle`を戻すことを表します。
このメソッドを使って、私たちは新しい`Circle`を任意のサイズに大きくすることができます。

# 随伴関数

あなたは`self`引数を受け取らない随伴関数を定義することもできます。
これはRustのコードでは非常に一般的なパターンです。

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn new(x: f64, y: f64, radius: f64) -> Circle {
        Circle {
            x: x,
            y: y,
            radius: radius,
        }
    }
}

fn main() {
    let c = Circle::new(0.0, 0.0, 2.0);
}
```

この「随伴関数」は新しい`Circle`を私たちのために作ります。
随伴関数は`ref.method()`構文ではなく`Struct::function()`構文で呼び出されるということに注意しましょう。
いくつかの他の言語は随伴関数を「スタティックメソッド」と呼びます。

# ビルダーパターン

私たちが私たちのユーザーを`Circle`を作ることができるようにしたいことを表現しましょう。ただし、私たちは彼らに彼らが関心を持つプロパティーだけをセットできるようにするでしょう。
そうしない場合は`x`属性と`y`属性は`0.0`になり、`radius`は`1.0`になるでしょう。
Rustにはメソッドオーバーロード、名前付引数、可変長引数はありません。
私たちはビルダーパターンを代わりに用います。
それはこのように見えます。

```rust
struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * (self.radius * self.radius)
    }
}

struct CircleBuilder {
    x: f64,
    y: f64,
    radius: f64,
}

impl CircleBuilder {
    fn new() -> CircleBuilder {
        CircleBuilder { x: 0.0, y: 0.0, radius: 1.0, }
    }

    fn x(&mut self, coordinate: f64) -> &mut CircleBuilder {
        self.x = coordinate;
        self
    }

    fn y(&mut self, coordinate: f64) -> &mut CircleBuilder {
        self.y = coordinate;
        self
    }

    fn radius(&mut self, radius: f64) -> &mut CircleBuilder {
        self.radius = radius;
        self
    }

    fn finalize(&self) -> Circle {
        Circle { x: self.x, y: self.y, radius: self.radius }
    }
}

fn main() {
    let c = CircleBuilder::new()
                .x(1.0)
                .y(2.0)
                .radius(2.0)
                .finalize();

    println!("area: {}", c.area());
    println!("x: {}", c.x);
    println!("y: {}", c.y);
}
```

私たちがここでしたことはもう1つの`struct`、`CircleBuilder`の作成です。
私たちは私たちのビルダーメソッドをその中に定義しています。
私たちは`Circle`の中に私たちの`area()`メソッドも定義しています。
私たちは`CircleBuilder`にさらにもう1つのメソッド、`finalize()`も定義しています。
今、私たちは型システムを私たちの関心事を実施するために使っています。つまり、私たちは`CircleBuilder`内のメソッドを私たちの選んだ方法で`Circle`を作ることを強制するために使うことができるということです。
