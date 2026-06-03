# 0003 — Standard Library Roadmap

- **Status**: deferred
- **Proposed**: 2026-05-29

## Context

After self-hosting (round-trip verified), the next major milestone is a standard library. Currently all I/O, filesystem, and system access is hardcoded as native intrinsics in the VM. The self-host compiler has no stdlib — every program starts from scratch.

## Decision

### Phase A — Migrate native bindings to stdlib modules

Delete hardcoded `io`/`fs`/`sys` dispatch. Create thin Kinglet wrappers:
- `std/native/io.kl` — exposes native opcodes as plain functions
- `std/native/fs.kl` — file read/write
- `std/native/sys.kl` — args, env

User-facing API:
- `std/io/mod.kl` — `println`, `readln`, etc.

### Phase B — Core stdlib modules

- `std/collections/array_list.kl` — generic dynamic array (wraps built-in)
- `std/collections/hash_map.kl` — key-value store
- `std/result.kl` — `Result<T, E>` enum + combinators
- `std/option.kl` — `Option<T>` enum
- `std/string/mod.kl` — builder, split, join
- `std/math/mod.kl` — min, max, abs, pow
- `std/iter/mod.kl` — Iterator + map/filter/reduce adaptors (depends on trait system redesign, see 0007)

### Phase C — Self-hosted VM (long-term)

Write the VM in Kinglet itself:
- `vm/value.kl` — tagged value representation
- `vm/chunk.kl` — bytecode loader
- `vm/vm.kl` — stack machine interpreter
- `vm/native.kl` — native function bindings

Once the VM runs on itself, the C++ bootstrap compiler is retired.

## Dependencies

- Trait system (0007) needed for `Iterator` and generic combinators.
- KBC P2 (0008) needed before self-hosted VM can load .kbc files.
