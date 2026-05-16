---
name: ocaml2moonbit-migration
description: Guide for migrating OCaml projects, libraries, modules, and test suites to idiomatic MoonBit. Use when translating OCaml code to MoonBit, planning a large OCaml-to-MoonBit port, preserving byte/string-heavy behavior, replacing OCaml variants/records/exceptions/refs/arrays, mapping OCaml APIs to MoonBit packages, or building verification and test strategy for a migration.
---

# OCaml to MoonBit Migration

Port behavior, data invariants, and public contracts first. Translate syntax only after the source semantics are classified. The most common porting bug is silently coercing OCaml `string` (a byte sequence) into MoonBit `String` (UTF-16 text); classify every field by meaning before choosing a type.

When in doubt, probe with `moon run -c '...'`. Probes in this guide were verified on `moon 0.1.20260512`-class toolchains; rerun the relevant probe if the local toolchain is newer and the behavior is load-bearing.

## Migration Workflow

1. Inventory the OCaml module boundary: public types, functions, exceptions, optional arguments, mutable state, lazy/deferred state, C/Unix/filesystem dependencies, and existing tests or golden fixtures.
2. Classify every OCaml `string` by meaning before choosing a MoonBit type. Do this field by field, even inside the same OCaml record.
3. Choose MoonBit package boundaries and imports before coding. Add imports to `moon.pkg`; MoonBit source files do not use OCaml-style `open`.
4. Port one behavioral slice at a time with tests. Prefer a thin public API skeleton, then fill parser/serializer/algorithm internals behind it.
5. Probe uncertain language or library behavior with `moon run -c` and, when needed, a small OCaml toplevel probe. Keep probes minimal.
6. Finish each slice with `moon check --warn-list +73`, `moon test`, `moon info`, and `moon fmt`.

## Type-Mapping Cheatsheet

| OCaml use | MoonBit default |
|---|---|
| binary payload, file contents, compressed/encrypted/checksummed data, parser input | `Bytes` or `BytesView` |
| human-readable text, diagnostics, labels that are truly Unicode | `String` |
| single byte with known 0..255 range | `Byte` |
| indexes, counts, small identifiers, deliberate signed 32-bit wrapping | `Int` |
| file offsets, serialized positions, large object numbers | `Int64` or `UInt64` |
| OCaml `array` with fixed length | `FixedArray[T]` |
| mutable growable builder | `Array[T]` |
| read-only sequence parameter | `ArrayView[T]` or `BytesView` |
| OCaml `ref` | `Ref[T]` |
| OCaml variant | `enum`, often `priv enum` for internal states |
| OCaml record | `struct`, with `{ ..old, field: value }` for immutable update |
| OCaml exception flow | `suberror` plus checked `raise` |
| lazy/deferred state | `Lazy[T]` |
| OCaml `int32` needing wrapping | `Int` |
| OCaml `int32` needing wide arithmetic | `Int64` or `UInt64` |
| OCaml `float` | `Double` |

There is no `Int32` type in the current toolchain.

## Bytes vs String

OCaml `String.length` counts bytes; MoonBit `String::length()` counts UTF-16 code units. The same source character takes different positions in the two languages, and that difference is silent.

```sh
ocaml -noprompt -noinit <<'EOF'
let s = "𝄞";;
Printf.printf "%d\n" (String.length s);;
Printf.printf "%d\n" (Char.code s.[0]);;
EOF
# 4
# 240
```

```sh
moon run -c 'fn main { let s = "𝄞"; println(s.length()); println(s.char_length()) }'
# 2
# 1
```

`String::length()` is UTF-16 code units, `String::char_length()` is Unicode scalars, and neither is bytes. For byte-oriented formats use `Bytes`/`BytesView`. Convert between `Bytes` and `String` only through a named encoding helper (`@ascii.encode`, UTF-8 decoder, etc.) that documents the encoding assumption.

```sh
moon run -c 'fn main { let raw : Array[Byte] = [65, 0, 255]; let b = Bytes::from_array(raw); println(b.length()); println(b[2].to_int()) }'
# 3
# 255
```

```sh
moon run -c 'fn main { let source : Array[Byte] = [1, 2]; let bytes = Bytes::from_array(source); source[0] = 9; println(bytes[0].to_int()); println(source[0].to_int()) }'
# 1
# 9
```

`Bytes::from_array` produces an immutable owned `Bytes`. Mutating the source array afterwards does not affect the frozen value. Port OCaml `Bytes` mutators either as `BytesView -> Bytes` transforms, or keep state in `Array[Byte]`/`FixedArray[Byte]` until the final freeze.

```sh
moon run -c 'fn main { let empty = Bytes::new(0); let zeros = Bytes::new(2); println(empty.length()); println(zeros.length()); println(zeros[0].to_int()) }'
# 0
# 2
# 0
```

