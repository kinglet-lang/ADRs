# 0027 — Filesystem Resource API

- **Status**: draft
- **Proposed**: 2026-07-10

Depends on: [0025](0025-namespace-qualified-type-names.md) (qualified type
names), [0026](0026-standard-io-capability-model.md) (`io::reader` /
`io::writer`)

## Context

Kinglet's `fs` module is currently three hardcoded intrinsics, not a
filesystem API:

```kinglet
using fs;

string data = fs::__read(path);
fs::__write(path, data);
string[] entries = fs::__listdir(path);
```

Concretely, today:

- `fs::__read`, `fs::__write`, `fs::__listdir` are the only members the type
  checker recognizes on the `fs` namespace (`type_checker.cc`, the
  `ns_callee->namespace_name == "fs"` branch that matches these three names
  literally and hand-validates their argument counts and types).
- Each lowers directly to one runtime call: `kl_native_fs_read`,
  `kl_native_fs_write`, `kl_native_fs_listdir` (`runtime/kinglet_rt_native.cc`,
  declared in `runtime/kinglet_rt_value.h`, referenced by name in the LLVM
  lowerer at `compiler/backend/codegen/llvm/llvm_function_lowerer.cc`).
- `fs::__read` reads an entire file into one `string` and `fs::__write`
  writes an entire `string` in one call. There is no open handle, no
  incremental I/O, no directory metadata beyond a name listing, and no
  existence check.
- The `__` prefix is a compiler-intrinsic naming convention, not a
  public-API convention (compare `io::out`, `io::in`, which are
  unprefixed). The current `fs` surface looks, and is documented internally
  as, a bootstrapping shim rather than a library.

This ADR refactors that shim into a public filesystem module, in the same
capability style established by [0026](0026-standard-io-capability-model.md):
filesystem resources (`fs::file`) expose filesystem-specific operations
directly, and hand off byte movement to `io::reader` / `io::writer`.

## Decision

### D1 — `fs::file` is the filesystem resource type

```kinglet
pub struct file {
  usize read(mut byte[] buffer);
  usize write(byte[] data);

  u64 size();
  void sync();

  io::reader reader();
  io::writer writer();

  void close();
  bool open();
}
```

`fs::file` owns a native file handle (POSIX fd or equivalent). Raw
`read`/`write` remain directly on `file` (matching the existing intrinsic
style of direct, unwrapped calls) so simple code does not need to construct
a reader/writer for one-shot access. Code that wants the convenience methods
from [0026](0026-standard-io-capability-model.md) (`exact`, `line`, `all`,
`flush`) calls `.reader()` / `.writer()`.

