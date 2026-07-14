# 0029 — Value Representation and Memory Layout

- **Status**: accepted
- **Proposed**: 2026-07-11

**Replaces**: [0023 — Data Types, Literals, and ABI](%5Bdeprecated%5D%200023-data-types-and-abi.md)
in full for its type-tier and layout decisions (D1, D9, D11). The literal
syntax and width-table content of 0023 (D2–D8, D10, D12) is **carried forward
unchanged** and restated here so this document is self-contained.

Depends on: none of this document's decisions require
[0028 — Ownership, Borrowing, and Value Transfer](0028-ownership-and-value-transfer.md)
to land first, but 0028's category-based ownership rules are defined in terms
of the type-tier classification this document establishes (D1).

## Context

Verified against the current bootstrap runtime (`kinglet_rt_value.h`,
`kinglet_rt_internal.h`, `kinglet_rt_agg.cc`) and against a competing draft
(`kinglet-ownership-and-memory-model.md`) proposing inline-by-default value
representation, two concrete problems motivate revisiting 0023's value
representation decisions (0023 D1, D9, D11):

**1. `struct` is heap-allocated today, contradicting both 0023's own stated
direction and the intuitive reading of a struct declaration.** Empirically
verified this session: evaluating a struct literal such as
`Point { 1i32, 2i32 }` always calls `kl_struct_new`, which does
`new KlStruct()` — a real heap allocation, every time, with no pooling. Each
field occupies a full 8-byte `kl_h` slot in `KlStruct::fields`
(`std::vector<kl_h>`) regardless of its declared width — a `Point` with two
`int32` fields occupies two 64-bit slots (16 bytes of field data) plus a
`KlHeader` and the `std::vector`'s own heap overhead, not the 8 bytes a
C-style `struct { int32_t x, y; }` would occupy. Reading a narrow field
(`unbox_wire_scalar`, `llvm_function_lowerer.cc:507-530`) truncates and
sign-extends purely for arithmetic/wraparound semantics — the underlying
storage never shrinks below 64 bits per field.

**2. `float` / `float64` is unexpectedly heap-boxed, while every other scalar
is not.** `Int`, `Bool`, and `Char` are genuinely inlined directly into the
64-bit `kl_h` wire value with no heap indirection. `Float` instead goes
through a heap object via `kl_float_new`, tagged the same way as `Array` /
`Map` / `Struct` / `String` (`0xFFFE<<48` heap mark). There is no semantic
reason for this: a `float64` is a fixed-width bit pattern exactly like an
`int64`, and nothing about it requires ownership tracking or heap backing.
This is very likely historical (folded into the same tagged-handle machinery
as the true heap types when the wire format was designed), not an intentional
design choice — 0023 D6.4 documents the boxed behaviour but does not justify
it.

Fixing (1) fully (inline struct/fixed-array layout) is a substantial codegen,
ABI, and runtime rewrite. This ADR **commits to that direction** rather than
deferring it, per explicit user direction that memory layout work should not
wait behind [0028](0028-ownership-and-value-transfer.md)'s ownership-rule
rollout — the two are independent axes: 0028's transfer/borrow rules operate
on **handles** (whether a handle happens to live on the stack or point at the
heap does not change how transfer/borrow checking works), so this document's
layout changes can proceed on their own schedule without blocking or being
blocked by 0028.

## Decision

### D1 — Type tiers

| Tier | Types | Memory | Ownership role (see [0028](0028-ownership-and-value-transfer.md)) |
|------|-------|--------|----------------------------------------------------------------------|
| **Scalars (Copy)** | `bool`, `char`, `int8`…`int64`, `uint8`…`uint64`, `float32`, `float64` | **Stack**, inline bit pattern, never boxed (D2, fixes the `Float` exception) | Always copies; ownership rules do not apply |
| **Aggregates** | `struct`, fixed-size arrays (`T[N]`) | **Stack**, inline contiguous layout (D3–D4) unless every field/element is itself heap-backed | Follows field/element categories; struct-level transfer/copy is fieldwise (0028 D5/D6 applied per field) |
| **Value types (heap handles)** | `string`, `T[]` (dynamic), `map<K,V>` | **Heap**, opaque owning handle (unchanged from 0023 D7/D8) | Copy-by-default assignment, transfer at call boundaries (0028 D5) |
| **Resource types** | `fs::file`, `net::tcpstream`, `mutex`, `thread`, non-cloneable user types, any `T?` on a type requiring owning indirect representation | **Heap**, exclusive owning handle (unchanged from 0023) | Always transfers, no clone (0028 D6) |
| **Borrows** | `const T&`, `T&` | Machine pointer to the referent's storage (stack slot, struct field slot, or heap object as appropriate) | Non-owning (0028 D10) |

