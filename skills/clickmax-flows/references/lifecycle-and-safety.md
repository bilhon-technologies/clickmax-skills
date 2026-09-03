# Flow lifecycle and safety

## Editable vs running modes

- Step writes only work in `draft` / `template`
- `active`, `scheduled`, `closed`, and `archived` reject granular step writes
- When the user wants to modify a running flow, first explain the mode constraint instead of retrying writes blindly

## Lifecycle tools

- `flows_get_mode` = cheap status check before planning edits or activation
- `flows_activate` = starts processing real contacts; validate first and ask before calling
- `flows_close` = stop a running flow without deleting it
- `flows_archive` = retire the flow from active use
- `flows_delete` = permanent destructive delete of the flow and every step

## Activation checklist

Before `flows_activate`:

1. confirm the user wants the automation live on real contacts
2. ensure the entry trigger exists
3. ensure triggerStart / triggerExit are configured
4. run `flows_validate`
5. report any remaining structural problems, including `incompleteChannelSteps` — message steps (email/telegram/WhatsApp) missing a real sender id. Resolve it with `email_sender_signatures_list` / `channel_instances_list` and `flows_step_update` before activating, rather than retrying `flows_activate` unchanged: a channel step without a real sender id fails every send no matter what activation itself currently checks

## Deletion checklist

Before `flows_delete` or `flows_step_delete`:

1. make sure the user explicitly wants deletion, not pause/stop/archive
2. explain that delete is irreversible
3. if deleting a step, warn that inbound targets to that step are also cleared
