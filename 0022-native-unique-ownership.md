# 0022 — Native-Only Toolchain and Unique Ownership

- **Status**: draft
- **Proposed**: 2026-06-21

## Context

Kinglet’s near-term work is **self-host on the native backend** (KIR → LLVM →
`libkinglet_rt`), not bytecode interpretation. [0014](0014-compilation-toolchain-architecture.md)
treated kbc as transitional; this ADR **ends that transition**: the **VM backend is
deleted immediately** — not deprecated, not gated on ownership phases or self-host
parity.

The dominant language-capability gap is **how owned values behave**: today
parameters and locals are always “by value” in the AST, but the implementation
mixes **deep copy**, **RC + COW** ([0010](0010-vm-redesign.md)), and **unowned
wire handles** without a single coherent model. Native lowering still lacks
scope-exit **drop** ([0015](0015-llvm-backend-roadmap.md), [native.md](../docs/native.md)).
[0021](0021-references-and-move.md) explored references and move semantics on a
dual-track VM/native assumption; that direction is **withdrawn** in favour of the
model below.

This ADR defines:

1. **Native-only** as the target execution and toolchain backend for language design.
2. **Unique ownership** for heap-backed types (option B — no shared refcount/COW in
   the language model).
3. **`&T` / `&mut T`** for non-owning borrows and in-place mutation.
4. **`unique T`** for explicit ownership transfer at API boundaries.
5. **Scope `drop`** as the primary destruction mechanism (0002).

There is **no `move()` keyword**, **no implicit last-use transfer**, and **no
move-semantics / moved-from** rules as a cross-cutting feature.

## Decision

### D1 — Native-only toolchain (delete VM now)

1. **Language design** targets the **native backend only** (Ref C++ today; self-host
   `ll_emit` follows). New checker, KIR, and runtime rules are **not** required to
   map to VM opcodes or kbc layout.
2. **Default `kinglet build`** output is a **native executable** ([0020](0020-project-manifest-and-targets.md)
   `build.backend = native` in `kinglet.nest`). The `vm` backend option is **removed**.
3. **Delete the VM backend immediately** upon acceptance of this ADR — in the same
   change series as the ADR, not deferred to ownership phases or self-host gates:
   - Ref compiler: VM codegen, kbc emitter, opcode tables, VM runtime interpreter
   - CLI: `--save-bytecode`, `--vm`, `backend = vm`, and any kbc output paths
   - Tests: migrate or drop VM-only suites; **no** retained “legacy VM hooks”
   - Self-host: `compiler/bytecode.kl` and kbc serialization helpers **removed**
     (historical kbc format remains documented in [0008](0008-kbc-format-evolution.md)
     and [0010](0010-vm-redesign.md) only)
4. **Shadow / differential** validation uses **behaviour comparison on native
   output** (exit code + stdout). Bytecode identity tests ([0013](0013-bootstrap-bytecode-delta.md))
   are **retired** with the VM.
5. **CI and local workflows** use native build/run only (`ensure_build_stamp`,
   `compiler.kbc`, `ensure_cli_kbc`, and similar VM bootstrap paths are **deleted**).

Rationale: keeping VM code forces dual maintenance (opcodes, RC+COW wire, kbc
golden tests) that no longer serves the shipping path. Ownership work (N1…N6)
proceeds on a **single** backend.

### D2 — Ownership model: unique heap, copy scalars

**Scalars and fixed-width numerics** (`int`, `bool`, `byte`, `float*`, `int32`, …):

- Remain **ordinary values** (stack slots, bitwise copy).
- **Copy by default** on assignment and by-value parameters.
- No `unique` qualifier (ill-formed).

**Heap-backed types** (`string`, `T[]`, maps, structs/enums containing heap fields):

- Each **owned** local, parameter, field, or return slot has **at most one owner**
  at a time (unique ownership).
- The runtime holds a single owning handle per owned value; **no refcount sharing**
  between two owned variables.
- **Assignment** `a = b` between two owned values of the same heap type performs a
  **copy** (allocate + deep clone in `libkinglet_rt`), then destroys the previous
  contents of `a` at the assignment point. **`b` remains valid** — this preserves
  [0002](0002-design-principles.md) “copy by default” for `=`.
- **Destruction** at scope exit runs **`drop`** on owned values still in scope
  (D7).

**Not in the model:** COW, shared handles between owned names, or implicit transfer
on assignment/call.

### D3 — Reference types: `&T` and `&mut T`

References are **non-owning borrows** of an **owned** or **referent** lvalue.

```kl
void inspect(&string s) { io::out.line(s.len()); }
void append(&mut Buffer b, string tail) {
  b.data = b.data + tail;   // mutates owned referent in place
}
```

Rules:

1. **`&T`**: shared read-only borrow; multiple `&x` may coexist.
2. **`&mut T`**: exclusive mutable borrow; see D5.
3. **`&expr`**: address-of an lvalue only; **no** borrow of temporaries or rvalues.
4. **Auto-deref** for field access, indexing, and UFCS on references (v1).
5. References **do not extend lifetime** beyond the referent’s owning scope (D5).

