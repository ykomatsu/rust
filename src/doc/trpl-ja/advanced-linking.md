% 一歩進んだリンク

Rustにおけるリンクの一般的なケースについては本書の前の方で説明しましたが、他言語から利用できるような幅広いリンクをサポートすることは、ネイティブライブラリーとのシームレスな相互利用を実現するために、Rustにとって重要です。

# リンク引数

`rustc`にカスタマイズされたリンクの仕方を教える方法の1つは、`link_args`属性を使うことです。
この属性は`extern`ブロックに適用され、生成物を作るときにリンカーに渡したいフラグをそのまま指定します。
使い方の例は次のようになります。

``` no_run
#![feature(link_args)]

#[link_args = "-foo -bar -baz"]
extern {}
# fn main() {}
```

これは実行するための認められた方法ではないため、この機能は現在`feature(link_args)`ゲートによって隠されているということに注意しましょう。
今は`rustc`がシステムリンカー（多くのシステムでは`gcc`、MSVCでは`link.exe`）に渡すので、追加のコマンドライン引数を提供することには意味がありますが、それが今後もそうだとは限りません。
将来、`rustc`がネイティブライブラリーをリンクするためにLLVMを直接使うようになるかもしれませんし、そのような場合には`link_args`は意味がなくなるでしょう。
`rustc`に`-C link-args`引数を付けることで、`link_args`属性と同じような効果を得ることができます。

この属性は使わ *ない* ことが強く推奨されているので、代わりにもっと正式な`#[link(...)]`属性を`extern`ブロックに使いましょう。

# スタティックリンク

スタティックリンクとは全ての必要なライブラリーを含めた出力を生成する手順のことで、そうすればコンパイルされたプロジェクトを使いたいシステム全てにライブラリーをインストールする必要がなくなります。
Rustのみで構築された依存関係はデフォルトでスタティックリンクされます。そのため、Rustをインストールしなくても、作成されたバイナリーやライブラリーを使うことができます。
対照的に、ネイティブライブラリー（例えば`libc`や`libm`）はダイナミックリンクされるのが普通です。しかし、これを変更してそれらを同様にスタティックリンクすることも可能です。

リンクは非常にプラットフォームに依存した話題であり、スタティックリンクのできないプラットフォームすらあるかもしれません！
このセクションは選んだプラットフォームにおけるリンクについての基本が理解できていることを前提とします。

## Linux

デフォルトでは、Linux上の全てのRustのプログラムはシステムの`libc`とその他のいくつかのライブラリーとリンクされます。
GCCと`glibc`（Linuxにおける最も一般的な`libc`）を使った64ビットLinuxマシンでの例を見てみましょう。

``` text
$ cat example.rs
fn main() {}
$ rustc example.rs
$ ldd example
        linux-vdso.so.1 =>  (0x00007ffd565fd000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fa81889c000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fa81867e000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007fa818475000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fa81825f000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fa817e9a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fa818cf9000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fa817b93000)
```

古いシステムで新しいライブラリーの機能を使いたいときや、実行するプログラムに必要な依存関係を満たさないシステムをターゲットにしたいときは、Linuxにおけるダイナミックリンクは望ましくないかもしれません。

スタティックリンクは代わりの`libc`である[`musl`](http://www.musl-libc.org/)によってサポートされています。
以下の手順に従い、`musl`を有効にした独自バージョンのRustをコンパイルして独自のディレクトリーにインストールすることができます。

```text
$ mkdir musldist
$ PREFIX=$(pwd)/musldist
$
$ # Build musl
$ curl -O http://www.musl-libc.org/releases/musl-1.1.10.tar.gz
$ tar xf musl-1.1.10.tar.gz
$ cd musl-1.1.10/
musl-1.1.10 $ ./configure --disable-shared --prefix=$PREFIX
musl-1.1.10 $ make
musl-1.1.10 $ make install
musl-1.1.10 $ cd ..
$ du -h musldist/lib/libc.a
2.2M    musldist/lib/libc.a
$
$ # Build libunwind.a
$ curl -O http://llvm.org/releases/3.7.0/llvm-3.7.0.src.tar.xz
$ tar xf llvm-3.7.0.src.tar.xz
$ cd llvm-3.7.0.src/projects/
llvm-3.7.0.src/projects $ curl http://llvm.org/releases/3.7.0/libunwind-3.7.0.src.tar.xz | tar xJf -
llvm-3.7.0.src/projects $ mv libunwind-3.7.0.src libunwind
llvm-3.7.0.src/projects $ mkdir libunwind/build
llvm-3.7.0.src/projects $ cd libunwind/build
llvm-3.7.0.src/projects/libunwind/build $ cmake -DLLVM_PATH=../../.. -DLIBUNWIND_ENABLE_SHARED=0 ..
llvm-3.7.0.src/projects/libunwind/build $ make
llvm-3.7.0.src/projects/libunwind/build $ cp lib/libunwind.a $PREFIX/lib/
llvm-3.7.0.src/projects/libunwind/build $ cd ../../../../
$ du -h musldist/lib/libunwind.a
164K    musldist/lib/libunwind.a
$
$ # Build musl-enabled rust
$ git clone https://github.com/rust-lang/rust.git muslrust
$ cd muslrust
muslrust $ ./configure --target=x86_64-unknown-linux-musl --musl-root=$PREFIX --prefix=$PREFIX
muslrust $ make
muslrust $ make install
muslrust $ cd ..
$ du -h musldist/bin/rustc
12K     musldist/bin/rustc
```

これで`musl`が有効になったRustのビルドが手に入りました！
独自のプレフィックスを付けてインストールしたので、試しに実行するときにはシステムがバイナリーと適切なライブラリーを見付けられることを確かめなければなりません。

```text
$ export PATH=$PREFIX/bin:$PATH
$ export LD_LIBRARY_PATH=$PREFIX/lib:$LD_LIBRARY_PATH
```

試してみましょう！

```text
$ echo 'fn main() { println!("hi!"); panic!("failed"); }' > example.rs
$ rustc --target=x86_64-unknown-linux-musl example.rs
$ ldd example
        not a dynamic executable
$ ./example
hi!
thread '<main>' panicked at 'failed', example.rs:1
```

成功しました！
このバイナリーは同じマシンアーキテクチャーであればほとんど全てのLinuxマシンにコピーして問題なく実行することができます。

`cargo build`も`--target`オプションを受け付けるので、あなたのクレートも普通にビルドできるはずです。
ただし、リンクする前にネイティブライブラリーを`musl`向けにリコンパイルする必要はあるかもしれません。
