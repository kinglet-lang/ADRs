# 0023 — Data Types, Literals, and ABI

- **Status**: accepted (P0 slices partial)
- **Proposed**: 2026-06-21

**Current companion**: [0022 — Native-Only Toolchain and Unique Ownership](0022-native-unique-ownership.md) defines ownership and borrows; this ADR defines scalar/data/ABI details for that native-only model.

## Context

Self-host chain debugging (gen1 **sh0** → gen2 → gen3) exposed that Kinglet’s
**type system is specified piecemeal** across [0015](0015-llvm-backend-roadmap.md) D7,
[0016](0016-typed-kir.md), and runtime docs, while **struct layout and enum wire**
behaviour is implicit. Bootstrap `kir_to_llvm.cc` and self-host `ll_emit.kl` diverge
on:

- cross-module **struct by-value** copy and field reads;
- **enum** compile-time ordinal vs runtime inline wire (`0xFFFD<<48`);
- **64-bit constants** (`0xFFFD000000000000`) and `(hi << 32) | lo` in Kinglet source;
- **`int::bits(float)`** for LLVM hex emission.

[0022](0022-native-unique-ownership.md) covers **ownership and borrows** on native;
this ADR is the companion for **what the types are**, **how literals parse**, and
**how values are laid out in memory and on the wire**. Together they form the **P0
self-host gate** agreed for language work before further stdlib or prove expansion.

**Non-goals here:** generic constraints ([0009](0009-concepts-landing.md)),
optional niches (`int?`), `comptime`, and full [0015](0015-llvm-backend-roadmap.md) D7
width opcode matrix — scheduled in phases below, not all in v1 of this ADR.

## Decision

### D1 — Type tiers

| Tier | Types | Role |
|------|-------|------|
| **Scalars (Copy)** | `bool`, `byte`, `char`, `int`, fixed-width integers (D2), `float`/`double` + fixed floats (D2) | Stack slots, bitwise copy, no ownership |
| **Handles (unique heap)** | `string`, `T[]`, `map<…>` | Opaque owning wire per [0022](0022-native-unique-ownership.md) D2 |
| **Aggregates** | `struct`, `enum` | Layout rules in D9–D10; may contain scalars and handles |
| **Borrows** | `&T`, `&mut T` | Non-owning aliases per [0022](0022-native-unique-ownership.md) D3 |
| **Transfer** | `unique T` | Consumption at API boundary per 0022 D4 |

`void` is not a value type. `auto` defers to initializer type (D12).

### D2 — Fixed-width numerics (complete surface)

Adopt [0015](0015-llvm-backend-roadmap.md) D7 as the **authoritative width table**:

| Kind | Types | Literal suffix (examples) |
|------|-------|---------------------------|
| Signed int | `int8` … `int64` | `42i32`, `0xFFi8` |
| Unsigned int | `uint8` … `uint64` | `255u8`, `0xFFFD0000u32` |
| Float | `float32`, `float64` | `1.0f32`, `3.14f64` |

**Aliases (fixed meaning on all targets):**

| Alias | Resolves to |
|-------|-------------|
| `int` | `int64` |
| `float` | `float64` |
| `double` | `float64` |
| `byte` | `uint8` |

Rules (from 0015 D7, normative here):

1. **No silent narrowing** — wider → narrower requires explicit cast.
2. **No mixed-width arithmetic** in one expression without explicit promotion.
3. Literals pick the **narrowest type that fits** unless a suffix forces width;
   unsuffixed integers that fit in `int32` may default to `int32` in checker v1,
   but **wire and native lowering** use the **declared** type of the slot.

Implementation phases: `int32` slice is **done**; remaining widths are **P0-4 /
P1** per D14.

### D3 — Wide integers and hex literals

**Problem:** Self-host `ll_emit` must construct enum marks and i64 wires in Kinglet
source; plain `int` and `0xFFFD0000` are unreliable (signed truncation, no full
64-bit hex).

**Decision:**

1. Add **`int64` / `uint64`** as first-class types (aliases `int` / preferred unsigned
   for bit masks).
