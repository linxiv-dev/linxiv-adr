# ADR 0010: Service layer is the only boundary for consumers

## Status

Accepted

## Context

Originally written about the Python backend: `api/app.py` imported write functions directly from `storage.db`, bypassing `service/paper.py`, which created two parallel write paths into storage — any invariant added to the service save path (e.g. opportunistic version capture on every paper save) would silently be skipped by the direct callers.

The Rust port replaced the single FastAPI consumer with **three** consumers of `linxiv-core`: the Tauri route dispatcher (`src-tauri/src/route/`), the CLI (`src-tauri/crates/cli/`), and the MCP server (`src-tauri/crates/mcp/`). The failure mode the rule prevents got three times worse: a rule enforced in only one consumer (validation, vault cleanup, note-merge on search) silently diverges on the other two. The 2026-08-06 architecture review documented exactly this happening — see TODO.md "Architecture review findings", sections "Domain operations assembled inside route handlers" and "Validation and error mapping repeated per consumer".

## Decision

Consumers of `linxiv-core` — the route dispatcher, the CLI, and the MCP server — call **`service::*` only**. Both lower layers are behind that boundary:

- **`storage::*`** — persistence. No consumer calls `storage::queries::*` or opens connections directly.
- **`sources::*`** — provider fetch (arXiv, OpenAlex, Crossref, feeds). No consumer orchestrates fetch pipelines itself; a fetch-and-store operation is a service function so all three surfaces get the same pipeline.

Domain rules (validation, guards, side effects like vault cleanup) live behind the service seam, never in a route handler, CLI command, or MCP tool.

**Accepted carve-out (unchanged from the Python era): thin pass-through delegates in `service/` are deliberate, not redundancy.** Roughly 25 of the 121 `service/` functions are one-line delegations to storage. They exist so the boundary holds and so business logic can be added in one place later without changing call sites. Do not "clean them up" — and do not bypass them because they look hollow; bypassing is what makes them look pointless.

## Current state (2026-08-10)

The rule is stated as binding, and it is currently widely violated: the review inventoried ~50 reach-past call sites across route/CLI/MCP into `storage::` and `sources::`, with `src/route/feed.rs` the extreme case (no service layer at all, which is why the RSS feed is GUI-only). The closure plan is the "Consumers reaching past the service layer" item and its siblings in TODO.md "Architecture review findings" — this ADR is the rule those items enforce, not a description of today's code.

## Consequences

### Positive
- A single path into storage and sources per operation; an invariant added in `service/` applies to all three surfaces automatically.
- Surface parity (route/CLI/MCP exposing the same behavior) becomes a matter of wiring, not of re-implementing pipelines per surface.

### Negative / limits
- The delegate wrappers add a layer that does nothing today; that overhead is chosen.
- Until the reach-past inventory is closed, the rule and the code disagree; new code must follow the rule even where neighboring code does not.

## References

- `src-tauri/src/route/`, `src-tauri/crates/cli/`, `src-tauri/crates/mcp/` — the three consumers
- `src-tauri/crates/core/src/service/` — the boundary
- ADR 0022 — the storage-side rule (what the storage seam itself looks like)
- TODO.md "Architecture review findings (2026-08-06)" — reach-past inventory and closure items

> Re-grounded on the Rust codebase, 2026-08-10 (the decision predates the Rust port; the boundary was widened from `api/app.py`-vs-`storage.*` to all three consumers vs `storage::` + `sources::`).
