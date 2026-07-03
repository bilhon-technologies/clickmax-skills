---
name: clickmax-pipelines
description: Use when the user wants to operate CRM pipelines, stages, opportunity cards, attendants, or pipeline analytics in Clickmax.
---

## When this applies

Use this skill for pipeline/stage/card operations: create or change pipelines and stages, move cards, assign attendants, inspect history/risk, import leads into pipelines, or read pipeline analytics/settings.

Not this skill:

- lead search and identity -> `clickmax-leads`
- manual tagging or segment/list membership -> `clickmax-tags` / `clickmax-list-segments`

## Key assumptions

- stage/pipeline ids must be resolved before mutation
- moving cards can change card status from the destination stage semantics
- `won` and `lost` are terminal stage meanings; only put them on stages meant to close opportunities
- auto-assignment strategy changes who owns new/changed cards and should be treated as operational policy, not cosmetic text
- settings can include idle SLA threshold, lost reasons, default priority, temperature, currency, and card display fields
- some move paths may be blocked by stage passage rules
- delete semantics are soft-hide/remove-from-board at the backend level, but still user-visible destructive actions
- attendant assignment/settings writes replace current structures, not patch arbitrary fragments

## Thought process

1. Resolve the exact pipeline/stage/card first.
2. Distinguish card mutation from pipeline metadata/settings mutation.
3. When moving or reordering, preserve board semantics and current anchors.
4. Confirm destructive changes when intent is not already explicit.

## Execute guide

- Resolve the board before writes: use `mcp__clickmax__pipelines_list` to find the pipeline, `mcp__clickmax__stages_list` with `pipelineId` to map stage ids, and `mcp__clickmax__cards_list` with `pipelineId`, `stageId`, pagination, and optional filters such as `search` or `tagIds` when the target card still needs disambiguation.

- Create cards with `mcp__clickmax__cards_create`, passing the target `leadIds`, `pipelineId`, and `stageId`; include `value` and `attendantIds` only when the user wants those set at creation time.

- Move cards with `mcp__clickmax__cards_move`, passing the card `id`, the destination `stageId`, and the current relative anchor (`beforeCardId` or `afterCardId`) so the board order stays intentional; include `lossReason` only when the move path requires or justifies it.

- Assign attendants in 2 steps: first read `mcp__clickmax__pipelines_attendant_types_get` for the pipeline's valid attendant type ids, then write `mcp__clickmax__cards_assign_attendants` with explicit `assignments` entries containing `attendantId`, `attendantTypeId`, and `isPrimary`.

- Import leads into a pipeline with `mcp__clickmax__cards_import_from_lists` only when the user wants board population from list/segment cohorts rather than one-off card creation.

- Inspect card behavior with `mcp__clickmax__cards_get` for one card, `mcp__clickmax__cards_list_by_lead` for a lead's pipeline presence, `mcp__clickmax__cards_history` for move/change history, and `mcp__clickmax__cards_at_risk` for cards currently flagged as operational risk.

- Update structure only after resolving the exact target object: use `mcp__clickmax__stages_create`, `mcp__clickmax__stages_update`, `mcp__clickmax__stages_delete`, or `mcp__clickmax__stages_reorder` for stage changes; use `mcp__clickmax__pipelines_create`, `mcp__clickmax__pipelines_update`, or `mcp__clickmax__pipelines_delete` for pipeline changes.

- Treat settings and analytics as separate from board CRUD: read current configuration with `mcp__clickmax__pipelines_settings_get` before `mcp__clickmax__pipelines_settings_update`, use `mcp__clickmax__pipelines_attendant_types_set` only when changing the pipeline's attendant-role structure itself, and use `mcp__clickmax__pipelines_analytics` for performance/throughput answers rather than card-by-card inspection.

## Report

- For structural changes: report what changed, where, and the resulting status.
- For card moves: report origin -> destination and any relevant status/owner effect.
- For analytics/risk: summarize the operational takeaway, not raw board payloads.

## Warnings

- Do not guess stage ids from names when several stages are similar.
- Card order uses relative positioning semantics; stale anchors can misplace cards.
- Importing from lists/segments skips leads already present in the pipeline.

## Anti-patterns

- Treating pipeline settings as harmless cosmetic edits.
- Moving cards without checking destination stage semantics.
- Using card deletion when the user only wants to hide/archive operationally.