`Bytes::new(length)` always zero-fills and requires an explicit length. Use it where OCaml `Bytes.empty` or `Bytes.make len '\000'` would have appeared.

`Bytes::copy` is deprecated because `Bytes` is immutable. When a port must materialize a fresh physical copy (e.g. to preserve an OCaml promise that no buffer is shared), use `Bytes::makei(len, i => src[i])`.

```sh
moon run -c $'let cached_filter = b"/Filter"\nfn main { let b = b"PDF"; println(b.length()); println(b[0].to_int()); println(cached_filter.length()); println(cached_filter[0].to_int()) }'
# 3
# 80
# 7
# 47
```

`b"..."` is a compile-time byte-string literal of type `Bytes`. Use it for ASCII format syntax (`b"/Filter"`, `b"PDF"`, etc.) — no `@ascii` import, no runtime call, no type annotation needed at top level. Reserve `@ascii.encode(text)` for `String → Bytes` conversion when the source is a dynamic `String`. Do not build binary file formats by `String` concatenation.

```sh
moon run -c $'fn main { let buf = @buffer.new(); buf.write_bytes(b"PDF"); buf.write_byte(10); let bytes = buf.contents(); println(bytes.length()); println(bytes[0].to_int()); println(bytes[3].to_int()) }'
# 4
# 80
# 10
```

The canonical MoonBit byte-builder is `@buffer.Buffer` (from `moonbitlang/core/buffer`). Construct with `@buffer.new()` (optionally `size_hint=N`), append static ASCII via `buf.write_bytes(b"...")`, single bytes via `buf.write_byte(n)`, dynamic text via `buf.write_bytes(@ascii.encode(text))`, binary payloads via `buf.write_bytes(view)`, then freeze with `buf.contents()`. Reserve `Array[Byte]` plus `Bytes::from_array` for cases that need random-access mutation of in-flight bytes; `Buffer` is the OCaml `Buffer.t` analogue.

```sh
moon run -c 'fn main { let s = "/UniJIS-UCS2-H"; println(s.has_prefix("/Uni")); println(s.contains("-UCS2-")); println(s.has_suffix("-H")); println(s[1:]); println(s.unsafe_substring(start=1, end=s.length())) }'
# true
# true
# true
# UniJIS-UCS2-H
# UniJIS-UCS2-H
```

`String::has_prefix`, `has_suffix`, and `contains` cover predicate work. Slice with `s[start:end]` — like `BytesView`, this returns a borrowed `StringView`, cheap and good for inspection/pattern matching. When the callee needs an owned `String` (e.g. for storage or a `String` parameter), use `s.unsafe_substring(start=..., end=...)` instead. Offsets are UTF-16 code-unit offsets, not bytes; use these only for ASCII-validated tokens or genuinely textual parsing.

## Bytes Views

```sh
moon run -c 'fn has_ab(view : BytesView) -> Bool { match view { [65, 66, ..] => true; _ => false } }
fn main { let bytes : Bytes = [65, 66, 67]; let view = bytes[:2]; println(view.length()); println(view[0].to_int()); println(has_ab(bytes[:])); println(has_ab(view)) }'
# 2
# 65
# true
# true
```

`BytesView` is the byte-sequence counterpart to `ArrayView`. Views are cheap slices, expose read-only byte operations, and support pattern matching with rest patterns. Prefer `BytesView` for read-only byte APIs; call `.to_owned()` only at explicit ownership boundaries. `BytesView::to_owned()` returns the original bytes for a whole view but allocates and copies for a partial slice.

`Bytes` is assignable to a `BytesView` parameter, so a single API can accept owned or borrowed input. A returned `BytesView` keeps its backing `Bytes` alive across function boundaries — useful for exposing decode/decrypt results without forcing a copy.

The reverse direction is **not** automatic: a `BytesView` does not type-check where owned `Bytes` is required, including in native extern declarations. Treat that as the ownership boundary and call `.to_owned()` deliberately. Equality works when the view is the left operand; if `Bytes` is on the left, slice it (`bytes[:]`) before comparing.

```sh
moon run -c 'fn main { let data : Bytes = [60, 65, 62, 0, 12, 60, 66, 62]; println(data.length()); println(data[3].to_int()); println(data[4].to_int()); let view = data[5:]; match view { [60, 66, 62] => println("match"); _ => println("miss") } }'
# 8
# 0
# 12
# match
```

NUL (`0`) and form-feed (`12`) are ordinary bytes in `Bytes`/`BytesView`. Port OCaml byte predicates literally; do not narrow a format whitespace predicate to a string-oriented space/tab check.

## Integers