`void` is not a value type. `auto` defers to the initializer's type (D12,
unchanged from 0023).

The critical change from 0023 D1 is that **struct and fixed-array are no
longer heap-backed** — 0023 implicitly left struct layout ambiguous (D9 called
it a "P0 blocker" without settling stack vs. heap), and actual bootstrap
behaviour defaulted to heap. This ADR settles it: **inline by default**,
matching the competing draft's §3.1/§3.3 direction, adopted because it is the
behaviour a struct declaration's syntax already implies and because nothing
about a plain aggregate of scalars requires heap indirection or ownership
tracking.

### D2 — Scalars: uniformly inline, `Float` boxing is a defect fix

All scalar types are stack values with a directly inline bit pattern —
**no exceptions**. This fixes the `Float` anomaly identified in Context: a
`float32` / `float64` local is a plain stack slot holding its IEEE-754 bit
pattern, exactly like an `int32` / `int64` local, with no `kl_float_new` call
and no `0xFFFE<<48` heap tag.

```
Before (current bootstrap):        After (this ADR):
Int/Bool/Char  -> inline, stack    Int/Bool/Char/Float -> inline, stack
Float          -> heap (kl_float_new + 0xFFFE tag)  [REMOVED]
```

This is purely a runtime/codegen fix, not a language-surface change — no
Kinglet source using `float` needs to change; only the wire representation and
`kl_float_new`'s heap allocation are removed.

### D3 — Struct layout: inline, declaration-order fields, no reordering

A struct value occupies **contiguous inline storage** sized to the sum of its
fields' sizes (plus any explicit padding), matching 0023 D9's original intent
but now actually enforced as stack/inline rather than left to default to
heap:

```kl
struct point {
    int32 x;
    int32 y;
}
point p = point { 1i32, 2i32 };
```

```
p (stack, 8 bytes total):
┌───────┬───────┐
│ x: 4B │ y: 4B │
└───────┴───────┘
```

Rules (carried forward from 0023 D9, restated in full since this document
supersedes it):

1. **Field order** in memory equals **declaration order** in the `struct`
   definition, not struct-literal source order.
2. **Struct literals** use declaration order or named fields; named fields
   are recommended and required for cross-module literals.
3. **Alignment and size** follow a documented C-like ABI: scalars use natural
   alignment; **no field reordering for padding** — padding is explicit and
   recorded in a published `StructLayout` metadata table used consistently by
   `kir_to_llvm.cc` and `ll_emit.kl`.
4. **Pass by value** copies all fields per the layout; passing `const T&` /
   `T&` passes the address of the caller's slot (unchanged from 0028's borrow
   lowering).
5. **Cross-module field access** goes through stable offsets published in the
   layout table; misaligned reads across modules are compiler defects against
   this ADR, not user errors — same standing 0023 gave this rule.

**Fields that are themselves value types or resource types** (a `string`
field, a `T?` field using owning-indirect representation) still occupy their
**handle's** size inline in the struct (8 bytes on 64-bit, a plain
pointer-width slot) — the struct itself does not need to be heap-allocated
merely because one of its fields points at the heap. Only a struct's *own*
storage location changes (stack, not heap); each field's own category (D1)
still governs where that field's *data* ultimately lives.

### D4 — Fixed-size arrays: inline, matching struct treatment

```kl
int32[4] values;
```

```
values (stack, 16 bytes):
┌────┬────┬────┬────┐
│i32 │i32 │i32 │i32 │
└────┴────┴────┴────┘
```

Fixed-size arrays (`T[N]`, compile-time-known length) follow the same
inline-storage rule as struct fields: contiguous stack storage, no heap
allocation, no handle indirection. This is distinct from the dynamic `T[]`
type (D8, unchanged heap-handle vector) — the two are different types with
different storage strategies, not the same type with two representations.

### D5 — `box<T>`: narrowed role, explicit heap indirection **(superseded by D14)**

