# ADR 0025: In-process Rust backend — no sidecar

## Status

Accepted. Supersedes [ADR 0018](0018-sidecar-lifecycle-and-reaping.md).

## Context

ADR 0018 built the machinery to spawn, health-check, identify, and reap a Python API sidecar process (`make_health_token`, `wait_for_api`, `reap_api`, `sweep_orphaned_sidecars`, `/api/health`). The Rust port removed the process boundary itself: `linxiv-core` is linked into the Tauri app, so there is nothing to spawn, no HTTP hop, and no orphan to sweep. All of 0018's machinery was deleted with the Python backend.

## Decision

The backend runs **in-process**. The webview calls the single Tauri `api` command (see ADR 0010 / CONTEXT.md § API Layer), which dispatches through `src-tauri/src/route/` into `linxiv-core` sharing one `AppState` (`src-tauri/src/state.rs`). Lifecycle is the app's lifecycle — no health tokens, no reaping, no startup wait.

What still leaves the process, deliberately none of it a sidecar:

- **CLI and MCP server** are *installed* standalone binaries (`src-tauri/src/integrations.rs` writes them into MCP client configs with `LINXIV_DATA_DIR`); the app never spawns or supervises them — they are independent processes sharing the SQLite file.
- **The pdf-metadata worker** is a short-lived subprocess (`linxiv pdf-meta <path>`, spawned per call with a timeout from `crates/core/src/sources/pdf_metadata.rs`) to isolate pdfium crashes; it exits when its one job is done, so no reaping machinery is needed.

## Consequences

- No sidecar failure modes: no port to bind, no orphaned child on crash, no startup race. The dev-only exception is `src-tauri/src/bin/dev_server.rs`, which serves `route()` over HTTP for browser dev and is run manually, not spawned by the app.
- Concurrency discipline moved from "processes over HTTP" to "one shared connection behind a mutex" — holding the DB mutex across `await` points is the hazard 0018 never had (see the two-phase import notes in the TODO's architecture findings).

## References

- `src-tauri/src/main.rs`, `src-tauri/src/state.rs` — in-process wiring (both note the sidecar's removal)
- `src-tauri/src/integrations.rs` — CLI/MCP install (not spawn)
- ADR 0019 — CLI/MCP bundled in the app, unsigned
