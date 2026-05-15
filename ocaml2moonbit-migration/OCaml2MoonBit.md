# OCaml to MoonBit Migration Guide

This document is a library-agnostic guide for transferring OCaml programs to
MoonBit. Update it whenever a reusable porting rule, language fact, API choice,
or test pattern is verified. Project-specific architecture and migration plans
belong in their own project plan documents.

## Verified MoonBit Facts

Verified with `moon 0.1.20260430` and OCaml 4.14.1.

The key byte/text contrast was validated on both sides:

```sh
ocaml -noprompt -noinit <<'EOF'
let s = "𝄞";;
Printf.printf "%d\n" (String.length s);;
Printf.printf "%d\n" (Char.code s.[0]);;
EOF
# 4
# 240
```

OCaml `String.length` counts bytes. The first byte of the UTF-8 spelling of
`𝄞` is 240.

Run small language/API probes with `moon run -c '...'` before committing a
porting pattern to the codebase. For MBTX snippets that need packages beyond
the default loaded set, put a MoonBit package import block at the beginning of
the snippet:

```sh
moon run -c $'import { "moonbitlang/x/crypto" }\nfn main { let digest = @crypto.sha256(b"abc"); println(digest.length().to_string()) }'
# 32
```

Use the imported package alias, for example `@crypto`; do not assume a
fully-qualified package path works inside the snippet without an import. The
MBTX import block form is required for `moon run -c`; a source-file-style
single package import such as `import "moonbitlang/core/encoding/ascii"` is
rejected in command snippets.

Useful verified probes:

```sh
moon run -c 'fn main { let s = "𝄞"; println(s.length()); println(s.char_length()) }'
# 2
# 1
```

MoonBit `String::length()` counts UTF-16 code units, not bytes and not Unicode
scalar values. `String::char_length()` counts Unicode characters.

```sh
moon run -c 'fn main { let s = "--2.5"; println(s.has_prefix("--")); println(s.unsafe_substring(start=1, end=s.length())) }'
# true
# -2.5
```

MoonBit strings provide prefix helpers such as `String::has_prefix` and owned
substring extraction with `String::unsafe_substring(start=..., end=...)`.
Remember that string offsets are UTF-16 code-unit offsets. Use this only for
validated ASCII grammar tokens or text-level parsing; use `BytesView` slices
for byte-oriented formats.

```sh
moon run -c 'fn main { let s = "/UniJIS-UCS2-H"; println(s.contains("-UCS2-")); println(s.has_suffix("-H")) }'
# true
# true
```

MoonBit strings also provide substring and suffix predicates such as
`String::contains` and `String::has_suffix`. These are convenient for
ASCII-only symbolic names or command flags. Do not use string predicates to
classify arbitrary binary format bytes; stay in `Bytes`/`BytesView` for that.

```sh
moon run -c 'fn main { let raw : Array[Byte] = [65, 0, 255]; let b = Bytes::from_array(raw); println(b.length()); println(b[2].to_int()) }'
# 3
# 255
```

Use `Bytes`/`Byte` for byte-addressed data. Integer literals can be
disambiguated by type context, so `Array[Byte] = [65, 0, 255]` is valid.
When calling a method directly on an integer literal, parenthesize the literal:
use `(65).to_byte()`, not `65.to_byte()`, because `65.` is parsed as the start
of a floating-point literal.

```sh
moon run -c 'fn main { let source : Array[Byte] = [1, 2]; let bytes = Bytes::from_array(source); source[0] = 9; println(bytes[0].to_int()); println(source[0].to_int()) }'
# 1
# 9
```

`Bytes::from_array` creates an immutable owned byte value. Later mutation of the
source `Array[Byte]` does not mutate the `Bytes`. Port OCaml byte-mutating
helpers such as `bytes_selfmap` either as a new `BytesView -> Bytes` transform,
or keep data in a mutable `Array[Byte]`/`FixedArray[Byte]` until the final
freeze.

```sh
moon run -c 'enum E { A(Int) } derive(Debug)
fn E::value(self : E) -> Int { match self { A(n) => n } }
fn main { let value = A(7); println(value.value()) }'
# 7
```

When calling a method on a freshly constructed enum value in package code, bind
the constructor result to a local first, then call the method. In black-box
tests or other places where constructor names come from an imported package,
add an explicit enum type annotation if inference has no expected type yet.
Parentheses can work in a one-off snippet, but `moon fmt` may remove them in
source files and reintroduce parser/name-resolution ambiguity.

```sh
moon run -c 'enum E { A(Int); B(Int) } derive(Debug)
fn main { let xs : Array[E] = [E::A(1), E::B(2)]; println(xs.length()) }'
# 2
```

When ported code crosses package boundaries or builds heterogeneous-looking
enum arrays, prefer explicit `Type::Constructor` forms or a concrete element
type annotation over relying on late constructor inference. This keeps
warning-enabled checks such as `moon check --warn-list +73` useful: remove
package annotations only when the compiler says they are truly unnecessary.

```sh
moon run -c 'priv enum Section { NoneSection; ActiveSection }
fn choose(flag : Bool) -> Section { if flag { ActiveSection } else { NoneSection } }
fn main { match choose(true) { ActiveSection => println("active"); NoneSection => println("none") } }'
# active
```

Use `priv enum` for internal helper variants that should not appear in the
package interface. This avoids visibility warnings when `moon check
--warn-list +73` or the project warning profile complains that a helper type is
not part of the public signature. In one-line probes, separate enum variants
with semicolons; in source files, normal block-style variant lines work after
`moon fmt`.

```sh
moon run -c 'priv struct S { value : Int }
fn make() -> S { { value: 1 } }
fn main { let s = make(); println(s.value) }'
# 1
```

Use `priv struct` for internal helper records that should not be exposed in the
generated package interface. Avoid deriving traits on private helpers unless a
test or debug path actually uses those implementations, because unused derives
can produce warnings under stricter project checks.

```sh
moon run -c 'struct S { a : Int; b : Int? } derive(Debug, Eq)
fn main { let s = { a: 1, b: None }; let t = { ..s, b: Some(2) }; match t.b { Some(value) => println(value); None => println(0) } }'
# 2
```

MoonBit supports record update syntax with `{ ..old_record, field: value }`.
Use this when a port needs to preserve most fields while overlaying a parsed or
validated value. Adding a field to a public struct is an API change: update all
record literals and run `moon info` so the generated `.mbti` interface records
the new field deliberately.

When an OCaml function returns a broad variant and later code relies on an
informal invariant, prefer a narrow private MoonBit enum for the post-checked
state. This makes impossible fallback branches explicit at the type boundary
instead of carrying broad `Object`-like values through the rest of the port.

```sh
moon run -c 'fn helper(x : Int) -> Int { x + 1 }
fn main { println(helper(1)) }'
# 2
```

Top-level helper functions other than `main` need explicit return type
annotations. This applies to quick `moon run -c` probes as well as package
source. Annotate helpers such as `fn helper(...) -> T { ... }` instead of
relying on return inference.

```sh
moon run -c 'suberror E
fn make(flag : Bool) -> ((Int) -> Int?) raise E { if flag { raise E } else { fn(x) { Some(x) } } }
fn main { println((try! make(false))(3).unwrap()) }'
# 3
```

