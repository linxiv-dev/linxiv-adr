# ADR 0026: The knowledge graph as a React component over a typed Rust payload

## Status

Accepted. Supersedes [ADR 0012](0012-graph-iframe-postmessage-protocol.md).

## Context

The Knowledge Graph was the one screen in the app that was not part of the app.
It lived in `public/graph/` as a static HTML page plus a 2,400-line browser
script and two vendored library bundles (`cytoscape.min.js`, `d3.v7.min.js`),
loaded into an `<iframe>` by `GraphPage.tsx` and talking to its host over the
postMessage protocol ADR 0012 describes.

Everything that was awkward about the graph followed from that boundary rather
than from the graph itself:

- **Nothing type-checked it.** `graph.js` was outside the bundler, and
  `GET /api/graph` was assembled as an inline `serde_json::Value` with no Rust
  struct behind it, so the two ends of that payload were joined by nothing. The
  hand-written mirror in `src/types/api.ts` had already drifted — a paper node's
  `id` was declared `string` where the backend emitted a bare integer, and eight
  of the fields the payload carried were missing altogether.
- **It could not use the app's own transport.** An iframe cannot `invoke`, so it
  fetched over the `linxiv://` custom scheme, which meant bridging four `/api`
  GET endpoints into that scheme. It also could not tell WHICH backend to talk
  to: `tauri dev` and browser dev both serve the guest from
  `http://localhost:5180`, so the host had to name the transport in the frame's
  `src` — sniffing sent the graph, alone in the app, to the dev server's
  database under `tauri dev`.
- **It could not use the app's own state.** The theme was pushed across on every
  palette edit; the load state came back as a `graph_loaded` reply the host
  turned into a spinner, with an eight-second fallback for a reply that never
  arrived; the selection existed in two copies that had to be kept in step.
- **Key events do not cross a frame boundary.** Every app-wide shortcut was dead
  on `/graph` alone from the first click on the canvas, so the host shipped its
  shortcut vocabulary into the guest and the guest handed matching keydowns
  back. Tauri's own capability docs note the related hazard: on Linux and
  Android the framework cannot distinguish a request from an embedded `<iframe>`
  from one made by the window itself.
- **It could only be tested through a simulator.** The one test file drove the
  real script inside a `vm` behind hand-written DOM, cytoscape and d3 stubs —
  about 4,000 lines of harness for logic that is pure.

## Decision

Draw the graph in the app. Split the work by who owns the knowledge:

**Rust owns every derivation the database can answer.** `linxiv_core::graph`
returns one typed `GraphView` — papers, authors, tags, edges, categories and
project options in a single reply — with the derivations resolved before the
payload is sent: the `0001-01-01` "no date" sentinel folded to null, tag
spellings resolved to `TAG.TAG`, per-paper tags deduped on their normalized key,
the lowercased author index the Author filter matches on, each author/tag node's
degree, the reading-list marker dropped from project tags, and a flag on each
project saying whether any paper on this canvas belongs to it. The wire types are
`#[derive(TS)]` and generated into `src/types/generated.ts`, so `npm run
types:check` is the drift check.

**TypeScript owns what only the DOM knows**, in pure modules under
`src/lib/graph/`: which nodes a filter matches (`filter.ts`), where a node starts
(`layout.ts`), how the viewport frames around the panel column (`fit.ts`), what
the hover inspector says (`tooltip.ts`), and the cytoscape stylesheet resolved
from the live theme (`style.ts`). Each is tested directly.

**One component stays imperative.** `GraphCanvas.tsx` owns the cytoscape
instance and the d3-force simulation, because those are two libraries with
mutable state that expect to be driven rather than re-rendered. It rebuilds only
when the payload changes; theme, forces, filter and selection drive the live
instance in place.

Filtering deliberately did **not** move to Rust. A paper the filter excludes is
still drawn — as an 8% ghost, pinned in the layout — so "matched" is a rendering
state, not a `WHERE` clause. Making it a query would mean the excluded papers
never arrive, and with them would go the ghost, the layout they hold, and the
counts the panels report about what is being held back.

**The graph never reloads itself.** Every other screen refetches as soon as its
data changes somewhere else, and that is right for them: a paper list just
re-renders with a new row, so being up to date costs nothing. The graph is not
free. A new payload rebuilds the force simulation, the physics re-runs, and every
node moves — so for this screen "refresh automatically" means "silently rearrange
the picture the user was working with", and if it lands mid-drag it pulls the
node out from under the cursor.

So the graph opts out and asks instead. When an operation elsewhere changes what
`/api/graph` would return, the invalidation registry in
`src/lib/paperMutations.ts` lights a dot on the Refresh button — *there is newer
data, load it when you are ready* — and nothing on screen moves until the user
clicks it. The dot is cleared by data ARRIVING, whichever control fetched it, so
it never contradicts what is drawn.

react-query starts fetches on its own in three ways, and all three are shut off
deliberately:

| Trigger | What it would do | Setting |
|---|---|---|
| An invalidation elsewhere | refetches any *mounted* query at once, whatever its `staleTime` | the graph key is invalidated with `refetchType: "none"` — mark stale, do not fetch |
| The window regains focus | refetches if the query is stale | `refetchOnWindowFocus: false` |
| The network reconnects | refetches if the query is stale | `refetchOnReconnect: false` |

`staleTime: Infinity` is a fourth guard behind those. Each is a separate door,
and opening any one of them brings the whole problem back — note especially that
the shell keeps this page mounted for the rest of the session once `/graph` has
been visited, so "mounted query" here means *forever*, not *while you are looking
at it*. This is not a micro-optimisation; it is the difference between a canvas
the user can arrange and one that rearranges itself without being asked.

## Consequences

### Positive
- The payload is typed end to end and the drift check is a build step.
- The graph reaches the backend through `apiFetch` like every other screen: no
  custom-scheme `/api` bridge, no transport parameter, no partial-failure
  protocol across four requests.
- Loading, empty and error states are react-query's; the `graph_loaded` reply
  and its eight-second fallback are gone.
- App-wide shortcuts work on `/graph` because they are the window's own.
- cytoscape and d3-force are npm dependencies the bundler can version, audit and
  tree-shake, instead of two checked-in minified files.
- The filter is a pure function with direct tests instead of a `vm` simulator.

### Negative / limits
- cytoscape and d3-force are ~470kB that only one screen needs, and this page is
  imported eagerly because the shell keeps it alive from boot. `GraphCanvas` is
  therefore `React.lazy`: the two libraries land in their own chunk, fetched on
  the first visit to `/graph` — the same point the old iframe used to load them,
  so nobody pays for the graph without opening it. The boundary is the canvas
  rather than the page, since the page itself must exist from boot.
- `GraphCanvas` mixes React and two imperative libraries, which is inherently
  the trickiest file in this area. The rule that keeps it honest is that the
  build effect depends on the payload alone and every other input drives the
  live instance through its own effect.
- `AppShell` still keeps the page alive behind `display: none` so the settled
  layout survives navigation, so the canvas still has to defer a fit taken while
  its container is 0×0.

## References

- `src-tauri/crates/core/src/graph.rs` — the typed payload and its derivations
- `src-tauri/src/route/graph.rs` — `GET /api/graph`
- `src/components/graph/GraphCanvas.tsx`, `GraphPanels.tsx`
- `src/lib/graph/` — the pure modules and their tests
- [ADR 0012](0012-graph-iframe-postmessage-protocol.md) — the protocol this replaces
- Tauri capabilities: <https://tauri.app/security/capabilities/>
