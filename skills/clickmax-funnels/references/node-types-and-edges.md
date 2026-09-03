# Funnel node types and edges

## Main node families

- `page` / `draft`
  - use embedded triggers
  - create/edit triggers with `funnels_node_create` / `funnels_node_triggers_update`
  - connect with `funnels_triggers_connect`
  - EXTERNAL page (hosted outside Clickmax): the node carries `config.externalUrl` (+ `pageId` of an editor3 external page); the deploy 302-redirects to that URL and the user must install the Browser SDK script. Build it by composing `pages_create_external` -> `funnels_node_create` (type `page`) -> `funnels_node_connect_page` (connecting an external page auto-sets `node.config.externalUrl`) -> `pages_get_external_script` (install snippets). Never use `pages_create` for an external URL (makes an empty internal page).

- `ab_test`
  - routes by variants
  - update all variants with `funnels_abtest_variants_update`
  - connecting a variant = filling `connectedTo`

- `traffic_source`
  - one output path
  - update with `funnels_traffic_source_update`
  - output target lives in `connectedTo`

- `conditional`
  - routes by branches
  - update all branches with `funnels_conditional_branches_update`
  - exactly one fallback false branch

- `workflow`
  - a PARALLEL side-effect off a page trigger, NOT a visitor step. Target only, no outgoing edge: it fires its linked Flow, it does NOT forward the visitor to a next page — never give it an outgoing edge or a `flow_completed`/`page_viewed` exit to "continue" the funnel
  - embeds an automation Flow via `flowId`; link/swap/unlink with `funnels_workflow_flow_set` (resolve `flowId` with `flows_list` or create one with `flows_create`; `flowId: null` unlinks) — or pass `config.flowId` at `funnels_node_create` time
  - connect a page trigger INTO it with `funnels_triggers_connect` (target = workflow id): with a linked flow this records the link in the workflow's `inputTriggers` (a real FlowTrigger, mapped event + constraints) and LEAVES the trigger's `connectedTo` (page route) intact — so the SAME trigger sends the visitor to a page AND starts the automation. Shows in `funnels_structure_get` as an edge `via: "workflow_input"`; disconnecting (`target: null`)/deleting removes it, relinking re-syncs
  - the exit event (`funnels_workflow_exit_trigger_set`) ENDS the lead's membership in the flow (a conversion/goal) — it is NOT a page-navigation edge
  - a workflow without `flowId` is flagged by `funnels_validate.workflowsMissingFlow`

- `notes`
  - informational node
  - usually excluded from publish node ids

## Unified edge model

`funnels_structure_get.edges` normalizes all node families into one list.

Each edge includes:

- `via` = `trigger | variant | branch | output`
- `handleId` = the trigger / variant / branch / output handle identifier

## Practical rule

Choose the connection tool by node family, not by habit:

- page trigger -> `funnels_triggers_connect`
- A/B variant -> `funnels_abtest_variants_update`
- traffic source output -> `funnels_traffic_source_update`
- conditional branch -> `funnels_conditional_branches_update`
- workflow flow link -> `funnels_workflow_flow_set` (page trigger INTO a workflow still uses `funnels_triggers_connect`)
