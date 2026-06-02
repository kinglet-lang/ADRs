# 0008 — KBC Bytecode Format Evolution

- **Status**: partial (P0 + P1 implemented; P2 + P3 deferred)
- **Date**: 2026-06-02

## Context

The self-host compiler (cli.kbc, 349 KB) is larger than its source (220 KB). Debug info (line/col per instruction) accounts for 62% of bytecode but is unused at runtime. The constant pool has no deduplication. The self-host compiler cannot yet produce .kbc files.

### Current kbc format (version 1)

```
Header:       16 bytes (magic, version, flags, reserved)
Constants:    u32 count + (u8 type + value)[]
Instructions: u32 count + (u8 op + i32 operand + i32 line + i32 col)[] = 13 bytes/insn
Functions:    u32 count + (str name + u64 entry + i32 param_count)[]
Structs:      u32 count + (str name + u32 field_count + str[] field_names)[]
Enums:        u32 count + (str name + u32 variant_count + (str + i32)[])[]
```

### Key data

| Metric | Value |
|--------|-------|
| Source total | 220 KB |
| cli.kbc (before) | 349 KB |
| Instructions | 22,578 |
| Instruction segment (with debug) | 293 KB (82% of total) |
| Instruction segment (without debug) | 110 KB |
| Debug-only savings | 176 KB (50.5%) |

### Verified facts

- VM never reads line/col (no references in vm.cc).
- `kFlagHasDebugInfo` was reserved in header but serialize always wrote 1, deserialize never checked.
- Constant pool `add_constant` had no dedup (bs and sh sides consistent).
- 92 OpCodes, operand is i32 (oversized for most instructions).

## Decision

### P0 — Strip debug mode (implemented)

**Files**: `src/vm/chunk.cc`, `src/kinglet/main.cc`

`serialize()` accepts `bool strip_debug`. When true: header flags = 0, instructions written as 5 bytes/insn (opcode + operand only). `deserialize()` reads flags, fills line=0/col=0 when `kFlagHasDebugInfo` absent.

`main.cc` gains `--strip-debug` flag, used with `--save-bytecode`.

Result: cli.kbc 349 KB → 176 KB (−50%).

### P1 — Constant pool deduplication (implemented)

**Files**: `src/vm/chunk.cc` (bs), `compiler/bytecode.kl` (sh)

`add_constant()` linear scan O(n) before insert. Dedup by type: Int/Double by value, Bool/Null by kind, String by content, Function by index, Enum by (type_index, variant_index, payload). Sh side inlines dedup in `write_constant()` (cannot call `add_constant` due to same-file function resolution limitation).

Result: cli.kbc 349 KB → 330 KB (−5.4%). Combined P0+P1: 349 KB → 150 KB (−57%).

### P2 — Self-hosted .kbc serialization (deferred)

**Goal**: Self-host compiler produces .kbc files via `--save-bytecode out.kbc source.kl` that the C++ VM can execute directly.

**Files**:
- `compiler/bytecode.kl` — new `serialize_chunk(Chunk, string path)` function
- `cli/main.kl` — new `--save-bytecode` flag routing + `run_save_bytecode()` function
- C++ VM native function extension — `fs::__write(path, string)` for binary file output

**Implementation**:
- Helper functions in bytecode.kl: `write_u8`, `write_u32`, `write_i32`, `write_i64`, `write_f64`, `write_str`
- Serialize in order: header → constants → instructions → functions → structs → enums
- Use string to accumulate binary data, write once at end (string-as-byte-buffer approach; cli.kbc stripped is ~170 KB, feasible)
- Handle strip-debug mode: check flag, conditionally write line/col

**Verification**: `kinglet --run cli.kbc --save-bytecode test.kbc test.kl && kinglet --run test.kbc` produces same output as `kinglet test.kl`.

Depends on: P0, P1.

### P3 — Variable-length operand encoding (deferred)

**Goal**: Replace fixed i32 (4 bytes) with LEB128-style variable-length encoding. Most operands are small indices (< 256), encoding to 1–2 bytes.

**Files**:
- `src/vm/chunk.cc` — serialize/deserialize with varint encode/decode
- Bump `kKbcVersion` to 2; VM selects decode path based on version header
- `compiler/bytecode.kl` — self-host compiler syncs implementation

**Verification**: Version 1 .kbc files still load (backward compatible). Version 2 .kbc files decode correctly. Expected additional reduction 5–10%.

Depends on: P2.

## Phase dependencies

```
P0 (strip debug) ──→ P2 (self-hosted serialization)
P1 (const dedup) ──→ P2
                      P2 ──→ P3 (varint encoding)
```

## Implementation repos

| Phase | bs repo | sh repo |
|-------|---------|---------|
| P0 | chunk.cc, main.cc | — |
| P1 | chunk.cc | bytecode.kl |
| P2 | vm native function extension | bytecode.kl, cli/main.kl |
| P3 | chunk.cc | bytecode.kl |

## Verification checklist

For each phase:
1. `kinglet --save-bytecode test.kbc test.kl` compile test file
2. `kinglet --run test.kbc` run and compare with `kinglet test.kl` output
3. `bash tests/codegen/run_golden.sh` regression suite
4. Build cli.kbc → `kinglet --run cli.kbc --bytecode some_file.kl` verify self-hosting
5. Compare kbc file sizes

## Consequences

- P0 and P1 are backward-compatible (new flag bit, old VMs ignore it gracefully).
- P2 is the key enabler for self-hosted toolchain independence.
- P3 requires version negotiation — old VMs cannot read v2 .kbc files.
