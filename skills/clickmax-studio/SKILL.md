---
name: clickmax-studio
description: Use when the user wants to plan, write, review, approve, produce, schedule or measure content in the Cx Studio pauta — editorial lines, the idea inbox, scripts, the video of a script, or the monthly results of a brand.
---

## When this applies

Use this skill for the Cx Studio pauta: the production board that takes a content idea to a published post, per brand (the client) and editorial line.

Not this skill:

- the brand voice itself (tone, words, samples) -> `brand_get` / `brand_save_draft`
- a social post that is not born from a script -> `clickmax-social`
- making the video by hand in the Cx Studio desktop app -> the person does it there

## Key assumptions

- BRAND = the client. LINE (editorial line, `studio_lines_*`) belongs to one brand and sets who speaks, instructions, the single deliverable (`outputType`: reel | carousel | image), video defaults, the weekly `cadence` slots, the review `gates` and the social `accounts`. Every script lives in one line.
- INBOX idea (`studio_ideas_*`) = no line and no date yet. Promoting it creates a script.
- SCRIPT status: `idea` -> `writing` -> `review` ⇄ `changes` -> `approved` -> `production` -> `produced` -> `scheduled` -> `published`; `discarded`. `published` comes from the social post.
- Content is versioned. Editing a script in `review` / `approved` sends it back to `writing`. From `production` on the copy is locked. Every edit needs the `revision` read last (409 = someone saved first: read again).
- Gates per line: `copyReview` / `visualReview` = `approval` (someone with "Gerenciar" approves) or `auto` (moves on alone). Approving, asking for changes, approving the video and redo need "Gerenciar" on Pauta do Studio (403 otherwise).
- The video is rendered by the Cx Studio DESKTOP app of the person in `assignedUserId`, only while it is open. The agent queues it; it cannot render.
- AI tools (`studio_scripts_plan`, `studio_scripts_write`, `studio_scripts_hooks`, `studio_ideas_suggest`) spend workspace AI credits on every call.
- Metrics = the social post readings at 24h and 7d, summed across networks.

## Thought process

1. Read before writing: `mcp__plugin_clickmax_clickmax__studio_lines_list` for line ids (and the brand of each). No line yet -> create one with the user (brand, name, deliverable).
2. Plan proposes, the person picks: `mcp__plugin_clickmax_clickmax__studio_scripts_plan` saves nothing; show the ideas, then create only the chosen ones.
3. Approvals are the person's call: never approve copy or video on your own judgment; ask, and only then transition/approve.
4. Prefer reversible: discard (`studio_scripts_transition` to `discarded`, `studio_ideas_update` `status: discarded`) or archive a line over deleting.

## Execute guide

- Monthly pauta: `mcp__plugin_clickmax_clickmax__studio_scripts_plan` (`studioProjectId`, `count`, `mix` of kinds, `from`/`to`, `notes`, `usePerformance`) -> show title, kind, date and `why` -> `mcp__plugin_clickmax_clickmax__studio_scripts_create_bulk` with the picked ones, each with its `idea`.
- Capture an idea: `mcp__plugin_clickmax_clickmax__studio_ideas_create`. Triage the inbox: `mcp__plugin_clickmax_clickmax__studio_ideas_suggest` -> read `suggestedProjectId` / `suggestedDiscard` -> confirm -> `mcp__plugin_clickmax_clickmax__studio_ideas_promote` (line, date, kind per idea) or discard.
- Write: `mcp__plugin_clickmax_clickmax__studio_scripts_write` (AI, in the brand voice) or write it yourself and save with `mcp__plugin_clickmax_clickmax__studio_scripts_update` (`aiAuthored: true`). Better opening: `mcp__plugin_clickmax_clickmax__studio_scripts_hooks`, then update the first scene.
- Copy review: `mcp__plugin_clickmax_clickmax__studio_scripts_transition` `to: review` (+ reviewer from `workspace_members_list`). Changes: `mcp__plugin_clickmax_clickmax__studio_scripts_comments_add` first, then `to: changes`. Approve after the user says yes: `to: approved` (+ who produces).
- Video: `mcp__plugin_clickmax_clickmax__studio_scripts_produce` (`assignedUserId`, `options`) -> later `mcp__plugin_clickmax_clickmax__studio_scripts_list` (`productionState`: `queued` | `rendering`) -> `produced` -> `mcp__plugin_clickmax_clickmax__studio_scripts_video_url` for the person to watch -> `mcp__plugin_clickmax_clickmax__studio_scripts_video_approve`, or `mcp__plugin_clickmax_clickmax__studio_scripts_video_request_changes` (video only) / `mcp__plugin_clickmax_clickmax__studio_scripts_redo` (copy and video).
- Schedule: `mcp__plugin_clickmax_clickmax__studio_lines_next_slot` -> `mcp__plugin_clickmax_clickmax__social_posts_create` (draft, the script video = Drive file `videoDriveFileId`, the script caption, the line accounts) -> confirm -> `mcp__plugin_clickmax_clickmax__social_posts_schedule` -> `mcp__plugin_clickmax_clickmax__studio_scripts_link_social_post`.
- Results: `mcp__plugin_clickmax_clickmax__studio_scripts_report` (`brandId`, `month`, optional line). Recreate a hit: `mcp__plugin_clickmax_clickmax__studio_scripts_create_similar` then write it.

## Report

- Name scripts by title, with line, planned date and status.
- After planning: one line per idea (date, kind, title, why).
- Results: planned / approved / scheduled / published, reach and saves; top and bottom of the ranking with `vsAverage` (1.5 = 50% above the brand average).

## Warnings

- `studio_lines_update` replaces `defaults`, `cadence`, `gates` and `accounts` as a whole: send the full value.
- A video in `production` only moves when the assigned person opens the Cx Studio app.
- Scheduling puts content on a PUBLIC profile: confirm caption, video, accounts and time first.

## Anti-patterns

- Creating every idea from `studio_scripts_plan` without the person choosing.
- Approving copy or video without an explicit yes.
- Asking for changes without a comment saying what to change.
- Promising the video is ready right after `studio_scripts_produce`.

---

Clickmax skill revision: `2d3df031680f`
