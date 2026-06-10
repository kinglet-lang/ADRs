# 0015 — LLVM Backend Roadmap

- **Status**: accepted
- **Proposed**: 2026-06-09

## Context

[0014](0014-compilation-toolchain-architecture.md) defines the toolchain around KIR
and defers native backend technology to a follow-up decision. M3 (native toolchain
for `core/main.kl`) is blocked until a native lowering path exists.

[0005](0005-backend-architecture.md) already places LLVM beside the VM backend:

```
KIR ├→ VM backend → kbc
    └→ LLVM backend → native binary
```

The VM backend remains required for execution tests, Shadow parity, and user
programs that target `backend = "vm"`. LLVM is the intended path for:

1. **Toolchain performance** — native `kinglet` driver without interpreting
   `compiler.kbc` (~3–4s → compile+link budget).
2. **Default application output** — `kinglet build` with `default_backend = "native"`.
3. **Distribution** — single executable or Klos `kind=native` objects.

This document schedules LLVM work, records initial technical choices, and ties
phases to [0014](0014-compilation-toolchain-architecture.md) milestones.

## Decision

### D1 — LLVM is the native backend for the Ref compiler

1. The first LLVM implementation lives in the **C++ reference compiler**
   (bootstrap tree: `kinglet-lang/bootstrap`, mirrored in CI as `bootstrap/`).
2. Lowering is **KIR → LLVM IR → object file → link**, not AST → LLVM directly.
3. A self-host `codegen/llvm.kl` module is **out of scope** until the C++ backend
   passes toolchain and regression tests. Shadow parity for native output is not
   required for M3; execution tests compare behaviour against the VM.

Rationale: LLVM integration (ABI, runtime, intrinsics, linker flags) is easier to
iterate in C++. The Kinglet compiler sources remain the semantic reference; LLVM is
an output adapter.

### D2 — Runtime library (`libkinglet_rt`)

Native code assumes a small **C ABI runtime** linked into every executable:

| Area | Initial approach |
|------|------------------|
| Program entry | `kinglet_rt_main` wraps user `main`, handles exit code |
| Stack / locals | LLVM `alloca` in entry block; no GC |
| Strings | Heap-allocated buffer + length (match VM `HeapString` layout where practical) |
| Arrays, structs, enums, maps | Opaque handles or struct layouts generated per type; RC matching [0010](0010-vm-redesign.md) semantics |
| Errors (`try` / `?:`) | Lower to tagged union or errno-style slot; align with [0006](0006-error-handling-unification.md) |
| Panic / unreachable | `kinglet_rt_abort(msg)` — process exit with non-zero status |
| Native I/O (`io`, `fs`, `sys`) | Calls into `libkinglet_rt` or platform libc wrappers |

Runtime sources live under `runtime/` (bootstrap repo) or `backend/rt/` (self repo
mirror for VM parity tests). Version the runtime ABI; stamp includes `rt_version`.

### D3 — Build system integration

| Component | Responsibility |
|-----------|----------------|
| `kinglet build --backend native` | KIR → `.o` into Klos, link → `.kinglet/out/<target>` |
| Stamp | Includes LLVM version, target triple, `rt_version`, KIR schema hash |
| CI | Optional job `native-smoke` on macOS; full matrix follows VM green |
| Prove | VM bytecode parity unchanged; native path adds **exec** equivalence tests |

LLVM discovery: CMake/`gn` optional dep, or `LLVM_CONFIG` env (document in
`docs/native.md`). CI installs LLVM via package manager on macOS/Linux when the
native job is enabled.

### D4 — Phased delivery

Phases **L0–L4** map to [0014](0014-compilation-toolchain-architecture.md) **M1/M3**.
KIR landing (M1) is a hard prerequisite for L1+.

| Phase | Scope | Depends on | Exit criterion |
|-------|--------|------------|----------------|
| **L0** | Spike: KIR constant fn → LLVM → run | — | `just42.kl` prints correct value via native binary |
| **L1** | Integer ops, control flow, calls, `main` | M1 KIR + ir_builder split | `tests/exec/` subset green on native |
| **L2** | Strings, arrays, structs, enums | L1 + `libkinglet_rt` | Native smoke 11/11 green (see `tests/native/manifest.txt`) |
| **L3** | `try`/`?:`, error propagation, module link | L2 | `tests/regression/` subset green |
| **L4** | Natives (`io`/`fs`/`sys`), link `core/main.kl` | L3 + M0 Klos | `kinglet build` produces native toolchain binary |
| **L5** | Optimisation, cross-target, embed in driver | L4 | [0014](0014-compilation-toolchain-architecture.md) M3 complete |

