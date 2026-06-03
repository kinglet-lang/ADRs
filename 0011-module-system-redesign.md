# 0011 — Module System Redesign

- **Status**: accepted
- **Proposed**: 2026-06-03

## Context

The current `import "file.kl" { sym1, sym2 }` system has fundamental design flaws that prevent normal software engineering practices (splitting files, reusing types across modules):

### Current Problems

1. **Duplicate registration corrupts indices** — importing the same module through multiple paths re-registers pub symbols under fresh indices, scrambling function dispatch. This forces the AST mirror pattern: `checker.kl` and `cli/main.kl` manually duplicate 100+ lines of type definitions instead of importing `ast.kl`.

2. **Single-file constraint** — `parser/parser.kl` (1200+ lines) cannot be split because struct-meta registration across module boundaries corrupts field access. `cli/main.kl` (1285 lines) cannot be split for the same reason.

3. **No proper scoping** — imported symbols land in a flat global namespace. Name collisions between modules are resolved by last-writer-wins, not by error.

4. **Transitive import is fragile** — the compiler deduplicates by resolved path (`str_arr_contains(loaded_paths, resolved)`), but the checker has no deduplication at all. Shared dependencies are re-processed per importer.

5. **Selective import is incomplete** — `import "x.kl" { foo }` filters functions but not struct/enum types that `foo`'s signature depends on, causing silent type resolution failures.

## Decision

### Syntax

```kl
// 1. Dependency declaration — top of file, exactly once
import {
  "../parser/ast.kl"             // file stem → namespace name: ast::
  "../parser/parser.kl"          // → parser::
  "../../utils/file.kl" as f     // explicit alias → f::
}

// 2. Stdlib namespaces (unchanged)
using io;
using sys;

// 3. Selective symbol import — pull into current scope
using ast { Program, Decl, Expr };
using parser { parse_program };

// 4. Usage
Program prog = parse_program(tokens);    // pulled in via using
ast::make_type("int");                   // qualified access
f::read_all("path");                     // alias access
io::out.line("hello");                   // stdlib unchanged
```

### Rules

1. **`import {}`** declares file dependencies and creates namespaces. Appears once, at the top of the file (before any `using`, function, or type declaration).

2. **One file = one module = one namespace.** The namespace name defaults to the filename without `.kl`. Conflicts require an explicit `as` alias.

3. **Singleton loading.** Each file is parsed and registered exactly once globally. Multiple files importing the same path share one instance — no re-registration, no index corruption.

4. **Transitive imports do not propagate.** A module's internal `import {}` is private. Importers only see the module's `pub` symbols, never its dependencies.

5. **`using module { sym };`** pulls selected `pub` symbols from an already-imported module into the current scope. Naming conflicts at this level are compile-time errors.

6. **`using name;`** opens a stdlib namespace (io, fs, sys). Unchanged from current behavior.

7. **No multi-level nesting.** `a::b::c` is not supported. If a module re-exports symbols from its own dependencies, it does so via `pub` wrapper functions or types — not via transitive namespace chaining.

### Visibility

`pub` controls whether a symbol is exported from its module:

```kl
// parser/ast.kl
pub struct Program { ... }          // visible to importers as ast::Program
pub enum Decl { ... }               // visible
TypeExpr make_helper() { ... }      // no pub → module-private
```

```kl
// other file
using ast { Program };              // OK
using ast { make_helper };          // compile error: not pub
```

Two levels only:
- `pub` = exported (accessible via `module::` or `using module { }`)
- no `pub` = private (only within the same file)

No `pub(crate)`, `internal`, or friend declarations.

### Circular Dependencies

Circular imports are a **compile-time error**. If `a.kl` imports `b.kl` and `b.kl` imports `a.kl`, the compiler reports the cycle and halts. Resolution: extract shared types into a third file imported by both.

This is the simplest rule. Lazy/deferred loading adds complexity that isn't justified for a compiled language without runtime module evaluation.

## Migration

### From current syntax

| Before | After |
|--------|-------|
| `import "../parser/ast.kl" { Program, Decl }` | `import { "../parser/ast.kl" }` + `using ast { Program, Decl };` |
| `import "../parser/parser.kl"` | `import { "../parser/parser.kl" }` |
| AST mirror in checker.kl (100+ lines) | Delete. `import { "../parser/ast.kl" }` + `using ast { ... };` |
| AST mirror in cli/main.kl (100+ lines) | Delete. Same as above. |

### Immediate unblocks

- `parser/parser.kl` can be split into `parser/parser.kl` + `parser/helpers.kl` + `parser/expressions.kl`
- `cli/main.kl` can be split into `cli/main.kl` + `cli/ast_printer.kl` + `cli/checker_driver.kl`
- No more AST mirror maintenance (eliminates ~200 lines of duplicated code)

## Consequences

- `import "file.kl" { syms }` syntax is deprecated and eventually removed.
- `import {}` block + `using module { syms };` replaces it.
- `pub` remains required for exported symbols (no change in semantics, clearer enforcement).
- Compiler implementation needs a global module registry keyed by resolved path.
- .kbc format may need a module table section (for the VM to resolve cross-module function indices).
- Bootstrap (C++) and self-host compilers must both implement the new semantics.

## Dependencies

- 0010 (VM redesign) — module singleton loading coordinates with the new Value model. Module-level constants can be heap-allocated once and shared via RC.
- Bootstrap compiler must implement first (to compile the self-host with new import syntax).

## Open Questions

1. **Relative path resolution** — relative to the importing file (current), or relative to a project root? File-relative is simpler but deeply nested paths get ugly (`"../../../../core/types.kl"`).
2. **Index file convention** — should `import { "../parser" }` resolve to `parser/index.kl` or `parser/parser.kl`? Or require explicit file paths always?
3. **Re-exports** — should a module be able to `pub using dep { Sym };` to re-export a dependency's symbol under its own namespace?
