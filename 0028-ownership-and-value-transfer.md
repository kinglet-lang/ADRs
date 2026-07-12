# 0028 — Ownership, Borrowing, and Value Transfer

- **Status**: draft
- **Proposed**: 2026-07-11

**Replaces**: [0022 — Native-Only Toolchain and Unique Ownership](%5Bdeprecated%5D%200022-native-unique-ownership.md)
for everything after D1 (native-only toolchain, already implemented and
carried forward unchanged — see D0 below). Also fully absorbs the borrow
exclusivity and escape rules originally proposed in
[0021 — References and Move Semantics](%5Bdeprecated%5D%200021-references-and-move.md)
(already superseded by 0022; restated here in full so this document is
self-contained).

Depends on: [0029 — Value Representation and Memory Layout](0029-value-representation-and-memory-layout.md)
for the type-tier classification (which types are Copy scalars vs. heap
handles vs. resource types) that the rules below are defined in terms of.

## Context

[0022](%5Bdeprecated%5D%200022-native-unique-ownership.md) accepted a **unique-heap, copy-scalar**
ownership model: heap types (`string`, `T[]`, maps, structs/enums with heap
fields) clone on every plain assignment and by-value parameter, and only an
explicit `unique T` parameter/return type transfers ownership. That model has
not progressed past N0 (native-only toolchain, already delivered) — no
`unique T` consumption, no KIR `Drop`, no `kl_drop_*` / clone API set exists in
bootstrap today.

Two problems surfaced while re-examining this direction against actual runtime
behaviour and against a competing draft (`kinglet-ownership-and-memory-model.md`,
a systems-language-style ownership proposal favouring implicit move):

**1. The "clone on plain assignment" default contradicts Kinglet's own
zero-cost pillar.** [0002](0002-design-principles.md) pillar 3 promises
zero-cost abstraction; 0022 D2 makes the single most common operation on a
heap type (`string b = a;`, or passing a `string` by value) silently allocate
and deep-copy. That is not zero-cost — it is a hidden allocation on every
plain use, the exact C++ implicit-copy-constructor cost Kinglet's design
principles were written to avoid.

