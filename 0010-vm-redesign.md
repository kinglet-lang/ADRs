# 0010 — VM Redesign and Embedded Self-Host Binary

- **Status**: accepted
- **Proposed**: 2026-06-03

## Context

The current C++ VM uses a `Value` type that embeds `std::vector<Value>` for composite values (struct fields, enum payloads, arrays). Every stack operation (`push`, `store`, `load`) performs a recursive deep copy. This causes:

1. **Exponential blowup on nested values** — parsing struct literals through the self-host compiler (cli.kbc) hangs indefinitely because AST nodes (which are deeply nested enum/struct trees) trigger O(tree_depth) copies on every stack push.
2. **40x slower than Python in IO-heavy debug scenarios** — profiled in earlier benchmarks. Python uses reference-counted pointers; its stack operations are O(1).
3. **Blocks self-host usability** — the self-host compiler cannot process real programs that use struct literals, making it a demo rather than a tool.

Additionally, the self-host compiler (`cli.kbc`) currently requires manual invocation via `kinglet --run cli.kbc <args>`. There is no standalone binary that users can invoke directly.

## Decision

### Part 1 — VM Value Representation Redesign

Replace the deep-copy value model with reference-counted heap objects:

```
Before: Value { tag, int_val, float_val, string_val, vector<Value> fields }
After:  Value { tag, union { int64, double, Rc<HeapObj>* } }
```

Where `HeapObj` covers strings, arrays, structs, enums, closures, and maps. Stack operations become O(1) pointer copies + refcount increment.

Key properties:
- **Cycle-free by construction** — Kinglet has no reference types in the language; all cycles would require explicit `&` (future RFC). Simple RC suffices; no tracing GC needed.
- **Copy-on-write for value semantics** — `int a = b;` at the language level shares the underlying HeapObj until mutation, then clones. Preserves the language's copy-by-default guarantee without physical copying on every assignment.
- **Thread-unsafe RC is acceptable** — the VM is single-threaded. Atomic refcounts are unnecessary overhead.

### Part 2 — Embedded cli.kbc Binary

Bundle `cli.kbc` as a compiled-in resource within the `kinglet` binary:

```
kinglet <file.kl>              # compile and run (current behavior)
kinglet --self-host <args>     # invoke embedded self-host compiler
```

Or produce a dedicated `kinglet-sh` binary that auto-loads the embedded bytecode. The embedding mechanism is a build step that converts `cli.kbc` to a C byte array (`xxd -i` or equivalent).

Prerequisite: Part 1 must land first, since the embedded compiler needs to handle struct literals without hanging.

## Open Questions

1. **COW granularity** — clone on any field write, or clone the entire struct? Per-field COW adds complexity; whole-struct COW is simpler but copies more.
2. **Small-value optimization** — should short strings (≤ 15 bytes) be stored inline in Value to avoid heap allocation? Adds complexity but benefits the hot path (identifiers, keywords).
3. **Interning** — should string constants be interned in a global pool? Reduces allocation for repeated literals but adds lookup cost.
4. **Embedding format** — raw .kbc bytes, or a stripped/compressed variant?

## Consequences

- VM stack operations become O(1) regardless of value complexity.
- Self-host compiler can process arbitrary .kl programs (unblocks struct literal parsing).
- Memory usage may increase slightly due to HeapObj headers (16 bytes per object for tag + refcount).
- `Value` size shrinks from ~72 bytes (current, with inline vector) to ~16 bytes (tag + pointer/immediate).
- Bytecode format unchanged — this is a VM-internal change, not a .kbc format change.
- Benchmark target: self-host compiler compiling itself should complete in < 10s (current: hangs on struct literals; theoretical floor with RC: ~2-5s based on Python-equivalent interpretation speed).

## Dependencies

- None (standalone infrastructure change).
- Unblocks: embedded binary (Part 2), REPL, constexpr evaluation, sandbox execution.
