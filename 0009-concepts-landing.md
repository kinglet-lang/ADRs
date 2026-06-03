# 0009 — Concepts Landing

- **Status**: accepted
- **Date**: 2026-06-03

## Context

RFC 0007 explored three approaches to replace Rust-style `trait`/`impl`/`self` with a system aligned to Kinglet's design principles (0002). After prototyping, Approach A (C++ Concepts style) was selected and further refined: the `where` clause was dropped in favor of an abbreviated syntax where concept names serve directly as parameter types.

## Decision

### Syntax

```kl
concept Printable<T> {
  string to_string(T value);
}

// Abbreviated style: concept name as parameter type → implicit generic
string greet(Printable item) {
  return to_string(item);
}

// Same concept name on multiple params → same concrete type
void swap(Sortable a, Sortable b) { ... }
```

### Semantics

- **Structural satisfaction**: any type with matching free-function signatures satisfies a concept; no explicit `impl` required.
- **Same-type rule**: multiple parameters typed with the same concept name bind to one concrete type.
- **UFCS**: a free function whose first parameter is a struct type is callable via dot syntax (`p.to_string()` resolves to `to_string(p)`).

### Scope (this RFC)

Implemented:
- `concept` and `where` as reserved keywords (lexer)
- `ConceptDecl` AST node with type parameters and method signatures
- Parser: `concept Name<T> { ret method(params); }`
- Checker: concept registry, UFCS method injection, two-pass registration
- Bootstrap and self-host aligned

Not implemented (deferred to future RFCs):
- Structural satisfaction checking at generic instantiation sites (requires monomorphization)
- Runtime polymorphism (`dyn`/`interface`)
- `where` clauses on struct/enum declarations
- Abbreviated concept parameter resolution in checker (currently concept-typed params are not checked for constraint satisfaction)

## Consequences

- `trait`, `impl`, `self` keywords removed from both bootstrap and self-host.
- `concept`, `where` reserved as keywords.
- `FunctionDecl` carries a `WhereClause[]` field (currently always empty; reserved for future use).
- UFCS enabled for any non-generic function whose first parameter is a known struct.
- Test coverage limited by a pre-existing self-host VM bug with struct literal parsing (does not affect correctness, only test expressiveness).
