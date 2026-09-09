# Typed structs are the standard for every serialized shape

Every wire shape — route envelope, request body, protocol envelope, and
log record — is a named `Serialize`/`Deserialize` struct, never an inline
`json!` literal. Set 2026-09-08/09 while closing the wire-typing TODOs:
the last inline envelopes (share_sync, the headless admin surface, the
linxiv-api/1 envelope) and even the "internal-only" log records
(`TransferEntry`, `RelayLogEntry`) were promoted. New code does not get
an untyped carve-out.

## Considered Options

- Inline `json!` for internal or one-off shapes: fewer lines at the call
  site, but shapes drift silently between producer and consumer, TS twins
  are hand-maintained, and "internal" shapes leak into wire surfaces
  (the admin log entries are served by a route). Rejected.
- Typed structs everywhere (chosen): one canonical serializer per shape,
  `#[derive(TS)]` where a TypeScript consumer exists, key order pinned by
  field order under `preserve_order` (byte-identical to the envelopes
  they replaced), `#[serde(default)]` where loads must tolerate partial
  legacy data.

## Consequences

- Adding an endpoint or log field is a struct edit plus `types:gen`; the
  frontend twin can never drift.
- Standing exceptions are only the ones TODO.md documents with a contract
  reason: share route bodies (bespoke per-field 422 parity), the
  free-form settings patch (`{updates: Map}` carries arbitrary JSON), and
  `SharedPaper::to_summary_value` (the documented one-home projection).
  Anything else untyped is a defect, not a style choice.
- Log records serialize all fields; tolerant loads mean old JSONL lines
  keep parsing without a migration.