**2. A fully implicit move-by-default model (the competing draft's direction)
creates its own class of problems that a systems language audience will hit
immediately:**

```kl
string a = "hi";
foo(a);   // if foo takes `string` by value and this is implicit move...
bar(a);   // ...this is now a compile error, even though the intent
          // was "let two functions each look at a", not "give a away"
```

If **every** plain assignment or by-value parameter passing implicitly moves a
heap-owning value, users must either restructure every such call to borrow
(`&T`) or sprinkle explicit `.clone()` everywhere — the exact "clone hell"
friction that makes Rust's learning curve steep, imported wholesale into a
language whose explicit design goal is to avoid that weight. Worse, implicit
move at **every** assignment site requires the checker to perform general
intra-procedural data-flow analysis to track which branches have or have not
consumed a given variable:

```kl
string a = "hi";
if cond {
    consume(a);   // moved on this path only
}
io::out.line(a);   // valid or not depends on which branch ran —
                    // requires definite-move / maybe-move dataflow tracking,
                    // not just a borrow-stack push/pop
```

The current borrow checker (`active_borrows_` in `type_checker.cc`) is a
simple push/pop stack keyed by variable name; it has no cross-branch dataflow
capability, and building one is a much larger investment than the ownership
model needs to pay for.

This ADR resolves both problems with a **three-tier default model** keyed off
value category, transfer **limited to call boundaries** (not general
assignment), and syntax built entirely from **existing tokens** (`&`, `const`,
and a stdlib `move()` function) rather than new keywords or a new `&&`
sigil — the ADR body records the sigil/keyword alternatives considered and
rejected in D3.

## Decision

### D0 — Carried forward unchanged from 0022

The following is **implemented** and not reopened by this ADR:

- **Native-only toolchain** (0022 D1): the VM backend is deleted; native
  (KIR → LLVM → `libkinglet_rt`) is the only execution and toolchain backend.
  Implemented 2026-06-20 (commit `ba20344`), with bytecode-container cleanup
  continuing through 2026-07-10 (`ea4f69a`, `54df929`).
- **Scope `drop` as the destruction mechanism** (0002, 0022 D7): every owned
  value in scope is destroyed in reverse construction order at scope exit.
  This ADR keeps that mechanism; D8 below restates and completes it.

Everything else in 0022 (D2 clone-on-assignment, D4 `unique T`, D6
consumed-state, D9 runtime clone API) is **superseded** by this document.

### D1 — Type categories (summary; full rules in 0029)

Ownership behaviour is defined per **type category**, established in
[0029](0029-value-representation-and-memory-layout.md):

| Category | Examples | Ownership behaviour |
|----------|----------|----------------------|
| **Scalar (Copy)** | `int8`…`int64`, `uint8`…`uint64`, `float32`, `float64`, `bool`, `char` | Always copies; never transfers; no heap backing |
| **Value type (heap-backed)** | `string`, `T[]`, `map<K,V>` | Copies on plain assignment; transfers only at call boundaries (D4) |
| **Resource type** | `box<T>`, `fs::file`, `net::tcpstream`, `mutex`, user-defined non-cloneable types | Always transfers; no clone exists |

A type's category is a property of the type itself (declared in 0029), not of
how it is used at a call site. The rules below apply this classification
uniformly.

### D2 — Scalars: copy always, transfer is meaningless

For scalar types, plain assignment, by-value parameter passing, and any
`move()` call are **all the same operation** — a bitwise copy. There is no
heap allocation, no handle, and therefore nothing that could be invalidated by
a "transfer":

```kl
void sink(int s) { }
int a = 2;
sink(a);
io::out.line(a);   // valid — a is untouched; sink(int) is a plain copy
```

`sink(int)` here is not a transfer site merely because its parameter is a bare
type — bareness only triggers transfer semantics for value types and resource
types (D4). Scalars are exempt. This is a hard rule, stated explicitly to
avoid readers over-generalizing "bare parameter type == transfer" to every
type in the language.

### D3 — Syntax: `T`, `const T&`, `T&` — no new sigils

The surface syntax uses only symbols already in the language:

```kl
void sink(string s);              // transfer: caller loses ownership
void inspect(const string& s);    // shared borrow: read-only, caller keeps ownership
void update(string& s);           // exclusive borrow: read/write, caller keeps ownership
```

`const T&` replaces 0022's `&T`; `T&` (no `mut`) replaces 0022's `&mut T`. The
`mut` pseudo-keyword is **removed** (D7). There is no `unique T` and no `&&T`.

**Alternatives considered and rejected:**

1. **`unique T` qualifier** (0022's original design). Rejected: reads as a
   qualifier bolted onto a type, whereas `T` bare in parameter position
   already unambiguously means "the callee owns this" once the transfer rule
   (D4) is fixed — a second keyword for the same idea is redundant surface
   area.
2. **`&&T` sigil**, modelled on C++ rvalue-reference syntax, to mean
   "transfer". Rejected on parser architecture grounds, not just taste: `&&`
   is already lexed as a single token (`AMP_AMP`) and is the **only** spelling
   of logical AND (`logical_and()` in `parse_expr.cc`, its sole consumer).
   Introducing `&&T` in type position is technically implementable — the
   parser already dispatches purely on **position** (`parse_type_expr()` vs.
   expression-precedence climbing, `unary()`), and logical AND can never
   appear in type position or at expression-prefix position, so there is no
   parse-time ambiguity. The objection is that it forces one token to carry
   two semantically unrelated meanings (boolean logic vs. memory ownership)
   distinguished **only** by position, which is a real maintenance cost for
   any tool that inspects tokens without a full parse (syntax highlighters,
   LSP hover, the `preen` formatter) and invites confusion with C++ `T&&`
   move semantics, which is a different mechanism (leaves a valid
   "moved-from" object; Kinglet transfer invalidates the binding outright).
3. **`~T`** as a transfer sigil. Rejected for a milder version of the same
   objection: `~` already has an unrelated meaning (bitwise NOT, prefix
   position in `unary()`); reusing it for ownership transfer is a smaller but
   still real instance of symbol overloading across unrelated domains.
4. **`own T` keyword**. Not rejected on technical grounds — this remains a
   viable alternative to bare `T` if a future revision decides bare-`T`
   transfer is too implicit to read safely at a call site. Not adopted now
   because D4 makes the transfer rule apply only at a narrow, syntactically
   visible location (call boundaries), which was judged sufficient without an
   additional keyword.

### D4 — Transfer sites: call boundaries only, not general assignment

**Transfer is limited to function call boundaries** — a value-type or
resource-type parameter declared with a bare type, or a bare-typed return —
and does **not** apply to plain local-to-local assignment:

```kl
void sink(string s) { }
string a = "hello";
sink(a);
io::out.line(a);   // must fail: a was transferred into sink's parameter
```

```kl
void inspect(const string& s) { }
string a = "hello";
inspect(a);
io::out.line(a);   // valid: inspect only borrowed a, a is untouched
```

```kl
string a = "hi";
string b = a;      // plain local assignment: copy (D5), NOT transfer
io::out.line(a);   // valid — a is still valid after a plain copy
```

This is the deliberate resolution of the Context section's tension: transfer
happens at a **small, lexically obvious set of positions** (call arguments,
return values) rather than at every assignment. The checker therefore never
needs cross-branch "was this maybe-moved on some paths" dataflow analysis —
each call site is checked once, independent of control flow, because the
transferred variable's scope-local liveness after the call is a strictly
local fact at that one call.

Resource types (`box<T>`, `fs::file`, `mutex`, …) transfer on **both** call
boundaries and plain assignment, per D6, because they have no copy to fall
back on — see D6 for why this does not reopen the dataflow problem.

### D5 — Value types: copy by default, transfer only via D4

For value types (`string`, `T[]`, `map<K,V>`), plain local assignment and
non-transfer uses default to **copy** (allocate + deep clone), matching
0022 D2's original assignment behaviour but **not** its by-value-parameter
behaviour, which D4 changes to transfer:

```kl
string a = "hi";
string b = a;      // copy: independent heap allocations
b = b + "!";       // does not affect a
io::out.line(a);   // "hi" — unaffected
```

Avoiding the copy for a plain local-to-local binding requires the explicit
`move()` stdlib function (D9), not a new operator.

### D6 — Resource types: always transfer, no clone exists

Resource types (`box<T>`, `fs::file`, `net::tcpstream`, `mutex`, `thread`, and
user-defined types that opt out of cloning) have no meaningful copy
operation — duplicating a file descriptor or a lock handle is not "the same
value in two places" the way duplicating a string's bytes is. For these
types, **every** plain assignment and every bare-typed call boundary is a
transfer:

```kl
box<Node> a = box(Node{});
box<Node> b = a;    // transfer: a has no clone to fall back to
sink(a);             // ERROR: a already transferred to b
```

This does not reintroduce the cross-branch dataflow problem from the Context
section, because the set of types subject to "assignment is always transfer"
is a **short, explicit, closed list of resource types** declared in 0029, not
"every heap type" — the common case (`string`, `T[]`, `map`) stays on the
simple copy-by-default rule (D5), and only code that opts into a resource
type takes on consumed-state tracking for that variable.

A `T&` / `const T&` parameter on a resource type is still just a borrow (D5's
sibling rule) and does not transfer — only bare `T` parameters and plain
assignment transfer for this category.

### D7 — `mut` is removed; `const` is overloaded across two independent axes

`mut` currently exists only as an `IDENTIFIER`-text comparison inside
`&` parsing (`parse_type.cc:113`, `parse_expr.cc:251`), never as a standalone
modifier — `mut T` in isolation has never parsed. This ADR removes it
entirely: mutability of a plain variable binding continues to be governed
solely by `const` (existing rule, `type_checker.cc:1816`:
`is_mutable = var_decl.storage != "const"` — variables are mutable **by
default**, and `const` is the explicit opt-out), and mutability of a
*reference* is now governed by the presence or absence of `const` in
`const T&` vs. `T&`.

`const` therefore carries **two independent meanings depending on position**,
the same overload C++ has used for decades and that Kinglet users are assumed
to already be familiar with:

```kl
const int y = 5;        // const on a plain binding: y cannot be reassigned
y = 6;                    // ERROR

void inspect(const string& s) { }   // const on a reference: the referent
                                      // cannot be mutated through s; the
                                      // caller's own variable is unaffected
```

These two axes (can the **binding** be reassigned; can the **referent** be
mutated through this particular reference) are orthogonal and do not need
separate keywords — this mirrors `const int&` in C++, not a new invention.

### D8 — Deterministic destruction (`drop`), carried forward from 0022 D7

Unchanged from 0022, restated for completeness since this document supersedes
0022 in full:

1. Every owned value in scope is destroyed in **reverse construction order**
   at scope exit ([0002](0002-design-principles.md)), on all control-flow
   exit paths (`break` / `continue` / `return`).
2. The checker records which slots own heap or resource-backed storage; KIR
   emits a `Drop` (or equivalent) instruction before slot destruction.
3. `libkinglet_rt` exposes `kl_drop_*` / type-specific destructors.
4. `&` / `const T&` / `T&` borrows **do not** drop their referent; only
   owning slots drop.
5. A variable already transferred out (D4/D6) is a no-op at scope exit —
   nothing is left to release.
6. Assignment `a = b` on a value type (D5) drops the previous contents of `a`
   before installing the freshly cloned copy.

### D9 — Explicit local transfer: `move()` as a plain stdlib function, not new syntax

Because D4 restricts transfer to call boundaries, an explicit local-to-local
transfer (skip the D5 copy) is expressed as a call to an ordinary identity
function in the standard library — **no new operator or keyword is
introduced**:

```kl
T move(T value) {
    return value;
}
```

Calling it is an ordinary call expression; the transfer happens because
`move`'s parameter is bare `T`, exactly per D4 — `move` requires no special-case
handling anywhere in the parser, checker, or codegen:

```kl
string a = "hi";
string b = move(a);   // a transferred into move()'s parameter, returned to b
io::out.line(a);       // ERROR: a already transferred
io::out.line(b);       // valid
```

```
before: a ──► [heap: "hi"]
after:  a ──► (invalidated)     b ──► [heap: "hi"]   (same allocation, handle relocated)
```

For resource types, `move(a)` is legal but redundant — plain `b = a;` already
transfers per D6.

### D10 — Borrowing rules (carried forward from 0021/0022, restated in full)

Checked **intra-procedurally**, with call-boundary reborrow — absorbed in
full from 0021 D4 / 0022 D5 so this document is self-contained; 0021's
original file is deprecated and this is now the canonical text.

**Exclusivity (`T&`, exclusive/mutable borrow):**

1. While a `T&` borrow of `x` is active, no other `const T&` of `x`, no other
   `T&` of `x`, no direct read/write of `x`, and no transfer of `x` is
   permitted in the same scope.
2. **Active** means: from the point the borrow is created until the end of
   the scope in which it was created, **except** that passing a `T&`
   argument into a function call suspends the caller-side exclusivity for the
   duration of that call (reborrow for the callee), then restores caller
   exclusivity afterward if the referent was not transferred during the call.

**Shared (`const T&`):**

1. Multiple `const T&` borrows of the same `x` may coexist.
2. While any `const T&` of `x` is active: no `T&` of `x`, no mutation of `x`,
   and no transfer of `x`.

**Escape (forbidden in all cases):**

- `return &local;` or returning any borrow of a stack slot that does not
  outlive the callee.
- Storing a borrow into a struct field, global, or any container that may
  outlive the referent (reference-typed struct members remain deferred, as in
  0021/0022).
- Borrowing a temporary / rvalue (`const string& v = &make_node();` is
  ill-formed — bind to a named local first).

**Non-lexical borrow duration:** borrow lifetime follows last actual use, not
lexical scope end — carried forward from 0021 §7.4 / 0022 D5's intent (no
change; both prior drafts already agreed on this point).

```kl
const string& view = &name;
io::out.line(view);
// last use of view is the line above
name.clear();   // valid — view's borrow already ended
```

Diagnostics cite the conflicting borrow or transfer; there are **no lifetime
labels** in ordinary code (D11).

### D11 — Lifetimes: inferred by default, no explicit syntax in v0

Ordinary code never writes a lifetime annotation. This position is shared by
both source drafts for this ADR (0022 D5's "no lifetime labels" and the
competing draft's "explicit lifetime annotations are advanced API syntax, not
routine syntax") and is adopted without modification. Elision follows the
existing intra-procedural borrow-check shape; cross-function lifetime
parameters remain out of scope for v0 and are not designed here.

### D12 — Borrow checker: known gap first, then place-based refinement

The current implementation gap is tracked as a **prerequisite bug fix**,
separate from any new design work this ADR requires:

1. **Existing gap (fix regardless of this ADR):** `check_referent_access`
   (`type_checker.cc:2110`, `2318`) is the only enforcement point for D10's
   exclusivity rules, and it is called from exactly two sites — identifier
   read and identifier-assignment. Field assignment (`x.field = v`) and index
   assignment (`arr[i] = v`) never call it at all, meaning D10's exclusivity
   rules are silently unenforced through those paths today. This is a defect
   against the already-accepted 0022 borrow rules, not a new decision, and
   should be fixed independent of the rest of this ADR's rollout.
2. **New refinement (this ADR, later phase):** `active_borrows_` currently
   tracks borrows by bare variable name (a flat list), so borrowing `p.left`
   and `p.right` on the same struct `p` are treated as conflicting even
   though the two fields do not overlap:

   ```kl
   struct pair { string left; string right; }
   pair p = ...;
   string& l = &p.left;    // should be independent of...
   string& r = &p.right;   // ...this, but the current checker conflates
                             // both to the single name "p"
   ```

   Moving from name-keyed to **place-keyed** tracking (variable + field-path,
   not just variable name) lets disjoint field borrows coexist. This is a
   natural extension once D4/D6 make "transferring a single field" a real
   operation — partial-move-style precision becomes necessary at that point,
   not merely nice to have. Dynamic indexing (`values[i]` vs. `values[j]`)
   stays **conservatively rejected** — the compiler cannot generally prove
   `i != j` — with a library-provided verified-split escape hatch
   (`values.splitat(index)`) rather than any attempt at static index-level
   proof, per the competing draft's §9.3 recommendation, which this ADR
   adopts as-is.

### D13 — Closures

Capture mode is **inferred**, not declared with a syntax block:

```
read-only use in the closure body   -> shared borrow (const T&)
mutation in the closure body        -> exclusive borrow (T&)
capture that must outlive the call  -> transfer (move-in)
```

This adopts the competing draft's inference table (§14) but rejects its
`borrow { }` / `move { }` explicit block syntax — requiring every closure to
be wrapped in a capture-mode block is added ceremony inconsistent with this
ADR's overall "infer by default" posture (D11). `move`-style forced capture
(e.g. handing a value to another thread) remains expressible by transferring
the value into a local **before** the closure literal, using D9's `move()`
where the inferred mode is ambiguous.

### D14 — Non-goals for v0 (explicit placeholders, not designed here)

The following are named so future work has a landing spot, but **no syntax or
semantics are decided in this ADR**:

- **Shared ownership** (`rc<T>`, `arc<T>`, `weak<T>`): not introduced in v0.
  `box<T>` (0029) remains the only owning-pointer type until a real use case
  justifies reference counting or weak references at the language level.
- **Raw pointers and `unsafe`** (`*T`, `*const T`, `unsafe { }` blocks):
  needed eventually for FFI and runtime implementation, but no syntax is
  specified here. When introduced, dereferencing a raw pointer or
  reconstructing a safe reference from one must be textually inside an
  `unsafe` region.
- **`send` / `sync` cross-thread capabilities**: the borrow checker does not
  attempt to prove concurrent correctness; cross-thread transfer capability
  markers are deferred.
- **Arbitrary self-referential safe structs, automatic cycle collection,
  pervasive typestate, static proof of lock ordering**: explicitly out of
  scope, matching the competing draft's §19 non-goals list, adopted as-is.

### D15 — Relationship to [0022](%5Bdeprecated%5D%200022-native-unique-ownership.md) and [0021](%5Bdeprecated%5D%200021-references-and-move.md)

| Topic | 0021 | 0022 | This ADR (0028) |
|-------|------|------|------------------|
| Native-only toolchain | — | D1 (implemented) | Carried forward unchanged (D0) |
| Assignment default for heap types | copy | copy (clone) | copy for value types (D5); transfer for resource types (D6) |
| By-value call parameter default | copy (v1); last-use move deferred | copy (clone) | **transfer** (D4) — this is the actual behaviour change from 0022 |
| Explicit transfer syntax | none (implicit move at sites) | `unique T` qualifier | bare `T` at call boundary (D4); `move()` stdlib fn for locals (D9) |
| Borrow syntax | `&T` / `&mut T` | `&T` / `&mut T` | `const T&` / `T&` (D3) |
| Borrow exclusivity/escape rules | D4 (original) | D5 (carried from 0021) | D10 (carried forward again, restated in full) |
| Lifetime annotations | none in v1 | none, no labels | none in v0 (D11) |
| Drop | — | D7 | D8 (carried forward) |

0022 is **superseded in full** by this document (D0 records what is kept
unchanged; everything else is replaced). 0021 was already superseded by 0022;
its substantive borrow-rule content is now fully absorbed here (D10) so no
reader needs to open either historical file to understand current behaviour.

## Consequences

### Positive

- Resolves the zero-cost violation in 0022 D2: the most common operation on a
  heap value (passing or transferring during a call) no longer forces a
  hidden allocation by default at every call site — only value-type
  *assignment* copies, and even that is explicit at the call boundary for
  transfer.
- Avoids the cross-branch dataflow analysis that a fully implicit,
  assignment-level move-by-default model would require — transfer is checked
  once per call site, independent of control flow.
- No new keywords or sigils; `T` / `const T&` / `T&` / `move()` all reuse
  existing tokens and an ordinary function, minimizing parser and lexer
  surface area.
- `mut` is removed as dead surface area — it never worked as a standalone
  token and its intent is now carried entirely by `const` position (D7).

### Risks and constraints

- **Resource-type consumed-state tracking** (D6) still requires the checker
  to mark a variable "transferred" and reject subsequent use — this is real
  new work, deliberately scoped down to a short, closed list of types rather
  than every heap type, to keep it tractable without general dataflow
  analysis.
- **D12's existing borrow-checker gap** (field/index assignment bypassing
  `check_referent_access`) should be fixed before or alongside this ADR's
  rollout, since D10's exclusivity rules depend on it being enforced
  uniformly.
- **Place-based borrow refinement** (D12 item 2) is scoped as later work, not
  blocking initial D1–D11 delivery.
- **Runtime clone API** for value types (D5) still needs `kl_clone_string`
  etc. — this repeats 0022 D9's requirement, just retargeted at assignment
  sites instead of "every heap-typed name-to-name binding."

### Acceptance criteria

- [ ] D2: bare scalar parameters never invalidate the caller's variable
- [ ] D4: bare value-type / resource-type call parameters transfer; caller's
      variable use after the call is a compile error
- [ ] D5: plain local assignment of a value type copies; caller's original
      remains valid
- [ ] D6: resource-type assignment and call-boundary transfer both invalidate
      the source
- [ ] D7: `mut` is rejected as a parse error in all positions; `const T&` /
      `T&` are the only spellings for borrow mutability
- [ ] D8: scope-exit `drop` runs on all control-flow exit paths for owned
      value-type and resource-type slots
- [ ] D9: `move()` is implementable purely as a stdlib function with no
      special-cased parser/checker/codegen support
- [ ] D10/D12 item 1: field assignment and index assignment go through
      exclusivity checking (fixes the existing gap)

## Dependencies

- [0002](0002-design-principles.md) — deterministic destruction; zero-cost
  pillar this ADR's D4/D5 restore compliance with.
- [0029](0029-value-representation-and-memory-layout.md) — type-tier
  classification (D1) that this ADR's category-based rules are defined over.
- [0005](0005-backend-architecture.md) — KIR seam for `Drop`.
- [0016](0016-typed-kir.md) — typed slots for drop and references.

## References

- Bootstrap checker: `bootstrap/compiler/frontend/checker/type_checker.cc`
  (`active_borrows_`, `check_referent_access` at lines 2110/2318, `is_mutable`
  at line 1816)
- Bootstrap parser: `bootstrap/compiler/frontend/parser/parse_type.cc` (`mut`
  text comparison at line 113), `parse_expr.cc` (line 251; `unary()` at 248;
  `AMP_AMP` / `logical_and()` at 117)
- Bootstrap lexer: `bootstrap/compiler/frontend/lexer/scanner.cc` (`AMP_AMP`
  token at line 154), `token.h` (line 31)
- Runtime: `bootstrap/runtime/kinglet_rt_agg.cc`, `kinglet_rt_internal.h`,
  `kinglet_rt_value.h`

## Open questions

1. **Runtime clone API naming** — `kl_clone_string` / `kl_clone_array` /
   `kl_clone_map` vs. a single generic `kl_clone(kl_h)` dispatch; carried over
   unresolved from 0022 D9.
2. **Resource-type opt-out spelling** — how a user-defined struct declares
   itself as a resource type (no clone) rather than a value type; this ADR
   assumes the distinction exists (D1/D6) but does not specify the
   declaration syntax, which belongs in [0029](0029-value-representation-and-memory-layout.md)
   or a follow-up.
3. **`own T` fallback** — if bare-`T`-transfers-at-call-boundary (D4) proves
   too implicit in practice, D3's rejected `own T` alternative is the
   documented fallback; revisit after real usage, not preemptively.
