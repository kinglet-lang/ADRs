# 0018 — Logical Module System

- **Status**: implemented (source-level modules; manifest shape amended by [0020](0020-project-manifest-and-targets.md))
- **Proposed**: 2026-06-17
- **Completed**: 2026-06-20

**Current refinement**: [0020 — Project Manifest (`.nest`) and Build Targets](0020-project-manifest-and-targets.md) replaces this ADR's earlier manifest-owned `modules {}` mapping with target blocks and source-declared module identity.

## Context

[0011](0011-module-system-redesign.md) replaced path-selective `import "file.kl" { syms }`
with `import {}` blocks and file-stem namespaces. That unblocked multi-file projects and
singleton loading, but path-based imports still leak filesystem layout into source:

- Deep relative paths (`"../../../../core/types.kl"`) are brittle.
- The namespace name is tied to the filename stem, not an explicit module identity.
- `kinglet.toml` and source files both carry path information with overlapping roles.

The next step separates **logical module names** (what code imports) from **physical
paths** (where modules live on disk). Physical paths move into the project manifest
([0020](0020-project-manifest-and-targets.md)); source files declare their exported
identity with `export module`.

Stdlib ergonomics (`using io;`, `io::out.line`) remain unchanged per
[0003](0003-stdlib-roadmap.md) and [0011](0011-module-system-redesign.md) Amendment
2026-06-17.

## Decision

### D1 — `export module` declares logical identity

Every compilable `.kl` file that participates in the module graph begins with:

```kl
export module <name>;
```

`<name>` is a single identifier (no `::` nesting). It is the canonical namespace for
qualified access (`math::add`) and for manifest `dependency` keys.

Rules:

1. One file exports exactly one module name.
2. Duplicate `export module` names in the same project are a compile-time error.
3. The name in `export module` must match the manifest `dependency` key when the file
   is registered under that key (see [0020](0020-project-manifest-and-targets.md) D2).

### D2 — `import <name>;` replaces `import {}`

Dependency declaration in source:

```kl
import math;
import io;          // only when not covered by prelude; stdlib unchanged ergonomically
```

Resolution:

1. Look up `<name>` in the project manifest `dependency` table ([0020](0020-project-manifest-and-targets.md)).
2. Load the resolved file once (singleton, same as 0011 D3).
3. Expose `pub` symbols under the logical namespace `<name>::`.

Transitive imports do not propagate (unchanged from 0011 D4).

### D3 — `using` rules (stdlib + selective + wildcard)

| Form | Meaning |
|------|---------|
| `using io;` | Open stdlib namespace `io` (unchanged) |
| `using namespace io;` | Open unqualified stdlib members (`out`, `err`, `in`) |
| `using math { add };` | Pull selected `pub` symbols from imported module `math` |
| `using math::*;` | Pull all `pub` symbols from `math` into current scope |

Wildcard rules:

1. `using math::*;` requires `import math;` (or prelude-equivalent registration).
2. A pulled symbol that collides with a name already in scope is a compile-time error
   (no silent shadowing).
3. Wildcard does not re-export; importers still see only `pub` symbols via `math::`.

### D4 — Visibility and cycles (unchanged semantics)

- `pub` / non-`pub` two-level visibility (0011 D visibility section).
- Circular imports are a compile-time error (0011).

### D5 — Syntax migration

| 0011 (retained until removal) | 0018 |
|-------------------------------|------|
| `import { "../math.kl" }` | `import math;` + manifest entry |
| `ast::Program` (stem namespace) | `ast::Program` (logical `export module ast`) |
| `using ast { Program };` | unchanged |
| — | `using ast::*;` (new) |

`import {}` block syntax is **deprecated** when 0018 lands and **removed** in a later
release after migration tooling and tests are green.

### D6 — Project root and `//` paths

Project root is the directory containing `kinglet.nest` ([0020](0020-project-manifest-and-targets.md)).

`//relative/path.kl` in manifest prelude or legacy tooling resolves relative to that
root. Source files do not embed dependency paths once 0018 is fully adopted.

### D7 — Implementation order

1. Manifest loader + `dependency` table (0020) in C++ Ref.
2. Parser/checker: `export module`, `import name`, `using ns::*`.
3. Migrate self-host sources and tests.
4. Shadow compiler parity ([0013](%5Bdeprecated%5D%200013-bootstrap-bytecode-delta.md)).
5. Preen + Perch grammar/LSP updates.

## Consequences

- [0011](0011-module-system-redesign.md) import syntax is superseded; singleton loading
  and visibility rules remain authoritative unless amended.
- Module registry keys shift from resolved path stem to logical name + path mapping in
  `.nest`.
- `.kbc` module tables may record logical names for native link (coordinate with
  [0015](0015-llvm-backend-roadmap.md) Amendment 2026-06-17).
- Open questions from 0011 §Open Questions #1 (path resolution) are **resolved**:
  paths live in manifest; imports use logical names.

## Open questions

1. **Re-exports** — `pub using dep { Sym };` deferred; not required for v1.
2. **External dependencies** — git/registry `dependency` forms defined in 0020; version
   resolution policy TBD.
3. **Transition** — whether Ref accepts both `import {}` and `import name` during one
   release (recommended: yes, with deprecation warning).

## Dependencies

- [0011](0011-module-system-redesign.md) — singleton loading, visibility, transitive rules.
- [0020](0020-project-manifest-and-targets.md) — `dependency` table and project root.
- [0003](0003-stdlib-roadmap.md) — prelude and stdlib module names.