When a MoonBit function can raise and returns another function, parenthesize the
returned function type before the `raise` clause: use
`-> ((A) -> B) raise E`, not `-> (A) -> B raise E`. Without the parentheses,
the `raise` clause belongs to the returned function type, which is not what
OCaml callback-returning APIs usually mean.

```sh
moon run -c 'fn main { let empty = Bytes::new(0); let zeros = Bytes::new(2); println(empty.length()); println(zeros.length()); println(zeros[0].to_int()) }'
# 0
# 2
# 0
```

`Bytes::new(length)` requires an explicit length and creates zero-filled bytes.
Use `Bytes::new(0)` or a typed empty byte builder when porting OCaml code that
used `Bytes.empty` or an empty byte string.

```sh
moon run -c 'fn main { let bytes = @ascii.encode("abc"); let copied = Bytes::makei(bytes.length(), i => bytes[i]); println(bytes == copied); println(physical_equal(bytes, copied)) }'
# true
# false
```

`Bytes::copy` is deprecated because `Bytes` is immutable. When a port needs a
deliberate physical byte copy, for example to preserve an OCaml API that
promises no shared stream buffers, use `Bytes::makei(bytes.length(), i =>
bytes[i])` at that ownership boundary.

```sh
moon run -c 'fn main { let b = @ascii.encode("PDF"); println(b.length()); println(b[0].to_int()) }'
# 3
# 80
```

Use ASCII encoding helpers for format syntax that is defined as ASCII bytes,
then append binary payloads as `Bytes`/`BytesView`. Do not build binary file
formats through `String` concatenation.

```sh
moon run -c $'import { "moonbitlang/core/encoding/ascii" }\nfn ascii_bytes(text : String) -> Bytes { @ascii.encode(text) }\nlet cached_filter : Bytes = ascii_bytes("/Filter")\nfn main { println(cached_filter.length()); println(cached_filter[0].to_int()) }'
# 7
# 47
```

For frequently reused ASCII grammar tokens, a private top-level `let` can cache
the encoded `Bytes` value once. This is a useful replacement for OCaml code
that repeatedly compares byte-oriented string constants in hot paths.
Give cached top-level values explicit type annotations; command snippets and
package-level declarations otherwise may not have enough expected type context
for inference.

```sh
moon run -c "fn make() -> BytesView { let b = Bytes::from_array([b'a']); b[:] }\nfn main { println(make().length()) }" --target native
# 1
```

A `BytesView` returned from a function can keep the newly allocated backing
`Bytes` alive. This is useful when a read-only API must expose the result of a
decode/decrypt operation as a view. The allocation has already happened,
though, so prefer direct borrowed views for raw slices and call `.to_owned()` or
other allocating transforms only at explicit ownership or transformation
boundaries.

```sh
moon run -c 'fn main { println(@env.current_dir() is Some(_)); println(@env.now() > 0UL) }'
# true
# true
```

`moonbitlang/core/env` provides small process-environment helpers such as
`@env.current_dir() -> String?` and `@env.now() -> UInt64`. Prefer these over
hard-coded paths or fixed timestamp-like names in tests and command-facing
wrappers. Match options with `value is Some(_)` in probes and tests; `.is_some()`
is deprecated in the current toolchain.

```sh
moon run -c 'fn main { let r = @random.Rand::new(); println(r.uint64().to_string()); let s = @random.Rand::new(); println(s.uint64().to_string()) }'
# 13219109469176600229
# 13219109469176600229
```

`moonbitlang/core/random` provides deterministic PRNGs such as `@random.Rand`.
The default `Rand::new()` sequence is reproducible, not an operating-system
entropy source. Use it for deterministic fixtures and simulations; do not
treat it as a secure random provider when porting cryptographic OCaml code
unless the seed/source is supplied from an appropriate entropy boundary.

```sh
moon run -c 'fn helper(flag : Bool) -> Unit raise { if flag { fail("x") } }
fn main { try! helper(false); println("helper uses raise") }'
# helper uses raise
```

Test helpers that can call `fail` must declare `raise`, even when they are only
called from `test` blocks. Keep this on shared assertion helpers so pattern
checks can be factored without losing checked-error correctness.

```sh
moon run -c $'fn parse(s : String) -> Int raise { @strconv.from_str(s) }\nfn helper() -> Int raise { parse("7") }\ntest "raising test body can propagate" { let value = parse("7"); @test.assert_eq(value, 7) }\nfn main { println((try! helper()).to_string()) }'
# 7
```

Do not add explicit `try!` around every fallible call inside a raising function,
test helper, or `test` body. MoonBit propagates checked errors from those
contexts directly. Use `try?` for tests that intentionally inspect an error
result, and use `try ... catch ... noraise` when success and error branches are
both part of the expression.

```sh
moon run -c $'fn may_fail(flag : Bool) -> Unit raise { if flag { fail("boom") } }\nfn main { let result : Result[Unit, Error] = try? may_fail(true); println((result is Err(_)).to_string()) }'
# true
```

When porting OCaml exception tests, do not assert the exact MoonBit error
constructor by default. Use `Err(_)` for ordinary "this should raise" behavior,
and match a specific error variant only when that variant is itself part of the
compatibility contract or a caller-visible API guarantee.

```sh
moon run --target native -c $'#cfg(target="native")\nfn target_name() -> String { "native" }\n#cfg(not(target="native"))\nfn target_name() -> String { "other" }\nfn main { println(target_name()) }'
# native
```

Use `#cfg(...)` for target-specific declarations when a port needs a native
implementation plus a portable fallback. Put the attribute on its own line
immediately before the declaration; placing `#cfg` and the declaration on the
same line can leave the attribute unused. This is useful for OCaml ports that
replace a C stub or Unix-only API on the native target while retaining a pure
MoonBit implementation for other backends.

```sh
moon run -c $'fn push_ascii(output : Array[Byte], text : String) -> Unit { for byte in @ascii.encode(text)[:] { output.push(byte) } }\nfn main { let output : Array[Byte] = []; push_ascii(output, "PDF"); output.push(10); let bytes = Bytes::from_array(output); println(bytes.length()); println(bytes[0].to_int()); println(bytes[3].to_int()) }'
# 4
# 80
# 10
```

For binary writers, use `Array[Byte]` as the mutable output builder. Append
ASCII format syntax by iterating over `@ascii.encode(text)[:]`, append binary
payloads from `BytesView`, and call `Bytes::from_array` once at the ownership
boundary. This is the usual MoonBit replacement for OCaml code that used
`Buffer`, `Bytes`, or byte-oriented `string` concatenation to construct output.

```sh
moon run -c 'fn main { let (b, u, u16, i64, u64) : (Byte, UInt, UInt16, Int64, UInt64) = (255, 7, 65535, 42, 42); println(b.to_int()); println(u.to_string()); println(u16.to_int()); println(i64.to_string()); println(u64.to_string()) }'
# 255
# 7
# 65535
# 42
# 42
```

MoonBit has useful scalar types for ports from OCaml: `Byte`, `Int16`,
`UInt16`, `Int`, `UInt`, `Int64`, `UInt64`, `Float`, and `Double`. In the
current toolchain there is no `Int32` type; use `Int` when deliberate 32-bit
wrapping is wanted, or `Int64`/`UInt64` when an OCaml `int32` value needs to be
represented without signed-32-bit truncation.