```sh
moon run -c 'fn main { let (b, u, u16, i64, u64) : (Byte, UInt, UInt16, Int64, UInt64) = (255, 7, 65535, 42, 42); println(b.to_int()); println(u.to_string()); println(u16.to_int()); println(i64.to_string()); println(u64.to_string()) }'
# 255
# 7
# 65535
# 42
# 42
```

Scalar types: `Byte`, `Int16`, `UInt16`, `Int`, `UInt`, `Int64`, `UInt64`, `Float`, `Double`. There is **no** `Int32`; use `Int` for deliberate signed 32-bit wrapping, or `Int64`/`UInt64` when an OCaml `int32` value must be represented without truncation. When calling a method on an integer literal, parenthesize: `(65).to_byte()`, not `65.to_byte()` — `65.` parses as the start of a float.

```sh
moon run -c 'fn main { let max = 2147483647; println(max + 1); println(1 << 31); println(0xF0 & 0x0F); println(0x80 >> 7); println(0xAA ^ 0xFF) }'
# -2147483648
# -2147483648
# 0
# 1
# 85
```

`Int` arithmetic wraps modulo 2^32 with signed interpretation. Symbolic operators map directly from OCaml: `&` (land), `|` (lor), `^` (lxor), `<<` (lsl), `>>` (right shift, arithmetic for signed).

```sh
moon run -c 'fn main { println((-8) >> 1); println(1 << 32); println(1 >> 32); let logical = ((-8).reinterpret_as_uint() >> 1).reinterpret_as_int(); println(logical) }'
# -4
# 1
# 1
# 2147483644
```

`Int` right shift is arithmetic, and shift counts are **masked to 5 bits**, so `1 << 32 == 1`. For OCaml `Int32.shift_right_logical`, reinterpret signed→unsigned, shift, reinterpret back. Numeric conversion (`.to_uint()`) is **not** the same as bit reinterpretation (`.reinterpret_as_uint()`).

```sh
moon run -c $'fn main { let parsed : Result[Int, Error] = try? @strconv.from_str("2147483648"); println(parsed is Err(_)); let mut value = 0; for digit in [50,49,52,55,52,56,51,54,52,56] { value = value * 10 + digit - 48 }; println(value); let mut wide = 0L; for digit in [50,49,52,55,52,56,51,54,52,56] { wide = wide * 10L + (digit - 48).to_int64() }; println(wide.to_string()) }'
# true
# -2147483648
# 2147483648
```

`@strconv.from_str<Int>` rejects out-of-range decimals, but handwritten digit accumulation in `Int` silently wraps. Accumulate offsets, lengths, object numbers, or serialized counters in `Int64`/`UInt64`; bounds-check before narrowing to `Int`.

```sh
moon run -c 'fn main { println(0x8EA2A1A1 < 0); println(0x8EA2A1A1); println(0x7FFFFFFF < 0x8EA2A1A1); println((-1) % 256) }'
# true
# -1901944415
# false
# -1
```

`Int` literals above `0x7FFFFFFF` are negative. A table sorted by unsigned byte order is **not** sorted for signed `Int` comparison — compare through `UInt` (`reinterpret_as_uint`) or store as `Int64`/`UInt64`. `%` preserves the sign of the left operand; normalize `(a - b) mod 256` style expressions into `0..255` before converting to `Byte`. For byte-codec arithmetic that multiplies by large radices, promote `Byte` to `UInt64`; `UInt64::to_int()` truncates to signed 32 bits for large values.

```sh
moon run -c 'fn rotr64(value : UInt64, bits : Int) -> UInt64 { (value >> bits) | (value << (64 - bits)) }
fn main { let value : UInt64 = 0x0123456789abcdef; let full : UInt64 = UInt64::lnot(0); println(rotr64(value, 8).to_string()); println((full + (1 : UInt64)).to_string()); println(((0x80 : UInt64) >> 7).to_string()) }'
# 17222085231038278605
# 0
# 1
```

`UInt64` arithmetic wraps modulo 2^64. Unsigned right shifts are logical. Provide explicit `UInt64` context (`x : UInt64`) when the literal exceeds signed `Int` range.

```sh
moon run -c 'fn main { println(Int::lnot(0)); println((0x0F).lnot()); println((!true).to_string()) }'
# -1
# -16
# false
```

Bitwise complement: `Int::lnot(value)` or `value.lnot()`. Boolean negation: `!expr`. Do not port OCaml `not x` literally.

## Floats

```sh
ocaml -noprompt -noinit <<'EOF'
print_endline (string_of_float 2.);;
print_endline (string_of_float infinity);;
EOF
# 2.
# inf
moon run -c 'fn main raise{ println((2.0).to_string()); println((1.0 / 0.0).to_string()); let tiny : Double =  @string.from_str("1e-10"); println(tiny.to_string()) }'
# 2
# Infinity
# 1e-10
```