2. **Hex / binary / decimal** literals accept **type suffixes**:

   ```kl
   0xFFFD000000000000u64
   0xFFFD0000u32
   0b1010i8
   ```

3. **`uint64` literals are parsed unsigned** — never narrowed to signed `int32`.
4. **64-bit bitwise ops** on `int64`/`uint64`: `|`, `&`, `^`, `<<`, `>>` with
   **defined modulo 2⁶⁴** semantics on `uint64` and **two’s-complement** on `int64`.
5. **Compose i64 wire** from halves (self-host and runtime):

   ```kl
   uint64 wire = 0xFFFD000000000000u64 | uint64(packed & 0xFFFFFFFFu32);
   ```

   Lowering must not depend on undefined overflow in plain `int`.

6. **Underscore separators** in numeric literals apply to all bases (existing lexer
   behaviour).

### D4 — `char` and `byte`

| Type | Size | Literal | Use |
|------|------|---------|-----|
| `byte` | 8-bit unsigned | `0xFF`, `'\\n'` (if char lit maps to byte) | Binary buffers, LLVM `\\HH` emission |
| `char` | Unicode code unit / language char | `'a'`, `'\\n'` | Lexer, `string[i]` as `int` code (existing) |

`int(s[i])` remains valid; **`string[i]` returns `int` code** in v1. A dedicated
`char` return type is optional P1.

### D5 — `bool` and `null`

- `bool`: `true` / `false`; **Copy**; native lowering as `i1` internally, wire as
  tagged or raw per [0016](0016-typed-kir.md).
- `null`: only where nullable types exist; v1 remains **error-value / sentinel**
  paths documented in [0006](0006-error-handling-unification.md). Full `T?` is out of
  scope for this ADR.

### D6 — Floating point and `int::bits`

1. **`float` / `double`** map to IEEE-754 binary32/64 ([0015](0015-llvm-backend-roadmap.md)).
2. **`int::bits(x)`** where `x` is `float` or `double` returns **`uint64`** bit
   pattern of `x` (quiet NaNs preserved). **Required for self-host LLVM emission**
   (`double 0x…` literals).
3. **`float::from_bits(uint64)`** (or `double::from_bits`) is the inverse; **native
   and semantic reference must match** (no VM-only paths after [0022](0022-native-unique-ownership.md) D1).
4. Runtime float values remain **boxed heap handles** on the wire (`0xFFFE<<48` tag)
   until a future unboxed float stack path lands; **bits intrinsics operate on the
   IEEE value**, not the wire handle.

### D7 — `string`

- **Heap-owned**, unique per 0022.
- Operations: `len`, `[]`, `+`, methods (`slice`, `split`, …), comparisons.
- **Copy** (`=`, by-value param) **clones** content (0022 D2).
- **Borrow** `&string` avoids clone for read-only APIs.
- Native lowering: `libkinglet_rt` C ABI; self-host must support **string locals** and
  **cross-module string fields** under D9.

### D8 — Arrays and maps

- **`T[]`**: heap-owned vector; handle wire; `len`, `[]`, `push`, `pop`, methods.
- **`map<K,V>`** (surface syntax may vary): heap-owned associative container.
- **Dense `T[][]`** layout per [0017](0017-dense-nested-array-layout.md) where applicable.
- **Element types** follow D2; container **assignment clones** (0022).
- **Typed `IndexGet`** per [0016](0016-typed-kir.md) for scalar unboxing at one or two
  levels.

### D9 — Struct ABI (P0 blocker)

Structs must have **predictable, cross-module-stable layout**.

**Rules:**

1. **Field order** in memory = **declaration order** in the `struct` definition, not
   struct literal source order.
2. **Struct literal** must use **declaration order** or **named fields** (named fields
   recommended; required for cross-module literals in P0 implementation):

   ```kl
   Parser p = Parser { current: 0, pending_greater: false, tokens: tok, errors: [] };
   ```

