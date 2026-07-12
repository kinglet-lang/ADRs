# 0003 — Standard Library Roadmap

- **Status**: accepted (phased; io/fs refined by 0026/0027)
- **Proposed**: 2026-05-29
- **Revised**: 2026-06-15

**Current refinements**: [0026 — Standard I/O Capability Model](0026-standard-io-capability-model.md) and [0027 — Filesystem Resource API](0027-filesystem-resource-api.md) define the current `io` and `fs` public surfaces; this ADR remains the umbrella roadmap.

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

- **Platform I/O API (decided 2026-06-15)**: the canonical user-facing surface is **`io::out.line` / `io::err.line` / `io::in`** (and related native members). There is **no L3 syntax-sugar layer** — no `println`, `readln`, or `stdlib/io/mod.kl` wrappers over I/O.

### Three-layer model

The relationship between stdlib and hardcoded builtins is layered. **VM opcodes are not removed**; names and signatures move into `stdlib/native/*.kl`:

```
L2  Platform bindings (stdlib/native/*.kl)
      extern native namespace io { … }  — user calls io::out.line(...)
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
| **L2** | `stdlib/native/{io,fs,sys}.kl` | Ordinary prelude modules; `using io;` + `io::out.line` |

In short: **VM opcodes remain builtins; the stdlib is the Kinglet declaration of those builtins — not a second naming layer on top.**

### Directory layout

```
kinglet-self/
  kinglet.toml          # [paths] + [prelude]
  stdlib/
    native/
      io.kl             # L2: extern native io { … }
      fs.kl
      sys.kl
    collections/        # Phase B (non-platform stdlib)
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
- `io::out.line(...)` → ordinary `extern native` symbol, no magic namespace special case

`using namespace io;` (see current compiler support) opens unqualified native members (`out`, `err`, `in`). User code should prefer **`using io;` + `io::out.line`** as the standard I/O style.

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
| `io::out.line` | N/A | Qualified access (canonical) |

Stdlib does **not** get a second import system. It is simply **Kinglet modules imported by default via prelude**.

### Phase A — Move platform bindings into stdlib (no user-visible behavior change)

Incremental steps; each can land independently and be rolled back:

1. **Scaffold** — Add `stdlib/native/{io,fs,sys}.kl` stubs; extend `kinglet.toml` with `[paths]` / `[prelude]`. Compiler still uses old hardcoded tables; behavior unchanged.
2. **Single manifest** — Merge the three native tables in `compiler_state.kl`, `codegen.kl`, and `checker.kl` into one (from manifest or L2 `.kl`). Remove duplicate `if` chains.
3. **Wire prelude** — Compiler injects prelude at startup; `using io` resolves to the loaded `stdlib/native/io` module.
4. **Drop magic namespaces** — Replace `is_native_namespace("io")` special cases with "module name + extern native marker"; `emit_native_call` only consults the L1 table.

**Explicitly out of scope for Phase A**: removing VM opcodes; moving `array.len()` and other language builtins; breaking existing `io::out.line` call sites; I/O syntax sugar (`println`, `readln`, etc.).

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

The L0 opcode set may **move in implementation** as the VM migrates, but the **semantic boundary** (platform vs language builtin) stays the same. See [0010](%5Bdeprecated%5D%200010-vm-redesign.md).

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
- [0008](%5Bdeprecated%5D%200008-kbc-format-evolution.md) — Phase C self-hosted VM loading `.kbc`.
- [0010](%5Bdeprecated%5D%200010-vm-redesign.md) — Embedded self-host binary and VM evolution.

## References

- Current native implementation: `compiler/compiler_state.kl`, `compiler/codegen.kl`, `checker/checker.kl`
- Module path resolution: `compiler/compiler.kl` (`find_project_root`, `//` paths)

## Amendments

### 2026-06-17 — Prelude and paths in `kinglet.nest` ([0020](0020-project-manifest-and-targets.md))

Directory layout and configuration examples that reference `kinglet.toml` with
`[paths]` / `[prelude]` are **amended** for new work:

| This ADR (0003) | Amended policy |
|-----------------|----------------|
| `kinglet.toml` `[paths]` + `[prelude]` | `kinglet.nest` `prelude { import "..." }` block ([0020](0020-project-manifest-and-targets.md) D2) |
| `import { "//stdlib/native/io.kl" }` in user code | Retained; logical `import io;` when stdlib is a manifest `dependency` ([0018](0018-logical-module-system.md)) |
| Project root = directory containing manifest | Directory containing `kinglet.nest` |

