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
|`isActive`|Whether the pack is listed/importable; funnels4 activation needs a published snapshot (see lifecycle)|
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
- order for funnels4: new version (the `snapshot` returned by `packs_append`, or `packs_resnapshot` when nothing was appended) → `packs_snapshot_publish` → poll `packs_snapshot_get` until `valid`/`warnings` (promoted to the published version) → THEN `packs_update` with `isActive: true`; activating before a published version returns 409
- imports read ONLY the published version (never a newer draft); no published version → import returns 409 "Este pacote ainda não foi publicado pelo produtor"
- the gate runs asynchronously; reach the status by polling (`packs_snapshot_get`), not synchronously

## Import resolution (funnels4 snapshot import)

- the import preview (`packs_import_preview`) lists what the importer must (or may) remap, because some references point to the author's workspace and cannot be cloned; for funnels4 it reads only the published snapshot (no published version → error)
- blocking: provider channels (webchat) and non-shared offers — the importer must supply their own before import; otherwise import is rejected
- optional: pages may embed lists/forms (the preview surfaces them per page); the importer can map each embedded list/form to one of their own, or leave it unmapped to keep the original reference (does not block import)
- external pages cannot be cloned — the importer supplies a URL per external page

## Execute guide (exact params — avoid live tool-discovery)

