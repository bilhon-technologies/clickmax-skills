# Flow lifecycle and safety

## Editable vs running modes

|Mode|Graph edits|
|-|-|
|`draft` / `template`|yes|
|`closed` (paused)|yes — stays paused after the edit; executions waiting on a removed step are canceled, same as editing in the canvas|
|`active`|no — ask the user, pause with `flows_close`, edit, then `flows_activate` only if they want it live again|
|`scheduled` / `archived`|no — never edit; explain the constraint|

- A rejected write names the mode; explain it instead of retrying blindly

## Lifecycle tools

- `flows_get_mode` = cheap status check before planning edits or activation
- `flows_activate` = starts processing real contacts; validate first and ask before calling
- `flows_close` = pause a running flow without deleting it (also the way to make an `active` flow editable)
- `flows_archive` = retire the flow from active use
- `flows_delete` = permanent destructive delete of the flow and every step
- Per-contact runs (not lifecycle): `flows_test_run_start`, `flows_execution_retry`, `flows_executions_retry_by_error` = real side effects on real contacts (need consent, state the count); `flows_execution_cancel` = permanent for that run. See [executions and testing](executions-and-testing.md)

## Activation checklist

Before `flows_activate`:

1. confirm the user wants the automation live on real contacts
2. ensure the entry trigger exists and has an output (`triggerWithoutOutput = false`)
3. ensure triggerStart / triggerExit are configured
4. run `flows_validate`
5. report any remaining structural problems (`orphanStepIds`, `danglingTargets`), including `incompleteChannelSteps` — message steps (email/telegram/WhatsApp) missing a real sender id. Resolve it with `email_sender_signatures_list` / `channel_instances_list` and `flows_step_update` before activating, rather than retrying `flows_activate` unchanged: a channel step without a real sender id fails every send no matter what activation itself currently checks

## Deletion checklist

Before `flows_delete` or a `remove` op / `flows_step_delete`:

1. make sure the user explicitly wants deletion, not pause/stop/archive
2. explain that delete is irreversible
3. removing a step clears inbound targets to it; the `remove` op reconnects predecessor → successor by default (`bridge`), `flows_step_delete` does not
