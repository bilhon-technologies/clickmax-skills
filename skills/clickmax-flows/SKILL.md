---
name: clickmax-flows
description: Use when the user wants to create, inspect, change, validate, or activate/archive a Clickmax automation flow and its step graph.
---

## When this applies

Use this skill when the user wants to operate a Clickmax automation flow: list/find one, inspect its graph, create/edit/connect/delete steps, configure entry events, validate, or change lifecycle mode.

Not this skill:

- Funnel page graph, offers, pages, traffic routing -> `clickmax-funnels`
- Building the cohort that will feed the flow -> resolve that with the relevant CRM/sales skill first, then come back to wire the automation
- Channel copy authoring in isolation -> create the message content first, then reference real ids here

## Key assumptions

- Scope = one workspace; never ask for workspace id
- Granular step writes only work while the flow is `draft` / `template`
- The graph is steps connected by each step output `target`; there is no separate edge object
- The entry is a single `trigger` step; a flow has at most one trigger step
- Trigger start/exit events live at flow level (`triggerStart` / `triggerExit`), not inside arbitrary step fields
- Standalone flows use flow-level trigger events; funnel-embedded flows (`funnelId` set) are started/exited by funnel workflow nodes instead
- Step ids are server-generated; always read real ids from create output or `flows_structure_get`
- When the user is inside the flow builder, the currently-open flow (its `flowId` + every step with full content + the edges between them) is published to screen context under `flows3.builder`; read it before asking which flow/step the user means, and target edits at that `flowId`. If `selectedStepId` is set, that is the node the user has focused on the canvas — prefer it when they say "this step/node" or open the assistant from a node without naming one. The `labels` map resolves the entity UUIDs inside step inputs (tags, lists, products) to human names — read names from there so you never echo a raw UUID back to the user
- Destructive deletes require explicit confirmation unless the user already made deletion explicit

- Read [lifecycle and safety](references/lifecycle-and-safety.md) when deciding between draft edits, activation, closure, archive, or destructive delete.
- Read [step types](references/step-types.md) when choosing which step type/action/input shape fits the requested automation.
- Read [trigger events](references/trigger-events.md) when mapping user intent to flow entry events + constraints.
- Read [examples](references/examples.md) when you need a concrete build/branch/inspect pattern.

## Thought process

1. Classify the request: read/list/inspect/validate vs create/edit/connect vs lifecycle/destructive.
2. Resolve `flowId` first. If `flows3.builder` screen context is present, use its `flowId` (the user is editing that flow); otherwise find an existing flow by name, or create one in `draft`.
3. Check whether the flow is editable before planning step writes.
4. Build trigger-first: create or inspect the entry trigger, then downstream steps, then connections, then validate.
5. Prefer `flows_structure_get` as the canonical compact graph view before connecting, deleting, or diagnosing.
6. Activate only after validation passes and the user explicitly wants the flow running on real contacts.

## Execute guide

- For a new flow, use `mcp__clickmax__flows_create` first, then add the single `trigger` step with `mcp__clickmax__flows_step_create`, then create downstream steps, connect them with `mcp__clickmax__flows_step_connect`, and finish with `mcp__clickmax__flows_validate`.
- For an existing flow, check editability with `mcp__clickmax__flows_get_mode`, inspect the current graph with `mcp__clickmax__flows_structure_get`, apply only the needed `mcp__clickmax__flows_step_*` mutations, and validate again before any lifecycle change.
- Use `mcp__clickmax__flows_update` only for flow metadata such as name/category; it does not edit the step graph.
- Read the valid entry/exit events from `mcp__clickmax__flows_triggers_catalog` before suggesting or setting a trigger — it returns each event's friendly `label`, `description`, and the `scopes` it can be narrowed by; pick the exact `eventName` and never invent one. See [trigger events](references/trigger-events.md).
- Use `mcp__clickmax__flows_step_triggers_set` only when changing entry events, and send the complete `triggerStart` / `triggerExit` arrays that should remain on the flow.
- For a flow linked to a funnel workflow node, keep flow-level trigger arrays empty unless the user is intentionally converting it into a standalone automation; wire entry/exit from the funnel skill instead.
- Use `mcp__clickmax__flows_list` to find candidate flows by name before asking for confirmation on ambiguous matches.
- Use `mcp__clickmax__flows_structure_get` as the canonical graph view before connecting, deleting, or diagnosing steps.
- Use `mcp__clickmax__flows_validate` before activation and surface `hasEntryTrigger`, `danglingTargets`, and `orphanStepIds`, not just `valid`.
- For branching, connect each branch explicitly with the correct `handle`, such as `true` / `false` for conditionals.

