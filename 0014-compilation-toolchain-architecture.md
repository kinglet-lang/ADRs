# 0014 — Compilation Toolchain Architecture

- **Status**: implemented
- **Proposed**: 2026-06-09
- **Completed**: 2026-06-10

## Context

The project currently treats self-hosted bytecode (`compiler.kbc`) as the primary
compiler artefact for development and tests. In practice:

| Path | Approximate time (local) | Role today |
|------|--------------------------|------------|
| Bootstrap C++ `--save-bytecode` | ~0.07s | Builds `compiler.kbc` in `ensure_cli_kbc` |
| Self-host VM `--save-bytecode` | ~3–4s | Round-trip and parity validation |
| Self-host VM `--save-bytecode --strip-debug` | ~3s | Faster, but not parity-gated |

Bootstrap parity for `core/main.kl` is achieved ([0013](%5Bdeprecated%5D%200013-bootstrap-bytecode-delta.md)).
The remaining cost is dominated by **interpreting the compiler on the VM**, not by
the on-disk `.kbc` format ([0008](%5Bdeprecated%5D%200008-kbc-format-evolution.md) P0–P3 are complete).

The `.kbc` format remains useful as a **VM execution image** for user programs, but
it is a poor fit as the centre of the **toolchain** pipeline: serialization in the
self-host path is string-based and slow, and requiring every developer workflow to
load `compiler.kbc` adds latency without improving correctness.

[0005](0005-backend-architecture.md) already proposes KIR as a shared IR between
frontends and backends. This decision defines how the **toolchain** is structured
around that IR, how build outputs are cached, and how the Kinglet-implemented
compiler relates to the C++ reference implementation.

## Definitions

| Term | Meaning |
|------|---------|
| **Ref compiler** | C++ bootstrap implementation (`kinglet-lang/bootstrap` / `KINGLET_BOOTSTRAP`). Authoritative for day-to-day builds. |
| **Shadow compiler** | Kinglet sources under `core/`, `parser/`, `compiler/`, `checker/`, `lexer/`, executed on the VM via `compiler.kbc`. Used for correctness validation, not default builds. |
| **KIR** | Kinglet IR — typed CFG between the type checker and backends ([0005](0005-backend-architecture.md)). |
| **Klos** | Kinglet object store — content-addressed cache under `.kinglet/objects/`. |
| **Stamp** | Fingerprint of inputs (source hashes, compiler version, flags, target) that names a cached build result. |
| **kbc** | Version-2 VM bytecode file ([0008](%5Bdeprecated%5D%200008-kbc-format-evolution.md)). A **backend output**, not the canonical compile result. |

## Decision

### D1 — Dual-track compilers: Ref for production, Shadow for proof

1. **Default compilation** uses the Ref compiler only.
2. **Shadow compilation** runs only when explicitly requested (`kinglet prove`) or
   in CI jobs configured for toolchain validation.
3. Shadow parity gates remain as defined in [0013](%5Bdeprecated%5D%200013-bootstrap-bytecode-delta.md)
   until KIR-level comparison supersedes bytecode comparison (see D4).

Rationale: the Kinglet compiler sources express **language semantics**; the C++
implementation provides **throughput** for the toolchain. This matches how the
project already builds `compiler.kbc` via bootstrap in `ensure_cli_kbc`.

### D2 — KIR is the primary compiler seam

Adopt the pipeline from [0005](0005-backend-architecture.md) as the target shape:

```
source → lexer → parser → checker → KIR emit → backend
```

Changes to the 0005 contract:

1. **KIR lives in memory** by default. Textual dump (`kinglet ir`, `--ir`) is for
   debugging and golden tests.
2. **Backends** produce typed blobs in Klos, not necessarily files with legacy
   extensions in the project root.
3. The **VM backend** lowers KIR → kbc. kbc is required only when the chosen
   execution mode is the VM.
4. A **native backend** (initially Ref-only) is the long-term default for the
   toolchain itself (`core/main.kl` and future `kinglet` driver).

Bytecode golden tests against C++ output remain until KIR emit is stable; then
prove jobs may compare KIR text before comparing kbc.

### D3 — kbc is a transitional VM target, not the toolchain currency

1. **Freeze** kbc format at version 2 ([0008](%5Bdeprecated%5D%200008-kbc-format-evolution.md)). No
   further format work for compile-speed goals.
2. **User-tier programs** may still emit kbc when `backend = "vm"` (or
   `kinglet run --vm`).
3. **Toolchain-tier** (`core/main.kl`, compiler modules) targets **native** once
   the native backend can build them. Until then, Ref `--save-bytecode` remains the
   bootstrap path for Shadow artefacts.
4. Self-host serialization helpers in `compiler/bytecode.kl` are maintained for
   parity tests; they are not on the Ref hot path.

`--save-bytecode` / `--strip-debug` become **debug and test commands**, not the
primary `kinglet build` output.

### D4 — Klos: content-addressed artefact store

Project-local layout:

```
.kinglet/
├── cache/           # stamp → object id mapping (optional index)
├── objects/         # <sha256> blobs, opaque on disk
├── out/             # symlinks or copies of default outputs
└── stamps/          # last-success fingerprints per target
```

Each object records metadata (kind, size, producer, stamp) in a sidecar
`.kinglet/objects/<id>.meta` or an index file. Kinds include at minimum:

| Kind | Description |
|------|-------------|
| `kbc` | VM bytecode |
| `native` | Executable or linkable object |

Kinds `kir` and `proof` (serialized KIR, prove-run markers) are **deferred** —
M0–M4 use `kbc` and `native` only. KIR remains in-memory; prove status is tracked
by CI jobs, not Klos blobs.

**Stamp** computation:

```
stamp = H( sorted(source_file_hashes), compiler_version, backend, flags, target_triple )
```

Cache hit: stamp unchanged → reuse object id, skip compilation.

Rationale: avoids ad-hoc `compiler.kbc` staleness checks scattered in shell helpers;
unifies user and toolchain outputs under one model.

### D5 — `kinglet build` and project manifest

Extend `kinglet.toml`:

```toml
[project]
name = "kinglet"
version = "0.1.0"

[build]
root = "core/main.kl"          # default compile root (toolchain projects)
cache_dir = ".kinglet/cache"
out_dir = ".kinglet/out"

[build.compiler]
engine = "ref"                 # ref | shadow (shadow discouraged locally)
default_backend = "native"     # native | vm

[build.prove]
shadow_root = "core/main.kl"   # input for kinglet prove
```

Commands (target interface):

| Command | Behaviour |
|---------|-----------|
| `kinglet build` | Compute stamp; Ref compile on miss; write Klos + `.kinglet/out/` |
| `kinglet run` | Run default output (native if available, else vm) |
| `kinglet ir <file>` | Emit KIR text to stdout |
| `kinglet prove` | Shadow compile + compare to Ref (KIR and/or kbc per CI tier) |
| `kinglet debug emit-kbc <out> <src>` | Explicit kbc emission for tests |

`klet` is an optional **CLI alias** for the same driver binary (`klet build` ≡
`kinglet build`). Canonical product and documentation name remains **Kinglet**;
manifest and cache paths stay `kinglet.toml` and `.kinglet/`. Do not introduce
`klet.toml`, `.klet/`, or a separate `klet` package manager. Install may ship
`klet` as a symlink or duplicate argv0 to `kinglet` (same subcommands). Avoid
`kl` as a command alias — it collides mentally with the `.kl` source extension.

`tests/common.sh` migrates from `ensure_cli_kbc` to `ensure_build_stamp` for
suites that only need a fresh compiler artefact; Shadow-only suites keep
`compiler.kbc` generation behind prove or a dedicated helper.

### D6 — Amend [0010](%5Bdeprecated%5D%200010-vm-redesign.md) Part 2 (embedded compiler)

[0010](%5Bdeprecated%5D%200010-vm-redesign.md) Part 2 proposed embedding `cli.kbc` in the `kinglet`
binary. That approach is **deferred** in favour of:

1. Embedding a **native toolchain artefact** (Ref-built compiler as shared library
   or static archive), or
2. Loading a **Klos object** by stamp at startup (no project-root `compiler.kbc`).

VM embedding of kbc remains an option for **user program** distribution, not for
the compiler driver. COW / RC work in 0010 Part 1 stays valid for VM execution.

### D7 — Phased implementation

| Phase | Scope | Exit criterion |
|-------|--------|----------------|
| **M0** | Klos directories, stamp, `kinglet build` wrapper calling Ref | `tests/run_all.sh` uses stamp; no mandatory self-host compile in common path |
| **M1** | KIR module + `kinglet ir`; Ref split per 0005 | KIR golden tests for small programs |
| **M2** | `kinglet prove`; CI tier for Shadow | Parity job explicit; local default is Ref-only |
| **M3** | Native backend for toolchain root ([0015](0015-llvm-backend-roadmap.md) L4–L5) | `core/main.kl` builds to native via LLVM without kbc on the hot path |
| **M4** | Per-module stamps + link | Incremental rebuild of compiler modules |

**Status (2026-06-10)**: M0–M4 complete. `kinglet build` defaults to the native
backend and emits `.kinglet/out/compiler`; the native driver passes
`tests/native/run_driver_smoke.sh` without `compiler.kbc` on the hot path.
Test suites keep building `compiler.kbc` via `--backend vm` for VM parity.
Native builds are incremental: a whole-build stamp (`compiler-native`) skips
unchanged builds entirely, and a per-module object cache under
`.kinglet/objects/native` (bootstrap `--obj-cache`) re-emits only edited
modules before the relink (`tests/native/run_incremental_smoke.sh`).

Phases may overlap; M0 does not require KIR to land first. LLVM phases L0–L5 are
defined in [0015](0015-llvm-backend-roadmap.md); L0 starts once a minimal KIR C++
representation exists (late M1). Solo-developer **commit-level** breakdown is in
0015 § Commit sequence.

## Open questions (resolutions)