> **D14 (Optional types) replaces the need for `box<T>` as a separate
> heap-indirection primitive.**  Recursive types use `T?` with an owning
> indirect representation; users write `node? next`, not `box<node> next`.
> This section is retained for historical context; D14 is the active design.

`box<T>` is the **only** way to put a value on the heap that would otherwise
be inline (D1–D4). Its role narrows from 0022's original framing (where it
sat alongside a language-wide unique-ownership model for *all* heap types) to
specifically: **user-requested heap indirection for recursive or
otherwise-inline-incompatible structures** (linked lists, trees, any type
that would otherwise have infinite size if inlined):

```kl
struct node {
    int32 value;
    box<node> next;   // heap indirection: node cannot inline-contain itself
}
```

```
box<T> (stack, one pointer):
┌─────────┐
│ pointer │────────────► T on heap
└─────────┘
```

`box<T>`:

- exclusively owns one `T` (resource-type category, 0028 D6);
- is not implicitly copyable — no clone exists;
- destroys `T` and frees storage at scope exit (0028 D8).

`string` / `T[]` / `map<K,V>` remain heap-backed **on their own terms** (D1,
unchanged from 0023) — they do not route through `box<T>`; a `string` field in
a struct is still just an 8-byte handle inline in the struct, independent of
whether `box<T>` exists at all. `box<T>` only matters for types that would
otherwise need to be inline (D3/D4 aggregates) but cannot be, due to
self-reference or explicit user choice to indirect.

### D6 — Fixed-width numerics (unchanged from 0023 D2–D3)

Carried forward verbatim; restated so this document is self-contained:

| Kind | Types | Literal suffix (examples) |
|------|-------|---------------------------|
| Signed int | `int8` … `int64` | `42i32`, `0xFFi8` |
| Unsigned int | `uint8` … `uint64` | `255u8`, `0xFFFD0000u32` |
| Float | `float32`, `float64` | `1.0f32`, `3.14f64` |

Aliases: `int` → `int64`, `float` / `double` → `float64`, `byte` → `uint8`.

Rules: no silent narrowing (wider→narrower requires explicit cast); no mixed-
width arithmetic without explicit promotion; literals pick the narrowest type
that fits unless a suffix forces width, but wire/native lowering always uses
the slot's declared type.

**Wide integers and hex literals** (0023 D3, unchanged): `int64`/`uint64` are
first-class; hex/binary/decimal literals accept type suffixes
(`0xFFFD000000000000u64`); `uint64` literals parse unsigned, never narrowed;
64-bit bitwise ops are defined modulo 2⁶⁴ on `uint64` and two's-complement on
`int64`; underscore separators apply to all bases.

### D7 — `char`, `byte`, `bool`, `null` (unchanged from 0023 D4–D5)

`byte` is 8-bit unsigned (binary buffers, hex emission); `char` is a Unicode
code unit (`string[i]` returns `int` code in v1, a dedicated `char` return
type remains optional future work). `bool` is `true`/`false`, Copy, lowers to
`i1` internally. `null` remains an error-value/sentinel path
([0006](0006-error-handling-unification.md)); full `T?` stays out of scope
here.

### D8 — `string`, dynamic arrays, and maps (unchanged from 0023 D7–D8)

- **`string`**: heap-owned handle (D1); operations (`len`, `[]`, `+`, `slice`,
  `split`, comparisons); assignment/by-value clones per
  [0028](0028-ownership-and-value-transfer.md) D5; `const string&` avoids the
  clone for read-only APIs.
- **`T[]`** (dynamic): heap-owned vector handle; `len`, `[]`, `push`, `pop`.
- **`map<K,V>`**: heap-owned associative container handle.
- **Dense `T[][]…[]`** layout per [0017](0017-dense-nested-array-layout.md)
  where applicable — unaffected by this ADR, since dense nested arrays are
  already a heap-handle type, not the fixed-size inline array of D4.

### D9 — Enum representation (unchanged from 0023 D10)

Enums have three representational contexts, unchanged: **ordinal** (`0…n-1`)
in the type checker and in struct fields; **inline wire tag**
(`0xFFFD<<48 | type<<16 | variant`) for tag-only enums at runtime; plain
integer ordinal in compiler-internal tables. Source-level `== EnumType::X`
compares ordinals after lowering; `enum_wire()` / `enum_ord()` stdlib helpers
convert between forms when runtime wire is explicitly needed.