**Schedule relative to 0014:**

```
M0 (Klos)     ─────────────────────────────►
M1 (KIR)      ─────────────►
L0–L1         ──────► (starts after KIR spike API frozen)
M2 (prove)              ──────►
L2–L3                         ──────────►
L4–L5 / M3                              ──────────►
M4 (incremental)                                    ───►
```

L0 may start in parallel with late M1 once a minimal KIR dump + C++ struct exists.

### D5 — Intentional limitations (initial releases)

1. **No self-host LLVM in Kinglet source** until L4 is stable.
2. **No GC** — deterministic destruction lowered to scope-exit calls or RC in RT
   (match language semantics; full RC in native RT may trail VM).
3. **Single target first** — `host` triple (e.g. `aarch64-apple-darwin` in CI).
4. **Debug info** — optional `DWARF` from KIR line tables; not required for L0–L2.
5. **WASM / other backends** — not scheduled; KIR seam allows them later.

Coverage may trail the VM while a phase is in flight; **numeric semantics must not**.
Missing opcodes or runtime helpers are temporary scope gaps, not an excuse to accept
behavioural drift on values the backend already claims to support.

### D6 — Numeric semantics parity (non-negotiable)

Native and VM must agree on every numeric value the native path can produce. The
native backend may omit features until they are implemented, but **must not**
silently narrow, truncate, or reinterpret numbers relative to the VM for supported
constructs. Concrete widths and aliases are fixed in **D7** below.

| Construct | VM reference | Native requirement |
|-----------|----------------|-------------------|
| `int` / `int64` literals and arithmetic | `int64_t`, exact integer ops | Full 64-bit range; no silent `i32` narrowing in KIR or lowering |
| `int32` literals and arithmetic | `int32_t`, exact integer ops | Same 32-bit range and wrap semantics once enabled in KIR |
| `float32` / `float64` literals and arithmetic | IEEE 754 `float` / `double` | Same width and rounding once native float is enabled |
| `int::bits(float)` / `float::from_bits` | Bit-exact reinterpret | Same bit patterns as VM |
| `main` exit code | `exit_code_from_value` rules | Match VM mapping (not ad-hoc truncation) |
| Heap vs integer wire format | Tagged handles separate from numbers | Negative integers and heap refs must remain distinguishable for all RT entry points |

**Gate:** before expanding the native smoke manifest for a numeric feature, add a
regression case that would fail on any precision or range mismatch. Native parity
debt is repaired in dedicated follow-up work; it is not deferred indefinitely.

**Parity status** (bootstrap; updated when native smoke manifest is green):

| Item | Status |
|------|--------|
| KIR `ConstInt` as full `int64` (low/high operands) | **Resolved** — `big_int` smoke |
| Heap vs integer wire format (`0xFFFE<<48` heap mark) | **Resolved** — negative ints no longer collide with heap tag |
| Inline enum wire + `kl_value_eq` | **Resolved** — `enum_no_payload`, `match_enum_simple` smoke |
| `main` exit code via `kl_exit_code` (0–255 clamp) | **Resolved** — matches VM `exit_code_from_value` |
| `float` / `double` KIR ops | **Open** — not lowered; keep float cases off native manifest until IEEE parity lands |
| Enum payload / guarded match | **Open** — simple no-payload `match` only; payload destructuring not yet implemented |
| `try` / `?:` error propagation | **Open** — not yet implemented |

### D7 — Fixed-width numeric types

Kinglet adopts **explicit fixed-width types** as the semantic contract; shorthand
names are **aliases with fixed meaning on every target**, including 32-bit hosts.

#### Width table and aliases

| Canonical type | Alias | Representation |
|----------------|-------|----------------|
| `int8` … `int64`, `uint8` … `uint64` | — | Two's-complement, width as named |
| `int64` | **`int`** | 64-bit signed integer on **all** platforms |
| `float32` | **`float`** | IEEE 754 binary32 |
| `float64` | **`double`** | IEEE 754 binary64 |
| `int8` | **`char`** (character literals) | 8-bit code unit |

