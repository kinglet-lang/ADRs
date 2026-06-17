# 0019 — Self-Host LLVM Backend

- **Status**: accepted — Route B; S0 + S1 integer/control-flow lowering delivered 2026-06-18
- **Proposed**: 2026-06-17

## Context

[0015](0015-llvm-backend-roadmap.md) delivered the native backend in the **C++ Ref
compiler**: KIR → LLVM IR → per-module `.o` → link with `libkinglet_rt`. Phases L0–L5
are complete; `kinglet build` can produce a native toolchain binary without interpreting
`compiler.kbc` on the hot path.

The self-hosted compiler still emits VM bytecode only. For a fully Kinglet-implemented
toolchain ([0014](0014-compilation-toolchain-architecture.md) Shadow track), the Shadow
compiler must lower KIR to native objects and link them without depending on the C++
`kir_to_llvm.cc` at runtime.

[0015](0015-llvm-backend-roadmap.md) explicitly deferred `codegen/llvm.kl`; this ADR
schedules that work and defines how it relates to the Ref backend.

## Decision

### D1 — Pipeline shape

Self-host native compilation follows the same seam as Ref ([0005](0005-backend-architecture.md)):

```
.kl → checker → KIR emit (ir_emit.kl) → codegen/llvm.kl → .o → lld → executable
```

- **No AST → LLVM direct lowering.**
- **KIR semantics** remain authoritative; Ref `ir_builder` and Shadow `ir_emit.kl` must
  agree (prove / golden tests).
- **`libkinglet_rt`** is linked unchanged; Shadow calls the same C ABI as Ref.

### D2 — LLVM integration strategy (phased)

| Phase | Scope | Exit criterion |
|-------|--------|----------------|
| **S0** | FFI spike: one KIR function → `.o` → run | `just42.kl` native via Shadow |
| **S1** | Parity with Ref on `tests/native/manifest.txt` L1 subset | stdout/exit match VM |
| **S2** | Aggregates, errors, multi-module link | L2–L3 manifest tiers green |
| **S3** | Cross-target triple + **lld** discovery | Linux + macOS smoke |
| **S4** | `kinglet build` engine = shadow native | driver smoke without Ref on hot path |

Initial implementation **may** call into LLVM through a thin C FFI shim generated from
Ref headers, or emit the same object layout as `KirToLlvm` via ported lowering logic.
Goal of S2: Shadow objects link interchangeably with Ref-produced objects for the same
KIR stamp (optional strict gate).

### D3 — Linker: lld

1. Default link driver: **lld** (`ld.lld` / `lld-link` by triple).
2. Fallback to platform `cc` driver only when lld is not found (documented env override).
3. Static link `libkinglet_rt.a` into every executable (same as 0015 D9).

### D4 — Cross-platform

| Concern | Policy |
|---------|--------|
| Target triple | Host by default; `--target <triple>` overrides |
| Object format | ELF / Mach-O / COFF per triple |
| lld path | `KINGLET_LLD`, `PATH`, or LLVM prefix next to `llvm-config` |
| Windows | S3+; Ref may remain primary until lld-link path is green |

Numeric semantics parity with VM/Ref is **non-negotiable** ([0015](0015-llvm-backend-roadmap.md) D6).

### D5 — Configuration

Shadow build orchestration reads **`kinglet.nest`** ([0020](0020-project-manifest-and-targets.md)),
not `kinglet.toml`. The self-host driver does **not** implement a TOML parser.

Ref compiler may continue reading `kinglet.toml` during transition (0020 D6).

### D6 — Ref remains bootstrap authority until S4

1. CI default: Ref native + VM parity gates unchanged.
2. Shadow native is an explicit tier (`kinglet prove --native` or dedicated CI job).
3. S4 completion does not remove Ref; it makes Shadow a peer for toolchain builds.

### D7 — Intentional non-goals (v1)

- JIT / ORC (spike only if needed for dev).
- Self-host DWARF emitter (may call Ref or defer `-g`).
- WASM / non-ELF-first targets.
- Replacing `libkinglet_rt` with Kinglet-implemented runtime.

## Consequences