```sh
moon run -c 'fn main { let b : Byte = 255; let x : UInt64 = b.to_uint64(); println(x.to_string()); println((x / 85).to_string()); println((x % 85).to_byte().to_int()) }'
# 255
# 3
# 0
```

For byte-codec arithmetic, promote `Byte` to `UInt64` before multiplying by
large radices. Convert back only after reducing the value into byte range,
because `UInt64::to_int()` truncates to 32 signed bits for large values.

```sh
moon run -c 'fn main { println((-1) % 256); let normalized = if (-1) % 256 < 0 { ((-1) % 256) + 256 } else { (-1) % 256 }; println(normalized); println(normalized.to_byte().to_int()) }'
# -1
# 255
# 255
```

MoonBit `%` keeps a negative remainder when the left operand is negative. When
porting OCaml byte-codec expressions such as `(a - b) mod 256`, normalize the
remainder into `0..255` before converting to `Byte`.

```sh
moon run -c 'fn main { println(0xF0 & 0x0F); println(0x01 << 7); println(0x80 >> 7); println(0xAA ^ 0xFF) }'
# 0
# 128
# 1
# 85
```

MoonBit uses symbolic integer bitwise operators: `&` for OCaml `land`, `|`
for `lor`, `^` for `lxor`, `<<` for `lsl`, and `>>` for right shift. This is
the direct replacement for byte-codec and bitstream code that masks, packs,
or inverts bytes.

```sh
moon run -c 'fn main { println((-8) >> 1); println(1 << 31); println(1 << 32); println(1 >> 32) }'
# -4
# -2147483648
# 1
# 1
```

MoonBit `Int` shifts are arithmetic for signed right shift and mask the shift
count to the 32-bit word width. Guard or normalize shift counts explicitly when
the OCaml source relied on `Int32.shift_*` preconditions or format-specific
width rules.

```sh
moon run -c 'fn main { let logical = ((-8).reinterpret_as_uint() >> 1).reinterpret_as_int(); println(logical); println((-8) >> 1) }'
# 2147483644
# -4
```

For OCaml `Int32.shift_right_logical`, reinterpret the signed value as `UInt`,
shift, then reinterpret back to `Int`. Numeric conversion is not the same as
bit reinterpretation.

```sh
moon run -c 'fn main { let max = 2147483647; println((max + 1).to_string()); println((-2147483648 - 1).to_string()); println((2147483647 * 2).to_string()); println((1 << 31).to_string()) }'
# -2147483648
# 2147483647
# -2
# -2147483648
```

MoonBit `Int` arithmetic wraps at 32-bit signed boundaries. This is useful
when porting OCaml code that deliberately used `Int32` arithmetic, but it also
means overflow needs explicit tests when the source relied on arbitrary-size or
64-bit integer behavior.

```sh
moon run -c $'fn main { let parsed : Result[Int, Error] = try? @strconv.from_str("2147483648"); println(parsed is Err(_)); let mut value = 0; for digit in [50,49,52,55,52,56,51,54,52,56] { value = value * 10 + digit - 48 }; println(value); let mut wide = 0L; for digit in [50,49,52,55,52,56,51,54,52,56] { wide = wide * 10L + (digit - 48).to_int64() }; println(wide.to_string()) }'
# true
# -2147483648
# 2147483648
```

`@strconv.from_str<Int>` rejects a decimal outside the signed 32-bit range,
but handwritten digit accumulation in `Int` wraps. When porting OCaml parsers
for file offsets, lengths, object numbers, or other serialized integer fields,
accumulate in `Int64`/`UInt64`, check the target range explicitly, and convert
to `Int` only after the bounds check.

```sh
moon run -c 'fn main { let x = 0x81308130; println(x); println(x.reinterpret_as_uint()); println((0x80000000).reinterpret_as_uint()) }'
# -2127527632
# 2167439664
# 2147483648
```

MoonBit `Int` literals and packed byte accumulators above `0x7fffffff` are
negative signed values. When the OCaml source used `int` as an unsigned
32-bit serialized key, either compare through `UInt` using
`.reinterpret_as_uint()` or switch the ported representation to `Int64`/`UInt`
before sorting, binary searching, or exposing values through the public API.

```sh
moon run -c 'fn main { println(0x8EA2A1A1 < 0); println(0x8EA2A1A1); println(0x7FFFFFFF < 0x8EA2A1A1) }'
# true
# -1901944415
# false
```

Packed four-byte keys that set the high bit compare before positive `Int`
values. If a table is sorted by serialized unsigned byte order, it is not
necessarily sorted for MoonBit `Int` binary search. Sort with the same ordering
used by lookup, compare through `UInt`, use `Int64`/`UInt64`, or use a linear
lookup when source order matters for duplicate precedence.

```sh
moon run -c 'fn main { let two = 0x20; let four = 0x8EA2EDA1; println(two < four); println(two.reinterpret_as_uint() < four.reinterpret_as_uint()) }'
# false
# true
```

For binary searches over serialized byte-order tables that mix short positive
keys and high-bit packed keys, keep the table ordering and lookup comparison in
the same unsigned domain. Once an unsigned comparison proves that a key is
inside one packed range, a same-range offset such as `code - range_start` is
still valid when the range endpoints share the same signed representation
region.

```sh
moon run -c 'fn rotr64(value : UInt64, bits : Int) -> UInt64 { (value >> bits) | (value << (64 - bits)) }
fn main { let value : UInt64 = 0x0123456789abcdef; let full : UInt64 = UInt64::lnot(0); println(rotr64(value, 8).to_string()); println((full + (1 : UInt64)).to_string()); println(((0x80 : UInt64) >> 7).to_string()) }'
# 17222085231038278605
# 0
# 1
```

MoonBit `UInt64` arithmetic wraps modulo 2^64, unsigned right shifts are
logical, and bit rotations can be built from paired shifts. This is useful for
porting OCaml code that used fixed-width SHA/AES helper arithmetic. In mixed
literal expressions, provide explicit `UInt64` context when the literal is
outside signed `Int` range.

```sh
moon run -c 'fn main { println(Int::lnot(0)); println((0x0F).lnot()); println((!true).to_string()) }'
# -1
# -16
# false
```

Use `Int::lnot(value)` or `value.lnot()` for OCaml integer `lnot`. Boolean
negation uses `!expr`; avoid porting OCaml `not x` literally.

```sh
moon run -c 'fn main { let tiny : Double = try! @strconv.from_str("1e-10"); println(tiny.to_string()) }'
# 1e-10
```

`Double::to_string()` may emit scientific notation. When porting binary or
document formats whose numeric grammar disallows exponents, add a named
formatter at the serialization boundary instead of writing `Double::to_string()`
directly.

```sh
moon run -c 'fn main { println((1.23456789012345).to_string()); println((999999.9999999).to_string()) }'
# 1.23456789012345
# 999999.9999999
```

