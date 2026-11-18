---
name: moonbit-proof
description: "Use when writing, refactoring, or debugging proof-carrying MoonBit code: proof_enabled packages with .mbtp logic-side predicates/lemmas, Why3+SMT-backed moon prove workflows, contracts (proof_require/proof_ensure/proof_decrease/proof_axiomatized), loop annotations (proof_invariant/proof_yield/proof_reasoning), #proof_pure shared helpers, #proof_external/#proof_import theory bridges (set.Fset, fmap.Fmap), abstraction models (model/elements), representation invariants, recursive verified data structures, proof failure triage via _build/verif reports, and shrinking trusted proof bridges."
---

# MoonBit Proof-Carrying Code

Use this skill when the task is to write, extend, or debug verified MoonBit code.

Typical triggers: adding contracts, defining abstract models or representation
invariants, verifying recursive data structures, connecting representations to
set/map models, replacing trusted bridges with lemmas, debugging proof failures
or lowering limits, and setting up the `moon prove` toolchain (Why3/solvers/`proof_enabled`).

Validated against moon 0.1.20261023 / moonc v0.10.17+d971670cd (2026-10-23) by live
compilation and proving; aligned with the official Formal Verification chapter
(docs.moonbitlang.com/en/latest/language/verification.html). Deeper material:
[toolchain-setup.md](references/toolchain-setup.md) (environment, CLI, artifacts),
[language-reference.md](references/language-reference.md) (syntax rules, placement table),
[debugging-cookbook.md](references/debugging-cookbook.md) (report schema, failure taxonomy),
[verified-ecosystem.md](references/verified-ecosystem.md) (fset/fmap shims, worked examples).

## Goal

Write proof-carrying code, not proof-shaped comments: the runtime code remains
executable and readable; the proof model is explicit; contracts talk about named
predicates/functions with consistent roles; local `proof_assert` steps explain why
the implementation satisfies the model.

## Step 0: Gate the Environment First

Before writing any proof code, confirm:

1. The package is proof-enabled — `proof_enabled = true` in its `moon.pkg`
   (new DSL format; `moon.pkg.json`/`moon.mod.json` are rejected — migrate with `moon fmt`).
2. Why3 is installed (opam, pinned `1.7.2` recommended) and at least one solver
   (`z3`, `cvc5`, or `alt-ergo`) is on PATH. Provers are auto-detected into
   `_build/verif/why3.conf`.
3. A 30-second self-test: make a tiny package with one trivially proved contract
   and run `moon prove <pkg>` — expect `Succeeded: N goals proved`, exit code 0.

If any of these fail, fix the environment before touching proof code (details: toolchain-setup.md).

## The Two-Side Model (Program vs Logic)

- `.mbt` = program side: executable code, plus contracts and proof annotations.
- `.mbtp` = logic side: `predicate`, `lemma`, logic functions, `#proof_import` stubs.
- Program-side execution never calls logic-side definitions; specs cannot call arbitrary
  program-side code. `#proof_pure` on a `.mbt` function shares one side-effect-free
  definition across both sides (no contracts/recursion on it yet; never place it in `.mbtp`).
- Type declarations (hence `#proof_external` carriers) live only in `.mbt`;
  `#proof_import` string-body stubs live only in `.mbtp`.

Full placement table: language-reference.md.

## Naming

Prefer `model(...)` as the default name for the proof-side semantic view of a value
(e.g. `model(set) : Fset[Int]`, `model(map) : Fmap[Int, Int]`, `model(tree) : Seq[Int]`).

Use a more specific name only when it is materially clearer:
- `elements(node)` when the model is literally the set of elements stored in a recursive subtree
- `domain(bitmap)` when the model is specifically the occupied index set
- `height(tree)` when it is a structural measure, not the main semantic model

Default rule: use `model` for the main semantic abstraction; use specialized names
like `elements`, `domain`, `height`, or `rank` for auxiliary views.

## Default Structure

Split the package into two layers.

- `.mbtp`: model functions (`model(x)`), representation invariants (`tree_inv(x)`),
  named proof predicates (`insert_pre(...)`, `insert_post(...)`), reusable lemmas.
- `.mbt`: executable code, contracts over the proof-side predicates, and local
  `proof_assert` steps after construction, branching, and loops.

Use this default unless there is a strong reason not to.

## Recommended Workflow

1. Gate the environment (Step 0).
2. Choose the abstract model first.
3. Define the smallest useful invariant.
4. State contracts with named `*_pre` / `*_post` predicates.
5. Implement the runtime code.
6. Add loop invariants for every proof-relevant loop.
7. Add local `proof_assert` steps where the solver needs help.
8. Introduce helper lemmas only after seeing actual failing VCs.
9. Shrink trusted bridges from constructors outward.

Do not start by writing a large pile of lemmas.

