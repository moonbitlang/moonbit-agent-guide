---
name: ocaml2moonbit-migration
description: Guide for migrating OCaml projects, libraries, modules, and test suites to idiomatic MoonBit. Use when translating OCaml code to MoonBit, planning a large OCaml-to-MoonBit port, preserving byte/string-heavy behavior, replacing OCaml variants/records/exceptions/refs/arrays, mapping OCaml APIs to MoonBit packages, or building verification and test strategy for a migration.
---

# OCaml to MoonBit Migration

Port behavior, data invariants, and public contracts first. Translate syntax only after the source semantics are classified.

## Reference

Use [references/OCaml2MoonBit.md](references/OCaml2MoonBit.md) as the detailed fact bank. It contains verified probes and migration notes from a large real-world port.

Load or search that reference when the task touches:

- byte/text boundaries, encodings, `Bytes`, `BytesView`, or `String`
- integer width, wrapping, shifts, unsigned ordering, byte codecs, or float rendering
- variants, records, array/view ownership, tables, callbacks, labelled/default arguments, or pattern matching
- checked errors, raising callbacks, exception tests, async I/O, native-only code, or MoonBit test layout

When using a rule from the reference, prefer the documented probe if it directly fits. If the local MoonBit toolchain is newer, rerun a small `moon run -c` probe before relying on behavior that is likely to change.

## Migration Workflow

1. Inventory the OCaml module boundary: public types, functions, exceptions, optional arguments, mutable state, lazy/deferred state, C/Unix/filesystem dependencies, and existing tests or golden fixtures.
2. Classify every OCaml `string` by meaning before choosing a MoonBit type. Do this field by field, even inside the same OCaml record.
3. Choose MoonBit package boundaries and imports before coding. Add imports to `moon.pkg`; MoonBit source files do not use OCaml-style `open`.
4. Port one behavioral slice at a time with tests. Prefer a thin public API skeleton, then fill parser/serializer/algorithm internals behind it.
5. Probe uncertain language or library behavior with `moon run -c` and, when needed, a small OCaml toplevel probe. Keep probes minimal and copy reusable discoveries back into the reference.
6. Finish each slice with targeted tests, then `moon check --warn-list +73`, `moon test`, `moon info`, and `moon fmt` when the repository is a MoonBit project.

## Type Choices

Default mappings are semantic, not mechanical:

| OCaml use | MoonBit default |
|---|---|
| binary payload, file contents, compressed/encrypted/checksummed data, parser input | `Bytes` or `BytesView` |
| human-readable text, diagnostics, labels that are truly Unicode | `String` |
| single byte with known range | `Byte` |
| indexes, counts, small identifiers, deliberate signed 32-bit wrapping | `Int` |
| file offsets, serialized positions, large object numbers | `Int64` or `UInt64` |
| OCaml `array` with fixed length | `FixedArray[T]` |
| mutable growable builder | `Array[T]` |
| read-only sequence parameter | `ArrayView[T]` or `BytesView` |
| OCaml `ref` | `Ref[T]` |
| OCaml variant | `enum`, often `priv enum` for internal states |
| OCaml record | `struct`, with `{ ..old, field: value }` for immutable update |
| OCaml exception flow | `suberror` plus checked `raise` |
| lazy/deferred state | explicit `enum` state machine |

Do not convert OCaml `string` to MoonBit `String` by default. OCaml `String.length` counts bytes; MoonBit `String::length()` counts UTF-16 code units. Use `Bytes`/`BytesView` for byte-addressed formats and named encoding helpers for the few places that legitimately cross into text.

## Porting Rules

- Build binary output with `Array[Byte]`, append ASCII syntax through `@ascii.encode(text)[:]`, append payloads from `BytesView`, and freeze once with `Bytes::from_array`.
- Prefer `BytesView` for read-only byte APIs. Convert to owned `Bytes` only at explicit ownership, storage, mutation, or FFI boundaries.
- Keep 32-bit wrapping behavior explicit. MoonBit `Int` wraps at signed 32-bit boundaries; promote to `Int64`/`UInt64` for serialized lengths, offsets, and wide accumulators.
- Treat unsigned serialized keys as unsigned. If high-bit packed values are sorted or binary-searched, compare through `UInt`, `UInt64`, or a deliberately matching ordering.
- Do not use `Double::to_string()` as a drop-in replacement for OCaml float serialization. Add an explicit formatter when snapshots, digests, or file grammars require OCaml spelling.
- Port OCaml `List.map`/`Array.map` with raising mappers as explicit loops in a raising function; comprehension bodies cannot call error-raising functions.
- Put specific match arms before guarded wildcard or broad tuple cases. MoonBit match arms run top to bottom.
- Include `raise` in callback parameter types and anonymous callback literals when the OCaml callback may throw or call fallible APIs.
- Use labelled defaults for OCaml optional arguments, and forward same-named optional labels with `label~`.
- Keep pure core transforms synchronous. Put filesystem/network wrappers at async or native-only package boundaries when using `moonbitlang/async`.

## Testing Strategy

Create migration tests with the same ownership boundary as the bug risk:

- Use `*_test.mbt` for black-box public API behavior and `*_wbtest.mbt` for private parser or table helpers.
- Pair focused edge tests with at least one round-trip test for parser/serializer, encoder/decoder, loader/writer, or cryptographic/hash-like code.
- Use `@test.assert_eq` for stable values. Use `try?` only when an expected failure is the assertion; otherwise let checked errors propagate from tests and helpers.
- Assert byte/text behavior with non-ASCII, NUL, form-feed, high-bit bytes, overflow, empty input, and boundary offsets when the OCaml source depended on those cases.
- Preserve golden fixtures where possible. If output spelling changes intentionally, document the compatibility decision.

## Update Discipline

When a migration teaches a reusable rule, update `references/OCaml2MoonBit.md` with:

- the OCaml behavior being replaced
- the MoonBit API or idiom chosen
- the verification command and observed output
- any known incompatibility, target limitation, or deferred behavior