`size()` and `sync()` are filesystem-specific and stay on `file`, not on
`io::reader` / `io::writer`, per the resource/capability split in
[0026](0026-standard-io-capability-model.md#rationale).

### D2 — Opening files

```kinglet
fs::file input = fs::open(path);
fs::file output = fs::create(path);
```

- `fs::open(path)` opens an existing file for reading. The file must exist.
- `fs::create(path)` opens a file for writing, creating it if needed and
  truncating existing contents.

Both return `fs::file`. A single resource type covers both read-mode and
write-mode files; callers relying on the wrong direction get a runtime
error from the underlying native open call, mirroring how the current
`fs::__read`/`fs::__write` already fail on a bad path today. A richer
`fs::options` (append, exclusive-create, explicit read+write) is deferred;
see Non-goals.

### D3 — Public whole-file convenience functions replace the `__` intrinsics

```kinglet
byte[] fs::read(path);
string fs::readtext(path);

void fs::write(path, data);
void fs::writetext(path, text);
```

- `fs::read(path)` / `fs::write(path, data)` are byte-oriented and become
  the public replacement for `fs::__read` / `fs::__write`'s current
  whole-file behavior, but operate on `byte[]` rather than `string`.
- `fs::readtext(path)` / `fs::writetext(path, text)` are the direct
  drop-in replacements for the current `string`-based
  `fs::__read` / `fs::__write` call sites, since today's intrinsics are
  already text-shaped (`string` in, `string` out).
- `fs::__listdir` is replaced by public `fs::list(path) -> string[]`,
  preserving current name-listing behavior under a public name.

### D4 — New predicates and metadata

```kinglet
bool fs::exists(path);
```

`fs::exists` has no current equivalent and is added because directory/file
existence checks are a basic filesystem operation any real program needs
before calling `fs::open`. Full `fs::stat` / `fs::info` metadata (size,
kind, timestamps, permissions) is deferred; see Non-goals.

### D5 — `byte[]` becomes a real return/parameter type here

Kinglet's array runtime layout stores elements as one 64-bit slot each, so
`byte[]` is not memory-compact the way a native byte buffer is. This ADR
still uses `byte[]` for `fs::read` / `fs::write` because:

- it is the correct semantic type for "binary-safe file content that may not
  be valid UTF-8", distinct from `string`, which existing `fs::readtext` /
  `fs::writetext` keep human/text-oriented;
- `io::reader.read(mut byte[] buffer)` / `io::writer.write(byte[] data)`
  already commit to `byte[]` for the same reason in
  [0026](0026-standard-io-capability-model.md).

Whether `byte[]` gets a more compact backing representation is a runtime
layout question orthogonal to this ADR's public API surface, and is not
blocked or implied by this decision.

### D6 — Deprecation path for the existing intrinsics

`fs::__read`, `fs::__write`, `fs::__listdir` remain for bootstrap and
existing test compatibility. New code, new tests, and new documentation use
the public names in D2–D4. The `__` intrinsics are not removed by this ADR;
removal is a follow-up once no in-tree code depends on them.

## Examples

Whole-file read (replaces `fs::__read` call sites):

```kinglet
using fs;

int main() {
  string source = fs::readtext("src/main.kl");
  return 0;
}
```

Streaming read through the `io::reader` capability from
[0026](0026-standard-io-capability-model.md):

```kinglet
using fs;

int main() {
  fs::file input = fs::open("in.bin");
  io::reader reader = input.reader();

  byte[] buffer(4096);
  usize n = reader.read(buffer);
  input.close();
  return 0;
}
```

Existence check before opening:

```kinglet
using fs;

int main() {
  if (!fs::exists("config.txt")) {
    return 1;
  }
  string config = fs::readtext("config.txt");
  return 0;
}
```

## Non-goals

- This ADR does not introduce `fs::options` (append, exclusive-create,
  explicit read+write mode). `fs::open` / `fs::create` cover the two modes
  the current intrinsics already support (read-only, write-truncate).
- This ADR does not introduce `fs::stat` / `fs::info` / `fs::entry` metadata
  types, `fs::kind`, or permissions. `fs::exists` is the only new predicate.
- This ADR does not introduce `fs::path` / `fs::pathview` as distinct types.
  Paths remain `string`, matching current intrinsic behavior.
- This ADR does not introduce `fs::walk`, `fs::mkdir`/`fs::mkdirs`,
  `fs::copy`, `fs::rename`, `fs::remove`/`fs::removeall`, or
  `fs::current`/`fs::chdir`/`fs::temp`. `fs::list` is the only directory
  operation carried over, from the existing `fs::__listdir`.
- This ADR does not introduce `fs::stream`, `fs::StreamReader`, or any
  bespoke fs-specific I/O type. Streaming access goes through
  `fs::file.reader()` / `.writer()` returning the generic
  [0026](0026-standard-io-capability-model.md) capability types. This
  explicitly supersedes the `fs::StreamReader` / `fs::StreamWriter` design
  considered in an earlier draft of this ADR slot; that design is not
  carried forward.
- This ADR does not change the underlying native runtime calls
  (`kl_native_fs_read` / `_write` / `_listdir`); it is a compiler-frontend
  and public-API change. Runtime-level changes needed to support `fs::file`
  as a live handle (open/close lifetime, `size`, `sync`) are implementation
  work for this ADR, not a separate decision, but are not specified in
  detail here.

## Consequences

- `fs::file` becomes a real static type, resolvable in type position via
  [0025](0025-namespace-qualified-type-names.md).
- The type checker's `fs`-namespace special-casing grows from three
  intrinsic names to the D2–D4 surface, but stays in the same
  checker/backend dispatch style already used for `io`
  (`io_intrinsics`-style tables), rather than introducing a new mechanism.
- The runtime gains a live file-handle representation (for `fs::file`'s
  `read`/`write`/`size`/`sync`/`close`/`open` state) in addition to the
  existing one-shot `kl_native_fs_read`/`_write`/`_listdir` calls, which
  remain for the deprecated `__` intrinsics.
- `fs::read`/`fs::write` return/accept `byte[]`, changing the whole-file
  binary path from `string` to `byte[]`; `fs::readtext`/`fs::writetext`
  preserve today's `string` behavior unchanged.

## Dependencies

- [0025](0025-namespace-qualified-type-names.md) — required for `fs::file`
  type syntax.
- [0026](0026-standard-io-capability-model.md) — required for
  `fs::file.reader()` / `.writer()`.

## References

- Current implementation: `fs::__read`/`__write`/`__listdir` special-casing
  in `compiler/frontend/checker/type_checker.cc`; runtime calls
  `kl_native_fs_read`/`_write`/`_listdir` in `runtime/kinglet_rt_native.cc`
  and `runtime/kinglet_rt_value.h`; LLVM declarations in
  `compiler/backend/codegen/llvm/llvm_function_lowerer.cc`.
