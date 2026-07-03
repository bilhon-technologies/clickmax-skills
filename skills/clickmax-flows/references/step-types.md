# Flow step types and wiring

## Core step types

- `trigger`
  - entry point
  - `input = { type: 'event' }`
  - flow-level events live in `triggerStart` / `triggerExit`

- `action`
  - requires `action` + action-specific `input`
  - canonical action names matter; do not invent aliases
  - common actions:
    - `addTag` / `removeTag` -> `{ tagId }`
    - `addContactToList` / `removeContactFromList` -> `{ listId }`
    - `assignOpportunity` -> `{ opportunityId, stageId, attendantId?, allowMoveToPreviousStage? }`
    - `httpRequest` -> `{ preset, method, url, ... }`
    - `debug` -> `{}`

- `conditional`
  - `input = { mode: 'AND'|'OR', statements: [...] }`
  - branch by `handle: 'true' | 'false'`

- `delay`
  - `input = { when }`
  - numeric `when` = hours
  - string `when` can be a date-like expression

- `send_message`
  - PREFER the channel-specific tools over the generic `flows_step_create`: `flows_send_email` | `flows_send_sms` | `flows_send_whatsapp` | `flows_send_voice` | `flows_send_telegram` | `flows_send_instagram`. They take a precise per-channel schema (just `content`, plus `subject` for email) and set `message.type` for you. Use the generic `flows_step_create` only for non-message steps.
  - Raw shape (if you must use `flows_step_create`): channel via `input.type`: `email` | `sms` | `gupshup` (WhatsApp) | `telegram` | `voice` | `instagram`
  - `message` is REQUIRED and channel-specific; the body lives in `message.content`. `message.type` is REQUIRED — set it per channel: sms/telegram/gupshup/instagram → `text`; voice → `audio`; email → `html` (or `text`) AND email also needs `message.subject`. (If you omit `message.type` the server backfills the channel default, but always send it.)
  - Sender ids are OPTIONAL at creation — build the node now, configure the sender before activating: email `emailSenderSignatureId`, telegram `telegramBotId`, gupshup `gupshupAppId`. Omit when you don't have a real id; the node renders with the body and the user picks the sender in the builder (activation validates it). gupshup with no app falls back to the workspace's default WABA. NEVER invent a fake uuid for these — omit instead.
  - **Personalization tokens** in free-text bodies (sms/email/telegram/voice) use a flat lead key in SINGLE braces: `{name}`, `{email}`, `{telephone}`. These are the only recognized lead fields. Never `{{name}}`, never `{{lead.name}}`, never `{nome}`/`{firstName}` — an unknown or misformatted key is sent to the recipient literally. State it as fact; never hedge or ask the user which variable format to use.
  - Outputs captured by an upstream HTTP-request step are referenced the same way: `{variableName}`.
  - Checkout links are a separate token system: `{checkout}` (flow's default offer) or `{checkout:offerId}`.
  - Exception — GupShup/WhatsApp **templates**: the body is the approved template, personalized via positional `paramMapping: string[]` whose items are `{{...}}` placeholders resolved against lead + flow state (not free-text `{name}` tokens).

- `timeout`
  - `input = { when }`
  - branch with `true` / `false`

- `invoke`
  - `input = { flowId, origin, data? }`
  - field is `flowId`, not `flow`

- `collect`
  - `input = { field, customFieldName? }`
  - branch with `true` / `false`

## Connection model

- There is no standalone edge object
- Connecting = setting a `target` on a step output
- Linear connection: one `target`
- Branching connection: `handle -> target`
- Read canonical ids and edges from `flows_structure_get`

## Trigger event placement

- Do not put entry/exit events into arbitrary step input fields
- Use `triggerStart` / `triggerExit` inline on trigger creation or via `flows_step_triggers_set`
- These arrays replace the current lists
