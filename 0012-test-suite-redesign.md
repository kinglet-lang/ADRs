# 0012 — Test Suite Redesign

- **Status**: accepted
- **Proposed**: 2026-06-08

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
  property/         # round-trip: src → ast → print → parse equivalence
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

Where bootstrap and selfhost legitimately differ (e.g. `?:` vs `??`, see
[0006](0006-error-handling-unification.md)), the `diff` case marks the expected
divergence rather than forcing equality.

### Phased rollout

Each phase is independently committable and reversible. No compiler changes.

| Phase | Deliverable | Acceptance |
|-------|-------------|------------|
| 0 | `harness/run.sh` + `directives.md`; `selfhost`/`check`/`diff` pipelines | harness runs new sample cases; old scripts untouched |
| 1 | `exec/` migrated from `run/` with `// RUN: selfhost`; remove dead `known_gaps` xfail stubs | exec runs through selfhost; `run/` retired |
| 2 | `differential/`; promote `regression/` MUST_PASS to dual pipeline | any new bootstrap/selfhost drift is visible |
| 3 | split `checker/` → `sema/{pass,fail}`; backfill from SYNTAX.md (map, generic struct fields, concepts, nullable, UFCS) | each major feature ≥1 pass + ≥1 fail |
| 4 | repair/refresh `codegen/` goldens; add compile+run dual assertion on key cases | codegen green with a runtime backstop |
| 5 | `property/` round-trip; fuzz-lite (random bytes → lexer/parser → no crash) | round-trip + one short fuzz run in CI |

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

### Migration notes

- `tests/run/` (bootstrap) → `tests/exec/` (selfhost). Retire after Phase 1.
- `tests/regression/` keeps its oracle-anchored cases but runs them through
  `differential/` (both pipelines) instead of bootstrap only.
- `tests/checker/` → `tests/sema/{pass,fail}/`.
- `tests/run/run_golden.sh` dead `known_gaps/` xfails are deleted or converted
  to real `diagnostics/` cases.
- `tests/probe/` and `tests/builtin_methods/` remain as selfhost capability
  snapshots; their cases may later adopt harness directives.

## Dependencies

- [0002](0002-design-principles.md) — no ownership tests (value semantics).
- [0003](0003-stdlib-roadmap.md) — `runtime/` tier waits on a real stdlib.
- [0006](0006-error-handling-unification.md) — sh-vs-bs `?:`/`??` divergence
  informs `diff` exceptions.
- [0011](0011-module-system-redesign.md) — module/import cases in `sema`/`exec`.

## References

- Current suites: `tests/{lexer,parser,checker,codegen,run,run_selfhost,diagnostics,kbc,regression,probe,builtin_methods}/`
- Shared helpers: `tests/common.sh` (`resolve_kinglet`, `ensure_cli_kbc`)
- Capability snapshots: `tests/probe/README.md`, `tests/builtin_methods/README.md`
