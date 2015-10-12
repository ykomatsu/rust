% 条件付きコンパイル

Rustには`#[cfg]`という特別な属性があります。これによって、あなたはコンパイラーに渡されるフラグに基づいてコードをコンパイルできるようになります。
それには2つの形式があります。

```rust
#[cfg(foo)]
# fn foo() {}

#[cfg(bar = "baz")]
# fn bar() {}
```

それらにはいくつかのヘルパーもあります。

```rust
#[cfg(any(unix, windows))]
# fn foo() {}

#[cfg(all(unix, target_pointer_width = "32"))]
# fn bar() {}

#[cfg(not(foo))]
# fn not_foo() {}
```

それらは好きなようにネストさせることもできます。

```rust
#[cfg(any(not(unix), all(target_os="macos", target_arch = "powerpc")))]
# fn foo() {}
```

どのようにしてそれらのスイッチの有効、無効を切り替えるのかというと、もしあなたがCargoを使っているのであれば、`Cargo.toml`の[`[features]`セクション][features]内でセットします。

[features]: http://doc.crates.io/manifest.html#the-features-section

```toml
[features]
# no features by default
default = []

# The “secure-password” feature depends on the bcrypt package.
secure-password = ["bcrypt"]
```

あなたがこうすると、Cargoは`rustc`にフラグを伝えます。

```text
--cfg feature="${feature_name}"
```

それらの`cfg`フラグを総合して、どれを有効にするか、そしてどのコードをコンパイルするかを決定します。
このコードを見ましょう。

```rust
#[cfg(feature = "foo")]
mod foo {
}
```

もし私たちがこれを`cargo build --features "foo"`でコンパイルすれば、`--cfg feature="foo"`フラグが`rustc`に送られます。そして、その出力には`mod foo`が含まれます。
もし私たちが通常の`cargo build`でコンパイルすれば、追加のフラグは送られないので、`foo`モジュールは生成されません。

# cfg_attr

あなたは`cfg_attr`付きの`cfg`変数に基づいて、別の属性をセットすることもできます。

```rust
#[cfg_attr(a, b)]
# fn foo() {}
```

もし`a`が`cfg`属性によってセットされていれば、これは`#[b]`と同じことになり、セットされていなければ、何もないのと同じことになります。

# cfg!

`cfg!`[構文拡張][compilerplugins]によって、あなたはそれらの種類のフラグをコードの他の場所でも使うことができるようになります。

```rust
if cfg!(target_os = "macos") || cfg!(target_os = "ios") {
    println!("Think Different!");
}
```

[compilerplugins]: compiler-plugins.html

設定によって、それらはコンパイル時に`true`か`false`に置き換えられます。
