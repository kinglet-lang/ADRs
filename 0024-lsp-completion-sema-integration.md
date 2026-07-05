# 0024 — LSP Completion: Sema-Backed Field-Access Resolution

- **Status**: draft
- **Proposed**: 2026-07-06

## Context

perch (the LSP server, `kinglet-lang/perch`) already injects the completion
cursor as a real `COMPLETION` token into bootstrap's real Scanner token
stream, and bootstrap's real Parser (`at_completion()` / `set_completion()`
in `parse_decl.cc` / `parse_expr.cc`) reports structured `CompletionInfo`
(position kind + context, e.g. `receiver_type`) when it reaches that token.
That much already matches how clangd drives completion off the real Sema
(`Sema::CodeCompleteXxx` callbacks fired from a `code_completion_token`
injected into the real Preprocessor/Parser token stream) — clangd does not
maintain a second parallel implementation for parsing/context detection, and
neither does perch today.

The gap is downstream of the parser. Three pieces of perch's completion
pipeline are hand-maintained approximations of information bootstrap's own
front end already computes authoritatively, and they drift out of sync with
bootstrap whenever a language feature grows:

1. **Keywords** — `token.h` (`TokenType`) + the single `keywords` map in
   `scanner.cc:417-449` is authoritative. `completion_resolver.cc` re-lists
   keywords across four separate hand-written arrays
   (`add_type_keywords` / `add_cast_keywords` / `add_statement_keywords` /
   `add_decl_keywords`); a new keyword must be added to every array it
   belongs to, and a missed one reproduces the `auto`/`bool`/`byte` noise bug
   found in the parameter-name completion regression.
