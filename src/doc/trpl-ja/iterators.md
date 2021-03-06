% イテレーター

ループについて話しましょう

Rustの`for`ループを覚えていますか。
これが例です。

```rust
for x in 0..10 {
    println!("{}", x);
}
```

あなたは前よりもRustの知識を得ているので、私たちはこれがどのように動作するのかについて詳細に話すことができます。
レンジ（`0..10`）は「イテレーター」です。
イテレーターとは、`.next()`メソッドを繰り返し呼び出すことのできるもので、イテレーターは何かのシーケンスを提供します。

こういう感じです。

```rust
let mut range = 0..10;

loop {
    match range.next() {
        Some(x) => {
            println!("{}", x);
        },
        None => { break }
    }
}
```

私たちはレンジのミュータブルな束縛を作りましたが、これが私たちのイテレーターです。
それから、私たちは内部の`match`を`loop`します。
この`match`は`range.next()`の結果を使います。それは私たちにイテレーターの次の値の参照を提供します。
この場合、`next`は`Option<i32>`を戻します。これは、私たちが値が得られれば`Some(i32)`に、値を使い切ってしまっていれば`None`になります。
もし私たちが`Some(i32)`を得れば、それを出力し、`None`を得れば、私たちはループの外に`break`します。

このコード例は私たちの`for`ループバージョンと基本的には同じです。
`for`ループはこの`loop`/`match`/`break`構造を手軽に書くための方法にすぎません。

しかし、イテレーターを使うのは`for`ループだけではありません。
あなたが自分でイテレーターを書くには、`Iterator`トレイトを実装しなければなりません。
それはこのガイドの範囲外ですが、Rustは様々な仕事を成し遂げるのに有用なイテレーターをいくつも提供しています。
しかし最初に、レンジについては知っておかなければならない制限がいくつかあります。

レンジは非常に原始的なので、私たちはもっとよい代替品を使うことができます。
Rustにおける次のアンチパターンを検討しましょう。Cスタイルの`for`ループをエミュレートするためにレンジを使うことです。
あなたはベクターの内容を繰り返さなければならなくなったとしましょう。
あなたはこのように書きたくなるかもしれません。

```rust
let nums = vec![1, 2, 3];

for i in 0..nums.len() {
    println!("{}", nums[i]);
}
```

これは実際のイテレーターを使うよりもどう見ても悪いです。
あなたはベクターを直接繰り返すことができるので、こう書くことができます。

```rust
let nums = vec![1, 2, 3];

for num in &nums {
    println!("{}", num);
}
```

これには2つの理由があります。
1つ目に、これはインデックスを使って繰り返し、それからベクターの要素をインデックスで特定するよりも私たちの意図を直接に表現しています。
2つ目に、このバージョンはより効率的です。最初のバージョンはインデックス、`nums[i]`を使っていて、余分な境界チェックを発生させるからです。
しかし、イテレーターを使えば、私たちはベクターの各要素の参照を得られるので、2つ目の例では境界チェックが発生しないのです。
これはイテレーターでは非常に一般的なことです。私たちは安全であることを認識しながら、不要な境界チェックを省くことができます。

ここではもう1つ細かいことがあります。`println!`がどのようにして動いているのかということが100％明らかではないということです。
`num`は実際には`&i32`型です。
つまり、`i32`の参照は`i32`そのものとは異なります。
`println!`は私たちのために参照解決を扱ってくれるので、私たちはそれを知らないのです。
このコードでもちゃんと動きます。

```rust
let nums = vec![1, 2, 3];

for num in &nums {
    println!("{}", *num);
}
```

今度は、私たちは明示的に`num`を参照解決しています。
なぜ`&nums`は参照を提供しているのでしょうか。
最初に、私たちは`&`を使って明示的にそれを要求しているからということです。
次に、もしそれがデータそのものを提供すれば、私たちはその所有者になってしまいます。それは、データのコピーを生成し、そのコピーされたデータを提供することを伴います。
参照を使うと、私たちは単にデータの参照をボローし、ムーブを必要とせずに参照を渡すことになります。

さて、私たちはレンジがしばしばあなたの望むものではないということを証明したので、あなたが代わりに必要とするものについて話しましょう。

ここで関連するものは、3つの大きな分類です。イテレーター、 *イテレーターアダプター* 、 *コンシューマー* です。
これが定義です。

* *イテレーター* はあなたに値のシーケンスを提供する
* *イテレーターアダプター* はイテレーターについて作業を行い、別のシーケンスを出力する新しいイテレーターを生成する
* *コンシューマー* はイテレーターについて作業を行い、値の最終的な組を生成するs

