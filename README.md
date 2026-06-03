# Design Decisions

Records of significant language and architecture decisions. Each entry captures **what** was decided, **why**, and **what it affects** — so we don't re-derive context from code later.

## Format

Every decision file follows this structure:

```
# NNNN — Title

- **Status**: draft / accepted / implemented / deferred / superseded-by NNNN
- **Date**: YYYY-MM-DD

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

Decisions can be **superseded** by a newer entry when direction changes.

## Index

| # | Title | Status | Date |
|---|-------|--------|------|
| 0001 | Pending syntax and performance items | draft | 2026-05-17 |
| 0002 | Design principles | accepted | 2026-05-21 |
| 0003 | Standard library roadmap | deferred | 2026-05-29 |
| 0004 | LSP roadmap | deferred | 2026-05-30 |
| 0005 | Backend architecture: KIR + dual backend | draft | 2026-05-31 |
| 0006 | Error handling: ??, try, and Cast unification | accepted | 2026-06-01 |
| 0007 | Trait system redesign | superseded-by 0009 | 2026-06-02 |
| 0008 | KBC bytecode format evolution | partial | 2026-06-02 |
| 0009 | Concepts landing | accepted | 2026-06-03 |
