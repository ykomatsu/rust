% ベクター

「ベクター」は動的又は「伸ばすことのできる」配列で、標準ライブラリーの[`Vec<T>`][vec]型として実装されています。
`T`は私たちが任意の型のベクターを持つことができるということを意味します（詳しくは[ジェネリック][generic]の章を見ましょう）。
ベクターは常にそれらのデータをヒープ上に割り当てます。
あなたはそれらを`vec!`マクロを使って作ることができます。

```rust
let v = vec![1, 2, 3, 4, 5]; // v: Vec<i32>
```

（私たちが過去に使っていた`println!`マクロと異なり、私たちは`vec!`マクロでは角括弧`[]`を使うことに注意しましょう。Rustでは私たちはどちらの状況でもどちらを使うこともできます。これは単なる慣習です。）

初期値を繰り返すために`vec!`には代わりの形式があります。

```rust
let v = vec![0; 10]; // ten zeroes
```

## 要素のアクセス

ベクターの特定のインデックスの値を取得するために、私たちは`[]`を使います。

```rust
let v = vec![1, 2, 3, 4, 5];

println!("The third element of v is {}", v[2]);
```

インデックスは`0`からカウントするので、3番目の要素は`v[2]`です。

あなたが`usize`型でインデックスを指定しなければならないことに気付くことは重要です。

```ignore
let v = vec![1, 2, 3, 4, 5];

let i: usize = 0;
let j: i32 = 0;

// works
v[i];

// doesn’t
v[j];
```

非`usize`型でインデックスを指定すると、このように見えるエラーが出ます。

```text
error: the trait `core::ops::Index<i32>` is not implemented for the type
`collections::vec::Vec<_>` [E0277]
v[j];
^~~~
note: the type `collections::vec::Vec<_>` cannot be indexed by `i32`
error: aborting due to previous error
```

そのメッセージの中にはたくさんの区切りがありますが、そのコアは意味を成しています。つまり、あなたは`i32`でインデックスを指定することはできないということです。

## 繰返し

一旦あなたがベクターを持てば、あなたはその要素を`for`を使って繰り返すことができます。
そこには3つのバージョンがあります。

```rust
let mut v = vec![1, 2, 3, 4, 5];

for i in &v {
    println!("A reference to {}", i);
}

for i in &mut v {
    println!("A mutable reference to {}", i);
}

for i in v {
    println!("Take ownership of the vector and its element {}", i);
}
```

ベクターにはもっとたくさんの便利なメソッドがあります。そしてあなたは[それらのAPIドキュメント][vec]でそれらについて読むことができます。

[vec]: ../std/vec/index.html
[generic]: generics.html
