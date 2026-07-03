---
name: clickmax-tags
description: Use when the user wants to inspect, create, update, delete, clone, or apply CRM tags to leads in Clickmax.
---

## When this applies

Use this skill when the user wants to manage CRM tags or use tags as cohort labels: inspect manual/system tags, create/update/delete/clone manual tags, or apply/remove tags from many leads.

Not this skill:

- raw event timelines -> `clickmax-leads-activity-analysis`
- manual lists or dynamic segments -> `clickmax-list-segments`

## Key assumptions

- system tags are event-derived and cannot be updated or deleted
- manual tag create always yields a non-system tag
- deleting a manual tag also removes assignments and related kanban links
- lead apply/remove tools mutate assignments, not the tag definition itself

## Thought process

1. Determine whether the need is taxonomy management or lead assignment.
2. Distinguish manual vs system tags early.
3. Use batch apply/remove only when the user really wants bulk cohort mutation.

## Execute guide

- Use `mcp__clickmax__tags_list` when the user starts broad and wants the full tag catalog.
- Use `mcp__clickmax__tags_manual_list` for editable CRM labels, optional name filtering, and growth windows such as `growthDays = 14`.
- Use `mcp__clickmax__tags_system_list` for event-derived labels; use `mcp__clickmax__tags_system_activities` when the user needs the activity basis behind one system tag.
- Use `mcp__clickmax__tags_get` before changing a specific tag so manual vs system status is explicit.
- Use `mcp__clickmax__tags_create` for new manual tags. If the user also wants the new tag applied immediately, create first, then use `mcp__clickmax__crm_tags_apply_to_leads` with the returned tag id.
- Use `mcp__clickmax__tags_update`, `mcp__clickmax__tags_clone`, and `mcp__clickmax__tags_delete` only for manual tags.
- Cloned tags are named from the original with a numeric suffix, such as `<original name> (1)`; rename after cloning if the user needs a business-friendly label.
- Use `mcp__clickmax__crm_tags_remove_from_leads` when the tag definition should stay but specific leads should lose the assignment.
- Use `mcp__clickmax__crm_tags_batch_apply_to_leads` only for intentional bulk cohort changes across many leads. When the result needs validation, inspect membership changes with `mcp__clickmax__tags_manual_leads_timeline`.
- Preferred order: inspect tag type -> inspect current definition or membership when relevant -> mutate the tag or assignments -> verify updated state for high-impact changes.

## Report

- Start with the tag job type: taxonomy change, assignment change, or inspection.
- For taxonomy changes: report tag name, tag id, and the exact change.
- For lead assignment changes: report affected lead count first, then skips/errors, then the tags involved.
- For inspections: show manual vs system status first because it determines what can be changed.
- If listing multiple tags, order the most relevant matches first and cap the list with `+N more`.
- Follow-up mutations are opt-in only.

## Warnings

- Do not offer update/delete for system tags.
- Do not treat manual-tag timelines as raw activity streams.
- Batch apply/remove is high-impact cohort mutation.

## Anti-patterns

- Editing system tags.
- Confusing tag assignment with segment/list membership.
- Applying tags to a broad unresolved cohort without confirming intent.
