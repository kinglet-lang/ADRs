# 0008 — KBC Bytecode Format Evolution

- **Status**: deprecated (implemented; bytecode path retired by 0022)
- **Proposed**: 2026-06-02
- **Completed**: 2026-06-03

**Current replacement**: [0022 — Native-Only Toolchain and Unique Ownership](0022-native-unique-ownership.md). KBC remains historical format documentation only; active compiler output is native LLVM.

## Context

The self-host compiler (cli.kbc, 349 KB) is larger than its source (220 KB). Debug info (line/col per instruction) accounts for 62% of bytecode but is unused at runtime. The constant pool has no deduplication. The self-host compiler cannot yet produce .kbc files.

### Format versions

**Version 1 (original)**

```
Header:       16 bytes (magic, version=1, flags, reserved)
Constants:    u32 count + (u8 type + value)[]
Instructions: u32 count + (u8 op + i32 operand + i32 line + i32 col)[] = 13 bytes/insn
Functions:    u32 count + (str name + u64 entry + i32 param_count)[]
Structs:      u32 count + (str name + u32 field_count + str[] field_names)[]
Enums:        u32 count + (str name + u32 variant_count + (str + i32)[])[]
```

**Version 2 (current)**

Same structure, except the instructions section uses Sleb128 for operand and debug fields:
```
Instructions: u32 count + (u8 op + sleb128 operand [+ sleb128 line + sleb128 col])[]
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

### P2 — Self-hosted .kbc serialization (implemented)

**Goal**: Self-host compiler produces .kbc files via `--save-bytecode out.kbc source.kl` that the C++ VM can execute directly.

**Files**:
- `compiler/bytecode.kl` — `serialize_chunk(Chunk, string path, bool strip_debug)` + helpers (`write_u8/u32/i32/i64/f64/str`)
- `cli/main.kl` — `--save-bytecode` flag routing + `run_save_bytecode()` function
- `src/vm/vm.cc` — `fs::__write(path, string)` native function for binary file output
- `src/vm/chunk.h` / `chunk.cc` — `FloatToBits` opcode for `int::bits(float)` used in `write_f64`

**Implementation**:
- String-as-byte-buffer: accumulate binary data, write once via `fs::__write`
- Serialize in order: header → constants → instructions → functions → structs → enums
- `write_f64` uses `int::bits()` (FloatToBits opcode) to reinterpret float bits as int64

**Verification**: `kinglet --run cli.kbc --save-bytecode v2.kbc cli/main.kl` produces byte-for-byte identical output to `kinglet --save-bytecode v1.kbc cli/main.kl`. cli.kbc size: 177 KB.

Depends on: P0, P1.

### P3 — Variable-length operand encoding (implemented)

**Goal**: Replace fixed i32 (4 bytes) with signed LEB128. Most operands are small indices (< 128), encoding to 1 byte.

**Files**:
- `src/vm/chunk.cc` — `encode_sleb128`, `decode_sleb128`, `write_sleb128`, `read_sleb128`; `serialize()` uses Sleb128 for operand and debug fields; `kKbcVersion` bumped to 2; `deserialize()` selects decode path by version field
- `compiler/bytecode.kl` — `write_sleb128()` helper; `serialize_chunk()` uses it for operand and debug fields, writes version=2

**Backward compatibility**: `deserialize()` accepts both version 1 (fixed i32) and version 2 (Sleb128). Old v1 .kbc files load correctly.

**Result**: cli.kbc with debug info 366 KB (v1) → 177 KB (v2), **−52%**. Combined with strip-debug: 177 KB → ~85 KB.

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
| P2 | vm.cc (fs::__write, FloatToBits), chunk.h/cc | bytecode.kl, cli/main.kl |
| P3 | chunk.cc | bytecode.kl |

## Size summary

| Variant | Size | vs original |
|---------|------|-------------|
| v1, with debug | 349 KB | baseline |
| v1, strip debug | 176 KB | −50% |
| v2, with debug | 177 KB | −49% |
| v2, strip debug | ~85 KB | ~−76% |

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
- P3 requires version negotiation — old VMs cannot read v2 .kbc files; v2 VMs read both.


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

## Amendments

### 2026-06-20 — Bytecode execution retired by native-only toolchain ([0022](0022-native-unique-ownership.md))

This ADR remains the historical record for the completed KBC format work, but KBC
is no longer an active compiler output path. Bootstrap commit `ba20344`
(`refactor(vm): remove VM interpreter and kbc execution paths`, 2026-06-20)
removed the VM interpreter, KBC execution paths, `--save-bytecode`, and `kinglet-vm`.

A later cleanup removed the remaining dead bytecode container from bootstrap in
commit `54df929` (`refactor(build): delete dead bytecode container`, 2026-07-10).
New toolchain work should follow [0022](0022-native-unique-ownership.md) and the
native LLVM path, not the KBC format.
