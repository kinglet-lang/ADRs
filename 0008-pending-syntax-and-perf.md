# 0008 — Pending Syntax and Performance Items

- **Status**: draft
- **Date**: 2026-06-02

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

## Performance (from P5)

- [ ] NaN-boxing migration for Value representation
- [ ] Trivial relocatability (P1144 / P2786): move as memcpy

## Concurrency (from P4, deferred)

- [ ] Structured concurrency (P2504): `scope { spawn f(); spawn g(); }`
- [ ] spawn / channel / select
- [ ] Work-stealing scheduler
