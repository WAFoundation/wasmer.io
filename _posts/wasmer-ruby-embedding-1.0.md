---
title: 'Wasmer Ruby embedding 1.0 take-off'
excerpt: 'Announcing the Wasmer Ruby embedding 1.0 availability!'
date: '2021-07-01T10:11:07.123Z'
author: Ivan Enderlin
---

We are thrilled to announce the release of [Wasmer Ruby embedding
1.0](https://github.com/wasmerio/wasmer-ruby).

About two years ago, we released `wasmer-ruby`, the Wasmer embedding
for Ruby. It allows Ruby developers to embed and natively reference
WebAssembly functions from their Ruby applications. Since its
inception, the embedding has gained hundreds if not thousands of users
and followers on Github and other forums.

Like its companions embeddings
[`wasmer-python`](/posts/wasmer-python-embedding-1.0) or
[`wasmer-go`](/posts/wasmer-go-embedding-1.0), a critical mass of
users, their use cases, and feedbacks have enabled us to deliver a 1.0
version of the product.

Numerous Github issues, feature requests, and contributions made it
clear that the Ruby community demands and deserves a Ruby embedding at
parity with our other projects. Support from the community enabled us
to rethink and rewrite the embedding from scratch. The rewritten
embedding is compatible with existing developer workflows and delivers
the best WebAssembly experience available in the market.

## Improved API

The new Ruby embedding improves the API and maintains idiomatic
semantics with the previous versions and other Wasmer
embeddings. Below is a block of code that outlines how to use the
embedding.

```ruby
require "wasmer"

# Create a new module from some WebAssembly in its text
# representation (for the sake of simplicity of the example).
wasm_bytes = Wasmer::wat2wasm(
  (<<~WAST)
  (module
    (type $add_one_t (func (param i32) (result i32)))
    (func $add_one_f (type $add_one_t) (param $value i32) (result i32)
      local.get $value
      i32.const 1
      i32.add)
    (export "add_one" (func $add_one_f)))
  WAST
)

# Create a store, that holds an engine, that holds a compiler.
store = Wasmer::Store.new

# Let's compile the Wasm module.
module_ = Wasmer::Module.new store, wasm_bytes

# Let's instantiate the module!
instance = Wasmer::Instance.new module_, nil

# We now have an instance ready to be used.
#
# From an `Instance` we can retrieve any exported entities.
#
# Here we are retrieving the exported function `add_one`,
# and calling it.
assert { instance.exports.add_one.(1) == 2 }
```

A few things to note:

* We have introduced a `Store` type. It holds an engine and a
  compiler. Similar to the previous version, the default engine is
  Universal, and the default compiler is Cranelift. However, we introduced
  significant performance improvements in the current release… more on
  that later. In future releases, developers will be able to change
  the default engine and compiler.
* There is a `wat2wasm` (and `wasm2wat`) function, which are handy!
* The `instance.exports` getter returns an object of type `Exports`:
  our standard API to retrieve exported “externals”, namely
  `Function`, `Global`, `Memory`, and `Table`. So yes, `wasmer-ruby`
  1.0 provides an API for `Global` and `Table` in addition to
  `Function` and `Memory`.

### Compiling Once

The compilation of the WebAssembly bytes to executable code happens
when calling `Module.new`. However, the previous version of the Ruby
embedding did not allow developers to serialize a module. The new
version eliminates complications. It's now possible to serialize the
module with `Module.serialize`, and deserialize it with
`Module.deserialize`.

### Symbol, Proc or Lambda as WebAssembly functions

Host functions are expressed outside of WebAssembly but passed to a
module as imports.  The new `wasmer-ruby` release enables the use of
`Symbol`s, `Proc`s or `Lambda`s as host functions.

Let's see an example that also illustrates the new `ImportObject` API
used to define the imports of an instance:

```ruby
# Let's define our WebAssembly module.
wasm_bytes = Wasmer::wat2wasm(
  (<<~WAST)
  (module
    (import "env" "sum" (func $sum (param i32 i32) (result i32)))
    (func (export "add_one") (param $x i32) (result i32)
      local.get $x
      i32.const 1
      call $sum))
  WAST
)

# Create a store.
store = Wasmer::Store.new

# Let's compile the Wasm module.
module_ = Wasmer::Module.new store, wasm_bytes

# Here we go.
#
# When creating an `Instance`, we can pass an `ImportObject`. All
# entities that must be imported are registered inside the
# `ImportObject`.
import_object = Wasmer::ImportObject.new

# Let's write the Ruby function that is going to be imported.
def sum(x, y)
  x + y
end

sum_host_function = Wasmer::Function.new(
  store,

  # The host function `sum` as a Ruby symbol.
  method(:sum),

  # The host function signature:
  #                         x                  y                    result
  Wasmer::FunctionType.new([Wasmer::Type::I32, Wasmer::Type::I32], [Wasmer::Type::I32])
)

# Now let's register the `sum` import inside the `env` namespace.
import_object.register(
  "env",
  {
    :sum => sum_host_function,
  }
)

# Let's instantiate the module!
instance = Wasmer::Instance.new module_, import_object

# And finally, call the `add_one` exported function!
assert { instance.exports.add_one.(41) == 42 }
```

Specifying the function type is mandatory because we cannot infer the
type of the function at runtime. Thus, it's up to the user to ensure
that the types match. Let's see how we could define
`sum_host_function` with `Proc`:

```ruby
sum_host_function = Wasmer::Function.new(
  …,
  # The host function `sum` as a Ruby proc.
  Proc.new { |x, y| x + y },
  …
)
```

Finally, as a lambda:

```ruby
sum_host_function = Wasmer::Function.new(
  …,
  # The host function `sum` as a Ruby lambda function.
  -> (x, y) { x + y },
  …
)
```

### Enumerable memories

The `Memory` type represents a memory (yes, we are good at
naming). It's a linear memory, that simply consists of a vector of
bytes, and an optional maximum size. A memory created by the host or
in WebAssembly code will be accessible and mutable from both host and
WebAssembly.

To make the life on the host side easier, we provide what we call
“views”. For example, the `Uint32View` is a view over the memory data
where elements are of kind `uint32`. We provide the following views:
`Uint32View`, `Int32View`, `Uint32View`, `Int32View`, `Uint32View`,
and `Int32View`. All these views include [the `Enumerable`
mixin](https://ruby-doc.org/core-3.0.1/Enumerable.html). So it's now
possible to do something like this:

```ruby
# Declare a WebAssembly module that:
#
# * exports a memory under the name `mem`,
# * exports a function under the name `hello`, which returns
#   a pointer to a string containing “Hello, World!”.
wasm_bytes = Wasmer::wat2wasm(
  (<<~WAST)
  (module
    (type $hello_t (func (result i32)))
    (func $hello (type $hello_t) (result i32)
      i32.const 42)
    (memory $memory 1)
    (export "hello" (func $hello))
    (export "mem" (memory $memory))
    (data (i32.const 42) "Hello, World!"))
  WAST
)

# Create a store, compile the module, instantiate the module.
store = Wasmer::Store.new
module_ = Wasmer::Module.new store, wasm_bytes
instance = Wasmer::Instance.new module_, nil

# OK, here go. First, let's call `hello`. It returns a
# pointer to the string in memory.
pointer = instance.exports.hello.()

# Since the pointer is a constant here, it's easy to assert
# its value.
assert { pointer == 42 }

# Now, let's read the string at position given by `pointer`.
memory = instance.exports.mem
reader = memory.uint8_view pointer
returned_string = reader.take(13).pack("U*")

assert { returned_string == 'Hello, World!' }
```

The instruction `reader.take(13)` is possible because all views are
enumerables now. Alternatively, we could have written
`reader.take_while { |n| n != 0 }` if the size of the string was
unknown, and so on.

## WASI

The `wasmer-ruby` gem now features
[WASI](https://github.com/WebAssembly/wasi) (The WebAssembly System
Interface) support with all the snapshot previews (that is, all the
versions).

First, the `Wasi::get_version` function can be used to know which
version of WASI a WebAssembly module is using or whether it's not
using WASI at all:

```ruby
module_ := Wasmer::Module.new store, wasm_bytes

puts Wasmer::Wasi::get_version module_, true
```

Second, to setup WASI, we start by creating a `Wasi::Environment` with
the help of the `Wasi::StateBuilder` API. We then use the generated
`Wasi::Environment` object to generate an `ImportObject`. This
contains all the imports that “bridge” the WebAssembly module to the
host to make WASI a reality. Of course, it's possible to use this
`ImportObject` to import your own host functions, memories etc., just
like with any other `ImportObject`.

Let's see this with an example. We want to execute [this Rust
program](https://github.com/wasmerio/wasmer-ruby/blob/c59fa32f128663a568b65ec029c53a88c902e53b/examples/appendices/wasi.rs),
that prints its arguments, its environment variables, and that lists
the contents of its current working directory.

```ruby
# Create a store and compile the module.
store = Wasmer::Store.new
module_ = Wasmer::Module.new store, wasm_bytes

# Here we go.
#
# First, let's extract the WASI version from the module.
# Why? Because WASI already exists in multiple versions,
# and it doesn't work the same way. So, to ensure
# compatibility, we need to know the version.
wasi_version = Wasmer::Wasi::get_version module_, true

# Second, create a `Wasi::Environment`. It contains
# everything related to WASI. To build such an
# environment, we must use the `Wasi::StateBuilder`.
#
# In this case, we specify the program name is
# `wasi_test_program`. We also specify the program is
# invoked with the `--test` argument, in addition to two
# environment variables: `COLOR` and `APP_SHOULD_LOG`.
# Finally, we map the `the_host_current_dir` to the
# current directory. There it is:
wasi_env =
  Wasmer::Wasi::StateBuilder.new('wasi_test_program')
    .argument('--test')
    .environment('COLOR', 'true')
    .environment('APP_SHOULD_LOG', 'false')
    .map_directory('the_host_current_dir', '.')
    .finalize

# From the WASI environment, we generate a custom import
# object. Why? Because WASI is, from the user perspective,
# a bunch of imports. Consequently `generate_import_object`…
# generates a pre-configured import object.
#
# Do you remember when we said WASI has multiple versions?
# Well, we need the WASI version here!
import_object = wasi_env.generate_import_object store, wasi_version

# Now we can instantiate the module.
instance = Wasmer::Instance.new module_, import_object
```

At this step, the WebAssembly module is ready to be executed. Which
exported function should be called from the instance to start the
program? It's `_start`:

```ruby
instance.exports._start.()
```

It prints:

```
Found program name: `wasi_test_program`
Found 1 arguments: --test
Found 2 environment variables: COLOR=true, APP_SHOULD_LOG=false
Found 1 preopened directories: DirEntry("/the_host_current_dir")
```

on the standard output.

## Faster

`wasmer-ruby` 1.0 is built on the shoulders of [Wasmer
2.0](./wasmer-2.0), which provides faster compilation-time, faster
execution-times, and crazy hot startup performance! Prior to this new
version, the Ruby embedding was relying on Wasmer 0.x. The Wasmer 1.0
version has bumped performances greatly, but the 2.0 version is
improving the performance even further! You will easily see an
improvement up to 3.5x faster for the compilation-time, and probably
4x faster for the execution-time.

## Better and richer documentation

We hacked quite a lot to provide you a [better and richer
documentation
experience](https://wasmerio.github.io/wasmer-ruby/wasmer_ruby/Wasmer/index.html). The
documentation is now generated by
[`rustdoc`](https://doc.rust-lang.org/rustdoc/index.html), which is
the standard program used to generate Rust documentation. Why doing
that? Because we have not found a way to document a Ruby library based
on a native extension.

Indeed, `wasmer-ruby` is written in Rust, but it exposes Ruby examples
inside the documentation. Thanks to a couple of hacks, we have been
able to generate a documentation that doesn't look like a Rust
documentation, but a real Ruby documentation, with Ruby code
everywhere, and a search engine, and automatic cross-links etc.

### Documentation-based testing

Cherry on the cake: All Ruby examples can be run as tests! Indeed,
`rustdoc` allows to extract code from code examples, and can run them
if they are written in Rust (that's the so-called “doctests”, or more
formely documentation-based testing). Thanks to a couple of other
hacks, we can boot a Ruby VM, load `wasmer-ruby`, and execute the Ruby
examples as if they were tests. That way, we ensure that all examples
present in the documentation are up-to-date and working!

### More examples

Apart from examples in the documentation, we provide a [collection of
examples](https://github.com/wasmerio/wasmer-ruby/tree/9f1e527212c823f95ea832d36fb211c88a74da25/examples). Each
example is extensively commented to provide as much information as
possible. We believe such a collection of examples is a nice way to
learn faster about WebAssembly, and `wasmer-ruby`. For example: How to
write a host function that can fail? [Check the
`imports_function_early_exit`
example](https://github.com/wasmerio/wasmer-ruby/blob/9f1e527212c823f95ea832d36fb211c88a74da25/examples/imports_function_early_exit.rb).

If an example is missing, feel free to reach out, we will be happy to
write more examples!

## More Ruby versions supported

Previously, we were supporting only Ruby 2.6. With `wasmer-ruby` 1.0,
we now support:

* Ruby 2.6,
* Ruby 2.7, and
* Ruby 3.0.

More platforms are also supported with Linux and macOS.

## Contributions to the Ruby and Rust ecosystems

This part is technical. We use [the Rust `rutie`
library](https://github.com/danielpclark/rutie) to write the Ruby
extension in Rust. It is a really important library to bridge the Ruby
and the Rust ecosystems together.

In order to create a proper ABI from Rust to Ruby, `rutie` provides
macros, what they call a “DSL” (Domain Specific Language). To make the
DSL more Rust idiomatic, we have written 2 new crates:
[`rutie-derive`](https://github.com/wasmerio/wasmer-ruby/tree/9f1e527212c823f95ea832d36fb211c88a74da25/crates/rutie-derive)
and `rutie-derive-macros`. They aren't published yet, it's still beta
code, but it's promising. [Daniel
P. Clark](https://github.com/danielpclark/), the author of `rutie`, is
[opened to integrate them inside the `rutie`
repository](https://github.com/danielpclark/rutie/issues/145), which
is exciting!

## Our Commitment to Open Source and the Ruby Community

As WebAssembly continues to grow and thrive, we plan to double down on
our commitment to the Ruby community and our mission to make
WebAssembly universally available. We plan to implement a more open
and transparent development process for future release of
`wasmer-ruby` on Github with milestones. The are now multiple ways the
community can actively help with our mission:

1. Keep downloading and using `wasmer-ruby`,
2. Provide as much feedback as possible to help make future release
   better or contribute directly with pull requests,
3. Become a sponsor and help fund development by donating to [our Open
   Collective page](https://opencollective.com/wasmerio).

## Conclusion

The 1.0 version is more than performance improvements: it provides a
stable and powerful API that fulfills more people's needs. We believe
that the new API design is a great improvement that provides more
power and flexibility than ever before.

Documentation and examples have been meticulously written to help
users new to WebAssembly, as well as advanced users. We believe it
will facilitate further usage of WebAssembly in the Ruby ecosystem.

[Join a community of Ruby and WebAssembly passionate
developers!](https://github.com/wasmerio/wasmer-ruby)

## About Wasmer

Headquartered in San Francisco, CA, Wasmer Inc. is behind the popular
open-source WebAssembly runtime Wasmer. In addition to the Wasmer
runtime, the company has made significant investments in
[WAPM](https://wapm.io/), the WebAssembly Package Manager, and many
other open-source projects in the WebAssembly ecosystem.

**Our mission is to make software universally available**. We are
committed to the open-source community and strive to contribute to
developers and companies worldwide to help make Wasmer and WebAssembly
a universal standard.
