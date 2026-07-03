# Funnel node types and edges

## Main node families

- `page` / `draft`
  - use embedded triggers
  - create/edit triggers with `funnels_node_create` / `funnels_node_triggers_update`
  - connect with `funnels_triggers_connect`

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
  - target only (no outgoing edge); embeds an automation Flow via `flowId`
  - link/swap/unlink the flow with `funnels_workflow_flow_set` (resolve `flowId` with `flows_list` or create one with `flows_create`; `flowId: null` unlinks) — or pass `config.flowId` at `funnels_node_create` time
  - connect a page trigger INTO it with `funnels_triggers_connect`: with a linked flow this auto-creates a real FlowTrigger (mapped event + constraints) recorded in `inputTriggers`; disconnecting/deleting removes it, relinking re-syncs
  - set the exit event (ends the lead's membership in the flow) with `funnels_workflow_exit_trigger_set`
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
