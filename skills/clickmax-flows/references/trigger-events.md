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
- Field → `valueType`: any `*Id` field is `UUID`; `text` / `messageText` / `commentText` / `status` / `channel` / `type` / `offset` / `contentType` / `buttonId` are `STRING`; `containsList` / `notContainsList` / `postIds` / `storyIds` take `valueJson` arrays.
- Common case = a single `EQ` + `UUID` constraint scoping to one entity (a tag, an offer, a portal). Resolve the real id first (never invent it); leave `constraints: []` to match the event for any entity.

## Catalog — exact `eventName` → available constraint fields

### CRM

- `crm.tag.lead.apply.v1` — tag applied · `[tagId]`
- `crm.tag.lead.remove.v1` — tag removed · `[tagId]`
- `crm.lead.captured.v1` — lead captured by a funnel page · `[ownerId, projectId, pageId, funnelId, publicIngestLinkId]`
- `crm.lead.created.v1` — lead created · `[ownerId, projectId, pageId, funnelId, publicIngestLinkId]`
- `crm.lead.birthday.reminder.v1` — lead birthday · `[leadId]`
- `crm.whatsapp.window.opened.v1` — 24h WhatsApp window opened · `[gupshupAppId]`
- `crm.whatsapp.reply.received.v1` — WhatsApp reply received · `[gupshupAppId]`
- `crm.whatsapp.window.expiring.v1` — window about to close · `[gupshupAppId, offset]`
- `crm.whatsapp.window.expired.v1` — window closed · `[gupshupAppId]`
- `crm.whatsapp.contact.silent.v1` — contact went silent (WhatsApp) · `[gupshupAppId, offset]`
- `crm.conversation.started.v1` — new conversation (multichannel) · `[channel, channelInstanceId]`
- `crm.conversation.contact.silent.v1` — contact silent (multichannel) · `[channel, offset]`

### Payments (scope with `offerId`; `type` on transactions; all `*Id` are UUID)

- `checkout.transaction.created.v1` — order created · `[offerId, ownerId, projectId, type]`
- `checkout.transaction.paid.v1` — **sale paid** · `[offerId, ownerId, projectId, type]`
- `checkout.page.access.v1` — checkout page accessed · `[offerId, ownerId, projectId]`
- `checkout.subscription.created.v1` — subscription created · `[offerId, ownerId, projectId]`
- `checkout.boleto.generated.v1` — boleto generated · `[offerId, ownerId, projectId]`
- `checkout.pix.generated.v1` — pix generated · `[offerId, ownerId, projectId]`
- `checkout.chargeback.requested.v1` — chargeback requested · `[offerId, ownerId, projectId]`
- `checkout.refund.requested.v1` — refund requested · `[offerId, ownerId, projectId]`
- `checkout.subscription.card.changed.v1` — subscription card changed · `[offerId, ownerId, projectId]`
- `checkout.subscription.canceled.v1` — subscription canceled · `[offerId, ownerId, projectId]`
- `cart.abandonment.v1` — **abandoned cart** · `[offerId, ownerId, projectId]`
- `overdue.boleto.v1` — boleto overdue · `[offerId, ownerId, projectId]`
- `pix.expired.v1` — pix expired · `[offerId, ownerId, projectId, buyerId, transactionId]`
- `trial.expired.v1` — trial expired · `[offerId, ownerId, projectId, leadId, transactionId]`
- `card.refused.v1` — **card refused** · `[offerId, ownerId, projectId]`
- `checkout.attempt.v1` — payment attempt · `[offerId, ownerId, projectId]`

### Members

- `members.area.accessed.v1` — members area accessed · `[portalId, memberUserId]`
- `members.banner.clicked.v1` — banner clicked · `[portalId, bannerId, memberUserId]`
- `members.lesson.completed.v1` — lesson completed · `[portalId, contentId, memberUserId]`
- `members.module.completed.v1` — module completed · `[portalId, contentId, memberUserId]`
- `members.course.completed.v1` — course completed · `[portalId, contentId, memberUserId]`

### Channels (Telegram / Instagram / WhatsApp-GupShup)

- `hermes.telegram.chat.start.v1` — Telegram chat started · `[telegramBotId]`
- `hermes.telegram.message.received.v1` — Telegram message · `[telegramBotId, text, contentType, containsList, notContainsList]`
- `hermes.instagram.comment.received.v1` — Instagram comment · `[businessId, postIds, commentText, containsList, notContainsList]`
- `hermes.instagram.story.reply.received.v1` — Instagram story reply · `[businessId, storyIds, containsList, notContainsList]`
- `hermes.instagram.message.received.v1` — Instagram DM · `[businessId, messageText, containsList, notContainsList]`
- `hermes.gupshup.interaction.template.button.v1` — WhatsApp template button click · `[buttonId, gupshupAppId]`
- `hermes.gupshup.interaction.message.received.v1` — WhatsApp message · `[gupshupAppId, text, contentType, containsList, notContainsList]`
- `hermes.gupshup.message.status.v1` — WhatsApp delivery status · `[status, gupshupAppId]`

## Notes

- `flows_step_triggers_set` edits each side independently: omit `triggerStart` or `triggerExit` to keep that side untouched; send `[]` to clear it; the array you send replaces only that side. So a "change the entry trigger" request needs `triggerStart` only — the exit stays intact.
- Use the exact `eventName` from this catalog; an unknown name silently never fires.
- Text-keyword channel triggers use `containsList` / `notContainsList` (arrays via `valueJson`) for "message contains / does not contain" matching.
