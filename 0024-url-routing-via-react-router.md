# ADR 0024: URL-based routing via React Router

## Status

Accepted. Supersedes [ADR 0001](0001-app-shell-tab-navigation.md).

## Context

ADR 0001 (PyQt era) kept a `QStackedWidget` index-based back stack with explicit pairwise wiring, and deliberately rejected "a generic router, route table, or structured navigation stack" as over-engineering for the widget shell. The React port removed the premise: the app is a webview, where a router is the native platform feature, not an abstraction added on top.

## Decision

Navigation is a `createBrowserRouter` route table (`react-router`) in `src/App.tsx` — `library/:sfk`, `projects/:id`, and so on. Back/deep-return is browser history, with advisory location state (e.g. `{ fromProjectId }`, per ADR 0003) where a return target needs context the URL doesn't carry. The Graph page is the one deliberate exception to plain routing: its route renders `element: null` and the graph iframe is kept alive in the shell, so switching tabs doesn't rebuild the simulation.

The old ADR's underlying concern — don't build navigation machinery beyond what the shell needs — still stands; it is now satisfied by the router rather than defended against it.

## References

- `src/App.tsx` — the route table and kept-alive graph iframe
- ADR 0003 — notes navigation and `fromProjectId` location state
- ADR 0012 — graph iframe postMessage protocol
