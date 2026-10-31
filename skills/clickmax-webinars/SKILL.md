---
name: clickmax-webinars
description: Use when the user wants to create or configure a Clickmax webinar — titles, slug, presenter/host, schedule and sessions, offer cards, room settings (simulated audience, sales, chat, reactions, appearance, ended page), automations — or publish it.
---

## When this applies

Building or changing a webinar (recorded/evergreen or planned live) end to end, except video and live control.

Not this skill:

- uploading/choosing the video, going live, control room → the app (webinar builder)
- the checkout the offer links to → `clickmax-offers` / `clickmax-products`
- generic automations not tied to a webinar → `clickmax-flows`

## Key assumptions

- Lifecycle: `draft` → `published` (registration page + room open) → `archived`. Changing status is `webinars_set_status`, never `webinars_update`.
- Schedule `mode`: `scheduled` = fixed dates (create sessions), `just_in_time` = one session per registrant after `jitDelayMinutes`, `checkout_triggered` = session after a purchase of `checkoutTriggerProductId`.
- Sessions: `startsAt` must be in the future (a past time is rejected — it would go on air at once). Only `timing: waiting` can be moved/deleted. Series = N independent sessions (max 52), rerun skips existing times.
- Offer card: shows at `offerStartSeconds` of the video, button → `checkoutUrl`. Prices are display text; the charge is in the linked checkout. Max 20 offers.
- Room sections (`webinars_room_update`): `audience` (simulated viewer count), `sales` (simulated purchase toasts), `chat` (scripted messages at video seconds), `reactions`, `appearance` (theme, brand, SEO), `endedPage` (replay, offer, community, support after the session). Partial update; arrays replace the whole list. `null` from `webinars_room_get` = defaults.
- Automations = managed flows on real registrants. Triggers: `registered`, `joined`, `joined_late`, `progress` (+`percent`), `reached_pitch`, `reached_offer`, `offer_clicked`, `session_start` (+`delayMinutes` + `sessionOffset` before/after). Channels: `internal`, `email`, `sms`, `whatsapp` (approved template), `tag`. Saving or turning one on asks the user for approval; `webinars_automations_pause` stops it at once, no approval.
- Hosts are reusable presenter profiles linked by `hostId`.

## Thought process

1. Existing or new? `webinars_list` (`search`) first.
2. Collect only what is missing: public title, when (fixed dates vs evergreen), what is sold (checkout URL or product), presenter.
3. Simulated audience/sales/chat look real to viewers: configure them only as the user asked; never invent buyer names or testimonials on your own initiative.
4. Publish only when the user says so.

## Execute guide

- New: `mcp__plugin_clickmax_clickmax__webinars_check_slug` → `mcp__plugin_clickmax_clickmax__webinars_create` → `mcp__plugin_clickmax_clickmax__webinars_schedule_upsert({ id, mode })` → for `scheduled`: `mcp__plugin_clickmax_clickmax__webinars_sessions_create_series({ id, startDate, time, weekdays, count, timezone })` or `mcp__plugin_clickmax_clickmax__webinars_sessions_create` → `mcp__plugin_clickmax_clickmax__webinars_offers_create` → `mcp__plugin_clickmax_clickmax__webinars_room_update` with the sections asked. Then tell the user to pick the video in the app.
- Presenter: `mcp__plugin_clickmax_clickmax__webinars_hosts_list` → reuse or `mcp__plugin_clickmax_clickmax__webinars_hosts_create` → `mcp__plugin_clickmax_clickmax__webinars_update({ id, hostId })`.
- Reminders: `mcp__plugin_clickmax_clickmax__webinars_automations_upsert({ id, config: { event: 'session_start', delayMinutes: 40, sessionOffset: 'before', channel: 'whatsapp', gupshupTemplateId } })`; edit with the `recipeKey` from `mcp__plugin_clickmax_clickmax__webinars_automations_list`. On/off: `mcp__plugin_clickmax_clickmax__webinars_automations_pause` / `mcp__plugin_clickmax_clickmax__webinars_automations_activate`.
- Edit room: `mcp__plugin_clickmax_clickmax__webinars_room_get({ id, sections })` → change → `mcp__plugin_clickmax_clickmax__webinars_room_update`. Sections save one after another: on error, the message lists what was already saved — resend only the rest.
- Publish: confirm slug, sessions/schedule and offers → `mcp__plugin_clickmax_clickmax__webinars_set_status({ id, status: 'published' })`.

## Report

- Summarize as the viewer will see it: title, link slug, next sessions (with timezone), offers and when they appear, what is simulated.
- List what is still missing to publish (video, sessions, offer).

## Warnings

- Deleting/moving a session affects its registrants — check `registrationsCount` and say it.
- A WhatsApp automation needs an approved template; e-mail needs a sender signature.
- `chat.messages`, `sales.purchases`, `reactions.timeline` replace the whole list — read first when adding to them.

## Anti-patterns

- Publishing right after create without sessions, offer or video.
- Creating sessions one by one when the user described a recurring pattern (use the series).
- Putting the offer price only in the card and assuming it changes the checkout price.

---

Clickmax skill revision: `862a8ca93146`
