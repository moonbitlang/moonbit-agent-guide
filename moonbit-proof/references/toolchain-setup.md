# Toolchain & Environment Setup (Empirically Verified)

Load this reference when setting up proof verification, diagnosing environment issues, or tuning solver behavior.

Validation stamp: verified against `moon 0.1.20261023 (3778465 2026-10-23)` / `moonc v0.10.17+d971670cd (2026-10-23)` on macOS, plus official docs at https://docs.moonbitlang.com/en/latest/language/verification.html.

## Component Map

| Component | Location / Requirement | Notes |
|---|---|---|
| `moon prove` | bundled with moon | drives whole-package proof; lowers to Why3 |
| `moonc-prove` | `~/.moon/bin/moonc-prove` | actual lowering backend invoked by moon |
| Why3 libraries | `~/.moon/lib/why3` + `~/.moon/share/why3` | 44 theory files incl. custom `fmap.mlw` |
| proof prelude | `~/.moon/lib/prelude_proof/moonbit_builtin_prelude.mlw` | math-integer models for builtin types |
| Why3 (external) | opam install, pinned `1.7.2` recommended | `why3` on PATH |
| SMT solver (≥1 required) | `z3`, `cvc5`, or `alt-ergo` | auto-detected from PATH into generated why3.conf |

Verified local combo: z3 4.15.4 (homebrew) + Alt-Ergo 2.4.3 (opam) — both detected automatically and used by the default strategy.

## Enabling Proofs Per Package

New `moon.pkg` DSL format (TOML-like):

```moonbit
proof_enabled = true
```

- Proof enablement is **package-local**: every proof-carrying package must declare it.
- `moon prove <pkg>` on a non-proof-enabled package prints: "Package ... is not proof-enabled; skipping `moon prove` for it." (not an error)
- Legacy formats are **rejected**: `moon.mod.json` and `moon.pkg.json` both fail with "no longer supported. Run `moon fmt` to migrate". Old option spellings like `options("proof-enabled": true)` exist only in legacy configs (e.g. moonbit-community/verified 0.0.2); new-format projects must use `proof_enabled = true`.

Import a proof library (e.g. the fset shim) in the package's `moon.pkg`:

```moonbit
import {
  "moonbit-community/verified/libs/fset" @fset,
}
```

Module dependency goes in `moon.mod`'s `import` block (add via `moon add moonbit-community/verified`).

## moon prove CLI Facts

- `moon prove` (no args): proves **all** proof-enabled packages in the module.
- `moon prove <path>`: single-package mode; **dependencies are assumed rather than reproved** (official docs).
- Exit codes (verified): success → 0; any proof failure or lowering error → 1.
- `--why3-config <PATH>`: use a user-supplied Why3 config instead of the generated default.
- `MOON_PROVE_PRELUDE_OVERRIDE=<dir>`: override the toolchain prelude; `<dir>` must contain `moonbit_builtin_prelude.mlw` (verified working with a copy of the stock prelude).
- `moon check` does NOT discharge proof obligations and does not require loop invariants to typecheck; `moon prove` is a separate, deeper pass.

## Generated Artifacts Layout

After `moon prove` (root package named `verify`, subpackage dir `probe`):

```text
_build/verif/
├── why3.conf                     # generated Why3 config (provers, limits, strategy)
├── verify.proof.json             # root package report: _build/verif/<pkg>.proof.json
├── pkg_6_proofy_6_verify.mlw     # generated WhyML for root package
├── probe/
│   ├── probe.proof.json          # subpackage report: _build/verif/<dir>/<pkg>.proof.json
│   └── pkg_6_proofy_6_verify_5_wloop.mlw
└── .why3harness/<hash>/
    ├── why3session.xml           # Why3 session (replayable)
    └── why3shapes
```

Report path rule: `_build/verif/[<pkg-dir>/]<pkg-name>.proof.json`.

## Generated why3.conf (Verified Defaults)

```ini
[main]
datadir = "~/.moon/share/why3"
libdir = "~/.moon/lib/why3"
memlimit = 1000          # MB per prover run
running_provers_max = 16
timelimit = 5.000000     # seconds, per goal ceiling

[strategy]  # name = "MoonBit_Auto"
# 1. Alt-Ergo 0.2s/1000MB; Z3 0.2s/1000MB
# 2. Alt-Ergo 1s | Z3 1s
# 3. compute_specified; split_vc
# 4. Alt-Ergo 2s/4000ms | Z3 2s/4000ms
```

Consequences:
- A goal that both provers cannot close within ~2s after `split_vc` reports Timeout, not Invalid.
- Goals needing more than the first 0.2s quick attempt are logged in `slow_goals` with per-prover attempt results.
- To experiment with different budgets/provers, copy the generated `why3.conf`, edit it, and pass `--why3-config`.

## moonc-prove Direct Interface (Debugging)

`moon prove` failures print the exact `moonc-prove` command line. Useful flags (from `moonc-prove --help`):

- `-whyml-output-path <path>`: keep generated WhyML at a stable path for inspection
- `-proof-report-output-path <path>` (required): structured proof report JSON
- `-emit-only`: emit WhyML + proof `.mi` **without invoking Why3** — use to inspect lowering or reproduce frontend issues
- `-why3-loadpath <dir>`: extra Why3 loadpath entry (default adds `~/.moon/lib/prelude_proof`)
- `-dep-proof <pkg>:<artifact_stem>`: map a dependency package to its generated WhyML artifact (cross-package proofs)

## Solver Installation Quick Reference

```bash
# Why3 (pinned 1.7.2 recommended)
opam install why3=1.7.2
# Solvers — at least one
brew install z3            # or: opam install alt-ergo / install cvc5
why3 config detect         # only needed for standalone why3 use; moon generates its own conf
```

`moon prove` needs no manual `why3 config` step — it writes `_build/verif/why3.conf` itself, auto-detecting provers on PATH.
