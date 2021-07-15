---
title: How do we Test and Document our C API?
excerpt: 'Wasmer has designed a unique solution to test and document its own C API.'
date: '2021-07-06T10:32:01.123Z'
author: Ivan Enderlin
published: true
---

Wasmer [server-side WebAssembly runtime](https://github.com/wasmerio/wasmer) comes in
multiple flavors. Written in Rust, it is embedded in different
languages, such as C, Python, Go, Ruby, PHP, Java, and many more
([discover all the
integrations](https://github.com/wasmerio/wasmer#-language-integrations)).

As many of our embeddings, the C embedding is implemented in Rust
itself. Rust is perfectly capable of exposing a C compatible ABI
(learn more by reading [The Rustnomicon Book, the Foreign Function
Interface
Chapter](https://doc.rust-lang.org/nomicon/ffi.html)). Moreover, Rust
comes with a first-class tooling for testing and documenting
projects. We wanted to leverage those tools to:

1. Test with `cargo test --lib` and with `cargo test --doc`,
2. Document with `cargo doc`.

Why? Because they are simple, they work everywhere, and every Rust
programmer knows them. They are part of the foundation of the
Rust ecosystem.

## Testing: let's write C directly in Rust

If you are a Rust developer, you are familiar with the `#[test]`
attribute that annotates a function that acts as a test:

```rust
#[test]
fn my_test() {
    let a = 3;
    let b = 1 + 2;
    assert_eq!(a, b);
}
```

To run that test, one simply runs `cargo test --lib`.

If you have ever written tests in C you know the struggle. Either with `CMake`,
or `Makefile`, you name it.

But what if we could write Rust code to test our C API?
Something like:

```rust
#[test]
fn my_test() {
    #include <assert.h>
    
    int a = 3;
    int b = 1 + 2;
    
    assert(a == b);
}
```

Well, first, thanks for asking dear reader! And second, what an
excellent idea! Good news: With a little modification, that's
possible. We have written [the `inline-c` Rust
crate](https://github.com/Hywan/inline-c-rs/) to fulfill this precise
purpose. Add `inline-c = "0.1"` to your `[dev-dependencies]` in your
`Cargo.toml` file, and let's see it in action:

```rust
use inline_c::assert_c;

#[test]
fn my_test() {
    (assert_c! {
        int main() {
            int a = 3;
            int b = 1 + 2;

            assert(a == b);

            return 0;
        }
    })
    .success();
}
```

Close enough! You will notice that we have assertions in C (from
[`assert.h`](https://www.freebsd.org/cgi/man.cgi?query=assert)). Running
`cargo test --lib` will:

1. Compile the C program,
2. Run it,
3. Report the outputs as tests.

It's a little bit more verbose because we have to write `int main() {
… }` etc. That's on purpose, because we might want to add some
`#include`, or more C code around it. It is also possible to assert on
the results of the C program. Let's see another example of how to add
assertions on `stdout` for example:

```rust
#[test]
fn my_test() {
    (assert_c! {
        #include <stdio.h>

        int main() {
            printf("Hello, World!");

            return 0;
        }
    })
    .success()
    .stdout("Hello, World!");
}
```

In this case, we include `stdio.h` to get `printf`. We also see that
assertions are on the Rust side, not in the C side.

### Real-Life Example

So, this solution is pretty flexible. Compiling C on multiple
platforms can be very challenging sometimes. `inline-c` is using [the
excellent `cc` Rust crate](https://github.com/alexcrichton/cc-rs/)
behind the scene to support as much platforms as possible, with a
little bit of black-magic.

So far, we were testing C code not implemented in Rust. Let's dive
into a tiny real-life example.

```
$ cargo new --lib hello
$ cd hello
```

Let's add the following lines to the `Cargo.toml` file:

```toml
[lib]
crate-type = ["cdylib"]

[dev-dependencies]
inline-c = "0.1"

[build-dependencies]
cbindgen = "0.19"
```

Let's edit the `src/lib.rs` file to:

1. Add a `hello` function that is C FFI compatible. This function
   takes a `who` argument and returns a full string that basically
   contains `Hello, {who}!`.
2. Add a C test that… tests our `hello` function.

We keep the code as simple and short as possible for the sake of
readability. We all know that we must check for `NULL` pointers etc. C
is a dangerous place.

```rust
use std::ffi::{CStr, CString};
use std::mem;
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn hello(who: *const c_char) -> *const c_char {
    let hello = CString::new(format!(
        "Hello, {}!",
        unsafe { CStr::from_ptr(who) }.to_string_lossy()
    ))
    .unwrap();
    let ptr = hello.as_ptr();

    mem::forget(hello);

    ptr
}

#[cfg(test)]
mod tests {
    use inline_c::assert_c;

    #[test]
    fn it_works() {
        (assert_c! {
            #include "hello.h"
            #include <stdio.h>

            int main() {
                const char* name = "Gordon";
                const char* full = hello(name);

                printf("> %s", full);

                free((void*) full);
            }
        })
        .success()
        .stdout("> Hello, Gordon!");
    }
}
```

Does it compile?

```
$ cargo build --release
$ nm target/release/libhello.dylib | rg ' T '
0000000000003754 T _hello
000000000001f0b4 T _rust_eh_personality
```

Great, the `hello` function is present in our shared object.

OK, let's try to get this working!

1. First step, we need to generate the `hello.h` file. Obviously, we
   won't write this file by hand. Hopefully for us, there is [the
   `cbindgen` project](https://github.com/eqrion/cbindgen/) that
   exists exactly for that:

   > cbindgen creates C/C++11 headers for Rust libraries which expose a
   > public C API.

2. Second step, we need to configure `inline-c` to correctly set the
   `CFLAGS` and the `LDFLAGS` so that everything links
   properly. Bonus, we will also set `_DEBUG` within `CFLAGS` to
   enable `assert.h` on all platforms.

To achieve this, let's edit the `build.rs` file:

```rust
use std::env;
use std::path::PathBuf;

fn main() {
    let manifest_dir = env::var("CARGO_MANIFEST_DIR").unwrap();
    
    // Step 1, let's generate the `hello.h` file automatically.
    cbindgen::Builder::new()
        .with_crate(&manifest_dir)
        .with_language(cbindgen::Language::C)
        .generate()
        .unwrap()
        .write_to_file("hello.h");

    // Step 2, let's set the `CFLAGS` and the `LDFLAGS` variables.
    let include_dir = manifest_dir.clone();
    let mut shared_object_dir = PathBuf::from(manifest_dir);
    shared_object_dir.push("target");
    shared_object_dir.push(env::var("PROFILE").unwrap());
    let shared_object_dir = shared_object_dir.as_path().to_string_lossy();

    println!(
        "cargo:rustc-env=INLINE_C_RS_CFLAGS=-I{I} -L{L} -D_DEBUG -D_CRT_SECURE_NO_WARNINGS",
        I = include_dir,
        L = shared_object_dir,
    );

    println!(
        "cargo:rustc-env=INLINE_C_RS_LDFLAGS={shared_object_dir}/{lib}",
        shared_object_dir = shared_object_dir,
        lib = "libhello.dylib",
    );
}
```

Is that it? Let's try:

```
$ cargo test --release --lib
[…]
running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.49s
```

Success 🎉! We have written a `hello` function in Rust, that can be
used in a C program. This `hello` function is tested with a C program
written in a Rust test, and compiled and executed with `cargo`.

There is more work in the `build.rs` file to make it compatible with
Linux, Windows and other platforms. We kept it short and simple for
the sake of readability (it's a matter of changing `libhello.dylib` to
`libhello.so` on Linux, and to `hello.dll` on Windows).

If you want to avoid configuring `inline-c` and `cbindgen`, you may
want to take a look at [the `cargo-c`
project](https://github.com/lu-zero/cargo-c). It does a lot of things
automatically for you, in addition to support `inline-c`:

> cargo applet to build and install C-ABI compatible dynamic and static libraries.
>
> It produces and installs a correct
> [`pkg-config`](https://www.freedesktop.org/wiki/Software/pkg-config/)
> file, a static library and a dynamic library, and a C header to be
> used by any C (and C-compatible) software.

It's really a useful project.

## Documenting: C examples that are tested

Let's push the concept further. So far, we are able to write C code
inside Rust. OK, and let's not forget that Rust has great comments
where blocks of code can be run as tests, like:

```rust
/// The `hello` function is very polite.
///
/// ```rust
/// # use hello::hello;
/// # fn main() {
/// hello("Gordon");
/// # }
/// ```
pub fn hello(who: &str) -> String {
    // …
}
```

Did you notice the `#` in front of some lines? It's a `rustdoc`'s
trick that makes it is possible to [hide portions of the
code](https://doc.rust-lang.org/rustdoc/documentation-tests.html#hiding-portions-of-the-example)
if they are prefixed by `#` + a space.

We can run it with `cargo test --doc`. And we can generate the
documentation with `cargo doc`. That's really super classic for a Rust
developer.

But what if we… write… and test… C code… with `cargo test` and `cargo
doc`? I really love your way of thinking dear reader! Let's try that!

```rust
/// The `hello` function is very polite.
///
/// ```rust
/// # fn main() {
/// # (inline_c::assert_c! {
/// #include "hello.h"
/// #include <stdio.h>
///
/// int main() {
///     const char* name = "Gordon";
///     const char* full = hello(name);
///
///     printf("> %s", full);
///
///     free((void*) full);
/// }
/// # })
/// # .success()
/// # .stdout("> Hello, Gordon!");
/// # }
/// ```
#[no_mangle]
pub extern "C" fn hello(who: *const c_char) -> *const c_char {
    // …
}
```

OK. Let's test it. Are you anxious?

```
$ cargo test --doc
[…]
running 1 test
test src/lib.rs - hello (line 7) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.96s
```

Of course it works. Never had a doubt about it. And let's try to
generate the documentation:

```
$ cargo doc --open
```

Here is the result:

![C code in Rust documentation](/images/blog/how-do-we-test-and-document-our-c-api/hello.png)

Fantastic! Our example in the Rust documentation is a C program, that
is actually tested. We are sure the documentation provides correct
examples to our awesome users.

## Conclusion

And that, dear reader, is <del>how I met your…</del> how the `wasmer_c_api`
crate is tested and documented. No need to say we have other kind of
tests, but our unit tests and documentation tests are powered by
`inline-c`. The documentation is published online at
[docs.rs/wasmer-c-api](https://docs.rs/wasmer-c-api). Check for
example [the `instance` “Rust
module”](https://docs.rs/wasmer-c-api/2.0.0/wasmer_c_api/wasm_c_api/instance/index.html)
which contains all the functions related to the `wasm_instance_t`
type.

Not only this technique is powerful because it relies on the great
Rust tooling and all its features (like intra-links in the
documentation, testing etc.), but it's “free” because we don't need
extra tools.

The astute reader you're have noticed that the syntax color
highlighting is incorrect. It's the scheme for Rust, instead of being
the scheme for C. There is [an issue opened for that in the
`rust-lang/rust` repository
(#78917)](https://github.com/rust-lang/rust/issues/78917). It will be
useful to other tricks like this.

At the time of writing, we have 23 unit tests, and 61 documentation
tests, powered by `inline-c`, in addition to more test suites. We had
a lot of fun developing this, and we believe it's pretty useful. It
allows to iterate very quickly over the implementation and the tests,
while keeping the same tooling. It provides consistency and comfort,
whcih is always welcomed when dealing with a C API, in addition to
provide better examples and better documentation to the users.