### D10 — Native wire format (summary, updated for D1–D5)

| Kind | Wire |
|------|------|
| Scalars (`int*`, `uint*`, `float*`, `bool`, `char`) | Inline bit pattern, stack slot — **no heap boxing for any scalar, including `float`** |
| Struct (fixed layout) | Inline contiguous stack storage per D3 — **no heap allocation for the struct itself** |
| Fixed-size array `T[N]` | Inline contiguous stack storage per D4 |
| `string`, dynamic `T[]`, `map<K,V>` | `0xFFFE<<48 \| ptr` heap handle (unchanged) |
| `T?` (owning indirect, recursive) | `0xFFFE<<48 \| ptr` heap handle pointing at a heap-allocated `T` (D14) |
| `T?` (tagged inline, scalar) | Inline value plus discriminator bit/byte, layout per type |
| Tag-only enum | `0xFFFD<<48 \| type<<16 \| variant` inline (unchanged) |

The only wire-format change from 0023 D11 is removing `Float`'s heap-boxed row
— every other row is unchanged. Struct's row changes from "heap handle" to
"no wire handle; inline stack storage," which is the substantial codegen/ABI
change this ADR commits to.

### D11 — `auto` (unchanged from 0023 D12)

`auto x = expr;` infers `x` from `expr`; `auto` without an initializer is
ill-formed; inference never widens to a borrow — `auto` from `&expr` becomes
`const T&` / `T&` once references are checked, per the initializer's own
borrow kind.

### D12 — Relationship to [0028](0028-ownership-and-value-transfer.md)

| Topic | This ADR (0029) | 0028 |
|-------|-------------------|------|
| Scalar / value-type / resource-type classification | **D1 (this document)** | Consumed by 0028 D1–D6 |
| Struct / fixed-array inline layout | **D3–D4 (this document)** | Unaffected — 0028's transfer/borrow rules operate on handles regardless of stack vs. heap placement |
| `T?` (Optional) role and representation | **D14 (this document)** | Consumed by 0028 D6 (resource-type transfer rules for owning-indirect `T?`) |
| Scalar copy-always rule | **D1/D2 (this document)** | Restated and relied upon by 0028 D2 |
| `&` / `const T&` / `T&` syntax and semantics | — | 0028 D3, D10 |
| Transfer, drop, `move()` | — | 0028 D4–D9 |

Neither document's core decisions are blocked on the other landing first —
0029's layout work and 0028's ownership-rule work can be implemented and
merged independently, in either order.

### D13 — Implementation order

| ID | Deliverable | Exit criterion |
|----|-------------|-----------------|
| **L0** | Remove `Float` heap boxing (D2); scalars uniformly inline | no `kl_float_new` call sites remain; float wire has no `0xFFFE` tag |
| **L1** | Struct ABI: `StructLayout` metadata table, inline stack allocation, declaration-order fields (D3) | struct literal evaluation no longer calls `kl_struct_new`; cross-module struct copy golden tests pass |
| **L2** | Fixed-size array inline layout (D4) | `T[N]` local evaluation allocates on the stack, not via `kl_array_new` |
| **L3** | Optional type `T?` representation, including owning indirect layout for recursive types (D14) | self-referential struct fields (`node? next`) compile and round-trip |

Bootstrap leads L0–L3.

### D14 — Optional types (`T?`)

`T?` is an abstract data type. The compiler selects a concrete representation
per `T`, not the user. Three representation strategies:

| T category | Representation | Notes |
|------------|----------------|-------|
| Scalar (`int?`, `float?`) | Inline tagged value | Reserved sentinel bit pattern (e.g. all-ones for `int?`) |
| Already-nullable heap handle (`string?`) | Inline, co-opted empty state | `""` or `0xFFFE<<48 | 0` serves as `none` without a separate tag |
| Struct/enum requiring recursion (`node?`) | Owning indirect (heap pointer) | `none` = null pointer; `some(v)` = pointer to heap-allocated `v` |

`string` and other heap-handle types that already carry an in-band null/empty
representation do not get a separate tag byte — the compiler reuses the
existing empty state.  Resource types with an invalid-handle sentinel
(`file?`) similarly co-opt the sentinel rather than adding a wrapping tag.