まずはコンシューマーについて話し、それからあなたが既に見ているイテレーター、レンジについて話しましょう。

## コンシューマー

*コンシューマー* はイテレーターを操作し、何らかの種類の値を戻します。
最も一般的なコンシューマーは`collect()`です。
このコードは完全にコンパイルすることはできませんが、意図を表しています。

```rust,ignore
let one_to_one_hundred = (1..101).collect();
```

見てのとおり、私たちはイテレーターの`collect()`を呼び出しています。
`collect()`はイテレーターが提供する値を全て受け取り、結果のコレクションを戻します。
なぜこのコードはコンパイルできないのでしょうか。
Rustはあなたが集めようとしているものの型を決定することができないので、あなたはそれを知らせる必要があります。
これがコンパイルできるバージョンです。

```rust
let one_to_one_hundred = (1..101).collect::<Vec<i32>>();
```

あなたは覚えているでしょう。`::<>`構文を使うと型ヒントを与えることができるので、私たちはRustに整数のベクターが欲しいということを教えています。
ただし、あなたはいつも完全な型を使う必要はありません。
`_`を使うと、あなたはヒントを部分的に与えることができます。

```rust
let one_to_one_hundred = (1..101).collect::<Vec<_>>();
```

これは「`Vec<T>`の中に集めてください。ただし、`T`が何かということは私のために推論してください」という意味です。
この理由から、`_`はときどき「型プレースホルダー」と呼ばれます。

`collect()`は最も一般的なコンシューマーですが、他のものもあります。
`find()`がそうです。

```rust
let greater_than_forty_two = (0..100)
                             .find(|x| *x > 42);

match greater_than_forty_two {
    Some(_) => println!("Found a match!"),
    None => println!("No match found :("),
}
```

`find`はクロージャーを受け取り、イテレーターの各要素の参照を使います。
このクロージャーは、もしその要素が私たちの探しているものであれば`true`を戻し、そうでなければ`false`を戻します。
`find`は指定した述語を満たす最初の要素を戻します。
私たちは述語を満たす要素を見付けられないかもしれないので、`find`はその要素そのものではなく`Option`を戻します。

もう1つの重要なコンシューマーは`fold`です。
それは、このような感じです。

```rust
let sum = (1..4).fold(0, |sum, x| sum + x);
```

`fold()`はこのような感じのコンシューマーです。
`fold(base, |accumulator, element| ...)`。
それは2つの引数を取ります。1つ目は *初期値* と呼ばれる要素です。
2つ目はクロージャーで、それ自体も2つの要素を取ります。1つ目は *アキュムレーター* と呼ばれるもので、2つ目は *要素* です。
各繰返しでは、クロージャーが呼び出され、その結果が次の繰返しではアキュムレーターになります。
1回目の繰返しでは、初期値がアキュムレーターの値になります。

はい、ちょっとややこしいです。
このイテレーターにおけるそれらの全ての値を検査しましょう。

| base | accumulator | element | closure result |
|------|-------------|---------|----------------|
| 0    | 0           | 1       | 1              |
| 0    | 1           | 2       | 3              |
| 0    | 3           | 3       | 6              |

私たちは`fold()`をそれらの引数で呼び出します。

```rust
# (1..4)
.fold(0, |sum, x| sum + x);
```

`0`が私たちの初期値で、`sum`がアキュムレーター、`x`が要素です。
1回目の繰返しでは、私たちは`sum`に`0`をセットし、`x`は`nums`の先頭の要素、`1`です。
それから私たちは`sum`と`x`とを足し、それは`0 + 1 = 1`になります。
2回目の繰返しでは、その値が私たちのアキュムレーター、`sum`になり、要素は配列の2番目の要素、`2`になります。
`1 + 2 = 3`、それが最後の繰返しのアキュムレーターの値になります。
その繰返しでは、`x`は最後の要素、`3`で、`3 + 3 = 6`、それが私たちの合計の最終的な結果です。
`1 + 2 + 3 = 6`、それが私たちの得た結果です。

うわっ。
あなたが`fold`を見るとき、始めの何回かはちょっと不思議に見えるかもしれません。しかし、一度理解すればあなたはそれをあらゆる場面で使うことができます。
何かのリストがあって、あなたが単一の結果を望んでいるのであればいつでも、`fold`は適切です。

私たちがまだ話していないイテレーターのもう1つの特性からも、コンシューマーは重要です。それは遅延です。
イテレーターについてもう少し話しましょう。あなたはなぜコンシューマーが重要なのかを理解できるでしょう。

