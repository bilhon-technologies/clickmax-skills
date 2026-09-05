# Flow step types and wiring

## Core step types

- `trigger`
  - entry point
  - `input = { type: 'event' }`
  - flow-level events live in `triggerStart` / `triggerExit`

- `action`
  - requires `action` + action-specific `input`
  - **Call `flows_actions_catalog` before writing any action not spelled out below.** It is the only authoritative list of valid `action` values, their `input` fields, and which ones are `comingSoon` (locked). Canonical action names matter; do not invent aliases.
  - `addTag` / `removeTag` -> `{ tagId }` — a single uuid OR an array of uuids. `addTag` also takes `applyToOpportunities?: boolean`: `true` also tags EVERY opportunity card of the contact. Without it the card is tagged only when the flow already runs in opportunity context (stage automation); a lead-centric flow (capture, tag, purchase) has no card in state and leaves the opportunity untagged.
  - `addContactToList` / `removeContactFromList` -> `{ listId }`
  - `assignOpportunity` -> moves/creates the contact's opportunity card. See the dedicated block below — its `opportunityId` field is a trap.
  - `updateOpportunityCard` -> `{ value?, priority?, temperature?, status?, lossReason? }`. `value` is an INTEGER IN CENTS. `status: 'lost'` REQUIRES `lossReason` (max 500). This action writes OPPORTUNITY fields; `updateContactField` writes on the contact instead.
  - `updateContactField` -> `{ mapping: { scope, target }, value, writeMode?, multiSelectMode? }`
    - `mapping.scope` is `'lead'` or `'custom'`. `mapping.target` is a `crm.Leads` column under `'lead'`, or `cf.<customFieldId>` under `'custom'` — literally the `cf.` prefix plus the uuid from `custom_fields_list`. Same grammar as the form/quiz builder's field mapping.
    - `value` is ALWAYS a string in the schema (max 2000) and may carry flow variables (`{{utm_source}}`, `{{custom.cpf}}`, a quiz answer) resolved at runtime. The engine coerces to the destination column's type and DISCARDS what does not fit (invalid date/number, option outside the select) instead of writing `NaN`/`Invalid Date`.
    - `writeMode`: `always` (default) | `ifEmpty`. `multiSelectMode`: `replace` (default) | `append`, only meaningful for a `multi_select` custom field.
  - `createLeadTask` -> `{ title, description?, type? }` — `type` is `task` | `call` | `whatsapp` | `meeting`.
  - `createMeeting` -> `{ offsetDays, timeOfDay, durationMinutes, subject?, meetingLink?, notes? }` — `timeOfDay` is `"HH:MM"` 24h; the meeting lands `offsetDays` after the lead reaches the step.
  - `consolidateProposal` -> `{ templateId, title? }`
  - `notifyOwner` -> `{ title, body, recipient?, redirectTo? }` — INTERNAL product notification. `recipient` is `workspaceOwner` (default) or `attendant`, never the lead. `redirectTo`: `lead` (default) | `opportunity` | `none`.
  - `distribute` -> `{ pipelineId, teamId? | attendantIds?, temperature? }` — round-robin/raffle picker that writes `state.selectedAttendantId` for a FOLLOWING `assignOpportunity` to consume. Exactly one of `teamId` (engine round-robin) or `attendantIds` (uniform raffle among hand-picked attendants).
  - `httpRequest` -> `{ preset, method, url, ... }` plus `onError?` (`continue` = default | `fail`), `retryAttempts?` (0–3, transient failures only), `timeoutSeconds?` (1–120, default 30).
  - `debug` -> `{}`

### `assignOpportunity` — read before writing one

- ⚠️ **`opportunityId` is the PIPELINE id, not an opportunity/card id.** The engine consumes it as `pipelineId` throughout (`packages/flows-engine/src/action/assign-opportunity.ts`). Resolve it with `pipelines_list`; passing a card id targets nothing.
- `stageId` = destination stage inside that pipeline — `stages_list` with `id` = that same pipeline id.
- The action MOVES the contact's existing card in that pipeline, or CREATES one when the contact has none there.
  - `moveOnly: true` suppresses the create branch: no card in the pipeline means no-op. Use it for event triggers ("bought") that must move the existing card instead of spawning a second one.
  - Create-branch-only fields, ignored on a move: `value` (INTEGER IN CENTS), `offerIds`, `productIds`, `title`, `origin`, `temperature` (`hot`|`warm`|`cold`|`frozen`), `priority` (`high`|`medium`|`low`|`none`). Omit them and the new card is born degraded (null value/offer/origin), which breaks the sales funnel, reports, and the commission calculated on `card.value`.
