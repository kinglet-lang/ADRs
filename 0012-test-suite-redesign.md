# 0012 — Test Suite Redesign

- **Status**: implemented
- **Proposed**: 2026-06-08
- **Completed**: 2026-06-08

## Context

The test suite grew feature-by-feature into ~11 ad-hoc shell scripts. Three
structural problems make green runs unreliable as a stability signal:

1. **Mixed pipelines.** Some suites compile through the **bootstrap** C++ binary
   (`tests/run/`, `tests/regression/`), others through the **selfhost**
   `compiler.kbc` (`tests/probe/`, `tests/checker/`). A green `tests/run/`
   therefore does **not** prove the selfhost compiler can run the program.

2. **Dead scaffolding.** `tests/run/run_golden.sh` references a
   `cases/known_gaps/` directory that does not exist; its four `xfail` entries
   are empty stubs. `tests/codegen/` goldens are currently 22/22 stale.

3. **No shared harness.** Each script re-implements `resolve_kinglet` /
   `ensure_cli_kbc` and its own assertion logic. There is no way to declare, per
   case, which pipeline to run or what to assert beyond bespoke bash.

Kinglet is **self-hosting**: two implementations exist — the bootstrap compiler
(C++, separate repo) and the selfhost compiler (`compiler.kbc`, this repo). This
is the single most important fact for test design and the current suite barely
exploits it.

### What does not apply here

General compiler-testing doctrine assumes an IR, an optimizer, an ownership
checker, and a C++ compiler binary. Kinglet has none of these:

- **No IR / optimizer** — `compiler/codegen.kl` lowers AST directly to VM
  bytecode. There is nothing to test between AST and bytecode.
- **No ownership / borrow checking** — see [0002](0002-design-principles.md)
  (value semantics, no borrow checker).
- **The compiler is written in Kinglet, not C++** — sanitizer passes
  (ASan/UBSan/TSan) belong to the bootstrap VM repository, not to `*.kl`.

So `ir/`, `opt/`, ownership tests, and sanitizer CI are explicitly out of scope.

## Decision

Restructure tests around **pipeline + semantics + end-to-end behavior**, driven
by a single harness with in-file directives. Differential testing
(bootstrap vs selfhost) is the backbone, not an advanced extra.

### Principles

1. Every layer has its own input, output, and assertion standard. Do not verify
   everything by "compile and run".
2. Every case declares its **pipeline** explicitly.
3. Diagnostics assert **error code + primary location + core text only** — never
   full message text (formatting still changes).
4. Each fixed bug adds a permanent regression case.

### Target layout

```
tests/
  harness/
    run.sh          # case discovery, directive parsing, pipelines, assertions
    directives.md   # executable spec for the directive language
  lexer/            # selfhost scanner → token golden
  parser/           # selfhost parser → AST golden / CHECK
  sema/
    pass/           # --check must pass
    fail/           # must error: code + location + core text
  codegen/          # --bytecode golden (the target artifact)
  exec/             # selfhost compile + run, end-to-end
  diagnostics/      # error messages (incl. COMPILE-FAIL negatives)
  differential/     # bootstrap vs selfhost on identical sources
  regression/       # one case per fixed bug (dual pipeline)
  property/         # AST/token dump stability + fuzz-lite (no crash/hang)
  common.sh
  run_all.sh
```

`ir/` and `opt/` are omitted (no such stages). `negative/` is folded into
`diagnostics/` via `COMPILE-FAIL`. `runtime/` is deferred until a real
`stdlib/` exists ([0003](0003-stdlib-roadmap.md)); `tests/builtin_methods/`
covers builtin methods in the interim.

### Harness directive language

Cases carry directives in header comments; `harness/run.sh` parses them.

| Directive | Meaning |
|-----------|---------|
| `RUN: <pipeline>` | `selfhost` \| `check` \| `diff` \| `bytecode` \| `ast` |
| `EXPECT-STDOUT:` / `EXPECT-STDERR:` / `EXPECT-EXIT:` | exact assertions |
| `CHECK:` / `CHECK-NOT:` | stdout contains / does not contain |
| `CHECK-ERR:` | stderr contains substring |
| `CHECK-ERR-AT:` | primary diagnostic at `line:col` |
| `COMPILE-FAIL` | compilation must exit non-zero |