`int` is **not** platform-dependent (contrast Go/C where `int` follows word size).
On a 32-bit CPU, `int` remains 64-bit; the backend lowers it as `i64`. This trades
some micro-architectural efficiency on 32-bit targets for **one semantics everywhere**:
the same source, manifest cases, and VM/native oracle hold on any triple.

#### Low-level access is not removed

Defaulting `int` to 64-bit does **not** erase systems programming capability.
Callers choose narrower types explicitly when they need register-sized arithmetic,
struct layout control, or foreign ABI widths:

```kl
int32 index = 0;
uint8 byte = 0xFF;
float32 sample = 0.0f32;
```

Narrow types are first-class in the type checker, KIR, and native lowering — not
secondary casts off a single machine `int`.

#### Conversion and mixed-width rules

1. **No silent narrowing** — `int64` → `int32` (and any wider → narrower) requires
   an explicit cast; the checker rejects implicit truncation.
2. **Mixed integer widths** — no automatic promotion across `int32` / `int64` in one
   expression; the programmer widens explicitly (or the checker inserts a documented
   rule later — default is **reject**).
3. **Mixed float widths** — `float32` with `float64` may promote to `float64` under
   one documented rule; never silently narrows to `float32`.
4. **Integer ↔ float** — requires explicit cast; no C-style usual arithmetic conversions.

Literals: integers within `int32` range default to `int32` type; wider literals are
`int64`. Suffixes (`42i32`, `42i64`, `1.0f32`, `1.0f64`) override. Assigning a
narrower literal to a wider variable is an explicit widening assignment (allowed).

#### Implementation schedule

Full fixed-width surface syntax and KIR opcodes (`ConstI32`, `IAdd64`, `FAdd32`, …)
trail the policy above. Until landed:

- VM continues to store alias `int` as `int64_t` ([0010](0010-vm-redesign.md)).
- Remaining parity gaps are listed in D6; integer and enum wire debt is cleared for the current native smoke manifest.
- Native manifest expands per width only after parity cases exist for that width.

#### Rationale

| Alternative | Why rejected |
|-------------|--------------|
| `int` = 32-bit on 32-bit hosts | Same program behaves differently by target; breaks VM/native parity gates |
| Only `int` / `float`, no widths | Hides precision in the implementation (current `ConstInt` / tagged `i64` debt) |
| `int` = pointer width | Couples numeric semantics to ABI; poor fit for a portable language VM |

### D8 — Testing strategy

| Layer | Test |
|-------|------|
| L0–L1 | `tests/exec/cases/*` selected via `native-smoke` manifest |
| L2+ | Expand manifest; compare stdout/stderr/exit code with VM |
| Toolchain | L4: native `kinglet --version` or equivalent smoke |
| Regression | Native does not replace bytecode goldens; both pipelines run in CI tier |

Add `tests/native/run_smoke.sh` and `NATIVE_MANIFEST` listing cases enabled per phase.

### D9 — Resolve [0014](0014-compilation-toolchain-architecture.md) open question #2

Native backend technology: **LLVM via C++ Ref compiler**, with `libkinglet_rt`.
Alternatives (direct machine code, emit-C only) are rejected for M3; emit-C may
remain an internal fallback for debugging (`--emit=c`).

## Open questions

1. **RC in native RT** — full refcount matching VM from aggregate RT landing, or copy
   semantics with eager free until optimisation phase?
