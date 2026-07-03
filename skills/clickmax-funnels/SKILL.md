---
name: clickmax-funnels
description: Use when the user wants to create, inspect, change, publish, deactivate, delete, or analyze a Clickmax funnel graph.
---

## When this applies

Use this skill when the user wants to operate a Clickmax funnel: inventory it, create/edit graph nodes, connect triggers/pages, validate, publish, deactivate, delete, or inspect analytics.

Not this skill:

- External script install only -> `external-page-setup`
- Lead/payment cohort analysis inside a funnel -> solve that with the relevant sales/CRM skill first
- Fine-grained page DESIGN (moving blocks, copywriting) -> done in the page editor UI. But you CAN create a page from a Clickmax template — including a page WITH a checkout bound to an offer — and link it to a node here; see [pages and checkout](references/pages-and-checkout.md).

## Build completion rule (mandatory)

- Creating nodes is only half the job. A funnel build is incomplete until its nodes are routed.
- After creating page nodes you must connect them with `funnels_triggers_connect` and, for non-page node families, with the type-specific tool (`funnels_abtest_variants_update`, `funnels_conditional_branches_update`, `funnels_traffic_source_update`).
- Never end the turn with page nodes whose triggers still have no `target`: unrouted triggers show up as loose, disconnected nodes with no edges in the editor.
- Multi-node builds should do create + connect as one continuous workflow, not in stop-and-go calls that might stop before the connect step.
- Before reporting done: call `funnels_structure_get`, confirm every intended route has an edge, then call `funnels_validate`; if there are leftover `disconnectedTriggers` or `orphanNodeIds`, surface them instead of silently finishing.

## Key assumptions

- Scope = one workspace + one project; never ask for workspace id
- Funnel lifecycle = `draft` -> `published` -> `unpublished_changes` after graph edits -> republish, or `disabled` after deactivate
- Graph = nodes + embedded triggers/variants/branches/outputs depending on node type
- Trigger, branch, and variant ids are server-generated; always read them from create/structure output
- Manual node creation is not a stopping point by itself; after `funnels_node_create`, always finish the wiring step
- Funnel workflow nodes own entry/exit routing for their linked flow; do not also configure standalone flow triggers for that embedded automation
- `funnels_validate.valid` is not enough by itself; still inspect disconnected triggers, orphan nodes, and missing page links
- Delete tools are destructive and should be confirmed unless deletion was already explicit
- Connections define the visual flow: the builder arranges the funnel left-to-right by following routed edges. Disconnected nodes have no graph flow, so they tend to stack in the first column.

- Read [lifecycle and safety](references/lifecycle-and-safety.md) when deciding between draft edits, publish, deactivate, or destructive delete.
- Read [node types and edges](references/node-types-and-edges.md) when choosing node types and the correct connection tool for each edge family.
- Read [templates and starters](references/templates-and-starters.md) when the user wants a standard funnel shape that maps to a sequence template.

## Thought process

1. Classify the request: read/list/analytics vs create/build/connect vs publish/deactivate/delete.
2. Resolve `projectId` and `funnelId` first.
3. Prefer template scaffolds for common funnel families; prefer manual graph creation only when the user describes a custom route.
4. Use `funnels_structure_get` as the canonical graph view before connecting, publishing, deleting, or diagnosing.
5. Keep graph wiring separate from page authoring: page nodes can exist before page ids are connected.
6. Always finish the wiring after node creation. A build that created nodes but skipped connecting them is not done.
7. Publish only after validation is clean enough and the user explicitly wants the funnel live.

## Execute guide

Use the tools in dependency order when later ids come from earlier results.

