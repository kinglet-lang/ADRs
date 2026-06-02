# 0005 — KBC Bytecode Format Evolution

- **Status**: partial (P0 + P1 implemented; P2 + P3 deferred)
- **Date**: 2026-06-02

## Context

The self-host compiler (cli.kbc, 349 KB) is larger than its source (220 KB). Debug info (line/col per instruction) accounts for 62% of bytecode but is unused at runtime. The constant pool has no deduplication. The self-host compiler cannot yet produce .kbc files.

## Decision

Four-phase bytecode format optimization:

### P0 — Strip debug mode (implemented)

Add `--strip-debug` flag. When set, serialize omits line/col from instructions (5 bytes/insn instead of 13). Header flags indicate presence/absence of debug info. Deserializer fills 0 when absent.

Result: cli.kbc 349 KB → 176 KB (−50%).

### P1 — Constant pool deduplication (implemented)

`add_constant()` checks for existing identical values before inserting. Linear scan O(n), acceptable for typical pool sizes (<1000 entries). Dedup by type: Int/Double by value, Bool/Null by kind, String by content, Function by index, Enum by (type_index, variant_index, payload).

Result: cli.kbc 349 KB → 330 KB (−5.4%). Combined P0+P1: 349 KB → 150 KB (−57%).

### P2 — Self-hosted .kbc serialization (deferred)

Self-host compiler gains ability to produce .kbc files via `--save-bytecode`. Requires binary file I/O support (`fs::__write`). Implementation in `compiler/bytecode.kl` (serialize_chunk) + `cli/main.kl` (flag routing).

Depends on: P0, P1.

### P3 — Variable-length operand encoding (deferred)

Most operands are small indices (< 256). Replace fixed i32 (4 bytes) with LEB128-style variable-length encoding. Bump kbc version to 2; VM decodes based on version header.

Depends on: P2.

## Current kbc format (version 1)

```
Header:       16 bytes (magic, version, flags, reserved)
Constants:    u32 count + (u8 type + value)[]
Instructions: u32 count + (u8 op + i32 operand + i32 line + i32 col)[]
Functions:    u32 count + (str name + u64 entry + i32 param_count)[]
Structs:      u32 count + (str name + u32 field_count + str[] field_names)[]
Enums:        u32 count + (str name + u32 variant_count + (str + i32)[])[]
```

## Consequences

- P0 and P1 are backward-compatible (new flag bit, old VMs ignore it gracefully).
- P2 is the key enabler for self-hosted toolchain independence.
- P3 requires version negotiation — old VMs cannot read v2 .kbc files.
