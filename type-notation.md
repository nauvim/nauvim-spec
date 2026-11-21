# Nauvim — Type notation

## 1 Scope

This supplement defines the abstract types and annotation syntax used in [the Nauvim specification](specification.md). Its requirements apply wherever that specification uses these notations. Interface-specific requirements, including value ranges, character encodings, field presence and error handling, shall also apply.

A type shall be one of the types defined below or a finite combination of them. Named types may refer to other named types, including recursively, as in the definition of `object`.

The annotations are descriptive notation. They are not source code in a particular programming language.

## 2 Language and interface bindings

In this supplement, a **binding** means a mapping of the abstract types to a programming language or an interface mechanism, such as a foreign function interface (FFI) or a remote procedure call (RPC) protocol.

Memory representations and wire encodings may differ between bindings. The types shall remain distinguishable either statically or dynamically.

Within the choices permitted below, numeric ranges and precision may differ between bindings when the differences are not significant in most normal uses of the interface. An explicit width or an interface-specific range shall take precedence over this permission. Numeric widths shall not, by themselves, prescribe a memory layout or wire encoding.

## 3 Scalar and byte-sequence types

### 3.1 `nil`

`nil` denotes the absence of a value.

### 3.2 `bool`

`bool` denotes a Boolean value, either `true` or `false`.

### 3.3 `int`

`int` denotes an integer. A binding may use a signed or unsigned integer type with a width of 16, 32 or 64 bits, subject to Clause 2.

Where signedness and width need to be specified, `i16`, `i32` and `i64` shall denote signed 16-, 32- and 64-bit integers, respectively. `u16`, `u32` and `u64` shall denote unsigned 16-, 32- and 64-bit integers, respectively.

An explicitly sized integer type shall specify the value range, rather than the binding's storage width. A binding may use a wider integer representation if it preserves that range.

### 3.4 `float`

`float` denotes a floating-point number with unspecified precision and is intended primarily for finite normal numbers. Where precision needs to be specified, `f32` and `f64` shall denote single-precision 32-bit and double-precision 64-bit floating-point numbers, respectively.

Floating-point representations and operations should conform to IEEE 754 [1]. Conformance to IEEE 754 or another floating-point standard is not required by this notation.

Support for and handling of NaN, infinities (`inf` and `-inf`), signed zeros (`+0.0` and `-0.0`) and subnormal numbers are therefore unspecified.

### 3.5 `string`

`string` denotes a byte sequence representing text. A `string` shall be valid UTF-8, shall not contain NUL (`U+0000`) and shall not begin with `U+FEFF` (ZERO WIDTH NO-BREAK SPACE).

The maximum supported length in bytes shall be represented by a signed or unsigned 32- or 64-bit integer. The choice may differ between bindings.

### 3.6 `bytes`

`bytes` denotes an arbitrary byte sequence. A `bytes` value may contain NUL bytes and may begin with any byte sequence, including a byte order mark. No text encoding is implied.

The maximum supported length in bytes shall be represented by a signed or unsigned 32- or 64-bit integer. The choice may differ between bindings.

## 4 Composite and callable types

### 4.1 `list[T]`

`list[T]` denotes a fixed-size, ordered sequence of elements of type `T`. A binding may represent it as a list, an array or another sequence type.

The number of elements shall be fixed for a given value, but need not be determined at compile time. Different values of `list[T]` may have different lengths. The maximum supported number of elements shall be represented by a signed or unsigned 32- or 64-bit integer.

### 4.2 `dict[K, V]`

`dict[K, V]` denotes a fixed-size dictionary with keys of type `K` and values of type `V`. A binding may call this a dictionary or a map.

The number of entries shall be fixed for a given value, but need not be determined at compile time. Different values of `dict[K, V]` may have different numbers of entries. The maximum supported number of entries shall be represented by a signed or unsigned 32- or 64-bit integer.

The data structure, algorithms and computational complexity of dictionary operations are unspecified.

### 4.3 `(T1, ..., TN)`

`(T1, ..., TN)` denotes a tuple of exactly `N` ordered components. Each component shall have the type specified at its position; the component types may differ.

Bindings are not required to represent or interpret a tuple as `list[T1 | ... | TN]`.

Parentheses containing a single type without a comma shall group that type expression. `(T,)` shall denote a one-component tuple, and `()` shall denote an empty tuple.

### 4.4 `func(T1, ..., TN) -> U`

`func(T1, ..., TN) -> U` denotes a function whose arguments have types `T1` through `TN`, in that order, and whose result has type `U`.

Parentheses may group a function type within a union or group its result type. A function may also return a tuple:

```text
string | (func(HostInfo) -> string)
func(HostInfo) -> (string | NvimProg)
func(string) -> (int, bool)
```

### 4.5 `T1 | ... | TN`

`T1 | ... | TN` denotes the union of a finite number of types `T1` through `TN`. A value shall belong to at least one of those types. A binding should distinguish the alternative types either statically or dynamically, subject to the type-distinction requirement in Clause 2.

A union shall denote the same type after duplicate occurrences of any type are removed. This requirement shall apply to unions of any finite number of types. In particular, `T | T` shall denote the same type as `T`.

EXAMPLE `T | U | T | T` denotes the same type as `T | U`.

<a id="45-object"></a>

### 4.6 `object`

`object` shall have the following recursive definition:

```text
object := nil | bool | int | float | string | list[object] | dict[string, object]
```

`bytes`, tuple types and function types are excluded from `object`, including within its lists and dictionaries.

## 5 Declarations and annotations

### 5.1 Type declarations

A type declaration shall use `:=`:

```text
T := U
```

This declares `T` as a name for `U`. The symbol `=` in value expressions shall not be interpreted as a type declaration.

### 5.2 Variables and function signatures

A variable annotation shall use `name: T`. A named function signature shall annotate its arguments in the same way:

```text
var: T
function_name(arg1: T1, arg2: T2) -> U
```

A dictionary shape may be written directly as a function argument's type. A signature may name the components of its result tuple, with their types and meanings defined in the accompanying text. In particular, the Nauvim Lua signatures use `-> (ret, err)` for the return convention specified in [11.2.1 of the main specification](specification.md#1121-lua-return-convention).

### 5.3 Dictionary shapes

Braces shall describe a dictionary by its named keys and their value types:

```text
{
    key1: V1,
    nested {
        key2: V2,
    },
}
```

`nested { ... }` shall be shorthand for `nested: { ... }`. Both forms describe a dictionary-valued field.

Field-presence requirements, defaults and the treatment of omitted or nil fields shall follow the interface using the shape. A shape alone shall not define those rules.

### 5.4 Values in type positions

Let `T1`, ..., `TN` be types and let `v1: T1`, ..., `vN: TN` be values. Each value `vi` may also be used in a type position. In that position, `vi` shall denote a subtype of `Ti` that accepts only the value `vi`.

The expression `v1 | ... | vN` may be used in a type position. It shall denote a subtype of `T1 | ... | TN` that accepts only the values `v1`, ..., `vN`.

EXAMPLE 1 `"+" | "*"` denotes a subtype of `string` that accepts only `"+"` or `"*"`.

EXAMPLE 2 `0 | false` denotes a subtype of `int | bool` that accepts only the integer `0` or the Boolean value `false`.

## Bibliography

[1] IEEE. [IEEE 754 — Standard for Floating-Point Arithmetic](https://standards.ieee.org/ieee/754/6210/). Supporting reference for the recommendation in 3.4; conformance is not required.