OCaml `string_of_float` and MoonBit `Double::to_string()` differ for some
values: OCaml renders whole finite floats with a trailing dot, and infinities
as `inf`/`-inf`, while MoonBit renders `2.0` as `2` and infinities as
`Infinity`/`-Infinity`. Preserve the OCaml spelling explicitly when a digest,
file format, or snapshot depends on the exact string.

```sh
ocaml -noprompt -noinit <<'EOF'
print_endline (string_of_float 2.);;
print_endline (string_of_float infinity);;
EOF
# 2.
# inf
moon run -c 'fn main { println((2.0).to_string()); println((1.0 / 0.0).to_string()) }'
# 2
# Infinity
```

`Double::to_string()` is also not a drop-in replacement for OCaml code that
used an explicit `format_float` precision such as `"%.12g"`. If the OCaml
program used a specific float formatter for serialized data, port that
formatter as an explicit boundary function and test rounding/carry cases.

```sh
moon run -c 'fn main { let a : Double = try! @strconv.from_str("1.23456789012345e20"); let b : Double = try! @strconv.from_str("1.23456789012345e21"); println(a.to_string()); println(b.to_string()) }'
# 123456789012345000000
# 1.23456789012345e+21
```

MoonBit and OCaml choose different thresholds and digit counts for default
float strings. When porting OCaml serialization code that used
`format_float "%.12g"` and then required exponent-free output, do not expand
`Double::to_string()` directly. Round to the OCaml-required significant-digit
policy first, then render into the target grammar.

```sh
moon run -c 'fn main { let divisor = 1.0e308 * 1.0e308; let scale = 1.0 / divisor; println(scale == 0.0); println(divisor); println(scale) }'
# true
# Infinity
# 0
```

MoonBit `Double` overflow can produce `Infinity`, and division by that infinity
can produce zero. When porting OCaml `float` code with defensive branches around
overflow, infinity, or zero scales, validate the exact edge behavior with a
small `moon run -c` probe and cover the branch directly.

```sh
moon run -c 'fn main { println(@math.pow(0.5, 2.0)) }'
# 0.25
```

MoonBit does not use OCaml's floating exponentiation operator `**`. Use
`@math.pow(base, exponent)` when porting OCaml `float ** float` calculations.

```sh
moon run -c 'fn main { println((4.0).sqrt()); println((-3.0).abs()); println((5.5).mod(2.0)) }'
# 2
# 3
# 1.5
```

Many OCaml `float` helpers map to `Double` methods in MoonBit. Verified method
forms include `value.sqrt()`, `value.abs()`, and `value.mod(other)`. The
`@math` package also provides free functions such as `@math.sin`, `@math.cos`,
`@math.atan2`, `@math.ln`, `@math.log10`, `@math.floor`, `@math.ceil`,
`@math.round`, `@math.trunc`, and `@math.exp`. Prefer a `moon ide doc` query
or `moon run -c` probe before assuming an OCaml Stdlib function has the same
MoonBit package path; for example `@math.sqrt` and `@math.abs` are not valid,
but `Double::sqrt`/`value.sqrt()` and `Double::abs`/`value.abs()` are.

```sh
moon run -c 'fn main { let r : Ref[Int] = Ref::{ val: 0 }; r.val += 1; let xs = [1, 2, 3]; xs[0] = 9; println(r.val); println(xs[0]) }'
# 1
# 9
```

Use `Ref[T]` for primitive mutability and mutable fields on structs for larger
state. MoonBit `Array` is a growable mutable vector; `FixedArray` is the closer
match for OCaml `array` because its length is fixed.

```sh
moon run -c $'fn len(xs : ArrayView[Int]) -> Int { xs.length() }\nfn main { let grow = [1, 2, 3]; let fixed : FixedArray[Int] = [1, 2, 3]; println(len(grow)); println(len(fixed)) }'
# 3
# 3
```

Use `ArrayView[T]` for read-only sequence parameters when callers should be
able to pass `Array`, `FixedArray`, or read-only arrays without copying.

```sh
moon run -c 'fn range_lookup(ranges : ArrayView[(Int, Int, Int)], key : Int) -> Int? { for range in ranges { if key >= range.0 && key <= range.1 { return Some(range.2 + key - range.0) } }; None }
fn main { let ranges = [(0x21, 0x23, 100), (0x30, 0x30, 200)]; println(range_lookup(ranges, 0x22).unwrap()); println(range_lookup(ranges, 0x24) is None) }'
# 101
# true
```

When porting large OCaml mapping tables, prefer compact `ArrayView`-accepted
range tables such as `(first, last, base)` plus small exception/sequence tables
over mechanically expanding every entry. This keeps generated MoonBit sources
smaller and makes native-first validation faster while preserving deterministic
lookup behavior.

```sh
moon run -c $'fn table_entry(i : Int) -> Int? { match i { 65 => Some(1); _ => None } }\nlet table : FixedArray[Int?] = FixedArray::makei(128, i => table_entry(i))\nfn lookup(byte : Int) -> Int? { if byte >= 0 && byte < table.length() { table[byte] } else { None } }\nfn main { println(lookup(65).unwrap()); println(lookup(66) == None) }'
# 1
# true
```

For small dense lookup domains that are hit repeatedly, a private top-level
`FixedArray[T?]` built with `FixedArray::makei` is a good replacement for
OCaml code that reconstructs table entries on every lookup. Keep the bounds
check at the lookup boundary, and use optional entries when only some indexes
are valid.

```sh
moon run -c 'fn main { let map : @hashmap.HashMap[Int, Int] = @hashmap.HashMap::new(); map[1] = 10; map[2] = 20; let mut total = 0; for key in map.keys() { total += key }; println(total) }'
# 3
```

MoonBit `for x in xs` works for any iterable value, not only arrays and views.
Do not convert an iterable to an `Array` just to loop over it. Reserve
`.to_array()`/`.to_owned()` for cases that need an owned snapshot, sorting,
indexing, or mutation.

```sh
moon run -c 'fn main { @env.set_env_var("MOONBIT_PORTING_PROBE", "ok"); println(@env.get_env_var("MOONBIT_PORTING_PROBE").unwrap()); @env.unset_env_var("MOONBIT_PORTING_PROBE"); println(@env.get_env_var("MOONBIT_PORTING_PROBE") == None); println(@env.args().length() >= 0); println(@env.current_dir() is Some(_)); println(@env.now().to_string().length() > 0) }'
# ok
# true
# true
# true
# true
```

Use core `@env` for process environment and command-line APIs:
`@env.get_env_var`, `@env.get_env_vars`, `@env.set_env_var`,
`@env.unset_env_var`, `@env.args`, `@env.current_dir`, and `@env.now`.
The current API is:
`args() -> Array[String]`, `current_dir() -> String?`,
`get_env_var(String) -> String?`, `get_env_vars() -> Map[String, String]`,
`now() -> UInt64`, `set_env_var(String, String) -> Unit`, and
`unset_env_var(String) -> Unit`. In particular, `current_dir()` is optional
because some platforms cannot always determine it. Prefer `x is Some(_)` or
pattern matching to test options; `Option::is_some` is deprecated.

```sh
moon run -c 'fn main { let r = @random.Rand::chacha8(seed=@ascii.encode("01234567890123456789012345678901")); println(r.uint(limit=256).to_string()); println(r.uint(limit=256).to_string()) }'
# 21
# 42
```

