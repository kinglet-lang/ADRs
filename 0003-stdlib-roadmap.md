# 0003 — Standard Library Roadmap

- **Status**: accepted (phased)
- **Proposed**: 2026-05-29
- **Revised**: 2026-06-15

## Context

After the self-host round-trip is verified, the standard library is the next major milestone. There is **no `stdlib/` directory today**; I/O, filesystem, and system access are implemented as VM opcodes plus hardcoded tables in the compiler.

### Current state: builtins live in three places

| Layer | Location | Role |
|-------|----------|------|
| **VM** | C++ `OpCode::NativeOut` / `NativeFsRead` … | Calls into the OS |
| **Compiler** | `codegen.kl` `emit_native_call` + `compiler_state.kl` `is_native_member` | `io::out.line` → emit opcode |
| **Checker** | Duplicated native tables in `checker.kl` | Type checking |

User syntax (see [0011](0011-module-system-redesign.md)) treats `using io;` / `io::out.line(...)` as **stdlib namespaces**, but the implementation is still compiler magic — **not** `.kl` modules loaded via `import {}`.

**Language builtins** are also opcode-dispatched. A finer boundary applies here (revised 2026-06-15, see scope note below): true language *syntax/operators* — array/map **literals**, the index operator `a[i]`, `match`, `const` lowering — stay compiler responsibilities and are never stdlib. But the *method surface* on built-in types (`string`/`array`/`map` methods such as `.len()`/`.push()`/`.split()`/`.has()`) **is in scope for a later phase**: those methods migrate to stdlib written over `extern native` collection primitives, so the compiler stops carrying a hardcoded method whitelist.

### Goal

Introduce a `stdlib/` tree so platform APIs (`io` / `fs` / `sys`) have a single Kinglet source of truth. The compiler should only hardcode **how to lower `extern native` to opcodes**, not scatter `if (ns == "io" && member == "out")` tables across three files.

## Decision

### Scope and landing target (decided 2026-06-15)

- **Landing pipeline**: the self-host compiler (`kinglet/*.kl`) is the target body. C++ bootstrap (`bootstrap/src/{checker,compiler,vm,codegen}`) is brought to parity afterward (new Phase D), reusing the same single-table approach. The L0 runtime (`vm.cc` opcode bodies, `kl_*`) does not move.
- **Migration scope, in priority order**:
  1. **Platform APIs** `io` / `fs` / `sys` first (Phase A) — the original goal.
  2. **Built-in collection methods** on `string` / `array` / `map` second (new Phase A2) — move the hardcoded method whitelist into stdlib over `extern native` primitives.
  3. Pure-Kinglet modules (Phase B) and self-hosted VM (Phase C) as before.
- **Out of scope (stays language semantics)**: array/map literals, the index operator `a[i]`, `match`, `const` lowering, and the value/copy + deterministic-destruction model from [0002](0002-design-principles.md).

### Four-layer model

The relationship between stdlib and hardcoded builtins is layered. **VM opcodes are not removed**; names and signatures move up into Kinglet:

```
L3  User-facing API (stdlib/, pure Kinglet)
      stdlib/io/mod.kl — println, readln, …
           ↓ calls
L2  Platform bindings (stdlib/, thin wrappers)
      stdlib/native/io.kl — extern native declarations + optional chained API
           ↓ lowers to
L1  Compiler/VM contract (irreducible)
      extern native manifest → opcode table (single source of truth)
           ↓ executes on
L0  Runtime (C++ VM today; see Phase C long-term)
      vm.cc — NativeOut, NativeFsRead, …
```

| Layer | Contents | Still "hardcoded"? |
|-------|----------|-------------------|
| **L0** | VM opcodes, OS hooks | Always retained |
| **L1** | `extern native` declarations or manifest; `emit_native_call` table lookup | Compiler hardcodes **lowering rules**, not the **API list** |
| **L2** | `stdlib/native/*.kl` | Ordinary modules, `import`-able |
| **L3** | `stdlib/io/`, `stdlib/collections/`, etc. | Pure Kinglet |

In short: **VM opcodes remain builtins; stdlib is the Kinglet surface and user API over those builtins.**

### Directory layout

```
kinglet-self/
  kinglet.toml          # [paths] + [prelude]
  stdlib/
    native/
      io.kl             # L2: extern native io { … }
      fs.kl
      sys.kl
    io/
      mod.kl            # L3: println, readln, …
    collections/
    result.kl
    option.kl
    …
```

