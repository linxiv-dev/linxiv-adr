# ADR 0022: Storage-query rule for the Rust codebase

## Status

Accepted. Supersedes [ADR 0007](0007-q-class-for-storage-queries.md).

## Context

ADR 0007 defined the Python storage-query rule around `storage/config/queries.py`: a composable `Q` predicate builder for single-table lookups, named `_UPPER_SNAKE_SQL` constants for JOINs, no per-row accumulation, and `_connect()` private to the module. The Rust port deleted all of that machinery. What exists now is `crates/core/src/storage/queries/` — one module per entity (`paper.rs`, `project.rs`, `note.rs`, `tag.rs`, …, ~6,300 lines), plain `pub fn`s taking a `rusqlite::Connection`, and `crates/core/src/storage/db.rs` owning connection opening (`open` / `open_in_memory`, which set `PRAGMA foreign_keys = ON`) and SQL value conversions.

## Decision

Codify current practice rather than port the Python machinery:

1. **SQL lives inline in named query functions.** Each query is a `pub fn` in the entity's module under `storage::queries`, with the SQL string written directly in `conn.prepare(...)` / `conn.execute(...)`. rusqlite's prepared statements and `params![]` provide the parameterisation the `Q` class existed to guarantee — string interpolation of values into SQL remains forbidden.
2. **A named `const *_SQL` is used only when the string is shared** by more than one function (e.g. `TAG_FK_BY_LABEL_SQL` in `tag.rs`). Single-use SQL stays inline with its function.
3. **Rows map into typed structs** (`*Details` and friends in `crates/core/src/models.rs`, or module-local row structs) inside the query function. Callers never see `rusqlite::Row`.
4. **No per-row accumulation of relational data** (carried from 0007 unchanged). Related data is fetched in a single JOIN or a single `IN (…)` batch, not a loop of single-row queries. Known violations — `service/paper.rs`'s `paper_by_id`, `get_categories`, and `get_papers_by_tag` load the whole library and filter in Rust — are tracked in TODO.md (“Architecture review findings” → big-modules items).
5. **The storage seam is the public functions of `storage::queries`** (carried from 0007). Nothing outside `storage::` opens a raw `rusqlite::Connection` — every connection goes through `storage::db::open`/`open_in_memory` so the pragmas hold. Which *layers* may call the storage seam at all is ADR 0010's rule, not this one.

## Deliberately left open

A move toward systematically named SQL constants, or a query-builder crate, is **not foreclosed** — if the storage layer's readability or line count starts to argue for it, that change is welcome and would supersede this ADR. It is simply not mandated today: adopting either retroactively would make ~6,300 existing lines non-compliant for no behavioral gain.

## References

- `src-tauri/crates/core/src/storage/queries/` — the per-entity query modules
- `src-tauri/crates/core/src/storage/db.rs` — connection opening, pragmas, SQL value conversions
- ADR 0010 — which layers may call the storage seam
- TODO.md “Architecture review findings (2026-08-06)” — tracked violations of rules 4 and 5
