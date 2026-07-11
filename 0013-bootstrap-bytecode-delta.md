# 0013 — Bootstrap Bytecode Parity

- **Status**: deprecated (implemented; bytecode parity retired by [0022](0022-native-unique-ownership.md))
- **Proposed**: 2026-06-08
- **Completed**: 2026-06-09

**Current replacement**: [0022 — Native-Only Toolchain and Unique Ownership](0022-native-unique-ownership.md). Byte-identical KBC parity is retired; current validation compares native behaviour.

## Context

Self-host round-trip reaches a fixed point (`S3 == S4`). We also want bootstrap
`compiler.kbc` (C++ kinglet compiling `core/main.kl`) to be **byte-identical**
to selfhost `S3` so the VM-hosted compiler matches the reference toolchain output.

Tooling:

- `tests/tools/kbc_diff.py` — structured section / enum / per-function diff
- `tests/differential/kbc_bootstrap_diff.sh` — builds S2/S3 and compares to bootstrap
- `tests/selfhost/run_roundtrip.sh` — fixed-point **and** bootstrap parity (gated)

## Historical delta (2026-06-08, before parity work)

| Metric | Bootstrap | S3 (selfhost) | Delta |
|--------|-----------|---------------|-------|
| File size | 214,468 B | 215,675 B | +1,207 B |
| Instructions | 34,150 | 34,396 | +246 |
| Identical insn prefix | — | — | 403 insns |

Root causes included: extra `IntResult`/`FloatResult` enum metas, match epilogue
codegen, import alias registration, loop break/continue stacks, constant pool
double-unescape (`"\\n"`), parser stmt locations, pass2b namespace order
(`expressions` vs `keywords`), and struct-literal debug columns.

## Resolution (2026-06-09)

| Metric | Bootstrap | S3 | Delta |
|--------|-----------|-----|-------|
| File size | 215,766 B | 215,766 B | 0 |
| Constants | 1,211 | 1,211 | 0 |
| Instructions | 34,365 | 34,365 | 0 |
| Functions | 440 | 440 | 0 |
| Enums | 11 | 11 | identical order |

`cmp -s compiler.kbc S3.kbc` passes. Round-trip `S3 == S4` still holds.

Key fixes (commits `d5fc0b2` … `4a9d34b`):

1. CastError-only builtin enums; reverse symbol resolution; per-loop break/continue.
2. Match/import/pass2b/return-type codegen alignment with bootstrap.
3. Raw string literal text in parser AST (`strip_string_quotes`); single unescape at emit.
4. Expression locations for expr stmts and arrow bodies; struct-lit start token for debug.
5. `pass2b_ns_rank`: `expressions`(12) before `keywords`(13) for libc++ map order.

## Decision

1. **Gate** bootstrap byte parity in `run_roundtrip.sh` (Step 5) and
   `kbc_bootstrap_diff.sh` (`exit 1` on mismatch).
2. Keep `kbc_diff.py` for diagnosing regressions when parity breaks.
3. `pass2b_ns_rank` remains platform-sensitive (macOS libc++ observation); re-verify
   parity when changing toolchain or adding imported modules.

Parity applies to **`core/main.kl` → `compiler.kbc`**. Small differential matrix
cases (guard, pipe, generics) may still differ in bytecode while behavior matches.

## Consequences

- Codegen changes must preserve `compiler.kbc == S3` or update bootstrap and selfhost
  together with an documented reason.
- Refresh `tests/codegen/cases/*.bytecode` via `tests/codegen/refresh_goldens.sh`
  after intentional bytecode output changes.
- New VM opcodes must be **appended** after existing ordinals (see `chunk.h`); e.g.
  `DenseArrayNew` ([0017](0017-dense-nested-array-layout.md)) — refresh goldens when
  bootstrap emits it for rectangular literals.
- Enum/constant pool drift in small programs is tracked separately in differential
  matrix (non-gating).

## Amendments

### 2026-06-20 — Bytecode parity retired by native-only validation ([0022](0022-native-unique-ownership.md))

This ADR was implemented for the bytecode era, but byte-identical KBC parity is no
longer an active validation target. [0022](0022-native-unique-ownership.md) retires
VM/KBC validation in favour of native behavioural comparison. Bootstrap commit
`ba20344` (`refactor(vm): remove VM interpreter and kbc execution paths`,
2026-06-20) removed the VM interpreter and KBC execution path; commit `54df929`
(`refactor(build): delete dead bytecode container`, 2026-07-10) removed the dead
bytecode container left after the IR vocabulary migration.

Keep this document as the historical parity target; do not use it as a current
self-host gate.