**Recursive type termination rule:**

A struct may not contain itself through an unconditional inline field.
A recursive cycle is permitted only when at least one edge is represented by
an indirection-capable type such as `T?`. For recursive `T?`, the compiler
uses an owning indirect representation — the same role `box<T>` would fill in
other designs, but surfaced as a general Optional construct rather than a
single-purpose heap-indirection primitive.

```kl
// Illegal — unconditional inline recursion, sizeof is unbounded
struct node {
    int value;
    node next;       // ERROR: node contains itself
}

// Legal — T? breaks the recursion
struct node {
    int value;
    node? next;      // OK: compiler uses owning indirect representation
}
```

**Syntax and usage:**

```kl
// Construction
node head = node {
    .value = 1,
    .next = node { .value = 2 },   // nested value; end-of-chain sentinel TBD
};

// Safe access via field? suffix — parsed as part of FieldAccessExpr
int? maybe_val = s.maybe?;          // returns Optional<int>, same as s.maybe
int safe = s.maybe? ?: -1;          // coalesce: -1 when maybe is null

// Transfer
node? tail = head.next;
```

`field?` (suffix `?` on a field name) is the safe access operation, distinct
from the `?:` null-coalesce operator.  When the parser sees `?` after a field
name and the next token is not `:`, it consumes the `?` as `optional_access`
on the `FieldAccessExpr`.  The type checker returns the field's `Optional<T>`
type after verifying the field is indeed Optional.  At the KIR layer, lowering
uses `FieldGet` followed by `JmpIfErr` to check the sentinel — if the
`Optional<T>` value is null-type, the result is null; the `?:` coalesce
operator then supplies the fallback value.