Module paths use the existing `//` project-root resolution (`find_project_root` + `kinglet.toml`), for example:

```kl
import { "//stdlib/native/io.kl" }
```

### Prelude (implicit imports)

To preserve the ergonomics of `using io;` from [0011](0011-module-system-redesign.md) without forcing every source file to hand-write stdlib imports, declare a prelude in `kinglet.toml`:

```toml
[paths]
stdlib = "stdlib"

[prelude]
import = [
  "//stdlib/native/io.kl",
  "//stdlib/native/fs.kl",
  "//stdlib/native/sys.kl",
]
```

The compiler injects prelude imports at the start of `compile_program` (same singleton loading as explicit `import {}`). After migration:

- `using io;` → opens the **prelude-imported** `stdlib/native/io` module namespace
- `io::out.line(...)` → ordinary module symbol declared `extern native`, no magic namespace special case

`using namespace io;` (see current compiler support) is for unqualified access inside L2 wrappers only. User code should still prefer `using io;` + `io::out.line` or L3 helpers like `println`.

### `extern native` (L1 contract)

Platform-binding `.kl` files self-describe the native API. The compiler reads those declarations to build the opcode map, replacing the three duplicated `is_native_member` tables.

Illustrative syntax (parser/checker TBD; semantics fixed here):

```kl
// stdlib/native/io.kl
extern native namespace io {
  void out(string s);
  void outln(string s);
  void err(string s);
  // …
}
```

During transition, a **manifest file** (`.kl` or generated data) may serve as the single table that checker and codegen both read, before `extern native` exists in the AST.

### Relationship to the 0011 module system

| Mechanism | User modules | stdlib (after migration) |
|-----------|--------------|--------------------------|
| `import {}` | Required; declares dependencies | Prelude auto-import + explicit `import "//stdlib/..."` |
| `using module { sym };` | Pull symbols from an imported module | Same |
| `using io;` | N/A | Open the prelude native module |
| `io::out` | N/A | Qualified access (recommended) |

Stdlib does **not** get a second import system. It is simply **Kinglet modules imported by default via prelude**.

### Phase A — Move platform bindings into stdlib (no user-visible behavior change)

Incremental steps; each can land independently and be rolled back:

1. **Scaffold** — Add `stdlib/native/{io,fs,sys}.kl` stubs; extend `kinglet.toml` with `[paths]` / `[prelude]`. Compiler still uses old hardcoded tables; behavior unchanged.
2. **Single manifest** — Merge the three native tables in `compiler_state.kl`, `codegen.kl`, and `checker.kl` into one (from manifest or L2 `.kl`). Remove duplicate `if` chains.
3. **Wire prelude** — Compiler injects prelude at startup; `using io` resolves to the loaded `stdlib/native/io` module.
4. **Drop magic namespaces** — Replace `is_native_namespace("io")` special cases with "module name + extern native marker"; `emit_native_call` only consults the L1 table.

Optional user-facing API (can run in parallel with steps 3–4):

- `stdlib/io/mod.kl` — `println`, `readln`, etc., calling into L2.

**Explicitly out of scope for Phase A**: removing VM opcodes; moving `array.len()` and other language builtins; breaking existing `io::out.line` call sites.

### Phase A2 — Builtin collection-method migration (string / array / map)

Moves the hardcoded built-in *method* whitelist into stdlib while keeping operators/literals as language syntax. Prerequisite: `extern native` (or a manifest entry) for the irreducible collection primitives that map to existing opcodes / `kl_*` (e.g. `array_push`, `array_len`, `str_split`, `map_has`). High-level methods are then ordinary Kinglet functions resolved via UFCS/methods over those primitives.

Order (least to most coupled):

1. `string` methods (`.len/.contains/.starts_with/.ends_with/.index_of/.slice/.replace/.split/.trim/.to_upper/.to_lower`) — most self-contained.
2. `map` methods (`.len/.has/.remove/.keys`).
3. `array` methods (`.len/.push/.pop/.remove/.contains/.clear/.insert/.index_of/.slice/.reverse/.resize`) — most coupled to literals and the dense layout from [0017](0017-dense-nested-array-layout.md).

