# 0031 - Diagnostic System: Error Codes, Severity, and Rendering

- **Status**: accepted
- **Proposed**: 2026-07-15

Depends on: nothing new. Builds on the existing `SourceLocation` (`ast.h`),
`TypeError` / `ParseError` / `CompileError` structs, and the current
`main.cc` stderr printing loop.

## Context

Kinglet's compiler currently has **three parallel error-reporting
structures** that do not share a common interface:

| Pipeline stage | Struct | Fields | Severity |
|----------------|--------|--------|----------|
| Lexer | `Token` (kind `ERROR`) + `had_lexer_error` flag | message in token text | implicit error |
| Parser | `ParseError` (`parser.h:18`) | `line`, `column`, `message` | implicit error |
| Type checker | `TypeError` (`type_checker.h:28`) | `location`, `message`, `severity` | Error / Warning / Info / Hint |
| Compiler | `CompileError` / `CompileWarning` (`compiler.h:23`) | `location`, `message` | separate structs |

Problems with the current state:

1. **No error codes.** Every diagnostic is a free-form string. Users
   cannot suppress, filter, or reference a specific error by code. LSP
   integration has no stable identifier to report.

2. **Three incompatible structs.** `ParseError` stores `line`/`column`
   as bare `int`s; `TypeError` uses `ast::SourceLocation`; `CompileError`
   also uses `SourceLocation` but lacks a severity field. Converting
   between them or building a unified diagnostic sink requires ad-hoc
   glue at each call site.

3. **No structured labels or fix-its.** Ownership and borrow errors
   typically need two spans (the use site and the original
   transfer/borrow site), but the current `TypeError` carries only one
   `location` + one `message`. The checker works around this by stuffing
   both locations into the message string, which is lossy and
   unparseable by tools.

4. **No warning groups or suppression flags.** All warnings are always
   emitted. There is no `-W` / `-Wno-` mechanism, no `-Werror`, and no
   way to categorise warnings for selective control.

5. **Rendering is hard-coded in `main.cc`.** The display format
   (`line:col: error: message`) is a `std::cerr <<` chain in the driver.
   Changing the output format (e.g. to add error codes, source context,
   or JSON mode for LSP) requires editing the driver, not the diagnostic
   producers.

6. **No source context in output.** The current format prints only
   `line:column: error: message` with no source snippet, no caret, and
   no secondary labels. This makes it hard to understand ownership and
   borrow errors that span multiple locations.

## Decision

### D1 - Unified `Diagnostic` struct replaces all three error structs

A single `Diagnostic` struct is the canonical representation for all
compiler diagnostics, from lexing through codegen:

```cpp
enum class Severity : std::uint8_t {
    Error,
    Warning,
    Note,
    Help,
};

struct SourceSpan {
    int line = 0;
    int column = 0;
    int length = 1;
};

struct DiagnosticLabel {
    SourceSpan span;
    std::string message;
};

struct FixIt {
    SourceSpan span;
    std::string replacement;
};

struct Diagnostic {
    std::string code;              // e.g. "K4001"
    Severity severity;             // Error / Warning / Note / Help
    std::string message;           // primary message
    std::vector<DiagnosticLabel> labels;  // primary + secondary spans
    std::vector<FixIt> fixes;      // suggested replacements
};
```

`ParseError`, `TypeError`, `CompileError`, and `CompileWarning` are
replaced by `Diagnostic`. Each pipeline stage produces
`std::vector<Diagnostic>`. The `severity` field distinguishes errors
from warnings; there is no longer a separate struct per severity level.

`Note` and `Help` are **secondary diagnostics** -- they are always
attached to a primary `Error` or `Warning` as additional labels or
help text, never emitted standalone with their own error code.

### D2 - Error code scheme: `K` + category block

Error codes are stable identifiers of the form `Kxxxx` (4 digits for
categories 0-9) or `Kxxxxx` (5 digits for categories 10+). The code
identifies the **semantic category**, not the message text -- message
text may change across versions, but a code once assigned is not
reused for a different meaning.