## Step 1: Pick the Right Abstract Model

Prefer the simplest model that matches the observable behavior.

- Membership-only structure: use a finite set.
- Key-value structure: use a finite map.
- Recursive tree/trie: use `model(node)` or, if clearer, `elements(node)`.
- Packed layout: separate domain/layout facts from semantic meaning.

For set/map models, prefer the verified shims over rolling your own theory
(see verified-ecosystem.md): `moonbit-community/verified` provides `@fset` (`Fset[T]`,
backed by Why3 `set.Fset`) and `@fmap` (`Fmap[K, V]`, backed by the bundled `fmap.Fmap`
theory). Add the dependency once, import it per package — never repeat raw
`#proof_import` stubs in every client.

Example:

```moonbit
fn model(set : HashSet) -> Fset[Int] {
  match set.root {
    None => @fset.fset_empty()
    Some(node) => model(node)
  }
}
```

If a recursive helper really denotes the element set of a subtree, this is also reasonable:

```moonbit
fn elements(node : Node) -> Fset[Int] {
  match node {
    Empty => @fset.fset_empty()
    Branch(l, x, r) => elements(l).union(elements(r)).add(x)
  }
}
```

Avoid putting the whole semantics directly into every contract.

## Step 2: Keep the Invariant Small

The invariant should mostly describe:
- shape
- bounds
- layout
- well-formedness

Semantic equalities usually belong in postconditions or lemmas, not inside `*_inv`.

Good:

```moonbit
predicate sparse_ok(sa : SparseArray) {
  sa.data.length() == count_value(sa.bitmap, 0) &&
  (∀ i : Int,
    valid_idx(i) && mem_value(sa.bitmap, i) →
      0 <= rank_value(sa.bitmap, i, 0) &&
      rank_value(sa.bitmap, i, 0) < sa.data.length())
}
```

Not good:
- putting every update theorem into the invariant
- encoding the entire semantic equality into `*_inv`

## Step 3: Use Named Postconditions

Prefer named predicates over repeating large formulas. As a default naming convention, use `*_inv`, `*_pre`, and `*_post`.

Good:

```moonbit
predicate singleton_post(res : SparseArray, idx : Int, value : Int) {
  sparse_ok(res) &&
  model(res).eq(@fmap.fmap_empty().add(idx, value))
}

pub fn singleton(idx : Int, value : Int) -> SparseArray where {
  proof_require: valid_idx(idx),
  proof_ensure: result => singleton_post(result, idx, value),
} {
  ...
}
```

Note: `result => ...` only fits non-Unit returns; a Unit-returning function needs a
plain Bool `proof_ensure`.

This keeps contracts short and gives the solver a reusable target.

## Step 4: Put the Math in `.mbtp`

Proof-side material belongs in `.mbtp`.

Examples:

```moonbit
fn model(t : Tree) -> Fset[Int] {
  match t {
    Empty => @fset.fset_empty()
    Node(l, x, r, _) => model(l).union(model(r)).add(x)
  }
}

predicate avl(t : Tree) {
  match t {
    Empty => true
    Node(l, x, r, h) =>
      avl(l) &&
      avl(r) &&
      all_lt(model(l), x) &&
      all_gt(x, model(r)) &&
      h == 1 + max2(height(l), height(r))
  }
}
```

Keep `.mbtp` focused on logic definitions, predicates, and lemmas.
When a structural measure like `height` is needed on BOTH sides, define it once in
`.mbt` under `#proof_pure` instead of duplicating it.

Two recurring helper patterns are especially useful:

1. Extensional equality hypotheses for abstract structures.

```moonbit
predicate fmap_eq_hyp(m1 : Fmap[Int, Int], m2 : Fmap[Int, Int]) {
  (∀ k : Int, m1.mem(k) == m2.mem(k)) &&
  (∀ k : Int, m1.mem(k) → m1.find(k) == m2.find(k))
}

lemma fmap_eq_intro(m1 : Fmap[Int, Int], m2 : Fmap[Int, Int]) where {
  proof_require: fmap_eq_hyp(m1, m2),
  proof_ensure: m1.eq(m2),
} {
}
```

2. Small transport lemmas for updates (add/remove `mem`/`find` at self and other
keys; cardinality after add/remove). Prefer several small transport lemmas over one
giant "everything changed correctly" theorem.

## Step 5: Guide the Solver in `.mbt`

After constructing data, assert the concrete facts the solver may miss. Use `proof_assert`
after record construction, array writes, case splits, and after loop bodies establish a
stronger relation. Prefer this over introducing a callable trusted wrapper function.

## Step 6: Write Loop Invariants Early

Any loop that is relevant to the proof should get invariants as soon as the loop
shape stabilizes. A bare loop over array reads yields an unprovable bounds VC
("index in array bounds" → Timeout) because nothing establishes `0 <= i`.

