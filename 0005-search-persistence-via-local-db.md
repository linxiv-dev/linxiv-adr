# ADR 0005: Search state persisted in local database, not keep-alive component

## Status

Accepted

## Context

The Search page was initially kept alive via CSS hide/show (component never unmounts) to preserve in-flight mutations and results across navigation. This works but ties up memory and doesn't survive app restarts.

The user's workflow often involves large batch searches (100+ results) against rate-limited APIs (arXiv, OpenAlex). Re-running those searches on every session is impractical.

## Decision

Search state is persisted in two local SQLite tables:

- **`SEARCH_HISTORY`**: a log of past searches (clauses, source, maxResults, timestamp). Surfaced as autocomplete suggestions in the clause input field after typing characters.
- **`SEARCH_STATE`** (single row; `RESULTS_JSON`): the current working set of results, which can be a mix accumulated from multiple searches (deduplicated by source ID). Persists across restarts. Cleared explicitly by the user. Append is also a second class behavior, users choose to use, papers don't pile up with out them being aware.

The Search page restores from `SEARCH_STATE` on mount — it no longer needs to be keep-alive. 
The Graph page remains keep-alive (iframe reload is unavoidable). But this is being reconsidered, a refresh button doesn't look good in UI, manually refreshing via right-click is impractical.

A plain **Search** button replaces the current working set. A **+** button adjacent to it appends new results to the existing set. Minus (-) for clearing results, to match.

Users can configure how many result batches to retain in advanced settings (default: 1). Not implemented, and not sure if I want to implement before reducing complexity of UX overall.

## Consequences

### Positive
- Large searches run once; results survive restarts without re-hitting rate limits.
- Enables cross-source mixing (arXiv + OpenAlex results in one working set).
- Enables future features: sort/filter on the full cached result set without a network call.
- Removes the keep-alive complexity for the Search component.

### Negative / limits
- Requires a local db migration to add the two tables.
- Result set can grow stale (paper may have been updated since it was fetched); this is a nitpicky complaint, and what most users would expect explicitly. 

## References

- `src-tauri/crates/core/src/storage/queries/search_history.rs`, `search_state.rs` — where the tables live (DDL in `src-tauri/crates/core/sql/tables/SEARCH_HISTORY.sql`, `SEARCH_STATE.sql`)
- `src/pages/SearchPage.tsx` — consumer (via `src/api/searchState.ts`)

> Re-grounded on the Rust codebase, 2026-08-10 (the decision predates the Rust port).