| Pipeline | Commands | Asserts |
|----------|----------|---------|
| `selfhost` | `kbc --save-bytecode tmp <f>` then `run tmp` | stdout / exit |
| `check` | `kbc <f> --check` | stderr pattern, pass/fail |
| `diff` | bootstrap `<f>` vs selfhost `<f>` | outputs identical |
| `bytecode` | `kbc --bytecode <f>` | golden text |
| `ast` | `kbc --ast <f>` | `CHECK` substrings |

Bootstrap follows selfhost for `?:` (null Elvis). `??` remains bootstrap-only for
Result/cast propagation ([0006](0006-error-handling-unification.md)). `diff`
cases expect behavioral equality unless explicitly marked otherwise.

### Phased rollout

Each phase is independently committable and reversible. No compiler changes.

| Phase | Deliverable | Status |
|-------|-------------|--------|
| 0 | `harness/run.sh` + `directives.md`; `selfhost`/`check`/`diff` pipelines | done |
| 1 | `exec/` migrated from `run/` with `// RUN: selfhost`; remove dead `known_gaps` xfail stubs | done |
| 2 | `differential/` gate + `run_matrix.sh`; regression selfhost oracle + drift report | done |
| 3 | `checker/` → `sema/{pass,fail}`; SYNTAX backfill (map, nullable, generic struct, UFCS) | done |
| 4 | refresh `codegen/` goldens; smoke compile+run with `.exit` sidecars | done |
| 5 | `property/` AST/token stability + fuzz-lite | done |

### CI tiers

```
Per commit (fast):
  lexer parser sema exec(smoke) differential(MUST_PASS) regression

Pre-merge / nightly (slow; includes ~85s compiler.kbc rebuild):
  full exec / codegen / full differential / round-trip / short fuzz
```

No IR/optimizer tier and no sanitizer tier (out of scope, see Context).

## Consequences

### Positive

- Green means **selfhost works**, not "bootstrap works".
- Bootstrap/selfhost drift is caught structurally by `differential/`.
- One harness replaces ~11 bespoke scripts; new cases are declarative.
- Diagnostics assertions survive cosmetic message changes.

### Risks and constraints

- **Rebuild cost**: dual-pipeline runs rebuild `compiler.kbc` (~85s). Mitigated
  by `ensure_cli_kbc` caching and fast/slow CI tiers.
- **bash 3.x on macOS**: no associative arrays; use the established `eval`
  counter pattern.
- **Legitimate divergence**: `diff` cases must allow marked exceptions, or they
  will fail on known sh-vs-bs semantic differences.
- **Scope creep**: enforce phase boundaries; one commit per phase.

### Migration notes (completed)

- `tests/run/` → `tests/exec/` (`RUN: selfhost`). `run/run_golden.sh` forwards.
- `tests/checker/` → `tests/sema/{pass,fail}/`. `checker/run_golden.sh` forwards.
- `tests/regression/` asserts selfhost against hand-verified oracles; reports
  bootstrap drift non-gating on runtime cases.
- `tests/differential/run_diff.sh` renamed to `run_matrix.sh` (snapshot);
  gating cases live in `differential/cases/` with `RUN: diff`.
- `tests/probe/` and `tests/builtin_methods/` remain capability snapshots.
- Operator guide: [tests/README.md](../tests/README.md).

## Dependencies

- [0002](0002-design-principles.md) — no ownership tests (value semantics).
- [0003](0003-stdlib-roadmap.md) — `runtime/` tier waits on a real stdlib.
- [0006](0006-error-handling-unification.md) — sh-vs-bs `?:`/`??` divergence
  informs `diff` exceptions.
- [0011](0011-module-system-redesign.md) — module/import cases in `sema`/`exec`.

## References

- Suite index: [tests/README.md](../tests/README.md)
- Harness spec: [tests/harness/directives.md](../tests/harness/directives.md)
- Shared helpers: `tests/common.sh` (`resolve_kinglet`, `resolve_bootstrap`, `ensure_cli_kbc`, `run_with_timeout`)
- Capability snapshots: `tests/probe/README.md`, `tests/builtin_methods/README.md`