Use core `@random` for deterministic or ordinary pseudo-random generation.
`Rand::chacha8(seed=...)` requires a 32-byte seed. Do not substitute
`@random` or `@env.now()` for a cryptographic random-byte source when porting
security-sensitive OCaml code such as AES IV, salt, or file-key generation;
keep an explicit random provider or use a target-specific secure API.

```sh
moon run -c 'fn has_ab(view : BytesView) -> Bool { match view { [65, 66, ..] => true; _ => false } }
fn main { let bytes : Bytes = [65, 66, 67]; let view = bytes[:2]; println(view.length()); println(view[0].to_int()); println(has_ab(bytes[:])); println(has_ab(view)) }'
# 2
# 65
# true
# true
```

Use `BytesView` for read-only byte slices. It is the byte-sequence counterpart
to `ArrayView`: views are cheap slices, expose read-only byte operations, and
support pattern matching with rest patterns. Prefer passing or returning
`BytesView` while data can remain borrowed; call `.to_owned()` only when an
owned `Bytes` value is required. `BytesView::to_owned()` returns the original
bytes for a whole view but allocates and copies for a partial slice.

```sh
moon run -c $'fn len_view(view : BytesView) -> Int { view.length() }\nfn main { let bytes = @ascii.encode("ABC"); let view = bytes[:2]; println(len_view(bytes)); println(bytes[:] == bytes); println(view == bytes) }'
# 3
# true
# false
```

`Bytes` can be passed directly to a `BytesView` parameter, which is useful for
APIs that should accept either owned byte payloads or borrowed slices without
forcing callers to allocate. Equality can compare a view against owned bytes
when the view is the left operand; if the owned `Bytes` is on the left, slice it
explicitly first so the comparison is typed as `BytesView` equality.

```sh
moon run -c $'fn accepts_bytes(data : Bytes) -> Int { data.length() }\nfn main { let data = Bytes::from_array([65, 66, 67]); println(accepts_bytes(data[1:]).to_string()) }'
# Error: [4014] BytesView wanted Bytes mismatch

moon run --target native -c $'#borrow(data)\nextern "C" fn accepts_native_bytes_for_probe(data : Bytes) -> Int = "Moonbit_array_length"\nfn main { let data = Bytes::from_array([65, 66, 67]); println(accepts_native_bytes_for_probe(data[1:]).to_string()) }'
# Error: [4014] BytesView wanted Bytes mismatch
```

The reverse direction is not automatic: a `BytesView` does not type-check where
an owned `Bytes` parameter is required, including native extern declarations.
Keep public and internal read-only APIs typed as `BytesView` where possible.
When a callee really requires `Bytes`, such as an ownership boundary, mutation
boundary, long-lived value, or current C FFI byte parameter, call
`.to_owned()` deliberately and account for the possible copy.

For byte-oriented section parsers, prefer one option-returning helper that both
finds a marker and returns the remaining `BytesView`. This avoids the common
porting shape `contains(marker)` followed by a second lookup with an unreachable
`None` branch.

```sh
moon run -c $'fn after_ascii(line : BytesView, word : String) -> BytesView? { let needle = @ascii.encode(word); guard needle.length() > 0 && line.length() >= needle.length() else { return None }; for index in 0..=(line.length() - needle.length()) { let mut matched = true; for offset in 0..<needle.length() { if line[index + offset] != needle[offset] { matched = false } }; if matched { break Some(line[index + needle.length():]) } } nobreak { None } }\nfn main { let bytes = @ascii.encode("1 beginbfchar <41><0041>"); match after_ascii(bytes[:], "beginbfchar") { Some(rest) => println(rest[0].to_int()); None => println(-1) }; println(after_ascii(bytes[:], "missing") == None) }'
# 32
# true
```

```sh
moon run -c 'fn main { let data : Bytes = [60, 65, 62, 0, 12, 60, 66, 62]; println(data.length()); println(data[3].to_int()); println(data[4].to_int()); let view = data[5:]; match view { [60, 66, 62] => println("match"); _ => println("miss") } }'
# 8
# 0
# 12
# match
```

Byte parsers should port the source predicate exactly. NUL (`0`) and form-feed
(`12`) are ordinary bytes in MoonBit `Bytes`/`BytesView`, so a format
whitespace predicate that accepted them in OCaml should not be narrowed to
string-oriented space/tab checks during migration.

```sh
moon run -c 'fn main { let fixed : FixedArray[Int] = [1, 2, 3]; let doubled = [ for x in fixed => x * 2 ]; println(doubled.length()); println(doubled[2]) }'
# 3
# 6
```

MoonBit array/list comprehension syntax is `[ for x in xs => expr ]`.
Use a simple identifier as the comprehension or `for ... in` binder; destructure
tuples inside the loop body or access tuple fields.

```sh
moon run -c 'fn main { let pairs = [(1, 2), (3, 4)]; let firsts = [ for pair in pairs => pair.0 ]; println(firsts[0]); println(firsts[1]) }'
# 1
# 3
```

```sh
moon run -c $'fn parse(s : String) -> Int raise { @strconv.from_str(s) }\nfn main { let _ = [ for s in ["1"] => parse(s) ]; println("done") }'
# Error: calling function with error is not allowed inside list comprehension.
```

Comprehension bodies cannot call error-raising functions. When porting OCaml
`List.map`/`Array.map` code whose mapper may raise, write an explicit loop in a
raising function, push each checked result into an output array, and let the
error propagate from the loop body.

```sh
moon run -c $'fn parse(s : String) -> Int raise { @strconv.from_str(s) }\nfn main {\n  try parse("7") catch {\n    _ => println(0)\n  } noraise {\n    value => println(value)\n  }\n}'
# 7
```

Use `try expr catch { ... } noraise { value => ... }` when porting OCaml
`try ... with ...` code that needs both success and error branches as an
expression. If you deliberately match the result of `try? expr`, wrap the
`try?` expression in parentheses before `match`; `match try? expr { ... }` is
not valid syntax.

```sh
moon run -c 'suberror E { A(Int); B }
fn f(kind : Int) -> Int raise E { if kind == 1 { raise A(5) }; if kind == 2 { raise B }; 1 }
fn g(kind : Int) -> Int raise E { f(kind) catch { A(_) => 2; error => raise error } }
fn main { let r : Result[Int, Error] = try? f(1); match r { Err(E::A(_)) => println("try?"); _ => println("other") }; println(g(0) catch { _ => -1 }); println(g(1) catch { _ => -1 }); println(g(2) catch { B => 3; _ => -1 }) }'
# try?
# 1
# 2
# 3
```

`try? expr` materializes failures as `Result[T, Error]`, which is convenient in
tests and probes but loses the narrower suberror type for re-raising. In
ordinary ported code that catches one constructor and propagates the rest, use
`expr catch { SpecificError => fallback; error => raise error }` so the
function can keep `raise ProjectError`. Payload-carrying suberror constructors
match with ordinary enum payload syntax such as `SpecificError(_)`.
`catch` arms are still checked for exhaustiveness, so do not omit the final
propagating arm when only one error constructor has special handling.

