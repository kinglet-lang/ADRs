# 0026 — Standard I/O Capability Model

- **Status**: draft
- **Proposed**: 2026-07-10

Depends on: [0025](0025-namespace-qualified-type-names.md), Namespace-qualified type names

## Context

Kinglet's `io` module is currently a small set of hardcoded intrinsics, not a
general-purpose I/O library. Today:

- `using io;` (or `using namespace io;`) unlocks three built-in objects:
  `io::out`, `io::err`, `io::in`.
- Each object exposes a fixed, hand-coded member set defined in
  `io_intrinsics::ostream_members()` / `io_intrinsics::istream_members()`
  (`frontend/module/io_intrinsics.{h,cc}`):
  - `io::out.line(...)`, `io::out.flush()`
  - `io::err.line(...)`, `io::err.flush()`
  - `io::in.secret(...)`
- The type checker special-cases these three object names and their member
  names directly (`type_checker.cc`, e.g. the `ns_access.namespace_name ==
  "io"` branches and the `io_intrinsics` lookups).
- The compiler backend special-cases the same three names again for codegen
  dispatch (`compiler.cc`, e.g. `callee_id->name == "out"` / `"err"` /
  `"in"`, and the corresponding `.line` / `.flush` / `.secret` branches).

This intrinsic-only design has two structural problems.

First, `io` has no generic capability types. There is no way to write code
that accepts "something bytes can be read from" or "something bytes can be
written to" without hardcoding `io::out` / `io::err` / `io::in` by name.
Every future I/O-producing module (filesystem, network, process pipes,
in-memory buffers) would otherwise need its own bespoke read/write surface,
and generic algorithms (parsers, copiers, codecs) could not be written once
against a shared contract.

Second, growing the intrinsic table does not scale. Each new stdio member
requires a synchronized change across `io_intrinsics`, the type checker, and
the compiler backend. This is acceptable for three fixed objects, but it is
the wrong foundation for a broader I/O module family (`fs::`, future
`net::`, `proc::`).

This ADR does not replace `io::out` / `io::err` / `io::in`. It introduces
`io::reader` and `io::writer` as generic capability types beneath them, and
defines how the existing stdio objects relate to those capabilities. This is
the necessary foundation for [0027](0027-filesystem-resource-api.md)
(`fs::file` exposing `io::reader` / `io::writer`) and for any future network
or process I/O module.

## Decision

### D1 — `io::reader` and `io::writer` are the generic capability types

```kinglet
pub struct reader {
  usize read(mut byte[] buffer);
}

pub struct writer {
  usize write(byte[] data);
}
```

`read` may return fewer bytes than the buffer size. A return value of `0`
means end of input. This matches the read contract used by files, sockets,
pipes, and any other partial-read source; requiring `read` to always fill
the buffer would make it unusable for interactive or streaming sources.

`write` may write fewer bytes than requested. Callers that need
all-or-nothing semantics use a convenience wrapper (`writer.all(data)`), not
a different primitive contract.

Convenience methods layered on the core contract:

```kinglet
pub struct reader {
  usize read(mut byte[] buffer);
  void exact(mut byte[] buffer);
  string line();
  bool end();
}

pub struct writer {
  usize write(byte[] data);
  void all(byte[] data);
  void write(string text);
  void line(string text);
  void flush();
}
```

These are the minimum needed to express text line I/O (already required by
`io::out.line`) and binary exact-read (needed by future `fs::file` binary
consumers), without inventing a larger surface than this ADR needs.
Additional convenience methods (`until`, `lines`, `read(limit)`, buffering
adapters, `io::copy`) are deferred; see Non-goals.

### D2 — Existing `io::in` / `io::out` / `io::err` are retained and gain capability accessors

The existing stdio objects keep their current member set and current
behavior:

```kinglet
io::out.line("hello");
io::out.flush();
io::err.line("failed: {}", msg);
string secret = io::in.secret();
```

No existing program breaks. This ADR adds capability accessors:

```kinglet
io::reader input = io::in.reader();
io::writer output = io::out.writer();
io::writer error = io::err.writer();
```

`io::out` and `io::err` are conceptually writers; `io::in` is conceptually a
reader. The accessors expose that relationship explicitly instead of
leaving it implicit in the intrinsic table.

