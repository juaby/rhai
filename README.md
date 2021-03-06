Rhai - Embedded Scripting for Rust
=================================

![GitHub last commit](https://img.shields.io/github/last-commit/jonathandturner/rhai)
[![Travis (.org)](https://img.shields.io/travis/jonathandturner/rhai)](http://travis-ci.org/jonathandturner/rhai)
[![license](https://img.shields.io/github/license/jonathandturner/rhai)](https://github.com/license/jonathandturner/rhai)
[![crates.io](https://img.shields.io/crates/v/rhai.svg)](https::/crates.io/crates/rhai/)
![crates.io](https://img.shields.io/crates/d/rhai)
[![API Docs](https://docs.rs/rhai/badge.svg)](https://docs.rs/rhai/)

Rhai is an embedded scripting language and evaluation engine for Rust that gives a safe and easy way
to add scripting to any application.

Rhai's current features set:

* Easy-to-use language similar to JS+Rust
* Easy integration with Rust [native functions](#working-with-functions) and [types](#custom-types-and-methods),
  including [getters/setters](#getters-and-setters), [methods](#members-and-methods) and [indexers](#indexers)
* Easily [call a script-defined function](#calling-rhai-functions-from-rust) from Rust
* Freely pass variables/constants into a script via an external [`Scope`]
* Fairly efficient (1 million iterations in 0.75 sec on my 5 year old laptop)
* Low compile-time overhead (~0.6 sec debug/~3 sec release for script runner app)
* [`no-std`](#optional-features) support
* Support for [function overloading](#function-overloading)
* Support for [operator overloading](#operator-overloading)
* Support for loading external [modules]
* Compiled script is [optimized](#script-optimization) for repeat evaluations
* Support for [minimal builds](#minimal-builds) by excluding unneeded language [features](#optional-features)
* Very few additional dependencies (right now only [`num-traits`](https://crates.io/crates/num-traits/)
  to do checked arithmetic operations); for [`no-std`](#optional-features) builds, a number of additional dependencies are
  pulled in to provide for functionalities that used to be in `std`.

**Note:** Currently, the version is 0.14.1, so the language and API's may change before they stabilize.

Installation
------------

Install the Rhai crate by adding this line to `dependencies`:

```toml
[dependencies]
rhai = "0.14.1"
```

Use the latest released crate version on [`crates.io`](https::/crates.io/crates/rhai/):

```toml
[dependencies]
rhai = "*"
```

Crate versions are released on [`crates.io`](https::/crates.io/crates/rhai/) infrequently, so if you want to track the
latest features, enhancements and bug fixes, pull directly from GitHub:

```toml
[dependencies]
rhai = { git = "https://github.com/jonathandturner/rhai" }
```

Beware that in order to use pre-releases (e.g. alpha and beta), the exact version must be specified in the `Cargo.toml`.

Optional features
-----------------

| Feature       | Description                                                                                                                           |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `unchecked`   | Exclude arithmetic checking (such as overflows and division by zero). Beware that a bad script may panic the entire system!           |
| `no_function` | Disable script-defined functions if not needed.                                                                                       |
| `no_index`    | Disable [arrays] and indexing features if not needed.                                                                                 |
| `no_object`   | Disable support for custom types and objects.                                                                                         |
| `no_float`    | Disable floating-point numbers and math if not needed.                                                                                |
| `no_optimize` | Disable the script optimizer.                                                                                                         |
| `no_module`   | Disable modules.                                                                                                                      |
| `only_i32`    | Set the system integer type to `i32` and disable all other integer types. `INT` is set to `i32`.                                      |
| `only_i64`    | Set the system integer type to `i64` and disable all other integer types. `INT` is set to `i64`.                                      |
| `no_std`      | Build for `no-std`. Notice that additional dependencies will be pulled in to replace `std` features.                                  |
| `sync`        | Restrict all values types to those that are `Send + Sync`. Under this feature, [`Engine`], [`Scope`] and `AST` are all `Send + Sync`. |

By default, Rhai includes all the standard functionalities in a small, tight package.
Most features are here to opt-**out** of certain functionalities that are not needed.
Excluding unneeded functionalities can result in smaller, faster builds as well as less bugs due to a more restricted language.

[`unchecked`]: #optional-features
[`no_index`]: #optional-features
[`no_float`]: #optional-features
[`no_function`]: #optional-features
[`no_object`]: #optional-features
[`no_optimize`]: #optional-features
[`no_module`]: #optional-features
[`only_i32`]: #optional-features
[`only_i64`]: #optional-features
[`no_std`]: #optional-features
[`sync`]: #optional-features

### Performance builds

Some features are for performance.  For example, using `only_i32` or `only_i64` disables all other integer types (such as `u16`).
If only a single integer type is needed in scripts - most of the time this is the case - it is best to avoid registering
lots of functions related to other integer types that will never be used.  As a result, performance will improve.

If only 32-bit integers are needed - again, most of the time this is the case - using `only_i32` disables also `i64`.
On 64-bit targets this may not gain much, but on some 32-bit targets this improves performance due to 64-bit arithmetic
requiring more CPU cycles to complete.

Also, turning on `no_float`, and `only_i32` makes the key [`Dynamic`] data type only 8 bytes small on 32-bit targets
while normally it can be up to 16 bytes (e.g. on x86/x64 CPU's) in order to hold an `i64` or `f64`.
Making [`Dynamic`] small helps performance due to more caching efficiency.

### Minimal builds

In order to compile a _minimal_build - i.e. a build optimized for size - perhaps for embedded targets, it is essential that
the correct linker flags are used in `cargo.toml`:

```toml
[profile.release]
opt-level = "z"     # optimize for size
```

Opt out of as many features as possible, if they are not needed, to reduce code size because, remember, by default
all code is compiled in as what a script requires cannot be predicted. If a language feature is not needed,
omitting them via special features is a prudent strategy to optimize the build for size.

Omitting arrays (`no_index`) yields the most code-size savings, followed by floating-point support
(`no_float`), checked arithmetic (`unchecked`) and finally object maps and custom types (`no_object`).
Disable script-defined functions (`no_function`) only when the feature is not needed because code size savings is minimal.

[`Engine::new_raw`](#raw-engine) creates a _raw_ engine which does not register _any_ utility functions.
This makes the scripting language quite useless as even basic arithmetic operators are not supported.
Selectively include the necessary operators by loading specific [packages](#packages) while minimizing the code footprint.

Related
-------

Other cool projects to check out:

* [ChaiScript](http://chaiscript.com/) - A strong inspiration for Rhai.  An embedded scripting language for C++ that I helped created many moons ago, now being led by my cousin.
* Check out the list of [scripting languages for Rust](https://github.com/rust-unofficial/awesome-rust#scripting) on [awesome-rust](https://github.com/rust-unofficial/awesome-rust)

Examples
--------

A number of examples can be found in the `examples` folder:

| Example                                                            | Description                                                                 |
| ------------------------------------------------------------------ | --------------------------------------------------------------------------- |
| [`arrays_and_structs`](examples/arrays_and_structs.rs)             | demonstrates registering a new type to Rhai and the usage of [arrays] on it |
| [`custom_types_and_methods`](examples/custom_types_and_methods.rs) | shows how to register a type and methods for it                             |
| [`hello`](examples/hello.rs)                                       | simple example that evaluates an expression and prints the result           |
| [`no_std`](examples/no_std.rs)                                     | example to test out `no-std` builds                                         |
| [`reuse_scope`](examples/reuse_scope.rs)                           | evaluates two pieces of code in separate runs, but using a common [`Scope`] |
| [`rhai_runner`](examples/rhai_runner.rs)                           | runs each filename passed to it as a Rhai script                            |
| [`simple_fn`](examples/simple_fn.rs)                               | shows how to register a Rust function to a Rhai [`Engine`]                  |
| [`repl`](examples/repl.rs)                                         | a simple REPL, interactively evaluate statements from stdin                 |

Examples can be run with the following command:

```bash
cargo run --example name
```

The `repl` example is a particularly good one as it allows you to interactively try out Rhai's
language features in a standard REPL (**R**ead-**E**val-**P**rint **L**oop).

Example Scripts
---------------

There are also a number of examples scripts that showcase Rhai's features, all in the `scripts` folder:

| Language feature scripts                             | Description                                                   |
| ---------------------------------------------------- | ------------------------------------------------------------- |
| [`array.rhai`](scripts/array.rhai)                   | [arrays] in Rhai                                              |
| [`assignment.rhai`](scripts/assignment.rhai)         | variable declarations                                         |
| [`comments.rhai`](scripts/comments.rhai)             | just comments                                                 |
| [`for1.rhai`](scripts/for1.rhai)                     | for loops                                                     |
| [`function_decl1.rhai`](scripts/function_decl1.rhai) | a function without parameters                                 |
| [`function_decl2.rhai`](scripts/function_decl2.rhai) | a function with two parameters                                |
| [`function_decl3.rhai`](scripts/function_decl3.rhai) | a function with many parameters                               |
| [`if1.rhai`](scripts/if1.rhai)                       | if example                                                    |
| [`loop.rhai`](scripts/loop.rhai)                     | endless loop in Rhai, this example emulates a do..while cycle |
| [`op1.rhai`](scripts/op1.rhai)                       | just a simple addition                                        |
| [`op2.rhai`](scripts/op2.rhai)                       | simple addition and multiplication                            |
| [`op3.rhai`](scripts/op3.rhai)                       | change evaluation order with parenthesis                      |
| [`string.rhai`](scripts/string.rhai)                 | [string] operations                                           |
| [`while.rhai`](scripts/while.rhai)                   | while loop                                                    |

| Example scripts                              | Description                                                                        |
| -------------------------------------------- | ---------------------------------------------------------------------------------- |
| [`speed_test.rhai`](scripts/speed_test.rhai) | a simple program to measure the speed of Rhai's interpreter (1 million iterations) |
| [`primes.rhai`](scripts/primes.rhai)         | use Sieve of Eratosthenes to find all primes smaller than a limit                  |
| [`fibonacci.rhai`](scripts/fibonacci.rhai)   | calculate the n-th Fibonacci number using a really dumb algorithm                  |
| [`mat_mul.rhai`](scripts/mat_mul.rhai)       | matrix multiplication test to measure the speed of Rhai's interpreter              |

To run the scripts, either make a tiny program or use of the `rhai_runner` example:

```bash
cargo run --example rhai_runner scripts/any_script.rhai
```

Hello world
-----------

[`Engine`]: #hello-world

To get going with Rhai, create an instance of the scripting engine via `Engine::new` and then call the `eval` method:

```rust
use rhai::{Engine, EvalAltResult};

fn main() -> Result<(), Box<EvalAltResult>>
{
    let engine = Engine::new();

    let result = engine.eval::<i64>("40 + 2")?;

    println!("Answer: {}", result);             // prints 42

    Ok(())
}
```

`EvalAltResult` is a Rust `enum` containing all errors encountered during the parsing or evaluation process.

### Script evaluation

The type parameter is used to specify the type of the return value, which _must_ match the actual type or an error is returned.
Rhai is very strict here.  Use [`Dynamic`] for uncertain return types.
There are two ways to specify the return type - _turbofish_ notation, or type inference.

```rust
let result = engine.eval::<i64>("40 + 2")?;     // return type is i64, specified using 'turbofish' notation

let result: i64 = engine.eval("40 + 2")?;       // return type is inferred to be i64

result.is::<i64>() == true;

let result: Dynamic = engine.eval("boo()")?;    // use 'Dynamic' if you're not sure what type it'll be!

let result = engine.eval::<String>("40 + 2")?;  // returns an error because the actual return type is i64, not String
```

Evaluate a script file directly:

```rust
let result = engine.eval_file::<i64>("hello_world.rhai".into())?;       // 'eval_file' takes a 'PathBuf'
```

### Compiling scripts (to AST)

To repeatedly evaluate a script, _compile_ it first into an AST (abstract syntax tree) form:

```rust
// Compile to an AST and store it for later evaluations
let ast = engine.compile("40 + 2")?;

for _ in 0..42 {
    let result: i64 = engine.eval_ast(&ast)?;

    println!("Answer #{}: {}", i, result);      // prints 42
}
```

Compiling a script file is also supported:

```rust
let ast = engine.compile_file("hello_world.rhai".into())?;
```

### Calling Rhai functions from Rust

Rhai also allows working _backwards_ from the other direction - i.e. calling a Rhai-scripted function from Rust via `call_fn`.

```rust
// Define functions in a script.
let ast = engine.compile(true,
    r"
        // a function with two parameters: String and i64
        fn hello(x, y) {
            x.len() + y
        }

        // functions can be overloaded: this one takes only one parameter
        fn hello(x) {
            x * 2
        }

        // this one takes no parameters
        fn hello() {
            42
        }
    ")?;

// A custom scope can also contain any variables/constants available to the functions
let mut scope = Scope::new();

// Evaluate a function defined in the script, passing arguments into the script as a tuple.
// Beware, arguments must be of the correct types because Rhai does not have built-in type conversions.
// If arguments of the wrong types are passed, the Engine will not find the function.

let result: i64 = engine.call_fn(&mut scope, &ast, "hello", ( String::from("abc"), 123_i64 ) )?;
//                                                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                                                          put arguments in a tuple

let result: i64 = engine.call_fn(&mut scope, &ast, "hello", (123_i64,) )?
//                                                          ^^^^^^^^^^ tuple of one

let result: i64 = engine.call_fn(&mut scope, &ast, "hello", () )?
//                                                          ^^ unit = tuple of zero
```

### Creating Rust anonymous functions from Rhai script

[`Func`]: #creating-rust-anonymous-functions-from-rhai-script

It is possible to further encapsulate a script in Rust such that it becomes a normal Rust function.
Such an _anonymous function_ is basically a boxed closure, very useful as call-back functions.
Creating them is accomplished via the `Func` trait which contains `create_from_script`
(as well as its companion method `create_from_ast`):

```rust
use rhai::{Engine, Func};                       // use 'Func' for 'create_from_script'

let engine = Engine::new();                     // create a new 'Engine' just for this

let script = "fn calc(x, y) { x + y.len() < 42 }";

// Func takes two type parameters:
//   1) a tuple made up of the types of the script function's parameters
//   2) the return type of the script function
//
// 'func' will have type Box<dyn Fn(i64, String) -> Result<bool, Box<EvalAltResult>>> and is callable!
let func = Func::<(i64, String), bool>::create_from_script(
//                ^^^^^^^^^^^^^ function parameter types in tuple

                engine,                         // the 'Engine' is consumed into the closure
                script,                         // the script, notice number of parameters must match
                "calc"                          // the entry-point function name
)?;

func(123, "hello".to_string())? == false;       // call the anonymous function

schedule_callback(func);                        // pass it as a callback to another function

// Although there is nothing you can't do by manually writing out the closure yourself...
let engine = Engine::new();
let ast = engine.compile(script)?;
schedule_callback(Box::new(move |x: i64, y: String| -> Result<bool, Box<EvalAltResult>> {
    engine.call_fn(&mut Scope::new(), &ast, "calc", (x, y))
}));
```

Raw `Engine`
------------

[raw `Engine`]: #raw-engine

`Engine::new` creates a scripting [`Engine`] with common functionalities (e.g. printing to the console via `print`).
In many controlled embedded environments, however, these are not needed.

Use `Engine::new_raw` to create a _raw_ `Engine`, in which _nothing_ is added, not even basic arithmetic and logic operators!

### Packages

Rhai functional features are provided in different _packages_ that can be loaded via a call to `load_package`.
Packages reside under `rhai::packages::*` and the trait `rhai::packages::Package` must be imported in order for
packages to be used.

```rust
use rhai::Engine;
use rhai::packages::Package                     // load the 'Package' trait to use packages
use rhai::packages::CorePackage;                // the 'core' package contains basic functionalities (e.g. arithmetic)

let mut engine = Engine::new_raw();             // create a 'raw' Engine
let package = CorePackage::new();               // create a package - can be shared among multiple `Engine` instances

engine.load_package(package.get());             // load the package manually
```

The follow packages are available:

| Package                | Description                                     | In `CorePackage` | In `StandardPackage` |
| ---------------------- | ----------------------------------------------- | :--------------: | :------------------: |
| `ArithmeticPackage`    | Arithmetic operators (e.g. `+`, `-`, `*`, `/`)  |       Yes        |         Yes          |
| `BasicIteratorPackage` | Numeric ranges (e.g. `range(1, 10)`)            |       Yes        |         Yes          |
| `LogicPackage`         | Logic and comparison operators (e.g. `==`, `>`) |       Yes        |         Yes          |
| `BasicStringPackage`   | Basic string functions                          |       Yes        |         Yes          |
| `BasicTimePackage`     | Basic time functions (e.g. [timestamps])        |       Yes        |         Yes          |
| `MoreStringPackage`    | Additional string functions                     |        No        |         Yes          |
| `BasicMathPackage`     | Basic math functions (e.g. `sin`, `sqrt`)       |        No        |         Yes          |
| `BasicArrayPackage`    | Basic [array] functions                         |        No        |         Yes          |
| `BasicMapPackage`      | Basic [object map] functions                    |        No        |         Yes          |
| `CorePackage`          | Basic essentials                                |                  |                      |
| `StandardPackage`      | Standard library                                |                  |                      |

Evaluate expressions only
-------------------------

[`eval_expression`]: #evaluate-expressions-only
[`eval_expression_with_scope`]: #evaluate-expressions-only

Sometimes a use case does not require a full-blown scripting _language_, but only needs to evaluate _expressions_.
In these cases, use the `compile_expression` and `eval_expression` methods or their `_with_scope` variants.

```rust
let result = engine.eval_expression::<i64>("2 + (10 + 10) * 2")?;
```

When evaluation _expressions_, no full-blown statement (e.g. `if`, `while`, `for`) - not even variable assignments -
is supported and will be considered parse errors when encountered.

```rust
// The following are all syntax errors because the script is not an expression.
engine.eval_expression::<()>("x = 42")?;
let ast = engine.compile_expression("let x = 42")?;
let result = engine.eval_expression_with_scope::<i64>(&mut scope, "if x { 42 } else { 123 }")?;
```

Values and types
----------------

[`type_of()`]: #values-and-types
[`to_string()`]: #values-and-types
[`()`]: #values-and-types

The following primitive types are supported natively:

| Category                                                                      | Equivalent Rust types                                                                                | `type_of()`           | `to_string()`         |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | --------------------- | --------------------- |
| **Integer number**                                                            | `u8`, `i8`, `u16`, `i16`, <br/>`u32`, `i32` (default for [`only_i32`]),<br/>`u64`, `i64` _(default)_ | `"i32"`, `"u64"` etc. | `"42"`, `"123"` etc.  |
| **Floating-point number** (disabled with [`no_float`])                        | `f32`, `f64` _(default)_                                                                             | `"f32"` or `"f64"`    | `"123.4567"` etc.     |
| **Boolean value**                                                             | `bool`                                                                                               | `"bool"`              | `"true"` or `"false"` |
| **Unicode character**                                                         | `char`                                                                                               | `"char"`              | `"A"`, `"x"` etc.     |
| **Unicode string**                                                            | `String` (_not_ `&str`)                                                                              | `"string"`            | `"hello"` etc.        |
| **Array** (disabled with [`no_index`])                                        | `rhai::Array`                                                                                        | `"array"`             | `"[ ?, ?, ? ]"`       |
| **Object map** (disabled with [`no_object`])                                  | `rhai::Map`                                                                                          | `"map"`               | `#{ "a": 1, "b": 2 }` |
| **Timestamp** (implemented in the [`BasicTimePackage`](#packages))            | `std::time::Instant`                                                                                 | `"timestamp"`         | _not supported_       |
| **Dynamic value** (i.e. can be anything)                                      | `rhai::Dynamic`                                                                                      | _the actual type_     | _actual value_        |
| **System integer** (current configuration)                                    | `rhai::INT` (`i32` or `i64`)                                                                         | `"i32"` or `"i64"`    | `"42"`, `"123"` etc.  |
| **System floating-point** (current configuration, disabled with [`no_float`]) | `rhai::FLOAT` (`f32` or `f64`)                                                                       | `"f32"` or `"f64"`    | `"123.456"` etc.      |
| **Nothing/void/nil/null** (or whatever you want to call it)                   | `()`                                                                                                 | `"()"`                | `""` _(empty string)_ |

All types are treated strictly separate by Rhai, meaning that `i32` and `i64` and `u32` are completely different -
they even cannot be added together. This is very similar to Rust.

The default integer type is `i64`. If other integer types are not needed, it is possible to exclude them and make a
smaller build with the [`only_i64`] feature.

If only 32-bit integers are needed, enabling the [`only_i32`] feature will remove support for all integer types other than `i32`, including `i64`.
This is useful on some 32-bit targets where using 64-bit integers incur a performance penalty.

If no floating-point is needed or supported, use the [`no_float`] feature to remove it.

The `to_string` function converts a standard type into a [string] for display purposes.

The `type_of` function detects the actual type of a value. This is useful because all variables are [`Dynamic`] in nature.

```rust
// Use 'type_of()' to get the actual types of values
type_of('c') == "char";
type_of(42) == "i64";

let x = 123;
x.type_of() == "i64";                           // method-call style is also OK
type_of(x) == "i64";

x = 99.999;
type_of(x) == "f64";

x = "hello";
if type_of(x) == "string" {
    do_something_with_string(x);
}
```

`Dynamic` values
----------------

[`Dynamic`]: #dynamic-values

A `Dynamic` value can be _any_ type. However, if the [`sync`] feature is used, then all types must be `Send + Sync`.

Because [`type_of()`] a `Dynamic` value returns the type of the actual value, it is usually used to perform type-specific
actions based on the actual value's type.

```rust
let mystery = get_some_dynamic_value();

if type_of(mystery) == "i64" {
    print("Hey, I got an integer here!");
} else if type_of(mystery) == "f64" {
    print("Hey, I got a float here!");
} else if type_of(mystery) == "string" {
    print("Hey, I got a string here!");
} else if type_of(mystery) == "bool" {
    print("Hey, I got a boolean here!");
} else if type_of(mystery) == "array" {
    print("Hey, I got an array here!");
} else if type_of(mystery) == "map" {
    print("Hey, I got an object map here!");
} else if type_of(mystery) == "TestStruct" {
    print("Hey, I got the TestStruct custom type here!");
} else {
    print("I don't know what this is: " + type_of(mystery));
}
```

In Rust, sometimes a `Dynamic` forms part of a returned value - a good example is an [array] with `Dynamic` elements,
or an [object map] with `Dynamic` property values.  To get the _real_ values, the actual value types _must_ be known in advance.
There is no easy way for Rust to decide, at run-time, what type the `Dynamic` value is (short of using the `type_name`
function and match against the name).

A `Dynamic` value's actual type can be checked via the `is` method.
The `cast` method then converts the value into a specific, known type.
Alternatively, use the `try_cast` method which does not panic but returns `None` when the cast fails.

```rust
let list: Array = engine.eval("...")?;          // return type is 'Array'
let item = list[0];                             // an element in an 'Array' is 'Dynamic'

item.is::<i64>() == true;                       // 'is' returns whether a 'Dynamic' value is of a particular type

let value = item.cast::<i64>();                 // if the element is 'i64', this succeeds; otherwise it panics
let value: i64 = item.cast();                   // type can also be inferred

let value = item.try_cast::<i64>().unwrap();    // 'try_cast' does not panic when the cast fails, but returns 'None'
```

The `type_name` method gets the name of the actual type as a static string slice, which you may match against.

```rust
let list: Array = engine.eval("...")?;          // return type is 'Array'
let item = list[0];                             // an element in an 'Array' is 'Dynamic'

match item.type_name() {                        // 'type_name' returns the name of the actual Rust type
    "i64" => ...
    "alloc::string::String" => ...
    "bool" => ...
    "path::to::module::TestStruct" => ...
}
```

The following conversion traits are implemented for `Dynamic`:

* `From<i64>` (`i32` if [`only_i32`])
* `From<f64>` (if not [`no_float`])
* `From<bool>`
* `From<String>`
* `From<char>`
* `From<Vec<T>>` (into an [array])
* `From<HashMap<String, T>>` (into an [object map]).

Value conversions
-----------------

[`to_int`]: #value-conversions
[`to_float`]: #value-conversions

The `to_float` function converts a supported number to `FLOAT` (`f32` or `f64`),
and the `to_int` function converts a supported number to `INT` (`i32` or `i64`).
That's about it. For other conversions, register custom conversion functions.

```rust
let x = 42;
let y = x * 100.0;                              // <- error: cannot multiply i64 with f64
let y = x.to_float() * 100.0;                   // works
let z = y.to_int() + x;                         // works

let c = 'X';                                    // character
print("c is '" + c + "' and its code is " + c.to_int());    // prints "c is 'X' and its code is 88"
```

Traits
------

A number of traits, under the `rhai::` module namespace, provide additional functionalities.

| Trait               | Description                                                                            | Methods                                 |
| ------------------- | -------------------------------------------------------------------------------------- | --------------------------------------- |
| `RegisterFn`        | Trait for registering functions                                                        | `register_fn`                           |
| `RegisterDynamicFn` | Trait for registering functions returning [`Dynamic`]                                  | `register_dynamic_fn`                   |
| `RegisterResultFn`  | Trait for registering fallible functions returning `Result<`_T_`, Box<EvalAltResult>>` | `register_result_fn`                    |
| `Func`              | Trait for creating anonymous functions from script                                     | `create_from_ast`, `create_from_script` |
| `ModuleResolver`    | Trait implemented by module resolution services                                        | `resolve`                               |

Working with functions
----------------------

Rhai's scripting engine is very lightweight.  It gets most of its abilities from functions.
To call these functions, they need to be registered with the [`Engine`].

```rust
use rhai::{Dynamic, Engine, EvalAltResult};
use rhai::RegisterFn;                           // use 'RegisterFn' trait for 'register_fn'
use rhai::{Dynamic, RegisterDynamicFn};         // use 'RegisterDynamicFn' trait for 'register_dynamic_fn'

// Normal function
fn add(x: i64, y: i64) -> i64 {
    x + y
}

// Function that returns a Dynamic value
fn get_an_any() -> Dynamic {
    Dynamic::from(42_i64)
}

fn main() -> Result<(), Box<EvalAltResult>>
{
    let engine = Engine::new();

    engine.register_fn("add", add);

    let result = engine.eval::<i64>("add(40, 2)")?;

    println!("Answer: {}", result);             // prints 42

    // Functions that return Dynamic values must use register_dynamic_fn()
    engine.register_dynamic_fn("get_an_any", get_an_any);

    let result = engine.eval::<i64>("get_an_any()")?;

    println!("Answer: {}", result);             // prints 42

    Ok(())
}
```

To return a [`Dynamic`] value from a Rust function, use the `Dynamic::from` method.

```rust
use rhai::Dynamic;

fn decide(yes_no: bool) -> Dynamic {
    if yes_no {
        Dynamic::from(42_i64)
    } else {
        Dynamic::from(String::from("hello!"))   // remember &str is not supported by Rhai
    }
}
```

Generic functions
-----------------

Rust generic functions can be used in Rhai, but separate instances for each concrete type must be registered separately.
This is essentially function overloading (Rhai does not natively support generics).

```rust
use std::fmt::Display;

use rhai::{Engine, RegisterFn};

fn show_it<T: Display>(x: &mut T) -> () {
    println!("put up a good show: {}!", x)
}

fn main()
{
    let engine = Engine::new();

    engine.register_fn("print", show_it as fn(x: &mut i64)->());
    engine.register_fn("print", show_it as fn(x: &mut bool)->());
    engine.register_fn("print", show_it as fn(x: &mut String)->());
}
```

This example shows how to register multiple functions (or, in this case, multiple overloaded versions of the same function)
under the same name. This enables function overloading based on the number and types of parameters.

Fallible functions
------------------

If a function is _fallible_ (i.e. it returns a `Result<_, Error>`), it can be registered with `register_result_fn`
(using the `RegisterResultFn` trait).

The function must return `Result<_, Box<EvalAltResult>>`. `Box<EvalAltResult>` implements `From<&str>` and `From<String>` etc.
and the error text gets converted into `Box<EvalAltResult::ErrorRuntime>`.

The error values are `Box`-ed in order to reduce memory footprint of the error path, which should be hit rarely.

```rust
use rhai::{Engine, EvalAltResult, Position};
use rhai::RegisterResultFn;                     // use 'RegisterResultFn' trait for 'register_result_fn'

// Function that may fail
fn safe_divide(x: i64, y: i64) -> Result<i64, Box<EvalAltResult>> {
    if y == 0 {
        // Return an error if y is zero
        Err("Division by zero!".into())         // short-cut to create Box<EvalAltResult::ErrorRuntime>
    } else {
        Ok(x / y)
    }
}

fn main()
{
    let engine = Engine::new();

    // Fallible functions that return Result values must use register_result_fn()
    engine.register_result_fn("divide", safe_divide);

    if let Err(error) = engine.eval::<i64>("divide(40, 0)") {
       println!("Error: {:?}", *error);         // prints ErrorRuntime("Division by zero detected!", (1, 1)")
    }
}
```

Overriding built-in functions
----------------------------

Any similarly-named function defined in a script overrides any built-in function.

```rust
// Override the built-in function 'to_int'
fn to_int(num) {
    print("Ha! Gotcha! " + num);
}

print(to_int(123));     // what happens?
```

Operator overloading
--------------------

In Rhai, a lot of functionalities are actually implemented as functions, including basic operations such as arithmetic calculations.
For example, in the expression "`a + b`", the `+` operator is _not_ built-in, but calls a function named "`+`" instead!

```rust
let x = a + b;
let x = +(a, b);        // <- the above is equivalent to this function call
```

Similarly, comparison operators including `==`, `!=` etc. are all implemented as functions, with the stark exception of `&&` and `||`.
Because they [_short-circuit_](#boolean-operators), `&&` and `||` are handled specially and _not_ via a function; as a result,
overriding them has no effect at all.

Operator functions cannot be defined as a script function (because operators syntax are not valid function names).
However, operator functions _can_ be registered to the [`Engine`] via `register_fn`, `register_result_fn` etc.
When a custom operator function is registered with the same name as an operator, it _overloads_ (or overrides) the built-in version.

```rust
use rhai::{Engine, EvalAltResult, RegisterFn};

let mut engine = Engine::new();

fn strange_add(a: i64, b: i64) -> i64 { (a + b) * 42 }

engine.register_fn("+", strange_add);               // overload '+' operator for two integers!

let result: i64 = engine.eval("1 + 0");             // the overloading version is used

println!("result: {}", result);                     // prints 42

let result: f64 = engine.eval("1.0 + 0.0");         // '+' operator for two floats not overloaded

println!("result: {}", result);                     // prints 1.0
```

Use operator overloading for custom types (described below) only.  Be very careful when overloading built-in operators because
script writers expect standard operators to behave in a consistent and predictable manner, and will be annoyed if a calculation
for '+' turns into a subtraction, for example.

Operator overloading also impacts script optimization when using [`OptimizationLevel::Full`].
See the [relevant section](#script-optimization) for more details.

Custom types and methods
-----------------------

Here's an more complete example of working with Rust.  First the example, then we'll break it into parts:

```rust
use rhai::{Engine, EvalAltResult};
use rhai::RegisterFn;

#[derive(Clone)]
struct TestStruct {
    field: i64
}

impl TestStruct {
    fn update(&mut self) {
        self.field += 41;
    }

    fn new() -> Self {
        TestStruct { field: 1 }
    }
}

fn main() -> Result<(), Box<EvalAltResult>>
{
    let engine = Engine::new();

    engine.register_type::<TestStruct>();

    engine.register_fn("update", TestStruct::update);
    engine.register_fn("new_ts", TestStruct::new);

    let result = engine.eval::<TestStruct>("let x = new_ts(); x.update(); x")?;

    println!("result: {}", result.field);           // prints 42

    Ok(())
}
```

All custom types must implement `Clone`.  This allows the [`Engine`] to pass by value.
You can turn off support for custom types via the [`no_object`] feature.

```rust
#[derive(Clone)]
struct TestStruct {
    field: i64
}
```

Next, we create a few methods that we'll later use in our scripts.  Notice that we register our custom type with the [`Engine`].

```rust
impl TestStruct {
    fn update(&mut self) {
        self.field += 41;
    }

    fn new() -> Self {
        TestStruct { field: 1 }
    }
}

let engine = Engine::new();

engine.register_type::<TestStruct>();
```

To use native types, methods and functions with the [`Engine`], we need to register them.
There are some convenience functions to help with these. Below, the `update` and `new` methods are registered with the [`Engine`].

*Note: [`Engine`] follows the convention that methods use a `&mut` first parameter so that invoking methods
can update the value in memory.*

```rust
engine.register_fn("update", TestStruct::update);   // registers 'update(&mut TestStruct)'
engine.register_fn("new_ts", TestStruct::new);      // registers 'new()'
```

Finally, we call our script.  The script can see the function and method we registered earlier.
We need to get the result back out from script land just as before, this time casting to our custom struct type.

```rust
let result = engine.eval::<TestStruct>("let x = new_ts(); x.update(); x")?;

println!("result: {}", result.field);               // prints 42
```

In fact, any function with a first argument (either by copy or via a `&mut` reference) can be used as a method call
on that type because internally they are the same thing:
methods on a type is implemented as a functions taking a `&mut` first argument.

```rust
fn foo(ts: &mut TestStruct) -> i64 {
    ts.field
}

engine.register_fn("foo", foo);

let result = engine.eval::<i64>("let x = new_ts(); x.foo()")?;

println!("result: {}", result);                     // prints 1
```

If the [`no_object`] feature is turned on, however, the _method_ style of function calls
(i.e. calling a function as an object-method) is no longer supported.

```rust
// Below is a syntax error under 'no_object' because 'len' cannot be called in method style.
let result = engine.eval::<i64>("let x = [1, 2, 3]; x.len()")?;
```

[`type_of()`] works fine with custom types and returns the name of the type.
If `register_type_with_name` is used to register the custom type
with a special "pretty-print" name, [`type_of()`] will return that name instead.

```rust
engine.register_type::<TestStruct>();
engine.register_fn("new_ts", TestStruct::new);
let x = new_ts();
print(x.type_of());                                 // prints "path::to::module::TestStruct"

engine.register_type_with_name::<TestStruct>("Hello");
engine.register_fn("new_ts", TestStruct::new);
let x = new_ts();
print(x.type_of());                                 // prints "Hello"
```

Getters and setters
-------------------

Similarly, custom types can expose members by registering a `get` and/or `set` function.

```rust
#[derive(Clone)]
struct TestStruct {
    field: i64
}

impl TestStruct {
    fn get_field(&mut self) -> i64 {
        self.field
    }

    fn set_field(&mut self, new_val: i64) {
        self.field = new_val;
    }

    fn new() -> Self {
        TestStruct { field: 1 }
    }
}

let engine = Engine::new();

engine.register_type::<TestStruct>();

engine.register_get_set("xyz", TestStruct::get_field, TestStruct::set_field);
engine.register_fn("new_ts", TestStruct::new);

let result = engine.eval::<i64>("let a = new_ts(); a.xyz = 42; a.xyz")?;

println!("Answer: {}", result);                     // prints 42
```

Indexers
--------

Custom types can also expose an _indexer_ by registering an indexer function.
A custom with an indexer function defined can use the bracket '`[]`' notation to get a property value
(but not update it - indexers are read-only).

```rust
#[derive(Clone)]
struct TestStruct {
    fields: Vec<i64>
}

impl TestStruct {
    fn get_field(&mut self, index: i64) -> i64 {
        self.fields[index as usize]
    }

    fn new() -> Self {
        TestStruct { fields: vec![1, 2, 42, 4, 5] }
    }
}

let engine = Engine::new();

engine.register_type::<TestStruct>();

engine.register_fn("new_ts", TestStruct::new);
engine.register_indexer(TestStruct::get_field);

let result = engine.eval::<i64>("let a = new_ts(); a[2]")?;

println!("Answer: {}", result);                     // prints 42
```

Needless to say, `register_type`, `register_type_with_name`, `register_get`, `register_set`, `register_get_set`
and `register_indexer` are not available when the [`no_object`] feature is turned on.
`register_indexer` is also not available when the [`no_index`] feature is turned on. 

`Scope` - Initializing and maintaining state
-------------------------------------------

[`Scope`]: #scope---initializing-and-maintaining-state

By default, Rhai treats each [`Engine`] invocation as a fresh one, persisting only the functions that have been defined
but no global state. This gives each evaluation a clean starting slate. In order to continue using the same global state
from one invocation to the next, such a state must be manually created and passed in.

All `Scope` variables are [`Dynamic`], meaning they can store values of any type.  If the [`sync`] feature is used, however,
then only types that are `Send + Sync` are supported, and the entire `Scope` itself will also be `Send + Sync`.
This is extremely useful in multi-threaded applications.

In this example, a global state object (a `Scope`) is created with a few initialized variables, then the same state is
threaded through multiple invocations:

```rust
use rhai::{Engine, Scope, EvalAltResult};

fn main() -> Result<(), Box<EvalAltResult>>
{
    let engine = Engine::new();

    // First create the state
    let mut scope = Scope::new();

    // Then push (i.e. add) some initialized variables into the state.
    // Remember the system number types in Rhai are i64 (i32 if 'only_i32') ond f64.
    // Better stick to them or it gets hard working with the script.
    scope.push("y", 42_i64);
    scope.push("z", 999_i64);

    // 'set_value' adds a variable when one doesn't exist
    scope.set_value("s", "hello, world!".to_string());  // remember to use 'String', not '&str'

    // First invocation
    engine.eval_with_scope::<()>(&mut scope, r"
        let x = 4 + 5 - y + z + s.len();
        y = 1;
    ")?;

    // Second invocation using the same state
    let result = engine.eval_with_scope::<i64>(&mut scope, "x")?;

    println!("result: {}", result);                     // prints 979

    // Variable y is changed in the script - read it with 'get_value'
    assert_eq!(scope.get_value::<i64>("y").expect("variable y should exist"), 1);

    // We can modify scope variables directly with 'set_value'
    scope.set_value("y", 42_i64);
    assert_eq!(scope.get_value::<i64>("y").expect("variable y should exist"), 42);

    Ok(())
}
```

Engine configuration options
---------------------------

| Method                   | Description                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| `set_optimization_level` | Set the amount of script _optimizations_ performed. See [`script optimization`].         |
| `set_max_call_levels`    | Set the maximum number of function call levels (default 50) to avoid infinite recursion. |

[`script optimization`]: #script-optimization

-------

Rhai Language Guide
===================

Comments
--------

Comments are C-style, including '`/*` ... `*/`' pairs and '`//`' for comments to the end of the line.

```rust
let /* intruder comment */ name = "Bob";

// This is a very important comment

/* This comment spans
   multiple lines, so it
   only makes sense that
   it is even more important */

/* Fear not, Rhai satisfies all nesting needs with nested comments:
   /*/*/*/*/**/*/*/*/*/
*/
```

Statements
----------

Statements are terminated by semicolons '`;`' - they are mandatory, except for the _last_ statement where it can be omitted.

A statement can be used anywhere where an expression is expected. The _last_ statement of a statement block
(enclosed by '`{`' .. '`}`' pairs) is always the return value of the statement. If a statement has no return value
(e.g. variable definitions, assignments) then the value will be [`()`].

```rust
let a = 42;             // normal assignment statement
let a = foo(42);        // normal function call statement
foo < 42;               // normal expression as statement

let a = { 40 + 2 };     // 'a' is set to the value of the statement block, which is the value of the last statement
//              ^ notice that the last statement does not require a terminating semicolon (although it also works with it)
//                ^ notice that a semicolon is required here to terminate the assignment statement; it is syntax error without it

4 * 10 + 2              // this is also a statement, which is an expression, with no ending semicolon because
                        // it is the last statement of the whole block
```

Variables
---------

[variables]: #variables

Variables in Rhai follow normal C naming rules (i.e. must contain only ASCII letters, digits and underscores '`_`').

Variable names must start with an ASCII letter or an underscore '`_`', must contain at least one ASCII letter,
and must start with an ASCII letter before a digit.
Therefore, names like '`_`', '`_42`', '`3a`' etc. are not legal variable names, but '`_c3po`' and '`r2d2`' are.
Variable names are also case _sensitive_.

Variables are defined using the `let` keyword. A variable defined within a statement block is _local_ to that block.

```rust
let x = 3;              // ok
let _x = 42;            // ok
let x_ = 42;            // also ok
let _x_ = 42;           // still ok

let _ = 123;            // <- syntax error: illegal variable name
let _9 = 9;             // <- syntax error: illegal variable name

let x = 42;             // variable is 'x', lower case
let X = 123;            // variable is 'X', upper case
x == 42;
X == 123;

{
    let x = 999;        // local variable 'x' shadows the 'x' in parent block
    x == 999;           // access to local 'x'
}
x == 42;                // the parent block's 'x' is not changed
```

Constants
---------

Constants can be defined using the `const` keyword and are immutable.  Constants follow the same naming rules as [variables].

```rust
const x = 42;
print(x * 2);           // prints 84
x = 123;                // <- syntax error: cannot assign to constant
```

Constants must be assigned a _value_, not an expression.

```rust
const x = 40 + 2;       // <- syntax error: cannot assign expression to constant
```

Numbers
-------

Integer numbers follow C-style format with support for decimal, binary ('`0b`'), octal ('`0o`') and hex ('`0x`') notations.

The default system integer type (also aliased to `INT`) is `i64`. It can be turned into `i32` via the [`only_i32`] feature.

Floating-point numbers are also supported if not disabled with [`no_float`]. The default system floating-point type is `i64`
(also aliased to `FLOAT`).

'`_`' separators can be added freely and are ignored within a number.

| Format           | Type             |
| ---------------- | ---------------- |
| `123_345`, `-42` | `i64` in decimal |
| `0o07_76`        | `i64` in octal   |
| `0xabcd_ef`      | `i64` in hex     |
| `0b0101_1001`    | `i64` in binary  |
| `123_456.789`    | `f64`            |

Numeric operators
-----------------

Numeric operators generally follow C styles.

| Operator | Description                                          | Integers only |
| -------- | ---------------------------------------------------- | :-----------: |
| `+`      | Plus                                                 |               |
| `-`      | Minus                                                |               |
| `*`      | Multiply                                             |               |
| `/`      | Divide (integer division if acting on integer types) |               |
| `%`      | Modulo (remainder)                                   |               |
| `~`      | Power                                                |               |
| `&`      | Binary _And_ bit-mask                                |      Yes      |
| `\|`     | Binary _Or_ bit-mask                                 |      Yes      |
| `^`      | Binary _Xor_ bit-mask                                |      Yes      |
| `<<`     | Left bit-shift                                       |      Yes      |
| `>>`     | Right bit-shift                                      |      Yes      |

```rust
let x = (1 + 2) * (6 - 4) / 2;  // arithmetic, with parentheses
let reminder = 42 % 10;         // modulo
let power = 42 ~ 2;             // power (i64 and f64 only)
let left_shifted = 42 << 3;     // left shift
let right_shifted = 42 >> 3;    // right shift
let bit_op = 42 | 99;           // bit masking
```

Unary operators
---------------

| Operator | Description |
| -------- | ----------- |
| `+`      | Plus        |
| `-`      | Negative    |

```rust
let number = -5;
number = -5 - +5;
```

Numeric functions
-----------------

The following standard functions (defined in the [`BasicMathPackage`] but excluded if using a [raw `Engine`]) operate on
`i8`, `i16`, `i32`, `i64`, `f32` and `f64` only:

| Function     | Description                       |
| ------------ | --------------------------------- |
| `abs`        | absolute value                    |
| [`to_float`] | converts an integer type to `f64` |

Floating-point functions
------------------------

The following standard functions (defined in the [`BasicMathPackage`](#packages) but excluded if using a [raw `Engine`]) operate on `f64` only:

| Category         | Functions                                                    |
| ---------------- | ------------------------------------------------------------ |
| Trigonometry     | `sin`, `cos`, `tan`, `sinh`, `cosh`, `tanh` in degrees       |
| Arc-trigonometry | `asin`, `acos`, `atan`, `asinh`, `acosh`, `atanh` in degrees |
| Square root      | `sqrt`                                                       |
| Exponential      | `exp` (base _e_)                                             |
| Logarithmic      | `ln` (base _e_), `log10` (base 10), `log` (any base)         |
| Rounding         | `floor`, `ceiling`, `round`, `int`, `fraction`               |
| Conversion       | [`to_int`]                                                   |
| Testing          | `is_nan`, `is_finite`, `is_infinite`                         |

Strings and Chars
-----------------

[string]: #strings-and-chars
[strings]: #strings-and-chars
[char]: #strings-and-chars

String and char literals follow C-style formatting, with support for Unicode ('`\u`_xxxx_' or '`\U`_xxxxxxxx_') and
hex ('`\x`_xx_') escape sequences.

Hex sequences map to ASCII characters, while '`\u`' maps to 16-bit common Unicode code points and '`\U`' maps the full,
32-bit extended Unicode code points.

Standard escape sequences:

| Escape sequence | Meaning                        |
| --------------- | ------------------------------ |
| `\\`            | back-slash `\`                 |
| `\t`            | tab                            |
| `\r`            | carriage-return `CR`           |
| `\n`            | line-feed `LF`                 |
| `\"`            | double-quote `"` in strings    |
| `\'`            | single-quote `'` in characters |
| `\x`_xx_        | Unicode in 2-digit hex         |
| `\u`_xxxx_      | Unicode in 4-digit hex         |
| `\U`_xxxxxxxx_  | Unicode in 8-digit hex         |

Internally Rhai strings are stored as UTF-8 just like Rust (they _are_ Rust `String`'s!), but there are major differences.
In Rhai a string is the same as an array of Unicode characters and can be directly indexed (unlike Rust).
This is similar to most other languages where strings are internally represented not as UTF-8 but as arrays of multi-byte
Unicode characters.
Individual characters within a Rhai string can also be replaced just as if the string is an array of Unicode characters.
In Rhai, there is also no separate concepts of `String` and `&str` as in Rust.

Strings can be built up from other strings and types via the `+` operator (provided by the [`MoreStringPackage`](#packages)
but excluded if using a [raw `Engine`]). This is particularly useful when printing output.

[`type_of()`] a string returns `"string"`.

```rust
let name = "Bob";
let middle_initial = 'C';
let last = "Davis";

let full_name = name + " " + middle_initial + ". " + last;
full_name == "Bob C. Davis";

// String building with different types
let age = 42;
let record = full_name + ": age " + age;
record == "Bob C. Davis: age 42";

// Unlike Rust, Rhai strings can be indexed to get a character
// (disabled with 'no_index')
let c = record[4];
c == 'C';

ts.s = record;                          // custom type properties can take strings

let c = ts.s[4];
c == 'C';

let c = "foo"[0];                       // indexing also works on string literals...
c == 'f';

let c = ("foo" + "bar")[5];             // ... and expressions returning strings
c == 'r';

// Escape sequences in strings
record += " \u2764\n";                  // escape sequence of '❤' in Unicode
record == "Bob C. Davis: age 42 ❤\n";   // '\n' = new-line

// Unlike Rust, Rhai strings can be directly modified character-by-character
// (disabled with 'no_index')
record[4] = '\x58'; // 0x58 = 'X'
record == "Bob X. Davis: age 42 ❤\n";

// Use 'in' to test if a substring (or character) exists in a string
"Davis" in record == true;
'X' in record == true;
'C' in record == false;
```

### Built-in functions

The following standard methods (defined in the [`MoreStringPackage`](#packages) but excluded if using a [raw `Engine`]) operate on strings:

| Function     | Parameter(s)                                                 | Description                                                                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `len`        | _none_                                                       | returns the number of characters (not number of bytes) in the string                              |
| `pad`        | character to pad, target length                              | pads the string with an character to at least a specified length                                  |
| `append`     | character/string to append                                   | Adds a character or a string to the end of another string                                         |
| `clear`      | _none_                                                       | empties the string                                                                                |
| `truncate`   | target length                                                | cuts off the string at exactly a specified number of characters                                   |
| `contains`   | character/sub-string to search for                           | checks if a certain character or sub-string occurs in the string                                  |
| `index_of`   | character/sub-string to search for, start index _(optional)_ | returns the index that a certain character or sub-string occurs in the string, or -1 if not found |
| `sub_string` | start index, length _(optional)_                             | extracts a sub-string (to the end of the string if length is not specified)                       |
| `crop`       | start index, length _(optional)_                             | retains only a portion of the string (to the end of the string if length is not specified)        |
| `replace`    | target sub-string, replacement string                        | replaces a sub-string with another                                                                |
| `trim`       | _none_                                                       | trims the string of whitespace at the beginning and end                                           |

### Examples

```rust
let full_name == " Bob C. Davis ";
full_name.len() == 14;

full_name.trim();
full_name.len() == 12;
full_name == "Bob C. Davis";

full_name.pad(15, '$');
full_name.len() == 15;
full_name == "Bob C. Davis$$$";

let n = full_name.index_of('$');
n == 12;

full_name.index_of("$$", n + 1) == 13;

full_name.sub_string(n, 3) == "$$$";

full_name.truncate(6);
full_name.len() == 6;
full_name == "Bob C.";

full_name.replace("Bob", "John");
full_name.len() == 7;
full_name == "John C.";

full_name.contains('C') == true;
full_name.contains("John") == true;

full_name.crop(5);
full_name == "C.";

full_name.crop(0, 1);
full_name == "C";

full_name.clear();
full_name.len() == 0;
```

Arrays
------

[array]: #arrays
[arrays]: #arrays
[`Array`]: #arrays

Arrays are first-class citizens in Rhai. Like C, arrays are accessed with zero-based, non-negative integer indices.
Array literals are built within square brackets '`[`' ... '`]`' and separated by commas '`,`'.
All elements stored in an array are [`Dynamic`], and the array can freely grow or shrink with elements added or removed.

The Rust type of a Rhai array is `rhai::Array`. [`type_of()`] an array returns `"array"`.

Arrays are disabled via the [`no_index`] feature.

### Built-in functions

The following methods (defined in the [`BasicArrayPackage`](#packages) but excluded if using a [raw `Engine`]) operate on arrays:

| Function     | Parameter(s)                                                          | Description                                                                                          |
| ------------ | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `push`       | element to insert                                                     | inserts an element at the end                                                                        |
| `append`     | array to append                                                       | concatenates the second array to the end of the first                                                |
| `+` operator | first array, second array                                             | concatenates the first array with the second                                                         |
| `insert`     | element to insert, position<br/>(beginning if <= 0, end if >= length) | insert an element at a certain index                                                                 |
| `pop`        | _none_                                                                | removes the last element and returns it ([`()`] if empty)                                            |
| `shift`      | _none_                                                                | removes the first element and returns it ([`()`] if empty)                                           |
| `remove`     | index                                                                 | removes an element at a particular index and returns it, or returns [`()`] if the index is not valid |
| `len`        | _none_                                                                | returns the number of elements                                                                       |
| `pad`        | element to pad, target length                                         | pads the array with an element to at least a specified length                                        |
| `clear`      | _none_                                                                | empties the array                                                                                    |
| `truncate`   | target length                                                         | cuts off the array at exactly a specified length (discarding all subsequent elements)                |

### Examples

```rust
let y = [2, 3];         // array literal with 2 elements

y.insert(0, 1);         // insert element at the beginning
y.insert(999, 4);       // insert element at the end

y.len() == 4;

y[0] == 1;
y[1] == 2;
y[2] == 3;
y[3] == 4;

(1 in y) == true;       // use 'in' to test if an item exists in the array
(42 in y) == false;     // 'in' uses the '==' operator (which users can override)
                        // to check if the target item exists in the array

y[1] = 42;              // array elements can be reassigned

(42 in y) == true;

y.remove(2) == 3;       // remove element

y.len() == 3;

y[2] == 4;              // elements after the removed element are shifted

ts.list = y;            // arrays can be assigned completely (by value copy)
let foo = ts.list[1];
foo == 42;

let foo = [1, 2, 3][0];
foo == 1;

fn abc() {
    [42, 43, 44]        // a function returning an array
}

let foo = abc()[0];
foo == 42;

let foo = y[0];
foo == 1;

y.push(4);              // 4 elements
y.push(5);              // 5 elements

y.len() == 5;

let first = y.shift();  // remove the first element, 4 elements remaining
first == 1;

let last = y.pop();     // remove the last element, 3 elements remaining
last == 5;

y.len() == 3;

for item in y {         // arrays can be iterated with a 'for' statement
    print(item);
}

y.pad(10, "hello");     // pad the array up to 10 elements

y.len() == 10;

y.truncate(5);          // truncate the array to 5 elements

y.len() == 5;

y.clear();              // empty the array

y.len() == 0;
```

`push` and `pad` are only defined for standard built-in types. For custom types, type-specific versions must be registered:

```rust
engine.register_fn("push", |list: &mut Array, item: MyType| list.push(Box::new(item)) );
```

Object maps
-----------

[`Map`]: #object-maps
[object map]: #object-maps
[object maps]: #object-maps

Object maps are dictionaries. Properties are all [`Dynamic`] and can be freely added and retrieved.
Object map literals are built within braces '`#{`' ... '`}`' (_name_ `:` _value_ syntax similar to Rust)
and separated by commas '`,`'.  The property _name_ can be a simple variable name following the same
naming rules as [variables], or an arbitrary [string] literal.

Property values can be accessed via the dot notation (_object_ `.` _property_) or index notation (_object_ `[` _property_ `]`).
The dot notation allows only property names that follow the same naming rules as [variables].
The index notation allows setting/getting properties of arbitrary names (even the empty [string]).

**Important:** Trying to read a non-existent property returns [`()`] instead of causing an error.

The Rust type of a Rhai object map is `rhai::Map`. [`type_of()`] an object map returns `"map"`.

Object maps are disabled via the [`no_object`] feature.

### Built-in functions

The following methods (defined in the [`BasicMapPackage`](#packages) but excluded if using a [raw `Engine`]) operate on object maps:

| Function     | Parameter(s)                        | Description                                                                                                                              |
| ------------ | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `has`        | property name                       | does the object map contain a property of a particular name?                                                                             |
| `len`        | _none_                              | returns the number of properties                                                                                                         |
| `clear`      | _none_                              | empties the object map                                                                                                                   |
| `remove`     | property name                       | removes a certain property and returns it ([`()`] if the property does not exist)                                                        |
| `mixin`      | second object map                   | mixes in all the properties of the second object map to the first (values of properties with the same names replace the existing values) |
| `+` operator | first object map, second object map | merges the first object map with the second                                                                                              |
| `keys`       | _none_                              | returns an [array] of all the property names (in random order), not available under [`no_index`]                                         |
| `values`     | _none_                              | returns an [array] of all the property values (in random order), not available under [`no_index`]                                        |

### Examples

```rust
let y = #{              // object map literal with 3 properties
    a: 1,
    bar: "hello",
    "baz!$@": 123.456,  // like JS, you can use any string as property names...
    "": false,          // even the empty string!

    a: 42               // <- syntax error: duplicated property name
};

y.a = 42;               // access via dot notation
y.baz!$@ = 42;          // <- syntax error: only proper variable names allowed in dot notation
y."baz!$@" = 42;        // <- syntax error: strings not allowed in dot notation

y.a == 42;

y["baz!$@"] == 123.456; // access via index notation

"baz!$@" in y == true;  // use 'in' to test if a property exists in the object map
("z" in y) == false;

ts.obj = y;             // object maps can be assigned completely (by value copy)
let foo = ts.list.a;
foo == 42;

let foo = #{ a:1, b:2, c:3 }["a"];
foo == 1;

fn abc() {
    #{ a:1, b:2, c:3 }  // a function returning an object map
}

let foo = abc().b;
foo == 2;

let foo = y["a"];
foo == 42;

y.has("a") == true;
y.has("xyz") == false;

y.xyz == ();            // a non-existing property returns '()'
y["xyz"] == ();

y.len() == 3;

y.remove("a") == 1;     // remove property

y.len() == 2;
y.has("a") == false;

for name in keys(y) {   // get an array of all the property names via the 'keys' function
    print(name);
}

for val in values(y) {  // get an array of all the property values via the 'values' function
    print(val);
}

y.clear();              // empty the object map

y.len() == 0;
```

### Parsing from JSON

The syntax for an object map is extremely similar to JSON, with the exception of `null` values which can
technically be mapped to [`()`].  A valid JSON string does not start with a hash character `#` while a
Rhai object map does - that's the major difference!

JSON numbers are all floating-point while Rhai supports integers (`INT`) and floating-point (`FLOAT`) if
the [`no_float`] feature is not turned on.  Most common generators of JSON data distinguish between
integer and floating-point values by always serializing a floating-point number with a decimal point
(i.e. `123.0` instead of `123` which is assumed to be an integer).  This style can be used successfully
with Rhai object maps.

Use the `parse_json` method to parse a piece of JSON into an object map:

```rust
// JSON string - notice that JSON property names are always quoted
//               notice also that comments are acceptable within the JSON string
let json = r#"{
                "a": 1,                 // <- this is an integer number
                "b": true,
                "c": 123.0,             // <- this is a floating-point number
                "$d e f!": "hello",     // <- any text can be a property name
                "^^^!!!": [1,42,"999"], // <- value can be array or another hash
                "z": null               // <- JSON 'null' value
              }
"#;

// Parse the JSON expression as an object map
// Set the second boolean parameter to true in order to map 'null' to '()'
let map = engine.parse_json(json, true)?;

map.len() == 6;                         // 'map' contains all properties in the JSON string

// Put the object map into a 'Scope'
let mut scope = Scope::new();
scope.push("map", map);

let result = engine.eval_with_scope::<INT>(r#"map["^^^!!!"].len()"#)?;

result == 3;                            // the object map is successfully used in the script
```

`timestamp`'s
-------------

[`timestamp`]: #timestamps
[timestamp]: #timestamps
[timestamps]: #timestamps

Timestamps are provided by the [`BasicTimePackage`](#packages) (excluded if using a [raw `Engine`]) via the `timestamp`
function.

The Rust type of a timestamp is `std::time::Instant`. [`type_of()`] a timestamp returns `"timestamp"`.

### Built-in functions

The following methods (defined in the [`BasicTimePackage`](#packages) but excluded if using a [raw `Engine`]) operate on timestamps:

| Function     | Parameter(s)                       | Description                                              |
| ------------ | ---------------------------------- | -------------------------------------------------------- |
| `elapsed`    | _none_                             | returns the number of seconds since the timestamp        |
| `-` operator | later timestamp, earlier timestamp | returns the number of seconds between the two timestamps |

### Examples

```rust
let now = timestamp();

// Do some lengthy operation...

if now.elapsed() > 30.0 {
    print("takes too long (over 30 seconds)!")
}
```

Comparison operators
--------------------

Comparing most values of the same data type work out-of-the-box for standard types supported by the system.

However, if using a [raw `Engine`], comparisons can only be made between restricted system types -
`INT` (`i64` or `i32` depending on [`only_i32`] and [`only_i64`]), `f64` (if not [`no_float`]), [string], [array], `bool`, `char`.

```rust
42 == 42;               // true
42 > 42;                // false
"hello" > "foo";        // true
"42" == 42;             // false
```

Comparing two values of _different_ data types, or of unknown data types, always results in `false`.

```rust
42 == 42.0;             // false - i64 is different from f64
42 > "42";              // false - i64 is different from string
42 <= "42";             // false again

let ts = new_ts();      // custom type
ts == 42;               // false - types are not the same
```

Boolean operators
-----------------

| Operator | Description                           |
| -------- | ------------------------------------- |
| `!`      | Boolean _Not_                         |
| `&&`     | Boolean _And_ (short-circuits)        |
| `\|\|`   | Boolean _Or_ (short-circuits)         |
| `&`      | Boolean _And_ (doesn't short-circuit) |
| `\|`     | Boolean _Or_ (doesn't short-circuit)  |

Double boolean operators `&&` and `||` _short-circuit_, meaning that the second operand will not be evaluated
if the first one already proves the condition wrong.

Single boolean operators `&` and `|` always evaluate both operands.

```rust
this() || that();       // that() is not evaluated if this() is true
this() && that();       // that() is not evaluated if this() is false

this() | that();        // both this() and that() are evaluated
this() & that();        // both this() and that() are evaluated
```

Compound assignment operators
----------------------------

```rust
let number = 5;
number += 4;            // number = number + 4
number -= 3;            // number = number - 3
number *= 2;            // number = number * 2
number /= 1;            // number = number / 1
number %= 3;            // number = number % 3
number <<= 2;           // number = number << 2
number >>= 1;           // number = number >> 1
```

The `+=` operator can also be used to build [strings]:

```rust
let my_str = "abc";
my_str += "ABC";
my_str += 12345;

my_str == "abcABC12345"
```

`if` statements
---------------

```rust
if foo(x) {
    print("It's true!");
} else if bar == baz {
    print("It's true again!");
} else if ... {
        :
} else if ... {
        :
} else {
    print("It's finally false!");
}
```

All branches of an `if` statement must be enclosed within braces '`{`' .. '`}`', even when there is only one statement.
Like Rust, there is no ambiguity regarding which `if` clause a statement belongs to.

```rust
if (decision) print("I've decided!");
//            ^ syntax error, expecting '{' in statement block
```

Like Rust, `if` statements can also be used as _expressions_, replacing the `? :` conditional operators in other C-like languages.

```rust
// The following is equivalent to C: int x = 1 + (decision ? 42 : 123) / 2;
let x = 1 + if decision { 42 } else { 123 } / 2;
x == 22;

let x = if decision { 42 }; // no else branch defaults to '()'
x == ();
```

`while` loops
-------------

```rust
let x = 10;

while x > 0 {
    x = x - 1;
    if x < 6 { continue; }  // skip to the next iteration
    print(x);
    if x == 5 { break; }    // break out of while loop
}
```

Infinite `loop`
---------------

```rust
let x = 10;

loop {
    x = x - 1;
    if x > 5 { continue; }  // skip to the next iteration
    print(x);
    if x == 0 { break; }    // break out of loop
}
```

`for` loops
-----------

Iterating through a range or an [array] is provided by the `for` ... `in` loop.

```rust
let array = [1, 3, 5, 7, 9, 42];

// Iterate through array
for x in array {
    if x > 10 { continue; } // skip to the next iteration
    print(x);
    if x == 42 { break; }   // break out of for loop
}

// The 'range' function allows iterating from first to last-1
for x in range(0, 50) {
    if x > 10 { continue; } // skip to the next iteration
    print(x);
    if x == 42 { break; }   // break out of for loop
}

// The 'range' function also takes a step
for x in range(0, 50, 3) {  // step by 3
    if x > 10 { continue; } // skip to the next iteration
    print(x);
    if x == 42 { break; }   // break out of for loop
}

// Iterate through object map
let map = #{a:1, b:3, c:5, d:7, e:9};

// Property names are returned in random order
for x in keys(map) {
    if x > 10 { continue; } // skip to the next iteration
    print(x);
    if x == 42 { break; }   // break out of for loop
}

// Property values are returned in random order
for val in values(map) {
    print(val);
}
```

`return`-ing values
-------------------

```rust
return;                     // equivalent to return ();

return 123 + 456;           // returns 579
```

Errors and `throw`-ing exceptions
--------------------------------

All of [`Engine`]'s evaluation/consuming methods return `Result<T, Box<rhai::EvalAltResult>>` with `EvalAltResult`
holding error information. To deliberately return an error during an evaluation, use the `throw` keyword.

```rust
if some_bad_condition_has_happened {
    throw error;            // 'throw' takes a string as the exception text
}

throw;                      // defaults to empty exception text: ""
```

Exceptions thrown via `throw` in the script can be captured by matching `Err(EvalAltResult::ErrorRuntime(` _reason_ `,` _position_ `))`
with the exception text captured by the first parameter.

```rust
let result = engine.eval::<i64>(r#"
    let x = 42;

    if x > 0 {
        throw x + " is too large!";
    }
"#);

println!(result);           // prints "Runtime error: 42 is too large! (line 5, position 15)"
```

Functions
---------

Rhai supports defining functions in script (unless disabled with [`no_function`]):

```rust
fn add(x, y) {
    return x + y;
}

print(add(2, 3));
```

### Implicit return

Just like in Rust, an implicit return can be used. In fact, the last statement of a block is _always_ the block's return value
regardless of whether it is terminated with a semicolon `';'`. This is different from Rust.

```rust
fn add(x, y) {              // implicit return:
    x + y;                  // value of the last statement (no need for ending semicolon)
                            // is used as the return value
}

fn add2(x) {
    return x + 2;           // explicit return
}

print(add(2, 3));           // prints 5
print(add2(42));            // prints 44
```

### No access to external scope

Functions are not _closures_. They do not capture the calling environment and can only access their own parameters.
They cannot access variables external to the function itself.

```rust
let x = 42;

fn foo() { x }              // <- syntax error: variable 'x' doesn't exist
```

### Passing arguments by value

Functions defined in script always take [`Dynamic`] parameters (i.e. the parameter can be of any type).
It is important to remember that all arguments are passed by _value_, so all functions are _pure_
(i.e. they never modifytheir arguments).
Any update to an argument will **not** be reflected back to the caller. This can introduce subtle bugs, if not careful.

```rust
fn change(s) {              // 's' is passed by value
    s = 42;                 // only a COPY of 's' is changed
}

let x = 500;
x.change();                 // de-sugars to change(x)
x == 500;                   // 'x' is NOT changed!
```

### Global definitions only

Functions can only be defined at the global level, never inside a block or another function.

```rust
// Global level is OK
fn add(x, y) {
    x + y
}

// The following will not compile
fn do_addition(x) {
    fn add_y(n) {           // <- syntax error: functions cannot be defined inside another function
        n + y
    }

    add_y(x)
}
```

Unlike C/C++, functions can be defined _anywhere_ within the global level. A function does not need to be defined
prior to being used in a script; a statement in the script can freely call a function defined afterwards.
This is similar to Rust and many other modern languages.

### Function overloading

Functions can be _overloaded_ and are resolved purely upon the function's _name_ and the _number_ of parameters
(but not parameter _types_, since all parameters are the same type - [`Dynamic`]).
New definitions _overwrite_ previous definitions of the same name and number of parameters.

```rust
fn foo(x,y,z) { print("Three!!! " + x + "," + y + "," + z) }
fn foo(x) { print("One! " + x) }
fn foo(x,y) { print("Two! " + x + "," + y) }
fn foo() { print("None.") }
fn foo(x) { print("HA! NEW ONE! " + x) }    // overwrites previous definition

foo(1,2,3);                 // prints "Three!!! 1,2,3"
foo(42);                    // prints "HA! NEW ONE! 42"
foo(1,2);                   // prints "Two!! 1,2"
foo();                      // prints "None."
```

Members and methods
-------------------

Properties and methods in a Rust custom type registered with the [`Engine`] can be called just like in Rust.

```rust
let a = new_ts();           // constructor function
a.field = 500;              // property access
a.update();                 // method call, 'a' can be changed

update(a);                  // this works, but 'a' is unchanged because only
                            // a COPY of 'a' is passed to 'update' by VALUE
```

Custom types, properties and methods can be disabled via the [`no_object`] feature.

`print` and `debug`
-------------------

The `print` and `debug` functions default to printing to `stdout`, with `debug` using standard debug formatting.

```rust
print("hello");             // prints hello to stdout
print(1 + 2 + 3);           // prints 6 to stdout
print("hello" + 42);        // prints hello42 to stdout
debug("world!");            // prints "world!" to stdout using debug formatting
```

### Overriding `print` and `debug` with callback functions

When embedding Rhai into an application, it is usually necessary to trap `print` and `debug` output
(for logging into a tracking log, for example).

```rust
// Any function or closure that takes an '&str' argument can be used to override
// 'print' and 'debug'
engine.on_print(|x| println!("hello: {}", x));
engine.on_debug(|x| println!("DEBUG: {}", x));

// Example: quick-'n-dirty logging
let logbook = Arc::new(RwLock::new(Vec::<String>::new()));

// Redirect print/debug output to 'log'
let log = logbook.clone();
engine.on_print(move |s| log.write().unwrap().push(format!("entry: {}", s)));

let log = logbook.clone();
engine.on_debug(move |s| log.write().unwrap().push(format!("DEBUG: {}", s)));

// Evaluate script
engine.eval::<()>(script)?;

// 'logbook' captures all the 'print' and 'debug' output
for entry in logbook.read().unwrap().iter() {
    println!("{}", entry);
}
```

Using external modules
----------------------

[module]: #using-external-modules
[modules]: #using-external-modules

Rhai allows organizing code (functions and variables) into _modules_.  A module is a single script file
with `export` statements that _exports_ certain global variables and functions as contents of the module.

Everything exported as part of a module is constant and read-only.

### Importing modules

A module can be _imported_ via the `import` statement, and its members accessed via '`::`' similar to C++.

```rust
import "crypto" as crypto;  // import the script file 'crypto.rhai' as a module

crypto::encrypt(secret);    // use functions defined under the module via '::'

print(crypto::status);      // module variables are constants

crypto::hash::sha256(key);  // sub-modules are also supported
```

`import` statements are _scoped_, meaning that they are only accessible inside the scope that they're imported.

```rust
let mod = "crypto";

if secured {                // new block scope
    import mod as crypto;   // import module (the path needs not be a constant string)

    crypto::encrypt(key);   // use a function in the module
}                           // the module disappears at the end of the block scope

crypto::encrypt(others);    // <- this causes a run-time error because the 'crypto' module
                            //    is no longer available!
```

### Creating custom modules from Rust

To load a custom module into an [`Engine`], first create a `Module` type, add variables/functions into it,
then finally push it into a custom [`Scope`].  This has the equivalent effect of putting an `import` statement
at the beginning of any script run.

```rust
use rhai::{Engine, Scope, Module, i64};

let mut engine = Engine::new();
let mut scope = Scope::new();

let mut module = Module::new();             // new module
module.set_var("answer", 41_i64);           // variable 'answer' under module
module.set_fn_1("inc", |x: i64| Ok(x+1));   // use the 'set_fn_XXX' API to add functions

// Push the module into the custom scope under the name 'question'
// This is equivalent to 'import "..." as question;'
scope.push_module("question", module);

// Use module-qualified variables
engine.eval_expression_with_scope::<i64>(&scope, "question::answer + 1")? == 42;

// Call module-qualified functions
engine.eval_expression_with_scope::<i64>(&scope, "question::inc(question::answer)")? == 42;
```

### Module resolvers

When encountering an `import` statement, Rhai attempts to _resolve_ the module based on the path string.
_Module Resolvers_ are service types that implement the [`ModuleResolver`](#traits) trait.
There are a number of standard resolvers built into Rhai, the default being the `FileModuleResolver`
which simply loads a script file based on the path (with `.rhai` extension attached) and execute it to form a module.

Built-in module resolvers are grouped under the `rhai::module_resolvers` module namespace.

| Module Resolver        | Description                                                                                                                                                                                                                                                                |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `FileModuleResolver`   | The default module resolution service, not available under the [`no_std`] feature. Loads a script file (based off the current directory) with `.rhai` extension.<br/>The base directory can be changed via the `FileModuleResolver::new_with_path()` constructor function. |
| `StaticModuleResolver` | Loads modules that are statically added. This can be used when the [`no_std`] feature is turned on.                                                                                                                                                                        |

An [`Engine`]'s module resolver is set via a call to `set_module_resolver`:

```rust
// Use the 'StaticModuleResolver'
let resolver = rhai::module_resolvers::StaticModuleResolver::new();
engine.set_module_resolver(Some(resolver));

// Effectively disable 'import' statements by setting module resolver to 'None'
engine.set_module_resolver(None);
```

Script optimization
===================

Rhai includes an _optimizer_ that tries to optimize a script after parsing.
This can reduce resource utilization and increase execution speed.
Script optimization can be turned off via the [`no_optimize`] feature.

For example, in the following:

```rust
{
    let x = 999;            // NOT eliminated: Rhai doesn't check yet whether a variable is used later on
    123;                    // eliminated: no effect
    "hello";                // eliminated: no effect
    [1, 2, x, x*2, 5];      // eliminated: no effect
    foo(42);                // NOT eliminated: the function 'foo' may have side effects
    666                     // NOT eliminated: this is the return value of the block,
                            // and the block is the last one so this is the return value of the whole script
}
```

Rhai attempts to eliminate _dead code_ (i.e. code that does nothing, for example an expression by itself as a statement,
which is allowed in Rhai). The above script optimizes to:

```rust
{
    let x = 999;
    foo(42);
    666
}
```

Constants propagation is used to remove dead code:

```rust
const ABC = true;
if ABC || some_work() { print("done!"); }   // 'ABC' is constant so it is replaced by 'true'...
if true || some_work() { print("done!"); }  // since '||' short-circuits, 'some_work' is never called
if true { print("done!"); }                 // <- the line above is equivalent to this
print("done!");                             // <- the line above is further simplified to this
                                            //    because the condition is always true
```

These are quite effective for template-based machine-generated scripts where certain constant values
are spliced into the script text in order to turn on/off certain sections.
For fixed script texts, the constant values can be provided in a user-defined [`Scope`] object
to the [`Engine`] for use in compilation and evaluation.

Beware, however, that most operators are actually function calls, and those functions can be overridden,
so they are not optimized away:

```rust
const DECISION = 1;

if DECISION == 1 {          // NOT optimized away because you can define
    :                       // your own '==' function to override the built-in default!
    :
} else if DECISION == 2 {   // same here, NOT optimized away
    :
} else if DECISION == 3 {   // same here, NOT optimized away
    :
} else {
    :
}
```

because no operator functions will be run (in order not to trigger side effects) during the optimization process
(unless the optimization level is set to [`OptimizationLevel::Full`]). So, instead, do this:

```rust
const DECISION_1 = true;
const DECISION_2 = false;
const DECISION_3 = false;

if DECISION_1 {
    :                       // this branch is kept and promoted to the parent level
} else if DECISION_2 {
    :                       // this branch is eliminated
} else if DECISION_3 {
    :                       // this branch is eliminated
} else {
    :                       // this branch is eliminated
}
```

In general, boolean constants are most effective for the optimizer to automatically prune
large `if`-`else` branches because they do not depend on operators.

Alternatively, turn the optimizer to [`OptimizationLevel::Full`]

Here be dragons!
================

Optimization levels
-------------------

[`OptimizationLevel::Full`]: #optimization-levels
[`OptimizationLevel::Simple`]: #optimization-levels
[`OptimizationLevel::None`]: #optimization-levels

There are actually three levels of optimizations: `None`, `Simple` and `Full`.

* `None` is obvious - no optimization on the AST is performed.

* `Simple` (default) performs relatively _safe_ optimizations without causing side effects
  (i.e. it only relies on static analysis and will not actually perform any function calls).

* `Full` is _much_ more aggressive, _including_ running functions on constant arguments to determine their result.
  One benefit to this is that many more optimization opportunities arise, especially with regards to comparison operators.

An [`Engine`]'s optimization level is set via a call to `set_optimization_level`:

```rust
// Turn on aggressive optimizations
engine.set_optimization_level(rhai::OptimizationLevel::Full);
```

If it is ever needed to _re_-optimize an `AST`, use the `optimize_ast` method:

```rust
// Compile script to AST
let ast = engine.compile("40 + 2")?;

// Create a new 'Scope' - put constants in it to aid optimization if using 'OptimizationLevel::Full'
let scope = Scope::new();

// Re-optimize the AST
let ast = engine.optimize_ast(&scope, &ast, OptimizationLevel::Full);
```

When the optimization level is [`OptimizationLevel::Full`], the [`Engine`] assumes all functions to be _pure_ and will _eagerly_
evaluated all function calls with constant arguments, using the result to replace the call. This also applies to all operators
(which are implemented as functions). For instance, the same example above:

```rust
// When compiling the following with OptimizationLevel::Full...

const DECISION = 1;
                            // this condition is now eliminated because 'DECISION == 1'
if DECISION == 1 {          // is a function call to the '==' function, and it returns 'true'
    print("hello!");        // this block is promoted to the parent level
} else {
    print("boo!");          // this block is eliminated because it is never reached
}

print("hello!");            // <- the above is equivalent to this
                            //    ('print' and 'debug' are handled specially)
```

Because of the eager evaluation of functions, many constant expressions will be evaluated and replaced by the result.
This does not happen with [`OptimizationLevel::Simple`] which doesn't assume all functions to be _pure_.

```rust
// When compiling the following with OptimizationLevel::Full...

let x = (1+2)*3-4/5%6;      // <- will be replaced by 'let x = 9'
let y = (1>2) || (3<=4);    // <- will be replaced by 'let y = true'
```

Function side effect considerations
----------------------------------

All of Rhai's built-in functions (and operators which are implemented as functions) are _pure_ (i.e. they do not mutate state
nor cause side any effects, with the exception of `print` and `debug` which are handled specially) so using
[`OptimizationLevel::Full`] is usually quite safe _unless_ you register your own types and functions.

If custom functions are registered, they _may_ be called (or maybe not, if the calls happen to lie within a pruned code block).
If custom functions are registered to replace built-in operators, they will also be called when the operators are used
(in an `if` statement, for example) and cause side-effects.

Function volatility considerations
---------------------------------

Even if a custom function does not mutate state nor cause side effects, it may still be _volatile_, i.e. it _depends_
on the external environment and is not _pure_. A perfect example is a function that gets the current time -
obviously each run will return a different value! The optimizer, when using [`OptimizationLevel::Full`], _assumes_ that
all functions are _pure_, so when it finds constant arguments it will eagerly execute the function call.
This causes the script to behave differently from the intended semantics because essentially the result of the function call
will always be the same value.

Therefore, **avoid using [`OptimizationLevel::Full`]** if you intend to register non-_pure_ custom types and/or functions.

Subtle semantic changes
-----------------------

Some optimizations can alter subtle semantics of the script.  For example:

```rust
if true {                   // condition always true
    123.456;                // eliminated
    hello;                  // eliminated, EVEN THOUGH the variable doesn't exist!
    foo(42)                 // promoted up-level
}

foo(42)                     // <- the above optimizes to this
```

Nevertheless, if the original script were evaluated instead, it would have been an error - the variable `hello` doesn't exist,
so the script would have been terminated at that point with an error return.

In fact, any errors inside a statement that has been eliminated will silently _disappear_:

```rust
print("start!");
if my_decision { /* do nothing... */ }  // eliminated due to no effect
print("end!");

// The above optimizes to:

print("start!");
print("end!");
```

In the script above, if `my_decision` holds anything other than a boolean value, the script should have been terminated due to
a type error. However, after optimization, the entire `if` statement is removed (because an access to `my_decision` produces
no side effects), thus the script silently runs to completion without errors.

Turning off optimizations
-------------------------

It is usually a bad idea to depend on a script failing or such kind of subtleties, but if it turns out to be necessary
(why? I would never guess), turn it off by setting the optimization level to [`OptimizationLevel::None`].

```rust
let engine = rhai::Engine::new();

// Turn off the optimizer
engine.set_optimization_level(rhai::OptimizationLevel::None);
```

`eval` - or "How to Shoot Yourself in the Foot even Easier"
---------------------------------------------------------

Saving the best for last: in addition to script optimizations, there is the ever-dreaded... `eval` function!

```rust
let x = 10;

fn foo(x) { x += 12; x }

let script = "let y = x;";  // build a script
script +=    "y += foo(y);";
script +=    "x + y";

let result = eval(script);  // <- look, JS, we can also do this!

print("Answer: " + result); // prints 42

print("x = " + x);          // prints 10: functions call arguments are passed by value
print("y = " + y);          // prints 32: variables defined in 'eval' persist!

eval("{ let z = y }");      // to keep a variable local, use a statement block

print("z = " + z);          // <- error: variable 'z' not found

"print(42)".eval();         // <- nope... method-call style doesn't work
```

Script segments passed to `eval` execute inside the current [`Scope`], so they can access and modify _everything_,
including all variables that are visible at that position in code! It is almost as if the script segments were
physically pasted in at the position of the `eval` call. But because of this, new functions cannot be defined
within an `eval` call, since functions can only be defined at the global level, not inside a function call!

```rust
let script = "x += 32";
let x = 10;
eval(script);               // variable 'x' in the current scope is visible!
print(x);                   // prints 42

// The above is equivalent to:
let script = "x += 32";
let x = 10;
x += 32;
print(x);
```

For those who subscribe to the (very sensible) motto of ["`eval` is **evil**"](http://linterrors.com/js/eval-is-evil),
disable `eval` by overriding it, probably with something that throws.

```rust
fn eval(script) { throw "eval is evil! I refuse to run " + script }

let x = eval("40 + 2");     // 'eval' here throws "eval is evil! I refuse to run 40 + 2"
```

Or override it from Rust:

```rust
fn alt_eval(script: String) -> Result<(), EvalAltResult> {
    Err(format!("eval is evil! I refuse to run {}", script).into())
}

engine.register_result_fn("eval", alt_eval);
```
