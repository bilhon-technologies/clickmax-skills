# Funnel Connector Operation Examples

## Create Scaffolded Funnel

- Resolve the project first; do not invent `projectId`.
- Use `mcp__plugin_clickmax_clickmax__funnels_create` with name, description, `projectId`, and optional `domainId`.
- Use `mcp__plugin_clickmax_clickmax__funnels_sequence_create` with `template: "sales"` when the user wants the standard starter graph.
- Validate with `mcp__plugin_clickmax_clickmax__funnels_validate`, then inspect with `mcp__plugin_clickmax_clickmax__funnels_structure_get` before reporting completion.

## Build Custom Graph

- Use `mcp__plugin_clickmax_clickmax__funnels_node_create` for each page/branch/logic node.
- Read the returned trigger and node ids; never guess them from labels.
- Use the node-family connection operation after each node exists, such as `mcp__plugin_clickmax_clickmax__funnels_triggers_connect`, `mcp__plugin_clickmax_clickmax__funnels_abtest_variants_update`, `mcp__plugin_clickmax_clickmax__funnels_traffic_source_update`, or `mcp__plugin_clickmax_clickmax__funnels_conditional_branches_update`.
- Re-read `mcp__plugin_clickmax_clickmax__funnels_structure_get` and run `mcp__plugin_clickmax_clickmax__funnels_validate` before any publish step.

## Link Pages, Validate, Publish

- Link existing pages with `mcp__plugin_clickmax_clickmax__funnels_node_connect_page` only after resolving both the funnel node and page.
- Validate with `mcp__plugin_clickmax_clickmax__funnels_validate` and inspect `mcp__plugin_clickmax_clickmax__funnels_structure_get`.
- Publish with `mcp__plugin_clickmax_clickmax__funnels_publish` only when the user explicitly wants the funnel live; pass only publishable node ids.