### D3 — `io::stream` is reserved but not defined by this ADR

A combined read+write capability (`io::stream`) is a plausible future type
for duplex channels. This ADR does not define it. Domain modules introduce
their own duplex types when the domain has real duplex semantics (e.g. a
future `net::tcpstream`); they are not required to wait on a generic
`io::stream`. See Non-goals.

### D4 — `io_intrinsics` is refactored incrementally, not replaced

The existing `io_intrinsics::Member` table mechanism (shared by the checker
and the backend so a new intrinsic type-checks and compiles from one
registration site) is sound and is kept. This ADR extends it with reader and
writer member tables rather than introducing a second, parallel dispatch
mechanism:

- `io_intrinsics::reader_members()` — `read`, `exact`, `line`, `end`
- `io_intrinsics::writer_members()` — `write`, `all`, `line`, `flush`

`io::in.reader()` / `io::out.writer()` / `io::err.writer()` return values
that dispatch against these new tables. The existing `ostream_members()` /
`istream_members()` tables and their checker/backend call sites are
unchanged.

## Non-goals

- This ADR does not implement `fs::`, `net::`, or `proc::`. Those are
  covered by their own ADRs ([0027](0027-filesystem-resource-api.md) for
  filesystem).
- This ADR does not define `io::stream`. The name is reserved; the type is
  deferred until a concrete duplex use case needs it.
- This ADR does not add buffering adapters (`io::buffer`), combinators
  (`io::limit`, `io::chain`, `io::tee`, `io::count`), or `io::copy`. These
  are useful but are not required to unblock `fs::file` in
  [0027](0027-filesystem-resource-api.md); they can be proposed once a
  concrete consumer needs them.
- This ADR does not add `io::seek`. Seeking is resource-specific
  (`fs::file.seek(...)`) and is defined, if at all, in the ADR that
  introduces the seekable resource.
- This ADR does not change `io::out.line` / `io::out.flush` / `io::in.secret`
  behavior or remove them.
- This ADR does not introduce asynchronous I/O of any kind.

## Consequences

- `io::reader` and `io::writer` become real static types, resolvable in type
  position via [0025](0025-namespace-qualified-type-names.md).
- `io::in`, `io::out`, `io::err` remain the primary user-facing stdio
  surface; capability accessors are additive.
- Any future resource module (`fs::file`, and later network/process
  resources) can expose `.reader()` / `.writer()` returning these types
  instead of inventing its own read/write member names.
- Generic algorithms can be written once against `io::reader` / `io::writer`
  instead of once per resource type.
- The type checker and compiler backend gain two new small dispatch tables
  (`reader_members()` / `writer_members()`) following the existing
  `io_intrinsics` pattern; no new dispatch mechanism is introduced.

## Rationale

Splitting capability (`reader` / `writer`, "what can I do with this") from
resource identity (`fs::file`, `net::tcpstream`, "what is this") is a
well-established pattern in systems languages that Kinglet's design already
leans toward stylistically (Rust's `Read` / `Write` traits, Zig's
`std.Io.Reader` / `std.Io.Writer`). It lets [0027](0027-filesystem-resource-api.md)
define `fs::file` as a filesystem resource with filesystem-specific
operations (`size()`, `stat()`, `sync()`) while still handing off to generic
`io::reader` / `io::writer` for the actual byte movement, so parsers and
copiers do not need to know about files at all.

Keeping `io::in` / `io::out` / `io::err` exactly as they are, and adding
accessors rather than replacing the objects, avoids a breaking change to the
only I/O surface Kinglet programs can use today.

## Dependencies

- [0025](0025-namespace-qualified-type-names.md) — required for `io::reader`
  / `io::writer` to be usable as qualified type names.

## References

- Current implementation: `compiler/frontend/module/io_intrinsics.{h,cc}`,
  `compiler/frontend/checker/type_checker.cc` (`io` namespace branches),
  `compiler/backend/compiler/compiler.cc` (`out`/`err`/`in` codegen
  dispatch).
- Depended on by [0027](0027-filesystem-resource-api.md) (`fs::file.reader()`
  / `.writer()`).
