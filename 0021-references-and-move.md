# 0021 — References and Move Semantics

- **Status**: superseded-by [0022](0022-native-unique-ownership.md)
- **Proposed**: 2026-06-21

## Amendments

### 2026-06-21 — Superseded by native unique ownership ([0022](0022-native-unique-ownership.md))

This draft is **withdrawn**. Kinglet no longer designs language features for the
VM backend; implicit move semantics and COW-based copy are replaced by **native-only
unique ownership**, **`&` / `&mut` borrows**, **`unique T` transfer**, and **scope
`drop`**. See [0022](0022-native-unique-ownership.md).

Original sections below are preserved for historical context.

---

- **Status**: draft
- **Proposed**: 2026-06-21

## Context

[0002](0002-design-principles.md) defines Kinglet as **copy-by-default** with
**opt-in reference semantics**, **deterministic destruction**, and **no ownership
tracking or borrow checking**. That direction was stated in 2025 but never landed in
the language surface:

- `&` in the lexer is **bitwise and** only (`&`, `&&`).
- Parameters, locals, and returns are always **by value** at the type level.
- Assignment and calls **copy** (heap aggregates use RC + COW in the VM per
  [0010](0010-vm-redesign.md); native lowering still lacks scope-exit **drop**
  insertion per [0015](0015-llvm-backend-roadmap.md) and [native.md](../docs/native.md)).

Meanwhile, real programs pay unnecessary copy costs on `string`, arrays, maps, and
structs, and APIs cannot express **in-place mutation** without returning a new value.
[0001](0001-pending-syntax-and-perf.md) lists **trivial relocatability** (P1144) as a
future performance item but does not define syntax.

This ADR is the **language-capability priority** ahead of stdlib migration ([0003](0003-stdlib-roadmap.md)),
concept instantiation checks ([0009](0009-concepts-landing.md)), and further manifest
work ([0020](0020-project-manifest-and-targets.md)). It specifies `&T`, `&mut T`, and
**implicit move semantics** in a **C++-family** model: references without lifetime
annotations, **no `move()` keyword**, and copy remaining the default at assignment and
ordinary use.

## Decision

### D1 — Alignment with design pillars (0002)

| Pillar | How this ADR honours it |
|--------|-------------------------|
| Full value semantics | **Copy remains the default.** `int a = b;` and `string s = t;` still mean independent values unless the RHS is a reference or a **transfer site** applies implicit move (D5). |
| Deterministic destruction | Every owned value is destroyed at scope exit. **Implicit move transfers ownership** at transfer sites; the source becomes **moved-from** and its destructor is a no-op for transferred members. |
| Zero-cost without ownership | References are **non-owning aliases** (no refcount change on borrow). Move on relocatable types lowers to **pointer/bitwise transfer**, not a global borrow graph. |

**Explicit non-goals (not Rust):**

- No `'` lifetime parameters or elision rules.
- No borrow checker across function boundaries beyond the rules in D4.
- No **`move(expr)`** keyword or builtin — move is **compiler-applied**, not written by the user.
- Move is **not** the default for arbitrary reads or assignments.
- No `dyn` / trait objects as part of this RFC.

### D2 — Reference types: `&T` and `&mut T`

Reference types are written **`&T`** (shared, read-only) and **`&mut T`** (exclusive,
mutable). They are **first-class in the type system** but **non-owning**.

```kl
void inspect(&string s) { io::out.line(s.len()); }
void append(&mut string s, string tail) { s = s + tail; }
```

Rules:

1. **`&T`**: may read the referent; may coerce to `&U` only under existing assignment
   subtyping (unchanged variance rules; references follow the referent).
2. **`&mut T`**: may read and write the referent; implicit coercion to `&T` is allowed
   (read-only view of the same location).
3. References may bind to **locals, parameters, struct fields, and indexed lvalues**
   when the base is an lvalue (see D3).
4. **Nullable references** are out of scope for v1 (`&T?` deferred).

### D3 — Address-of: `&expr`

**`&expr`** takes the address of an **lvalue** and produces `&T` or `&mut T`:

| Operand | Result type |
|---------|-------------|
| `x` where `x` is `T` and mutable | `&mut T` |
| `x` where `x` is `T` and not mutable (e.g. `const`, read-only field path) | `&T` |
| `&mut x` used where `&T` expected | coerces to `&T` |

**Not an lvalue** (compile error for `&expr`):

- Rvalues: literals, call results, temporaries from expression operators, unless an
  explicit **lifetime extension** rule applies (v1: **no** extended temporaries; bind to
  a named local first).
- Moved-from objects (D6).

**Dereference** v1: field access and index on a reference **auto-deref** (UFCS and
method calls use the referent type). A postfix `.*` / explicit `*` operator is deferred
unless formatter or parsing ambiguity requires it.

### D4 — Exclusivity and escape (no lifetime syntax)

Borrowing is checked **intra-procedurally** with simple call-tree rules — stricter than
C++, weaker than Rust.

**Exclusivity (`&mut`):**

