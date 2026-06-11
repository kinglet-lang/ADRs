# 0016 — Typed KIR for Native Lowering

- **Status**: implemented (phase 1)
- **Proposed**: 2026-06-10
- **Completed**: 2026-06-10

## Context

The native backend used a single `int64_t` wire format (`kl_h`) for all values.
Arithmetic, comparisons, and casts went through runtime tag dispatch even when
the Kinglet type checker had already proved scalar types. That caused:

- Float constants boxed on every use
- `int64` values in the heap-tag range misclassified as heap references
- `bool`/`null` printing as `1`/`0` when cast to string

## Decision — Phase 1 (implemented)

Introduce a flat `KirType` on KIR metadata and a per-function inference pass.
The type checker exports function signatures and struct field types into
`KirModule`; `infer_kir_types()` annotates each instruction with a result type.
LLVM lowering uses these types to emit raw `i64`/`double` ops for known scalars
and only boxes at wire boundaries (returns, runtime calls, polymorphic fallbacks).

Bool and null casts to `string` are allowed in the type checker; native lowering
calls `kl_bool_to_string` / `kl_null_to_string` when the inferred source type
is `Bool` or `Null`.

### Phase 1 exit criterion

- `KirType` on instructions; KIR dump goldens include `: int` / `-> int`
- Native smoke: `bool_null_print`, `int64_high` green alongside existing manifest
- VM parity unchanged for programs not using typed paths

## Phase 2 (planned — V0 / follow-up)

Not required to close [0015](0015-llvm-backend-roadmap.md) or this ADR.

| Item | Notes |
|------|--------|
| CFG merge typing | Join types at `CondBr` merge points instead of widening to `Any` |
| Container unboxing | `Array`/`Map` element loads when field types are known scalars |
| Fixed-width types | Surface `int8`–`int64`, `float32`/`float64` per [0015](0015-llvm-backend-roadmap.md) D7 |
| Generic `io` / `+` | Unified bool/null printing without wire `1`/`0` on format paths |

See project `Kinglet-TODO.md` and [native.md](../docs/native.md) known gaps.

## Consequences

- Scalar hot paths avoid `kl_value_*` dispatch when inference is precise
- Phase 2 items remain tracked in [0015](0015-llvm-backend-roadmap.md) D6 open rows
  and V0 backlog; they do not block ADR 0014/0015 closure
