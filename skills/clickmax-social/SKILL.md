---
name: clickmax-social
description: Use when the user wants to write, approve, schedule, publish or retry social posts, answer post comments, manage queue, templates, goals or social reports in Clickmax.
---

## When this applies

Use this skill for the Redes sociais module: publishing accounts, posts and their per-network targets, approval, internal notes, the comments inbox, the posting queue, templates/groups, goals, the calendar, results and e-mailed reports.

Not this skill:

- Instagram DMs as conversations -> messaging / live chat
- paid ads in Tráfego -> not covered here
- social numbers inside an Analytics dashboard -> `clickmax-insights-dashboards` (entity `social`)

## Key assumptions

- POST = one caption + media. TARGET = the post on ONE account (post × account): own status, time, caption override, network options, post type, permalink, error and metrics. Statuses act on targets.
- target status: `draft` -> `scheduled` -> `publishing` -> `published` | `failed`; `cancelled`. Only `draft`, `scheduled` and `failed` change (time, options, cancel). A cancelled target never comes back: duplicate the post.
- networks: instagram | facebook | tiktok | youtube. YouTube is beta and may be off for the account. TikTok posts always go to the creator's DRAFTS in the TikTok app (the person finishes there); they never get metrics.
- accounts = the Meta Pages (with linked Instagram) and Pinterest accounts connected in Traffic -> Connections. Connecting, reconnecting and disconnecting happen ONLY there, in the person's browser; no tool does it. `blockedReason` on `mcp__plugin_clickmax_clickmax__social_accounts_list` = nothing publishes on it until the person fixes it there.
- paused account = its scheduled targets wait; resuming publishes what came due during the pause within ~1 minute.
- approval: `pending_approval` = waiting for someone with manage; approving SCHEDULES the targets (goes live at the time).
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

- Connect / reconnect an account: tell the person to open Traffic -> Connections in Clickmax; then `mcp__plugin_clickmax_clickmax__social_accounts_list` again.
- Pinterest target: board from `mcp__plugin_clickmax_clickmax__social_accounts_boards_list`.
- Approval flow: `mcp__plugin_clickmax_clickmax__social_posts_request_approval` (`approverId` from `mcp__plugin_clickmax_clickmax__social_posts_approvers_list`, optional `scheduledAt`) -> approver: `mcp__plugin_clickmax_clickmax__social_posts_approve` after confirmation, or `mcp__plugin_clickmax_clickmax__social_posts_request_changes` with a `comment`. Team-only remarks: `mcp__plugin_clickmax_clickmax__social_posts_notes_*`.
- Reuse: start a post from `mcp__plugin_clickmax_clickmax__social_templates_list`; accounts of a group from `mcp__plugin_clickmax_clickmax__social_account_groups_list`; hashtags from `mcp__plugin_clickmax_clickmax__social_hashtag_groups_list`; campaign names from `mcp__plugin_clickmax_clickmax__social_campaigns_list`.
- Comments: `mcp__plugin_clickmax_clickmax__social_comments_list` (`status`, `accountId`, page with `cursor`) -> draft the answer (or use `mcp__plugin_clickmax_clickmax__social_saved_replies_list`) -> confirm -> `mcp__plugin_clickmax_clickmax__social_comments_reply`. Handled without answer -> `mcp__plugin_clickmax_clickmax__social_comments_update` `resolved`.
- Pause / resume an account after confirmation: `mcp__plugin_clickmax_clickmax__social_accounts_pause` / `mcp__plugin_clickmax_clickmax__social_accounts_resume` (check `mcp__plugin_clickmax_clickmax__social_calendar` for overdue targets before resuming).
- Write a caption: `mcp__plugin_clickmax_clickmax__social_caption_generate` (`generate` from a topic, `rewrite` / `shorten` an existing text; `field: title` for a YouTube title). Mention it uses AI credits.
- New post: `mcp__plugin_clickmax_clickmax__social_posts_create` with `mode: draft`, one target per account (`accountId`, `postType`, `options.network` = that account's network), optional `defaultScheduledAt` or per-target `scheduledAt`. "Next free queue slot": `mcp__plugin_clickmax_clickmax__social_queue_next_free_slot` -> its `scheduledAt` on the target -> schedule after confirmation.
- Schedule after confirmation: `mcp__plugin_clickmax_clickmax__social_posts_schedule` (all drafts, or `targetIds`). Publish now after confirmation: `mcp__plugin_clickmax_clickmax__social_posts_publish_now`, then `mcp__plugin_clickmax_clickmax__social_posts_get` a moment later to report `published` / `failed`.
- Change one network only, after confirmation: `mcp__plugin_clickmax_clickmax__social_targets_update` (caption override, options, post type, `scheduledAt` = reschedule).
- Change the post: `mcp__plugin_clickmax_clickmax__social_posts_update` (caption, default time, `addTargets`, `removeTargetIds`).
- Media: `mcp__plugin_clickmax_clickmax__social_posts_set_media` is a FULL REPLACE in carousel order; send the whole list.
- Failed target: read `errorCode` in `mcp__plugin_clickmax_clickmax__social_posts_get`. `auth_expired` -> reconnect first; `rate_limited` / `provider_unavailable` -> retry later; `media_invalid` / `policy_rejected` -> change media or caption first; then `mcp__plugin_clickmax_clickmax__social_targets_retry` after confirmation.
- Queue: `mcp__plugin_clickmax_clickmax__social_queue_slots_list` -> `mcp__plugin_clickmax_clickmax__social_queue_slots_replace` with the FULL list (keep the slots the person did not mention). `mcp__plugin_clickmax_clickmax__social_queue_next_free_slot` to propose a time.
- What is planned: `mcp__plugin_clickmax_clickmax__social_calendar` (`from`/`to`; `truncated: true` -> shorter period). Find a post: `mcp__plugin_clickmax_clickmax__social_posts_list` (paginated: read every page before saying a post does not exist).
- Results: `mcp__plugin_clickmax_clickmax__social_insights` (`from`, `to`, `timezone`) for totals, per day, per network, best weekday × hour and top posts. Vs previous period / per campaign: `mcp__plugin_clickmax_clickmax__social_report`. E-mailed reports: `mcp__plugin_clickmax_clickmax__social_report_schedules_*` (confirm recipients and frequency).
- Goals: `mcp__plugin_clickmax_clickmax__social_goals_list`; `mcp__plugin_clickmax_clickmax__social_goals_replace` = FULL list.

## Report

- Name the post by the start of its caption, then each network with its status and local time.
- After scheduling: list each target with network, account and local date/time.
- After publishing: give the permalink of each published target and the error of each failed one.
- Results: views and engagement (likes + comments + shares + saves), per network, with the period; say that the numbers are the latest reading.

## Warnings

- Scheduling, publishing now, approving, retrying, editing a target and replying to a comment put content on a PUBLIC profile: confirm text, media, networks and time first.
- Deleting or cancelling in Clickmax does NOT remove a post already published on the network.
- `social_queue_slots_replace`, `social_posts_set_media` and `social_goals_replace` replace the whole list.

## Anti-patterns

- Scheduling or publishing straight from a request without showing the draft.
- Trying to connect, log in or "finish" the OAuth for the person.
- Promising TikTok metrics or a TikTok post going live automatically.
- Recreating a post to change one network's time (use `social_targets_update`).
- Uploading through `social_media_upload_url` without being able to PUT the file.

---

Clickmax skill revision: `94984d83e296`