- `mcp__plugin_clickmax_clickmax__packs_list` REQUIRES `projectId` (no default/workspace-wide mode) — resolve the project first (`clickmax-projects`) before listing.
- `mcp__plugin_clickmax_clickmax__packs_snapshot_get` and `mcp__plugin_clickmax_clickmax__packs_snapshot_publish` both take `{packId, version}` — `version` is NOT optional/latest-implied. Source it from `snapshot.version` of the last `mcp__plugin_clickmax_clickmax__packs_append`, the `{id, version}` of the last `mcp__plugin_clickmax_clickmax__packs_resnapshot`, or from `packs_snapshot_drift`'s `latestVersion`. Never guess a version number.
- `mcp__plugin_clickmax_clickmax__packs_create` takes `projectId` + `name` (required); `description`/`category`/`level`/`thumbnailUrl`/`funnelsEngine` optional (`funnelsEngine` omitted = `funnels4`; `funnels3` only for funnels3 funnels — `packs_append` rejects other-engine funnels with 409). Add contents afterward with `packs_append` — creation itself is always empty.
- `mcp__plugin_clickmax_clickmax__packs_append` `funnelModes` (`full` | `skeleton` = structure only, no pages/automations/offers) is stored per funnel and reused by every later version; re-send a funnel already in the pack with the other mode to switch it, omit it to keep its mode.
- `mcp__plugin_clickmax_clickmax__packs_import`'s dependency resolution is FIVE separate maps, not one — pass the one matching what the preview flagged, not the generic-sounding one:
  - `valueMap` — automation dependencies (preview `dependencies[]`): `{[kind]: {[originId]: ownId | null}}`; one choice per author resource, applied to every step + trigger using it. Missing or `null` → the copy gets the field empty (never the author's id).
  - `resolutions` — per automation (keyed by preview automation `id`): `{type: 'automation', skip: true}` = don't create it.
  - `offerResolutions` — non-shared offer remap (keyed by the pack's offer reference, `{offerId}` = the importer's own offer).
  - `channelResolutions` — blocking provider channel remap (keyed by the pack's channel reference, `{channelId}` = the importer's own channel).
  - `pageResolutions` — per-page list/form remap (keyed by page, `{listIdMap, formIdMap}`).
  - Plus `externalPageUrls` (map of external-page ref → URL) and `overrideFlowsIds` (`[{originFlowId, importedFlowId}]`) for flow-id overrides.
  - Stuffing a channel/offer remap into `resolutions` (the automation-only map) silently does nothing for that dependency — it stays unresolved and the import comes back `partial`.

## Rules

- list + get only return non-deleted packs
- `packs_append` (funnels4) already generates the new draft version and returns it as `snapshot` → publish THAT version; don't `packs_resnapshot` again. `packs_remove` requires `packs_resnapshot` before the shared version reflects the change
- a funnel deleted at the source leaves the pack automatically (on delete and on the next version)
- an inactive offer affiliation blocks sharing — the gate reports it as an error
- funnels4: never activate (`isActive: true`) before the published snapshot reaches `valid`/`warnings`; content changes reach importers only after a new version is published
- deleting a pack stops it being importable but does not delete the underlying funnels/pages/flows
- importing is asynchronous and may finish `success`, `partial` (some offers unresolved), or `failed`
- any own resource the importer supplies for a remap (channel, offer, list, form) must belong to the importer

## Consumer tips

- check drift (`packs_snapshot_drift`) before sharing an edited pack; re-snapshot when the source changed
- resolve the blocking preview items (provider channels, own offers for non-shared ones, external page URLs) before importing to avoid rejection or a `partial` result
- page list/form remap is optional — map them to keep imported pages wired to the importer's own audience, or skip to leave the original references
- a published pack with a `warnings` snapshot is still importable; read the warnings to set expectations
- track an in-flight import by its returned id (`packs_imported_get`) rather than assuming it completed
- importing workspaces stay PINNED to whichever snapshot version they imported — a later `packs_resnapshot`/`packs_snapshot_publish` on the source pack does NOT automatically propagate to already-imported workspaces; they pick it up with `packs_imported_update_get` → `packs_imported_update_apply` (see "Updating an imported pack")

## After import: "Conecte para ativar"

An import only finishes setup when its account slots are resolved; the imported automations stay without an account until then. `importId` = `id` from `packs_imported_list` (not the `packId`).

- `mcp__plugin_clickmax_clickmax__packs_imported_slots_list` → for each `open` slot with `fillable: true`, fill it with `mcp__plugin_clickmax_clickmax__packs_imported_slots_update` using one of its `candidates[].id` (pre-pick `suggestedId`); several slots of the same `entity` → `applyToSameEntity: true` once. No candidate → the importer must connect that account first (route them to the integration); `resolvedId: null` skips the slot instead.
- `fillable: false` slots are configured inside the automation editor — tell the user which automations (`locations`) instead of trying to fill them.
- `gupshupTemplate` slots: the importer needs their own approved WhatsApp templates. With `template.replicable: true` and an official number in `whatsappNumbers`, confirm the list with the user, then `mcp__plugin_clickmax_clickmax__packs_imported_slots_templates_create` (`channelInstanceId` + `slotIds`). Slots go `pending` until Meta approves and fill themselves; report `failed` entries and re-read the list later to see `review.status`/`reason`.

## Updating an imported pack

Imports stay pinned to the version they imported. When the author publishes a new one: `mcp__plugin_clickmax_clickmax__packs_imported_update_get` → if not `upToDate`, show the `items`; for each `ask: true` item ask the user `keep` or `take` (`changed`) / `keep` or `delete` (`removed`), warning when `edited` (their changes are lost on `take`) or `live` → `mcp__plugin_clickmax_clickmax__packs_imported_update_apply` with `decisions` keyed `kind:sourceId`. Never decide for the user. New automations can open new slots → re-run the slots flow above.

## Author: sharing, extra assets and cover

- Before appending funnels/pages, `mcp__plugin_clickmax_clickmax__packs_checkout_offers_preview` (`funnels`/`pages` ids) shows which offers their checkouts sell and whether each is shared; tell the user which products go along.
- `mcp__plugin_clickmax_clickmax__packs_offer_sharing_update` (`packId` + `offerId` + `isShared`) decides if importers sell the author's offer (`true`) or must bring their own (`false`, blocking in their import preview). Gated: changes what every importer sells. Needs the affiliation-packs feature.
- `mcp__plugin_clickmax_clickmax__packs_snapshot_assets_list` (`packId` + `version`) lists the extra assets a version carries (quiz, custom fields…) — use it to answer "what goes with the pack".
- Cover: ask the user to upload the image in the chat (an `image` question), then `mcp__plugin_clickmax_clickmax__packs_cover_set` with `packId` + the uploaded `imageUrl`; `mcp__plugin_clickmax_clickmax__packs_cover_remove` clears it. `packs_create`/`packs_update` do not take a cover URL, and images from other sites are refused.

---

Clickmax skill revision: `2c281119c67a`