```sh
moon run -c 'enum E { A(Int); B }
fn f(e : E) -> Int { match e { A(n) if n > 0 => n; _ => 0 } }
fn main { println(f(A(3))); println(f(A(-1))); println(f(B)) }'
# 3
# 0
# 0
```

Pattern guards are written on the same match arm as the pattern:
`Pattern if condition => ...`. Do not split the `if` onto the next line as if
it were a standalone expression; the parser expects `=>` after the pattern
line.

```sh
moon run -c 'enum C { Indexed; DeviceGray }
fn f(c : C, bpc : Int) -> String { match (c, bpc) { (_, n) if n == 1 => "generic1"; (Indexed, 1) => "indexed1"; _ => "other" } }
fn g(c : C, bpc : Int) -> String { match (c, bpc) { (Indexed, 1) => "indexed1"; (_, n) if n == 1 => "generic1"; _ => "other" } }
fn main { println(f(Indexed, 1)); println(g(Indexed, 1)); println(g(DeviceGray, 1)) }'
# generic1
# indexed1
# generic1
```

MoonBit match arms are tested top to bottom. Put specific tuple or variant
cases before broad wildcard or guarded catch-all arms; otherwise the broad arm
will shadow the later case. The compiler catches some fully unreachable arms,
but guarded broad arms can still make a later specific case behaviorally dead.

```sh
moon run -c 'fn main { let xs = [(2, "b"), (1, "a")]; xs.sort_by(fn(a, b) { a.0.compare(b.0) }); println(xs[0].0) }'
# 1
```

MoonBit `Array::sort_by` sorts the mutable array in place. Its comparator
returns an `Int`, so OCaml `List.sort compare` or `Array.sort compare` ports can
usually become `xs.sort_by(fn(a, b) { ...compare... })` after copying to an
owned `Array` if the source is an `ArrayView` or other read-only view.

```sh
moon run -c 'fn main { let xs = [1, 2, 3]; let ys = xs.rev(); println(ys[0]); println(xs[0]) }'
# 3
# 1
```

MoonBit `Array::rev()` returns a reversed copy and leaves the original array
unchanged. This is useful when porting OCaml list-building code that conses into
reverse order and then conditionally applies `List.rev`.

```sh
moon run -c 'fn first_sorted(xs : ArrayView[Int]) -> Int { let ys = xs.to_owned(); ys.sort(); ys[0] }
fn main { let fixed : FixedArray[Int] = [3, 1, 2]; let grow = [4, 2, 3]; println(first_sorted(fixed)); println(first_sorted(grow)); println(fixed[0]); println(grow[0]) }'
# 1
# 2
# 3
# 4
```

Use `.to_owned()` when a view must become a mutable owned `Array`, for example
before `sort()`/`sort_by()`. This allocates, so keep it at explicit ownership
boundaries; it also prevents in-place algorithms from mutating an `Array`
caller or requiring a copy from `FixedArray`.

```sh
moon run -c 'fn[A, B, C] apply_pair(f : (A, B) -> C, a : A, b : B) -> (C, A) { (f(a, b), a) }
fn main { let result = apply_pair(fn(x, y) { x + y }, 2, 3); println(result.0); println(result.1) }'
# 5
# 2
```

Polymorphic helper functions use type parameters before the function name:
`fn[A, B, C] name(...) -> ...`. The older `fn name[A, B, C](...)` spelling is
deprecated. This is the direct shape for porting small OCaml helpers with type
variables such as `('a -> 'b -> 'c) -> 'a -> 'b -> ...`.

```sh
moon run -c 'fn apply_twice(f : (Int) -> Int, value : Int) -> Int { f(f(value)) }
fn main { println(apply_twice(fn(x) { x + 1 }, 40)) }'
# 42
```

Callback parameters use function types such as `(Int) -> Int`, while callback
values are usually written as `fn(x) { ... }`. Capturing mutable `Ref` state in a
closure is useful when porting OCaml callbacks that close over refs.

```sh
moon run -c 'fn apply(f : (Int) -> Int raise, x : Int) -> Int raise { f(x) }
fn main { println(try! apply(fn(x) raise { if x == 0 { fail("zero") }; x + 1 }, 1)) }'
# 2
```

Callback parameters that may fail include `raise` in the function type, for
example `(Int) -> Int raise`. Annotate anonymous raising callbacks with
`fn(x) raise { ... }`; relying on inferred raising effects for `fn` literals is
deprecated. If the callback slot expects a narrower project error type, annotate
the literal with that type, for example `fn(x) raise ProjectError { ... }`.

```sh
moon run -c $'fn apply(f : (Int) -> Int raise Error) -> Int raise Error { f(1) }\nfn main { let result : Result[Int, Error] = try? apply(fn(x) raise Error { if x > 0 { fail("x") } else { x } }); println(result is Err(_)) }'
# true
```

Raising and non-raising callback types are not interchangeable. A closure that
may raise must be accepted by a callback slot whose function type includes
`raise`, and a non-raising callback slot cannot call a raising provider.
Intentional probes that pass `(Int) -> Int` where `(Int) -> Int raise Error` is
expected, or pass `(Int) -> Int raise Error` where `(Int) -> Int` is expected,
produce type mismatches in the current toolchain. When porting OCaml callbacks
whose implementation may later use filesystem, async, random, or other fallible
APIs, put the checked error effect in the callback type from the start instead
of trying to hide the failure inside the callback.

```sh
moon run -c 'fn main { let xs = [1, 2, 3]; match xs { [_, .. rest] => { let owned = [ for x in rest => x ]; println(rest.length()); println(owned.length()); println(owned[0]) }; _ => () } }'
# 2
# 2
# 2
```

Array rest patterns bind read-only views. If an enum payload or API requires an
owned `Array[T]`, copy the rest view deliberately with a comprehension such as
`[ for x in rest => x ]`.

```sh
moon run -c 'fn main { let m : @hashmap.HashMap[Int, String] = @hashmap.HashMap::new(); m[7] = "seven"; println(m.get(7).unwrap()) }'
# seven
```

Code files do not contain OCaml-style `open`. Add package imports in
`moon.pkg`, then call imported packages with their `@alias`.

```sh
moon run -c 'fn main { let alias = 1; println(alias) }'
# Warning: [0035] reserved_keyword
# 1
```

`alias` is reserved for possible future use. It still compiles today, but new
MoonBit code should avoid it as a local name, loop binder, or helper name so
warning-enabled checks stay clean.

When validating ports on the JavaScript backend, also avoid JavaScript sentinel
names such as `undefined` for local result variables in tests. Simple command
snippets may compile, but cross-target package tests can still expose generated
JavaScript name interactions. Prefer descriptive domain names such as
`missing_key_result` or `undefined_byte_result`.

```sh
moon run -c 'fn first_positive(xs : Array[Int]) -> Int? { let mut i = 0; while i < xs.length() { if xs[i] > 0 { break Some(xs[i]) }; i += 1 } nobreak { None } }
fn main { println(first_positive([-2, 0, 7]).unwrap()); println(first_positive([-2, 0]) is None) }'
# 7
# true
```

MoonBit `while` loops may produce a value with `break value`; the branch used
when the loop condition becomes false is `nobreak { ... }`. Older examples may
spell this as `else`, but that form is deprecated.

