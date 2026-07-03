---
name: clickmax-packs
description: Use when the user wants to inspect, create, edit, snapshot, publish (share), or import Clickmax packs — shareable bundles of funnels, pages, flows, and affiliated offers — including funnels4 snapshot/drift/gate handling and import remap resolution.
---

## When this applies

Use this skill for the pack lifecycle: a pack is a shareable bundle of funnels, pages, flows, and affiliated offers that a creator publishes for other workspaces to import. A pack belongs to one workspace and project. `funnelsEngine` is `funnels3` (legacy) or `funnels4` (snapshot-based sharing).

Not this skill:

- offer-level pricing/checkout control → use the offers tools
- standalone funnel/page/flow editing outside a pack → use the funnels/pages/flows tools

## Entity model

|field|meaning|
|-|-|
|`id`|Pack UUID|
|`name`|Display name|
|`description`|Author-facing summary|
|`category` / `level`|Catalog classification|
|`isActive`|Whether the pack is listed/importable|
|`funnelsEngine`|`funnels3` or `funnels4`; snapshot/drift apply only to `funnels4`|
|`funnels` / `pages` / `flows`|Contents referenced by the pack|
|`offers`|Affiliated offers with `commission` and affiliation active state|
|`shareCount`|How many times the pack has been imported|
|`publishedAt`|When the pack became publicly shareable (null = unpublished)|

## Snapshot lifecycle (funnels4 only)

- editing a pack's contents changes its source; `drift` means the source moved past the latest snapshot
- a snapshot is an immutable, versioned copy of the contents taken for sharing
- a snapshot version carries a gate `status`: `draft` (not validated) | `validating` (gate running) | `valid` (clean) | `warnings` (publishable, with caveats) | `blocked` (errors, not shareable)
- only `valid` or `warnings` snapshots become the importable (shared) version
- the gate runs asynchronously; reach the status by polling (`packs_snapshot_get`), not synchronously

## Import resolution (funnels4 snapshot import)

- the import preview (`packs_import_preview`) lists what the importer must (or may) remap, because some references point to the author's workspace and cannot be cloned
- blocking: provider channels (webchat) and non-shared offers — the importer must supply their own before import; otherwise import is rejected
- optional: pages may embed lists/forms (the preview surfaces them per page); the importer can map each embedded list/form to one of their own, or leave it unmapped to keep the original reference (does not block import)
- external pages cannot be cloned — the importer supplies a URL per external page

## Rules

- list + get only return non-deleted packs
- changing contents (`packs_append`/`packs_remove`) requires a new snapshot (`packs_resnapshot`) before the shared version reflects the change
- an inactive offer affiliation blocks sharing — the gate reports it as an error
- deleting a pack stops it being importable but does not delete the underlying funnels/pages/flows
- importing is asynchronous and may finish `success`, `partial` (some offers unresolved), or `failed`
- any own resource the importer supplies for a remap (channel, offer, list, form) must belong to the importer

## Consumer tips

- check drift (`packs_snapshot_drift`) before sharing an edited pack; re-snapshot when the source changed
- resolve the blocking preview items (provider channels, own offers for non-shared ones, external page URLs) before importing to avoid rejection or a `partial` result
- page list/form remap is optional — map them to keep imported pages wired to the importer's own audience, or skip to leave the original references
- a published pack with a `warnings` snapshot is still importable; read the warnings to set expectations
- track an in-flight import by its returned id (`packs_imported_get`) rather than assuming it completed