2. **Native namespace members** — `type_checker.cc` / `compiler.cc` (plus the
   `io_intrinsics` table landed in bootstrap PR #50) is authoritative for
   `io`/`fs`/`sys` members. `completion_resolver.cc` re-lists them in three
   separate places (`add_io_members`, `add_namespace_completions`,
   `resolve_namespace_access`). Confirmed live drift: `fs::__listdir` exists
   in `type_checker.cc` but is absent from perch's `fs` completion list.
3. **Scope symbols / member-access type resolution (the largest gap)** —
   `TypeChecker` (`type_checker.h`/`.cc`) is the only place that correctly
   resolves variable scopes, struct field types, method return types,
   generics, and UFCS. perch does not use it for completion at all. Instead
   `analysis.cc`'s `SymbolCollector` hand-walks the AST to approximate a
   symbol table, and `completion_resolver.cc`'s `walk_access_chain()` /
   `member_type()` hand-roll a miniature type resolver that only understands
   plain struct fields and function return types — it has no notion of
   generics, imports, or UFCS, all of which `TypeChecker` already handles
   correctly for the compiler's own diagnostics. This is the actual
   structural difference from clangd: clangd's completion reads `Type`s and
   `Decl`s straight out of `Sema`'s live analysis; perch computes its own
   second, weaker approximation.

## Decision

Phase A (keyword table) and Phase B (native member table generalization)
were considered and explicitly **dropped**: with a single developer as both
author and sole user of perch, the manual-sync cost they eliminate is lower
than the cost of building and maintaining the table infrastructure itself.
A drifted keyword or namespace member list is fixed by editing the one
hand-written array on the spot, when noticed — that is cheaper than a
standing table + accessor + two-repo consumer wiring at this project's
current scale. This ADR proceeds directly to Phase C, which is not a
sync-cost problem but a *correctness* problem: perch's field-access
completion runs its own weaker type resolver and can produce wrong or
missing completions independent of team size.

- **Phase C — Sema completion hook.** Give `Parser` a way to keep parsing
  past a completion point instead of abandoning the current statement:
  wrap the already-parsed partial expression in a new
  `ast::CompletionMarkerExpr` node rather than returning `nullptr` at the
  completion site. Give `TypeChecker` an optional completion callback;
  when its expression dispatch reaches a `CompletionMarkerExpr`, it first
  runs `check_expr()` on the wrapped receiver through the *normal* type-check
  path (so generics/imports/UFCS resolve exactly as they do for a real
  diagnostic), then hands the callback the real `Type` plus the live
  `scopes_` / `method_registry_` at that point in the walk. perch's
  `completion_resolver.cc` becomes a formatting layer over that callback's
  output; `SymbolCollector` and the hand-rolled `walk_access_chain()` /
  `member_type()` are deleted once the callback covers their use cases.

Phase C is split into two sub-phases because it touches `TypeChecker`'s error
recovery model, which is the actual risk surface:

- **C1 — TypeChecker fault tolerance.** `TypeChecker::check()` today assumes
  it is checking a complete, valid program; LSP input is routinely
  incomplete (mid-edit). Before wiring in a completion callback, confirm (and
  where necessary harden) that a `CompletionMarkerExpr` sitting inside an
  otherwise-incomplete AST does not cascade unrelated false diagnostics
  through the existing `error_at()` / recovery paths. This sub-phase produces
  no user-visible behavior change — it is a hardening pass gated on the
  existing sema/parser test suite staying green, plus new fixtures built
  from real incomplete-source LSP scenarios.
- **C2 — Completion callback wiring.** Add the `CompletionMarkerExpr` AST
  node, the `TypeChecker` callback hook, and a dedicated "LSP mode" entry
  point (Scanner+Parser+TypeChecker given source + cursor position, isolated
  from the `kinglet build`/`kinglet check` entry points so LSP-only AST
  states never influence normal compilation). Rewrite perch's
  `completion_resolver.cc` field-access / scope-symbol paths to consume the
  callback's structured output.

Open design questions to resolve during C1/C2 implementation (not blocking
this ADR's acceptance, but blocking C2's start):

1. After a `CompletionMarkerExpr`, does parsing resume within the same
   statement, or does the statement stop there while sibling
   statements/declarations still parse normally? First cut: the latter
   (matches current behavior for `nullptr`-returning completion sites) —
   full same-statement multi-cursor recovery is out of scope for v1.
2. Does "LSP mode" `TypeChecker` need genuinely incremental/fault-tolerant
   checking (survive arbitrary malformed trailing input), or is
   "tolerate one well-formed marker node" sufficient for the completion
   scenarios perch actually hits? Decide via the C1 fixture set before
   starting C2.
3. Nested completion points (e.g. `foo(bar.█` — a marker inside another
   unfinished call) — v1 targets the outermost recognizable completion site
   only, matching every existing `at_completion()` call site's current
   behavior; nested handling is a later iteration.

## Consequences

- Once C2 lands, a new struct field, method, generic instantiation, or UFCS
  extension method automatically appears correctly in perch's field-access
  completion — no perch-side change needed, because completion reads the
  same `Type` the compiler's own diagnostics read.
- `analysis.cc`'s `SymbolCollector` and `completion_resolver.cc`'s
  `walk_access_chain()` / `member_type()` are deleted after C2, removing the
  second, weaker type-resolution implementation perch currently maintains.
- C1 is a `TypeChecker` hardening change with no immediate user-visible
  payoff on its own; it must land and hold the full sema/parser suite green
  before C2 begins, since C2's correctness depends on C1's fault-tolerance
  guarantees.
- This is a bootstrap-core change (Sema error-recovery model), not a
  perch-only change — regressions here risk degrading normal compiler
  diagnostic quality, not just LSP completion. Treat C1 with the same
  scrutiny as any other `TypeChecker` change, independent of the LSP
  motivation.
- Keyword and native-member completion lists remain hand-maintained arrays
  in `completion_resolver.cc` for now (Phase A/B dropped per above). If
  contributor count grows beyond the current single-developer scale, revisit
  Phase A/B — the drift risk they address becomes real again once more than
  one person is adding language features without also touching perch.
