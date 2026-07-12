# 0026 — Standard I/O Capability Model

- **Status**: draft
- **Proposed**: 2026-07-10
- **Revised**: 2026-07-11

Depends on: [0025](0025-namespace-qualified-type-names.md) (namespace-qualified
type names), and the language's existing `concept` mechanism.

## Context

Kinglet's `io` module is currently a small set of hardcoded intrinsics, not a
general-purpose I/O library. Today:

- `using io;` (or `using namespace io;`) unlocks three built-in objects:
  `io::out`, `io::err`, `io::in`.
- Each object exposes a fixed, hand-coded member set defined in
  `io_intrinsics::ostream_members()` / `io_intrinsics::istream_members()`
  (`frontend/module/io_intrinsics.{h,cc}`): `io::out.line(...)`,
  `io::out.flush()`, `io::err.line(...)`, `io::err.flush()`, `io::in.secret(...)`.
- The type checker and the compiler backend both special-case these three
  object names and their members directly (`type_checker.cc` and `compiler.cc`,
  the `namespace_name == "io"` branches).

This intrinsic-only design has no generic capability surface. There is no way
to write code that accepts "something bytes can be read from" or "something
bytes can be written to" without hardcoding `io::out` / `io::err` / `io::in`
by name. Every future I/O-producing module (filesystem, network, process
pipes, in-memory buffers) would otherwise need its own bespoke read/write
surface, and generic algorithms (parsers, copiers, codecs) could not be
written once against a shared contract.

### Why this ADR was reframed

An earlier draft of this ADR modeled `io::reader` / `io::writer` as `pub struct`
types with hand-maintained member tables (`io_intrinsics::reader_members()` /
`writer_members()`), and added `io::in.reader()` / `io::out.writer()` accessors
to bridge the existing stdio objects into those types.

That approach is superseded. It fought the language's own generic-dispatch
mechanism and produced empty wrapper objects with no state of their own. An
accessor like `file.reader()` that only rewraps an existing resource, adding no
new state, behavior, or lifetime, has no reason to exist: `parse(file)` is
strictly better than `parse(file.reader())` when `fs::file` already satisfies
the reader capability directly.

The correct model separates two distinct concepts:

- **Resource** answers *what is it?* Examples: `fs::file`, `net::tcpstream`,
  `proc::pipe`, `mem::buffer`.
- **Capability** answers *what can it do?* Examples: `io::reader`, `io::writer`.

A capability is not an object. It is a constraint on any type that provides the
required operations. Kinglet already has the language machinery for this: the
`concept` mechanism, which is satisfied structurally by free functions and
dispatched through UFCS (`obj.method()` desugars to `method(obj, ...)`). This
ADR builds `io::reader` / `io::writer` on that mechanism instead of inventing a
parallel struct + member-table dispatch path.

## Decision

### D1 — `io::reader` and `io::writer` are capabilities (concepts), not object types

```kinglet
concept reader<T> {
  uint64 read(T self, &mut byte[] buffer);
}

concept writer<T> {
  uint64 write(T self, byte[] data);
}
```

`read` may return fewer bytes than the buffer size. A return value of `0` means
end of input. This matches the read contract used by files, sockets, pipes, and
any other partial-read source; requiring `read` to always fill the buffer would
make it unusable for interactive or streaming sources.

`write` may write fewer bytes than requested. Callers that need all-or-nothing
semantics use a convenience free function (`all(writer, data)`), not a different
primitive contract.

A type satisfies a capability by providing the corresponding free function whose
first parameter is that type. For example, `fs::file` satisfies `io::reader` by
providing:

```kinglet
uint64 read(fs::file self, &mut byte[] buffer) { ... }
```

No `satisfies` keyword is needed. Satisfaction is checked structurally at the
point a concrete type is passed into a capability-typed parameter, using the
existing concept mechanism. UFCS lets callers spell the call as
`resource.read(buffer)` even though `read` is a free function.

### D2 — Generic algorithms consume capabilities through concept-typed parameters

```kinglet
int parse(reader input) {
  byte[] buffer = [];
  buffer.resize(4096, 0);
  uint64 n = input.read(buffer);
  // ...
  return 0;
}
```

Any resource that satisfies `reader` can be passed directly:

```kinglet
parse(file);            // fs::file
parse(socket);          // net::tcpstream (future)
parse(pipe);            // proc::pipe (future)
```

The algorithm does not know or care where the bytes come from. This is the
already-working concept-generic dispatch path: a function whose parameter type
is a concept name is monomorphized per concrete argument type, and calls on that
parameter resolve to the concrete type's free functions.

### D3 — Resources keep their own domain operations; capabilities do not absorb them

A capability describes only the shared byte-movement contract. Resource-specific
operations stay on the resource and are not part of `io::reader` / `io::writer`:

```kinglet
file.seek(0);      // fs::file domain operation, not a reader/writer method
file.size();       // fs::file domain operation
file.close();      // fs::file lifetime operation
```

The capability/resource split keeps generic algorithms (which only need
`read` / `write`) decoupled from resource identity (which carries seek, size,
lifetime, and so on). See [0027](0027-filesystem-resource-api.md) for how
`fs::file` layers domain operations on top of the reader/writer capabilities.

### D4 — No accessor wrappers; adapters exist only when they add state

There are deliberately no `io::in.reader()` / `io::out.writer()` /
`fs::file.reader()` accessors. A resource that satisfies a capability *is* usable
wherever that capability is required, with no conversion step.

