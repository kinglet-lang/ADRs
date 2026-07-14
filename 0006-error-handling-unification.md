# 0006 — Error Handling: ??, try, and Cast Unification

- **Status**: implemented
- **Proposed**: 2026-06-01
- **Completed**: 2026-06-02

## Context

Cast used `else` for fallback (`int(x) else 0`), conflicting visually with `if/else`. `??` and `try` were added as operators on Result types. Cast produces untyped `null` on failure, so Cast results cannot be operated on by `??`/`try`. The error-handling surface is fragmented.

## Decision

### D1: Remove `else` from Cast, adopt `??`

```kl
// Before
int(s) else 0

// After
int(s) ?? 0
```

`else` remains only for `if/else`. One fewer fallback syntax to learn.

### D2: Fallible casts produce `Result<T, CastError>`

```kl
int(s)        // → Result<Int, CastError>   (fallible, needs ?? or try)
float(42)     // → Float                     (infallible, no wrapper)
```

Infallible casts (int↔float, num→string) stay unwrapped. Fallible casts produce Result enum variants (`Ok(v)` / `Err(CastError::X)`) using the existing `CastError`, `IntResult`, `FloatResult` enums already registered in the checker.

### D3: `try` and `??` work transparently on Cast

```kl
int parse_or_zero(string s) {
  return int(s) ?? 0;        // fallible cast + fallback → int
}

IntResult parse_or_err(string s) {
  return try int(s);         // Ok(v) → v, Err(e) → early return
}
```

No new operators needed — `??` and `try` are generic over any Result-producing expression.

### D4: Bare fallible casts are a compile error

```kl
let result = int(s);   // Error: fallible cast must be consumed by ?? or try
```

Prevents silent error swallowing.

### Syntax surface (end state)

```kl
// Infallible casts — no wrapping, no ?? needed
float(42)             // 42.0
int(3.14)             // 3
string(42)            // "42"

// Fallible casts with fallback
int(s) ?? 0
int(s) ?? |e| { io::err(e.string()); 0 }

// Error propagation
try int(s)            // Ok → unwrap, Err → early return

// ?? and try on any Result type (not just Cast)
some_result ?? default
some_result ?? |e| handle_error(e)
try some_result
```

## Implementation

| Step | What | Files |
|------|------|-------|
| 1 | Pre-register `IntResult`, `FloatResult`, `CastError` enum metas | `bytecode.kl`, `compiler.kl` |
| 2 | Remove fallback field from `CastExpr` | `ast.kl` |
| 3 | Remove `else` parsing from Cast | `parser.kl` |
| 4 | CastTo emits Result enum variant (Ok/Err) for fallible casts | `compiler.kl` |
| 5 | Add `??` parser (`null_coalesce` at precedence between assignment and pipeline) | `ast.kl`, `parser.kl` |
| 6 | Add `try` parser (soft keyword in unary) | `ast.kl`, `parser.kl` |
| 7 | `??` compiler: Dup + EnumVariant + JmpFalse + fallback/Ok paths | `compiler.kl` |
| 8 | `try` compiler: StoreLocal + EnumVariant + JmpFalse + early Return/Ok paths | `compiler.kl` |
| 9 | Checker: fallibility matrix, Result-type resolution for `??`/`try` | `checker.kl` |
| 10 | Golden tests: cast with ??, try on cast, try on Result, ?? on Result | `tests/codegen/cases/` |

### Cast fallibility matrix

| From | To | Category |
|------|----|----------|
| int | float | infallible |
| float | int | infallible |
| int/float | string | infallible |
| string | int | fallible |
| string | float | fallible |

### AST changes

| Node | Change |
|------|--------|
| `CastExpr` | Remove `fallback` field. Only stores `target`, `value`. |
| `NullCoalesceExpr` | Unchanged — already supports `left ?? right` and `left ?? \|e\| right`. |
| `TryExpr` | Unchanged — `try expr`. |

### Opcode change

Extend `CastTo` to produce Result enum variants instead of `null`: on success push `Ok(value)` (variant 0 with payload), on failure push `Err(CastError::X)` (variant 1 with error variant index).

## Consequences

- `else` keyword scope reduced (only `if/else`).
- All error handling flows through two operators: `??` (recover) and `try` (propagate).
- Fallible casts are impossible to ignore silently.
- Existing code using `Cast ... else` must migrate to `??`.

## Amendments

### 2026-07-14 — Cast matrix corrected and widened (bootstrap [#115](https://github.com/kinglet-lang/bootstrap/pull/115))

The D2 fallibility matrix above only ever documented `int`/`float`/`string`;
it never covered `char`, `byte`, `bool`, `enum`, or `null` as cast sources,
and an implementation bug independently made the true behavior diverge from
what any doc claimed. `check_cast()` resolved `char`/`byte` targets through
the same branch used for canonical integer widths (`canonical_int_type_name`
maps `"char"` → `"int8"` and `"byte"` → `"uint8"`, both truthy), so the
dedicated char/byte source whitelist never actually ran — `Float` and `Enum`
sources silently cast to `char(...)`/`byte(...)` and produced a bare `Int`
type, not the intended `Char`/`Byte`. Fixed in bootstrap PR #115 by moving
the char/byte dispatch ahead of the integer-width branch.

The same PR audited the full source × target matrix against the runtime
lowering in `kinglet_rt_enum.cc` and opened three infallible conversions the
runtime already implemented but the checker rejected: `bool → int`,
`bool → float`, `char → float`, and `enum → float` (`enum → int` was already
supported). The corrected and completed matrix, superseding the D2 table
above:

| Source \ Target | int | float | string | char | byte |
|---|---|---|---|---|---|
| int | infallible | infallible | infallible | infallible | infallible |
| float | infallible | infallible | infallible | rejected | rejected |
| string | fallible | fallible | infallible | fallible | fallible |
| char | infallible | infallible | infallible | infallible | infallible |
| byte | infallible | infallible | infallible | infallible | infallible |
| bool | infallible | infallible | infallible | rejected | rejected |
| enum | infallible | infallible | rejected | rejected | rejected |
| null | rejected | rejected | infallible | rejected | rejected |

`byte` has no independent `TypeKind` — `byte_type()` is
`make_int_type("uint8")`, sharing `TypeKind::Int` with `int` — so it carries
the same source privileges as `int` for every target.

The remaining `rejected` cells are deliberate design boundaries, not gaps:
`char → float` and `bool → char/byte` would blur char/bool's semantic
identity into "small integer"; `enum → string` has no variant-name
stringification (the runtime only exposes the ordinal, and `string(enum)`
producing an ordinal string would read as if it produced the variant name);
`null → <numeric>` has no sensible numeric value (`null` is not `0`).
