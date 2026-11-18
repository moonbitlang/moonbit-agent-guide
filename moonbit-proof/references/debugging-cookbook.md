# Proof Debugging Cookbook (Empirically Verified)

Load this reference when a proof fails, times out, crashes, or when triaging `moon prove` output. All report fields and behaviors verified against moon 0.1.20261023 / moonc v0.10.17 (2026-10-23).

## Console Output Semantics

Success:
```text
proofy/verify
  Succeeded: 5 goals proved

Summary:
  2 of 2 packages proved
  6 goals proved
```

Failure (with actionable pointers):
```text
Error: failed to prove proofy/verify (wasm): /Users/.../moonc-prove -error-format json ... -whyml-output-path _build/verif/pkg_..._.mlw -proof-report-output-path _build/verif/verify.proof.json ...
proofy/verify
  Failed: 1 goals proved, 1 timeout
  WhyML: _build/verif/pkg_6_proofy_6_verify.mlw
  Report: _build/verif/verify.proof.json
  Failed goals: 1

Summary:
  0 of 1 packages proved
  1 goals proved, 1 timeout
```

The `Error:` line reveals the full `moonc-prove` invocation — reuse it directly to reproduce with `-emit-only` or custom flags.

## Report JSON Schema (Verified)

Path: `_build/verif/[<pkg-dir>/]<pkg-name>.proof.json`

```json
{
  "result": "success | proof_failure",
  "summary": {
    "valid": 1, "invalid": 0, "timeout": 0,
    "oom": 0, "step_limit": 0, "unknown": 0, "failure": 0
  },
  "failures": [
    {
      "theory": "Pkg_6_proofy_6_verify",
      "goal": "bad_add_one'vc",
      "full_name": "Pkg_6_proofy_6_verify.bad_add_one'vc",
      "location": { "file": "...mlw", "line1": 163, "col1": 6, "line2": 163, "col2": 17 },
      "explanation": "postcondition",
      "headline": "cannot prove postcondition",
      "result": "Timeout",
      "task_pretty": "--------------------------- Local Context ---------------------------\n..."
    }
  ],
  "slow_goals": [
    {
      "goal": "binary_search'vc",
      "result": "Valid",
      "attempts": [ { "prover": "Alt-Ergo", "result": "Timeout" }, { "prover": "Z3", "result": "Valid" } ],
      "total_time": 0.221112
    }
  ]
}
```

Key fields:
- `goal` name: `<fn>'vc` for the function's aggregate VC; `explanation` names the obligation kind.
- Verified `explanation` values: `"VC for <fn>"`, `"postcondition"`, `"index in array bounds"`.
- `task_pretty`: the full Why3 local context + goal — read it before editing anything; it shows exactly which hypotheses the solver had.
- `slow_goals`: goals that outlived the first 0.2s quick pass. A goal proved only by the second prover/attempt is healthy but worth watching; growth here predicts future timeouts.

## Failure Taxonomy → Fix Map

| Symptom | Likely cause | First fix |
|---|---|---|
| `timeout` on `index in array bounds` | loop lacks (bounds) invariant | add `proof_invariant: 0 <= i, i <= limit` |
| `timeout` on `postcondition` | missing semantic bridge or assert | inspect `task_pretty`; add local `proof_assert` at the construction site; introduce transport lemma |
| `invalid` (rare) | contradictory spec | spec is wrong — fix the predicate, not the code |
| `unknown` / `oom` / `step_limit` | solver capacity | simplify/split the VC; add intermediate asserts; raise budgets via `--why3-config` |
| `failure` | prover crashed or lowering bug | reproduce in a probe package; check `moonc-prove -emit-only` output |
| parse error [3002] in `.mbtp` | misplaced construct (type decl, `#proof_pure`, etc.) | check the what-goes-where table in `language-reference.md` |
| "not proof-enabled; skipping" | missing package config | add `proof_enabled = true` to that package's `moon.pkg` |
| "moon.pkg.json ... no longer supported" | legacy manifest format | run `moon fmt` to migrate to `moon.mod`/`moon.pkg` |

Critical semantic: **a false proof obligation usually reports Timeout, not Invalid** — SMT provers fail to prove it and run out of budget. Treat persistent timeouts on a goal as "likely false or under-specified", not merely "slow".

## Reproducer Protocol

1. Isolate the construct in a tiny probe package (`proof_enabled = true`, one function, minimal predicates).
2. Run `moon check <probe>` — if it fails, the issue is syntax/types, not proofs.
3. Run `moon prove <probe>` — classify: check-fails / prove-crashes / VC-times-out.
4. Inspect `_build/verif/<probe>/<probe>.proof.json` (`task_pretty` especially).
5. Optionally inspect the generated WhyML `_build/verif/...mlw` to see the exact lowering (goal shape, prelude aliases, `let function` for `#proof_pure`).
6. Only then decide: modeling error, missing solver guidance, or frontend/lowering limitation.
7. For suspected lowering issues, run the printed `moonc-prove` command with `-emit-only` to separate frontend from solver.

## Solver Perturbation

- Adding lemmas to a shared shim can perturb unrelated proofs (trigger selection). Move the new lemma into the client package and re-run to confirm.
- After editing any shared proof layer, re-run `moon prove` on ALL dependent packages, not just the client.
- Cross-check flaky goals with a copied `why3.conf` (longer timelimit / different prover set) via `moon prove --why3-config`.

## Regression Discipline

```bash
moon check <pkg>        # types + syntax gate
moon prove <pkg>        # proof gate (exit 0 = all goals proved)
moon test <pkg>         # runtime still correct (contracts are logic-only; no runtime checks)
```

- `moon prove` exit code (verified): 0 = success, 1 = any failure. Wire CI to the exit code.
- Single-package mode assumes dependencies — a green `moon prove <pkg>` does NOT re-verify dependency proofs; run module-wide `moon prove` before handoff.
