---
name: clickmax-broadcasts
description: Use when the user wants to build, review, schedule, send, pause, resume, cancel or reuse a one-time email or WhatsApp campaign (disparo/broadcast) to a list or segment, or manage email templates.
---

## When this applies

A campaign (disparo) = one message sent once, now or at a date, to a frozen audience (lists / segment / picked contacts). Also the email template library (own + Clickmax gallery).

Not this skill:

- message that reacts to an event, repeats, or is one step of a sequence → `clickmax-flows`
- campaign performance only → `clickmax-analytics`; multi-week plan → `clickmax-campaign-planning`
- SMS / voice campaign creation → screen only (`broadcasts_get`/`send_now`/`schedule`/`pause`/`cancel` still work on an existing one)

## Key assumptions

- Lifecycle: `draft` → (`scheduled`) → `queued` → `sending` → `sent` | `failed` | `cancelled`. Only `draft`/`scheduled` are editable.
- Audience is FROZEN when the draft is created/updated with `audience`. A list/segment that grows later is NOT picked up — re-pass `audience` on `broadcasts_update` right before scheduling a send that is days away. Up to 5 000 `leadIds` and 50 `listIds` per audience; more contacts → a list or segment.
- Segment = its synced list: `segments_get` → `listId` → `audience.listIds`.
- Email needs a CONFIRMED sender (`email_sender_signatures_list`, `confirmed: true`) to schedule/send; a draft may skip it.
- WhatsApp campaign = approved template only (free text has no 24h window here). `paramMapping` all or none, one token per `positionParams`: `{{name}}`, `{{email}}`, `{{custom.<key>}}`.
- Sending costs credits; schedule/send fails upfront (402) without balance, or 409 when the WhatsApp number is blocked or its use in campaigns is off.
- Pause works only on the new send engine (409 otherwise → cancel is the only stop). Cancel is final.
- Email templates: gallery (`isOwn: false`) is read-only → `email_templates_duplicate` to customize. Category `scheduling` = layout with the `{{conteudo}}` slot, not a campaign body.

## Thought process

1. Audience first: resolve list/segment names to ids; size it with `broadcasts_reach_estimate` (per-list, overlaps double-count).
2. Content: user's text → email-safe HTML (table layout, inline styles, sender logo/address from `whoami`, never Clickmax branding); or start from `email_templates_get` html; WhatsApp → pick an approved template.
3. `broadcasts_create_draft` → report `recipientCount` (exact unique audience).
4. Review with the user (subject, sender, audience size, time, credits). Change via `broadcasts_update`.
5. Only on explicit confirmation → `broadcasts_schedule` (date) or `broadcasts_send_now`. Both pause for approval.

## Execute guide

- New email campaign: `mcp__plugin_clickmax_clickmax__email_sender_signatures_list` → `mcp__plugin_clickmax_clickmax__broadcasts_create_draft` `{name, email:{subject, html, senderSignatureId}, audience:{listIds}}`.
- From a template: `mcp__plugin_clickmax_clickmax__email_templates_list` (`search`/`category`) → `mcp__plugin_clickmax_clickmax__email_templates_get` → its `html` into `email.html`.
- WhatsApp: `mcp__plugin_clickmax_clickmax__gupshup_templates_list` `{status:'approved'}` → `mcp__plugin_clickmax_clickmax__broadcasts_create_draft` `{name, whatsapp:{templateId, paramMapping}, audience}`.
- Reuse/resend a past campaign: `mcp__plugin_clickmax_clickmax__broadcasts_duplicate` → `mcp__plugin_clickmax_clickmax__broadcasts_update` (new audience/name) → schedule/send.
- Unschedule: `mcp__plugin_clickmax_clickmax__broadcasts_update` (any field or none) returns it to draft.
- Edit content: `email.html` replaces only the HTML body (attachments and text part stay); a new `whatsapp` template keeps the saved media — if the new template needs other media, the user changes it in the app.
- Running campaign: `mcp__plugin_clickmax_clickmax__broadcasts_get` (progress, `paused`, `dispatchError`) → `mcp__plugin_clickmax_clickmax__broadcasts_pause` / `mcp__plugin_clickmax_clickmax__broadcasts_resume` / `mcp__plugin_clickmax_clickmax__broadcasts_cancel`.
- Save a design for reuse: `mcp__plugin_clickmax_clickmax__email_templates_create` with a category. Create/update/duplicate return `htmlLength`, not the HTML — read it back with `mcp__plugin_clickmax_clickmax__email_templates_get` only if needed.

## Report

- After the draft: name, channel, audience size, sender/template, pacing, status "rascunho — nada foi enviado", then ask to schedule or send.
- After schedule/send: date/time in the user's timezone, recipients, and how to stop it (pause/cancel/unschedule).
- When offering to open it, use `action="open-page"`; the user reviews and sends from the campaign screen too.

## Warnings

- Never schedule/send without the user confirming content, audience and time in this conversation.
- `broadcasts_update` on a scheduled campaign unschedules it — say so and reschedule if that was not the intent.
- Never fabricate a sender signature, template id or list id.
- A `failed` campaign shows `dispatchError` — report it; do not retry blindly with `send_now`.

## Anti-patterns

- Summing per-list reach as the audience size instead of the draft's `recipientCount`.
- Editing a gallery template (fails) instead of duplicating it.

---

Clickmax skill revision: `862a8ca93146`
