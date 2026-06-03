# Design Decisions

Records of significant language and architecture decisions. Each entry captures **what** was decided, **why**, and **what it affects** — so we don't re-derive context from code later.

## Format

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
| 0003 | [Standard library roadmap](0003-stdlib-roadmap.md) | deferred | 2026-05-29 | |
| 0004 | [LSP roadmap](0004-lsp-roadmap.md) | deferred | 2026-05-30 | |
| 0005 | [Backend architecture: KIR + dual backend](0005-backend-architecture.md) | draft | 2026-05-31 | |
| 0006 | [Error handling: ??, try, and Cast unification](0006-error-handling-unification.md) | implemented | 2026-06-01 | 2026-06-02 |
| 0007 | [Trait system redesign](0007-trait-system-redesign.md) | superseded-by [0009](0009-concepts-landing.md) | 2026-06-02 | |
| 0008 | [KBC bytecode format evolution](0008-kbc-format-evolution.md) | implemented | 2026-06-02 | 2026-06-03 |
| 0009 | [Concepts landing](0009-concepts-landing.md) | implemented | 2026-06-03 | 2026-06-03 |
| 0010 | [VM redesign and embedded self-host binary](0010-vm-redesign.md) | draft | 2026-06-03 | |
| 0011 | [Module system redesign](0011-module-system-redesign.md) | accepted | 2026-06-03 | |