2. **Link model** — always static link `libkinglet_rt`, or dynamic `.so` for dev?
3. **Monomorphization** — confirm all before KIR ([0005](0005-backend-architecture.md) OQ #1).
4. **Cross-repo layout** — LLVM backend only in bootstrap, or mirror `backend/llvm/`
   in self repo for VM/LLVM shared tests?
5. **LLVM version pin** — minimum LLVM 16 vs 18 for CI and dev docs.

## Consequences

- Fixed-width types (D7) require checker, KIR, VM, and LLVM changes; alias `int` =
  `int64` is policy-locked before the surface syntax fully lands.
- M3 timeline is bounded by L0–L4; KIR work (M1) becomes the gating priority.
- Bootstrap repo gains `ir/`, `codegen/llvm/`, `runtime/`; self repo documents and
  tests against released bootstrap binaries.
- CI cost increases when `native-smoke` lands; keep it a separate job from VM gate.
- [0005](0005-backend-architecture.md) build step 6 is superseded by this phased plan.
- Shadow `codegen/llvm.kl` remains a future optional project, not on the critical path.

## Dependencies

- [0005](0005-backend-architecture.md) — KIR form and VM backend contract.
- [0006](0006-error-handling-unification.md) — error lowering in native RT.
- [0010](0010-vm-redesign.md) — value representation reference for RT design.
- [0014](0014-compilation-toolchain-architecture.md) — Klos, `kinglet build`, M3/M4.

## Commit sequence (solo developer)

Granularity below is **one logical commit each**. Prefix `[sh]` = this repo
(`kinglet-self`); `[bs]` = bootstrap C++ compiler (`kinglet-lang/bootstrap`). Run
`tests/run_all.sh` at checkpoints unless noted.

### M0 — Klos and build wrapper

| # | Commit | Repo |
|---|--------|------|
| M0-1 | `[sh] chore: add .kinglet/ layout to .gitignore and document directories` | sh |
| M0-2 | `[sh] feat(build): add stamp helper (source hashes + compiler version)` | sh |
| M0-3 | `[sh] feat(build): write and read Klos objects with .meta sidecar` | sh |
| M0-4 | `[sh] feat(build): extend kinglet.toml with [build] and [build.compiler]` | sh |
| M0-5 | `[sh] feat(cli): add kinglet build calling bootstrap on cache miss` | sh |
| M0-6 | `[sh] refactor(tests): replace ensure_cli_kbc with ensure_build_stamp` | sh |
| M0-7 | `[sh] docs: update README and common.sh timing notes` | sh |

**Checkpoint**: `bash tests/run_all.sh` — no implicit self-host compile on warm stamp.

### M1 — KIR (blocks LLVM)

| # | Commit | Repo |
|---|--------|------|
| M1-1 | `[bs] feat(ir): add KIR data structures (module, bb, instr, types)` | bs |
| M1-2 | `[bs] refactor(compiler): extract bytecode_emitter from compiler.cc` | bs |
| M1-3 | `[bs] feat(ir): add ir_builder AST→KIR for literals and arithmetic` | bs |
| M1-4 | `[bs] feat(compiler): wire ir_builder → bytecode_emitter; pass codegen goldens` | bs |
| M1-5 | `[bs] feat(ir): extend ir_builder for control flow, calls, locals` | bs |
| M1-6 | `[bs] feat(cli): add --ir flag printing KIR text` | bs |
| M1-7 | `[sh] feat(ir): add ir.kl, ir_emit.kl, ir_print.kl mirroring C++ shape` | sh |
| M1-8 | `[sh] test(ir): add KIR golden suite for small programs` | sh |
| M1-9 | `[sh] docs: accept 0005; KIR form locked` | sh |

**Checkpoint**: `tests/codegen/run_golden.sh` + new KIR goldens green.

### M2 — Prove path

| # | Commit | Repo |
|---|--------|------|
| M2-1 | `[sh] feat(cli): add kinglet prove orchestrating shadow vs ref` | sh |
| M2-2 | `[sh] feat(cli): add kinglet debug emit-kbc` | sh |
| M2-3 | `[sh] ci: split fast (ref) and prove (shadow) workflow jobs` | sh |
| M2-4 | `[sh] refactor(tests): move selfhost/differential behind prove entry` | sh |

**Checkpoint**: local `kinglet build` does not run shadow; `kinglet prove` passes.

### L0 — LLVM spike

| # | Commit | Repo |
|---|--------|------|
| L0-1 | `[bs] build: optional LLVM via LLVM_CONFIG / gn args` | bs |
| L0-2 | `[bs] feat(rt): add libkinglet_rt with main shim and exit` | bs |
| L0-3 | `[bs] feat(llvm): add KirToLlvm lowering skeleton` | bs |
| L0-4 | `[bs] feat(llvm): lower KIR literal return; link just42 native` | bs |
| L0-5 | `[sh] test(native): add run_smoke.sh and manifest (just42)` | sh |
| L0-6 | `[sh] docs: add docs/native.md (LLVM install, smoke command)` | sh |

**Checkpoint**: `bash tests/native/run_smoke.sh` — `just42` native matches VM.

### L1 — Exec subset native

| # | Commit | Repo |
|---|--------|------|
| L1-1 | `[bs] feat(llvm): lower iadd/isub/imul and comparisons` | bs |
| L1-2 | `[bs] feat(llvm): lower branches, loops, function calls` | bs |
| L1-3 | `[bs] feat(cli): add --backend native and -o for object/exe` | bs |
| L1-4 | `[sh] test(native): enable exec manifest tier L1 cases` | sh |
| L1-5 | `[sh] feat(build): Klos kind=native from kinglet build --backend native` | sh |

**Checkpoint**: L1 manifest cases match VM stdout/exit.

### L2 — Aggregates and runtime

| # | Commit | Repo |
|---|--------|------|
| L2-1 | `[bs] feat(rt): string allocate, length, basic ops` | bs |
| L2-2 | `[bs] feat(llvm): lower string and array KIR ops` | bs |
| L2-3 | `[bs] feat(llvm): lower struct and enum KIR ops` | bs |
| L2-4 | `[sh] test(native): enable L2 exec/regression manifest cases` | sh |

**Checkpoint**: struct/array smoke green.

### L3 — Errors and modules

| # | Commit | Repo |
|---|--------|------|
| L3-1 | `[bs] feat(rt): try/?: lowering helpers (tagged result)` | bs |
| L3-2 | `[bs] feat(llvm): lower error propagation; complete match (payload, guards)` | bs |
| L3-3 | `[bs] feat(llvm): emit per-module objects; link step` | bs |
| L3-4 | `[sh] test(native): enable L3 regression manifest cases` | sh |

**Checkpoint**: error-handling cases native vs VM.

### L4–L5 — Toolchain native (M3)

| # | Commit | Repo |
|---|--------|------|
| L4-1 | `[bs] feat(rt): io/fs/sys native bindings for LLVM path` | bs |
| L4-2 | `[bs] feat(llvm): lower native intrinsics to rt calls` | bs |
| L4-3 | `[bs] feat(llvm): compile multi-module graph (imports)` | bs |
| L4-4 | `[bs] feat(cli): native build of core/main.kl entry` | bs |
| L4-5 | `[sh] feat(build): default native for toolchain; stamp includes rt_version` | sh |
| L4-6 | `[sh] test: smoke native kinglet driver (--version or equiv)` | sh |
| L5-1 | `[bs] feat(llvm): DWARF from KIR line tables (optional flag)` | bs |
| L5-2 | `[bs] feat(embed): optional link native compiler into kinglet binary` | bs |
| L5-3 | `[sh] docs: mark 0014 M3 complete; native dev workflow` | sh |

**Checkpoint**: `kinglet build && ./.kinglet/out/...` runs without `compiler.kbc`.
**Reached 2026-06-10** — L4–L5 landed: native `kinglet build` default, driver
smoke green, `-g` DWARF, optional embedded self-host compiler
(`kinglet selfhost`). See [docs/native.md](../docs/native.md).

### M4 — Incremental (after M3)

| # | Commit | Repo |
|---|--------|------|
| M4-1 | `[sh] feat(build): module dependency graph from imports` | sh |
| M4-2 | `[sh] feat(build): per-module stamp and Klos object` | sh |
| M4-3 | `[bs] feat(link): link multiple module objects into one exe` | bs |
| M4-4 | `[sh] feat(cli): kinglet objects gc` | sh |

### Suggested order for one developer

Work **strictly in table order** within each block. Across blocks:

```
M0-1 … M0-7  →  M1-1 … M1-9  →  L0-1 … L0-6  →  L1 … L2 … L3 … L4 … L5
                      ↓
                 M2-1 … M2-4  (any time after M0-7; before L4 if CI noise matters)
```

M2 can slot in during M1 or early L1 when prove CI separation becomes useful.
Do not start L2 until L1 checkpoint is green.

## References

- LLVM documentation: IR builder, ORC JIT (spike only), target triples.
- `tests/exec/` — primary behavioural oracle for L1–L3.