```sh
moon run -c 'fn count_until(limit : Int) -> Int { let mut count = 0; let mut done = false; while !done { count += 1; if count >= limit { done = true } }; count }
fn main { println(count_until(3)) }'
# 3
```

Avoid the older functional `loop ... { ... }` form in new ports; MoonBit now
warns on it. Use an explicit `while` loop with mutable state for simple
translation of recursive OCaml loops, or a modern `for` loop with loop binders
when carrying structured state.

```sh
moon run -c $'fn greet(name? : String = "pdf") -> String { name }\nfn main { println(greet()); println(greet(name="moon")) }'
# pdf
# moon
```

MoonBit default arguments are labelled arguments. Write `name? : T = default`
in the function declaration and call it as `name=value`. Do not write a default
on an unlabelled positional parameter.

```sh
moon run -c 'fn inner(flag? : Bool = false) -> Bool { flag }
fn outer(flag? : Bool = false) -> Bool { inner(flag~) }
fn main { println(outer()); println(outer(flag=true)) }'
# false
# true
```

When wrapping a function with the same labelled optional argument name, forward
the current value with `label~`. This is the MoonBit counterpart to preserving
OCaml optional argument semantics through helper layers.

```sh
moon run -c 'fn takes_view(xs : ArrayView[Int]) -> Int { xs.length() }
fn sample(xs? : ArrayView[Int] = []) -> Int { takes_view(xs) }
fn main { println(sample()); println(sample(xs=[1, 2, 3])) }'
# 0
# 3
```

Optional arguments with view types can use an empty array literal as the
default. If a function parameter is optional-only, pass a non-default value with
its label, for example `sample(xs=[1, 2, 3])`, not `sample([1, 2, 3])`.

```sh
moon run -c 'fn f(a : Int, flag? : Bool = true, xs? : ArrayView[Int] = []) -> Int { if flag { a + xs.length() } else { a - xs.length() } }
fn main { println(f(3, xs=[1, 2], flag=false)); println(f(3, flag=true)); println(f(3)) }'
# 1
# 3
# 3
```

After required positional arguments, multiple optional arguments can be supplied
by label in call-site order independent of their declaration order. This is a
good fit for OCaml APIs that grow additional optional flags beside an existing
optional input slice.

```sh
moon run -c 'fn f(x? : BytesView? = None) -> Int { match x { Some(v) => v.length(); None => -1 } }
fn main { let b = @ascii.encode(""); println(f()); println(f(x=Some(b[:]))) }'
# -1
# 0
```

Optional labelled arguments can themselves have optional view types. Use this
when porting OCaml `string option` parameters where an absent value and an
explicit empty byte string must remain different cases.

```sh
moon run -c 'fn f(x? : Int? = None) -> Int { match x { Some(v) => v; None => -1 } }
fn g(x? : Int? = None) -> Int { f(x~) }
fn main { println(g()); println(g(x=Some(7))) }'
# -1
# 7
```

The `label~` forwarding shorthand also works when the labelled argument's type
is itself optional. Use it in wrappers so `None` remains "argument absent" and
`Some(value)` remains an explicit supplied value.

```sh
moon run -c 'struct R { x : Int } derive(Debug, Eq)
fn default_r() -> R { { x: 1 } }
fn f(r? : R = default_r()) -> Int { r.x }
fn main { println(f()); println(f(r={ x: 2 })) }'
# 1
# 2
```

Default argument values can call local functions. This is useful when porting
OCaml optional arguments whose defaults are structured records or arrays.

```sh
moon run --target native -c 'async fn main { println("async ok") }'
# Error: Cannot use `async fn main`: package moonbitlang/async is not imported.
```

Async entry points and async tests need the `moonbitlang/async` package in the
relevant `moon.pkg` import set; syntax alone is not enough.

```sh
moon run --target native -c $'import {\n  "moonbitlang/async",\n  "moonbitlang/async/fs" @fs,\n}\nasync fn main {\n  let result = try? @fs.read_file("/tmp/definitely-missing-pdflite-fixture.pdf")\n  println(result is Err(_))\n}'
# true
```

MBTX import blocks use comma-separated package entries. This matters when a
quick probe needs both `moonbitlang/async` and a subpackage such as
`moonbitlang/async/fs`.

```sh
moon run -c 'struct Item { value : Int; label : String } derive(Debug)
fn main { let old = Item::{ value: 1, label: "a" }; let next = { ..old, value: 2 }; println(next.value); println(next.label) }'
# 2
# a
```

MoonBit record update uses `{ ..old, field: value }`, which is the direct
replacement for OCaml `{ old with field = value }` when translating immutable
record updates.

```sh
moon run -c 'struct S { values : Int } derive(Debug)
fn make(values : Int) -> S? { Some({ values, }) }
fn main { println(make(7).unwrap().values) }'
# 7
```

For single-field struct literals that use field shorthand inside another
expression, write a trailing comma such as `{ values, }`. This avoids the
ambiguous-block warning and keeps `moon check --warn-list +73` clean.

## Core Porting Rules

### Strings and Bytes

OCaml `string` is a byte sequence. MoonBit `String` is UTF-16 text.
Do not mechanically port OCaml `string` to MoonBit `String`.

Default rule:

- OCaml values used as binary payloads, protocol frames, checksums, encrypted
  data, compressed data, file contents, or parser input become `Bytes`.
- Single byte values become `Byte` where range is known to be 0..255, otherwise
  use `Int` at parser boundaries and convert deliberately.
- Human text, diagnostics, file/source labels, and API names that are truly
  Unicode text use `String`.
- Format-specific identifiers that are byte-oriented should get a small newtype
  or alias instead of being represented as raw `String`.
- Only convert `Bytes` to `String` through named helpers that document the
  encoding assumption: ASCII, UTF-8, UTF-16BE/LE, a domain-specific encoding,
  or unchecked debug output.
- MoonBit `Bytes` is immutable. Build mutable byte data with
  `FixedArray[Byte]`, `Array[Byte]`, or a buffer, then freeze it into `Bytes`.
- Classify OCaml `string` fields independently, even within the same source
  record. It is common for one OCaml record to use `string` for byte-preserving
  payloads, human text, and symbolic format names; port those fields to
  `Bytes`, `String`, or a small name/newtype according to their meaning.

### Numbers

OCaml `int` is used for many distinct concepts. In MoonBit, choose the narrow
meaning:

- Identifiers, array indexes, and counts: usually `Int`.
- File offsets and large serialized positions: `Int64` unless the API is
  explicitly bounded by in-memory `Bytes.length()`.
- Raw bytes: `Byte`.
- Bit-level unsigned arithmetic: `UInt`, `UInt64`, or `Byte` as appropriate.
  For byte-oriented algorithms ported from OCaml `int array`, use `BytesView`
  for immutable key/data inputs, `Array[Int]` for mutable permutation or table
  state, and convert at the byte boundary with `.to_int()` / `.to_byte()`.
- OCaml `float`: usually `Double`.

```sh
moon run -c 'fn main { println(0x50 ^ 0xeb); println((0xbb).to_byte().to_int()) }'
# 187
# 187
```

