% 他言語関数インターフェイス

# 導入

このガイドでは、他言語のコードのためのバインディングを書く導入に[snappy](https://github.com/google/snappy)という圧縮・展開ライブラリーを使います。
Rustは現在、C++ライブラリーを直接呼び出すことができませんが、snappyはCのインターフェイスを持っています（ドキュメントが[`snappy-c.h`](https://github.com/google/snappy/blob/master/snappy-c.h)にあります）。

以下はsnappyがインストールされていればコンパイルできる、他言語の関数を呼び出す最小の例です。

```no_run
# #![feature(libc)]
extern crate libc;
use libc::size_t;

#[link(name = "snappy")]
extern {
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
}

fn main() {
    let x = unsafe { snappy_max_compressed_length(100) };
    println!("max compressed length of a 100 byte buffer: {}", x);
}
```

`extern`ブロックは他言語のライブラリーの中の関数のシグネチャー、今回はそのプラットフォーム上のC ABIによるもののリストです。
`#[link(...)]`属性は、シンボルが解決できるように、リンカーに対してsnappyのライブラリーをリンクするよう指示するために使われています。

他言語の関数はアンセーフとみなされるので、それらを呼び出すには、この中に含まれているすべてのものが本当に安全であるということをコンパイラーに対して約束するために、`unsafe {}`で囲まなければなりません。
Cライブラリーは、スレッドセーフでないインターフェイスを公開していることがありますし、ポインターを引数に取る関数のほとんどは、ポインターの指示先が不正になる可能性を有しているので、すべての入力に対して有効なわけではありません。そして、生のポインターはRustの安全なメモリーモデルから外れてしまいます。

他言語の関数について引数の型を宣言するとき、Rustのコンパイラーはその宣言が正しいかどうかを確認することができません。それを正しく指定することは、実行時のバインディングを正しく保つことの一部です。

`extern`ブロックはsnappyのAPI全体をカバーするように拡張することができます。

```no_run
# #![feature(libc)]
extern crate libc;
use libc::{c_int, size_t};

#[link(name = "snappy")]
extern {
    fn snappy_compress(input: *const u8,
                       input_length: size_t,
                       compressed: *mut u8,
                       compressed_length: *mut size_t) -> c_int;
    fn snappy_uncompress(compressed: *const u8,
                         compressed_length: size_t,
                         uncompressed: *mut u8,
                         uncompressed_length: *mut size_t) -> c_int;
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
    fn snappy_uncompressed_length(compressed: *const u8,
                                  compressed_length: size_t,
                                  result: *mut size_t) -> c_int;
    fn snappy_validate_compressed_buffer(compressed: *const u8,
                                         compressed_length: size_t) -> c_int;
}
# fn main() {}
```

# 安全なインターフェイスの作成

生のC APIは、メモリーの安全性を提供し、ベクターのようなもっと高レベルの概念を使うようにラップしなければなりません。
ライブラリーは安全で高レベルなインターフェイスのみを公開するように選択することができ、不安全な内部の詳細を隠すことができます。

バッファーを期待する関数をラップするには、Rustのベクターをメモリーへのポインターとして操作するために`slice::raw`モジュールを使います。
Rustのベクターは隣接したメモリーのブロックであることが保証されています。
その長さは現在含んでいる要素の数で、容量は割り当てられたメモリーの要素の合計のサイズです。
長さは、容量以下です。

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_validate_compressed_buffer(_: *const u8, _: size_t) -> c_int { 0 }
# fn main() {}
pub fn validate_compressed_buffer(src: &[u8]) -> bool {
    unsafe {
        snappy_validate_compressed_buffer(src.as_ptr(), src.len() as size_t) == 0
    }
}
```

上の`validate_compressed_buffer`ラッパーは`unsafe`ブロックを使っていますが、関数のシグネチャーを`unsafe`から外すことによって、その呼出しがすべての入力に対して安全であることが保証されています。

結果を保持するようにバッファーを割り当てなければならないため、`snappy_compress`関数と`snappy_uncompress`関数はもっと複雑です。

`snappy_max_compressed_length`関数は、圧縮後の結果を保持するために必要な最大の容量のベクターを割り当てるために使うことができます。
そして、そのベクターは結果の引数として`snappy_compress`関数に渡されます。
結果の引数は、長さをセットするために、圧縮後の本当の長さを取得するためにも渡されます。

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_compress(a: *const u8, b: size_t, c: *mut u8,
#                           d: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(a: size_t) -> size_t { a }
# fn main() {}
pub fn compress(src: &[u8]) -> Vec<u8> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen = snappy_max_compressed_length(srclen);
        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        snappy_compress(psrc, srclen, pdst, &mut dstlen);
        dst.set_len(dstlen as usize);
        dst
    }
}
```

snappyは展開後のサイズを圧縮フォーマットの一部として保存していて、`snappy_uncompressed_length`が必要となるバッファーの正確なサイズを取得するため、展開も同様です。

```rust
# #![feature(libc)]
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t) -> c_int { 0 }
# fn main() {}
pub fn uncompress(src: &[u8]) -> Option<Vec<u8>> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen: size_t = 0;
        snappy_uncompressed_length(psrc, srclen, &mut dstlen);

        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        if snappy_uncompress(psrc, srclen, pdst, &mut dstlen) == 0 {
            dst.set_len(dstlen as usize);
            Some(dst)
        } else {
            None // SNAPPY_INVALID_INPUT
        }
    }
}
```

参照のために、ここで使った例は[GitHub上のライブラリー](https://github.com/thestinger/rust-snappy)としても置いておきます。

# デストラクター

他言語のライブラリーはリソースの所有権を呼出先のコードに手渡してしまうことがあります。
そういうことが起きる場合には、安全性を提供し、それらのリソースが解放されることを保証するために、Rustのデストラクターを使わなければなりません（特にパニックした場合）。

デストラクターについて詳しくは、[Dropトレイト](../std/ops/trait.Drop.html)を見てください。

# CのコードからRustの関数へのコールバック

外部のライブラリーの中には、現在の状況や中間的なデータを呼出元に報告するためにコールバックを使わなければならないものがあります。
Rustで定義された関数を外部のライブラリーに渡すことは可能です。
これをするために必要なのは、Cのコードから呼び出すことができるように正しい呼出規則に従って、コールバック関数を`extern`としてマークしておくことです。

そして、登録呼出しを通じてコールバック関数をCのライブラリーに送ることができるようになり、後でそれらから呼び出すことができるようになります。

基本的な例は次のとおりです。

Rustのコードです。

```no_run
extern fn callback(a: i32) {
    println!("I'm called from C with value {0}", a);
}

#[link(name = "extlib")]
extern {
   fn register_callback(cb: extern fn(i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    unsafe {
        register_callback(callback);
        trigger_callback(); // Triggers the callback
    }
}
```

Cのコードです。

```c
typedef void (*rust_callback)(int32_t);
rust_callback cb;

int32_t register_callback(rust_callback callback) {
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(7); // Will call callback(7) in Rust
}
```

この例では、Rustの`main()`がCの`trigger_callback()`を呼び出し、今度はそれが、Rustの`callback()`をコールバックしています。

## Rustのオブジェクトを対象にしたコールバック

先程の例では、グローバルな関数をCのコードから呼ぶための方法を示してきました。
しかし、特別なRustのオブジェクトをコールバックの対象にしたいことがあります。
これは、そのオブジェクトをそれぞれCのオブジェクトのラッパーとして表現することで可能になります。

これは、そのオブジェクトへの生のポインターをCライブラリーに渡すことで実現できます。
そして、Cのライブラリーはその通知の中のRustのオブジェクトへのポインターを含むことができるようになります。
これにより、そのコールバックは参照されるRustのオブジェクトにアンセーフな形でアクセスできるようになります。

Rustのコードです。

```no_run
#[repr(C)]
struct RustObject {
    a: i32,
    // other members
}

extern "C" fn callback(target: *mut RustObject, a: i32) {
    println!("I'm called from C with value {0}", a);
    unsafe {
        // Update the value in RustObject with the value received from the callback
        (*target).a = a;
    }
}

#[link(name = "extlib")]
extern {
   fn register_callback(target: *mut RustObject,
                        cb: extern fn(*mut RustObject, i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    // Create the object that will be referenced in the callback
    let mut rust_object = Box::new(RustObject { a: 5 });

    unsafe {
        register_callback(&mut *rust_object, callback);
        trigger_callback();
    }
}
```

Cのコードです。

```c
typedef void (*rust_callback)(void*, int32_t);
void* cb_target;
rust_callback cb;

int32_t register_callback(void* callback_target, rust_callback callback) {
    cb_target = callback_target;
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(cb_target, 7); // Will call callback(&rustObject, 7) in Rust
}
```

## 非同期コールバック

先程の例では、コールバックは外部のCライブラリーへの関数呼出しに対する直接の反応として呼びだされました。
実行中のスレッドの制御はコールバックの実行のためにRustからCへ、そしてRustへと切り替わりますが、最後には、コールバックはコールバックを引き起こした関数を呼び出したものと同じスレッドで実行されます。

外部のライブラリーが独自のスレッドを生成し、そこからコールバックを呼び出すときには、事態はもっと複雑になります。
そのような場合、コールバックの中のRustのデータ構造へのアクセスは特にアンセーフであり、適切な同期メカニズムを使わなければなりません。
ミューテックスのような古典的な同期メカニズムの他にも、Rustではコールバックを呼び出したCのスレッドからRustのスレッドにデータを転送するために（`std::sync::mpsc`の中の）チャネルを使うという手もあります。

もし、非同期のコールバックがRustのアドレス空間の中の特別なオブジェクトを対象としていれば、それぞれのRustのオブジェクトが破壊された後、Cのライブラリーからそれ以上コールバックが実行されないようにすることが絶対に必要です。
これは、オブジェクトのデストラクターでコールバックの登録を解除し、登録解除後にコールバックが実行されないようにライブラリーを設計することで実現できます。

# リンク

`extern`ブロックの中の`link`属性は、rustcに対してネイティブライブラリーをどのようにリンクするかを指示するための基本的な構成ブロックです。
今のところ、2つの形式のリンク属性が認められています。

* `#[link(name = "foo")]`
* `#[link(name = "foo", kind = "bar")]`

これらのどちらの形式でも、`foo`はリンクするネイティブライブラリーの名前で、2つ目の形式の`bar`はコンパイラーがリンクするネイティブライブラリーの種類です。
3つのネイティブライブラリーの種類が知られています。

* ダイナミック - `#[link(name = "readline")]`
* スタティック - `#[link(name = "my_build_dependency", kind = "static")]`
* フレームワーク - `#[link(name = "CoreFoundation", kind = "framework")]`

フレームワークはOSXターゲットでのみ利用可能であることに注意しましょう。

異なる`kind`の値はリンク時のネイティブライブラリーの使われ方の違いを意味します。
リンクの視点からすると、Rustコンパイラーは2種類の生成物を作ります。
部分生成物（rlib/staticlib）と最終生成物（dylib/binary）です。
ネイティブダイナミックライブラリーとフレームワークの依存関係は最終生成物の境界を伝えますが、スタティックライブラリーの依存関係は全く伝えません。なぜなら、スタティックライブラリーはその後に続く生成物に直接統合されてしまうからです。

このモデルをどのように使うことができるのかという例は次のとおりです。

*  ネイティブビルドの依存関係。
   ときどき、Rustのコードを書くときにC/C++のグルーが必要になりますが、ライブラリーの形式でのC/C++のコードの配布はまさに重荷です。
   このような場合、`libfoo.a`からコードを得て、それからRustのクレートで`#[link(name = "foo", kind = "static")]`によって依存関係を宣言します。

   クレートの出力の種類にかかわらず、ネイティブスタティックライブラリーは出力に含まれます。これは、ネイティブスタティックライブラリーの配布は不要だということを意味します。

*  通常のダイナミックな依存関係。
   （`readline`のような）一般的なシステムライブラリーは多くのシステムで利用可能となっていますが、それらのライブラリーのスタティックなコピーは見付からないことがしばしばあります。
   この依存関係をRustのクレートに含めるときには、（rlibのような）部分生成物のターゲットはライブラリーをリンクしませんが、（binaryのような）最終生成物にrlibを含めるときには、ネイティブライブラリーはリンクされます。

OSXでは、フレームワークはダイナミックライブラリーと同じ意味で振る舞います。

# アンセーフブロック

生のポインターの参照解決やアンセーフであるとマークされた関数の呼出しなど、いくつかの作業はアンセーフブロックの中でのみ許されます。
アンセーフブロックは不安全性を隔離し、コンパイラーに対して不安全性がブロックの外に漏れ出さないことを約束します。

一方、アンセーフな関数はそれを全世界に向けて広告します。
アンセーフな関数はこのように書きます。

```rust
unsafe fn kaboom(ptr: *const i32) -> i32 { *ptr }
```

この関数は`unsafe`ブロック又は他の`unsafe`な関数からのみ呼び出すことができます。

# 他言語のグローバルな値へのアクセス

他言語のAPIはしばしばグローバルな状態を追跡するようなことをするためのグローバルな値をエクスポートします。
それらの値にアクセスするために、それらを`extern`ブロックの中で`static`キーワードを付けて宣言します。

```no_run
# #![feature(libc)]
extern crate libc;

#[link(name = "readline")]
extern {
    static rl_readline_version: libc::c_int;
}

fn main() {
    println!("You have readline version {} installed.",
             rl_readline_version as i32);
}
```

あるいは、他言語のインターフェイスが提供するグローバルな状態を変更しなければならないこともあるかもしれません。
これをするために、スタティックな値を変更することができるように`mut`付きで宣言することができます。

```no_run
# #![feature(libc)]
extern crate libc;

use std::ffi::CString;
use std::ptr;

#[link(name = "readline")]
extern {
    static mut rl_prompt: *const libc::c_char;
}

fn main() {
    let prompt = CString::new("[my-awesome-shell] $").unwrap();
    unsafe {
        rl_prompt = prompt.as_ptr();

        println!("{:?}", rl_prompt);

        rl_prompt = ptr::null();
    }
}
```

`static mut`の付いた作用は全て、読込みと書込みの双方についてアンセーフであることに注意しましょう。
グローバルでミュータブルな状態の扱いには多大な注意が必要です。

# 他言語呼出規則

他言語で書かれたコードの多くはC ABIをエクスポートしていて、Rustは他言語の関数の呼出しのときのデフォルトとしてそのプラットフォーム上のCの呼出規則を使います。
他言語の関数の中には、特にWindows APIですが、他の呼出規則を使うものもあります。
Rustにはコンパイラーに対してどの規則を使うかを教える方法があります。

```rust
# #![feature(libc)]
extern crate libc;

#[cfg(all(target_os = "win32", target_arch = "x86"))]
#[link(name = "kernel32")]
#[allow(non_snake_case)]
extern "stdcall" {
    fn SetEnvironmentVariableA(n: *const u8, v: *const u8) -> libc::c_int;
}
# fn main() { }
```

これは`extern`ブロック全体に適用されます。
サポートされているABIの規則は次のとおりです。

* `stdcall`
* `aapcs`
* `cdecl`
* `fastcall`
* `Rust`
* `rust-intrinsic`
* `system`
* `C`
* `win64`

このリストのABIのほとんどは名前のとおりですが、`system` ABIは少し変わっています。
この規則はターゲットのライブラリーを相互利用するために適切なABIを選択します。
例えば、x86アーキテクチャーのWin32では、使われるABIは`stdcall`になります。
しかし、x86_64では、Windowsは`C`の呼出規則を使うので、`C`が使われます。
先程の例で言えば、`extern "system" { ... }`を使って、x86のためだけではなく全てのWindowsシステムのためのブロックを定義することができるということです。

# 他言語コードの相互利用

`#[repr(C)]`属性が適用されている場合に限り、Rustは`struct`のレイアウトとそのプラットフォーム上のCでの表現方法との互換性を保証します。
`#[repr(C, packed)]`を使えば、パディングなしで構造体のメンバーをレイアウトすることができます。
`#[repr(C)]`は列挙型にも適用することができます。

Rust独自のボックス（`Box<T>`）はコンテンツオブジェクトを指すハンドルとして非ヌルポインターを使います。
しかし、それらは内部のアロケーターによって管理されるため、手で作るべきではありません。
参照は型を直接指す非ヌルポインターとみなすことが間違いなくできます。
しかし、ボローチェックやミュータブルについてのルールが破られた場合、安全性は保証されません。もしコンパイラーが同数ほどそれらをみなすことができず、それが必要なときには、生のポインター（`*`）を使いましょう。

ベクターと文字列は基本的なメモリーレイアウトを共有していて、`vec`モジュールと`str`モジュールの中のユーティリティーはC APIで扱うために使うことができます。
ただし、文字列は`\0`で終わりません。
Cと相互利用するためにNUL終端の文字列が必要であれば、`std::ffi`モジュールの`CString`型を使う必要があります。

[crates.ioの`libc`クレート][libc]は`libc`モジュール内にCの標準ライブラリーの型の別名や関数の定義を含んでいて、Rustは`libc`と`libm`をデフォルトでリンクします。

[libc]: https://crates.io/crates/libc

# 「ヌルになり得るポインターの最適化」

いくつかの型は非`null`であると定義されています。
このようなものには、参照（`&T`、`&mut T`）、ボックス（`Box<T>`）、そして関数ポインター（`extern "abi" fn()`）があります。
Cとのインターフェイスにおいては、ヌルになり得るポインターが使われることがしばしばあります。
特別な場合として、ジェネリックな`enum`がちょうど2つのバリアントを持ち、そのうちの1つが値を持っていなくてもう1つが単一のフィールドを持っているとき、それは「ヌルになり得るポインターの最適化」の対象になります。
そのような列挙型が非ヌルの型でインスタンス化されたとき、それは単一のポインターとして表現され、データを持っていない方のバリアントはヌルポインターとして表現されます。
`Option<extern "C" fn(c_int) -> c_int>`は、C ABIで使われるヌルになり得る関数ポインターの表現方法の1つです。

# CからのRustのコードの呼出し

RustのコードをCから呼び出せる方法でコンパイルしたいときがあるかもしれません。
これは割と簡単ですが、いくつか必要なことがあります。

```rust
#[no_mangle]
pub extern fn hello_rust() -> *const u8 {
    "Hello, world!\0".as_ptr()
}
# fn main() {}
```

`extern`は先程「[他言語呼出規則](ffi.html#foreign-calling-conventions)」で議論したように、この関数をCの呼出規則に従うようにします。
`no_mangle`属性はRustによる名前のマングリングをオフにして、リンクしやすいようにします。

# FFIとパニック

FFIを扱うときに`panic!`に注意することは重要です。
FFIの境界をまたぐ`panic!`の動作は未定義です。
もしあなたがパニックし得るコードを書いているのであれば、他のスレッドで実行して、パニックがCに波及しないようにすべきです。

```rust
use std::thread;

#[no_mangle]
pub extern fn oh_no() -> i32 {
    let h = thread::spawn(|| {
        panic!("Oops!");
    });

    match h.join() {
        Ok(_) => 1,
        Err(_) => 0,
    }
}
# fn main() {}
```

# 不透明構造体の表現

ときどき、Cのライブラリーが何かのポインターを要求してくるにもかかわらず、その要求されているものの内部的な詳細を教えてくれないことがあります。
最も単純な方法は`void *`引数を使うことです。

```c
void foo(void *arg);
void bar(void *arg);
```

Rustではこれを`c_void`型で表現することができます。

```rust
# #![feature(libc)]
extern crate libc;

extern "C" {
    pub fn foo(arg: *mut libc::c_void);
    pub fn bar(arg: *mut libc::c_void);
}
# fn main() {}
```

これはこの状況に対処するための完全に正当な方法です。
しかし、もっとよい方法があります。
これを解決するために、いくつかのCライブラリーでは、代わりに`struct`を作っています。そこでは構造体の詳細とメモリーレイアウトはプライベートです。
これは型の安全性をいくらか満たします。
それらの構造体は「不透明」と呼ばれます。
Cによる例です。

```c
struct Foo; /* Foo is a structure, but its contents are not part of the public interface */
struct Bar;
void foo(struct Foo *arg);
void bar(struct Bar *arg);
```

これをRustで実現するために、`enum`で独自の不透明型を作りましょう。

```rust
pub enum Foo {}
pub enum Bar {}

extern "C" {
    pub fn foo(arg: *mut Foo);
    pub fn bar(arg: *mut Bar);
}
# fn main() {}
```

バリアントなしの`enum`を使って、バリアントがないためにインスタンス化できない不透明型を作ります。
しかし、`Foo`型と`Bar`型は異なる型であり、2つものの間の型の安全性を満たすので、`Foo`のポインターを間違って`bar()`に渡すことはなくなります。
