---
name: ocaml2moonbit-migration
description: Port OCaml codebases, modules, libraries, tests, or algorithms to idiomatic MoonBit. Use when translating OCaml syntax, module structure, exceptions, variants, records, parsers, byte/string logic, tests, or public APIs into MoonBit while preserving behavior and improving modularity.
---

# OCaml to MoonBit Migration

## Goal

Port behavior first, then make it idiomatic MoonBit. Avoid line-by-line transliteration when it preserves OCaml incidental structure instead of the program's invariants.

## Workflow

1. **Inventory behavior before editing**
   - Identify public OCaml APIs, core data types, exceptions, parser/IO boundaries, and known edge cases.
   - Keep reference outputs from the OCaml implementation when possible: golden strings, serialized bytes, parse trees, error cases, and round trips.
   - Decide what does not need compatibility. If there are no external users, prefer simpler MoonBit APIs over compatibility shims.

2. **Port stable slices**
   - Start with pure data models and small pure functions.
   - Add focused tests immediately for each slice.
   - Move parsers, serializers, IO, native stubs, and large orchestration after the core model is checked.

3. **Use MoonBit package boundaries deliberately**
   - A MoonBit package is a directory with `moon.pkg`; files are only organizational, not modules.
   - Keep MoonBit source in `///|` blocks and move blocks freely when reorganizing files.
   - Split files freely inside a package. Split packages only when dependencies and ownership are clear.
   - A package cannot add methods to a type owned by another package. Move the owning type and its main methods together, or use free functions/wrapper types as the extension point.
   - Prefer explicit `@pkg.name` during broad migrations. Remove temporary `using` imports once call sites are updated.

4. **Validate after each migration slice**
   - Run `moon check` frequently.
   - Run targeted tests for the migrated area, then `moon test`.
   - Run `moon info && moon fmt` before review. Inspect generated `.mbti` diffs to confirm public API changes are intentional.
   - For native or FFI-sensitive code, also run `moon test --target native` and native warning checks.

## OCaml to MoonBit Mapping

### Modules and Packages

- OCaml modules map either to MoonBit packages or cohesive files inside one package. Do not create one package per OCaml file by default.
- OCaml functors usually become explicit functions, records of operations, traits, or package-level specialization. Prefer the simplest shape needed by the call sites.
- If an OCaml module mainly namespaces helpers, keep those helpers package-private unless downstream packages need them.

### Types

- OCaml variants map naturally to MoonBit enums. Prefer regular constructors with typed pattern matching.
- OCaml records map to MoonBit structs. Mark fields mutable only when mutation is part of the invariant.
- Replace polymorphic variants with explicit enums unless callers truly need open extension.
- Use `derive(Debug, Eq, ToJson)` only when tests, diagnostics, or serialization actually need it.

### Errors

- Map OCaml exceptions to a domain error enum plus `raise` on functions that can fail.
- Let MoonBit error propagation work through raising calls. Do not add explicit `try` just to propagate.
- Use `try?` only when converting a raising call into `Result`.
- Keep error constructors precise enough for tests to assert specific failures.
- When the expected error type is known, omit redundant enum prefixes:

```mbt
guard result is Err(ParseExpected) else { fail("expected parse error") }
```

### Labels and Optional Arguments

- OCaml labelled arguments map to MoonBit labelled arguments such as `name~ : T`.
- OCaml optional arguments map to `name? : T = default` only when callers may omit them.
- If every call site supplies the value, use a required labelled argument instead. Warning `0032` points this out.
- Avoid carrying compatibility-only optional defaults when the migrated API has no external users.

### Collections and Bytes

- OCaml lists are often better as `Array[T]` or `ArrayView[T]` in MoonBit, especially for parser output, bytes, and repeated PDF-like structures.
- Use `BytesView`, `StringView`, and `ArrayView` for borrowed inputs.
- Use owned `Bytes` or `Array[T]` only when the result must outlive the input or be mutated independently.
- Prefer structured parsers and byte cursors over ad hoc string slicing for binary formats.

### Control Flow

- Convert recursive OCaml loops to MoonBit `for` loops with explicit state when that is clearer or avoids stack concerns.
- Use pattern matching and `is` guards for concise validation paths.
- Prefer direct array/string/view patterns where MoonBit supports them.

## Testing Strategy

- Preserve behavior with golden tests before changing API shape.
- Cover both success and failure paths: empty input, truncated input, malformed syntax, unsupported features, boundary integers, nested structures, duplicate references, and round trips.
- Use assertion tests for stable behavior. Use snapshots for large serialized outputs when exact bytes/text matter.
- Keep blackbox tests qualified with `@package` when testing exported APIs.
- Keep whitebox tests for parser internals, reconstruction logic, and migration-only invariants that are not public API.

## Refactoring After the Port

- Once behavior is covered, make APIs idiomatic:
  - remove compatibility wrappers that no caller needs,
  - shrink public helper APIs,
  - move helpers to lower-level or internal packages,
  - replace temporary package re-exports with explicit call sites,
  - simplify redundant enum annotations and imports using compiler warnings.
- Treat warnings as migration work items. Useful buckets include:
  - `0025`: blackbox tests should qualify package APIs,
  - `0032`: optional defaults are unused and can become required labels,
  - `0065`: read-only arrays can be marked as such,
  - `0072`: local `using` imports should become explicit qualification,
  - `0073`: redundant enum/type annotations can be removed,
  - `0074`: public APIs need docs.

## Review Checklist

- The MoonBit API expresses the migrated behavior, not just the OCaml module layout.
- Package dependencies are acyclic and type ownership matches method placement.
- Public helpers are documented or made private.
- Tests cover the OCaml reference behavior and important corner cases.
- `moon info`, `moon fmt`, `moon check`, and `moon test` pass.
- Native-specific behavior is checked with `--target native` when relevant.
