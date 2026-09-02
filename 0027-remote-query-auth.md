# Remote query auth: one member list, node-side enforcement, transport refusal

Remote Query Mode (ALPN `linxiv-api/1`) lets an admitted device query a
headless node's library like its own DB. We decided: (1) a single Member
List file (`{id, role}`, upgraded from the relay allowlist) governs both
relay admission (presence) and query rights (role) — a known coupling,
chosen over two files kept in lockstep; (2) the role check happens at the
node's ALPN accept against the authenticated endpoint id, never inferred
from "the connection came through our relay" — peers can arrive via direct
hole-punch or public relays; (3) non-members are refused at the transport
after logging the knocking endpoint id, so an unadmitted device cannot
distinguish "node offline" from "not admitted."

## Considered Options

- Accept unknown peers and answer 403 "awaiting admission": friendlier
  join UX, but reveals the node's existence to anyone holding its address.
  Rejected — the internet is hostile; the operator's access log still shows
  the knock, so admission works without leaking anything to the knocker.
- Separate files for relay admission and query roles: cleaner semantics,
  rejected as two sources of truth for one trust decision.

## Consequences

- "Query rights without relay admission" is inexpressible — accepted.
- `read-write` is a data-plane role: settings, storage, and share
  administration are operator-only for every role, and external Provider
  fetches require the separate Provider Access capability.
- Client UX must present one honest state for refused connections.