`Double::to_string()` is **not** a drop-in for OCaml `string_of_float` or `Printf "%.12g"`. Whole floats render as `2` (not `2.`); infinities render as `Infinity`/`-Infinity` (not `inf`/`-inf`); small values use scientific notation. When a digest, file format, or snapshot depends on the exact spelling, port the OCaml formatter as an explicit boundary function. Overflow produces `Infinity`; division by infinity produces `0`. Cover those edges with a probe when porting defensive `float` branches.

```sh
moon run -c 'fn main { println(@math.pow(0.5, 2.0)); println((4.0).sqrt()); println((-3.0).abs()); println((5.5).mod(2.0)) }'
# 0.25
# 2
# 3
# 1.5
```

There is no `**` operator — use `@math.pow(base, exponent)`. Common helpers are `Double` methods: `value.sqrt()`, `value.abs()`, `value.mod(other)`. Free functions live in `@math`: `sin`, `cos`, `atan2`, `ln`, `log10`, `floor`, `ceil`, `round`, `trunc`, `exp`. Note `@math.sqrt`/`@math.abs` are **not** valid — those are method-only. Probe with `moon ide doc @math` before assuming a path.

## Data Structures

Map OCaml variants to MoonBit `enum`. Derive `Debug`, `Eq`, and `ToJson` for types that will be inspected in tests.

```sh
moon run -c 'enum E { A(Int) } derive(Debug)
fn E::value(self : E) -> Int { match self { A(n) => n } }
fn main { let value = A(7); println(value.value()) }'
# 7
```

```sh
moon run -c 'enum E { A(Int); B(Int) } derive(Debug)
fn main { let xs : Array[E] = [E::A(1), E::B(2)]; println(xs.length()) }'
# 2
```

When code crosses package boundaries or builds heterogeneous-looking enum arrays, prefer explicit `Type::Constructor` forms or a concrete element-type annotation. This keeps `moon check --warn-list +73` useful — strip annotations only when the compiler reports them as truly unnecessary.

```sh
moon run -c 'priv enum Section { NoneSection; ActiveSection }
fn choose(flag : Bool) -> Section { if flag { ActiveSection } else { NoneSection } }
fn main { match choose(true) { ActiveSection => println("active"); NoneSection => println("none") } }'
# active
```

```sh
moon run -c 'priv struct S { value : Int }
fn make() -> S { { value: 1 } }
fn main { let s = make(); println(s.value) }'
# 1
```

Use `priv enum`/`priv struct` for internal helpers that should not appear in the package interface. Avoid deriving traits on private helpers unless a debug path actually uses the implementation.

```sh
moon run -c 'struct S { a : Int; b : Int? } derive(Debug, Eq)
fn main { let s = { a: 1, b: None }; let t = { ..s, b: Some(2) }; match t.b { Some(value) => println(value); None => println(0) } }'
# 2
```

Record update is `{ ..old, field: value }`, the direct replacement for OCaml `{ old with field = value }`.

```sh
moon run -c 'struct S { values : Int } derive(Debug)
fn make(values : Int) -> S? { Some({ values, }) }
fn main { println(make(7).unwrap().values) }'
# 7
```

For single-field struct literals using field shorthand inside another expression, write a trailing comma (`{ values, }`) to silence the ambiguous-block warning.

When an OCaml function returns a broad variant and later code relies on an informal invariant, introduce a narrow private MoonBit enum for the post-checked state. This makes impossible fallback branches explicit at the type boundary rather than carrying broad `Object`-like values through the rest of the port.

## Lookup Tables

```sh
moon run -c 'fn range_lookup(ranges : ArrayView[(Int, Int, Int)], key : Int) -> Int? { for range in ranges { if key >= range.0 && key <= range.1 { return Some(range.2 + key - range.0) } }; None }
fn main { let ranges = [(0x21, 0x23, 100), (0x30, 0x30, 200)]; println(range_lookup(ranges, 0x22).unwrap()); println(range_lookup(ranges, 0x24) is None) }'
# 101
# true
```

For large OCaml mapping tables, prefer compact `ArrayView`-accepted range tables (`(first, last, base)` plus small exception/sequence tables) over mechanically expanding every entry. Generated MoonBit sources stay smaller and native-target validation runs faster, with the same deterministic lookup behavior.

```sh
moon run -c $'fn table_entry(i : Int) -> Int? { match i { 65 => Some(1); _ => None } }\nlet table : FixedArray[Int?] = FixedArray::makei(128, i => table_entry(i))\nfn lookup(byte : Int) -> Int? { if byte >= 0 && byte < table.length() { table[byte] } else { None } }\nfn main { println(lookup(65).unwrap()); println(lookup(66) == None) }'
# 1
# true
```

