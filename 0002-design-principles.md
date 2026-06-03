# 0002 — Design Principles

- **Status**: implemented
- **Proposed**: 2026-05-21
- **Completed**: 2026-05-25

## Context

Kinglet's original vision was "C++ after completing worthwhile standard committee proposals." During trait system implementation, syntax and semantics drifted toward Rust (`trait` / `impl` / `self` keywords), departing from that vision.

Root cause: no written design principles to constrain feature direction.

## Decision

Kinglet is defined by three pillars. Any new feature must be consistent with all three, or justify an exception with strong reasoning.

### 1. Full Value Semantics

Copy is the default. `int a = b;` leaves `a` and `b` independent — no ownership tracking, no borrow checker, no lifetime annotations. Reference semantics are opt-in, not the foundation.

### 2. Deterministic Destruction

Variables are destroyed at a known point: end of scope, in reverse construction order. RAII is the resource management model. No GC delays, no finalizer semantics, no hidden deferred cleanup.

### 3. Zero-Cost Abstraction Without Ownership

Compile-time polymorphism via monomorphization (like C++ templates). Runtime polymorphism via explicit opt-in (like `virtual`). Zero-cost abstraction is achieved without ownership tracking or borrow checking.

### Key Differences from Rust

| Dimension | Rust | Kinglet |
|-----------|------|---------|
| Parameter passing | move by default | copy by default |
| Ownership | core mechanism | absent |
| Borrow checking | compile-time enforced | none |
| Generic constraints | trait bounds | concept-like constraints |
| Runtime polymorphism | `dyn Trait` | explicit opt-in (TBD) |
| Destruction | Drop trait + ownership | deterministic scope exit |
| Error handling | `Result<T, E>` + `?` | `try` + `??` (already shipped) |

## Consequences

- **`self` keyword**: contradicts value-semantics-as-default. **Remove.**
- **Trait syntax**: current `trait`/`impl` mimics Rust. **Redesign** (see 0007).
- **`fn` keyword**: Kinglet uses C-style `int main()` declarations. **Do not introduce `fn`; keep C-style declarations.**
- Every future feature proposal must answer: does this serve all three pillars?
  If not, the feature needs exceptional justification.

## Implementation Status

| Consequence | Status |
|---|---|
| `self` keyword removed | Done (0009) |
| `trait`/`impl` replaced with `concept` + structural satisfaction | Done (0009) |
| `fn` keyword not introduced; C-style declarations retained | No action needed |
