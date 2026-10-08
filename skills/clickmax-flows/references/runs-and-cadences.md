# Manual runs and cadences

Manual run ("Executar") = one mass entry of a PUBLISHED flow for an audience chosen at that moment: contacts picked one by one (max 500), a list, a tag or a segment. Each contact becomes an ordinary production execution tagged with the run. Tool contracts (fields, errors) live in each tool description; this file = sequencing + traps.

## Which tool for which question

|Question|Tool|
|-|-|
|How many would receive it? Does the balance cover it?|`flows_run_estimate`|
|Send / schedule the flow for list X|`flows_run_start`|
|Did the send go out? How many entered/failed? Is something scheduled?|`flows_runs_list`|
|When does it finish? Which cycle failed?|`flows_run_blocks_list`|
|Who failed in that run?|`flows_executions_list` with `runId`|
|Stop what has not gone out yet|`flows_run_cancel`|
|Send the next cycle now|`flows_run_block_promote`|
|Which message of which automation is this contact on? What is next and when?|`flows_lead_cadences`|

## Start a run

1. Resolve the flow and check it is `active` (`flows_get_mode`). Draft → the user wants a test (`flows_test_run_start`, one contact) or must publish first; ask, never activate on your own just to reach a list.
2. Resolve the source to a real id (list/tag/segment lookup, or contact ids). Prefer a list/tag/segment over hundreds of picked contacts (500 cap).
3. `flows_run_estimate` → tell the user: source name, `audience`, how many sit on the cooldown (`debounceSeconds` window; they are skipped by default), and the cost as an order of magnitude when it matters (`enough = false` → say so before starting).
4. Ask: include contacts on cooldown (`skipDebounce`)? now or a date (`scheduledAt`, future, workspace timezone made explicit in the reply)?
5. `flows_run_start` (needs approval) → report status `running` or `scheduled` and how to follow it.

Pace comes from the flow (`dispatchBatchSize` default 250 per cycle, `dispatchIntervalMinutes` default 0 = all cycles queued at once). To spread a big send over hours, change those on the flow with `flows_update` BEFORE starting; a run in flight keeps the pace it started with.

## Traps

- One live run per flow: a `scheduled` run also blocks a new one (409). Check `flows_runs_list` before starting; cancel only if the user wants to replace it.
- Audience freezes when the run LAUNCHES — at `scheduledAt` for a scheduled run, not when it was scheduled. Segments are re-evaluated at that moment.
- Empty audience still leaves a `failed` run in the history; a scheduled run can turn `failed` (balance, missing sender) or `cancelled` (flow paused/unpublished, number down) at launch time — read `errorMessage`.
- Cost estimate prices only the messages before the first branch/wait/reply; later messages are extra. Never present it as the final bill.
- Cancelling a run ≠ pausing the flow ≠ stopping executions: the run cancel stops pending cycles only; the flow keeps reacting to its trigger; contacts already in keep going (stop one with `flows_execution_cancel`). Pausing (`flows_close`), archiving or deleting the flow also cancels its live run.
- Steps that need the trigger's context (Instagram comment/story reply, WhatsApp through the number that received the conversation) make the flow non-runnable (412): fix the step (e.g. pin the sending number) instead of retrying.
- Promote is only useful when the flow spaces cycles with an interval; with interval 0 every cycle is already queued.

## Cadences (`flows_lead_cadences`)

- For an opportunity, take its contacts' lead ids from `cards_get` and query them together (max 100 contacts, 50 executions in total, newest first; `truncated` → query fewer contacts).
- Answer with: automation name, the message the contact is on (`current`) or last received (`sent` + delivery status), the `next` one and when (`resumesAt`, or the `waits` before it). No `next` while parked on a reply/no-reply branch = it depends on the contact; say both paths.
- Only flows that send messages and only production executions appear; an AI SDR conversation is read from `flows_execution_get` instead.
- To take a contact out of a cadence, cancel that execution (`flows_execution_cancel` with its flow id + execution id), after confirming.

## Report

- Runs: source name + contacts entered / skipped by cooldown / failed, status in plain words (scheduled, sending, finished, cancelled, failed + reason), who started it. Never a run or block id.
- Cadences: per automation, "on message N (WhatsApp: '…'), next is '…' in 2 days" — message previews, not step ids.
