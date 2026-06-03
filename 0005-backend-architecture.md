# 0005 — Backend Architecture: KIR + Dual Backend

- **Status**: draft
- **Proposed**: 2026-05-31

## Context

The self-host compiler currently emits bytecode directly from AST. If an LLVM (or WASM, Cranelift, etc.) backend is added later, it must either reuse the AST→bytecode logic (lossy) or re-walk the AST independently (duplication). A shared IR between frontend and backends avoids both problems.

## Decision

### D1: Layered pipeline with shared IR (KIR)

```
source → lexer → parser → checker → IR-emit → KIR
                                               ├→ VM backend → bytecode
                                               └→ LLVM backend → native binary
```

One frontend, two backends (VM + LLVM), with KIR as the seam. Adding a third backend later is "write another lowering", not "rewrite the compiler".

### D2: KIR form

Typed CFG with basic blocks, SSA values. Each basic block ends in a terminator (`Br`, `CondBr`, `Ret`, `Switch`, `Unreachable`).

Lowering level — what KIR sees:
- `match` desugared to `Switch` + bind instructions
- `for`/`while` desugared to `Br`/`CondBr`
- Generic functions monomorphized before IR (KIR sees no `<T>`)
- Method calls resolved to function references (no dynamic dispatch yet)
- Cast / pipeline / chained-comparison sugar already desugared
- `using io;` / native intrinsics are calls to known extern declarations

Memory model: stack slots as `Alloca` + `Load`/`Store`. SSA for register-like temporaries only.

Serialization: in-memory only initially. Textual dump via `--ir` for debugging and golden tests. No binary on-disk format until needed.

### D3: VM backend contract

VM backend output is **identical** to what C++ `compiler.cc` emits today — same opcodes, same constant pool layout. Passes bytecode golden tests byte-for-byte against the C++ output.

### D4: C++ side — Path B (refactor to KIR layers)

Split `compiler.cc` into `ir_builder.cc` (AST→KIR) + `bytecode_emitter.cc` (KIR→bytecode). Same KIR shape on both bs and sh sides. Enables diffing KIR text output between implementations.

Path A (leave C++ alone) was rejected because KIR design without a reference implementation risks happening in a vacuum.

## KIR examples

```
fn add(a: int, b: int) -> int {
  bb0:
    %0 = iadd a, b
    ret %0
}

fn describe(c: Color) -> int {
  bb0:
    %tag = enum_tag c
    switch %tag, default bb_unreachable, [
      0 -> bb_red, 1 -> bb_green, 2 -> bb_blue
    ]
  bb_red:    ret 1
  bb_green:  ret 2
  bb_blue:   ret 3
  bb_unreachable: unreachable
}
```

## Build order

1. Lock down KIR form, lowering level, monomorphization timing, intrinsic protocol.
2. Refactor C++ `compiler.cc` into ir_builder + bytecode_emitter (reference implementation).
3. Self-host `ir/` module: `ir.kl` (data definitions), `ir_emit.kl` (AST + types → KIR), `ir_print.kl` for `--ir` dump.
4. Self-host VM backend: `codegen/vm.kl`. Golden tests diff bytecode against C++ VM backend.
5. Bootstrap fixpoint test: bootstrap-built vs self-built compiler produce byte-identical bytecode.
6. LLVM backend (`codegen/llvm.kl`) — only after everything above is green.

## Open questions

1. **Generics: mono before or after IR?** Current draft says before (KIR sees no `<T>`). Alternative: KIR is generic too, mono is per-backend. Leaning toward centralized mono.
2. **Closures:** when they land, show up as KIR `Closure` or lowered to env-struct + function pointer at IR-emit time? Leaning toward the latter — keeps backends simple.
3. **Panics / aborts:** LLVM backend needs a story. KIR could have `Unreachable` and `Panic(string)`.
4. **Source locations in KIR:** probably yes, every instruction tags source span. Cheap, enables post-IR diagnostics.
5. **Diff format:** what "byte-identical bytecode" compares — opcodes? constant pool order? debug tables?

## Out of scope

- Standard library layout
- `??` / `try` syntax (see 0006)
- Concurrency model
- GC strategy (LLVM path will need one eventually)

## Consequences

- Major refactor of C++ compiler.cc (split into two modules).
- New self-host modules: `ir.kl`, `ir_emit.kl`, `ir_print.kl`, `codegen/vm.kl`.
- Golden test infrastructure needs KIR text comparison.
- Self-host compiler codegen path becomes AST → KIR → bytecode (one more step than today).
