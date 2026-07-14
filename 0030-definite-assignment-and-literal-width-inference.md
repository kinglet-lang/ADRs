# 0030 — Definite Assignment and Literal Width Inference

- **Status**: implemented
- **Proposed**: 2026-07-14
- **Completed**: 2026-07-14

## Context

The current bootstrap compiler accepts local declarations without an initializer and lowers them through `emit_default_value()` in `compiler/backend/compiler/compiler.cc`.

The current behavior is inconsistent at the language level:

- `T?` lowers to the runtime null sentinel. This is intentional: null is the value-level state of an Optional.
- `string` lowers to the empty string.
- dynamic arrays lower to an empty array.
- maps lower to an empty map.
- structs lower to a heap struct whose fields are filled with the same null sentinel.
- scalars and enums fall through to the same null sentinel.

The null sentinel is currently the concrete `kl_h` bit pattern `0xFFFB000000000000` (`KL_NULL_SENTINEL` in `kinglet_rt_value.h`). This bit pattern has two meanings today:

- a real, user-visible null state for Optional and error paths;
- an implementation placeholder for ordinary non-Optional locals that were declared but not initialized.

Those two meanings should not share the same language semantics. A plain `int a;` can currently be read, printed as `null`, assigned into `int?`, or used in arithmetic where the sentinel bit pattern is interpreted as an integer. This is not a desirable default value model. It also lets uninitialized ordinary values leak into Optional null checks.

Separately, fixed-width integer declarations are too noisy today. The current checker treats a suffixless integer literal as default `int` before considering the target type, so this fails:

```kl
int32 i = 1;
```

Users must write:

```kl
int32 i = 1i32;
```

That is stricter than the intended C-family ergonomics for literal initialization. A suffixless literal in a typed initializer should be allowed when the value fits the target integer type. This should not become general implicit narrowing of non-literal values.

## Decision

### D1 — Local variables have definite assignment

A local variable declared without an initializer is not readable until the checker can prove it has been assigned on every path reaching the read.

```kl
int main() {
  int a;
  return a;        // error: variable 'a' may be uninitialized
}
```

```kl
int main() {
  int a;
  a = 1;
  return a;        // OK
}
```

For branches, assignment must occur on every live branch:

```kl
int main(bool cond) {
  int a;
  if cond {
    a = 1;
  }
  return a;        // error: the false path leaves 'a' uninitialized
}
```

```kl
int main(bool cond) {
  int a;
  if cond {
    a = 1;
  } else {
    a = 2;
  }
  return a;        // OK
}
```

Loops are conservative in the first implementation. Assignment inside a loop body does not make a variable definitely assigned after the loop, because the loop might execute zero times.

### D2 — Only explicitly default-constructible types are initialized by declaration alone

A declaration without an initializer is immediately readable only when the type has a language-level default state.

The initial defaultable set is:

- `T?`, defaulting to `null`;
- `string`, defaulting to `""`;
- dynamic arrays, defaulting to an empty array;
- maps, defaulting to an empty map;
- structs with an applicable zero-argument `@init` path, once that constructor path is modeled as part of initialization.

Everything else starts as unassigned:

- integer types, including `int8`, `int16`, `int32`, `int64`, and unsigned variants;
- floating-point types, including `float`, `float32`, `double`, and `float64`;
- `bool`, `char`, and `byte`;
- enums;
- structs without an applicable zero-argument initializer.

This means `int a;` is not a declaration of zero. It is a declaration of a local name whose value must be assigned before use.

### D3 — Null is not the default value of non-Optional types

The runtime null sentinel belongs to Optional, fallible cast, and error propagation paths. It must not be observable as the default value of an ordinary non-Optional local.

The checker should prevent reads of unassigned locals. Lowering should no longer rely on `LoweringOp::Null` as the language-level default for ordinary non-Optional locals.

A debug-only uninitialized sentinel may still be used internally to catch checker bugs, but observing it at runtime should be treated as a compiler bug, not as valid program behavior.

### D4 — Struct fields participate in definite assignment

Struct variables without an initializer may be initialized field-by-field. A field read is valid only if that field has been assigned. A whole-struct read is valid only if every field has been assigned, or the struct was initialized by a literal or constructor path.

```kl
struct Point {
  int x;
  int y;
}

int main() {
  Point p;
  p.x = 1;
  return p.x;      // OK
}
```

```kl
int main() {
  Point p;
  p.x = 1;
  return p.y;      // error: field 'p.y' may be uninitialized
}
```

```kl
int dist(Point p);

int main() {
  Point p;
  p.x = 1;
  dist(p);         // error: 'p' is only partially initialized
}
```

This should reuse the existing place/path machinery used by borrow checking where practical. The first implementation may choose a simpler whole-variable-only rule if field-level tracking proves too large, but the intended language design is field-aware definite assignment.

### D5 — Suffixless integer literals may adopt an explicit integer target type

A suffixless integer literal used directly as a typed initializer may adopt the target integer type when the value fits that type.

```kl
int32 i = 1;       // OK, equivalent to int32 i = 1i32;
uint16 port = 443; // OK, value fits uint16
```

Out-of-range literals are rejected at compile time:

```kl
int8 x = 300;      // error: integer literal out of range for int8
```

This is not general implicit narrowing. It applies only to a bare integer literal with no width suffix.

The following remains rejected:

```kl
int32 i = some_int64_value;  // error: non-literal narrowing is not implicit
int32 j = 1i64;              // error: explicit suffix says int64
```

The first implementation applies this rule to variable declaration initializers. Function call arguments, struct literal fields, index assignment, and ordinary assignment expressions may adopt the same target-typed literal rule later, but they are not required for the initial decision.

### D6 — Suffixless literals are still default `int` without a target type

The target-typed literal rule does not change the default type of a standalone integer literal. If no target type exists, a suffixless integer literal remains default `int`.

This preserves current behavior for `auto`:

```kl
auto x = 1;        // x is int, not int8 or int32
```

It also preserves the current binary-expression model unless a later ADR explicitly extends target typing into expression trees.

## Implementation sketch

### Checker state

Extend `TypeChecker::VarInfo` with an initialization state.

Possible shape:

```cpp
enum class InitState {
  Unassigned,
  Initialized,
  PartiallyInitialized,
};
```

Each local tracks:

- its current initialization state;
- initialized field paths for partially initialized structs;
- the existing mutable, used, transferred, and location metadata.

### Declaration rules

When checking a `VarDeclStmt`:

- initializer present: check the initializer and mark initialized;
- no initializer and type is defaultable: mark initialized and record the default path;
- no initializer and type is not defaultable: mark unassigned;
- `auto` without initializer remains an error, because there is no source type to infer from.

### Read rules

When checking an identifier or field access:

- reading an unassigned variable is an error;
- reading an unassigned field path is an error;
- reading a partially initialized struct as a whole is an error;
- reading initialized paths is allowed.

### Assignment rules

Assignments update definite-assignment state:

- `a = expr` marks `a` initialized;
- `p.x = expr` marks field path `p.x` initialized;
- when every field path of a struct is initialized, the struct becomes initialized as a whole.

Transfer state and initialization state are separate. A variable can be initialized but later transferred; the existing transfer checks still govern post-transfer use.

### Control-flow merge

For `if` with an `else`, merge states by intersection: a variable or field is definitely initialized after the branch only if it is initialized on every live branch.

For `if` without `else`, the missing branch preserves the incoming state.

For loops, the first implementation is conservative: assignments inside the loop body are not promoted to the state after the loop unless they were already true before the loop.

### Lowering

Lowering should emit default values only for types with a language-level default state: Optional null, empty string, empty array, empty map, and explicit constructor paths.

For locals that are declared but not yet assigned, lowering should allocate a local slot without treating `Null` as the semantic value. If the backend requires an initialized slot, it may use an internal uninitialized sentinel, but user code must never observe it if the checker is correct.

### Literal target typing

Add a helper for checking a bare integer literal against a target integer type:

```cpp
std::optional<Type> try_check_target_typed_int_literal(
    const ast::Expr &expr,
    const Type &target_type,
    SourceLocation loc);
```

For `VarDeclStmt` with an explicit non-`auto` type:

- resolve the declared type first;
- if the initializer is a bare `IntLiteralExpr` with empty `width_suffix` and the declared type is an integer type, check `integer_fits_width(lit.value, int_width_info(var_type))`;
- if it fits, treat the initializer type as the declared type;
- if it does not fit, emit an out-of-range diagnostic;
- otherwise fall back to the existing `check_call_arg_type()` and `types_assignable()` path.

## Consequences

Programs that read locals before assignment become compile-time errors instead of observing null sentinel behavior. This is a deliberate breaking change.

Existing code that relied on `string s;`, `T[] a;`, or `{K: V} m;` as empty values continues to work because those types have explicit language-level defaults.

Existing code that uses `int a; a = 1;` continues to work if every read happens after assignment.

Existing code that uses `int a; return a;` becomes an error.

`int32 i = 1;` becomes accepted, but `int32 i = some_int_value;` remains rejected. The change improves literal ergonomics without weakening ordinary assignment rules.

The current null sentinel collision with the full `int64` value space remains a separate runtime representation issue. This ADR removes the uninitialized-read path that exposes the sentinel accidentally, but it does not redesign `kl_h` or guarantee that every possible `int64` bit pattern is distinguishable from internal tags. A future runtime ABI ADR may revisit that separately.

## Non-goals

This ADR does not add `float16`. Half-precision floating point remains deferred until Kinglet has a concrete GPU, tensor, packed-binary-format, or foreign-ABI use case that justifies it.

This ADR does not add general implicit narrowing. Only bare suffixless integer literals in a target-typed context are relaxed.

This ADR does not require target typing through arbitrary expression trees. A future extension may decide whether compound constant expressions should also be accepted when they are compile-time-known and fit the target type.

This ADR does not redesign Optional representation or the `kl_h` wire format.

## References

- Runtime value tags: `bootstrap/runtime/kinglet_rt_value.h`
- Current default emission: `bootstrap/compiler/backend/compiler/compiler.cc`, `emit_default_value()`
- Current local declaration lowering: `bootstrap/compiler/backend/compiler/compiler.cc`, `VarDeclStmt` handling
- Current integer literal typing: `bootstrap/compiler/frontend/checker/type_checker.cc`, `check_int_literal()`
- Current binary literal promotion: `bootstrap/compiler/frontend/checker/type_checker.cc`, `try_promote_integer_binary()` call sites
- Related design: [0029 — Value Representation and Memory Layout](0029-value-representation-and-memory-layout.md)
