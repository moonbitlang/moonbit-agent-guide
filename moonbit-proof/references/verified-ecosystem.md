# Verified Ecosystem: moonbit-community/verified (Empirically Verified)

Load this reference when you need Fset/Fmap proof shims, worked examples, or want to publish/reuse verified packages. Verified against mooncakes `moonbit-community/verified@0.0.2` source (local registry cache) + `@0.0.3` registry metadata (published 2026-10-13). Repo: https://github.com/moonbit-community/verified

## Setup

```bash
moon add moonbit-community/verified
```

then in the client package's `moon.pkg`:

```moonbit
proof_enabled = true
import {
  "moonbit-community/verified/libs/fset" @fset,
  "moonbit-community/verified/libs/fmap" @fmap,
}
```

## @fset API (libs/fset)

Carrier: `Fset[T]` — opaque via `#proof_external("set.Fset", "fset")`; the abstraction focuses on membership laws, not representation.

Values/functions: `empty`, `add`, `remove`, `mem`, `union`, `subset`, `filter`, `cardinal`, `eq`, plus `fset_empty` and `fset_interval` (integer intervals, from `set.FsetInt`).

Under it, raw stubs (in fset.mbtp) map to Why3 `set.Fset` symbols: `mem`, `empty`, `add`, `union`, `(==)`, `remove`, `filter`, `subset`, `cardinal`, `interval`. Public wrappers (e.g. `pub fn Fset::mem(self : Fset[Int], x : Int) -> Bool`) re-expose them in MoonBit shape.

## @fmap API (libs/fmap)

Carrier: `Fmap[K, V]` — opaque via `#proof_external("fmap.Fmap", "fmap")`, backed by the toolchain-bundled custom `fmap.mlw` theory (built on `set.Fset` + `map.Map`).

Functions: `mk` (build from key set + function), `add`, `remove`, `find`, `mem`, `eq`, `size`.

Why3-side symbols include: `(==)`, `mem`, `mapsto`, `is_empty`, `mk`, `empty`, `add`, `([<-])`, `find`, `remove`, `size`, plus lemma `mem_mapsto`.

## Worked Example Packages (36 in @0.0.2)

Algorithms & proofs: `abs`, `arreq`, `binary_search`, `bubble_sort`, `selection_sort`, `checksorted`, `clamp`, `count`, `div`, `find`, `gauss`, `isqrt`, `lowerbound`, `maxarr`, `maxfn`, `sumbounds`, `two_sum`, `invpred`.

Verified data structures: `avl`, `leftist_heap`, `skew_heap`, `pairing_heap`, `sparse_array`, `stack_min`, `bitmap32`, `bv32`.

Domain applications: `batch_auction`, `cpmm_swap`, `ltv_lending`, `risk_limits`, `stablecoin_engine`, `threshold_multisig`, `vesting_stream`.

Also on mooncakes: `bobzhang/loop_invariants_verified` (verified binary search, count_nonneg with loop-invariant patterns).

Use these as calibration references before inventing a new proof shape — especially `avl` (model + invariant style) and `sparse_array` (bitmap/rank/packed-layout ladder style).

## Shim Pattern (Copy This Structure)

```moonbit
// shim.mbt — the abstract carrier lives on the program side
///|
#proof_external("set.Fset", "fset")
pub type Fset[T]
```

```moonbit
// shim.mbtp — raw stubs (param order matches the Why3 symbol!), then wrappers
#proof_import("set.Fset")
fn raw_fset_mem(x : Int, s : Fset[Int]) -> Bool = "mem"

#proof_import("set.Fset")
fn raw_fset_empty() -> Fset[Int] = "empty"

///|
pub fn Fset::empty() -> Fset[Int] {
  raw_fset_empty()
}
```

Rules:
- one shim package per theory family; keep it minimal
- stub parameter order must match the Why3 symbol's argument order (Why3 `mem : 'a -> fset 'a -> bool` means element first — note `Fset::mem(self, x)` wrapper flips it for MoonBit ergonomics)
- operator symbols are imported with parenthesized names: `= "(==)"`
- client packages import the shim; do NOT repeat raw `#proof_import` stubs in every client
- monomorphic Int-only shims prove faster than generic ones; generalize only after the monomorphic slice verifies

## Cross-Package Type References

fmap.mbt references the fset carrier with:

```moonbit
using @fset {type Fset}
```

with `@fset` aliased in the fmap package's moon.pkg import block.
