# 0004 — LSP Roadmap

- **Status**: deferred
- **Date**: 2026-05-30

## Context

The bs-side LSP (C++) provides diagnostics, completion, go-to-definition, hover, document symbols, and signature help. The sh-side has no LSP yet. A self-hosted LSP is needed for editor integration without depending on the C++ binary.

## Decision

### Phase A — sh-side LSP core

Implement in `lsp/` module:
- JSON-RPC 2.0 message handling
- textDocument/didOpen, textDocument/didChange (document sync)
- textDocument/diagnostics (reuse checker pipeline)

### Phase B — Language features

- Completion: scope-aware, type-aware, import-aware
- Go-to-definition: cross-module resolution
- Hover: type information + doc comments
- Document symbols: AST outline
- Signature help: parameter hints
- Code actions: quick-fix for common errors (e.g., add `using io;`)

### Phase C — Advanced features

- Rename symbol
- Find all references
- Folding ranges
- Semantic tokens (full syntax highlighting)
- Inlay hints (type annotations)

## Dependencies

- Import system (already working in sh compiler)
- Module loader for multi-file awareness
