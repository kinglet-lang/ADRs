# 0020 — Project Manifest (`.nest`) and Build Targets

- **Status**: draft
- **Proposed**: 2026-06-17

## Context

[0014](0014-compilation-toolchain-architecture.md) introduced `kinglet.toml` for
`[build]`, cache paths, and compiler engine selection. The parser is an ad-hoc subset
in C++ (`project_config.cc`) and a line-scanner in self-host (`project_config.kl` for
`[prelude]` only).

Problems:

1. **TOML in self-host** — a correct TOML implementation in Kinglet is poor ROI; each
   new field (`[[target]]`, nested tables) encourages another partial parser.
2. **`main.kl` coupling** — `[build].root = "src/main.kl"` conflates filename convention
   with executable entry; library projects and multiple binaries are awkward.
3. **Split responsibilities** — [0018](0018-logical-module-system.md) moves dependency
   paths into the manifest; build entry should be explicit **targets**, not a single root
   path.

## Decision

### D1 — Project manifest file: `kinglet.nest`

| Property | Value |
|----------|--------|
| **Filename** | `kinglet.nest` (fixed name at project root) |
| **Suffix** | `.nest` — project **nest** manifest; not compilable `.kl` |
| **Format** | **PML** (Project Manifest Language): Kinglet-*flavored* syntax subset |
| **Parser** | Dedicated manifest lexer/parser; does **not** use the full KL parser |