- Minimal build pattern: [create -> trigger -> action](references/examples.md#create---trigger---action)
- Linear automation pattern: [trigger -> delay -> send_message](references/examples.md#trigger---delay---send_message)
- Branch pattern: [conditional -> true/false branch](references/examples.md#conditional---truefalse-branch)
- Read-only diagnostics: [inspect -> validate](references/examples.md#inspect---validate)

## Report

- Write every user-facing reply in plain business language: use entity **names** (never UUIDs), never show code, tool names, or internal field names (`triggerStart`, `triggerExit`, `eventName`, `tagId`). Those are for your own reasoning, not the reply — e.g. say "this flow now starts when the **VIP** tag is applied", not the event name or id.
- For list/find: compact candidate table (`name`, `mode`, `category`), not full raw graph dumps
- For create/build/edit: confirm what changed in user terms (which trigger, which steps, what happens next) — not step ids or edge internals
- For validate: always surface `hasEntryTrigger`, `danglingTargets`, and `orphanStepIds`, even when `valid=true`
- For lifecycle: explain the new mode in user terms (`active` = processing real contacts; `closed`/`archived` = stopped)
- Cap long step/edge lists; summarize rather than dumping giant payloads

## Warnings

- `flows_create` needs a real `projectId`; resolve it, never invent it
- `flows_update` is metadata-only and does not edit the graph
- `flows_delete` removes the flow and all steps permanently
- `delay` / `timeout` numeric `when` values are hours, not minutes or days
- `flows_step_triggers_set` edits the entry/exit events independently: send only the side you are changing (`triggerStart` or `triggerExit`) and omit the other to keep it; an explicit `[]` clears a side, and the array you do send replaces that side. Use the exact `eventName` from the [trigger events](references/trigger-events.md) catalog — an unknown/guessed name silently never fires. Scope with the entity id the catalog lists for that event (e.g. `tagId`, `offerId`)
- Conditional, `collect`, and `timeout` branches depend on correct `handle` wiring; missing branch targets usually show up as dangling targets or orphaned paths
- Unknown `action` names or wrong `input` shapes are rejected; fix the payload, do not assume partial success
- Message personalization uses single-brace lead tokens — `{name}`, `{email}`, `{telephone}` — never `{{name}}` or `{{lead.name}}`; an unknown/misformatted key is delivered to the lead literally. Use them as fact, do not ask the user which format applies (GupShup/WhatsApp templates are the only exception: positional `{{...}}` `paramMapping`). See [step types](references/step-types.md).
- Never echo a raw UUID to the user. Step inputs store tags/lists/products by id; resolve them to names via the `flows3.builder` `labels` map (or the matching `clickmax-tags`/list/product lookup tool when an id is absent from `labels`). A UUID in your reply is a bug — report "the tag **Black Friday**", not its id.

## Anti-patterns

- Asking the user for workspace id or a hidden platform id
- Guessing `flowId` or step ids instead of resolving them
- Editing steps on non-editable modes instead of stopping and explaining the constraint
- Treating `valid=true` as publish-ready while ignoring dangling targets or orphan steps
- Activating a flow without explicit user intent to start real processing