Typical invariants:
- index bounds
- relationship between the accumulator and the abstract model so far
- prefix/suffix copy facts
- preservation of unchanged fields

Example (annotated loop with `proof_yield` and `proof_reasoning`):

```moonbit
for i = 0, j = xs.length(); i < j; {
  let h = i + (j - i) / 2
  if xs[h] < key {
    proof_assert ∀ idx : Int, (0 <= idx) && (idx < h + 1) → xs[idx] < key
    continue h + 1, j
  } else if key < xs[h] {
    proof_assert ∀ idx : Int, (h <= idx) && (idx < xs.length()) → key < xs[idx]
    continue i, h
  } else {
    proof_assert xs[h] == key
    break Some(h)
  }
} nobreak {
  None
} where {
  proof_invariant: 0 <= i,
  proof_invariant: i <= j,
  proof_invariant: j <= xs.length(),
  proof_invariant: ∀ idx : Int, (0 <= idx) && (idx < i) → xs[idx] < key,
  proof_invariant: ∀ idx : Int, (j <= idx) && (idx < xs.length()) → key < xs[idx],
  proof_yield: res => binary_search_ok(xs, key, res),
  proof_reasoning: (
    #| Shrinking window `[i, j)`: before `i` all `< key`; from `j` all `> key`.
  ),
}
```

Rules:
- `proof_yield` states what the loop's yielded value (from `break` and the `nobreak`
  default) satisfies; only allowed on loops yielding non-Unit values.
- `proof_reasoning` records the proof idea: `#|` block form, plain string, or single
  `#|` line.
- `proof_invariant` may repeat; every line must hold initially and be preserved.
- For array-copy loops, use staged invariants (`*_prefix_ok` at `i`, strengthened after
  the inserted/removed element is handled).
- Optional lints for unannotated loops: `warnings = "+missing_invariant+missing_reasoning"`.

## Step 7: Verify the Natural API Surface

If the public API is method-oriented, verify the methods directly.

```moonbit
pub fn HashSet::contains(self : HashSet, key : Int) -> Bool where {
  proof_require: set_inv(self),
  proof_ensure: result => result == model(self).mem(key),
} {
  ...
}
```

Use top-level verified helper functions only when they improve structure or reuse,
not as a workaround for method contracts.

## Step 8: Use Structural Proof Shape for Recursive Code

For recursive data structures:
- define a semantic view like `model(node)` or `elements(node)`
- define a shape invariant like `node_ok(node, level)`
- recurse structurally
- add `proof_decrease`

```moonbit
fn contains_at(node : Node, key : Int, level : Int) -> Bool where {
  proof_decrease: node,
  proof_require: node_ok(node, level),
  proof_ensure: result => result == model(node).mem(key),
} {
  match node {
    Flat(k) => key == k
    Branch(children) => ...
  }
}
```

If the solver resists tail-recursive loops in contracted functions, try structurally
recursive code first. Keep mutation local — escaping `FixedArray` uses are not
supported by the verifier.

## Step 9: For Packed/Indexed Representations, Prove Concrete Updates First

When a representation is packed, indexed, or incrementally rebuilt, do not jump
straight from low-level mutation to the final semantic theorem.

A common progression:
1. basic domain or indexing facts
2. local bounds or position facts
3. concrete update facts for unchanged and changed regions
4. a full concrete-update predicate
5. the final semantic `*_post` theorem

```moonbit
predicate update_prefix_ok(before_data, after_data, upto) { ... }
predicate update_middle_ok(before_data, after_data, pos, value, upto) { ... }
predicate update_data_ok(before, key, value, after) { ... }

lemma update_model_lemma(...) where {
  proof_require: update_data_ok(...),
  proof_ensure: update_post(...),
} {
  ...
}
```

A ladder like `*_prefix_ok`, `*_fill_ok`, `*_data_ok` is one useful instance, not a
universal template. See `sparse_array` in moonbit-community/verified for the reference shape.

## Step 10: Keep Shared Shim Packages Small

Put reusable proof imports or theories in shim packages (finite-set wrappers,
finite-map wrappers, bitmap domain/rank/count helpers).

The verified shim pattern:

```moonbit
// shim.mbt — abstract carrier on the program side
///|
#proof_external("set.Fset", "fset")
pub type Fset[T]
```

```moonbit
// shim.mbtp — raw stubs (parameter order matches the Why3 symbol!), then wrappers
#proof_import("set.Fset")
fn raw_fset_mem(x : Int, s : Fset[Int]) -> Bool = "mem"
```

Rules:
- the string body names a Why3 symbol (`"mem"`, `"(==)"`); it is proof-stub syntax,
  not FFI, and these mappings are a trusted boundary
- one theory family per shim; keep it minimal — large shared lemma sets can perturb
  unrelated proofs