3. **Alignment and size** follow a documented **C-like ABI** (bootstrap publishes
   `StructLayout` metadata used by both `kir_to_llvm.cc` and `ll_emit.kl`):
   - scalars: natural alignment;
   - `string`, `T[]`, maps: **pointer-sized handle** (8 bytes on 64-bit);
   - **no reordering** of fields for padding in v1 (padding explicit in layout table).
4. **Pass by value** copies **all fields** according to layout; **pass `&` / `&mut`**
   passes address of the caller’s slot.
5. **Struct fields that are enums** store **compile-time ordinal** in the struct slot
   unless the field type is explicitly documented as **wire enum** (see D10). Mixing
   ordinals and wire in the same struct is **ill-formed**.
6. **Cross-module access** to fields goes through **stable offsets**; misaligned reads
   (gen2 `double_value` / `Parser.current` bugs) are **compiler defects** against this
   ADR, not user errors.

**Workaround policy:** same-module accessor functions are **transitional** until D9 is
implemented in bootstrap and self-host; new code should not add accessors unless ABI
fix is blocked.

### D10 — Enum representation

Kinglet uses enums in **three contexts**; this ADR **separates** them:

| Context | Representation | Comparison |
|---------|----------------|------------|
| **Type-checker / struct fields** | **Ordinal** `0…n-1` | `==` on enum type compares ordinals |
| **Runtime value on wire** | **Inline tag** `0xFFFD<<48 \| type<<16 \| variant` for tag-only enums | `kl_value_eq` |
| **Compiler-internal tables** (e.g. opcode) | **Plain `u8`/`u32` ordinal** in structs; optional `32768+` encoding in **chunk wire only** | Integer compare or `vm_opcode_ord()` |

**Language rules:**

1. Source **`== TokenType::Eof`** compares **ordinals** after lowering; if runtime wire
   is needed, use **`enum_wire(e)`** / **`enum_ord(e)`** stdlib helpers (names TBD).
2. **Match** on enum uses **ordinal** semantics at compile time; runtime match lowers
   to wire compare where values are on the stack.
3. **`KirInstr.op`**, **`Instruction.op`**, and similar **compiler structs store
   `int` ordinals**, not enum-typed fields, until ll_emit proves enum field layout
   (recommended permanently for portability).
4. Documentation and diagnostics must state which form a API uses.

### D11 — Native wire format (summary)

Unchanged from [0015](0015-llvm-backend-roadmap.md) and [native.md](../docs/native.md),
amended for 0022:

| Kind | Wire |
|------|------|
| `int64` / pointers | plain `i64` |
| Heap owned | `0xFFFE<<48 \| ptr` (handle) |
| Tag-only enum | `0xFFFD<<48 \| …` |
| Float boxed | heap object via `kl_float_new` |

Unique ownership: two **owned** `string` variables do **not** share a refcount bump
on copy (0022).

### D12 — `auto`

- `auto x = expr;` infers `x` from `expr` (existing).
- `auto` without initializer is ill-formed.
- Inference never widens to borrow; `auto` from `&expr` is `&T` when references land.

### D13 — Relationship to [0022](0022-native-unique-ownership.md)

| Topic | ADR |
|-------|-----|
| Copy vs unique heap, `drop`, `unique T` | 0022 |
| Scalar widths, literals, struct layout, enum forms | **0023 (this)** |
| `&` / `&mut` syntax | 0022 |
| `&` / `&mut` require correct struct addresses | **0023 D9** |

Implement **D9 + D10 + D3** before **0022 N4–N6** in the self-host critical path.

### D14 — P0 implementation order (self-host gates)

