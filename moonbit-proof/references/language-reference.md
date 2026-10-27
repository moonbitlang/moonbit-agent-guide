# Proof Language Reference (Empirically Verified)

Load this reference when writing `.mbtp` files, contracts, or proof attributes. Verified against moonc v0.10.17+d971670cd (2026-10-23) via live compilation, `moon explain --attribute`, and official docs (https://docs.moonbitlang.com/en/latest/language/verification.html).

## Program Side vs Logic Side (Core Mental Model)

- `.mbt` files = **program side**: executable code. Contracts, `proof_assert`, and loop annotations are the only logic-side reasoning inside.
- `.mbtp` files = **logic side**: predicates, logic functions, lemmas, `#proof_import` stubs. Not runtime code; program code never calls them during execution.
- Logic-side specs cannot call arbitrary program-side definitions. Bridge with `#proof_pure`.

## What Goes Where (Verified Rules)

| Item | `.mbt` | `.mbtp` |
|---|---|---|
| executable functions | ✅ | ❌ |
| contracts (`where { proof_require/proof_ensure/... }`) | ✅ | (lemmas use `where` too) |
| `proof_assert`, loop `where` annotations | ✅ | — |
| `predicate` definitions | ❌ | ✅ |
| `lemma` definitions | ❌ | ✅ |
| `fn` logic functions (usable in specs) | ❌ | ✅ |
| type declarations (`pub type`/`struct`/`enum`) | ✅ | ❌ parse error [3002] |
| `#proof_pure` functions | ✅ | ❌ breaks subsequent parsing |
| `#proof_external(module, symbol)` on a type | ✅ | ❌ (type can't exist there) |
| `#proof_import(module)` stubs (string body) | ❌ treated as invalid inline wasm | ✅ |

## Contract Fields

```moonbit
pub fn f(xs : FixedArray[Int], key : Int) -> Option[Int] where {
  proof_require: sorted(xs),                      // precondition
  proof_ensure: result => binary_search_ok(xs, key, result),  // postcondition (non-Unit result)
  proof_decrease: n,                              // termination measure
  proof_axiomatized: true,                        // assume this contract (trusted)
} { ... }
```

- `result => ...` lambda form only for non-Unit return types. Unit-returning functions use a plain Bool: `proof_ensure: true` — the lambda form is a type error there (verified [4014]).
- Multiple `proof_require` / `proof_ensure` lines allowed.
- `proof_axiomatized: true` on a function = verifier assumes the contract; on a lemma = assumes the conclusion. Becomes trusted surface.

## proof_assert

```moonbit
proof_assert xs[h] == key
proof_assert ∀ idx : Int, (0 <= idx) && (idx < i) → xs[idx] < key
```

Unicode quantifiers (`∀`/`∃`) and arrows (`→`) work in predicates and asserts.

## Loop Annotations

```moonbit
for i = 0, j = xs.length(); i < j; {
  ...
  continue h + 1, j     // rebind loop vars
  break Some(h)         // yield a value
} nobreak {
  None                  // value when loop exits without break
} where {
  proof_invariant: 0 <= i,
  proof_invariant: i <= j,              // multiple invariants allowed
  proof_yield: res => binary_search_ok(xs, key, res),
  proof_reasoning: (
    #| The loop maintains a candidate window `[i, j)`.
    #| Every index before `i` holds a value `< key`.
  ),
}
```

- `proof_invariant`: must hold at every iteration boundary (initial + preserved).
- `proof_yield`: facts about values yielded by the whole loop expression (from `break` and the `nobreak` default). **Only allowed on loops yielding non-Unit** (compiler: `where_clause_proof_yield_on_unit_loop`).
- `proof_reasoning`: three accepted forms (verified in moonc strings + docs):
  1. block: `proof_reasoning: ( #| line1  #| line2, )`
  2. plain string: `proof_reasoning: "The function returns its argument unchanged."`
  3. single line: `proof_reasoning: #| loop terminates because ...`
- Optional lint warnings (default off): `missing_invariant` (#38) and `missing_reasoning` (#39) — enable with `warnings = "+missing_invariant+missing_reasoning"` in moon.pkg.
- A bare loop with array reads and NO invariant produces an unprovable "index in array bounds" VC (`0 <= model i` unverifiable) → Timeout. Always write at least index-bounds invariants.

## #proof_pure (Share One Definition Across Both Sides)

```moonbit
// in .mbt
#proof_pure
fn height(t : Tree) -> Int {
  match t { Empty => 0, Node(_, _, _, h) => h }
}
```

```moonbit
// in .mbtp — now usable in specs
predicate cached_height_ok(t : Tree, result : Int) {
  result == height(t)
}
```

Verified limitations (official docs + `moon explain --attribute proof_pure`):
- only regular top-level functions and ordinary methods
- NO verification contracts, direct recursion, or mutual recursion on `#proof_pure` definitions
- best treated as pure specification helpers (height, rank, summary measures)
- putting `#proof_pure` in `.mbtp` breaks that file's parsing — keep it in `.mbt`

## #proof_external / #proof_import (Why3 Theory Bridges)

`#proof_external(module, symbol)` in `.mbt` maps an abstract MoonBit type to a Why3 type:

```moonbit
///|
#proof_external("set.Fset", "fset")
pub type Fset[T]      // lowered to set.Fset.fset 'T in proof logic
```

`#proof_import(module)` in `.mbtp` declares a proof-only stub whose **string body** names the Why3 symbol; parameter order must match the Why3 symbol:

```moonbit
#proof_import("set.Fset")
fn raw_fset_mem(x : Int, s : Fset[Int]) -> Bool = "mem"

#proof_import("set.Fset")
fn raw_fset_eq(s1 : Fset[Int], s2 : Fset[Int]) -> Bool = "(==)"   // operators use "(==)" form
```

- The string body is proof-stub syntax, NOT an executable expression and unrelated to FFI.
- These declarations lower only; they provide no runtime implementation.
- Signatures + symbol mappings are a **trusted integration boundary**.
- Public wrappers over raw stubs may live in `.mbtp` (logic-side `pub fn Fset::mem(...) { raw_fset_mem(x, self) }` pattern from moonbit-community/verified).

## Why3 Theories Available (Bundled Stdlib)

`~/.moon/share/why3/stdlib/` ships 44 theory files: `set` (Fset with mem/empty/add/union/remove/filter/subset/cardinal/(==)), custom `fmap` (Fmap with mk/empty/add/find/remove/mem/(==)/size/mapsto, built on set.Fset + map.Map), `seq`, `map`, `array`, `int`, `option`, `ref`, `list`, `bintree`, `bv`, `bag`, `queue`, `stack`, `pqueue`, `graph`, `real`, `relations`, `number`, etc. Reference them as `#proof_external("set.Fset", "fset")`, `("fmap.Fmap", "fmap")`, `("seq.Seq", ...)` etc.

## Built-in Type Modeling (Prelude)

`~/.moon/lib/prelude_proof/moonbit_builtin_prelude.mlw` provides modules: `Int`, `UInt`, `Int64`, `UInt64`, `Byte`, `Bytes`, `FixedArray`, `BytesView`, `ReadOnlyArray`.

- All machine-integer families lower to **unbounded mathematical integers** (`type t = int`); `in_bounds` is trivially `true`.
- Arithmetic proofs do NOT model runtime overflow — a program can verify yet still need explicit range discipline at runtime (machine-integer verification is planned, not default).
- `Bytes`/`BytesView`/`ReadOnlyArray` model via Why3 `seq.Seq`; `FixedArray` lowers to Why3 `array` with `.FixedArray.length`/`get`.
- Generated WhyML aliases prelude modules, e.g. `use moonbit_builtin_prelude.Int as Moonbit_builtin_prelude_Int`; literals become `of_int 0`, comparisons `le`/`lt` functions.
- `#proof_pure` helpers lower to `let function` (logic functions) in the generated WhyML.

## Mutation Constraints (Officially Documented)

- Mutable state is expected to remain **local**.
- **Escaping uses of `FixedArray` are currently not supported** in proofs.
- Functional-style formulations are preferred when both exist.

## What moon prove Checks

- preconditions imply the body is safe to execute
- postconditions hold on every return path
- each `proof_assert` is valid
- loop invariants hold initially and are preserved
- termination measures decrease (when the loop form supports them)
- bounds checks and similar safety properties

## Trust Surface Summary

1. Mathematical integers replace machine integers (no overflow modeling).
2. `proof_axiomatized` items are assumed, not proved.
3. `#proof_external`/`#proof_import` declarations correctly describe the intended Why3 types/operations.
