# Flow trigger events and constraints

Flow entry/exit is driven by **events** matched against **constraints**. Set them
inline when creating the trigger step (`triggerStart` / `triggerExit`) or later
via `flows_step_triggers_set` (which replaces the whole list). A flow has at most
one trigger step; the events live at flow level, not in arbitrary step fields.

## Live catalog + labels

`flows_triggers_catalog` is the authoritative, always-current list of valid trigger
events: each row carries `eventName`, a friendly `label`, a `description`, its
`group`, and the `scopes` (offer/project/tag/…) it can be narrowed by. Read it to
pick a valid `eventName` and to get the label to show the user; the table below
mirrors it as a quick reference, but the tool is the source of truth.

Reads (`flows_structure_get` / `flows_get`) return a `triggerLabel` next to each
stored `eventName` — show that label to the user. Never surface a raw `eventName`
or a constraint `field` name (`offerId`, `projectId`, `tagId`, …); those are internal.

## Event shape

```js
{
  type: 'start',                      // 'start' (entry) | 'exit'
  eventName: 'crm.tag.lead.apply.v1', // exact string from the catalog below — never invent one
  constraints: [
    { field: 'tagId', op: 'EQ', valueType: 'UUID', valueUuid: '<uuid>' }
  ]
}
```

## Constraint shape

`{ field, op, valueType, value<Type>, caseSensitive? }` — max **5** constraints per trigger.

- `op` ∈ `EQ` `NEQ` `GT` `GTE` `LT` `LTE` `EXISTS` `IN` `CONTAINS` `NOT_CONTAINS` `STARTS_WITH` `REGEX` `WORD_MATCH`
- `valueType` ∈ `STRING` `NUMBER` `BOOLEAN` `DATE` `UUID` — put the value in the matching field:
  `valueString` / `valueNumber` / `valueBoolean` / `valueDate` / `valueUuid`. For `IN` (and list matches) use `valueJson` with an array.