For small dense lookup domains hit repeatedly, build a private top-level `FixedArray[T?]` with `FixedArray::makei`. Keep the bounds check at the lookup boundary.

## Mutation, Refs, Arrays

```sh
moon run -c $'fn first_sorted(xs : ArrayView[Int]) -> Int { let ys = xs.to_owned(); ys.sort(); ys[0] }\nfn main { let r : Ref[Int] = Ref::{ val: 0 }; r.val += 1; println(r.val); let fixed : FixedArray[Int] = [3, 1, 2]; let grow = [4, 2, 3]; println(first_sorted(fixed)); println(first_sorted(grow)); println(fixed[0]); println(grow[0]) }'
# 1
# 1
# 2
# 3
# 4
```

`Ref[T]` for primitive mutability; `mut` fields inside structs for larger state. `Array` is a growable mutable vector; `FixedArray` matches OCaml `array` more directly. `ArrayView[T]` is the read-only sequence parameter type — callers can pass `Array`, `FixedArray`, or another view without copying. `.to_owned()` materializes a mutable `Array` at explicit ownership boundaries (e.g. before `sort()`/`sort_by()`); avoid it in hot paths. Represent lazy/deferred states as explicit `enum` variants instead of nested mutable containers.

```sh
moon run -c 'fn main { let xs = [(2, "b"), (1, "a")]; xs.sort_by(fn(a, b) { a.0.compare(b.0) }); println(xs[0].0); let ys = [1, 2, 3]; let zs = ys.rev(); println(zs[0]); println(ys[0]) }'
# 1
# 3
# 1
```

`Array::sort_by` sorts in place with an `Int`-returning comparator. `Array::rev()` returns a reversed **copy** and leaves the original unchanged — useful when porting OCaml list-building code that conses in reverse and conditionally applies `List.rev`.

Use ordered arrays of pairs when insertion order or stable rendering matters; use `@hashmap.HashMap[K, V]` (from `moonbitlang/core/hashmap`) for unordered lookup once the import is in `moon.pkg`.

## Errors

OCaml exceptions (`Not_found`, `End_of_file`, `Invalid_argument`, domain errors) should not be copied as unchecked control flow.

```sh
moon run -c 'suberror E
fn make(flag : Bool) -> ((Int) -> Int?) raise E { if flag { raise E } else { fn(x) { Some(x) } } }
fn main { println((try! make(false))(3).unwrap()) }'
# 3
```

```sh
moon run -c $'fn parse(s : String) -> Int raise { @strconv.from_str(s) }\nfn helper() -> Int raise { parse("7") }\ntest "raising test body can propagate" { let value = parse("7"); @test.assert_eq(value, 7) }\nfn main { println((try! helper()).to_string()) }'
# 7
```

Define a project-level `suberror` once the first fallible functions are ported. Fallible functions declare `raise ProjectError` or plain `raise`. Do **not** add explicit `try!` around every fallible call inside a raising function, test helper, or `test` body — checked errors propagate from those contexts automatically.

```sh
moon run -c 'fn helper(flag : Bool) -> Unit raise { if flag { fail("x") } }
fn main { try! helper(false); println("helper uses raise") }'
# helper uses raise
```

Test helpers that can call `fail` must declare `raise`, even when only called from `test` blocks.

When a raising function returns another function, parenthesize the function type before `raise`: `-> ((A) -> B) raise E`, **not** `-> (A) -> B raise E` (the latter binds `raise` to the returned function type).

```sh
moon run -c $'fn may_fail(flag : Bool) -> Unit raise { if flag { fail("boom") } }\nfn main { let result : Result[Unit, Error] = try? may_fail(true); println((result is Err(_)).to_string()) }'
# true
```

```sh
moon run -c $'fn parse(s : String) -> Int raise { @strconv.from_str(s) }\nfn main {\n  try parse("7") catch {\n    _ => println(0)\n  } noraise {\n    value => println(value)\n  }\n}'
# 7
```

`try? expr` materializes failure as `Result[T, Error]` — convenient in tests, but loses the narrower suberror type for re-raising. `try expr catch { ... } noraise { value => ... }` is the OCaml `try ... with ...` analogue with explicit success and error branches.

To catch one constructor and propagate the rest while keeping the project error type, use `expr catch { SpecificError => fallback; error => raise error }`. `catch` arms are checked for exhaustiveness; do not omit the final propagating arm. When porting OCaml exception tests, default to `Err(_)`; match a specific variant only when that variant is itself the contract under test.

