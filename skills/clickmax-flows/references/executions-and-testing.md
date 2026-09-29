# Executions, testing and links

Execution = one contact's run through one flow. Tool contracts (filters, fields, errors) live in each tool description; this file = sequencing + traps.

## Which tool for which question

|Question|Tool|
|-|-|
|How many runs / stuck / failed? Top failure reasons? Did contact X go through?|`flows_executions_list`|
|Where did this run stop and why? Which branch did it take?|`flows_execution_get`|
|Does the flow work end to end?|`flows_test_run_start` → `flows_execution_get`|
|What did the trigger event carry?|`flows_trigger_samples_list`|
|What is this id in the run data?|`flows_entity_refs_resolve`|
|Which other automations start from what this step does?|`flows_link_targets_list`|
|Which automations feed this trigger?|`flows_link_sources_list`|
|Redo a failed run / all runs of one error|`flows_execution_retry` / `flows_executions_retry_by_error`|
|Stop one run|`flows_execution_cancel`|
|Stop new contacts entering|`flows_close` (not a cancel)|

## Diagnose a failure

1. `flows_executions_list` with `status: 'failed'` (add `stepId` for one node, `q` for one contact): read `counts` and `topErrors`.
2. `flows_execution_get` on one failed run: `steps[].error` on the failing node, `haltReason`.
3. Translate the cause to plain words (dead WhatsApp number, unapproved template, deleted tag/stage…). Resolve raw ids with `flows_entity_refs_resolve`; never show a UUID.
4. Fix the cause (a running flow's steps are locked: see [lifecycle and safety](lifecycle-and-safety.md)), then offer the retry with the number of contacts.

Defaults that mislead: `kind` omitted = production only (test runs hidden; `kind: 'all'` for both) | `counts` ignore `status`/`q`/date filters | `topErrors` is flow-wide, not the current page | `kind: 'message'` errors (send failures) are NOT retryable, only `kind: 'execution'` ones.

Production runs have no per-step trace, only step order and the failing step's error. The full trace (branch evaluation, effect, input/output) exists for test runs only, so to explain "why did it take the false branch" for a real contact, reproduce with a test run for that contact.

## Test run (`flows_test_run_start`)

- REAL run: sends messages, applies tags, moves cards, spends credits, honors waits, counts in metrics. Not a simulation and not activation.
- Runs the saved graph for ONE contact, no published flow needed, no same-contact debounce (repeat freely).
- Starts at the start step: it does not prove the real trigger event fires the flow.
- Payload is deduced from the trigger's scope (offer/pipeline/stage); send `payload` only to fill or override; a real one comes from `flows_trigger_samples_list`.
- Returns immediately; poll `flows_execution_get` (a wait step holds the run for its whole duration).
- Confirm before running, naming the recipient; default to the user's own contact.

## Retry / stop

- Retry re-runs the failed node with its side effect, then continues. Same cause → fails again.
- Bulk: `errors` = exact `topErrors[].error` labels of `kind: 'execution'`; at most 1000 per call, `remaining` > 0 → call again (confirm again: it touches more contacts).
- Archived flow → 409 on retry; paused (`closed`) flow can be retried.
- Cancel: permanent for that run, the in-flight step may still finish; 409 = nothing to stop.

## Validation issues

`flows_validate` → `issues[]` (code, severity, `stepId`, `detail`). `valid` = no error; warnings never flip it. Activation itself blocks only a missing entry trigger and incomplete channel steps, so the other errors still need fixing before activating.

|Severity|Codes|
|-|-|
|error|`missingEntryTrigger` · `danglingTarget` · `incompleteChannelStep` · `whatsappTemplateNotApproved` · `brokenReference` (tag/stage/pipeline/offer/list/number/template deleted) · `senderNumberUnavailable` (pinned WhatsApp number archived, deactivated or automations disabled)|
|warning|`unreachableStep` · `stepWithoutNext` (may be intended) · `emptyConditionalBranch` · `triggerWithoutScope` · `variableOutsideTriggerContext` (message uses a variable the trigger never provides; renders empty)|

Numbers taken from the trigger (`numberStrategy: 'context'`) cannot be checked before runtime.

## Links between automations

- `flows_link_targets_list` = what starts downstream of a tag/stage/win/message outcome; `flows_link_sources_list` = who produces what a trigger listens to.
- Build `refs` yourself from ids in the flow (`flows_structure_get` / `flows_get`): `tag:<id>`, `stage:<id>`, `won`, `lost`, `emailClicked:<sendStepId>`… (full grammar in the tool description).
- Only literal ids are detectable; an empty result means "no detectable link", not "no relation".
- Results include `draft`/`closed` flows: check `mode` before saying something will fire; `stats` = production runs only.
- Typical use: before deleting/archiving a flow or changing a tag, list who reacts to it.
