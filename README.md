# Design Decisions

Records of significant language and architecture decisions. Each entry captures **what** was decided, **why**, and **what it affects** — so we don't re-derive context from code later.

## Format

Decision files (RFCs) are written in **English**.

Every decision file follows this structure:

```
# NNNN — Title

- **Status**: draft / accepted / implemented / deferred / superseded-by NNNN
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
- **superseded** — replaced by a newer decision (link provided)

## Index

| # | Title | Status | Proposed | Completed |
|---|-------|--------|----------|-----------|
| 0001 | [Pending syntax and performance items](0001-pending-syntax-and-perf.md) | draft | 2026-05-17 | |
| 0002 | [Design principles](0002-design-principles.md) | implemented | 2026-05-21 | 2026-05-25 |
| 0003 | [Standard library roadmap](0003-stdlib-roadmap.md) | draft | 2026-05-29 | |
| 0004 | [LSP roadmap](0004-lsp-roadmap.md) | deferred | 2026-05-30 | |
| 0005 | [Backend architecture: KIR + dual backend](0005-backend-architecture.md) | implemented | 2026-05-31 | 2026-06-09 |
| 0006 | [Error handling: ??, try, and Cast unification](0006-error-handling-unification.md) | implemented | 2026-06-01 | 2026-06-02 |
| 0007 | [Trait system redesign](0007-trait-system-redesign.md) | superseded-by [0009](0009-concepts-landing.md) | 2026-06-02 | |
| 0008 | [KBC bytecode format evolution](0008-kbc-format-evolution.md) | implemented | 2026-06-02 | 2026-06-03 |
| 0009 | [Concepts landing](0009-concepts-landing.md) | implemented | 2026-06-03 | 2026-06-03 |
| 0010 | [VM redesign and embedded self-host binary](0010-vm-redesign.md) | implemented | 2026-06-03 | 2026-06-08 |
| 0011 | [Module system redesign](0011-module-system-redesign.md) | implemented | 2026-06-03 | 2026-06-04 |
| 0012 | [Test suite redesign](0012-test-suite-redesign.md) | implemented | 2026-06-08 | 2026-06-08 |
| 0013 | [Bootstrap bytecode parity](0013-bootstrap-bytecode-delta.md) | implemented | 2026-06-08 | 2026-06-09 |
| 0014 | [Compilation toolchain architecture](0014-compilation-toolchain-architecture.md) | implemented | 2026-06-09 | 2026-06-10 |
| 0015 | [LLVM backend roadmap](0015-llvm-backend-roadmap.md) | implemented | 2026-06-09 | 2026-06-10 |
| 0016 | [Typed KIR for native lowering](0016-typed-kir.md) | implemented (phase 1; phase 2 partial) | 2026-06-10 | 2026-06-10 |
| 0017 | [Dense layout for `T[][]…[]` syntax](0017-dense-nested-array-layout.md) | implemented (v1: 2D literals) | 2026-06-12 | 2026-06-12 |
| 0018 | [Logical module system](0018-logical-module-system.md) | draft | 2026-06-17 | |
| 0019 | [Self-host LLVM backend](0019-self-host-llvm-backend.md) | draft | 2026-06-17 | |
| 0020 | [Project manifest (`.nest`) and build targets](0020-project-manifest-and-targets.md) | draft | 2026-06-17 | |
