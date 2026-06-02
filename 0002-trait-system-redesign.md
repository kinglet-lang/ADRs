# 0002 — Trait System Redesign

- **Status**: draft
- **Date**: 2026-06-02

## Context

The current trait implementation (6da693d) uses Rust-style syntax:

```kl
trait Printable {
  fn to_string(self) -> string;
}

impl Printable for int {
  fn to_string(self) -> string { ... }
}
```

Problems:
1. `self` contradicts value-semantics-as-default (see 0001).
2. `trait`/`impl` separation doesn't match C++ concepts' "constraint-as-interface" model.
3. Syntax clashes with Kinglet's C-style function declarations (`int main()`).

## Decision

*TBD — three candidate approaches are on the table. A direction must be chosen before implementation begins.*

### Approach A — C++ Concepts Style

```kl
concept Printable(T) {
  string to_string(T value);
}

void print<T>(T value) where Printable(T) {
  io::out(to_string(value));
}
```

- Constraints declared in a `concept` block, not bound to a type.
- Generic functions declare constraints via `where` clauses.
- Satisfaction is structural (matching signatures suffice, no explicit `impl`).
- Runtime polymorphism: separate `interface` mechanism (TBD).

### Approach B — Explicit-First-Parameter Traits

```kl
trait Printable {
  string to_string(T);
}

impl Printable for int {
  string to_string(int value) { ... }
}
```

- Drop `self`; first parameter is explicit.
- Keep `trait`/`impl` structure but with different semantics.
- Closest to current implementation — lower migration cost.

### Approach C — Hybrid

`concept` for pure compile-time constraints, `trait` retained for runtime polymorphism scenarios.

## Open Questions

1. **concept syntax**: block with signatures, or something more compact?
2. **Structural vs nominal satisfaction**: C++ concepts are structural (any type with matching signatures satisfies); Rust traits are nominal (requires explicit `impl`). Which for Kinglet?
3. **Runtime polymorphism syntax**: `virtual` / `interface` / `dyn` / something else?
4. **Interaction with enums**: can concepts constrain enum variants?
5. **Monomorphization strategy**: concrete types at compile time, or link-time?

## Consequences

- Affects: `lexer/`, `parser/ast.kl`, `checker/`, `compiler/`.
- Existing trait test cases will need rewriting.
- Depends on `self` removal being completed first.

## Dependencies

- 0001 (design principles) — accepted.
- `self` keyword removal — must be done before this.
