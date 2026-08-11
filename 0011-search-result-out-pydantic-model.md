# ADR 0011: SearchResultOut as the single wire shape for search and fetch results

## Status

Accepted

## Context

Originally written about the Python backend, where `api/app.py` had three divergent serialization paths for search/fetch responses (arXiv search, arXiv fetch, OpenAlex search) that disagreed on `source_id` format (bare vs namespaced) and `entry_id` format (namespaced string vs URL), with no model to validate against.

The decision survived the Rust port: the type is now `SearchResultOut` in `src-tauri/crates/core/src/models.rs` ("SERIALIZER 1"), with `impl From<PaperMetadata>` as the single mapping point, and a route test pinning the exact wire shape and key order. It is deliberately distinct from `PaperDetails` (D16 — search results and library papers are different views; do not unify).

## Decision

All search/fetch-style responses that return provider metadata use `SearchResultOut`, produced only via `SearchResultOut::from(PaperMetadata)`.

Key choices (unchanged from the original decision, now in Rust):

- **`source_id` is always bare** — the namespace is stripped via `models::strip_namespace` (`"arxiv:2204.12985"` → `"2204.12985"`).
- **`entry_id` is the full namespaced source_id** — the one field that keeps the namespace; the frontend treats the two as interchangeable and never builds URLs from `entry_id`.
- **`published` serializes the `date.min` sentinel as `""`**, else ISO date.
- Typed source errors map to precise statuses: the Python exception hierarchy became `CoreError` variants (`ArxivNotFound`, `OpenAlexNotFound`, `OpenAlexInputError`, …) with `CoreError::http_status()` as the single mapping (`src-tauri/crates/core/src/error.rs`).

**Scope: all three surfaces.** The canonical shape governs the route dispatcher, the CLI, and the MCP server. What a search returns must not depend on which surface ran it.

## Current state (2026-08-10)

Only the route layer complies (`src/route/sources.rs` — all four call sites go through `SearchResultOut::from`). `linxiv search` and MCP `search_papers` still serialize raw `PaperMetadata`, so the same query yields `url`/`category`/namespaced `source_id` on those surfaces and `paper_url`/`primary_category`/bare `source_id` on the GUI. Converting CLI and MCP to the canonical shape is tracked debt (TODO.md "Architecture review findings" — the ADR-0011 item and the serializer-convention items). New search-shaped endpoints on any surface must use `SearchResultOut` from the start.

## Consequences

### Positive
- One struct, one `From` impl; `source_id`/`entry_id` semantics are consistent wherever the rule is applied, and the wire shape is pinned by test.

### Negative / limits
- Converting the CLI and MCP outputs is a breaking change for anything parsing their current key names (goldens and MCP tool schemas will need the same pass).

## References

- `src-tauri/crates/core/src/models.rs` — `SearchResultOut`, `strip_namespace`, D16 note
- `src-tauri/src/route/sources.rs` — the compliant consumer + wire-shape test
- `src-tauri/crates/core/src/error.rs` — `CoreError` variants and `http_status()`
- ADR 0002 — namespaced source IDs
- ADR 0010 — service layer as the consumer boundary

> Re-grounded on the Rust codebase, 2026-08-10 (the decision predates the Rust port; scope widened from the FastAPI routes to all three surfaces).
