# 0032 — Text Encoding API

- **Status**: draft
- **Proposed**: 2026-07-16

Depends on: [0025](0025-namespace-qualified-type-names.md) (namespace-qualified names), [0027](0027-filesystem-resource-api.md) (byte-oriented filesystem I/O)

## Context

[0027](0027-filesystem-resource-api.md) intentionally makes public whole-file
filesystem I/O byte-oriented:

```kinglet
byte[] fs::read(path);
void fs::write(path, byte[] data);
```

Bootstrap PR #138 removed the string-shaped `fs::readtext` / `fs::writetext`
helpers. That keeps filesystem I/O aligned with `fs::file` and the
`io::reader` / `io::writer` capability model, but it leaves a usability gap for
common text files:

```kinglet
byte[] defaults = [byte(100), byte(101), byte(102), byte(97), byte(117), byte(108), byte(116)]; // "default"
fs::write("config.txt", defaults);
```

The missing operation is not filesystem-specific. It is text encoding:
converting between Kinglet `string` and a concrete byte representation such as
UTF-8 or GBK. Putting that conversion back into `fs` would reintroduce a parallel
text-file surface and blur the boundary between files, bytes, and text.

This ADR defines the standard library shape for explicit text encoding modules.

## Decision

### D1 — Text encoding lives under `txt`, not `fs`

Text encoding APIs are provided under a new `txt` namespace family. Filesystem
APIs continue to operate on `byte[]` and `fs::file` resources.

```kinglet
using fs;

int main() {
  byte[] data = txt::utf8.encode("default");
  fs::write("config.txt", data);
  return 0;
}
```

`fs` owns paths and file handles. `txt` owns string/byte encoding boundaries.
This separation keeps `fs::read` / `fs::write` binary-safe and avoids adding
text convenience wrappers to the filesystem API.

### D2 — Each encoding is a namespace module

Statically known encodings are exposed as subnamespaces/modules:

```kinglet
byte[] txt::utf8.encode(string text);
string txt::utf8.decode(byte[] data);

byte[] txt::gbk.encode(string text);
string txt::gbk.decode(byte[] data);
```

The call site names the concrete codec in the namespace path:

```kinglet
byte[] utf8 = txt::utf8.encode("hello");
byte[] gbk = txt::gbk.encode("中文");
```

This is preferred over a single stringly-typed dispatcher such as
`txt::encode(text, "utf-8")`, because:

- misspelled encoding names become static namespace/member errors rather than
  runtime string errors;
- API documentation and completion can attach to each codec module;
- each codec can grow codec-specific options later without turning one central
  function into a large switch over all encodings.

### D3 — UTF-8 is the first required codec

`txt::utf8` is the first standard codec. It is required before text-oriented
examples should use `fs::write` with string literals.

Minimal API:

```kinglet
byte[] txt::utf8.encode(string text);
string txt::utf8.decode(byte[] data);
```

GBK is a planned codec with the same shape:

```kinglet
byte[] txt::gbk.encode(string text);
string txt::gbk.decode(byte[] data);
```

The standard library may add more codecs (`utf16le`, `utf16be`, etc.) by adding
more submodules with the same two core operations.

### D4 — Runtime-selected encodings are a future layer

This ADR does not make dynamic codec selection the primary API. If programs later
need to choose an encoding at runtime, add a separate layer:

```kinglet
enum txt::encoding {
  utf8,
  gbk,
}

byte[] txt::encode(string text, txt::encoding encoding);
string txt::decode(byte[] data, txt::encoding encoding);
```

The static module APIs remain the normal path:

```kinglet
txt::utf8.encode(text);      // static, preferred
txt::encode(text, enc);      // dynamic, optional future layer
```

A string parameter such as `"utf-8"` is deliberately not part of the core API.

### D5 — Decode error policy is explicit implementation work

`decode` can encounter invalid byte sequences. This ADR fixes the namespace/API
shape but leaves the exact error policy to implementation work or a follow-up
ADR tied to the broader error model:

- strict decode may return `string?` or a future `Result<string, txt::decode_error>`;
- lenient decode may return `string` and replace invalid sequences;
- both modes may eventually coexist as distinct functions or options.

Until that is decided, examples should avoid depending on invalid-input behavior.
The stable part of this ADR is the separation of `txt::<codec>` from `fs` and the
static codec module shape.

## Examples

Write a default config file:

```kinglet
using fs;

int main() {
  if (!fs::exists("config.txt")) {
    fs::write("config.txt", txt::utf8.encode("default"));
  }
  return 0;
}
```

Read a UTF-8 text file:

```kinglet
using fs;

int main() {
  byte[] raw = fs::read("config.txt");
  string config = txt::utf8.decode(raw);
  return 0;
}
```

Use a file handle with explicit encoding at the boundary:

```kinglet
using fs;

int main() {
  fs::file out = fs::create("message.txt");
  byte[] data = txt::utf8.encode("hello");
  out.write(data);
  out.sync();
  out.close();
  return 0;
}
```

## Non-goals

- This ADR does not reintroduce `fs::readtext` / `fs::writetext`.
- This ADR does not make `fs::read` / `fs::write` accept `string` directly.
- This ADR does not define path encodings or OS filename encoding behavior.
- This ADR does not define Unicode normalization, collation, grapheme clusters,
  or locale-aware text processing.
- This ADR does not require GBK to be implemented at the same time as UTF-8.

## Consequences

- Text examples can become readable again without weakening the byte-oriented
  filesystem API:
  `txt::utf8.encode("default")` replaces long manual byte arrays.
- `fs` remains a binary/resource API. Text conversion is an explicit boundary
  operation owned by `txt`.
- Tooling and docs can expose codec-specific completions such as
  `txt::utf8.encode` and `txt::gbk.decode`.
- A future dynamic encoding API can be layered on top without replacing the
  static module APIs.

## Implementation status

| Decision | Status | Notes |
|----------|--------|-------|
| D1 `txt` owns string/byte conversion | ❌ not implemented | New stdlib namespace family |
| D2 codec modules (`txt::utf8`, `txt::gbk`) | ❌ not implemented | Static namespace-qualified calls |
| D3 UTF-8 codec | ❌ not implemented | First required codec |
| D4 dynamic codec dispatcher | 🔜 deferred | Optional future layer |
| D5 decode error policy | 🔜 deferred | Depends on richer error model |

## Relationship to 0027

[0027](0027-filesystem-resource-api.md) deliberately removed public text-file
helpers and kept filesystem I/O byte-oriented. This ADR supplies the missing text
conversion layer without moving text semantics back into `fs`.

```kinglet
byte[] raw = txt::utf8.encode(text);
fs::write(path, raw);
```

That is the intended replacement for the removed `fs::writetext` convenience
shape.
