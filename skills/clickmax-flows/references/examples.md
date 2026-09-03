# Flow Connector Operation Examples

## Create, Trigger, Action

- Resolve the project/tag/list ids first; do not invent hidden ids.
- Use `mcp__plugin_clickmax_clickmax__flows_create` for the flow shell.
- Use `mcp__plugin_clickmax_clickmax__flows_step_create` for the trigger and action steps.
- Use `mcp__plugin_clickmax_clickmax__flows_step_connect` to connect each created step.
- Validate with `mcp__plugin_clickmax_clickmax__flows_validate` before reporting the flow is ready.

## Trigger, Delay, Message

- Create or resolve the trigger step with `mcp__plugin_clickmax_clickmax__flows_step_create`.
- Set or replace start-trigger constraints with `mcp__plugin_clickmax_clickmax__flows_step_triggers_set` when the trigger conditions are not already correct.
- Create the delay and message steps with `mcp__plugin_clickmax_clickmax__flows_step_create`.
- Connect trigger → delay → message with `mcp__plugin_clickmax_clickmax__flows_step_connect`.
- Inspect with `mcp__plugin_clickmax_clickmax__flows_structure_get` and validate with `mcp__plugin_clickmax_clickmax__flows_validate`.

## Conditional Branch

- Create the conditional step with `mcp__plugin_clickmax_clickmax__flows_step_create` and explicit statements.
- Create each branch target step before wiring.
- Connect true/false handles with `mcp__plugin_clickmax_clickmax__flows_step_connect`; pass `handle` when the branch matters.
- Inspect `mcp__plugin_clickmax_clickmax__flows_structure_get` after wiring to confirm edge direction.
