# Flow step types and wiring

## Core step types

- `trigger`
  - entry point
  - `input = { type: 'event' }`
  - flow-level entry events live in `triggerStart` (exits = flow goal, read-only here)

- `action`
  - requires `action` + action-specific `input`
  - **Call `flows_actions_catalog` before writing any action not spelled out below.** It is the only authoritative list of valid `action` values, their `input` fields, and which ones are `comingSoon` (locked). Canonical action names matter; do not invent aliases.
  - **Conditional requirements are NOT checked at save.** Fields marked "required with X" below are optional in the schema; a step missing one saves fine and becomes a logged no-op at run time. Always send them.
  - `addTag` / `removeTag` -> `{ tagId, applyToOpportunities? }` — `tagId` = a single uuid OR an array of uuids (`tags_list`). `applyToOpportunities: true` also tags (`addTag`) or untags (`removeTag`) EVERY opportunity card of the contact. Without it only the card already in state is touched (stage automation); a lead-centric flow (capture, tag, purchase) leaves the opportunities as they were. These two do NOT use the automatic opportunity lookup below.
  - `addContactToList` / `removeContactFromList` -> `{ listId }`
  - `adjustLeadTemperature` -> `{ signal }` with `signal` one of `warmLight` | `warm` | `cool` | `coolStrong` (closed domain). It does NOT write a number: it records a synthetic activity that the Temperature engine weighs like any other (it decays with time; the cooling signals stay out of recency). A flow instance with no contact is skipped. Needs a contact in state.
  - `setData` -> `{ fields: [{ name, type, value }] }` (1..50) | `name` = `^[A-Za-z_][A-Za-z0-9_]*$`, unique | `type` = `string`|`number`|`boolean`|`json` (failed coercion → `null`, `number` accepts `150,50`) | `value` = text with variables/data expressions. Publishes `{name: value}` as the step output; writes nothing to the contact (that is `updateContactField`).
  - `studioCreative` -> `{ templateId, variables: [{ name, value }] }` (0..30) | `templateId` = a Video Studio template saved by the user ("Salvar como modelo"), mirrored to the server when the Studio opens; there is no tool to list them, so ask the user to pick it in the editor | `name` = template variable (`^[a-z][a-z0-9_]{0,31}$`), `value` ≤ 400 chars with variables/data expressions. Writes a request to the Studio queue and continues at once; the creative is generated later on the user's computer, when the Studio is open in Clickmax. Output: `requestId`.
  - `assignOpportunity` -> moves/creates the contact's opportunity card. See the dedicated block below — its `opportunityId` field is a trap.
  - `updateOpportunityCard` -> `{ value?, priority?, temperature?, status?, lossReason?, closingDateOffsetDays?, clearClosingDate? }`. `value` is an INTEGER IN CENTS. `priority`: `high`|`medium`|`low`|`none`; `temperature`: `hot`|`warm`|`cold`|`frozen`; `status`: `won`|`lost` — `lost` REQUIRES `lossReason` (1..500). Expected closing date is RELATIVE: `closingDateOffsetDays` (integer 0..3650) = the day the execution reaches the step + N days (date only, UTC day); `clearClosingDate: true` empties it and wins over the offset. There is no fixed-date field. This action writes OPPORTUNITY fields (opportunity resolved as in [which opportunity](#which-opportunity-an-opportunity-action-uses)); `updateContactField` writes on the contact instead.
  - `updateContactField` -> `{ mapping: { scope, target }, value, writeMode?, multiSelectMode? }`
    - `mapping.scope` is `'lead'` or `'custom'`. `mapping.target` is a `crm.Leads` column under `'lead'`, or `cf.<customFieldId>` under `'custom'` — literally the `cf.` prefix plus the uuid from `custom_fields_list`. Same grammar as the form/quiz builder's field mapping.
    - `value` is ALWAYS a string in the schema (max 2000) and may carry flow variables (`{{utm_source}}`, `{{custom.cpf}}`, a quiz answer) resolved at runtime. The engine coerces to the destination column's type and DISCARDS what does not fit (invalid date/number, option outside the select) instead of writing `NaN`/`Invalid Date`.
    - `writeMode`: `always` (default) | `ifEmpty`. `multiSelectMode`: `replace` (default) | `append`, only meaningful for a `multi_select` custom field.
  - `createLeadTask` -> `{ title, description?, type?, deadlineOffsetDays?, deadlineTimeOfDay?, contactFallback? }` | `title` 1..255, `description` ≤ 2000 | `type`: `task` (default) | `call` | `whatsapp` | `meeting` | deadline = day the execution reaches the step + `deadlineOffsetDays` (integer 0..365) at `deadlineTimeOfDay` (`"HH:MM"` 24h, **UTC** — convert from the user's local time); send BOTH or the task has no deadline. The task goes on the opportunity ([which opportunity](#which-opportunity-an-opportunity-action-uses)); no open opportunity → skipped, unless `contactFallback: true`, which then creates a contact-only task.
  - `createMeeting` — two modes, chosen by `appointmentScheduleId`:
    - **Agenda mode (use this for new steps)** -> `{ appointmentScheduleId, offsetDays, subject?, notes? }`. Books the contact into the FIRST FREE SLOT of that agenda starting today + `offsetDays` (agenda timezone; `0` = from now). `offsetDays` must be an INTEGER 0..365 — a fraction or anything above 365 books nothing (`invalid_offset`). Duration, host/team member, lead time, daily cap and location (Clickmax Meet / Google Meet / fixed link) all come from the agenda: `timeOfDay`, `durationMinutes`, `meetingProvider`, `meetingLink` are ignored here. Group agendas are not supported. No MCP tool lists agendas — ask the user to pick the agenda in the step editor (or give its id); never invent one.
    - It never fails the execution: output `{ booked: true, scheduledCallId, startAt }` or `{ booked: false, reason }`, `reason` ∈ `no_slots` | `group_schedule` | `lead_missing` | `invalid_offset` | `crm_error` | `crm_not_configured`. A booked meeting is stored in state, so a later `cancelMeeting` with `which: 'execution'` cancels exactly it.
    - **Legacy fixed-time mode (no `appointmentScheduleId`)** -> `{ offsetDays, timeOfDay, durationMinutes, meetingProvider?, meetingLink?, subject?, notes? }` — fixed time `offsetDays` after the step at `timeOfDay` (`"HH:MM"`, UTC), `durationMinutes` ≥ 1, no attendant. `timeOfDay` + `durationMinutes` are REQUIRED in this mode (missing → no-op; malformed `timeOfDay` fails the step). `meetingProvider`: `clickmax` (a Clickmax Meet room is created) | `google` (no link; Google Meet is generated by the calendar sync when the attendant has Google connected) | `custom` (uses `meetingLink`). Omitted → `meetingLink` if set, else Google. Only for keeping old steps; prefer the agenda mode.
  - `consolidateProposal` -> `{ templateId, title? }` | `templateId` from `proposal_templates_list`, `title` ≤ 255 (default = template title). Available to every workspace. Generates the frozen proposal for the opportunity ([which opportunity](#which-opportunity-an-opportunity-action-uses)); with no open opportunity it still falls back to the contact's latest opportunity (won/lost included). Needs a contact or an opportunity in state.
  - `notifyOwner` -> `{ title, body, recipient?, userId?, redirectTo? }` — INTERNAL product notification, never to the lead. `title` 1..255, `body` non-empty. `recipient`:
    - `workspaceOwner` (default);
    - `attendant` = the attendant of the MEETING that started the execution (scheduling triggers only; any other trigger → no-op);
    - `user` = a fixed teammate: `userId` REQUIRED — the `id` from `workspace_members_list` (person/user id, NOT an attendant id). Someone no longer an active member → no-op.
    - `redirectTo`: `lead` (default) | `opportunity` | `none`. `opportunity` without a card in state uses the contact's open opportunity (and stores it for later steps).
  - `distribute` -> `{ pipelineId, teamId? | attendantIds?, temperature? }` — round-robin/raffle picker that writes `state.selectedAttendantId` for a FOLLOWING `assignOpportunity` to consume. Exactly one of `teamId` (engine round-robin) or `attendantIds` (uniform raffle among hand-picked attendants).
  - `httpRequest` -> `{ preset, method, url, ... }` plus `onError?` (`continue` = default | `fail`), `retryAttempts?` (0–3, transient failures only), `timeoutSeconds?` (1–120, default 30).
  - `debug` -> `{}`
  - `findOpportunity` -> `{ pipelineId }` — **LEGACY: never create it** (absent from `flows_actions_catalog`). Old steps still run and pin the contact's open card of that pipeline; new flows do not need it, opportunity actions resolve the card on their own (next section).

  Opportunity actions — all resolve the card as in [which opportunity](#which-opportunity-an-opportunity-action-uses); no open opportunity → step skipped, execution continues. Ids: pipelines `pipelines_list`, stages `stages_list`, attendants `attendants_list`.
  - `setOpportunityField` -> `{ customFieldId, value, writeMode? }` | `customFieldId` = an OPPORTUNITY custom field (`custom_fields_list` with `entityType: 'opportunities'`; a contact field is a no-op) | `value` 1..2000, text with variables/data expressions | `writeMode`: `always` (default) | `ifEmpty`. Coerced to the field type like `updateContactField` (what does not fit is discarded); a value that resolves empty clears nothing.
  - `setOpportunityResponsible` -> `{ attendantId }` — changes the card's responsible WITHOUT moving it. The attendant must belong to the workspace AND to the card's pipeline, else no-op.
  - `transferOpportunity` -> `{ pipelineId, stageId, keepResponsible? }` — moves the card to a stage of ANOTHER pipeline (`stageId` must belong to `pipelineId`). Card already in that pipeline → no-op (to move between stages of the same pipeline use `assignOpportunity` with `moveOnly: true`). The target stage rules apply (blocked → no-op). `keepResponsible` default `true`, kept only if that attendant is a member of the target pipeline; roles are kept. Fires the target stage automations.
  - `removeOpportunity` -> `{ pipelineIds }` — archives (soft-deletes) the card, like "Excluir" in the CRM. `pipelineIds` is REQUIRED (max 50): `[]` = any pipeline, otherwise only a card of those pipelines is removed.
  - `updateOpportunityOffers` -> `{ offerIds, mode, applyOfferValue? }` | `offerIds` 1..50 (offers of a product: `products_list` → `products_get`) | `mode`: `add` | `remove` | `replace` | `applyOfferValue: true` sets the card `value` to the sum of the current prices of the offers left on it (cents; none left = 0). On `add`/`replace` unknown or deleted offers are ignored; `remove` accepts any id.
  - `addOpportunityNote` -> `{ body }` — note on the card timeline, `body` 1..5000 with variables; resolves empty → no-op. There is no contact-only note.
  - `linkCompanyToOpportunity` -> `{ source, companyId? }` | `source`: `fixed` (`companyId` REQUIRED) | `leadCompany` (the contact's company; with several, the one linked longest). Adds — never replaces — and becomes primary only when the card has none. No MCP tool lists companies: read ids from `cards_get` (companies of a card) or `leads_context` (companies of a contact), or prefer `leadCompany`.
  - `addContactToOpportunity` -> `{ makePrimary? }` — links the execution's contact to the card. Useful when the card came from state (stage automation, a meeting of another contact); with the automatic lookup the contact is already on it. `makePrimary: true` makes them the primary contact (the previous one stays, as secondary); otherwise primary only if the card has none.
  - `duplicateOpportunity` -> `{ pipelineId, stageId, copyCustomFields?, copyTags?, responsibleAttendantId?, continueWithNew? }` — new card (end of that stage) copying title, value, probability, offers/products, priority, temperature, closing date, origin, referral, recurrence, ALL contacts and companies; never history, status, roles or commissions. `copyCustomFields` / `copyTags` default `true` (unique custom fields are never copied). `responsibleAttendantId` must be a member of the TARGET pipeline, else the card is created without one (omitted = no responsible). If the primary contact already has an OPEN card in the target pipeline, that card is reused and nothing is copied — duplicating an open card into its own pipeline is a no-op. `continueWithNew: true` makes the following steps act on the new card. Output: `cardId`.

  Contact and execution:
  - `removeFromOtherFlows` -> `{ mode, flowIds? }` | `mode`: `all` | `selected` (`flowIds` REQUIRED, max 50, from `flows_list`) — cancels the contact's RUNNING executions in the OTHER flows of the workspace. The current flow is always excluded (to leave it, just end the path) and test runs are untouched. Output: `canceledCount`.
  - `updateMessagePreferences` -> `{ channels, subscribe? }` | `channels` 1..6 unique of `email` | `whatsapp` | `sms` | `telegram` | `instagram` | `voice` | `subscribe` omitted/`false` = UNSUBSCRIBE (opt-out, same as the unsubscribe link); `true` = accept messages again. Needs a contact.
  - `formatData` -> `{ fields: [{ name, value, operation }] }` (1..20) — like `setData`, publishes each `name` as step output (`{{$step.<id>.<name>}}`) and writes nothing to the contact. `name` = `setData` rule (max 64); `value` ≤ 4000, text with variables/data expressions. `operation`: `phoneE164` (digits WITHOUT `+`, e.g. `5511987654321` — the contact's phone format) | `digitsOnly` | `uppercase` | `lowercase` | `capitalize` (each word) | `trim` | `number` (accepts `1.234,56`) | `dateIso` (`YYYY-MM-DD`) | `dateBr` (`DD/MM/YYYY`). Dates read `DD/MM/YYYY` or ISO. Input that does not fit the operation → `null`, never the raw text. Only `number` outputs a number.

  Tasks and calendar:
  - `completeLeadTasks` -> `{ scope, types?, outcomeNote? }` | `scope`: `opportunity` (tasks of the resolved opportunity; none → skipped) | `contact` (contact-only tasks + tasks of ALL the contact's cards) | `types` subset of `task` | `call` | `whatsapp` | `meeting`, omitted/empty = all | `outcomeNote` ≤ 1000 with variables. Only OPEN tasks are completed (cancelled stay cancelled); a follow-up cadence is NOT advanced to its next step. Output: `completedCount`.
  - `cancelMeeting` -> `{ which, reason? }` | `which`: `execution` (the meeting that started the execution, or the one an earlier `createMeeting` booked) | `next` (nearest future open meeting) | `allFuture` | `reason` ≤ 500. `next`/`allFuture` look at the contact and at the card ALREADY in state — no automatic opportunity lookup. Canceled through the CRM: participants are notified and the external calendar event is removed. Output: `canceledCount`.
  - `blockCalendar` -> `{ attendantSource, attendantId?, offsetDays, startTime?, endTime?, reason? }` | `attendantSource`: `fixed` (`attendantId` REQUIRED) | `meetingAttendant` (attendant of the meeting that started the execution) | day = today in the ATTENDANT's timezone + `offsetDays` (integer 0..365) | `startTime`/`endTime` `"HH:MM"` in the attendant's local time, both or neither (neither = whole day), end after start, else no-op | `reason` ≤ 200, only kept in the execution log.

  Custom objects (ids: `custom_objects_list`; fields `custom_fields_list` with `objectId`; relations `custom_object_relations_list`):
  - `createCustomObjectRecord` -> `{ objectId, values: [{ customFieldId, value }], linkRelationId?, linkTarget? }` | `values` 1..50, `value` ≤ 2000 with variables, coerced to the field type (misfit discarded, empty skipped) | the object's TITLE field must be in `values` and resolve non-empty, else no record | `linkRelationId` REQUIRES `linkTarget` (`lead` | `opportunity`) and must be a relation of THIS object pointing to that side; links to the contact or to the resolved opportunity. Archived object, foreign field, or a unique-value clash → no-op. Output: `recordId`.
  - `updateCustomObjectRecord` -> `{ objectId, customFieldId, value, recordSource, sourceStepId?, relationId?, linkTarget? }` — writes ONE field of ONE record. `recordSource`: `step` (`sourceStepId` REQUIRED = the `FlowsStep.id` of an UPSTREAM `createCustomObjectRecord`) | `linked` (`relationId` + `linkTarget` REQUIRED; updates only when EXACTLY one live record is linked to the contact/opportunity — zero or several → no-op). `value` ≤ 2000; resolving empty clears nothing.

  Members area (classrooms: `portals_list` → `classrooms_list` per portal):
  - `grantClassroomAccess` -> `{ classroomIds, sendAccessEmail? }` | `classroomIds` 1..20, ALL must belong to the workspace or the whole step is a no-op | `sendAccessEmail` default `true` (sent only when the contact gains a new portal). The contact needs an e-mail (it is the login), else no-op.
  - `revokeClassroomAccess` -> `{ classroomIds }` (1..20) — removes the contact from those classrooms, and from the portal when no classroom is left there. Needs the contact's e-mail; unknown classrooms are ignored.

  Team:
  - `emailTeammate` -> `{ userId, subject, body }` — SYSTEM e-mail (Clickmax layout and sender) to a workspace member, not to the lead: no credits, no unsubscribe check, nothing in the contact's conversation. `userId` = `id` from `workspace_members_list`; not an active member → no-op. `subject` 1..200, `body` 1..5000, PLAIN text (no HTML; line breaks kept), both with variables; resolving empty → no-op.

### Which opportunity an opportunity action uses

Every opportunity action above (and `updateOpportunityCard`, `createLeadTask`, `consolidateProposal`, `notifyOwner` with `redirectTo: 'opportunity'`, `completeLeadTasks` with `scope: 'opportunity'`, custom-object links with `linkTarget: 'opportunity'`) picks its card in this order:

1. `state.cardId` — the card of the execution: opportunity triggers / stage automations, or set by an earlier step.
2. `state.opportunityCardId` — scheduling triggers carry the meeting's card here.
3. The contact's most recent OPEN opportunity in ANY pipeline (newest created; won/lost are never picked). It is stored as `state.cardId`, so every later step acts on that same card even if a newer one appears meanwhile.

No open opportunity → the step is skipped (logged, visible in the execution trace) and the execution continues; nothing is created. So a lead-centric flow (form, tag, purchase, webhook) needs no lookup step, but if the contact may have several open opportunities in different pipelines the newest wins — when the pipeline matters, build it as a stage automation of that pipeline, or create/move the card first with `assignOpportunity`. Not covered: `addTag`/`removeTag` (use `applyToOpportunities`), `cancelMeeting` (card already in state only) and `assignOpportunity` (its own pipeline logic).

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
  - Each statement is `{ type: <kind>, ...that kind's own fields }`. **Call `flows_conditionals_catalog` first** — it is the only place each `type` and its field list is documented, and an invented `type` cannot be evaluated by the engine. The kinds: `tagApplied`, `belongsToList`, `businessHours`, `contactId`, `email`, `httpStatus`, `leadFilter`, `matchLeadAttribute`, `matchReply`, `matchStateVariable`, `messageText`, `name`, `pageVisited`, `phone`, `whatsappWindow`, `instagramWindow`, `instagramFollows`, `dataExpression`.
  - `dataExpression` -> `{ left, operator, right? }` | `left`/`right` = text with data expressions (below) | operators `equals`|`notEquals`|`contains`|`notContains`|`startsWith`|`endsWith` (case-insensitive text; numbers compare as numbers) · `gt`|`gte`|`lt`|`lte` (numeric; non-number → false) · `isEmpty`|`isNotEmpty` (no `right`). No regex operator.
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
  - Inside a `flows_graph_apply` change set, a message is an `add` op with the raw shape below. For one message alone, the channel-specific tools (`flows_send_email` | `flows_send_sms` | `flows_send_whatsapp` | `flows_send_voice` | `flows_send_telegram` | `flows_send_instagram`, each accepting `after: {step, handle?}`) take a precise per-channel schema (just `content`, plus `subject` for email) and set `message.type` for you.
  - Email has three body-authoring modes (plain text / recolored default template / fully custom HTML) — see [email authoring](email-authoring.md) before writing one, especially before styling it.
  - Raw shape (`add` op / `flows_steps_create`): channel via `input.type`: `email` | `sms` | `gupshup` (WhatsApp) | `telegram` | `voice` | `instagram`
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
  - **Data from previous steps** (any text field: message, HTTP url/body, `updateContactField.value`, `askAiAgent.instruction`, `setData`, `dataExpression`): `{{$trigger.path}}` (raw trigger payload) · `{{$prev.path}}` (step executed right before) · `{{$step.<FlowsStep.id>.path}}` | path = `a.b[0]["key with space"]` | fallback `{{$prev.reply|default}}` | no JS. Step must be UPSTREAM in the graph, else empty + `flows_validate` warns `invalidStepReference`. A value that is exactly one `{{$...}}` keeps its type (number/list/object) in `setData`, conditions and JSON HTTP bodies.
    - Outputs per step: `httpRequest` → `status`, `body` · `askAiAgent` → `reply` · `aiConversation` → `exit`, `reason` · collect / message waiting reply → `reply` (+ `option` in menu) · `setData` / `formatData` → each `name` · `studioCreative` → `requestId` · `removeFromOtherFlows` / `cancelMeeting` → `canceledCount` · `completeLeadTasks` → `completedCount` · `duplicateOpportunity` → `cardId` · `createCustomObjectRecord` → `recordId`.
  - Checkout links are a separate token system: `{checkout}` (flow's default offer) or `{checkout:offerId}`.
  - Exception — GupShup/WhatsApp **templates**: personalized via positional `paramMapping: string[]` whose items are `{{...}}` placeholders resolved against lead + flow state (not free-text `{name}` tokens) — never the `content` field, which is the template's `elementName` in this mode (see above).

- `timeout` / `collect` — hidden helpers of a message that waits for a reply; create them with the `capture` option of `add`, never as standalone steps, and wire them through the message's `invalid` / `timeout` handles

- `invoke`
  - `input = { flowId, origin, data? }`
  - field is `flowId`, not `flow`

## Connection model

- No standalone edge object: connecting = setting a step output's `target`
- `connect {from, handle, to}` replaces that output's current destination; `handle` omitted/null = main output
- Read real ids and each step's `outputs[]` from `flows_structure_get`

## Output handles

Unknown handle → rejected with the step's valid outputs; `flows_structure_get` lists free outputs too (`target: null`).

|Step|handles|
|-|-|
|simple (`action`, `invoke`, SMS/email/Telegram, Instagram replies, `trigger`)|main only|
|`conditional`|`true`, `false`|
|randomizer|each `input.branches[].key`|
|switch|each branch key, `else`|
|`delay`|`continue`; + `noSchedule` when `mode` ∈ `scheduled`/`dateField`/`absolute`|
|WhatsApp/Instagram with buttons|each button id, `default`, `noReply` (WhatsApp)|
|voice with branches|`next`, `default`, `gte<N>`, `noAnswer`, `failed`|
|message waiting for a reply (`capture`)|main = valid reply, `invalid`, `timeout` (if set), each menu case key|
|sequence|main = after its last inner step|

## Trigger event placement

- Do not put entry/exit events into arbitrary step input fields
- Use `triggerStart` via the `setTriggers` op of `flows_graph_apply` (or `flows_steps_triggers_set`); exits are the flow goal, configured by the user in the editor
- These arrays replace the current lists
