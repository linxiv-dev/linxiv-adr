# Short tickets accept an offline-LAN join regression (left open)

Share tickets used to embed the peer's full address set (every direct
IP/LAN/Docker/VPN address), making them long and leaky. linxiv-p2p
`feat/short-tickets` made tickets discovery-bound: they carry only the
relay entry and the peer's addresses are resolved through discovery at
connect time; a full-address ticket is still minted as a fallback when no
relay is known. We accept the resulting regression: on an offline LAN
(same network, no internet), a ticket minted while a relay was configured
names only an unreachable relay and the join fails, where the old fat
ticket carried the LAN address and worked.

## Considered Options

- Conditionally embed local addresses alongside the relay entry: restores
  offline-LAN joins but re-fattens tickets and re-leaks the address set —
  the problem short tickets exist to solve. Rejected for now.
- Accept and document (chosen): offline-LAN share joining was never a
  stated feature, and Remote Query Mode requires a relay by design.

## Consequences

- Deliberately left open, not foreclosed: iroh is widely used inside
  private networks, so no future change may make offline-LAN operation
  impossible. The intended recovery path is local peer discovery
  (mDNS-style, supported by iroh) resolving the address that the ticket no
  longer carries — additive, without re-fattening tickets.
- Known inconsistency: beelay `ProjectInvite` still mints full addresses,
  so invites leak what tickets no longer do. Align when invites are next
  touched.
