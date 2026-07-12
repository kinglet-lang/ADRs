# 0010 — VM Redesign and Embedded Self-Host Binary

- **Status**: deprecated (Part 1 implemented; active VM path retired by [0022](0022-native-unique-ownership.md)); Part 2 deferred — superseded by [0014](0014-compilation-toolchain-architecture.md) D6
- **Proposed**: 2026-06-03
- **Completed**: 2026-06-08 (Part 1 — `backend/vm` RC/COW value model)

**Current replacement**: [0022 — Native-Only Toolchain and Unique Ownership](0022-native-unique-ownership.md) for the retired VM execution path; [0014 — Compilation Toolchain Architecture](0014-compilation-toolchain-architecture.md) D6 for the deferred embedded self-host binary direction.

## Context

The current C++ VM uses a `Value` type that embeds `std::vector<Value>` for composite values (struct fields, enum payloads, arrays). Every stack operation (`push`, `store`, `load`) performs a recursive deep copy. This causes:

1. **Exponential blowup on nested values** — parsing struct literals through the self-host compiler (cli.kbc) hangs indefinitely because AST nodes (which are deeply nested enum/struct trees) trigger O(tree_depth) copies on every stack push.
2. **40x slower than Python in IO-heavy debug scenarios** — profiled in earlier benchmarks. Python uses reference-counted pointers; its stack operations are O(1).
3. **Blocks self-host usability** — the self-host compiler cannot process real programs that use struct literals, making it a demo rather than a tool.

Additionally, the self-host compiler (`cli.kbc`) currently requires manual invocation via `kinglet --run cli.kbc <args>`. There is no standalone binary that users can invoke directly.

## Decision

### Part 1 — VM Value Representation Redesign

- **Status**: implemented
- **Files**: `backend/vm/value.h`, `value.cc`, `cow.h`, `cow.cc`, `vm.cc`

> **Update (2026-06-12)**: `backend/vm` (the standalone C++ VM copy) has been
> removed. Self-host suites now use the bootstrap Ref compiler as the VM host
> (`--run`); the RC/COW value model is maintained in the bootstrap tree
> (`src/vm`). The self-host backend is to be reimplemented in Kinglet itself.

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

- **Status**: deferred — superseded by [0014](0014-compilation-toolchain-architecture.md) D6
- **Reason**: Toolchain embedding targets a native compiler artefact or Klos object,
  not `compiler.kbc`. Optional kbc embedding may still apply to **user programs**;
  see 0014.

The original proposal is retained below for reference.

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

### Part 1 (landed)

- VM stack operations are O(1) pointer copies for heap-backed values.
- Self-host compiler can process programs with struct literals without deep-copy blowup.
- Memory usage may increase slightly due to HeapObj headers (tag + refcount).
- `Value` holds immediates or `RcPtr<HeapObj>` instead of inline `vector<Value>`.
- Bytecode format unchanged — VM-internal only ([0008](%5Bdeprecated%5D%200008-kbc-format-evolution.md)).
- Self-host compile time for `core/main.kl` is ~3–4s on the VM (2026-06-09); further
  toolchain speed work is in [0014](0014-compilation-toolchain-architecture.md).

### Part 2 (deferred)

- Embedding scope moved to [0014](0014-compilation-toolchain-architecture.md) D6
  (native artefact or Klos; optional user-program kbc embed).
- `kinglet --self-host` / embedded `compiler.kbc` driver is not implemented.

## Dependencies

- None for Part 1 (standalone VM change).
- Part 1 unblocks: Shadow compiler usability, REPL, constexpr evaluation, sandbox execution.
- Part 2 follow-up: [0014](0014-compilation-toolchain-architecture.md) M0–M3 (Klos, native toolchain).

### 2026-06-20 — VM path retired by native-only toolchain ([0022](0022-native-unique-ownership.md))

Part 1 was implemented on 2026-06-08 and remains historical context for the RC/COW
VM value model. The active VM/KBC execution path was retired by bootstrap commit
`ba20344` (`refactor(vm): remove VM interpreter and kbc execution paths`,
2026-06-20), which deleted `kinglet-vm`, VM execution, `--save-bytecode`, and the
REPL. Current language and toolchain design targets the native LLVM backend per
[0022](0022-native-unique-ownership.md).