Lowering (native): references are **pointers** to the storage of the referent
(alloca, struct field slot, or heap object header as appropriate).

### D4 — Explicit transfer: `unique T`

**`unique`** is a type qualifier for **consuming** parameters and return positions.
It means: **ownership of the heap resource is transferred**; the source variable
must not be used afterward.

```kl
void sink(unique string s) { ... }

void enqueue(&mut Queue q, unique string item) {
  q.push(item);   // item ownership moves into the queue; caller's item is consumed
}

int main() {
  string msg = "hi";
  sink(msg);      // OK: msg consumed
  // inspect(msg);  // ERROR: use after consume
  return 0;
}
```

Rules:

1. **`unique T`** is only valid when `T` is a type that uses heap ownership (D2);
   for scalars it is ill-formed.
2. Passing `x` to a **`unique T`** parameter **consumes** `x` (D6).
3. **`unique`** on return types transfers ownership to the caller (v1).
4. There is **no** `move()` expression; consumption is visible from the **type
   signature**.
5. **`&` / `&mut` and `unique` are mutually exclusive** on the same value at the
   same time (cannot pass `x` to `unique` while `&x` or `&mut x` is active).

### D5 — Borrowing rules (no lifetime syntax)

Checked **intra-procedurally** with call-boundary reborrow (same spirit as
[0021](0021-references-and-move.md) D4, without move semantics):

**`&mut` exclusivity:**

- While `&mut x` is active, no other `&x`, `&mut x`, direct read/write of `x`,
  or `unique` consumption of `x`.
- Passing `&mut x` into a function suspends caller-side exclusivity for the call,
  then restores it if `x` was not consumed.

**`&` shared:**

- Multiple `&x` allowed.
- While any `&x` is active: no `&mut x`, no mutation of `x`, no `unique` consume.

**Escape (forbidden):**

- `return &local;`
- Store `&` into struct fields, globals, or containers that may outlive the referent
  (**reference members** deferred).

Diagnostics cite the conflicting borrow or consume; **no lifetime labels**.

### D6 — Consumed state (not “move semantics”)

After **consumption** (passing to `unique T`, or receiving from `unique` return):

1. The source variable is **consumed** — may not be read, borrowed, assigned from
   on the RHS of copy, or passed to `unique` again.
2. **Reinitialization**: `x = expr;` where `x` was consumed assigns a **new owned
   value** to `x`; `x` is usable again.
3. At scope exit, **`drop` on consumed variables is a no-op** (nothing left to release).

This is **not** a general moved-from lattice; it applies only to **`unique`
transfer sites**.

### D7 — Deterministic destruction (`drop`)

Every **owned** value in scope is destroyed in **reverse construction order** at
scope exit ([0002](0002-design-principles.md)):

1. Checker records which slots own heap resources.
2. KIR emits **`Drop`** (or equivalent) before slot destruction on all paths
   (including `break`/`continue`/`return` — aligned with existing control-flow
   work).
3. `libkinglet_rt` exposes `kl_drop_*` / type-specific destructors that release
   unique heap storage.
4. **`&` / `&mut` do not drop** the referent early; only owning slots drop.

Assignment `a = b` **drops the previous value of `a`** before installing the new
copy (D2).

### D8 — Copy vs borrow vs unique (summary)

| Intent | Surface syntax | Ownership effect |
|--------|----------------|------------------|
| Read without cloning | `&T` param / `&x` | None; borrow |
| Mutate in place | `&mut T` param / `&mut x` | None; exclusive borrow |
| Duplicate value | `T` param, `a = b` | **Copy** (clone heap if needed) |
| Transfer value | `unique T` param / return | **Consume** source |
| End of scope | (implicit) | **drop** owned slot |

**Function examples:**

```kl
void f(string s);              // callee owns a copy of caller's s
void g(&string s);             // borrow
void h(&mut Buffer b);         // exclusive mutate
void k(unique string s);       // caller loses s
```

### D9 — Runtime (`libkinglet_rt`) consequences

1. **Remove reliance on shared refcount** as the language semantics for `string` /
   `array` / `map` between owned variables (RC may remain **inside** runtime
   implementation details for small optimizations, but **not** observable as shared
   ownership between two Kinglet variables).
2. **Clone** API for assignment and copy constructors: `kl_clone_string`, etc.
3. **Drop** API per type: invoked from KIR `Drop`.
4. Wire format ([0015](0015-llvm-backend-roadmap.md)): owned heap handle is an
   opaque owning pointer or tagged handle with **destructor responsibility** tied
   to the owning slot, not a shared RC bump on copy.

Legacy `0xFFFE<<48` tagging from the deleted VM era may remain in RT structs during
refactor; semantic contract is **unique handle per owner**.

### D10 — Generics, structs, and enums

1. **Structs**: each field follows D2 per field type; struct **assignment** is
   fieldwise copy; struct **drop** is fieldwise drop in reverse field order.
