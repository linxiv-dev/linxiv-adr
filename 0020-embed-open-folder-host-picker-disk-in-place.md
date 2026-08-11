# 20. Embed "Open Folder": host-native picker, disk-in-place editing

Date: 2026-06-06

## Status

Accepted

## Context

The embedded TeXbrain editor (ADR 0017) runs in a cross-origin iframe
(`texbrain://localhost`). Inside that iframe **neither** filesystem path the
standalone editor uses is available:

- Tauri IPC is not injected into cross-origin iframes (`__TAURI_INTERNALS__`
  is absent), so the editor's own `tauri-fs.ts` adapter cannot run.
- WebKitGTK has no File System Access API (`showOpenFilePicker`), so the
  browser path fails too — clicking "Open Folder" surfaced a misleading
  "This feature requires Chrome or Edge" toast.

Vault projects already work: `texbrain:doc:open` mounts a postMessage FS root
and every `texbrain:fs` op is resolved by the host against the note vault
(`/api/editor/vault/<noteId>/fs`). The gap is opening an **arbitrary disk
folder** from inside the embed.

## Decision

Route the pick and the I/O through the **host**, and edit the picked folder
**in place**:

- New bridge RPC, additive within bridge protocol 1:
  `texbrain:pick:folder` (guest→host) / `texbrain:pick:folder:result`
  (host→guest, `name` = folder basename, `null` = cancelled).
- The host shows its native directory dialog and re-roots a new
  `HostFsRouter`: while a disk root is set, every `texbrain:fs` op resolves
  against that folder via `tauri-plugin-fs` (newly registered); otherwise ops
  go to the vault responder as before. A vault `doc:open` clears the disk
  root.
- The pick itself runs in a host **Rust command** (the texbrain plugin's
  `pick_folder`, invoked as `plugin:texbrain|pick_folder`), not
  the JS dialog plugin: the dialog plugin's automatic fs-scope grant is
  non-recursive (subfolder listings fail with "forbidden path"), while the
  Rust side can `allow_directory(path, recursive=true)` for exactly the
  folder the user picked. The capability file only lists the fs *operation*
  permissions, no static directory grants.
- Guest-side, the picked folder mounts through the existing postMessage FS
  root — same machinery as vault projects, so saves, tree operations, and
  drawio export all work unchanged. Writes land directly in the user's
  folder, matching standalone TeXbrain's "Open Folder" semantics.

## Consequences

- "Open Folder" works in the embed with native-app semantics; edits live
  where the folder lives, no copy, no divergence.
- A disk project is **not** a linXiv editor project: it has no note id, does
  not appear in the project dropdown (shown only as a placeholder label), and
  is not tracked by the vault. Host-side features keyed on notes (e.g.
  project listing, future metadata) don't apply to it.
- **Deferred — import into vault**: the complementary flow ("pick a folder,
  copy/register it as a vault project, open it like any other note-backed
  project") is intentionally not built yet. It needs an import API endpoint
  and a dedup/refresh story, but would give disk content the full project
  treatment (dropdown, persistence guarantees, future sync). Revisit when the
  Notes page lands; the pick RPC added here is reusable for it as-is.
- A host older than this ADR never answers `texbrain:pick:folder`; the
  guest's promise stays pending (documented in the contract). Hosts and
  editors ship together pre-release, so no protocol bump was spent on this.