- Attendant fields — three different things, never collapse them:
  - `attendantId` — a FIXED attendant.
  - `attendantTypeId` — resolve the attendant by the ROLE they already hold on the card ("no-show, hand it back to the SDR"). Takes precedence over `attendantId`, which becomes the explicit fallback. Ids from `pipelines_attendant_types_get` — it is an id and not a role name because attendant types are free per workspace.
  - `assignRoleTypeId` — the role the chosen attendant STARTS to hold on the card; this is where commission hangs. `assignRoleWhenTaken`: `keep` (default) | `replace`. `assignResponsible: false` writes only the role and leaves the card's responsible attendant untouched.
- `allowMoveToPreviousStage?: boolean`.
- There is NO `removeFromOtherPipelines` on this action. The flows3 drawer writes that key into the input, but it is not in the schema and no engine code reads it — it does nothing here. The real field of that name lives on CRM card creation (`cards_create`).

- `conditional`
  - `input = { mode: 'AND'|'OR', statements: [...] }`
  - branch by `handle: 'true' | 'false'`
  - Each statement is `{ type: <kind>, ...that kind's own fields }`. **Call `flows_conditionals_catalog` first** — it is the only place each `type` and its field list is documented, and an invented `type` cannot be evaluated by the engine. The kinds: `tagApplied`, `belongsToList`, `businessHours`, `contactId`, `email`, `httpStatus`, `leadFilter`, `matchLeadAttribute`, `matchReply`, `matchStateVariable`, `messageText`, `name`, `pageVisited`, `phone`, `whatsappWindow`, `instagramWindow`, `instagramFollows`.
  - Example: `{ type: 'tagApplied', tagId: '<uuid>', operator: 'is' }` (`is` | `is-not`).
  - Ids inside a statement are real workspace entities — resolve the tag/list/page/custom-field/business-hours-calendar id first; `httpStatus` instead references a `FlowsStep.id` from the flow being built.

- `delay`
  - `input = { when, mode?, direction?, referenceEventTypeId?, dateFieldId?, recheckOnResume? }`
  - **There is no unit/`type` field, and no minutes or days unit.** A numeric `when` is ALWAYS HOURS — use fractions for less (`0.5` = 30 min), multiples for more (`72` = 3 days). A string `when` is an ISO instant (UTC) and only means anything under `mode: 'absolute'`.
  - `mode`: `now` (default — offset from the moment the lead arrives) | `scheduled` (offset around the lead's next ScheduledCall; narrow it with `referenceEventTypeId`) | `dateField` (offset around a DATE custom field, id in `dateFieldId`) | `absolute` (`when` is a fixed date/time, the same for every lead).
  - `direction`: `before` | `after` (default) — only for the anchored modes (`scheduled`/`dateField`).
  - Targets are `continue` AND `noSchedule`, not one target. `noSchedule` is the branch for a lead with no anchor (no upcoming call, empty date field) or, under `absolute`, "the target date already passed". Leave it unwired and those leads dead-end.
  - `recheckOnResume?: boolean` re-reads the anchor on wake-up: pushed forward → reschedules, moved into the past or deleted → cancels. It is what makes "2h before the meeting" follow a rescheduling.

- `send_message`
  - PREFER the channel-specific tools over the generic `flows_step_create`: `flows_send_email` | `flows_send_sms` | `flows_send_whatsapp` | `flows_send_voice` | `flows_send_telegram` | `flows_send_instagram`. They take a precise per-channel schema (just `content`, plus `subject` for email) and set `message.type` for you. Use the generic `flows_step_create` only for non-message steps.
  - Email has three body-authoring modes (plain text / recolored default template / fully custom HTML) — see [email authoring](email-authoring.md) before writing one, especially before styling it.
  - Raw shape (if you must use `flows_step_create`): channel via `input.type`: `email` | `sms` | `gupshup` (WhatsApp) | `telegram` | `voice` | `instagram`
  - `message` is REQUIRED and channel-specific; the free-text body lives in `message.content`. `message.type` is REQUIRED — set it per channel: sms/telegram/instagram → `text`; gupshup → `text` (free-form) or `template` (approved template — see below, matches `format`); voice → `audio`; email → `html` (or `text`) AND email also needs `message.subject`. (If you omit `message.type` the server backfills the channel default, but always send it.)
  - **WhatsApp is template-first**: `format: 'text'` is rejected unless `numberStrategy: 'context'` — free-form text only delivers inside the 24h customer-care window. See [WhatsApp templates](whatsapp-templates.md) for how to pick, author, and submit one.
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