1. **KIR serialization** — **Resolved for M0–M4**: in-memory KIR + text dump
   (`--ir`) and golden hashes are sufficient. Binary on-disk KIR for distributed
   cache is deferred.
2. ~~**Native backend technology**~~ — resolved by [0015](0015-llvm-backend-roadmap.md) D9 (LLVM + `libkinglet_rt`).
3. **Shadow prove contract** — **Resolved for M0–M4**: bytecode identity for
   `core/main.kl` remains gated ([0013](%5Bdeprecated%5D%200013-bootstrap-bytecode-delta.md)).
   KIR-level prove may supplement later; it does not replace kbc parity yet.
4. **Cross-platform stamps** — **Known issue**: `pass2b_ns_rank` may differ by
   libc++ map order. Shadow kbc comparison is macOS-gated; stamp may gain a
   `target` component if multi-platform prove is required.
5. **`kinglet init`** — **Deferred** to V0 (project template syntax in
   [0003](0003-stdlib-roadmap.md)).

## Post-0014 (V0 — out of scope for this ADR)

The following D5 commands are **not** required to close 0014; they belong to the
shipped user CLI (see project `Kinglet-TODO.md`):

| Command | Notes |
|---------|--------|
| `kinglet run` | Run default `.kinglet/out/` artefact |
| `kinglet init` | New project template |
| Native `kinglet` binary | `init` / `build` / `run` only; dev tools stay Bash + bootstrap |

`kinglet ir <file>` remains a **bootstrap flag** (`--ir`) on the dev path; wrapping
it in the Bash driver is optional.

### `kinglet clean` (M4-4, deferred to V0)

Remove Klos blobs under `.kinglet/objects/` that no **stamp** still references.
This is **build-cache housekeeping only** — not language/runtime GC (see
[0002](0002-design-principles.md) deterministic destruction). Name **`kinglet clean`**
to avoid confusion with heap collection. Implementation deferred with the V0 CLI;
until then: `rm -rf .kinglet/objects` or selective prune by hand.

`ensure_cli_kbc` in `tests/common.sh` is **intentionally retained** for Shadow/
prove suites; fast paths use `ensure_build_stamp` as planned in M0.

## Consequences

- **Faster local iteration**: default path avoids VM interpretation of the full
  compiler (~3s → Ref ~0.07s for bytecode emission; native target removes kbc
  from the loop entirely when M3 lands).
- **Clearer roles**: kbc size/format optimisations ([0008](%5Bdeprecated%5D%200008-kbc-format-evolution.md))
  no longer block toolchain performance work.
- **Test layout**: `selfhost/`, `differential/`, and `codegen/` remain; prove
  orchestration moves behind `kinglet prove` instead of implicit `ensure_cli_kbc`
  self-host rebuilds.
- **Documentation debt**: README and build docs updated for native-default
  `kinglet build` (2026-06-10).
- **0010 Part 2** embedding scope changes; VM embed is for user binaries, not
  the compiler driver.
- **0005** status should move from draft → accepted once KIR form is locked in
  implementation; this ADR does not change KIR structure, only toolchain placement.

## Dependencies

- [0005](0005-backend-architecture.md) — KIR definitions and backend split.
- [0015](0015-llvm-backend-roadmap.md) — LLVM native backend phases L0–L5.
- [0008](%5Bdeprecated%5D%200008-kbc-format-evolution.md) — kbc v2 as VM backend output (frozen).
- [0010](%5Bdeprecated%5D%200010-vm-redesign.md) — VM runtime (Part 1); Part 2 amended by D6.
- [0013](%5Bdeprecated%5D%200013-bootstrap-bytecode-delta.md) — Shadow bytecode parity until prove
  contract evolves.

## References

- Measured compile times (2026-06-09, local): bootstrap save ~0.07s; self-host
  save strip ~3s; self-host save debug ~4s.
- `tests/common.sh` — `ensure_cli_kbc`, `compile_kl`, `compile_selfhost`.
- `core/main.kl` — current CLI entry for Shadow compiler.

## Amendments

### 2026-06-17 — Project manifest and build targets ([0020](0020-project-manifest-and-targets.md))

D5 `kinglet.toml` project schema is **amended** for new projects:

| D5 (this ADR, 2026-06-10) | Amended policy ([0020](0020-project-manifest-and-targets.md)) |
|---------------------------|---------------------------------------------------------------|
| `[build].root = "core/main.kl"` | `binary <artifact> -> module "<logical>"` in `kinglet.nest` |
| `kinglet.toml` manifest | `kinglet.nest` (PML); TOML dual-read during transition only |
| Single implicit executable | Explicit `binary` / `library` target registration |

**Unchanged:** Klos layout (D4), stamp model, `kinglet build` / prove commands, Ref vs
Shadow dual-track (D1), kbc as VM backend output (D3).

Shadow / self-host build orchestration reads `.nest` only and does not implement TOML
([0020](0020-project-manifest-and-targets.md) D5, [0019](0019-self-host-llvm-backend.md) D5).

Original D5 text and examples above are preserved for historical context.
