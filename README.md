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

### Status column for decision-level ADRs

Some ADRs (e.g. those with many independent `### D<N>` / `### L<N>` decisions)
carry their own **Implementation status** table in the document body, tracking
each decision or deliverable separately. For these, the index's **Status**
column must **not** restate implementation detail in free text — it shows a
plain `accepted · X/Y delivered` ratio, where X/Y counts the ✅ rows against
all substantive rows (rows with an actual done/not-done state; documentation-only
rows such as "non-goals" or "relationship to NNNN" are excluded from the
denominator). Optionally name the remaining items in parentheses if that fits
in one line.

This ratio is a **derived summary** of the body's own table — whenever a
decision's status changes in the body table, update the index ratio in the
same commit. Do not let the two drift; a wrong ratio here is worse than an
outdated one because readers trust the index without opening the file.

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

**Progress overview**: 32 ADRs total — 16 implemented · 5 accepted (in progress) · 2 draft · 1 deferred · 8 deprecated/superseded.

### Active

#### Implemented

| # | Title | Proposed | Completed | Notes |
|---|-------|----------|-----------|-------|
| 0002 | [Design principles](0002-design-principles.md) | 2026-05-21 | 2026-05-25 | |
| 0005 | [Backend architecture: KIR + dual backend](0005-backend-architecture.md) | 2026-05-31 | 2026-06-09 | |
| 0006 | [Error handling: ??, try, and Cast unification](0006-error-handling-unification.md) | 2026-06-01 | 2026-06-02 | cast matrix corrected/widened by [bootstrap#115](https://github.com/kinglet-lang/bootstrap/pull/115) |
| 0009 | [Concepts landing](0009-concepts-landing.md) | 2026-06-03 | 2026-06-03 | |
| 0012 | [Test suite redesign](0012-test-suite-redesign.md) | 2026-06-08 | 2026-06-08 | |
| 0014 | [Compilation toolchain architecture](0014-compilation-toolchain-architecture.md) | 2026-06-09 | 2026-06-10 | |
| 0015 | [LLVM backend roadmap](0015-llvm-backend-roadmap.md) | 2026-06-09 | 2026-06-10 | |
| 0016 | [Typed KIR for native lowering](0016-typed-kir.md) | 2026-06-10 | 2026-06-10 | phase 1; phase 2 partial |
| 0017 | [Dense layout for `T[][]…[]` syntax](0017-dense-nested-array-layout.md) | 2026-06-12 | 2026-06-12 | v1: 2D literals |
| 0018 | [Logical module system](0018-logical-module-system.md) | 2026-06-17 | 2026-06-20 | manifest shape amended by [0020](0020-project-manifest-and-targets.md) |
| 0020 | [Project manifest (`.nest`) and build targets](0020-project-manifest-and-targets.md) | 2026-06-17 | 2026-07-03 | target-block layout |
| 0024 | [LSP completion: Sema-backed field-access resolution](0024-lsp-completion-sema-integration.md) | 2026-07-06 | 2026-07-06 | |
| 0025 | [Namespace-qualified type names](0025-namespace-qualified-type-names.md) | 2026-07-10 | 2026-07-11 | |
| 0026 | [Standard I/O capability model](0026-standard-io-capability-model.md) | 2026-07-10 | 2026-07-16 | `io::reader` / `io::writer`; `fs::file` satisfaction via [0027](0027-filesystem-resource-api.md) |
| 0027 | [Filesystem resource API](0027-filesystem-resource-api.md) | 2026-07-10 | 2026-07-16 | `fs::file` resource; direct `io::reader` / `io::writer` satisfaction |
| 0030 | [Definite assignment and literal width inference](0030-definite-assignment-and-literal-width-inference.md) | 2026-07-14 | 2026-07-14 | |

#### Accepted (in progress)

| # | Title | Delivered | Remaining | Proposed |
|---|-------|-----------|-----------|----------|
| 0003 | [Standard library roadmap](0003-stdlib-roadmap.md) | phased | platform APIs still hardcoded; Phase A–D pending | 2026-05-29 |
| 0019 | [Self-host LLVM backend](0019-self-host-llvm-backend.md) | S0 + S1 (95 test cases) | S2 aggregates/errors · S3 cross-target · S4 shadow-native driver | 2026-06-17 |
| 0028 | [Ownership, borrowing, and value transfer](0028-ownership-and-value-transfer.md) | 13/14 decisions | D9 `move()` stdlib fn · D13 closures | 2026-07-11 |
| 0029 | [Value representation and memory layout](0029-value-representation-and-memory-layout.md) | 6/9 deliverables | L1 struct inline · L2 fixed-array inline · `none` literal | 2026-07-11 |
| 0031 | [Diagnostic system: error codes, severity, and rendering](0031-diagnostic-system.md) | 0/7 deliverables | D1 Diagnostic struct · D3 first 20 codes · D5 fix-its · D6 warning groups · D7 rendering · cascade suppression · migration | 2026-07-15 |

#### Draft

| # | Title | Proposed | Blocked by |
|---|-------|----------|------------|
| 0001 | [Pending syntax and performance items](0001-pending-syntax-and-perf.md) | 2026-05-17 | - (parking lot; items promoted to standalone ADRs when ready) |
| 0032 | [Text encoding API](0032-text-encoding-api.md) | 2026-07-16 | decode error policy / richer error model |

#### Deferred

| # | Title | Proposed |
|---|-------|----------|
| 0004 | [LSP roadmap](0004-lsp-roadmap.md) | 2026-05-30 |

### Deprecated / Superseded

| # | Title | Proposed | Superseded by |
|---|-------|----------|---------------|
| 0007 | [\[deprecated\] Trait system redesign](%5Bdeprecated%5D%200007-trait-system-redesign.md) | 2026-06-02 | [0009](0009-concepts-landing.md) |
| 0008 | [\[deprecated\] KBC bytecode format evolution](%5Bdeprecated%5D%200008-kbc-format-evolution.md) | 2026-06-02 | deprecated (implemented; bytecode path retired by [0022](%5Bdeprecated%5D%200022-native-unique-ownership.md)) |
| 0010 | [\[deprecated\] VM redesign and embedded self-host binary](%5Bdeprecated%5D%200010-vm-redesign.md) | 2026-06-03 | deprecated (Part 1 implemented; VM path retired by [0022](%5Bdeprecated%5D%200022-native-unique-ownership.md)) |
| 0011 | [Module system redesign](0011-module-system-redesign.md) | 2026-06-03 | deprecated (core retained; syntax superseded by [0018](0018-logical-module-system.md)) |
| 0013 | [\[deprecated\] Bootstrap bytecode parity](%5Bdeprecated%5D%200013-bootstrap-bytecode-delta.md) | 2026-06-08 | deprecated (implemented; parity retired by [0022](%5Bdeprecated%5D%200022-native-unique-ownership.md)) |
| 0021 | [\[deprecated\] References and move semantics](%5Bdeprecated%5D%200021-references-and-move.md) | 2026-06-21 | [0022](%5Bdeprecated%5D%200022-native-unique-ownership.md) → [0028](0028-ownership-and-value-transfer.md) |
| 0022 | [\[deprecated\] Native-only toolchain and unique ownership](%5Bdeprecated%5D%200022-native-unique-ownership.md) | 2026-06-21 | [0028](0028-ownership-and-value-transfer.md) (D1 native-only toolchain carried forward) |
| 0023 | [\[deprecated\] Data types, literals, and ABI](%5Bdeprecated%5D%200023-data-types-and-abi.md) | 2026-06-21 | [0029](0029-value-representation-and-memory-layout.md) (literal/width-table content carried forward) |