| ID | Deliverable | Exit criterion |
|----|-------------|----------------|
| **P0-0** | **Delete VM execution backend** ([0022](0022-native-unique-ownership.md) D1) | no `--run` / `--backend vm` / `kinglet-vm`; tests use native compiler binary |
| **T0** | This ADR accepted | — |
| **T1** | Struct ABI in bootstrap `kir_to_llvm` + tests | cross-module struct copy golden |
| **T1** | Same ABI in `ll_emit.kl` | struct field read test native |
| **T2** | Enum rules D10 + `enum_ord`/`enum_wire` helpers | no raw `== Enum::X` in compiler paths |
| **T3** | `uint64`/`int64` suffix literals + D3 ops | `0xFFFD000000000000u64` in source |
| **T3** | `int::bits` / `from_bits` native parity | float hex manifest green |
| **T4** | String / struct / array **locals** in ll_emit | `struct_local`, `arr_local` in compiler modules |
| **T5** | `&T` / `&mut T` (0022 N2–N3) | parser/checker `&mut` session |
| **T6** | **self2.ll == self3.ll** | gen2 emits identical LLVM for `core/main.kl` |
| **T7** | gen2 **--check core/main.kl** | no sema errors on full compiler |
| **T8** | `run_smoke_shadow.sh` uses **sh0** path | 95/95 shadow manifest (incl. `dir_import`) |

Bootstrap leads T1–T4; self-host tracks per slice.

### D15 — Amendments to prior ADRs

| ADR | Amendment |
|-----|-----------|
| [0015](0015-llvm-backend-roadmap.md) D7 | Full width table **normative here**; implementation schedule in D14 |
| [0016](0016-typed-kir.md) | Phase 2b open items **tracked under D14 T4** |
| [0002](0002-design-principles.md) | Scalars Copy; heap + struct layout per 0023 + 0022 |
| [0022](0022-native-unique-ownership.md) | Ownership on types defined in this ADR |

## Consequences

### Positive

- Single document for **types, literals, ABI** — reduces self-host guesswork.
- **`u64` hex** removes magic decimals and fragile `sub`/`or` in `ll_emit`.
- **Struct ABI** lets compiler sources drop accessor boilerplate after T1.
- Clear **enum** story ends ordinal/wire schizophrenia.

### Risks

- **Named struct literals** may require parser work if not present.
- **Full D7 width matrix** is large; phased delivery must not block T6 on all widths.
- **Wire format change** for unique strings is breaking at RT level (0022).

### Acceptance criteria (P0 complete)

- [ ] D9 ABI tests green in bootstrap and self-host
- [ ] D3 `u64` hex constants used in `ll_emit` enum wire emission
- [ ] D10 ord helpers used in `parser` / `compiler` / `ll_emit` (no stray wire compare)
- [ ] T6: `diff self2.ll self3.ll` empty
- [ ] T7: gen2 checks full compiler tree
- [ ] T8: shadow manifest via sh0 native

## Dependencies

- [0002](0002-design-principles.md), [0005](0005-backend-architecture.md)
- [0015](0015-llvm-backend-roadmap.md), [0016](0016-typed-kir.md), [0017](0017-dense-nested-array-layout.md)
- [0022](0022-native-unique-ownership.md)

## References

- Self-host chain: `core/main.kl`, `compiler/ll_emit.kl`, `ir/ir_emit.kl`
- Bootstrap lowering: `bootstrap/src/codegen/llvm/kir_to_llvm.cc`
- Runtime wire: `bootstrap/runtime/kinglet_rt_value.h`

## Open questions

1. **Named struct literals** — new syntax vs factory-only cross-module construction.
2. **`char` vs `int` for `string[i]`** — defer to P1 or lock in v1?
3. **Padding** — explicit `{i32, i32}` vs packed structs for compiler tables.
4. **Promotion rules** — full 0015 D7 table vs minimal `int32`/`int64` for T6.

## Amendments

### 2026-06-21 — P0 slices landed for self-host native debugging

This ADR is accepted as the current type/literal/ABI direction, but remains partial.
The self-host repository landed early P0 slices on 2026-06-20 and 2026-06-21:

- `6a17c95` (2026-06-20): `FuncEntry` accessors for cross-module native ABI.
- `5b123ad` (2026-06-20): stable integer ordinals for `Type.kind`.
- `1e26a49` (2026-06-21): wide `u64` hex and enum wire helpers.

Those commits cover pieces of D3 and D10 needed by the native self-host chain.
The full ADR is not implemented: D9 cross-module struct ABI, the full fixed-width
numeric surface, complete enum representation cleanup, and the P0 acceptance gates
in D14 still require separate verification before this ADR can be marked
implemented.
