# 0017 — Dense layout for `T[][]…[]` syntax

- **Status**: implemented (v1: 2D literals)
- **Proposed**: 2026-06-12
- **Completed**: 2026-06-12

## Context

Surface syntax is stacked suffixes only: `T[]`, `T[][]`, `T[][][]`, `{K: V}`.
There is no separate `Matrix<T>` or `Buffer<T>` type.

Today every `T[][]` value is **jagged** at runtime: `Array<Array<T>>`, and
`m[i][j]` is two `IndexGet` hops through two heap objects. That matches general
semantics (rows may differ in length) but wastes memory and cache locality for
the common **rectangular** case, e.g. `[[1, 2], [3, 4]]`.

Items 1–3 (nested container typing, CFG merge, checker `Array<…>` encoding) are
orthogonal: they make chained `IndexGet` correct on the native path. This
decision adds an **optional dense representation** for the same syntax when
shape is known rectangular.

Map shares the **typing** story (`{K: V[]}`, chained index) but not **layout**:
maps are not row×column grids. No change to `KlMap` here.

## Decision

### D1 — Same syntax, optional dense storage

1. **No new surface type.** Users still write `int32[][] m` and `m[i][j]`.
2. When the compiler can prove a **rectangular** nested array value (see D2), it
   may construct a **row-major flat** `KlArray` with rank metadata instead of
   nested row objects.
3. When proof fails (unequal row lengths, dynamic construction, row mutation that
   breaks invariants), behavior stays **jagged** as today.

### D2 — When to use dense layout (v1)

**Dense eligible:**

- Nested **array literals** at compile time: `[[…], […]]`, all rows same length,
  innermost elements are scalars (or nested literals that recursively satisfy
  the same rule for `T[][][]`, etc.).
- Optional later: `array.repeat(rows, cols, fill)`-style APIs that explicitly
  request a grid.

**Stay jagged:**

- Rows built separately (`row0`, `row1` then `[row0, row1]` if rows are distinct
  heap arrays).
- Any row length mismatch in a literal.
- After `m[i].push(…)` / `m[i] = other` / resize that breaks a fixed column
  count — dense value **degrades** to jagged (copy-out) or documents UB for v1;
  prefer **copy-on-write degrade** to preserve safety.

### D3 — Runtime shape (bootstrap `KlArray`)

Extend `KlArray` with optional dense metadata (no new `KlKind` in v1):

```text
elements     — flat row-major payload (length = product(dims))
dense_dims   — empty ⇒ normal jagged vector of kl_h
               non-empty ⇒ rank = dense_dims.size(), elements interpreted as
               ∏ dense_dims[i] cells
```

Access rules:

| Operation | Dense `T[][]` | Jagged `T[][]` (unchanged) |
|-----------|---------------|----------------------------|
| `m[i][j]` read | `elements[i * cols + j]` | two `IndexGet` |
| `m[i]` | **new** `T[]` row copy (value semantics) | existing row handle |
| `m.len()` | `dense_dims[0]` (rows) | outer length |
| `m[i].len()` | `dense_dims[1]` (cols) | inner length |

Row copy on `m[i]` matches [0002](0002-design-principles.md) (copy by default)
and avoids row views aliasing the parent grid.

Native fast path: RT helper `kl_dense_index(kl_h grid, int rank, const int *indices)`
or rank-specialized `kl_dense2d_get(grid, i, j)`; LLVM lowers typed `m[i][j]`
to one call when KIR container rank ≥ 2 and value is known dense.

### D4 — Compiler / KIR

1. **AST pass** on nested `ArrayLiteralExpr`: compute `dense_dims` or mark jagged.
2. New KIR op **`DenseArrayNew`** (operands: flat element count + dim count + dims
   in constant pool) or an `ArrayNew` variant flag — prefer explicit opcode for
   dumps and LLVM.
3. **`kir_typing`**: `DenseArrayNew` pushes `Array` with full nested container
   metadata (same as nested `ArrayNew` today).
4. **`IndexGet`**: unchanged opcode; dense vs jagged resolved at runtime via
   `dense_dims.empty()`. LLVM may fuse two `IndexGet` for `T[][]` when intervening
   value does not escape (later peephole).

VM and native share the same RT functions ([0015](0015-llvm-backend-roadmap.md)).

### D5 — Rank > 2

Same model: `dense_dims = [d0, d1, d2, …]`, index
`i0, i1, …` maps to linear offset via row-major stride product. v1 implements
**`T[][]` only**; `T[][][]` follows the same rules in a follow-up once 2D is
stable.

### D6 — Explicit non-goals (v1)

- No syntax change (`m[i, j]`, `int32[2][3]`, `Matrix<T>`).
- No dense `Map` layout.
- No zero-copy row views (`m[i]` mutating parent).
- No automatic dense promotion for non-literal code paths.

## Consequences

**Positive**

- Rectangular literals allocate one buffer + metadata instead of 1 + rows objects.
- Native `m[i][j]` can become one indexed load with typing from existing KIR.
- Surface language stays `T[]` / `T[][]` only.

**Costs**

- `KlArray` grows; `kl_array_get` / `kl_index_get` branch on dense vs jagged.
- `m[i]` on dense copies a row (extra alloc on hot path if rows are read often).
- Compiler must degrade or copy-on-write when user mutates a row incompatibly.

**Tests**

- VM + native smoke: rectangular literal `[[1,2],[3,4]]`, `m[1][0]`.
- Differential: jagged literal (`row0`, `row1`) unchanged behavior.
- Sema: no type changes; optional `tests/sema/pass/dense_array_literal.kl`.

## Implementation order

1. RT: `dense_dims` on `KlArray`, `kl_dense_array_new`, dense branches in
   `kl_array_get` / `kl_index_get`.
2. Compiler: literal shape analysis → `DenseArrayNew` in bytecode + KIR recorder.
3. VM opcode handler (or shared RT from opcode).
4. LLVM: lower `DenseArrayNew`; optional `kl_dense2d_get` fusion for typed
   double index.
5. Rank-3+ and non-literal dense constructors.

## Related

- [0016](0016-typed-kir.md) — container metadata for chained `IndexGet`
- [0002](0002-design-principles.md) — value semantics, row copy
- [0015](0015-llvm-backend-roadmap.md) — native RT parity