1. While `&mut x` is **active** (see below), no other `&x`, `&mut x`, or use of `x`
   that reads/writes the value is allowed in the same scope.
2. **Active** means: from the point `&mut x` is created until the end of the scope in
   which it was created, **except** that passing `&mut x` into a function call ends
   exclusivity for the **duration of that call** (reborrow for callee), then restores
   exclusivity of the caller binding if the referent still exists.

**Shared (`&`):**

1. Multiple `&x` may coexist.
2. While any `&x` is active, **`&mut x` and mutating `x` directly are forbidden**.
3. While `&mut x` is active, **`&x` is forbidden**.

**Escape (always forbidden in v1):**

- `return &local;` or returning any reference to a stack slot that does not outlive the
  callee.
- Storing a reference into a struct field, global, or container that may outlive the
  referent (**reference members** deferred to a later RFC).
- `&` of a **temporary** binding (including `&make_node(1)`).

Diagnostics should cite the conflicting borrow or escape (no lifetime labels).

### D5 — Implicit move semantics (no `move()` syntax)

Kinglet has **move semantics** but **no `move(expr)`** in source. The compiler **transfers
ownership** at defined **transfer sites** instead of emitting a copy. Users write ordinary
expressions; the checker marks operands as moved when a transfer site consumes them.

```kl
string consume(string s) {
  return s;                    // implicit move: s → return slot
}

void push(&mut string[] arr, string item) {
  arr.push(item);              // implicit move at last-use call arg (M2)
}
```

**Transfer sites (v1 → v2):**

| Site | v1 | Later |
|------|----|-------|
| `return local;` | **Implicit move** of `local` when type is relocatable (D7) | NRVO hints for native |
| `return expr` (non-local) | Copy / move per expr rules (rvalue → move into return slot) | — |
| By-value call argument | **Copy** (unchanged default) | **Implicit move** when operand is a local and **last use** in scope (M2) |
| Assignment `a = b` | **Copy** | Last-use move optional future RFC |
| Struct literal field `S { f: x }` | Copy | Last-use move per field (partial move RFC) |

Rules:

1. There is **no** `move` keyword, builtin, or stdlib helper required for ordinary transfer.
2. After implicit move from variable `x`, **`x` is moved-from** (D6).
3. Implicit move on a **borrowed** lvalue is a compile error.
4. Implicit move through a **reference** (`&T`) is forbidden; transfer applies to **owned**
   values only.
5. For trivially relocatable scalars, implicit move is **observationally equivalent** to
   copy (D7); moved-from diagnostics still apply where the checker tracks ownership.

### D6 — Moved-from state

After an implicit move from variable `x`:

1. **`x` may not be read**, written, transferred again, or borrowed (`&` / `&mut`).
2. **`x` may be assigned** a new value (reinitialization); after assignment, `x` is a
   normal variable again.
3. At scope exit, destructor runs on **`x`**; for moved-from state the destructor is a
   **no-op** for members already transferred (deterministic destruction preserved).

Partial moves (individual struct fields) are **phase 2**; v1 transfers the **whole value**
at each transfer site.

### D7 — Type coverage

All **value types** participate in the reference and implicit-move model:

| Type class | `&` / `&mut` | Implicit move | Notes |
|------------|--------------|---------------|-------|
| `int`, `bool`, `byte`, fixed-width ints | yes | trivial (≈ copy) | References mainly for generics/API uniformity |
| `float` / `float32` / `float64` | yes | trivial | Same |
| `string` | yes | **transfers heap handle** | Primary beneficiary |
| `T[]`, maps | yes | **transfers heap handle** | Primary beneficiary |
| `struct`, `enum` (with payload) | yes | **memberwise** when all fields relocatable | Non-relocatable fields block move until drop/ABI defined |
| `void`, function types | no | no | Ill-formed |

**Trivial relocatability** (P1144): scalars and raw heap handles are trivially
relocatable; struct move is memberwise when every field is relocatable; enum move
transfers active variant payload. The compiler **does not** expose a `relocatable`
trait in v1 — analysis is internal.

### D8 — Interaction with copy and parameters

| Site | Default when type is `T` | With `&T` / `&mut T` | At transfer site (D5) |
|------|--------------------------|----------------------|------------------------|
| Parameter | by-value **copy** | alias, no copy | by-value **transfer** when last-use rule applies |
| Return `local` | — | forbidden (escape) | **implicit move** |
| Assignment `a = b` | **copy** | N/A (`a` must be value) | copy in v1 |

**Function declarations** use the same type syntax as today:

```kl
void f(string s);           // callee gets a copy (v1) or transfer (M2 last-use)
void g(&string s);          // borrow for read
void h(&mut Buffer b);      // borrow for write
```

No `fn` keyword; C-style declarations retained ([0002](0002-design-principles.md)).

### D9 — Lowering and backends

**VM (phase R1 / M1):**

- `&T` / `&mut T` parameters: pass **stable slot id** or indirect pointer to the
  referent; callee loads through indirection.
