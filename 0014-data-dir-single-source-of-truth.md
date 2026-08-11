# ADR 0014: LINXIV_DATA_DIR is the single source of truth for the runtime data dir

## Status

Accepted

## Context

linXiv writes several kinds of per-user runtime data to disk: the SQLite database (`papers.db`), managed PDFs (`pdfs/`), user settings (`user_settings.json`), the embedded-editor LaTeX vaults (`vaults/`), and the arXiv rate-limit timestamp (`.arxiv_ratelimit`). All four processes — the Tauri desktop app, the headless CLI (`crates/cli`), the MCP server (`crates/mcp`), and the migration tool (`crates/migrate`) — must agree on where the library lives.

Previously `config.data_dir()` returned `$LINXIV_DATA_DIR` when set (Tauri sets it on the spawned subprocess) and **otherwise fell back to the repository root** (`_PROJECT_ROOT`). That fallback was convenient in development but had real problems:

- It conflated the **source tree** (a code artifact) with **user data** (runtime state). A developer running the CLI or MCP directly accumulated `papers.db`, `pdfs/`, and `.arxiv_ratelimit` inside the checkout.
- The location was resolved inconsistently: some sites (`storage/db.py`, `storage/config/core.py`) froze `DB_PATH` once at import time; others resolved per call.
- Not every path went through `data_dir()` at all. The Obsidian vault and the arXiv rate-limit file were pinned to `Path(__file__).parent.parent` (the source tree), so they did not follow `$LINXIV_DATA_DIR` even in a packaged build — notes and the rate-limit file were written next to the source instead of into the user's data dir.

A full audit of every explicit and implicit hardcoding of the DB location preceded this decision (working note `PATHING_AUDIT.md` at the repo root).

## Decision

`$LINXIV_DATA_DIR` is the single source of truth for the runtime data dir. The repo root is no longer a fallback for runtime data.

`crates/core/src/config.rs` exposes:

- **`data_dir()`** — returns `PathBuf::from($LINXIV_DATA_DIR)` if set, else the built-in default. Resolved on **every call** so it tracks the env var dynamically; never cached at import.
- **the default fallback** (inlined in `data_dir()`, via the `directories` crate) — the OS per-user app-data dir for the bundle identifier `com.linxiv.app`, matching Tauri's `app_data_dir()` per platform (Linux: `$XDG_DATA_HOME` or `~/.local/share`; macOS: `~/Library/Application Support`; Windows: `%APPDATA%`). Used only when `$LINXIV_DATA_DIR` is unset (dev / CLI / MCP launched without Tauri). Never the repo.
- **`init_data_dir()`** — resolves `data_dir()`, writes it back to `$LINXIV_DATA_DIR` (pinning it for the process lifetime and inherited by child processes), and `mkdir`s it. Called once at startup of **every** entry point: the Tauri app's `AppState::new` (`src-tauri/src/state.rs`), the CLI's `Ctx::open` (`crates/cli/src/ctx.rs`), the MCP server's init (`crates/mcp/src/main.rs`), and `crates/migrate/src/main.rs`. This is what "initialize the data dir on any run" means in practice.
- **`repo_dir()`** — the source-tree root, for developer/repo artifacts only. The dev `.env` stays repo-anchored: `ENV_PATH = repo_dir() / ".env"`. Deliberately distinct from `data_dir()`.
- Read-only bundled resources (SQL schema, default settings) are now compiled into the binary via `include_str!` (`storage/schema.rs`; `BUNDLED_DEFAULTS` in `config.rs`), replacing `resources_dir()`. A separate axis from `data_dir()`; unchanged by this decision.

All runtime paths resolve through `data_dir()` per use:

| Path | Resolver |
|---|---|
| `papers.db` | `config::db_path()` |
| `pdfs/` | `config::pdf_dir()` |
| `vaults/` | `config::vault_dir()` |
| `.arxiv_ratelimit` | `sources::http` (`cooldown_remaining`, `record_ratelimit`; callers pass `config::data_dir()`) |
| `user_settings.json` | `config::UserSettings` (`load`/`save`) |

Tests pin `LINXIV_DATA_DIR` to a temp dir (the `EnvVarGuard` in `src-tauri/src/route/storage.rs` tests, sequential `env::set_var` in `crates/cli/src/cmd/misc.rs` and `crates/core/src/config.rs` tests, and `.env("LINXIV_DATA_DIR", …)` on spawned processes in `crates/cli/tests/`) so they never read or write the real OS data dir. Because `pdf_dir()` resolves dynamically, per-module path overrides are unnecessary.

The alternative — keeping the repo-root dev fallback — was rejected because it is the root cause of source-tree pollution and of the "which DB am I actually using?" ambiguity that motivated the audit.

## Consequences

### Positive

- One unambiguous answer to "where is the library?" across all four processes.
- Directly-launched CLI/MCP/API use the same OS location as the packaged app, instead of scattering data into the checkout.
- The editor vaults and arXiv rate-limit file now follow `$LINXIV_DATA_DIR` like everything else.
- `init_data_dir()` guarantees the directory exists before any DB/PDF/vault access on any run.

### Negative / limits

- `init_data_dir()` pins `$LINXIV_DATA_DIR` for the process lifetime. Tests that redirect the data dir must set/restore the env var (it is process-global, so the Rust tests guard and serialize it); documented in the function's docstring.
- The dev `.env` and runtime data now live in different places by design. Anyone expecting a single repo-local folder must look in the OS app-data dir (or set `$LINXIV_DATA_DIR`).
- The DB connection is opened once at startup at the then-current `db_path()` (`AppState::new`, `Ctx::open`, the MCP server init). Reopening per use is the prerequisite for any runtime DB relocation; it is a behaviour change and was deliberately left out of this ADR.
- A repo-dir ⇄ data-dir **sync** feature (dev ⇄ installed data) is potentially destructive (direction, conflict policy, atomicity unresolved) and must be designed before implementation.

## References

- `src-tauri/crates/core/src/config.rs` — `data_dir`, `init_data_dir`, `db_path`, `pdf_dir`, `vault_dir`, `UserSettings`
- `src-tauri/crates/core/src/sources/http.rs` — `cooldown_remaining`, `record_ratelimit` (`.arxiv_ratelimit`)
- `src-tauri/src/state.rs` (`AppState::new`), `src-tauri/crates/cli/src/ctx.rs` (`Ctx::open`), `src-tauri/crates/mcp/src/main.rs`, `src-tauri/crates/migrate/src/main.rs` — `init_data_dir()` startup calls
- `src-tauri/src/integrations.rs` — `install_mcp` writes `LINXIV_DATA_DIR` (from `app_data_dir()`) into installed MCP client configs
- `src-tauri/src/route/storage.rs`, `src-tauri/crates/cli/src/cmd/misc.rs`, `src-tauri/crates/cli/tests/` — env-pin test isolation

> Re-grounded on the Rust codebase, 2026-08-10 (the decision predates the Rust port).