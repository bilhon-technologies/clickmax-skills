# Flow graph patterns

Every pattern = ONE `mcp__plugin_clickmax_clickmax__flows_graph_apply` call; `ops` run in order, `$name` refs resolve inside the same call, real ids come from `flows_structure_get` / `flows3.builder`. Done only when the result has `orphanStepIds = []` and `triggerWithoutOutput = false`; otherwise send a follow-up call that fixes exactly what it reports.

## Create, trigger, action

- Resolve project/tag/list ids first; never invent hidden ids.
- `mcp__plugin_clickmax_clickmax__flows_create` for the shell, then one call with `ops`:
  1. `{op:'add', ref:'$t', type:'trigger', input:{type:'event'}}`
  2. `{op:'setTriggers', triggerStart:[{type:'start', eventName:<from flows_triggers_catalog>, constraints:[…]}]}`
  3. `{op:'add', ref:'$tag', type:'action', action:'addTag', input:{tagId}, after:{step:'$t'}}`
- `mcp__plugin_clickmax_clickmax__flows_validate` before reporting the flow is ready.

## Trigger, delay, message

- Same call shape: `add $t trigger` → `setTriggers` → `add $wait delay {when: 24}` (HOURS) `after:{step:'$t'}` → `add $msg send_message {…raw input}` `after:{step:'$wait'}`.
- Each `after` chains onto the previous step's main output — no separate `connect`.

## Conditional branch

- `add $k conditional {mode, statements}` `after:{step:<prev>}` (statements from `flows_conditionals_catalog`).
- `add $yes …` `after:{step:'$k', handle:'true'}` and `add $no …` `after:{step:'$k', handle:'false'}` in the same call.
- Leaving `false` unwired is a choice, not a default — the lead simply ends there; say so to the user.

## Insert in the middle

- Graph `A → B`, user wants `X` between them: `{op:'add', ref:'$x', …, after:{step:A, handle:<A's output>}}`.
- Result `A → X → B`: B becomes X's main output automatically. Never `add` without `after` and `connect` later — the step is loose on the canvas in between.

## Swap two steps

- Graph `T → A → B → C`, want `T → B → A → C`: `connect T→B`, `connect B→A`, `connect A→C` in one call. `connect` replaces that output's current destination, so no `disconnect` is needed.
- Remove instead of swap: `{op:'remove', step:A}` — `bridge` (default true) reconnects A's predecessor to A's successor; pass `bridge:false` only when the user wants that path to end.

## Message that waits for a reply

- `{op:'add', ref:'$ask', type:'send_message', input:{…WhatsApp/Telegram/Instagram}, capture:{timeoutMinutes:60}, after:{step:<prev>}}` — `timeoutMinutes` is MINUTES (unlike `delay.when` hours); omit it for no timeout.
- Wire its outputs by handle: main (`handle` omitted/null) = valid reply → `connect {from:'$ask', to:<next>}`; `invalid` = reply didn't match; `timeout` = no reply in time.
- Never target the hidden helper steps the capture creates; always address the message and its handle.

## Inspect and validate

- `mcp__plugin_clickmax_clickmax__flows_structure_get` → each visible step with `outputs[]` (`handle`, `label`, `target`; `target:null` = free output you can wire).
- `mcp__plugin_clickmax_clickmax__flows_validate` → surface `hasEntryTrigger`, `danglingTargets`, `orphanStepIds`, `triggerWithoutOutput`, `incompleteChannelSteps` even when `valid=true`.