Avoid `match (try? f()) { Ok(...) => ...; Err(...) => ... }` — MoonBit warns on it. Prefer `try f() catch { ... } noraise { ... }`. If you must match a `try?` result, wrap it in parentheses; `match try? expr { ... }` is invalid syntax.

## Functions and Callbacks

```sh
moon run -c 'fn helper(x : Int) -> Int { x + 1 }
fn main { println(helper(1)) }'
# 2
```

Top-level helper functions (anything other than `main`) need explicit return-type annotations. This applies to `moon run -c` probes too.

```sh
moon run -c 'fn[A, B, C] apply_pair(f : (A, B) -> C, a : A, b : B) -> (C, A) { (f(a, b), a) }
fn main { let result = apply_pair(fn(x, y) { x + y }, 2, 3); println(result.0); println(result.1) }'
# 5
# 2
```

Polymorphic functions write type parameters **before** the name: `fn[A, B, C] name(...)`. The older `fn name[A, B, C](...)` spelling is deprecated.

```sh
moon run -c 'fn apply_twice(f : (Int) -> Int, value : Int) -> Int { f(f(value)) }
fn main { println(apply_twice(fn(x) { x + 1 }, 40)) }'
# 42
```

Callback parameters use function types like `(Int) -> Int`; callback literals are `fn(x) { ... }`. Capturing mutable `Ref` state in a closure is a direct port of OCaml closures over refs.

```sh
moon run -c 'fn apply(f : (Int) -> Int raise, x : Int) -> Int raise { f(x) }
fn main { println(try! apply(fn(x) raise { if x == 0 { fail("zero") }; x + 1 }, 1)) }'
# 2
```

Raising callbacks must include `raise` in the parameter type and in the literal: `(Int) -> Int raise` and `fn(x) raise { ... }`. Inferred raising effects on `fn` literals are deprecated. For a narrower project error type, annotate: `fn(x) raise ProjectError { ... }`.

```sh
moon run -c $'fn apply(f : (Int) -> Int raise Error) -> Int raise Error { f(1) }\nfn main { let result : Result[Int, Error] = try? apply(fn(x) raise Error { if x > 0 { fail("x") } else { x } }); println(result is Err(_)) }'
# true
```

Raising and non-raising callback types are **not** interchangeable. A raising closure cannot satisfy a non-raising slot, and vice versa. When porting OCaml callbacks whose implementation may later touch fallible APIs (filesystem, async, random), declare `raise` in the type from the start; do not hide failure inside the callback.

## Labelled and Default Arguments

```sh
moon run -c $'fn greet(name? : String = "pdf") -> String { name }\nfn main { println(greet()); println(greet(name="moon")) }'
# pdf
# moon
```

Defaults attach to **labelled** parameters: `name? : T = default`. Call sites pass `name=value`. Do not place a default on an unlabelled positional parameter.

```sh
moon run -c 'fn inner(flag? : Bool = false) -> Bool { flag }
fn outer(flag? : Bool = false) -> Bool { inner(flag~) }
fn main { println(outer()); println(outer(flag=true)) }'
# false
# true
```

`label~` forwards the current value of a same-named labelled argument through wrapper layers — the MoonBit counterpart to preserving OCaml optional-argument semantics across helpers.

```sh
moon run -c 'fn takes_view(xs : ArrayView[Int]) -> Int { xs.length() }
fn sample(xs? : ArrayView[Int] = []) -> Int { takes_view(xs) }
fn main { println(sample()); println(sample(xs=[1, 2, 3])) }'
# 0
# 3
```

Optional view parameters can default to `[]`. Optional-only arguments must be passed by label (`sample(xs=[1, 2, 3])`, not `sample([1, 2, 3])`).

```sh
moon run -c 'fn f(a : Int, flag? : Bool = true, xs? : ArrayView[Int] = []) -> Int { if flag { a + xs.length() } else { a - xs.length() } }
fn main { println(f(3, xs=[1, 2], flag=false)); println(f(3, flag=true)); println(f(3)) }'
# 1
# 3
# 3
```

After required positional arguments, multiple optionals can be supplied by label in any order.

```sh
moon run -c 'fn f(x? : Int? = None) -> Int { match x { Some(v) => v; None => -1 } }
fn g(x? : Int? = None) -> Int { f(x~) }
fn main { println(g()); println(g(x=Some(7))) }'
# -1
# 7
```

Optional labelled arguments can themselves be `Option`-typed — use this when porting OCaml `string option` where "absent" and "explicitly empty" must remain distinct. `label~` forwarding works through `Option` too. Default values can call local functions, useful for structured-record or array defaults.

## Pattern Matching

```sh
moon run -c 'enum E { A(Int); B }
fn f(e : E) -> Int { match e { A(n) if n > 0 => n; _ => 0 } }
fn main { println(f(A(3))); println(f(A(-1))); println(f(B)) }'
# 3
# 0
# 0
```