## References

- Self-host module loader: `compiler/compiler.kl`
- C++ module loader: `bootstrap/src/compiler/compiler.cc`

## Amendments

### 2026-06-17 — Platform runtime without `import io` ([0020](0020-project-manifest-and-targets.md))

§D2 example `import io;` is **amended**: `io`, `fs`, and `sys` are **not** manifest
dependencies. They are compiler-known runtime namespaces lowered to C ([0003](0003-stdlib-roadmap.md)
Amendment 2026-06-17). Use `using io;` directly; no prelude or `import io;` required.

Original §D2–D3 text above is preserved for historical context.

### 2026-06-17 — `using namespace` retained ([0003](0003-stdlib-roadmap.md))

`using namespace io;` (and `fs`, `sys` when applicable) is **permanently supported**,
alongside `using io;` + qualified `io::` access. See [0003](0003-stdlib-roadmap.md)
Amendment 2026-06-17 (ergonomics table).

### 2026-06-17 — Hierarchical module ids and revised `using` forms

§D1–D3 are **amended** for hierarchical logical names and a narrower `using` surface.

#### Module ids (dots) vs qualified paths (colons)

Logical module names are **dot-separated** identifiers:

```kl
// src/compiler/ast.kl
export module compiler.ast;
```

Manifest key matches exactly:

```nest
modules {
  compiler.ast = "compiler/ast.kl"
}
```

In source, the module id maps to a **qualified prefix** by replacing `.` with `::`:

| Module id | Qualified prefix | Example |
|-----------|------------------|---------|
| `math` | `math::` | `math::abs` |
| `compiler.ast` | `compiler::ast::` | `compiler::ast::Node` |

Dots appear only in `export module`, `import`, manifest keys, and the right-hand side of
a module alias. Call sites always use `::`.

#### Import and access

```kl
import compiler.ast;

int main() {
  compiler::ast::Node node = compiler::ast::parse();
  return 0;
}
```

Or with a **module alias** (preferred when the prefix is long):

```kl
import compiler.ast;
using ast = compiler.ast;

int main() {
  ast::Node node = ast::parse();
  return 0;
}
```

Rules:

1. `import <module-id>;` is required before qualified or aliased use (transitive imports
   still do not propagate).
2. `using <alias> = <module-id>;` binds a short qualifier; `<alias>::sym` desugars to
   the module's qualified prefix + `sym`.
3. Only `pub` symbols are visible through import/alias/namespace-open.

#### `using` forms (revised)

| Form | Status | Meaning |
|------|--------|---------|
| `using io;` | **retained** (platform) | Qualified `io::` access for compiler-known runtime namespaces |
| `using namespace io;` | **retained** | Unqualified platform members (`out`, `err`, `in`) |
| `using namespace <module-id>;` | **supported** | Unqualified `pub` members from an imported module (e.g. `using namespace compiler.ast;`) |
| `using <alias> = <module-id>;` | **new** | Short qualifier alias (see above) |
| `using math { add };` | **removed** | No selective symbol import blocks for user modules |
| `using math::*;` | **removed** | No wildcard symbol import |

Rationale: block and wildcard `using` duplicate import visibility, make shadowing harder
to reason about, and read like a second import system. Prefer `import` + qualified
`::` access, `using alias = module`, or `using namespace module`.

#### Visibility (unchanged)

Non-`pub` items are visible only inside the defining module file:

```kl
export module compiler.ast;

pub Node parse() { return parse_impl(); }

Node parse_impl() { ... }   // module-private
```

#### Migration from flat names (current toolchain)

Today's self-host tree uses flat ids (`export module ast`, manifest key `ast`). Target
layout groups by package prefix:

| Current | Target |
|---------|--------|
| `export module ast` | `export module parser.ast` |
| `import ast;` / `ast::Program` | `import parser.ast;` / `parser::ast::Program` or `using ast = parser.ast;` |
| `using checker { … }` | remove; use qualified or `using namespace` |

Implementation order update: dotted `module-id` parsing → alias/`using namespace` on
module ids → checker/compiler qualified-path resolution → migrate `kinglet.nest` and
self-host sources → remove `using { }` and `using ::*` parsing.

Original §D1–D3 and §D5 rows for `using ast { … }` / `using ast::*` above are preserved
for historical context.

### 2026-06-20 — Implemented in bootstrap; source-level modules are current

Bootstrap landed the source-level module model in the 2026-06-17 through
2026-06-20 change series: dotted module ids, `export module`, `import
<module-id>;`, module aliases, and `using namespace <module-id>;`. The current
parser and module loader contain `parse_module_id`, `UsingAliasDecl`,
`module_id_to_qualifier`, and `ModuleLoader::resolve_logical`; import block syntax
is rejected with a diagnostic that points users to `import module-id;`.

Implementation landmarks:

- `34a5d44` (2026-06-17): `export module`, logical import, and wildcard import
  parsing.
- `e133309` / `d745adc` / `ec790ec` (2026-06-17): hierarchical module ids,
  checker resolution, and compiler resolution.
- `8ce52de` (2026-06-20): completed T1 import and `using namespace` rules and
  rejected selective platform using blocks.

The manifest mapping shape described in earlier sections was later changed by
[0020](0020-project-manifest-and-targets.md): current `kinglet.nest` declares
build targets and source lists, while `export module` in source provides module
identity.