- New modules under `compiler/codegen/llvm/` (or `codegen/llvm.kl` tree) in the self repo.
- Bootstrap FFI bridge or shared object description may live in bootstrap for S0 only.
- CI adds optional `shadow-native-smoke` job after S1.
- [0015](0015-llvm-backend-roadmap.md) consequence "Shadow codegen/llvm.kl deferred" is
  superseded by this ADR.

## Open questions

1. **FFI vs pure KL lowering** — S0 chooses FFI to Ref `KirToLlvm` vs port; document
   in implementation plan after spike.
2. **Object interchange** — require bit-identical `.o` or only behavioral link compatibility?
3. **Embedded compiler** — whether S4 replaces [0010](0010-vm-redesign.md) / 0015 L5-2
   embed path for distribution.

## Dependencies

- [0015](0015-llvm-backend-roadmap.md) — KIR lowering reference, `libkinglet_rt`, manifest tests.
- [0016](0016-typed-kir.md) — typed KIR opcodes for native lowering.
- [0014](0014-compilation-toolchain-architecture.md) — Klos, stamps, dual-track compilers.
- [0020](0020-project-manifest-and-targets.md) — Shadow build configuration.

## References

- C++ lowering: `bootstrap/src/codegen/llvm/kir_to_llvm.cc`
- KIR mirror: `kinglet/compiler/ir_emit.kl`, `ir.kl`
- Native smoke: `tests/native/manifest.txt`, `tests/native/run_smoke.sh`

## Amendments

### 2026-06-18 — Route B delivery (textual .ll, no C++ FFI)

S0 is implemented and the open **FFI vs pure-KL** question (Open Question 1) is
resolved in favour of **pure-KL lowering**: the Shadow emits **textual LLVM IR
(`.ll`)** directly, with no FFI to Ref `KirToLlvm` and no C++ on the path. This
matches [0015](0015-llvm-backend-roadmap.md) D-data-layout defaults; the host
`clang++` (lld fallback per D3 is deferred — S3) assembles and links.

Pipeline (delivered):

```
.kl → Shadow compile_program → Chunk
    → build_kir_from_chunk (ir/ir_emit.kl, Chunk→KirModule post-pass)
    → emit_ll (compiler/ll_emit.kl, KirModule→.ll)
    → clang++ -c → clang++ link libkinglet_rt.a → native binary
```

Files: `ir/ir.kl` (`KirType`), `ir/ir_emit.kl` (`build_kir_from_chunk`), the
`Compiler.kir` field + `compile_program` hook (`compiler_state.kl`,
`compiler/compiler.kl`), `compiler/ll_emit.kl` (`emit_ll`), the `--emit-ll` flag
(`core/main.kl`), and `tests/native/run_smoke_shadow.sh` + `shadow_manifest.txt`.

S0 exit criterion met: `just42.kl` → native binary exits 42 (no C++ `KirToLlvm`).

S1 integer lowering is implemented with an SSA virtual stack for expression
values and **alloca-backed locals** (`StoreLocal`→`store`, `LoadLocal`→`load`),
so mutable variables work across control flow without phi nodes. Lowered:
`int` constants, arithmetic (`Add`/`Subtract`/`Multiply`/`Divide`/`Modulo`, incl.
`I32` variants), local variables, bitwise/shift (`BitAnd`/`BitOr`/`BitXor`/
`Shl`/`Shr`), comparisons (`Eq`/`Neq`/`Lt`/`Gt`/`Le`/`Ge` → `icmp` + `zext i1 → i64`),
unary (`Negate`/`BitNot`), **control flow** — basic-block reconstruction
from `Jmp`/`JmpFalse`, covering `if`/`else` and `while` loops, including variables
mutated across branches and iterations — and **function calls** (direct calls,
including recursion; `kinglet_fn_<name>` mangling, `main`→`kinglet_user_main`;
each function lowered to its own `define` with `i64` parameters spilt to
alloca slots). Verified by ~42 cases in `shadow_manifest.txt`. Integer
arithmetic is lowered as `i64`, matching the VM for non-overflowing values (i32
overflow parity is a later tier). Typed-pointer syntax (`i64*`) is used so the
emitted `.ll` assembles on LLVM 14 as well as 15+.

Deferred: aggregates (structs/arrays/strings), and errors.