- if a helper serves one package, keep it local instead of exporting from the shim

## Step 11: Treat Trust as Temporary

Trusted helpers are acceptable as narrow bridges, but they should not be the design
endpoint. The full trusted surface is: mathematical-integer modeling (no overflow
checks), every `proof_axiomatized` item, and every `#proof_external`/`#proof_import`
signature mapping.

If trust is unavoidable:

```moonbit
fn singleton_bridge(res : SparseArray, idx : Int, value : Int) -> Unit where {
  proof_axiomatized: true,
  proof_require: valid_idx(idx),
  proof_require: res.data.length() == 1,
  proof_require: res.data[0] == value,
  proof_ensure: singleton_ok(res, idx, value),
} {
  ()
}
```

Keep preconditions concrete, target one named predicate, keep the mathematical
statement in `.mbtp`. Then remove trusted bridges in this order:
1. constructors
2. observers
3. update functions
4. primitive machine-word bridges

## Debugging Rule: Inspect the Actual Failure First

After a proof failure:
- run `moon prove <pkg>`
- inspect the report at `_build/verif/[<pkg-dir>/]<pkg-name>.proof.json`

Read `failures[].task_pretty` (the exact Why3 local context and goal) before editing.
Classify the problem: missing arithmetic/index fact, missing semantic bridge, bad
quantifier instantiation, solver perturbation from a new lemma, or frontend/lowering
limitation. Different causes need different fixes (full failure taxonomy and fix map:
debugging-cookbook.md).

Critical semantic: false obligations usually surface as **Timeout**, not Invalid —
treat a persistent timeout as "likely false or under-specified", not merely slow.
The default strategy (MoonBit_Auto) runs Alt-Ergo/Z3 at 0.2s, then 1s, then
`split_vc` at 2s; goals that outlive the first quick pass appear in `slow_goals`.

Reproducer strategy:
- isolate the construct in a tiny probe package (`proof_enabled = true`)
- check whether `moon check` fails, `moon prove` crashes, or the VC merely times out
- inspect the generated WhyML (`_build/verif/*.mlw`) and, for lowering issues, rerun
  the printed `moonc-prove` command with `-emit-only`
- only then decide whether the issue is modeling, solver guidance, or compiler lowering

## Regression Discipline

After every proof edit:

```text
moon check <pkg>
moon prove <pkg>     # exit 0 = all goals proved; 1 = failure
moon test <pkg>      # if runtime code changed
```

Single-package `moon prove <pkg>` assumes dependencies rather than reproving them —
run module-wide `moon prove` before handoff, and rerun dependent packages after
editing any shared proof layer.

## AVOID — Anti-Patterns

Each with its verified consequence:

- starting proof work before `proof_enabled` + a trivial self-proof — `moon prove` silently skips the package
- repeating raw `#proof_import` stubs per client — duplicates the trusted boundary; centralize in one shim
- type declarations, `#proof_pure`, or executable code in `.mbtp` — parse error [3002];
  `#proof_import` stubs in `.mbt` trigger invalid-inline-wasm [0044]
- `result => ...` in `proof_ensure` of Unit-returning functions — type mismatch [4014]; use a plain Bool
- bare loops over arrays without bounds invariants — unprovable "index in array bounds" VC reported as Timeout
- trusting a Timeout as "just slow" — false obligations surface as Timeout, not Invalid, under SMT
- large inline contract formulas instead of named predicates — blows up solver budget, no reusable target
- changing abstraction design and solver guidance in one step — cannot attribute the next failure
- adding many helper lemmas without checking the proof report — new lemmas can perturb unrelated proofs
- storing semantic theorems only in trusted `.mbt` functions — silently widens the trusted surface
- verifying methods first when top-level functions would be simpler — harder VCs, no structural benefit
- introducing generic abstractions too early — a monomorphic first slice proves measurably faster
- relying on `moon prove <pkg>` alone before handoff — single-package mode assumes dependency proofs

## Minimal Checklist

Before handing off a verified MoonBit change, confirm:
- environment gate passed: `proof_enabled = true`, Why3 + ≥1 solver, trivial self-proof green
- a semantic `model(...)` exists, or there is a clear reason for `elements(...)`/`domain(...)`
- a `*_inv` predicate exists; contracts use named `*_pre` / `*_post` where appropriate
- extensional equality is handled explicitly when the abstract model is a set/map
- proof-specific logic is mostly in `.mbtp`; dual-side helpers use `#proof_pure` in `.mbt`
- runtime code has local `proof_assert` where needed; proof-relevant loops have
  `proof_invariant`, `proof_yield` (non-Unit loops), and `proof_reasoning`
- the trusted surface (`proof_axiomatized`, theory bridges) is explicit and small
- `moon check`, module-wide `moon prove`, and any needed `moon test` ran; exit codes checked
