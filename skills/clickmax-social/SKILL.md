---
name: clickmax-social
description: Use when the user wants to connect Instagram, Facebook, TikTok or YouTube accounts, write, schedule, publish, reschedule, cancel or retry social media posts, manage the posting queue, or read social media results in Clickmax.
---

## When this applies

Use this skill for the Redes sociais module: connected accounts, posts and their per-network targets, the posting queue, the calendar and the results of published posts.

Not this skill:

- Instagram DMs or comments as conversations -> messaging / live chat
- paid ads in Tráfego -> not covered here
- social numbers inside an Analytics dashboard -> `clickmax-insights-dashboards` (entity `social`)

## Key assumptions

- POST = one caption + media. TARGET = the post on ONE account (post × account): own status, time, caption override, network options, post type, permalink, error and metrics. Statuses act on targets.
- target status: `draft` -> `scheduled` -> `publishing` -> `published` | `failed`; `cancelled`. Only `draft`, `scheduled` and `failed` change (time, options, cancel). A cancelled target never comes back: duplicate the post.
- networks: instagram | facebook | tiktok | youtube. YouTube is beta and may be off for the account. TikTok posts always go to the creator's DRAFTS in the TikTok app (the person finishes there); they never get metrics.
- account `status = needs_reconnect` -> nothing publishes on it until `mcp__plugin_clickmax_clickmax__social_accounts_reconnect`.
- connecting is OAuth in the PERSON's browser. The agent cannot log in: it hands over the `url`, then polls the result by `nonce` (session valid 30 minutes).
- every write answers the post detail with `issues` = network rules that block scheduling (media required, caption too long, wrong media type…). A rule failure on create/update is a 422 and writes nothing.
- times are instants (ISO with offset). Never in the past (400). `timezone` = the zone the person works in (IANA).
- media: Drive files (`source: drive` + `driveFileId`, find them with `mcp__plugin_clickmax_clickmax__drive_search`) or uploads. An upload needs an HTTP PUT of the file bytes to a presigned URL; without file access, use Drive files.
- metrics = the latest reading of each target, read every 6 hours for 30 days after publishing. No daily history.
- AI captions spend workspace AI credits on every call.
- permissions: read / create / update / delete on Redes sociais; editing the queue and connecting accounts is "Gerenciar". A 403 means the person lacks it; a `MODULE_DISABLED` answer means the module is off for the account -> tell the user to contact support.

## Thought process

1. Read before writing: `mcp__plugin_clickmax_clickmax__social_accounts_list` for account ids and their status.
2. Draft first. Create and add targets only with `mode: draft` (the tools accept nothing else), show the person the caption, networks, media and time, then schedule or publish only after an explicit yes. Scheduling and publishing go to the person's PUBLIC profile.
3. Fix `issues` before scheduling: they name the target and the rule.
4. Prefer reversible steps: reschedule a target instead of cancel + recreate; cancel instead of delete when the person may want the post back as a copy.

## Execute guide

- Connect an account: `mcp__plugin_clickmax_clickmax__social_accounts_connect` (`network`) -> give the `url` to the person ("abra este link e faça login") -> `mcp__plugin_clickmax_clickmax__social_accounts_connect_result` with the `nonce` until it leaves `pending`. `select_pages` (Facebook) -> `mcp__plugin_clickmax_clickmax__social_accounts_connect_pages` -> ask which pages -> `mcp__plugin_clickmax_clickmax__social_accounts_connect_select_pages`. `failed` -> explain `errorCode`.
- Reconnect: `mcp__plugin_clickmax_clickmax__social_accounts_reconnect` (account `id`) -> same `url` + `nonce` flow.
- Write a caption: `mcp__plugin_clickmax_clickmax__social_caption_generate` (`generate` from a topic, `rewrite` / `shorten` an existing text; `field: title` for a YouTube title). Mention it uses AI credits.
- New post: `mcp__plugin_clickmax_clickmax__social_posts_create` with `mode: draft`, one target per account (`accountId`, `postType`, `options.network` = that account's network), optional `defaultScheduledAt` or per-target `scheduledAt`. "Next free queue slot": `mcp__plugin_clickmax_clickmax__social_queue_next_free_slot` -> its `scheduledAt` on the target -> schedule after confirmation.
- Schedule after confirmation: `mcp__plugin_clickmax_clickmax__social_posts_schedule` (all drafts, or `targetIds`). Publish now after confirmation: `mcp__plugin_clickmax_clickmax__social_posts_publish_now`, then `mcp__plugin_clickmax_clickmax__social_posts_get` a moment later to report `published` / `failed`.
- Change one network only, after confirmation: `mcp__plugin_clickmax_clickmax__social_targets_update` (caption override, options, post type, `scheduledAt` = reschedule).
- Change the post: `mcp__plugin_clickmax_clickmax__social_posts_update` (caption, default time, `addTargets`, `removeTargetIds`).
- Media: `mcp__plugin_clickmax_clickmax__social_posts_set_media` is a FULL REPLACE in carousel order; send the whole list.
- Failed target: read `errorCode` in `mcp__plugin_clickmax_clickmax__social_posts_get`. `auth_expired` -> reconnect first; `rate_limited` / `provider_unavailable` -> retry later; `media_invalid` / `policy_rejected` -> change media or caption first; then `mcp__plugin_clickmax_clickmax__social_targets_retry` after confirmation.
- Queue: `mcp__plugin_clickmax_clickmax__social_queue_slots_list` -> `mcp__plugin_clickmax_clickmax__social_queue_slots_replace` with the FULL list (keep the slots the person did not mention). `mcp__plugin_clickmax_clickmax__social_queue_next_free_slot` to propose a time.
- What is planned: `mcp__plugin_clickmax_clickmax__social_calendar` (`from`/`to`; `truncated: true` -> shorter period). Find a post: `mcp__plugin_clickmax_clickmax__social_posts_list` (paginated: read every page before saying a post does not exist).
- Results: `mcp__plugin_clickmax_clickmax__social_insights` (`from`, `to`, `timezone`) for totals, per day, per network, best weekday × hour and top posts.

## Report

- Name the post by the start of its caption, then each network with its status and local time.
- After scheduling: list each target with network, account and local date/time.
- After publishing: give the permalink of each published target and the error of each failed one.
- Results: views and engagement (likes + comments + shares + saves), per network, with the period; say that the numbers are the latest reading.

## Warnings

- Scheduling, publishing now, retrying and editing a target put content on a PUBLIC profile: confirm text, media, networks and time first.
- Disconnecting an account cancels every target of it that is not published yet.
- Deleting or cancelling in Clickmax does NOT remove a post already published on the network.
- `social_queue_slots_replace` and `social_posts_set_media` replace the whole list.

## Anti-patterns

- Scheduling or publishing straight from a request without showing the draft.
- Trying to log in on the network or "finish" the OAuth for the person.
- Promising TikTok metrics or a TikTok post going live automatically.
- Recreating a post to change one network's time (use `social_targets_update`).
- Uploading through `social_media_upload_url` without being able to PUT the file.

---

Clickmax skill revision: `cee6a983010c`