MoonBit integer bitwise XOR uses `^`. Keep algorithm state in `Int` when the
OCaml code relies on arithmetic modulo 256, then freeze output as `Bytes`.

```sh
moon run -c 'fn main { println(1 << 31); println(((1 << 31) >> 31) & 1); println((-4 >> 2) & 1); println((-4).to_string()) }'
# -2147483648
# 1
# 1
# -4
```

MoonBit `Int` can represent OCaml `Int32`-style signed bitmasks directly when
the format wants 32-bit wrapping behavior. Setting bit 32 produces a negative
signed value, and arithmetic right shift followed by `& 1` still inspects
individual permission bits correctly.

### Data Structures

Map OCaml variants to MoonBit `enum` values. Derive `Debug`, `Eq`, and `ToJson`
for types that will be inspected in tests.

OCaml `array` maps most directly to MoonBit `FixedArray`. MoonBit `Array` is
resizable. Prefer `ArrayView[T]` for read-only function parameters so callers
can pass fixed, growable, or read-only arrays by coercion.

```sh
moon run -c 'fn main { let fixed : FixedArray[Byte] = [1, 2, 3]; let bytes = Bytes::from_array(fixed); println(bytes.length()); println(bytes[2].to_int()) }'
# 3
# 3
```

Use `Bytes::from_array` for byte arrays at ownership boundaries. It accepts a
`FixedArray[Byte]` through the same read-only array coercion; avoid the
deprecated fixed-array-specific constructor.

For an OCaml variant, start with a direct MoonBit `enum` shape and make payload
types explicit:

```mbt
enum Value {
  Null
  Boolean(Bool)
  Integer(Int)
  Real(Double)
  String(Bytes)
  Array(Array[Value])
} derive(Debug, Eq, ToJson)
```

Use ordered arrays of pairs when the OCaml code relies on insertion order or
stable rendering. Use `@hashmap.HashMap` for non-ordered lookup tables once the
package imports are added to `moon.pkg`.

### Mutation and Laziness

OCaml uses `ref`, mutable records, and lazy object parsing. MoonBit equivalents:

- `Ref[T]` for a direct `ref`.
- `mut` fields inside structs for document/object-map state.
- Structs with mutable fields for larger state, instead of hiding mutation
  inside tuple refs or nested mutable containers.
- Immutable record updates use `{ ..old, field: value }`.
- Represent lazy/deferred states as explicit `enum` variants.

### Errors

OCaml exceptions such as domain-specific errors, `Not_found`, `End_of_file`, and
`Invalid_argument` should not be copied as unchecked control flow.

MoonBit uses checked errors:

- Define a project-level `suberror` once the first fallible functions are
  ported.
- Functions that can fail should declare `raise ProjectError` or plain `raise`
  if the error set is intentionally broad.
- In tests and raising helpers, call fallible success-path APIs directly and let
  errors propagate. Use `try? f()` only when the test is asserting or inspecting
  the failure result. Prefer `Err(_)` unless the exact error type is the
  behavior under test.
- In ordinary code, avoid `match (try? f()) { Ok(...) => ...; Err(...) => ... }`.
  MoonBit warns on that anti-pattern; prefer `try f() catch { ... } noraise
  { ... }`.

### Async I/O

MoonBit supports async I/O and encourages explicit async boundaries. There is
no `await` keyword; async functions call other async functions directly.

Migration rule:

- Keep CPU-bound pure transformations over `Bytes` synchronous.
- Make filesystem/network-facing entry points async when they touch
  `moonbitlang/async` APIs.
- Prefer async wrappers that load file contents into `Bytes`, then call the
  synchronous core. This avoids making every recursive helper async.
- Async tests require native target support and package imports for
  `moonbitlang/async` in the test section of `moon.pkg`.
- If a port should keep its pure core available on non-native targets, put
  async/file-system wrappers in a native-only package or native-targeted files
  rather than importing async packages into the core package. MoonBit imports
  are package-level, while `targets`/`supported_targets` control backend
  inclusion.
- If native-only tests need extra imports such as `moonbitlang/async/fs`,
  prefer placing them in a small native-only package with
  `supported_targets = "+native"`. A single test file can be target-gated with
  `options(targets: { "file_test.mbt": [ "native" ] })`, but imports are still
  package-level and may become unused on non-native checks. Verified in this
  project with `moon check --target all --warn-list +73` plus
  `moon check markdown/fixture_acceptance --target native --warn-list +73`.

## MoonBit Testing Model

Testing should be part of each migration slice.

Test file roles:

- `*_test.mbt`: black-box tests. They call only public APIs through the package
  alias.
- `*_wbtest.mbt`: white-box tests. They run inside the package and may test
  private helpers.
- `*.mbt.md`: documentation with checked code blocks. Use `mbt check` for code
  that should compile and test, and `mbt nocheck` for illustrative code.

Assertion style:

- Use `@test.assert_eq` for stable scalar or structural results; it has a
  `Debug` bound and avoids the deprecated `Show`-based assertion path.
- Boolean assertions can use the same API, for example
  `@test.assert_eq(condition, true)`, instead of assuming an
  `@test.assert_true` helper exists. Verified with
  `moon run -c 'fn helper() -> Unit raise Error { @test.assert_eq(true, true); @test.assert_eq([1, 2], [1, 2]) }\nfn main { try! helper() }'`.
- `@test.assert_eq` accepts a named `msg` argument for stable diagnostic
  context in shared assertion helpers. Verified with
  `moon run -c 'import { "moonbitlang/core/test" }\nfn helper() -> Unit raise { @test.assert_eq(true, true, msg="named assertion") }\nfn main { try! helper(); println("ok") }'`.
- Shared test helpers that call `@test.assert_eq`, `fail`, or fallible APIs
  should declare `-> Unit raise Error` unless they use a narrower project error
  type. Verified with
  `moon run -c 'fn helper() -> Unit raise Error { @test.assert_eq(1, 1) }\nfn main { try! helper() }'`.
- Use `assert_true(value is Pattern(...))` or `guard ... else { fail(...) }`
  for pattern checks.
- Use `inspect(value, content="...")` for snapshot tests of small values.
- Use `json_inspect(value, content=...)` for complex values with `ToJson`.
- If the expected snapshot is unknown, write `inspect(value)`, run
  `moon test --update`, then review the updated `content=...` diff.
- For success paths through raising functions, call the fallible API directly
  and let the test fail on any error. For expected failures, use
  `let result : Result[T, Error] = try? f()` and assert or inspect the result.
- For parser/serializer, encoder/decoder, or loader/writer pairs, pair focused
  edge-case tests with at least one public API round-trip test when practical.
  This catches ownership, byte/text, and object-boundary mistakes that isolated
  unit tests can miss.

Useful commands:

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
2. Add any `moon run -c` probe to this document if it taught us a reusable rule.
3. Run targeted `moon test` while developing.
4. Finish with `moon check --warn-list +73`, `moon test`, and
   `moon info && moon fmt`.

## Update Discipline

When a reusable OCaml pattern is encountered, record the MoonBit decision here
before or alongside the code change. Each rule should leave behind:

- The OCaml behavior being replaced.
- The MoonBit API or idiom chosen.
- The verification command(s), especially `moon run -c` probes.
- Any known incompatibility or deferred behavior.
