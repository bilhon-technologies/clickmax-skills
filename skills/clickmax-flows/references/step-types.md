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
  - Email has three body-authoring modes (plain text / recolored default template / fully custom HTML) — see [email authoring](email-authoring.md) before writing one, especially before styling it.
  - Raw shape (if you must use `flows_step_create`): channel via `input.type`: `email` | `sms` | `gupshup` (WhatsApp) | `telegram` | `voice` | `instagram`
  - `message` is REQUIRED and channel-specific; the free-text body lives in `message.content`. `message.type` is REQUIRED — set it per channel: sms/telegram/instagram → `text`; gupshup → `text` (free-form) or `template` (approved template — see below, matches `format`); voice → `audio`; email → `html` (or `text`) AND email also needs `message.subject`. (If you omit `message.type` the server backfills the channel default, but always send it.)
  - **GupShup `content` with `format: 'template'` is NOT the message body** — it holds the approved template's `elementName`, and the actual copy comes from the template itself (personalized via `paramMapping`, below). Writing prose into `content` in this mode breaks the builder (`templateNotFound` — it looks the node up by that exact name) and is exactly the case `gupshupTemplateId` (below) exists to pin down instead of leaving the send to guess. Resolve the real template first with `gupshup_templates_list`/`gupshup_templates_get` and pass its `elementName` as `content` + its id as `gupshupTemplateId`.
  - Sender ids: no safe fallback for any of them — a node saved without one fails on every send, permanently, not just once the flow activates. Always try to resolve the real id first; never invent a fake uuid. NEVER refuse to build the automation just because a channel isn't configured yet — create the node with content/copy anyway and flag it in the reply (see below). This applies the same way to email and telegram; only WhatsApp's `numberStrategy: 'fixed'` behaves differently (see below).
    - `telegramBotId` (telegram): try `channel_instances_list` (`channel: 'telegram'`, `status: 'live'`) first and pass it whenever one exists. OPTIONAL in the tool schema — omit it ONLY when the workspace has no live bot connected at all. If you omit it, say so plainly in your reply: the user has no Telegram bot connected yet, so this node will not send until they connect one.
    - `emailSenderSignatureId` (email): try `email_sender_signatures_list` (pick `confirmed: true`) first and pass it whenever one exists. OPTIONAL in the tool schema — omit it ONLY when the workspace has no confirmed signature at all. If you omit it, say so plainly in your reply: the user has no sender email configured yet, so this node will not send until they add and verify one.
    - `gupshupAppId` (WhatsApp): required only when `numberStrategy: 'fixed'` — under `'context'` the runtime resolves it from the trigger instead. **There is no "workspace default WABA" fallback under `'fixed'`** — omitting it throws a permanent `JobError` at send time (COPS-2362; this was a real production incident, 38 flows). Resolve it with `channel_instances_list` (`channel: 'gupshup'`, `status: 'live'`).
    - Resolve every sender id at CREATION time when it exists — never leave it for later. When it doesn't exist yet, build the node anyway (content, copy, everything else) and name the gap in your reply instead of blocking the whole automation on it — `flows_validate`'s `incompleteChannelSteps` and the activation gate both catch a missing sender/bot later, so nothing silently ships broken; you're just not required to stall the build waiting for the user to go configure a channel first.
  - **Personalization tokens** in free-text bodies (sms/email/telegram/voice, and GupShup `format: 'text'`) use a flat lead key in SINGLE braces: `{name}`, `{email}`, `{telephone}`. These are the only recognized lead fields. Never `{{name}}`, never `{{lead.name}}`, never `{nome}`/`{firstName}` — an unknown or misformatted key is sent to the recipient literally. State it as fact; never hedge or ask the user which variable format to use.
  - Outputs captured by an upstream HTTP-request step are referenced the same way: `{variableName}`.
  - Checkout links are a separate token system: `{checkout}` (flow's default offer) or `{checkout:offerId}`.
  - Exception — GupShup/WhatsApp **templates**: personalized via positional `paramMapping: string[]` whose items are `{{...}}` placeholders resolved against lead + flow state (not free-text `{name}` tokens) — never the `content` field, which is the template's `elementName` in this mode (see above).

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