| Range | Category |
|-------|----------|
| `K0xxx` | Lexical, syntax, and parse |
| `K1xxx` | Names, declarations, and scope |
| `K2xxx` | Type system and type inference |
| `K3xxx` | Value category, mutability, and assignment |
| `K4xxx` | Ownership, transfer, and move |
| `K5xxx` | Borrow, reference, and lifetime |
| `K6xxx` | Initialization, destruction, and resource lifecycle |
| `K7xxx` | Functions, parameters, and calls |
| `K8xxx` | Control flow and reachability |
| `K9xxx` | Expressions and operators |
| `K10xxx` | Optional, fallible cast, and unwrap |
| `K11xxx` | Arrays, indexing, slicing, and dimensions |
| `K12xxx` | Generics and compile-time evaluation |
| `K13xxx` | Pattern matching and exhaustiveness |
| `K14xxx` | FFI, ABI, and external symbols |
| `K15xxx` | Modules, imports, and visibility |
| `K16xxx` | Concurrency, atomics, and thread safety |
| `K17xxx` | Attributes, built-in capabilities, and language constraints |
| `K18xxx` | Unused code, style, and suspicious patterns |
| `K19xxx` | Compiler limits and internal diagnostics |

The full per-code table (code, default severity, description, example)
is maintained as a separate document (`docs/diagnostics.md`) and is the
authoritative reference. This ADR establishes the scheme and the first
batch of codes; new codes are added by appending to that document
without a new ADR.

### D3 - First batch: 20 core codes

The initial implementation assigns codes to the 20 most common
diagnostics already emitted by the compiler. These are the codes that
appear in user-visible output first; the rest migrate incrementally.

| Code | Severity | Current message pattern | Category |
|------|----------|------------------------|----------|
| `K1001` | error | `Unknown type '...'` / `Undeclared variable '...'` | Name resolution |
| `K1002` | error | `Duplicate declaration of '...'` | Name resolution |
| `K2001` | error | `Cannot assign ... to variable of type ...` | Type mismatch |
| `K2008` | error | `Type '...' recursively contains itself` | Recursive layout |
| `K3002` | error | `Cannot modify const variable '...'` | Mutability |
| `K4001` | error | `Variable '...' was transferred` | Ownership |
| `K4005` | error | `Field moved, cannot use whole object` | Partial move |
| `K5001` | error | `Conflicting borrow of '...'` | Borrow |
| `K5004` | error | `Borrow outlives referent` | Lifetime |
| `K6001` | error | `Variable '...' may be uninitialized` | Definite assignment |
| `K7001` | error | `No matching overload for '...'` | Call resolution |
| `K7007` | error | `Not all paths return a value` | Return paths |
| `K7010` | warning | `Expression result is unused` | Unused result |
| `K8001` | warning | `Unreachable code` | Reachability |
| `K10001` | error | `Cannot use T? where T is required` | Optional |
| `K10002` | error | `Fallible cast result must be handled` | Fallible cast |
| `K11001` | error | `Array index out of bounds` | Array bounds |
| `K13001` | error | `Non-exhaustive match` | Exhaustiveness |
| `K15001` | error | `Module '...' not found` | Module resolution |
| `K18001` | warning | `Unused variable '...'` | Unused code |

### D4 - Secondary labels for multi-span diagnostics

Ownership, borrow, and definite-assignment errors often need to point
at two locations: the use site (primary) and the original
transfer/borrow/declaration site (secondary). The `DiagnosticLabel`
vector supports this:

```cpp
Diagnostic d;
d.code = "K4001";
d.severity = Severity::Error;
d.message = "variable 'a' was transferred and is no longer valid";
d.labels.push_back({use_span, ""});           // primary (no message)
d.labels.push_back({transfer_span, "value transferred here"});  // secondary
```

The first label is the primary span (rendered with `^`); subsequent
labels are secondary (rendered with `-`). The renderer determines
rendering from label order, not from a `primary` boolean -- the first
label is always primary.

### D5 - Fix-its

A `FixIt` is a suggested source replacement attached to a diagnostic.
Fix-its are only generated when the fix is **semantically reliable**
(applying it produces valid code that resolves the diagnostic). When
the fix is uncertain, the diagnostic carries `Help` text instead, not a
`FixIt`.