Pattern guards belong on the same arm: `Pattern if condition => ...`. Do not split the `if` onto its own line.

Match arms run **top to bottom**. Put specific tuple or variant cases before broad wildcard or guarded catch-all arms — a guarded broad arm can make a later specific case behaviorally dead, and the compiler does not always catch that.

```sh
moon run -c 'fn main { let xs = [1, 2, 3]; match xs { [_, .. rest] => { let owned = [ for x in rest => x ]; println(rest.length()); println(owned.length()); println(owned[0]) }; _ => () } }'
# 2
# 2
# 2
```

Array rest patterns (`.. rest`) bind read-only views. If an enum payload or API needs an owned `Array[T]`, copy with a comprehension: `[ for x in rest => x ]`.

## Loops and Comprehensions

```sh
moon run -c 'fn main { let fixed : FixedArray[Int] = [1, 2, 3]; let doubled = [ for x in fixed => x * 2 ]; println(doubled.length()); println(doubled[2]) }'
# 3
# 6
```

```sh
moon run -c 'fn main { let pairs = [(1, 2), (3, 4)]; let firsts = [ for pair in pairs => pair.0 ]; println(firsts[0]); println(firsts[1]) }'
# 1
# 3
```

Comprehension syntax: `[ for x in xs => expr ]`. Use a simple identifier as the binder and destructure tuples or access fields inside the body.

```sh
moon run -c $'fn parse(s : String) -> Int raise { @strconv.from_str(s) }\nfn main { let _ = [ for s in ["1"] => parse(s) ]; println("done") }'
# Error: calling function with error is not allowed inside list comprehension.
```

Comprehension bodies **cannot** call error-raising functions. Port OCaml `List.map`/`Array.map` with raising mappers as an explicit loop in a raising function: push each result into an output array and let the error propagate.

```sh
moon run -c 'fn main { let map : @hashmap.HashMap[Int, Int] = @hashmap.HashMap::new(); map[1] = 10; map[2] = 20; let mut total = 0; for key in map.keys() { total += key }; println(total) }'
# 3
```

`for x in xs` iterates any iterable, not just arrays/views. Do not convert iterables to `Array` just to loop; reserve `.to_array()`/`.to_owned()` for cases that need an owned snapshot, sorting, indexing, or mutation.

```sh
moon run -c 'fn first_positive(xs : Array[Int]) -> Int? { let mut i = 0; while i < xs.length() { if xs[i] > 0 { break Some(xs[i]) }; i += 1 } nobreak { None } }
fn main { println(first_positive([-2, 0, 7]).unwrap()); println(first_positive([-2, 0]) is None) }'
# 7
# true
```

`while` loops may produce a value with `break value`; the branch when the condition becomes false is `nobreak { ... }`. The older `else` spelling is deprecated. Avoid the older functional `loop ... { ... }` form in new ports — MoonBit warns on it.

## Surface Hazards

- `alias` is a reserved keyword; avoid it as a local name, loop binder, or helper name to keep warning-enabled checks clean.
- On the JavaScript backend, also avoid sentinel names like `undefined` for local test results — cross-target package tests can expose generated-JS name collisions.
- Source files do **not** contain OCaml-style `open`. Add package imports in `moon.pkg`, then call imported packages with their `@alias`.
- For `moon run -c` snippets that need extra packages, use an **MBTX import block** at the start: `import { "moonbitlang/x/crypto" }` (comma-separated entries for multiple packages). The source-file form `import "moonbitlang/core/encoding/ascii"` is rejected in command snippets.

## Process and Random

`@env` (in `moonbitlang/core/env`) provides `args() -> Array[String]`, `current_dir() -> String?`, `get_env_var(String) -> String?`, `get_env_vars() -> Map[String, String]`, `now() -> UInt64`, `set_env_var`, `unset_env_var`. `current_dir()` is optional — some platforms cannot always determine it. Prefer `x is Some(_)` over the deprecated `.is_some()`.

```sh
moon run -c 'fn main { let r = @random.Rand::chacha8(seed=@ascii.encode("01234567890123456789012345678901")); println(r.uint(limit=256).to_string()); println(r.uint(limit=256).to_string()) }'
# 21
# 42
```

`@random` provides deterministic PRNGs. `Rand::new()` is **reproducible**, not OS entropy. Do not substitute `@random` or `@env.now()` for a cryptographic random source when porting security-sensitive OCaml code (AES IVs, salts, file keys); keep an explicit random provider or use a target-specific secure API.

## Targets and Async I/O

```sh
moon run --target native -c $'#cfg(target="native")\nfn target_name() -> String { "native" }\n#cfg(not(target="native"))\nfn target_name() -> String { "other" }\nfn main { println(target_name()) }'
# native
```