## イテレーター

私たちが前に言ったように、イテレーターは`.next()`メソッドを繰り返し呼び出すことのできるものであって、何かのシーケンスを提供するものです。
あなたはそのメソッドを呼び出す必要があるので、これはイテレーターを *遅延* させることができ、全ての値があらかじめ生成されているとは限らないということを意味します。
例えば、このコードは実際に数`1-99`を生成するのではなく、単にシーケンスを表現する値を作っています。

```rust
let nums = 1..100;
```

私たちはレンジに対して何もしていないので、それはシーケンスを生成しません。
コンシューマーを追加しましょう。

```rust
let nums = (1..100).collect::<Vec<i32>>();
```

今度は、`collect()`がレンジにいくつかの数値を要求するので、それはシーケンスを生成する作業を行います。

レンジはあなたが見ることになる2つの基本的なイテレーターのうちの1つです。
もう1つは`iter()`です。
`iter()`はベクターを、各要素を順番に提供する単純なイテレーターに変換します。

```rust
let nums = vec![1, 2, 3];

for num in nums.iter() {
   println!("{}", num);
}
```

これらの2つの基本的なイテレーターはあなたに十分なものをもたらすはずです。
一歩進んだイテレーターがあと少しあり、その中には無限を表すものも含まれています。

これでイテレーターについては十分です。
イテレーターアダプターはイテレーターに関して私たちが話すべき最後の概念です。
行きましょう！

## イテレーターアダプター

*イテレーターアダプター* はイテレーターを受け取り、それに何らかの変更を加え、新しいイテレーターを生成します。
最も単純なイテレーターは`map`と呼ばれるものです。

```rust,ignore
(1..100).map(|x| x + 1);
```

`map`はイテレーターから呼び出され、各要素の参照が引数として与えられたクロージャーを適用される新しいイテレーターを生成します。
よって、私たちはこれで`2-100`の数値を得ることになります。
ああ、もう少しです！
もしあなたがこの例をコンパイルすれば、警告が出るでしょう。

```text
warning: unused result which must be used: iterator adaptors are lazy and
         do nothing unless consumed, #[warn(unused_must_use)] on by default
(1..100).map(|x| x + 1);
 ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

また遅延に突き当たりました！
そのクロージャーが実行されることはないでしょう。
この例は何の数値も出力しません。

```rust,ignore
(1..100).map(|x| println!("{}", x));
```

もしあなたが副作用を目的としてイテレーターでクロージャーを実行しようとするのであれば、代わりに単なる`for`を使いましょう。

興味深いイテレーターアダプターがたくさんあります。
`take(n)`は元のイテレーターの要素を`n`個生成するイテレーターを戻します。
無限のイテレーターでこれを試しましょう。

```rust
for i in (1..).take(5) {
    println!("{}", i);
}
```

これは次のものを出力します。

```text
1
2
3
4
5
```

`filter()`はクロージャーを引数として受け取るアダプターです。
このクロージャーは`true`か`false`を戻します。
`filter()`の生成する新しいイテレーターは、そのクロージャーが`true`を戻す要素だけを生成します。

```rust
for i in (1..100).filter(|&x| x % 2 == 0) {
    println!("{}", i);
}
```

これは1から100までの間の全ての奇数を出力します。
（次の点に注意しましょう。`filter`は繰り返される要素を変更しないので、それは各要素の参照を渡します。そのため、フィルターとなる述語は整数そのものを得るために`&x`パターンを使います。）

あなたは3つのもの全てを連結することができます。イテレーターから始めて、イテレーターアダプターを何回か適用し、それからコンシューマーで結果を得ます。
確認しましょう。

```rust
(1..)
    .filter(|&x| x % 2 == 0)
    .filter(|&x| x % 3 == 0)
    .take(5)
    .collect::<Vec<i32>>();
```

あなたはこれで`6`、`12`、`18`、`24`、`30`を含むベクターを得ることができます。

これはイテレーター、イテレーターアダプター、コンシューマーがどれだけあなたの役に立つかということのほんの一口にすぎません。
本当に便利なイテレーターがいくつもあり、あなたは自分で独自のイテレーターを書くこともできます。
イテレーターはあらゆる種類のリストを安全で効率的に操作する方法を提供します。
最初は少し変わって見えますが、それらで遊んでみれば、やがて癖になるでしょう。
その他のイテレーターやコンシューマーの全てのリストについては、[イテレーターモジュールのドキュメント](../std/iter/index.html)をチェックしましょう。