Fix-its are consumed by:
- The CLI renderer (displayed as a diff snippet).
- LSP (sent as `textDocument/codeAction` results).
- The formatter / refactorer (if the user accepts the suggestion).

### D6 - Warning groups and suppression

Warnings are organised into groups for selective control:

| Group | Codes | Description |
|-------|-------|-------------|
| `unused` | K18001–K18006 | Unused variables, parameters, fields, functions, types, expressions |
| `conversion` | K2005, K2015 | Lossy or redundant conversions |
| `ownership` | K4012, K4013 | Immediate-transfer and expensive-copy warnings |
| `borrow` | K5013, K5014 | Unnecessary mutable/any borrows |
| `style` | K0018, K0019, K3011, K9011 | Cosmetic and style warnings |
| `performance` | K4013, K7014, K11011 | Performance-impacting warnings |
| `ffi` | K14009, K14010, K14014 | FFI safety warnings |

CLI flags:

```
-Wunused              # enable all warnings in the group
-Wno-unused           # suppress all warnings in the group
-Werror               # treat all warnings as errors
-Werror=conversion    # treat only conversion warnings as errors
-Wno-K18001           # suppress a specific code
```

When no `-W` flags are given, all warnings are enabled by default.

### D7 - Rendering: human-readable default, JSON for tools

The **renderer** is separate from the diagnostic producers. The CLI
driver selects a renderer based on flags:

**Human-readable (default):**

```text
src/main.kl:34:11: error[K4001]: variable 'a' was transferred and is no longer valid
   |
31 | sink(a);
   |      - value transferred here
34 | inspect(a);
   |         ^ use of transferred value
   |
help: pass a shared borrow if the function does not need ownership
```

**JSON (`--diagnostics=json`):**

```json
[
  {
    "code": "K4001",
    "severity": "error",
    "message": "variable 'a' was transferred and is no longer valid",
    "labels": [
      {"line": 34, "column": 11, "length": 1, "message": ""},
      {"line": 31, "column": 6, "length": 1, "message": "value transferred here"}
    ],
    "fixes": [],
    "source": "src/main.kl"
  }
]
```

LSP diagnostics are produced from the same `Diagnostic` struct by the
LSP server, mapping `Error`/`Warning`/`Info`/`Hint` to LSP severity
values 1/2/3/4 respectively.

### D8 - Source file identification

`SourceSpan` currently carries only `line`/`column`/`length`. The
renderer needs to know **which file** the span refers to. The
`Diagnostic` struct carries source identity implicitly: each pipeline
stage knows which source file it is processing, and the driver attaches
the file path when rendering. For multi-file diagnostics (e.g. module
import errors), the `DiagnosticLabel` may carry a file path; if absent,
the primary file from the enclosing `Diagnostic` is assumed.

This avoids threading a `SourceId` / file-path string through every
`SourceSpan` in the common single-file case, while still supporting
multi-file diagnostics when needed.

### D9 - Cascade suppression

When a declaration fails (e.g. `Unknown type 'Foo'`), downstream uses
of `Foo` should not each produce a separate `K1001` error. The
checker maintains a set of **already-failed names**; a second use of
the same failed name within the same scope is suppressed. This keeps
error output focused on root causes, not cascades.

Cascade suppression is per-name, per-scope. A use in a different scope
or a genuinely different unknown name still produces an error.

### D10 - Migration strategy

The migration is incremental -- the compiler does not stop working
while diagnostics are being converted:

1. **Define `Diagnostic` and renderer** in a new
   `frontend/diagnostics/` module. The renderer handles the
   human-readable format and JSON format.
2. **Replace `TypeError` with `Diagnostic`** in the type checker. The
   `error_at()` helper gains an optional `code` parameter; existing
   call sites that don't pass a code produce `Diagnostic`s with an
   empty `code` string (rendered without the `[Kxxxx]` tag). This
   allows gradual code assignment.
3. **Replace `ParseError` with `Diagnostic`** in the parser. Same
   approach: `code` defaults to empty, assigned later.
4. **Replace `CompileError`/`CompileWarning` with `Diagnostic`** in the
   compiler backend.
