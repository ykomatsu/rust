% インラインアセンブリー

極めて低レベルの作業のためや性能上の理由のために、CPUを直接制御したいと思うかもしれません。
Rustはこれを行うために`asm!`マクロによるインラインアセンブリーの使用をサポートします。
構文はGCCとClangのそれと大体一致します。

```ignore
asm!(assembly template
   : output operands
   : input operands
   : clobbers
   : options
   );
```

`asm`の使用は全て（許可するためにクレートでの`#![feature(asm)]`を要求する）機能ゲートの内側にあり、もちろん`unsafe`ブロックを要求します。

> **注意** ：ここでの例はx86/x86-64アセンブリーで与えられますが、全てのプラットフォームがサポートされています。

## アセンブリーテンプレート

`assembly template`は唯一要求される引数で、リテラル文字列（つまり`""`）でなければなりません。

```rust
#![feature(asm)]

#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
fn foo() {
    unsafe {
        asm!("NOP");
    }
}

// other platforms
#[cfg(not(any(target_arch = "x86", target_arch = "x86_64")))]
fn foo() { /* ... */ }

fn main() {
    // ...
    foo();
    // ...
}
```

（今後は`feature(asm)`と`#[cfg]`を省略します。）

出力オペランド、入力オペランド、クロバー、オプションは全て任意です。しかし、もしあなたがそれらを飛ばすのであれば、あなたは正しい個数の`:`を付けなければなりません。

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() { unsafe {
asm!("xor %eax, %eax"
    :
    :
    : "{eax}"
   );
# } }
```

ホワイトスペースも問題にはなりません。

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() { unsafe {
asm!("xor %eax, %eax" ::: "{eax}");
# } }
```

## オペランド

入力オペランドと出力オペランドは同じ書式、`: "constraints1"(expr1), "constraints2"(expr2), ..."`に従います。
出力オペランドの式はミュータブルなlvalueか未割当てのどちらかでなければなりません。

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
fn add(a: i32, b: i32) -> i32 {
    let c: i32;
    unsafe {
        asm!("add $2, $0"
             : "=r"(c)
             : "0"(a), "r"(b)
             );
    }
    c
}
# #[cfg(not(any(target_arch = "x86", target_arch = "x86_64")))]
# fn add(a: i32, b: i32) -> i32 { a + b }

fn main() {
    assert_eq!(add(3, 14159), 14162)
}
```

しかし、もしあなたがこの位置に実際のオペランドを使いたいのであれば、あなたは波括弧`{}`をあなたの求めるレジスターの周りに書かなければなりません。そして、あなたはオペランドの特定のサイズを書かなければなりません。
これはあなたが使うレジスターが重要になる、非常に低レベルのプログラミングのために便利です。

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# unsafe fn read_byte_in(port: u16) -> u8 {
let result: u8;
asm!("in %dx, %al" : "={al}"(result) : "{dx}"(port));
result
# }
```

## クロバー

いくつかの命令はレジスターを変更します。それがなければ、そのレジスターは異なる値を保持したままだったかもしれません。そのため、私たちはそれらのレジスターに読み込まれた全ての値が正しいままだとみなさないようにコンパイラーに指示するためにクロバーリストを使います。

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() { unsafe {
// Put the value 0x200 in eax
asm!("mov $$0x200, %eax" : /* no outputs */ : /* no inputs */ : "{eax}");
# } }
```

入力レジスターと出力レジスターを一覧に入れる必要はありません。なぜなら、その情報は既に与えられた制約によって伝えられているからです。
そうでなければ、黙示的にせよ明示的にせよ使われるその他のレジスターは全て一覧に入れるべきです。

もしアセンブリーが条件コードレジスターを変更するのであれば、`cc`はクロバーの1つとして明示されるべきです。
同様に、もしアセンブリーがメモリーを変更するのであれば、`memory`も明示されるべきです。

## オプション

最後のセクション、`options`はRust特有です。
書式はコンマ区切りのリテラル文字列（つまり、`:"foo", "bar", "baz"`）です。
それはインラインアセンブリーについての追加の情報を明示するために使われます。

現在の正しいオプションは次のとおりです。

1. *volatile* - この指定はGCCとClangでの`__asm__ __volatile__ (...)`と似たものである
2. *alignstack* - ある命令はスタックが一定の手順で整列されることを期待する（つまりSSE）。
   そしてこの指定はその通常のスタック整列コードを挿入することをコンパイラーに指示する
3. *intel* - デフォルトのAT&Tの代わりにインテルの構文を使う

```rust
# #![feature(asm)]
# #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]
# fn main() {
let result: i32;
unsafe {
   asm!("mov eax, 2" : "={eax}"(result) : : : "intel")
}
println!("eax is currently {}", result);
# }
```

## さらなる情報

`asm!`マクロの現在の実装は[LLVMのインラインアセンブラー式][llvm-docs]に直接バインドしています。そのため、クロバー、制約などについてのさらなる情報を得るために[それらのドキュメントも][llvm-docs]必ずチェックしましょう。

[llvm-docs]: http://llvm.org/docs/LangRef.html#inline-assembler-expressions