- **Implicit move**: transfer heap handle in the wire representation; source slot set to
  **null / empty** sentinel; RC decrement skipped for transferred handle.
- Existing RC + COW ([0010](0010-vm-redesign.md)) remains; `&mut` mutation may trigger
  COW split on write as today.

**Native (phase R1 / M1 — blocks on drop):**

- References lower to **pointers** to allocas or struct slots with typed KIR ([0016](0016-typed-kir.md)).
- **Implicit move requires scope-exit drop** ([0015](0015-llvm-backend-roadmap.md) D5):
  insert `drop`/`destruct` at scope end for owned values; transfer elides drop on
  moved-from members.
- Self-host (`ll_emit`) and bootstrap (`kir_to_llvm`) must stay behaviour-aligned on
  supported cases; VM leads, native follows per existing dual-backend policy.

### D10 — Implementation order

Landing target: **self-host compiler first**, bootstrap parity per slice ([0005](0005-backend-architecture.md)).

| Phase | Deliverable | Gate |
|-------|-------------|------|
| **R0** | This ADR accepted | — |
| **R1** | `&T`, `&expr`, read-only borrows, auto-deref, escape errors | `tests/sema` + probe |
| **R2** | `&mut T`, exclusivity diagnostics | sema fail cases for conflicting borrows |
| **M1** | Implicit move on `return local;`, moved-from, use-after-move errors | probe + VM golden |
| **M2** | Last-use implicit move into by-value call arguments | sema + behaviour tests |
| **N1** | Native drop + implicit-move lowering | `tests/native` manifest expansion |

**Deferred after M2:**

- Partial struct field move
- Reference members in structs
- `&T?` nullable references
- Explicit `*` / `.*` operators (if auto-deref proves insufficient)
- User-facing `move()` syntax (not planned; revisit only with strong justification)

### D11 — Amendments to pending items

Promotes from [0001](0001-pending-syntax-and-perf.md):

- **Trivial relocatability** → internal analysis in D7 (not a user-facing keyword).
- Reference types and move semantics → this ADR (no `move()` surface syntax).

Does **not** implement [0001](0001-pending-syntax-and-perf.md) items: `once`, `retry`,
inline tests, `scope` blocks, `?` propagation, `int?` niche, struct match patterns.

## Consequences

### Positive

- Large aggregates become cheap to return (`return local;`) and eventually to pass without
  a `move()` noise keyword.
- `&mut` enables in-place APIs (`push`, `sort`, parsers) without copy-update-return.
- Deterministic destruction stays explicit; transfer sites clarify **when** ownership moves.
- Full type coverage avoids a “second-class” set of types that cannot be borrowed or moved.

### Risks and constraints

- **Checker complexity** rises (borrow conflicts + last-use analysis for M2); diagnostics
  must be actionable (“value moved here” without pointing at a `move()` call).
- **Native drop insertion** is prerequisite for correct implicit move on aggregates; VM
  can ship earlier.
- **Generic functions** over `&T` require monomorphization and reference-aware codegen
  (phase R2+).
- **Bytecode / KIR opcode** additions must append only ([0008](0008-kbc-format-evolution.md),
  [0013](0013-bootstrap-bytecode-delta.md)); expect golden refreshes.

### Acceptance criteria (M1 complete)

- [ ] `&T` / `&mut T` in parameter and local types; `&expr` on lvalues
- [ ] Exclusivity and escape rules enforced with tests
- [ ] Implicit move on `return local;` with moved-from and use-after-move errors
- [ ] `string`, `struct`, `array` covered on VM path
- [ ] Bootstrap and self-host agree on sema diagnostics and runtime behaviour for gated cases
- [ ] Native drop + implicit move for at least `string` and one struct case in `tests/native`

## Dependencies

- [0002](0002-design-principles.md) — copy default, opt-in references, deterministic destruction.
- [0010](0010-vm-redesign.md) — RC + COW heap representation.
- [0015](0015-llvm-backend-roadmap.md) — native drop / destruction lowering.
- [0016](0016-typed-kir.md) — typed slots and aggregate metadata for native refs.
- [0012](0012-test-suite-redesign.md) — sema / probe / native gates.

## References

- Self-host checker: `checker/checker.kl`
- Self-host codegen: `compiler/codegen.kl`
- Bootstrap checker: `bootstrap/src/checker/type_checker.cc`
- Bootstrap VM: `bootstrap/src/vm/`
- Native lowering: `bootstrap/src/codegen/llvm/kir_to_llvm.cc`, `kinglet/compiler/ll_emit.kl`
- C++ relocatability: [P1144](https://wg21.link/p1144), [P2786](https://wg21.link/p2786)

## Open questions

1. **Assignment to reference slots** — `&mut x = &mut y` rebind vs illegal (v1: illegal;
   only assignment through paths on the referent, i.e. `x = expr` on the value).
2. **Match bindings** — `match v { Some(&mut x) => ... }` deferred; v1 match arms bind
   by value copy as today.
3. **Formatter** — whether `& mut` spacing is enforced by `preen` (cosmetic; follow
   `SYNTAX.md` when updated).
