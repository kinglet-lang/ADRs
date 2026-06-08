# 0013 — Bootstrap Bytecode Delta Analysis

- **Status**: accepted
- **Proposed**: 2026-06-08
- **Completed**: —

## Context

Self-host round-trip reaches a fixed point (`S3 == S4`), but bootstrap `compiler.kbc` (C++ kinglet) and selfhost `S3` are not byte-identical. We need a repeatable way to measure the delta and prioritize codegen alignment work.

Tooling:

- `tests/tools/kbc_diff.py` — structured section / enum / per-function diff
- `tests/differential/kbc_bootstrap_diff.sh` — builds S2/S3 and runs the diff (informational; non-failing)

## Findings (2026-06-08 snapshot)

| Metric | Bootstrap | S3 (selfhost) | Delta |
|--------|-----------|---------------|-------|
| File size | 214,468 B | 215,675 B | +1,207 B |
| Constants | 1,199 | 1,194 | −5 |
| Instructions | 34,150 | 34,396 | +246 |
| Functions | 442 | 442 | 0 |
| Identical insn prefix | — | — | 403 insns |

**First instruction divergence** at index 403: bootstrap emits `type_int` (`EnumVariant type_idx=10`); selfhost emits the start of `make_type` (`LoadLocal`). Same 403-instruction prefix, then function emission order diverges.

**Enum metadata** (root cause of index drift): selfhost registers two extra enums before parser AST enums — `IntResult` and `FloatResult` — so bootstrap `TokenType` at index 1 becomes selfhost index 3, and `TypeKind` shifts from 10 → 12. Selfhost has 13 enum metas vs bootstrap 11.

**Struct metadata**: 44 vs 43 (−1 struct meta in selfhost).

**Constant pool**: bootstrap has 5 more strings; both have 1 char, 23 int, 442 function constants.

**Largest per-function instruction deltas (S3 − bootstrap)**:

| Δ | Function |
|---|----------|
| +18 | `bind_match_pattern` |
| +16 | `load_module`, `compile_match`, `bind_enum_pattern` |
| +12 | `assignment`, `_compile_enum_arm` |
| +10 | `emit_method_call`, `emit_call`, `compile_program`, `_compile_array_arm` |

Many small helpers differ by +2 insns (extra `Jmp`/`Null`/`Return` epilogue patterns in match lowering).

After the first mismatch, all 442 function entry offsets cascade (same names, shifted entries).

## Decision

1. Track bootstrap byte parity as a **separate informational metric** from round-trip fixed-point (`S3 == S4`).
2. Use `kbc_bootstrap_diff.sh` after codegen changes to see whether delta shrinks.
3. Fix priority for parity (when pursued):
   - **Enum metadata registration order** — align `add_enum_meta` traversal with bootstrap `compiler.cc` so `EnumVariant` operands match.
   - **Match epilogue codegen** — `compile_match`, wildcard/binding arms, `bind_match_pattern`.
   - **`load_module` transitive registration** — import graph side effects on chunk metadata.
   - **+2 insn helper tails** — systematic extra jumps in selfhost-compiled small functions.

Full byte-identical bootstrap parity is **not blocking** self-hosting correctness while regression + round-trip pass.

## Consequences

- CI can run `kbc_bootstrap_diff.sh` without failing the suite (parity is reported, not gated).
- Enum index drift explains early divergence; fixing registration order may realign a large prefix but not all +246 insns.
- String constant deduplication differences may account for size delta independent of instruction count.
