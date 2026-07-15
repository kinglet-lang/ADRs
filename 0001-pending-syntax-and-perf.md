# 0001 — Pending Syntax and Performance Items

- **Status**: draft
- **Proposed**: 2026-05-17

## Context

Miscellaneous features from the original bs TODO that don't warrant individual decision entries yet. Grouped here for tracking. Promote to a standalone entry when design work begins.

## Syntax (from P1)

- [ ] `once` lazy init block: memoize first evaluation, zero-cost on subsequent calls
- [ ] `retry N { ... }` loop: built-in retry semantics with optional delay
- [ ] Inline tests: `test "name" { ... }` blocks, compiled out in release, run with `kinglet test`
- [ ] `scope` resource management: auto-call `.close()` on scope exit without RAII classes

## Types and Patterns (from P2)

- [ ] Error propagation `?` postfix operator (requires Result/Optional type)
- [ ] Zero-overhead optional: `int? x = null;` with niche optimization
- [ ] `[[nodiscard]]`: warn on unused return values
- [ ] Struct patterns in match and structured binding
- [x] Reference types `&T` / `&mut T` and ownership model — **promoted to [0022](%5Bdeprecated%5D%200022-native-unique-ownership.md)** (native unique ownership; no move semantics)
- [x] Data types, literals, struct ABI, wide integers — **promoted to [0023](%5Bdeprecated%5D%200023-data-types-and-abi.md)** (P0 self-host gate)

## Performance (from P5)

- [ ] NaN-boxing migration for Value representation
- [x] Trivial relocatability (P1144 / P2786): memcpy transfer for `unique` — **promoted to [0022](%5Bdeprecated%5D%200022-native-unique-ownership.md)** (internal analysis; no user `relocatable` keyword)

## Amendments

### 2026-06-21 - Native unique ownership ([0022](%5Bdeprecated%5D%200022-native-unique-ownership.md))

Reference types, borrows, and heap ownership are specified in
[0022](%5Bdeprecated%5D%200022-native-unique-ownership.md) (supersedes draft
[0021](%5Bdeprecated%5D%200021-references-and-move.md)). No `move()` keyword; transfer uses `unique T`.

Original §Syntax / §Performance lists above are preserved for historical context.

### 2026-07-14 - Optional types and `?` operator implemented ([0028](0028-ownership-and-value-transfer.md), [0029](0029-value-representation-and-memory-layout.md))

Two items from §Types and Patterns are now implemented via later ADRs:

- [x] Error propagation `?` postfix operator (requires Result/Optional type)
      — **implemented**: `field?` safe access (ADR 0029 D14, PR #113/#114) and
      `?:` null-coalesce operator provide Optional-based error propagation.
- [x] Zero-overhead optional: `int? x = null;` with niche optimization
      — **implemented**: `TypeKind::Optional`, null sentinel `0xFFFB<<48`,
      inline tagged representation for scalars (ADR 0029 D14, PR #109–#112).

Original §Types and Patterns list above is preserved for historical context.

## Concurrency (from P4, deferred)

- [ ] Structured concurrency (P2504): `scope { spawn f(); spawn g(); }`
- [ ] spawn / channel / select
- [ ] Work-stealing scheduler