2. **Enums with payload**: owned discriminator + owned payload drop on the active
   variant.
3. **Generics**: `unique T` / `&T` / `&mut T` require monomorphization; reference
   and consume rules apply per instantiation.
4. **`match`**: v1 bindings copy by value (as today); `match` on `unique` or by-ref
   patterns deferred.

### D11 — Relationship to [0021](0021-references-and-move.md)

[0021](0021-references-and-move.md) is **superseded** by this ADR for:

- Implicit move / transfer sites
- Dual VM/native lowering notes
- COW as part of the language copy model

[0021](0021-references-and-move.md) borrow exclusivity and escape rules are
**carried forward** in D5 (unchanged intent, native-only lowering).

### D12 — Implementation order

| Phase | Deliverable | Gate |
|-------|-------------|------|
| **N0** | This ADR accepted; **VM backend deleted** (D1); CI on native only | CI green |
| **N1** | KIR `Drop` + RT `kl_drop_*` for `string` (minimal) | native test |
| **N2** | `&T`, `&expr`, auto-deref, escape errors | sema |
| **N3** | `&mut T` + exclusivity | sema fail cases |
| **N4** | `unique T` consume + consumed-state errors | sema + native |
| **N5** | Clone on assign for heap types; drop on reassign | behaviour tests |
| **N6** | Extend to `array`, `map`, struct, enum payload | native manifest |
| **N7** | Self-host `ll_emit` parity on native | self-host chain |

Bootstrap (Ref) leads each phase; self-host `ll_emit` follows per slice. **N0 is not
blocked on N1–N7** — VM removal lands with ADR acceptance.

### D13 — Amendments to prior ADRs (summary)

| ADR | Amendment |
|-----|-----------|
| [0002](0002-design-principles.md) | Heap types use **explicit unique ownership**; scalars stay copy-by-default. Pillar 3 clarified: no **pervasive** Rust-style ownership, but **localized** ownership for heap resources. |
| [0010](0010-vm-redesign.md) | RC+COW model is **historical** (VM deleted). |
| [0014](0014-compilation-toolchain-architecture.md) | D3: kbc/VM **removed**; native is the **only** backend. |
| [0015](0015-llvm-backend-roadmap.md) | Drop insertion is **required** for language ownership (not optional perf). |
| [0021](0021-references-and-move.md) | **Superseded** by this document. |

## Consequences

### Positive

- One coherent story: **borrow with `&` / `&mut`, own with scope `drop`, transfer
  with `unique`** — readable without `move()` or implicit last-use.
- Native lowering maps cleanly to **pointers + destructors** (LLVM + RT).
- Self-host gen2/gen3 parity targets **ABI and behaviour** on native only.
- **Immediate VM deletion** removes dual-design tax in one step (no deprecation period).

### Risks and constraints

- **N0 VM deletion** may temporarily shrink test coverage until native suites replace
  dropped kbc tests; accept that trade-off rather than keeping VM alive.
- **Assignment copy** on heap types is **O(n)** until optimized clones or small-string
  optimization exist; hot paths should use **`&` / `&mut` / `unique`**, not repeated
  `=`.
- **Checker** must track borrows, consumption, and drop paths on all CFG edges.
- **Runtime migration** from de-facto shared handles to unique+clone is breaking at
  RT level; staged per type (N1…N6).
- **0002 table** (“ownership absent”) is outdated for heap types — documentation and
  diagnostics must use the amended story (D13).

### Acceptance criteria (N4 + N5)

- [ ] `&T` / `&mut T` / `&expr` with D5 diagnostics on native
- [ ] `unique T` consume with consumed-state errors
- [ ] `string` assignment clones; prior value dropped
- [ ] Scope exit drops owned `string` on all exit paths
- [ ] Bootstrap + self-host native agree on stdout/exit for gated cases

## Dependencies

- [0002](0002-design-principles.md) — deterministic destruction (amended for heap ownership).
- [0005](0005-backend-architecture.md) — KIR seam.
- [0015](0015-llvm-backend-roadmap.md) — native backend and RT.
- [0016](0016-typed-kir.md) — typed slots for drop and references.
- [0020](0020-project-manifest-and-targets.md) — native default build.
- [0021](0021-references-and-move.md) — superseded.

## References

- Native gaps: [native.md](../docs/native.md)
- Ref lowering: `bootstrap/src/codegen/llvm/kir_to_llvm.cc`
- Self-host lowering: `kinglet/compiler/ll_emit.kl`
- RT: `kinglet/kinglet_cruntime/`, `bootstrap/runtime/`

## Open questions

1. **Keyword vs qualifier** — `unique T` as a type prefix (preferred) vs `own T`;
   finalize spelling before parser work.
2. **Small buffer optimization** — inline string storage in the value slot to reduce
   heap+clone pressure (implementation, not language).
3. **0002 formal amendment** — separate one-paragraph amendment entry vs inline D13
   only (recommend: add Amendments section to 0002 when N1 lands).