`#cfg(...)` on its own line immediately before a declaration provides target-specific implementations. Combining `#cfg` and the declaration on the same line leaves the attribute unused.

MoonBit async has no `await` keyword — async functions call other async functions directly.

- Keep CPU-bound pure transformations over `Bytes` synchronous.
- Make filesystem/network entry points async when they touch `moonbitlang/async`.
- Prefer async wrappers that load file contents into `Bytes`, then call the synchronous core. This avoids making every recursive helper async.
- Async entry points and async tests need `moonbitlang/async` in the relevant `moon.pkg`; syntax alone is not enough.

```sh
moon run --target native -c 'async fn main { println("async ok") }'
# Error: Cannot use `async fn main`: package moonbitlang/async is not imported.
```

```sh
moon run --target native -c $'import {\n  "moonbitlang/async",\n  "moonbitlang/async/fs" @fs,\n}\nasync fn main {\n  let result = try? @fs.read_file("/tmp/definitely-missing-fixture")\n  println(result is Err(_))\n}'
# true
```

If a port should keep its pure core available on non-native targets, put async/file-system wrappers in a native-only package rather than importing async into the core. Imports are package-level; `supported_targets = "+native"` controls backend inclusion. A single test file can be target-gated with `options(targets: { "file_test.mbt": [ "native" ] })`, but imports remain package-level and may report unused on non-native checks.

## Testing

Test file roles:

- `*_test.mbt`: black-box tests. Call only public APIs through the package alias.
- `*_wbtest.mbt`: white-box tests. Run inside the package; may test private helpers.
- `*.mbt.md`: documentation with checked code blocks. `mbt check` for code that must compile and test; `mbt nocheck` for illustrative-only snippets.

Assertion style:

- `@test.assert_eq` for stable scalar or structural results (`Debug` bound; avoids the deprecated `Show`-based path).
- Boolean assertions: `@test.assert_eq(condition, true)` — there is no `@test.assert_true` helper.
- `@test.assert_eq` accepts a named `msg=` argument for shared assertion helpers.
- Shared test helpers that call `@test.assert_eq`, `fail`, or fallible APIs should declare `-> Unit raise Error` unless they use a narrower project error type.
- Pattern checks: `assert_true(value is Pattern(...))` or `guard ... else { fail(...) }`.
- Snapshots: `inspect(value, content="...")` for small values; `json_inspect(value, content=...)` for complex values with `ToJson`. If the expected snapshot is unknown, write `inspect(value)`, run `moon test --update`, then review the diff.
- Success paths through raising functions: call the fallible API directly and let the test fail on any error. Expected failures: `let result : Result[T, Error] = try? f()` and assert or inspect the result.
- Use `Err(_)` for "this should raise"; match a specific error variant only when that variant is itself part of the compatibility contract.

```sh
moon run -c 'fn helper() -> Unit raise Error { @test.assert_eq(true, true); @test.assert_eq([1, 2], [1, 2]) }
fn main { try! helper() }'
```

```sh
moon run -c 'import { "moonbitlang/core/test" }
fn helper() -> Unit raise { @test.assert_eq(true, true, msg="named assertion") }
fn main { try! helper(); println("ok") }'
# ok
```

For parser/serializer, encoder/decoder, or loader/writer pairs, pair focused edge tests with at least one public-API round-trip test. Round-trips catch ownership, byte/text, and object-boundary mistakes that isolated unit tests miss.

Cover the byte/text edges that OCaml callers depended on: non-ASCII, NUL, form-feed, high-bit bytes, integer overflow, empty input, and boundary offsets.

## Command Cheatsheet

```sh
moon run -c 'fn main { ... }'              # quick language/API probe
moon check --warn-list +73                 # fast type check with extra warnings
moon test --outline                        # list discovered tests
moon test                                  # run all tests
moon test path/to/file_test.mbt            # run one test file
moon test package/dir --filter 'glob'      # targeted test glob
moon test --update                         # refresh snapshots, then review diff
moon test --target native                  # required for async I/O tests
moon coverage analyze > uncovered.log      # coverage report
moon info && moon fmt                      # final interface update and format
```

Validation rule for each migration patch:

1. Add or update tests before or with the ported code.
2. Run targeted `moon test` while developing.
3. Finish with `moon check --warn-list +73`, `moon test`, `moon info`, and `moon fmt`.

## Update Discipline

When a migration teaches a reusable rule that contradicts or refines this guide, update this file with:

- The OCaml behavior being replaced.
- The MoonBit API or idiom chosen.
- A minimal `moon run -c` probe and its observed output.
- Any known incompatibility, target limitation, or deferred behavior.