Project root discovery: walk parents from the current file or cwd until `kinglet.nest`
is found (same algorithm as today's `kinglet.toml` search).

### D2 — PML v1 grammar (informal)

```kl
project "myapp" version "0.1.0"

dependency math -> "src/math.kl"
dependency io -> "//stdlib/native/io.kl"

binary app -> module "app"
library math -> module "math"

build {
  default "app"
  backend native | vm
  cache ".kinglet/cache"
  out ".kinglet/out"
}

prelude {
  import "//stdlib/native/io.kl"
  import "//stdlib/native/fs.kl"
  import "//stdlib/native/sys.kl"
}

compiler {
  engine ref | shadow
}
```

**Statements:**

| Construct | Purpose |
|-----------|---------|
| `project "name" version "semver"` | Project identity |
| `dependency <id> -> "<path>"` | Maps logical name → file ([0018](0018-logical-module-system.md)) |
| `binary <artifact> -> module "<logical>"` | Executable target; entry module must export `pub int main()` |
| `library <artifact> -> module "<logical>"` | Compile/cache module object; no link step |
| `build { ... }` | Default target, backend, cache/out dirs |
| `prelude { import "..." ... }` | Implicit imports ([0003](0003-stdlib-roadmap.md)) |
| `compiler { engine ... }` | Ref vs Shadow ([0014](0014-compilation-toolchain-architecture.md)) |

**Lexical rules (v1):**

- Double-quoted strings only; `\"` escape supported.
- `#` line comments.
- Blocks use `{` `}`; one statement per line inside blocks.
- No arbitrary KL expressions, functions, or types in v1.

### D3 — Target registration

`binary` / `library` lines register **build targets**. They are distinct from
`dependency`:

| Concept | Declared in | Consumed by |
|---------|-------------|-------------|
| `dependency` | manifest | `import <id>;` ([0018](0018-logical-module-system.md)) |
| `binary` / `library` | manifest | `kinglet build [artifact]` |

Rules:

1. `module "<logical>"` must match an `export module` in the file resolved by
   `dependency <logical>` (or the same id as dependency key).
2. `kind = binary` requires `pub int main()` on the entry module (signature fixed in
   implementation docs; exit code rules match VM/native [0015](0015-llvm-backend-roadmap.md)).
3. If exactly one `binary` exists, it is the default when `build.default` is omitted.
4. If `build.default` is set, it must name a registered `binary` artifact.
5. Target dependency graph follows **source `import` only** — no duplicate `deps` in
   manifest (avoids two graphs).

**No `main.kl` requirement.** `kinglet init` generates e.g. `src/app.kl` with
`export module app;` and:

```kl
binary app -> module "app"
```

### D4 — CLI mapping

| Command | Behaviour |
|---------|-----------|
| `kinglet build` | Build `build.default` binary target |
| `kinglet build <artifact>` | Build named `binary` or `library` target |
| `kinglet run` | Run default binary output under `build.out` |
| `kinglet run <artifact>` | Run named binary |
| `kinglet run path/to/file.kl` | Single-file mode; **no** manifest required |
| `kinglet init` | Create `kinglet.nest` + starter `src/<module>.kl` |

Stamp / Klos ([0014](0014-compilation-toolchain-architecture.md)): include target
artifact name and manifest hash in stamp input.

### D5 — Self-host does not parse TOML

The Shadow compiler and Kinglet-implemented build driver read **`kinglet.nest` only**
via `manifest/` loader modules. TOML parsing is **not** part of the self-host bootstrap
surface.

### D6 — Transition from `kinglet.toml`

| Phase | Ref (C++) | Shadow (KL) |
|-------|-----------|-------------|
| T0 | `kinglet.toml` | prelude scan of `kinglet.toml` (current) |
| T1 | Read `.nest` **or** `.toml` (`.nest` wins if both) | `kinglet.nest` only |
| T2 | `kinglet init` writes `.nest`; `kinglet migrate` converts toml → nest | full manifest |
| T3 | Deprecate `kinglet.toml`; remove dual-read |

`kinglet migrate` is implemented in C++ (one-shot conversion); not required in Shadow.

### D7 — Ecosystem

| Tool | Action |
|------|--------|
| Perch | Optional `kinglet-manifest` language id for `*.nest` |
| Preen | Format rules for manifest files (extensions TBD) |
| Tests | `tests/manifest/` golden parse + build integration |

## Consequences

- [0014](0014-compilation-toolchain-architecture.md) D5 `kinglet.toml` schema is
  superseded for new projects; see Amendment 2026-06-17.
- `project_config.kl` TOML scanner is replaced by `manifest/parser.kl` (or equivalent).
- C++ `default_output_name` special case for `core/main.kl` → `compiler` becomes
  `binary compiler -> module "main"` in kinglet-self's `kinglet.nest`.
- Open [0014](0014-compilation-toolchain-architecture.md) `kinglet init` deferred item
  closes when T2 lands.

## Open questions

1. **`library` v1 scope** — emit Klos object only, or also installable `.a`?
2. **External `dependency`** — `dependency foo -> git "url" rev "sha"` syntax deferred.
3. **Monorepo** — multiple `.nest` files not supported in v1.
4. **Fixed filename** — only `kinglet.nest` vs allowing `*.nest` search (v1: fixed name).

## Dependencies

- [0018](0018-logical-module-system.md) — `dependency` + `import <name>`.
- [0014](0014-compilation-toolchain-architecture.md) — Klos, stamps, build commands.
- [0019](0019-self-host-llvm-backend.md) — Shadow reads `.nest` for native builds.
- [0003](0003-stdlib-roadmap.md) — prelude block.

## References

- C++ config today: `bootstrap/src/module/project_config.{h,cc}`
- Self-host prelude scan: `kinglet/compiler/project_config.kl`

## Amendments

### 2026-06-17 — PML v2 layout; no manifest prelude; runtime via lowering

**PML v2 (preferred)** replaces the flat `dependency` / `binary` / `library` lines in
§D2 with grouped blocks:

```kl
project "myapp" version "0.1.0"

modules {
  math = "lib/core/math.kl"
  app  = "apps/demo/main.kl"
}

targets {
  demo = binary "app"
  math = library
}

build {
  default = "demo"
  backend = native
  out     = ".kinglet/out"
}
```

| Block | Meaning |
|-------|---------|
| `modules { <id> = "<path>" }` | Logical module registry ([0018](0018-logical-module-system.md)); keys match `export module` |
| `targets { <artifact> = binary "<module>" }` | Executable; `<module>` must be a `modules` key with `pub int main()` |
| `targets { <artifact> = library }` | `<artifact>` equals the module key; compile/cache only |

`compiler { engine ... }` is optional (default `ref`). Omitted in small projects.

**No `prelude` block.** Platform I/O (`io`, `fs`, `sys`) is **not** configured per
project. It is a **compiler-known runtime surface**:

1. Declarations live under `stdlib/{io,fs,sys}/` ([0003](0003-stdlib-roadmap.md)
   Amendment 2026-06-17).
2. Calls lower to **C ABI** in `libkinglet_rt` (VM opcodes or LLVM native calls).
3. `using io;` and `using namespace io;` work without `import` or manifest entries.

User modules still use `import <id>;` from `modules { }` only. Runtime namespaces are
orthogonal to the project dependency graph. See [0003](0003-stdlib-roadmap.md)
Amendment 2026-06-17 (runtime model).

§D2 v1 grammar (`dependency` / `prelude` lines) remains valid for transition; v2 is
the recommended style for new `kinglet.nest` files.

Original §D2 text above is preserved for historical context.
