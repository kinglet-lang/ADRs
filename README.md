# Design Decisions

Records of significant language and architecture decisions. Each entry captures **what** was decided, **why**, and **what it affects** — so we don't re-derive context from code later.

## Format

Decision files (RFCs) are written in **English**.

Every decision file follows this structure:

```
# NNNN — Title

- **Status**: draft / accepted / implemented / deferred / deprecated / superseded-by NNNN
- **Proposed**: YYYY-MM-DD
- **Completed**: YYYY-MM-DD (when implemented)

## Context
Why this decision is needed.

## Decision
What we decided to do.

## Consequences
What changes and what to watch out for.
```

### Amendments

When a later decision changes or extends an **implemented** ADR, append an
`## Amendments` section at the **end** of the original file. Do **not** edit the
original decision text in place.

Each entry:

```markdown
## Amendments

### YYYY-MM-DD — Short title ([NNNN](NNNN-slug.md))

<What is superseded, amended, or deferred; pointer to the new ADR.>

Original sections above are preserved for historical context.
```

Update the index **Status** column when an ADR is superseded (e.g.
`superseded-by [0018](0018-…)` for partial supersession, note in Amendments).

### Full absorption vs superseded-by link

When a **new** ADR absorbs the substantive content of an old one (its decisions
become part of the new document's own text, not just cross-referenced), the new
ADR must **copy the relevant content in full** into its own body — a link back
to the old file is not sufficient. Readers of the new ADR should not need to
open the old one to understand the current design.

The old ADR is then renamed with a **`[deprecated] ` filename prefix**
(e.g. `0021-references-and-move.md` → `[deprecated] 0021-references-and-move.md`)
so the deprecated status is visible from the filename itself, not only from the
index table or an `## Amendments` footnote. Update every inbound link across the
`ADRs/` tree (including the index table's title text, prefixed with
`[deprecated] ` too) to point at the new filename. Because the filename contains
`[`, `]`, and a space, Markdown link targets must URL-encode it:
`%5Bdeprecated%5D%20NNNN-slug.md`.

This applies retroactively — any ADR already in `deprecated` / `superseded-by`
status gets the filename prefix too, not just newly-superseded ones going
forward.

## Lifecycle

```
draft → accepted → implemented
  ↑        ↓
  └── deferred / superseded
```

- **draft** — still under discussion, direction not yet decided
- **accepted** — design finalized, awaiting implementation
- **implemented** — code landed and behavior is live
- **deferred** — decision postponed, not actively pursued
- **deprecated** — implemented or historical, but no longer the active design path
- **superseded** — replaced by a newer decision (link provided)

## Index

| # | Title | Status | Proposed | Completed |
|---|-------|--------|----------|-----------|
| 0001 | [Pending syntax and performance items](0001-pending-syntax-and-perf.md) | draft | 2026-05-17 | |
| 0002 | [Design principles](0002-design-principles.md) | implemented | 2026-05-21 | 2026-05-25 |
| 0003 | [Standard library roadmap](0003-stdlib-roadmap.md) | accepted (phased; io/fs refined by [0026](0026-standard-io-capability-model.md)/[0027](0027-filesystem-resource-api.md)) | 2026-05-29 | |
| 0004 | [LSP roadmap](0004-lsp-roadmap.md) | deferred | 2026-05-30 | |
| 0005 | [Backend architecture: KIR + dual backend](0005-backend-architecture.md) | implemented | 2026-05-31 | 2026-06-09 |
| 0006 | [Error handling: ??, try, and Cast unification](0006-error-handling-unification.md) | implemented | 2026-06-01 | 2026-06-02 |
| 0007 | [\[deprecated\] Trait system redesign](%5Bdeprecated%5D%200007-trait-system-redesign.md) | superseded-by [0009](0009-concepts-landing.md) | 2026-06-02 | |
| 0008 | [\[deprecated\] KBC bytecode format evolution](%5Bdeprecated%5D%200008-kbc-format-evolution.md) | deprecated (implemented; bytecode path retired by [0022](0022-native-unique-ownership.md)) | 2026-06-02 | 2026-06-03 |
| 0009 | [Concepts landing](0009-concepts-landing.md) | implemented | 2026-06-03 | 2026-06-03 |
| 0010 | [\[deprecated\] VM redesign and embedded self-host binary](%5Bdeprecated%5D%200010-vm-redesign.md) | deprecated (Part 1 implemented; VM path retired by [0022](0022-native-unique-ownership.md)) | 2026-06-03 | 2026-06-08 |
| 0011 | [Module system redesign](0011-module-system-redesign.md) | deprecated (core retained; syntax superseded by [0018](0018-logical-module-system.md)) | 2026-06-03 | 2026-06-04 |
| 0012 | [Test suite redesign](0012-test-suite-redesign.md) | implemented | 2026-06-08 | 2026-06-08 |
| 0013 | [\[deprecated\] Bootstrap bytecode parity](%5Bdeprecated%5D%200013-bootstrap-bytecode-delta.md) | deprecated (implemented; parity retired by [0022](0022-native-unique-ownership.md)) | 2026-06-08 | 2026-06-09 |
| 0014 | [Compilation toolchain architecture](0014-compilation-toolchain-architecture.md) | implemented | 2026-06-09 | 2026-06-10 |
| 0015 | [LLVM backend roadmap](0015-llvm-backend-roadmap.md) | implemented | 2026-06-09 | 2026-06-10 |
| 0016 | [Typed KIR for native lowering](0016-typed-kir.md) | implemented (phase 1; phase 2 partial) | 2026-06-10 | 2026-06-10 |
| 0017 | [Dense layout for `T[][]…[]` syntax](0017-dense-nested-array-layout.md) | implemented (v1: 2D literals) | 2026-06-12 | 2026-06-12 |
| 0018 | [Logical module system](0018-logical-module-system.md) | implemented (source-level modules; manifest shape amended by [0020](0020-project-manifest-and-targets.md)) | 2026-06-17 | 2026-06-20 |
| 0019 | [Self-host LLVM backend](0019-self-host-llvm-backend.md) | accepted (S0 + S1 lowering delivered) | 2026-06-17 | |
| 0020 | [Project manifest (`.nest`) and build targets](0020-project-manifest-and-targets.md) | implemented (target-block layout; earlier layouts deprecated) | 2026-06-17 | 2026-07-03 |
| 0021 | [\[deprecated\] References and move semantics](%5Bdeprecated%5D%200021-references-and-move.md) | superseded-by [0022](0022-native-unique-ownership.md) | 2026-06-21 | |
| 0022 | [Native-only toolchain and unique ownership](0022-native-unique-ownership.md) | accepted (N0 native-only implemented; ownership pending) | 2026-06-21 | |
| 0023 | [Data types, literals, and ABI](0023-data-types-and-abi.md) | accepted (P0 slices partial) | 2026-06-21 | |
| 0024 | [LSP completion: Sema-backed field-access resolution](0024-lsp-completion-sema-integration.md) | implemented | 2026-07-06 | 2026-07-06 |
| 0025 | [Namespace-qualified type names](0025-namespace-qualified-type-names.md) | implemented | 2026-07-10 | 2026-07-11 |
| 0026 | [Standard I/O capability model](0026-standard-io-capability-model.md) | draft | 2026-07-10 | |
| 0027 | [Filesystem resource API](0027-filesystem-resource-api.md) | draft | 2026-07-10 | |