5. **Assign codes** to the first batch (D3) and update the corresponding
   `error_at()` / `error()` call sites.
6. **Add `-W` flag parsing** to the CLI driver and wire it to the
   renderer's suppression logic.
7. **Add source-context rendering** (the `|` + `^` snippet format) to
   the human-readable renderer. This requires the renderer to read
   source file lines, which the driver provides.

Steps 1-4 are mechanical refactors that do not change user-visible
output. Steps 5-7 add the new capabilities incrementally.

### D11 - Non-goals

- **No semantic error recovery** beyond what the checker already does.
  The diagnostic system reports errors; it does not change what the
  checker accepts or rejects.
- **No error database file**. The code table lives in
  `docs/diagnostics.md` as a hand-maintained Markdown document. An
  auto-generated database can be introduced later if the code count
  grows large enough to warrant it.
- **No i18n / localized messages** in v1. All messages are in English.
  The `message` field is a format string with named placeholders, so
  translation could be layered on later without changing the struct.
- **No machine-readable fix-it application**. Fix-its are suggestions
  displayed to the user; the compiler does not automatically apply
  them. LSP clients may apply them via `textDocument/codeAction`.

## Consequences

### Positive

- Users can reference errors by stable codes in bug reports,
  documentation, and `-Wno-Kxxxx` suppression.
- Multi-span diagnostics (ownership, borrow) become readable with
  source context and secondary labels.
- LSP integration gets structured data (labels, fixes, severity)
  instead of parsing free-form stderr text.
- Warning groups give users control over noise without an all-or-nothing
  choice.
- The renderer is decoupled from producers, so format changes (JSON,
  short mode, machine-readable) are localized to one module.

### Risks and constraints

- **Migration cost**: replacing three error structs with one touches
  every `error_at()` call site in the checker, parser, and compiler.
  The incremental approach (D10) keeps this manageable but it is still
  a large mechanical refactor.
- **Error code stability**: once a code is assigned and users start
  referencing it in `-Wno-Kxxxx`, the code must not be reused for a
  different meaning. Codes for diagnostics that are later removed are
  retired, not recycled.
- **Source file access in renderer**: the human-readable renderer needs
  to read source lines to print context snippets. This requires the
  driver to keep source text in memory (or re-read from disk) during
  rendering. For the common case (a few files, a few errors) this is
  negligible; for large builds with many files it is a memory
  consideration.

## Acceptance criteria

- [ ] `Diagnostic` struct defined in `frontend/diagnostics/diagnostic.h`
- [ ] `TypeError`, `ParseError`, `CompileError`, `CompileWarning` all
      replaced by `Diagnostic`
- [ ] Human-readable renderer with source context (`|` + `^` format)
- [ ] JSON renderer (`--diagnostics=json`)
- [ ] First 20 error codes (D3) assigned and visible in output
- [ ] `-W` / `-Wno-` / `-Werror` flag parsing for warning groups
- [ ] At least one multi-span diagnostic (K4001 transfer) rendered with
      primary + secondary labels

## Dependencies

- [0006](0006-error-handling-unification.md) - established `CastError`
  as the built-in error type; K10xxx codes formalise the diagnostics
  for cast failures.
- [0028](0028-ownership-and-value-transfer.md) - ownership and borrow
  diagnostics (K4xxx, K5xxx) are the primary consumers of multi-span
  labels.
- [0030](0030-definite-assignment-and-literal-width-inference.md) -
  definite-assignment diagnostics (K6xxx) are already emitted as
  free-form messages; this ADR assigns them codes.

## References

- Current error structs: `compiler/frontend/parser/parser.h:18`
  (`ParseError`), `compiler/frontend/checker/type_checker.h:28`
  (`TypeError`), `compiler/backend/compiler/compiler.h:23`
  (`CompileError` / `CompileWarning`)
- Current rendering: `compiler/driver/kinglet/main.cc:446-532`
- Current `SourceLocation`: `compiler/frontend/ast/ast.h:60`
- Current `DiagnosticSeverity` enum: `compiler/frontend/checker/type_checker.h:28`
  (already has Error/Warning/Info/Hint, reused by this ADR)
- Full code table: `docs/diagnostics.md` (to be created)