A new object is introduced only when it carries genuinely new state, behavior,
or lifetime. Buffering, decompression, and limiting are the motivating cases:

```kinglet
parse(buffered(file));       // buffered_reader owns a buffer
parse(gzip_reader(file));    // gzip::reader owns decompression state
```

These adapter types are themselves resources that satisfy `io::reader` /
`io::writer`, so they compose. This ADR does not define any adapter; it only
establishes that adapters, not accessors, are the mechanism for added state.

### D5 — `io::in` / `io::out` / `io::err` are unchanged and are not retrofitted here

The existing stdio objects keep their current behavior and their current
intrinsic implementation:

```kinglet
io::out.line("hello");
io::out.flush();
io::err.line("failed: {}", msg);
string secret = io::in.secret();
```

No existing program breaks.

Critically, this ADR does **not** make `io::out` / `io::err` / `io::in` satisfy
`io::writer` / `io::reader`. In the current compiler these are not real typed
values at all: the checker resolves them as `native_fn` placeholder types at
call sites, not as struct-shaped values with a type identity that could satisfy
a concept. Promoting them to first-class capability-satisfying values is a
larger change to the checker and backend and is deferred to a future ADR. Until
then, the stdio objects remain a parallel, convenience-only surface, and the
first concrete satisfier of these capabilities is `fs::file` in
[0027](0027-filesystem-resource-api.md).

### D6 — Capabilities are single-type-parameter concepts

`io::reader` / `io::writer` each take exactly one type parameter (the concrete
resource type). This matches the concept mechanism's supported shape: a concept
with more than one type parameter is not currently checked for satisfaction.
Multi-type-parameter capabilities (for example, a transform keyed on both an
input and an output type) are out of scope for this ADR.

## Non-goals

- This ADR does not implement `fs::`, `net::`, or `proc::`. Those are covered by
  their own ADRs ([0027](0027-filesystem-resource-api.md) for filesystem).
- This ADR does not make `io::in` / `io::out` / `io::err` real values, and does
  not retrofit them to satisfy `io::reader` / `io::writer`. That is deferred.
- This ADR does not define `io::stream` or any combined read+write (duplex)
  capability. Duplex resources (for example a future `net::tcpstream`) satisfy
  both `io::reader` and `io::writer` independently.
- This ADR does not define buffering, decompression, or limiting adapters
  (`buffered`, `gzip::reader`, `limited`). It only establishes that such
  adapters are the mechanism for added state.
- This ADR does not add a `usize` type. Byte counts and sizes use the existing
  `uint64`; read/write buffers use the existing `byte[]` (mutable buffers as
  `&mut byte[]`). A pointer-width `usize` alias, if wanted, is a separate
  type-system decision.
- This ADR does not introduce asynchronous I/O of any kind.

## Consequences

- `io::reader` / `io::writer` are concepts resolvable as qualified names via
  [0025](0025-namespace-qualified-type-names.md), used in type position for
  generic algorithm parameters.
- Capability satisfaction and dispatch reuse the existing concept + free
  function + UFCS mechanism. No struct member tables (`reader_members()` /
  `writer_members()`) and no second dispatch path are introduced; the earlier
  draft's `io_intrinsics` extension plan is dropped.
- Any resource module (`fs::file`, and later network/process resources) satisfies
  the capabilities by providing `read` / `write` free functions, rather than
  inventing its own read/write member names.
- Generic algorithms are written once against `io::reader` / `io::writer`
  instead of once per resource type.
- `io::out` / `io::err` / `io::in` continue to work exactly as today, as a
  separate intrinsic convenience surface, until a future ADR promotes them to
  real capability-satisfying values.

## Rationale

Splitting capability (`reader` / `writer`, "what can I do with this") from
resource identity (`fs::file`, `net::tcpstream`, "what is this") is a
well-established pattern in systems languages Kinglet's design already leans
toward stylistically (Rust's `Read` / `Write` traits, Zig's `std.Io.Reader` /
`std.Io.Writer`). It lets [0027](0027-filesystem-resource-api.md) define
`fs::file` with filesystem-specific operations (`size`, `seek`, `sync`) while
handing off byte movement to the generic `io::reader` / `io::writer`
capabilities, so parsers and copiers do not need to know about files at all.

Building the capabilities on the existing `concept` mechanism, rather than on a
new struct + member-table dispatch path, means no new language feature is
required: the frontend and backend machinery for concept-typed parameters,
structural satisfaction, and UFCS dispatch already exists. Rejecting empty
accessor wrappers (`file.reader()`) keeps the model honest: a resource that can
already `read` is already a reader.

## Dependencies

- [0025](0025-namespace-qualified-type-names.md) — required for `io::reader` /
  `io::writer` to be usable as qualified type names.
- The language's `concept` mechanism (already implemented), including
  concept-typed parameter dispatch through free functions and UFCS.

## References

- Current implementation: `compiler/frontend/module/io_intrinsics.{h,cc}`,
  `compiler/frontend/checker/type_checker.cc` (`io` namespace branches and the
  `concept` machinery), `compiler/backend/compiler/compiler.cc`
  (`out`/`err`/`in` codegen dispatch and concept-generic monomorphization).
- Depended on by [0027](0027-filesystem-resource-api.md) (`fs::file` as the
  first concrete satisfier of `io::reader` / `io::writer`).