**Chain access** (`head.next?.value`) — accessing a struct field through an
Optional struct field — works as of [#114](https://github.com/kinglet-lang/bootstrap/pull/114).
`field?` unwraps to the element type (not `Optional<T>`), so `.value` on the
result of `next?` type-checks as an ordinary field access on `node`. Multi-level
chains (`head.next?.next?.value`) work the same way. At runtime, a null
anywhere in the chain short-circuits to the null sentinel via `JmpIfErr`
rather than dereferencing a null pointer.

`none` as a literal value for the empty Optional is **not yet implemented**.
The `null` literal serves as the empty Optional value in the current
implementation; `node? next = null;` is the idiomatic way to construct an
empty Optional.  End-of-chain null values on recursive struct fields fall
through to the runtime sentinel — safe when guarded by `?:` coalesce or
explicit null checks.

**Rationale for Optional over `box<T>`:** A single-purpose heap-indirection
primitive (`box<T>`) duplicates the representation machinery already needed
for `T?` on recursive types — both are owning indirect pointers to a `T` on
the heap.  Surfacing the same mechanism through `T?` gives users Optional
semantics (a real feature useful beyond recursion) while the compiler reuses
the same indirection layout for the recursive-termination case.  Users never
type `box<T>` — they write `T?` and the compiler picks the right layout.

## Consequences

### Positive

- Fixes an actual defect (`Float` heap boxing) that had no design
  justification and was inconsistent with every other scalar type.
- Struct and fixed-array types finally match their declaration-implied
  memory model — a `struct { int32 x; int32 y; }` is 8 bytes, not a heap
  allocation plus two 64-bit slots plus vector overhead.
- `T?` Optional type as a unified mechanism — handles scalar tagging, string
  co-opted empty state, and recursive indirection in one construct rather
  than a separate `box<T>` primitive.
- Struct-by-value copies become genuinely cheap for small aggregates
  (register/stack copy) instead of always paying a heap-allocation cost.

### Risks and constraints

- **L1 (struct ABI)** is the largest single piece of work here: it touches
  codegen (`kir_to_llvm.cc`), the self-host emitter (`ll_emit.kl`), and the
  runtime struct-field accessor functions (`kl_struct_field_at` /
  `kl_struct_field_set`), which currently assume a heap object exists at all.
  This is a breaking runtime-representation change, not additive.
- **Large structs passed by value** still copy all fields (D3 rule 4) — this
  ADR does not introduce a size threshold above which structs fall back to
  heap allocation; if profiling later shows this matters, that is a separate,
  narrower follow-up, not a default behaviour change here.
- **Existing `KlStruct` / `kl_struct_new` / `kl_struct_field_at` /
  `kl_struct_field_set` runtime API is retired** for the fixed-layout case;
  any code (including compiler-internal Kinglet source in self-host) relying
  on struct-as-heap-handle behaviour needs updating alongside L1.

### Acceptance criteria

- [x] D2: no runtime call site allocates a heap object for a `float32` /
      `float64` value; float wire carries no `0xFFFE` tag — verified: no
      `new KlFloat` call sites remain, `kl_float_new` returns a tagged
      inline `KL_INLINE_FLOAT_MARK | float32-bits` value
- [ ] D3: a struct literal's evaluation does not call any heap-allocation
      runtime function; `sizeof`-equivalent struct size matches the
      declared-field-sum-plus-padding layout table, not a handle size
- [ ] D4: `T[N]` locals are stack-allocated, not heap-allocated
- [x] D14: self-referential struct fields with `T?` compile and round-trip
      with owning indirect representation (`is_indirect` on `FieldInfo`);
      `field?` safe access works with chain access and `?:` coalesce
- [ ] D14: `none` literal is recognised

### Implementation status

| Deliverable | Status | PR |
|-------------|--------|-----|
| L0: Float unboxing (D2) | ✅ implemented | [#108](https://github.com/kinglet-lang/bootstrap/pull/108) |
| L1: Struct inline layout (D3) | ❌ not implemented | — |
| L2: Fixed-size array inline (D4) | ❌ not implemented | — |
| L3: Optional types (D14) | ✅ core implemented | [#109](https://github.com/kinglet-lang/bootstrap/pull/109), [#110](https://github.com/kinglet-lang/bootstrap/pull/110) |
| L3: Null sentinel fix | ✅ implemented | [#111](https://github.com/kinglet-lang/bootstrap/pull/111) |
| L3: `TypeKind::Optional`, `T?` type syntax | ✅ implemented | [#112](https://github.com/kinglet-lang/bootstrap/pull/112) |
| L3: `field?` safe access, recursive `node?` | ✅ implemented | [#113](https://github.com/kinglet-lang/bootstrap/pull/113) |
| L3: Chain access fix (`field?` unwrap) | ✅ implemented | [#114](https://github.com/kinglet-lang/bootstrap/pull/114) |
| L3: `none` literal | ❌ not implemented | future work |

## Dependencies

- [0002](0002-design-principles.md) — zero-cost abstraction pillar this
  ADR's inline-layout work directly serves.
- [0015](0015-llvm-backend-roadmap.md), [0016](0016-typed-kir.md) — typed KIR
  slots and native lowering that L1–L3 build on.
- [0017](0017-dense-nested-array-layout.md) — unaffected dense nested-array
  layout, noted for contrast in D8.
- [0028](0028-ownership-and-value-transfer.md) — consumes this document's
  type-tier classification (D1); no blocking dependency in either direction
  (D12).

## References

- Runtime: `bootstrap/runtime/kinglet_rt_value.h` (wire format, heap
  tagging), `kinglet_rt_internal.h` (`KlHeader`, `KlStruct` at line 43),
  `kinglet_rt_agg.cc` (`kl_struct_new` at line 139, `kl_array_new`)
- Codegen: `bootstrap/compiler/backend/codegen/llvm/llvm_function_lowerer.cc`
  (`unbox_wire_scalar` at line 507, struct/array RT function bindings at
  lines 120-136 and 210-230)
- Compiler: `bootstrap/compiler/backend/compiler/compiler.cc`
  (`compile_struct_literal` at line 1974, `StructNew` emission at lines 2026
  and 2379)
- Self-host lowering: `kinglet/compiler/ll_emit.kl`

## Open questions

1. **Large-struct-by-value threshold** — whether a future revision should
   introduce an implicit indirection for very large structs passed by
   value, or leave that entirely to explicit `T?` opt-in; not decided
   here (Risks section).
2. **`StructLayout` metadata format** — exact on-disk/in-memory shape of the
   layout table shared between `kir_to_llvm.cc` and `ll_emit.kl`; carried
   over unresolved from 0023 D9's original framing.
3. **Padding control** — whether user code ever needs explicit control over
   struct padding (e.g. `#[packed]`-equivalent), or whether the natural-
   alignment default (D3 rule 3) is sufficient indefinitely; not raised as a
   need yet, tracked here in case it becomes one.