Self-host must **not** grow a TOML parser; prelude wiring moves to the PML loader
([0020](0020-project-manifest-and-targets.md) D5). C++ Ref may dual-read TOML during
transition (0020 D6).

Original `kinglet.toml` examples in §Directory layout and §Prelude are preserved for
historical context.

### 2026-06-17 — Runtime via C lowering; no project prelude ([0020](0020-project-manifest-and-targets.md))

The **prelude-in-manifest** approach (inject `stdlib/native/*.kl` via `kinglet.toml`
or `kinglet.nest`) is **deferred** in favour of:

| Topic | Policy |
|-------|--------|
| `io` / `fs` / `sys` | Compiler-known platform namespaces; **lowered to C** (`libkinglet_rt` / VM natives) |
| User syntax | `using io;`, `io::out.line(...)` — no per-project prelude config |
| Stdlib `.kl` files | Optional **type stubs** for checker/LSP; not loaded via manifest |
| Manifest | `modules` + `targets` only ([0020](0020-project-manifest-and-targets.md) PML v2) |

Rationale: runtime capability is a **toolchain contract** (KL → KIR → C calls), not a
dependency users wire in each `kinglet.nest`. Avoids duplicate graphs and ugly prelude
blocks.

Previous Amendment 2026-06-17 (prelude in `.nest`) is superseded by this entry.

### 2026-06-17 — Stdlib directory tree; `using namespace`; C ABI boundary

**Directory layout** replaces §Directory layout `stdlib/native/`:

```
stdlib/
  manifest.kl           # L1: native API registry → opcode / C ABI table
  io/
    io.kl               # L2 stub: platform I/O (C ABI only)
  fs/
    fs.kl
  sys/
    sys.kl
  math/
    math.kl             # pure Kinglet (no C ABI)
  algorithms/
    algorithms.kl
  map/
    map.kl              # pure KL over language map type; primitives via C ABI
  …
```

| Category | Path pattern | Implementation |
|----------|--------------|----------------|
| **Platform** | `stdlib/{io,fs,sys}/<name>.kl` | **C ABI** (`libkinglet_rt`, VM natives); KL file is declaration + docs |
| **Pure stdlib** | `stdlib/<pkg>/<pkg>.kl` | Ordinary Kinglet; no manifest entry required |
| **L1 table** | `stdlib/manifest.kl` | Single `is_native_*` source for checker/codegen until `extern native` syntax lands |

`stdlib/native/` is **deprecated**; new paths are `//stdlib/io/io.kl`, etc.

**User ergonomics (long-term, both retained):**

| Form | Meaning |
|------|---------|
| `using io;` | Open namespace; call `io::out.line(...)` |
| `using namespace io;` | Open unqualified members (`out.line`, `err.line`, `in.secret`) |

No deprecation of `using namespace io;`. Checker registers native aliases via
`manifest::native_members_of` when `using namespace` is parsed.

**C ABI boundary:** any capability that touches the OS, process environment, or runtime
wire format is implemented in **C** (today: `libkinglet_rt` + VM opcodes). Kinglet
sources declare the API; the compiler **lowers** calls to C symbols. Pure modules
(`math`, `algorithms`, …) have no `extern native` entries in `manifest.kl`.

**Project manifest:** user `kinglet.nest` does **not** list stdlib platform modules.
Optional explicit `import math;` only when a user project adds `stdlib/math/math.kl` to
its own `modules { }` block (toolchain repo paths are separate).

Toolchain `kinglet.toml` `[prelude]` may still list platform stubs during C++/TOML
transition; removed when compiler injects platform namespaces without prelude
(0020 Amendment 2026-06-17).

Original §Three-layer model and §Directory layout text above are preserved for
historical context.

### 2026-07-10 — Standard I/O and filesystem surface refined ([0026](0026-standard-io-capability-model.md), [0027](0027-filesystem-resource-api.md))

The platform I/O roadmap remains the umbrella decision, but the concrete public
surfaces for `io` and `fs` are refined by later ADRs:

- [0026](0026-standard-io-capability-model.md) defines the `io::reader` /
  `io::writer` capability model while preserving existing `io::in`, `io::out`,
  and `io::err` behaviour.
- [0027](0027-filesystem-resource-api.md) defines `fs::file` and the resource API
  over the current hardcoded `fs::__read`, `fs::__write`, and `fs::__listdir`
  intrinsic surface.

Original roadmap sections above are preserved for historical context.