- `caseSensitive` (text ops only): omitted = case-insensitive.
- Field → `valueType`: any `*Id` field is `UUID` (except `productIds` on Hubla/Hotmart events, which is `STRING` — the platform's own external id, not a Clickmax UUID); `channel`/`status` are `STRING`; `offset` (inactivity-window minutes) is `NUMBER`.
- `ownerId` is NEVER a settable scope/constraint field on any event — it is injected by the matcher outside the catalog (`packages/flows-elements/src/triggers/catalog.ts` comment: "ownerId é injetado pelo matcher, fora do catálogo"). Do not offer it as something the user can narrow by.
- Common case = a single `EQ` + `UUID` constraint scoping to one entity (a tag, an offer, a portal). Resolve the real id first (never invent it); leave `constraints: []` to match the event for any entity.
- This table is generated from `packages/flows-elements/src/triggers/catalog.ts` (`TRIGGER_CATALOG`, the actual single source of truth — re-derive this table from there, not from memory, if it's ever suspected stale). `flows_triggers_catalog` returns the same data live.

## Catalog — exact `eventName` → scopes (constraint fields the event can be narrowed by)

### Pages / leads

- `crm.lead.captured.v1` — a visitor submits a capture form on a page · `[funnelId, pageId]`

### Checkout / payments (scope = `offerId`, `projectId` unless noted)

- `checkout.page.access.v1` — checkout page accessed
- `cart.abandonment.v1` — **cart abandoned** (started checkout, did not finish)
- `checkout.pix.generated.v1` — pix charge generated
- `pix.expired.v1` — generated pix expires unpaid
- `checkout.boleto.generated.v1` — boleto generated
- `overdue.boleto.v1` — generated boleto becomes overdue unpaid
- `card.refused.v1` — **card payment attempt refused**
- `checkout.transaction.created.v1` — purchase transaction created (any payment method)
- `checkout.transaction.paid.v1` — **purchase approved/paid**
- `checkout.attempt.v1` — payment attempt at checkout
- `checkout.chargeback.requested.v1` — chargeback requested
- `checkout.refund.requested.v1` — refund requested

### Subscriptions (scope = `offerId`, `projectId`)

- `checkout.subscription.created.v1` — subscription created
- `checkout.subscription.canceled.v1` — subscription canceled
- `checkout.subscription.card.changed.v1` — subscription card changed
- `trial.expired.v1` — subscription trial period expires

### Hubla — external platform (scope = `productIds`, the Hubla product's own external id, `STRING` not UUID)

- `hubla.purchase.approved.v1` — Hubla invoice paid/approved
- `hubla.refund.v1` — Hubla invoice refunded
- `hubla.cart.abandonment.v1` — Hubla checkout filled but not finished
- `hubla.payment.failed.v1` — Hubla credit-card charge attempt fails
- `hubla.invoice.expired.v1` — Hubla invoice expires unpaid
- `hubla.subscription.created.v1` — Hubla subscription created
- `hubla.subscription.canceled.v1` — Hubla subscription canceled

### Hotmart — external platform (scope = `productIds`, the Hotmart product's own external id, `STRING` not UUID)

- `hotmart.purchase.approved.v1` — Hotmart purchase approved (payment confirmed)
- `hotmart.purchase.refunded.v1` — Hotmart purchase refunded
- `hotmart.purchase.chargeback.v1` — Hotmart chargeback
- `hotmart.purchase.canceled.v1` — Hotmart purchase canceled
- `hotmart.purchase.expired.v1` — Hotmart billet/pix purchase expires unpaid
- `hotmart.purchase.delayed.v1` — Hotmart recurring charge delayed
- `hotmart.billet.printed.v1` — Hotmart billet generated (awaiting payment)
- `hotmart.cart.abandonment.v1` — Hotmart checkout filled but not finished
- `hotmart.subscription.canceled.v1` — Hotmart subscription canceled
- `hotmart.subscription.switch.v1` — Hotmart subscription switches plan
- `hotmart.club.first.access.v1` — student's first access to the Hotmart members area
- `hotmart.club.module.completed.v1` — student completes a Hotmart course module

### Contacts / CRM

- `crm.lead.created.v1` — new lead created in the CRM · no scopes (fires for every lead)
- `crm.tag.lead.apply.v1` — tag applied to a lead · `[tagId]`
- `crm.tag.lead.remove.v1` — tag removed from a lead · `[tagId]`
- `crm.lead.birthday.reminder.v1` — it is a lead's birthday · no scopes (fires per lead workspace-wide, not narrowable to one lead)

### Opportunities (CRM kanban)

- `crm.opportunity.card.moved.v1` — an opportunity card moves between pipeline stages · `[pipelineId, fromStageId, toStageId]`
  - "entered stage X" = constrain `toStageId`; "left stage X" = constrain `fromStageId`. There is no separate entered/exited event.
  - Does not fire for cards with no primary lead (card↔lead is N:N; events without `leadId` are dropped).

### Members area (scope = `portalId`; lesson/module/course also take `contentId`)

- `members.area.accessed.v1` — members area accessed
- `members.banner.clicked.v1` — banner clicked in the members area
- `members.lesson.completed.v1` — lesson completed · `[portalId, contentId]`
- `members.module.completed.v1` — module completed · `[portalId, contentId]`
- `members.course.completed.v1` — course completed · `[portalId, contentId]`

### WhatsApp (scope = `gupshupAppId` unless noted)

- `crm.whatsapp.window.opened.v1` — 24h WhatsApp service window opens
- `crm.whatsapp.window.expiring.v1` — window about to expire · `[gupshupAppId, offset]`
- `crm.whatsapp.window.expired.v1` — window expired
- `crm.whatsapp.reply.received.v1` — contact replies on WhatsApp
- `hermes.gupshup.interaction.message.received.v1` — WhatsApp message received from a contact
- `crm.whatsapp.contact.silent.v1` — contact silent for a set time · `[gupshupAppId, offset]`
- `hermes.gupshup.interaction.template.button.v1` — contact taps a WhatsApp template button · `[gupshupAppId, buttonId]`

### Instagram (scope = `businessId` for message/comment events, `channelInstanceId` for window/silent events)

- `hermes.instagram.comment.received.v1` — comment received on an Instagram post · `[businessId, mediaId (IN, resolve via instagram_posts_list — never guess), commentText (CONTAINS/NOT_CONTAINS keyword)]`
- `hermes.instagram.story.reply.received.v1` — reply received to an Instagram story · `[businessId]`
- `hermes.instagram.message.received.v1` — direct message received on Instagram · `[businessId]`
- `hermes.instagram.message.ad.clicked.v1` — contact messages after clicking an Instagram ad · `[businessId]`
- `hermes.instagram.live.comment.received.v1` — comment received during an Instagram live · `[businessId]`
- `crm.instagram.window.opened.v1` — Instagram service window opens · `[channelInstanceId]`
- `crm.instagram.window.expiring.v1` — Instagram service window about to expire · `[channelInstanceId, offset]`
- `crm.instagram.window.expired.v1` — Instagram service window expired · `[channelInstanceId]`
- `crm.instagram.contact.silent.v1` — Instagram contact silent for a set time · `[channelInstanceId, offset]`

### Telegram / multichannel conversation

- `hermes.telegram.chat.start.v1` — contact starts a chat with the Telegram bot · `[telegramBotId]`
- `hermes.telegram.message.received.v1` — Telegram message received from a contact · `[telegramBotId]`
- `crm.conversation.started.v1` — new conversation on any connected channel · `[channelInstanceId, channel]`
- `crm.conversation.contact-silent.v1` — conversation contact silent for a set time · `[channel, offset]` (note the HYPHEN in `contact-silent`, not a dot — a guessed `crm.conversation.contact.silent.v1` is wrong and silently never fires)

### Mass automation (broadcast pseudo-events — valid `eventName` values but not real AMQP events)

- `manual` — one-off mass send to every contact on a list · `[listId (resolve via lists_list search — never guess; empty data means not found, check suggestions and offer those instead of inventing an id)]`
- `manual-tag` — one-off mass send to every contact with a tag · `[tagId]`

## Notes

- `flows_step_triggers_set` edits each side independently: omit `triggerStart` or `triggerExit` to keep that side untouched; send `[]` to clear it; the array you send replaces only that side. So a "change the entry trigger" request needs `triggerStart` only — the exit stays intact.
- Use the exact `eventName` from this catalog; an unknown name silently never fires.
- No scopes for an event (e.g. `crm.lead.created.v1`) means `constraints: []` is the only valid shape — there is nothing to narrow it by.
