# 0025 — Namespace-Qualified Type Names

- **Status**: implemented
- **Proposed**: 2026-07-10
- **Completed**: 2026-07-11

**Implementation**: [kinglet-lang/bootstrap#89](https://github.com/kinglet-lang/bootstrap/pull/89)
(commit `6293db3`) — parser accepts `::`-qualified type names in type
position, the checker canonicalizes qualified names through module aliases,
and imported struct/enum types resolve to the same type under either their
bare or qualified spelling. Covered by `tests/abi/qualified_type_name/`,
`tests/parser/cases/qualified_type_names_ast.kl`, and
`tests/sema/fail/qualified_type_unknown.kl`.

## Context

Kinglet already supports namespace-qualified value access in expression
position:

```kinglet
using io;
io::out.line("hello");
io::err.line("error");
```

However, type positions only accept a single unqualified identifier followed
by generic arguments, array suffixes, or nullable suffixes. The parser
therefore accepts:

```kinglet
Point p;
string[] names;
Box<int> box;
```

but not:

```kinglet
module::Type value;
```

This becomes a limitation as built-in modules grow beyond simple functions.
A module that wants to expose an opaque object type (for example, a resource
handle) cannot spell it as an unqualified name without polluting the global
type namespace and risking collisions with user-defined structs. The natural
spelling is a module-qualified type name, matching the existing
namespace-qualified value syntax.

This issue will appear for any standard module that needs named opaque
handle types, such as process handles, sockets, compiler objects, or other
resource wrappers. [0026](0026-standard-io-capability-model.md)
(`io::reader` / `io::writer`) and [0027](0027-filesystem-resource-api.md)
(`fs::file`) are the first concrete consumers of this ADR; they depend on
qualified type names to introduce new resource and capability types without
adding unqualified names to the global type namespace. This ADR itself
stays independent of any particular module.

## Decision

Introduce namespace-qualified type names in type position.

The grammar accepts one or more `::`-separated identifiers wherever a type
name is currently accepted:

```kinglet
io::Output out;
some.module::Type value;
```

Generic arguments, array suffixes, references, and nullable suffixes
continue to apply to the full qualified name:

```kinglet
module::Box<int> box;
io::Output? maybe_out;
some.module::Type[] values;
&module::Box<int> box_ref;
```

The AST may represent this initially as a flattened type name string such as
`"module::Type"` to minimize parser churn. A structured representation can
be introduced later if module-aware type resolution needs more precision.

Type checking resolves a qualified type name through the same module alias
mechanism used by qualified value access. For example, if `using parser.ast`
aliases a module path, `ast::Node` resolves through that alias rather than
as a global type named literally `ast::Node`.

Built-in modules may define built-in namespaced types, such as
`module::Type`. These names are not visible as an unqualified `Type` unless
a future import rule explicitly introduces that behavior.

## Non-goals

- This ADR does not introduce user-defined namespaces.
- This ADR does not change expression namespace access.
- This ADR does not define any specific module's API. New module types built
  on top of this ADR (e.g. `io::reader`/`io::writer` in
  [0026](0026-standard-io-capability-model.md) and `fs::file` in
  [0027](0027-filesystem-resource-api.md)) are covered by their own ADRs.
- This ADR does not require return-type-based overload resolution.
- This ADR does not require implicit imports of namespaced types.

## Consequences

- The parser must accept `::` in type expressions.
- The type checker must resolve qualified type names, including built-in
  module types and imported module types.
- Error messages should preserve the source spelling where possible:

  ```
  Unknown type 'module::Type'
  Module 'module' is not imported
  ```

  If built-in module types were allowed without a corresponding `using`,
  this ADR would need to state that explicitly. The preferred rule is to
  require a `using` declaration for consistency with namespace-qualified
  value access.
- The compiler must treat namespaced built-in types as real static types so
  method lookup can distinguish methods on one qualified type from another,
  e.g. `module::TypeA.method` from `module::TypeB.method`.

## Implementation notes

The current `parse_type_expr` consumes one identifier as the type name and
then checks for generic arguments, array suffixes, and nullable suffixes. It
should be extended to consume repeated `:: identifier` segments before
generic arguments are parsed.

The initial internal representation can be:

```cpp
TypeExpr{name = "module::Type"}
```

The type checker can split on `::` when resolving type names. This keeps the
first implementation small while preserving the option to introduce a
structured qualified-name AST later.

## Open questions

1. Should qualified type names require the module to be imported with
   `using`? The recommended answer is yes, matching existing
   namespace-qualified value access rules such as `io::out`.
2. Should user-defined module types become addressable with the same syntax
   in the same change, or should this first land only for built-in module
   types? The recommended answer is to support the general syntax now and
   implement built-in module types first, because the syntax should not be
   special-cased to any one module.
3. Should nullable qualified types require any extra grammar work beyond
   applying the existing `?` suffix to the full qualified name? The
   recommended answer is no; nullable suffixes should continue to apply
   after the complete type name is parsed.

## Dependencies

- None (this is language-layer infrastructure).

## References

- Depended on by [0026](0026-standard-io-capability-model.md) and
  [0027](0027-filesystem-resource-api.md), the first concrete consumers of
  qualified type names.