Each step removes that group from the compiler's method whitelist (self-host first; bootstrap parity in Phase D) and replaces it with a stdlib definition, locked by the existing probe/golden suites.

**Constraint**: heavier stdlib allocation amplifies the no-GC gap (manual `new`, drops not yet inserted; see [native.md](../docs/native.md)). Evaluate whether KIR drop insertion (ADR 0002) must land before array migration.

### Phase B — Core stdlib modules (pure Kinglet)

After L2 is stable:

- `stdlib/collections/array_list.kl` — generic dynamic array (wraps built-in `T[]`, does not replace language arrays)
- `stdlib/collections/hash_map.kl`
- `stdlib/result.kl` — `Result<T, E>` + combinators
- `stdlib/option.kl` — `Option<T>`
- `stdlib/string/mod.kl` — builder, split, join
- `stdlib/math/mod.kl` — min, max, abs, pow
- `stdlib/iter/mod.kl` — Iterator + map/filter/reduce (depends on concepts; see [0009](0009-concepts-landing.md))

### Phase C — Self-hosted VM (long-term)

Implement the VM in Kinglet; C++ bootstrap becomes a stub only:

- `vm/value.kl` — tagged value representation
- `vm/chunk.kl` — bytecode loader
- `vm/vm.kl` — stack machine interpreter
- `vm/native.kl` — L0 opcode bindings in the self-hosted VM

The L0 opcode set may **move in implementation** as the VM migrates, but the **semantic boundary** (platform vs language builtin) stays the same. See [0010](0010-vm-redesign.md).

### Phase D — C++ bootstrap parity

After the self-host pipeline is stable, fold the same single-table / `extern native` approach into the C++ bootstrap so both pipelines agree: the platform namespace tables in [type_checker.cc](../../bootstrap/src/checker/type_checker.cc) and the seams in [compiler.cc](../../bootstrap/src/compiler/compiler.cc) (`:998-1094` namespaces, `:1241-1347` method whitelist) read one manifest instead of scattered `if` chains. L0 stays untouched: `vm.cc` opcode bodies and the `kl_*` runtime (`declare_runtime` in [kir_to_llvm.cc](../../bootstrap/src/codegen/llvm/kir_to_llvm.cc), impls in `bootstrap/runtime/`) do not change.

## Consequences

### Positive

- Single API table; checker, codegen, and docs no longer maintained in three places.
- Stdlib modules use normal `import` / `pub` / tests, same as compiler sources.
- `using io` stays ergonomic while ceasing to be a compiler special case.

### Risks and constraints

- **Dual-track transition**: while manifest and legacy hardcoding coexist, golden/probe tests must lock behavior. The probe matrix should continue to run entirely via `compiler.kbc` (self-host semantics).
- **Prelude vs explicit import**: library authors who want zero prelude dependency must `import "//stdlib/..."` explicitly; docs must state prelude is on by default.
- **Bootstrap vs self-host**: `extern native` syntax must land in both pipelines, or Phase A uses a manifest data file first to avoid parser gaps.
- **Keep operators/syntax as language semantics**: array/map literals, the index operator `a[i]`, `match`, and top-level `const` lowering stay compiler responsibilities. The revised scope (2026-06-15) *does* migrate the built-in **method surface** (`.len()`/`.push()`/`.split()` etc.) to stdlib in Phase A2 — but only the methods, lowered over `extern native` primitives; the underlying opcodes/`kl_*` remain.

### Acceptance criteria (Phase A complete)

- [ ] `stdlib/native/*.kl` exists and matches the manifest
- [ ] Checker and codegen share one native API table
- [ ] `using io;` + `io::out.line` matches current probe/regression behavior
- [ ] Adding a native member requires only stdlib/manifest changes, not three hardcoded tables

## Dependencies

- [0011](0011-module-system-redesign.md) — `import {}`, singleton loading, `using` rules; stdlib builds on this.
- [0009](0009-concepts-landing.md) — Phase B `Iterator` and generic combinators.
- [0008](0008-kbc-format-evolution.md) — Phase C self-hosted VM loading `.kbc`.
- [0010](0010-vm-redesign.md) — Embedded self-host binary and VM evolution.

## References

- Current native implementation: `compiler/compiler_state.kl`, `compiler/codegen.kl`, `checker/checker.kl`
- Module path resolution: `compiler/compiler.kl` (`find_project_root`, `//` paths)
