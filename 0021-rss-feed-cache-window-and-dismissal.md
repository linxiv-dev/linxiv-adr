# ADR 0021: RSS feed cache window, dedup, and dismissal semantics

## Status

Accepted

## Context

The home-page RSS/Atom feed (`GET /api/feed`, `src-tauri/src/route/feed.rs` +
`crates/core/src/storage/queries/rss.rs`) was originally backed by an in-memory
per-URL cache of the raw upstream response, keyed by a short TTL. Two problems
with that design:

1. **Unbounded growth / no persistence.** The cache held the last raw fetch
   only — nothing survived a restart, and there was no way to retain more
   history than one response.
2. **Empty upstream responses clobbered good data.** arXiv (and other feeds)
   can legitimately return zero new entries for a period (e.g. arXiv's weekend
   `skipDays`). A cache keyed on "last raw response" would show an empty feed
   during that window even though nothing was actually wrong.

The redesign (2026-07-19) replaced the raw-response cache with a durable
per-entry table, `RSS_CACHE_ENTRY`, pruned to a configurable retention window
(`rss_cache_retention_days`, default 30). This ADR records the decisions made
in that redesign that aren't obvious from reading the code in isolation.

## Decision

### Additive merge, never overwrite

Each fetch inserts new entries into `RSS_CACHE_ENTRY` via `INSERT OR IGNORE`
against a `(FEED_URL, DEDUP_KEY)` unique index (`rss::merge_cache_entries`).
An empty or short upstream response just merges in zero new rows — the
response sent to the client is always built from the DB window
(`load_cache_entries`), never from the raw fetch directly. This is what makes
the empty-fetch problem above a non-issue.

### Dedup key precedence

`feed::to_cache_entry` picks a dedup key in this order: arXiv `id+version` (a
later version is a genuinely distinct entry and must not overwrite the
earlier one), else the entry's link, else its title. `None` (entry dropped)
only when none of those three are present.

`source_id` (the arXiv id without version) is stored on the row too, but it
is *not* consulted for cache-level dedup or dismissal — it's carried purely
for reference/debugging. Durable dismissal is checked directly against
`RSS_PAPER_ROOTS` / `RSS_PAPER` in `annotate_and_filter`, independent of
whether a cache row exists.

### Retention window is measured by fetch time, not published time

`load_cache_entries` and `prune_cache_entries` window on `FETCHED_AT`, not
`PUBLISHED_AT`. An arXiv entry's `published` is its *original* submission
date — a just-updated v2/v3 of a paper from years ago would have an old
`published` date despite being fetched today. Windowing on `PUBLISHED_AT`
would drop such an entry from the response on the very fetch that added it.
The response is still *sorted* newest-published-first
(`COALESCE(PUBLISHED_AT, FETCHED_AT) DESC`); only the retention cutoff uses
fetch time.

### Dismissed entries are not exempt from window pruning

Permanently-dismissed entries (`RSS_PAPER_ROOTS.REMOVAL_TYPE = 'DOI'`) age
out of the cache window like any other row — pruning does not check
dismissal state. This is intentional: the durable fact of dismissal lives in
`RSS_PAPER_ROOTS`, not in the presence of a `RSS_CACHE_ENTRY` row. If a
dismissed entry's cache row is pruned and the paper is later re-fetched
upstream, `annotate_and_filter` still filters it out via
`blocked_source_ids`/`dismissed_versions` — nothing depends on keeping the
cache row around. Covered by
`window_excludes_old_entries_even_when_permanently_dismissed` in `rss.rs`.

> Note: `DONE.md`'s original changelog entry for this redesign stated
> dismissed entries are "exempt from the window prune and kept indefinitely."
> That was aspirational/inaccurate at ship time — the implemented behavior,
> and the behavior this ADR documents, is that dismissal state and cache rows
> are decoupled, and cache rows always age out.

### Two-tier dismissal

`rss::dismiss` supports two removal types on a per-request `permanent` flag:
- **Permanent** (`RSS_PAPER_ROOTS.REMOVAL_TYPE = 'DOI'`) blocks the whole
  paper, every version, forever.
- **Per-version** (`RSS_PAPER.REMOVAL_TYPE = 'VER'`, the default) dismisses
  only the exact `(source_id, version)` row. A later, higher version is a
  distinct row and resurfaces undismissed — this is deliberate: a new version
  of a paper is new information the user hasn't seen yet.

### Load cap vs. response cap, and truncation ordering

`load_cache_entries` pulls up to `MAX_LOADED_ENTRIES` (500) rows from the DB;
`annotate_and_filter` truncates the *filtered* result to 200 before it goes to
the client. Truncation happens after filtering, not before, so dismissed/
rule-hidden entries don't eat into the 200 the client actually sees — the gap
between 500 and 200 is the headroom for that filtering.

**Accepted trade-off:** because the DB load is capped at 500 and sorted
newest-published-first, a just-fetched update to a very old paper could in
principle sort past the 500-row cap on a very high-volume feed/long retention
window, and never reach the truncation step at all. Not addressed — revisit
if `MAX_LOADED_ENTRIES` needs to grow for a real feed source.

### Throttle failure behavior

`LAST_FETCH` (in-memory, per-URL: last-fetch `Instant` + channel title) throttles
upstream calls to once per `CACHE_TTL` (5 min). No throttle entry is written on
a failed fetch, so the *next* request retries immediately rather than waiting
out the TTL while upstream is down. A failed fetch falls back to serving the
existing DB window rather than erroring outright; the fetch error only
surfaces to the client if that window is also empty (nothing cached yet, or
fully pruned).

## Consequences

### Positive
- Feed history survives restarts and short upstream outages/blank responses.
- Dismissal is a pure DB-state check, independent of cache row lifecycle —
  no special-casing needed when cache rows are pruned or re-fetched.
- Retrying immediately after a failed fetch (rather than waiting out the TTL)
  keeps the feed responsive when upstream recovers quickly.

### Negative / limits
- `RSS_CACHE_ENTRY` can retain rows for dismissed papers until they age out of
  the retention window — not an immediate cleanup, just a bounded one.
- The 500-row load cap can theoretically starve the 200-row response of a
  genuinely-newest entry on a high-volume feed (see load cap trade-off above).
- `LAST_FETCH` is an unbounded in-process map; fine while one home-feed URL is
  in practical use, would need bounding if the app grows multi-feed support.

## References

- `src-tauri/src/route/feed.rs` — `get_feed`, `annotate_and_filter`, `to_cache_entry`, `LAST_FETCH`
- `src-tauri/crates/core/src/storage/queries/rss.rs` — `merge_cache_entries`, `load_cache_entries`, `prune_cache_entries`, `dismiss`, `blocked_source_ids`, `dismissed_versions`
- `DONE.md` — RSS Feeds — persistent cache redesign (2026-07-19)
