# ADR 0016: Typed KIR for Native Lowering

## Status

Accepted (2026-06-10)

## Context

The native backend used a single `int64_t` wire format (`kl_h`) for all values.
Arithmetic, comparisons, and casts went through runtime tag dispatch even when
the Kinglet type checker had already proved scalar types. That caused:

- Float constants boxed on every use
- `int64` values in the heap-tag range misclassified as heap references
- `bool`/`null` printing as `1`/`0` when cast to string

## Decision

Introduce a flat `KirType` on KIR metadata and a per-function inference pass.
The type checker exports function signatures and struct field types into
`KirModule`; `infer_kir_types()` annotates each instruction with a result type.
LLVM lowering uses these types to emit raw `i64`/`double` ops for known scalars
and only boxes at wire boundaries (returns, runtime calls, polymorphic fallbacks).

Bool and null casts to `string` are allowed in the type checker; native lowering
calls `kl_bool_to_string` / `kl_null_to_string` when the inferred source type
is `Bool` or `Null`.

## Consequences

- Scalar hot paths avoid `kl_value_*` dispatch when inference is precise
- KIR dump and goldens include type annotations (`: int`, `-> int`)
- Container element unboxing and fixed-width `int8`–`int64` types remain future work
- Inference is conservative: control-flow merges widen to `Any`