- Resolve the target project with `mcp__clickmax__projects_filters` or list funnels with `mcp__clickmax__funnels_list` when the project or funnel is not yet known.
- Read one funnel with `mcp__clickmax__funnels_get`, passing the `funnelId`.
- Use `mcp__clickmax__funnels_structure_get`, passing the `funnelId`, as the canonical graph view before wiring, publishing, deleting, or diagnosing.
- For a standard starter, create the funnel with `mcp__clickmax__funnels_create`, then scaffold the graph with `mcp__clickmax__funnels_sequence_create`.
- For a custom graph, create nodes with `mcp__clickmax__funnels_node_create`, then read back the generated trigger, branch, variant, or output ids before routing them.
- For page nodes or other trigger-based draft nodes, connect outgoing edges with `mcp__clickmax__funnels_triggers_connect`.
- For A/B test nodes, route each variant with `mcp__clickmax__funnels_abtest_variants_update`.
- For traffic source nodes, route the output with `mcp__clickmax__funnels_traffic_source_update`.
- For conditional nodes, route each branch with `mcp__clickmax__funnels_conditional_branches_update`.
- When real page ids already exist, attach them to the correct page nodes with `mcp__clickmax__funnels_node_connect_page` before publishing.
- Validate with `mcp__clickmax__funnels_validate`, passing the `funnelId`, then re-read structure if you need to confirm every intended edge is present.
- Publish only when the user wants the funnel live: use `mcp__clickmax__funnels_publish` with the `funnelId` and the node ids that should go live.

Common flows:

- Template funnel = use `mcp__clickmax__funnels_create`, then `mcp__clickmax__funnels_sequence_create`, then `mcp__clickmax__funnels_validate`, then publish only if the user asked for it.
- Custom funnel = use `mcp__clickmax__funnels_node_create`, then the correct connection tool for that node family, then `mcp__clickmax__funnels_structure_get`, then `mcp__clickmax__funnels_validate`.
- Go live with existing pages = use `mcp__clickmax__funnels_node_connect_page`, then `mcp__clickmax__funnels_validate`, then `mcp__clickmax__funnels_publish`.
- Page WITH a checkout = `mcp__clickmax__pages_templates_list` (`type: ["checkout"]`, pick `canUse: true`), then `mcp__clickmax__pages_create` (`type: "checkout"` + `templateId` + `offerId` + `funnelId`) so the offer is auto-bound to the checkout, then `mcp__clickmax__funnels_node_connect_page` to link it. See [pages and checkout](references/pages-and-checkout.md).

- Read [templates and starters](references/templates-and-starters.md) to choose between `sales`, `lead-magnet`, `webinar`, `tripwire`, `vsl-auto`, and `upsell-downsell`.
- Read [node types and edges](references/node-types-and-edges.md) when deciding which connection tool owns each edge.
- Read [lifecycle and safety](references/lifecycle-and-safety.md#publish-checklist) before publishing, deactivating, or deleting.

## Report

- For reads: `name | id | projectId | status | publishedAt | node count | key problems`
- For create/build: return funnel id, created node labels/types, validation summary, and next action
- For publish: report whether the funnel went live, how many nodes were published, and any remaining manual element wiring
- For deactivate: explain that the funnel is offline, not deleted
- For analytics: summarize the main period metrics instead of dumping raw payloads
- Cap long node/trigger lists and prefer graph summaries

## Warnings

- Resolve real `projectId`, `funnelId`, `nodeId`, `triggerId`, and `pageId`; never invent them
- Publish from fresh structure data, not stale ids captured before later edits
- `valid=true` can still hide non-blocking but important issues like missing pages or disconnected triggers
- API-created graphs may need visual rearrangement in the builder; do not promise tidy coordinates
- Use the correct edge tool for the node family (`funnels_triggers_connect`, `funnels_abtest_variants_update`, `funnels_traffic_source_update`, `funnels_conditional_branches_update`)
- For a `workflow` node, link its flow (`funnels_workflow_flow_set`, resolving `flowId` via `flows_list`/`flows_create`) and set its exit event (`funnels_workflow_exit_trigger_set`); a workflow without a linked flow does not fire automation and is flagged by `funnels_validate.workflowsMissingFlow`
- For a `workflow` node, configure entry/exit from the funnel side; flow-level start events are for standalone automations, not funnel-embedded flows
- Creating nodes without connecting them leaves a broken-looking graph: no routed edges and stacked nodes

## Anti-patterns

- Asking for workspace id
- Using raw `funnels_get` as the only planning source when `funnels_structure_get` gives the compact graph view
- Connecting nodes by slug/label instead of real ids
- Creating page nodes and stopping before `funnels_triggers_connect`, leaving loose unrouted nodes in the editor
- Splitting create and connect across separate incomplete steps so the build stops before routing is finished
- Deleting a funnel when the user only wants it offline or paused
- Creating a BLANK page when the user wanted a checkout/sales page — start from a template (`pages_templates_list`) and pass `offerId` so the checkout is bound, instead of leaving an empty page
