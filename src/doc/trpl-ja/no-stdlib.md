% 標準ライブラリーの不使用

デフォルトでは、`std`は全てのRustのクレートにリンクされます。
ある状況ではこれは望ましくありません。そして、これはクレートに`#![no_std]`属性を付けることによって避けることができます。

明らかに、単なるライブラリーよりも大切なものが人生にはあります。`#[no_std]`を実行ファイルに付けて使うことができ、エントリーポイントの制御は2通りの方法で可能です。それは、`#[start]`属性、そしてCの`main`関数のためのデフォルトのシムをあなた独自のもので上書きすることです。

`#[start]`とマークされた関数はCと同じ書式でコマンドライン引数を渡されます。

```rust
# #![feature(libc)]
#![feature(lang_items)]
#![feature(start)]
#![feature(no_std)]
#![no_std]

// Pull in the system libc library for what crt0.o likely requires
extern crate libc;

// Entry point for this program
#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    0
}

// These functions and traits are used by the compiler, but not
// for a bare-bones hello world. These are normally
// provided by libstd.
#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
# #[lang = "eh_unwind_resume"] extern fn rust_eh_unwind_resume() {}
# // fn main() {} tricked you, rustdoc!
```

コンパイラーの挿入する`main`のシムを上書きするために、それを`#![no_main]`を付けて無効にして、それから正しいABIと正しい名前を付けた適切なシンボルを作成しなければなりません。それは、コンパイラーの名前のマングリングの上書きも要求します。

```rust
# #![feature(libc)]
#![feature(no_std)]
#![feature(lang_items)]
#![feature(start)]
#![no_std]
#![no_main]

extern crate libc;

#[no_mangle] // ensure that this symbol is called `main` in the output
pub extern fn main(argc: i32, argv: *const *const u8) -> i32 {
    0
}

#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
# #[lang = "eh_unwind_resume"] extern fn rust_eh_unwind_resume() {}
# // fn main() {} tricked you, rustdoc!
```

コンパイラーは現在実行ファイルの中で呼び出すことができるシンボルについていくつかの仮定を置いています。
通常はそれらの関数は標準ライブラリーによって提供されますが、それがなければあなたはあなた自身のものを定義しなければなりません。

それら2つの関数の1つ目、`eh_personality`はコンパイラーの失敗のメカニズムによって使われます。
これはしばしばGCCのパーソナリティー関数にマップされます（さらなる情報のために[libstdの実装](../std/rt/unwind/index.html)を見ましょう）が、パニックの引き金を持たないクレートはこの関数が決して呼び出されないことを保証することができます。
2つ目の関数、`panic_fmt`もコンパイラーの失敗のメカニズムによって使われます。

## libcoreの使用

> **注意** ：コアライブラリーの構造は不安定です。代わりに標準ライブラリーを使うことが可能な限り推奨されます。

前のテクニックによって私たちはいくつかのRustのコードを実行するベアメタルの実行ファイルを手に入れました。
標準ライブラリーによって提供される機能にはたくさんのものがありますが、しかし、Rustではそれは生産性を高めるために必要です。
もし標準ライブラリーが効率的でなければ、そのときは代わりに[libcore](../core/index.html)が使われることが予定されています。

コアライブラリーの依存関係は非常に少なく、標準ライブラリー自体よりももっとポータブルです。
加えて、コアライブラリーは慣習的で実践的なRustのコードを書くために必要な機能のほとんどを持ちます。
`#![no_std]`を使うとき、Rustは私たちが`std`を使うときにそれのために行ったのとちょうど同じように、自動的に`core`クレートを注入します。

例として、これはRustの慣習を使って、Cから提供された2つのベクターのドット積を計算するプログラムです。

```rust
# #![feature(libc)]
#![feature(lang_items)]
#![feature(start)]
#![feature(no_std)]
#![feature(core)]
#![feature(core_slice_ext)]
#![feature(raw)]
#![no_std]

extern crate libc;

use core::mem;

#[no_mangle]
pub extern fn dot_product(a: *const u32, a_len: u32,
                          b: *const u32, b_len: u32) -> u32 {
    use core::raw::Slice;

    // Convert the provided arrays into Rust slices.
    // The core::raw module guarantees that the Slice
    // structure has the same memory layout as a &[T]
    // slice.
    //
    // This is an unsafe operation because the compiler
    // cannot tell the pointers are valid.
    let (a_slice, b_slice): (&[u32], &[u32]) = unsafe {
        mem::transmute((
            Slice { data: a, len: a_len as usize },
            Slice { data: b, len: b_len as usize },
        ))
    };

    // Iterate over the slices, collecting the result
    let mut ret = 0;
    for (i, j) in a_slice.iter().zip(b_slice.iter()) {
        ret += (*i) * (*j);
    }
    return ret;
}

#[lang = "panic_fmt"]
extern fn panic_fmt(args: &core::fmt::Arguments,
                    file: &str,
                    line: u32) -> ! {
    loop {}
}

#[lang = "eh_personality"] extern fn eh_personality() {}
# #[lang = "eh_unwind_resume"] extern fn rust_eh_unwind_resume() {}
# #[start] fn start(argc: isize, argv: *const *const u8) -> isize { 0 }
# fn main() {}
```

前の例とは異なり、追加の言語要素が1つあることに注意しましょう。`panic_fmt`です。
これはlibcoreの利用者によって定義されなければなりません。なぜなら、コアライブラリーはパニックを宣言しますが、それを定義しないからです。
`panic_fmt`言語要素はこのクレートのパニックの定義で、それは決してリターンしないことが保証されなければなりません。

この例で見られるように、コアライブラリーはプラットフォームの要求にかかわらず、あらゆる環境でRustの力を提供することを意図しています。
プラットフォーム特有の他の仮定を置く、liballocのようなより高機能なライブラリーはlibcoreに機能を追加します。しかし、コアライブラリーは標準ライブラリー自体よりもポータブルであり続けます。
