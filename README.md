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

| # | Title | Status |
|---|-------|--------|
| 0001 | Design principles | accepted |
| 0002 | Trait system redesign | draft |
| 0003 | Error handling: ??, try, and Cast unification | accepted |
| 0004 | Backend architecture: KIR + dual backend | draft |
| 0005 | KBC bytecode format evolution | partial |
| 0006 | Standard library roadmap | deferred |
| 0007 | LSP roadmap | deferred |
| 0008 | Pending syntax and performance items | draft |
