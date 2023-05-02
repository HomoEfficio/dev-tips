# Rust feature may not be used on the release channel error

`cargo run`을 하면 대충 아래처럼 생긴 컴파일 에러를 만날 때가 있는데,

```
error[E0554]: `#![feature]` may not be used on the stable release channel
   --> /Users/user/.cargo/registry/src/github.com-1ecc6299db9ec823/anyhow-1.0.68/src/lib.rs:214:32
    |
214 | #![cfg_attr(backtrace, feature(error_generic_member_access, provide_any))]
    |                                ^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0554]: `#![feature]` may not be used on the stable release channel
   --> /Users/user/.cargo/registry/src/github.com-1ecc6299db9ec823/anyhow-1.0.68/src/lib.rs:214:61
    |
214 | #![cfg_attr(backtrace, feature(error_generic_member_access, provide_any))]
    |                                                             ^^^^^^^^^^^

```

검색해보면 대부분 아래와 같이 nightly를 사용하라고 나온다. https://jonathanmh.com/rust-nightly-stable-rustup-may-not-used-error/

```
rustup install nightly
cargo +nightly build
cargo run
```

그런데 그보다는 다음과 같이 `cargo check`를 하는 게 나아 보인다. https://stackoverflow.com/questions/74524817/why-is-anyhow-not-working-in-the-stable-version

```
rm -rf target
cargo check
cargo run
```

