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

- BRAND = the client. LINE (editorial line, `studio_lines_*`) belongs to one brand and sets who speaks, instructions, the single deliverable (`outputType`: reel | carousel | image), video defaults, the weekly `cadence` slots, the review `gates` (both also `agent`), `automation` (what runs alone) and the social `accounts`. Every script lives in one line.
- LINE `defaults.voice` = fixed narration voice of every video of the line: `clickmax` (AI voice catalog or a voice the workspace trained) or `elevenlabs` (`connectorId` of the ElevenLabs key in Studio connectors + `voice_id`). Absent = the agent picks per video.
- LINE `automation` (all off by default; runs every 10 min): `autoPlan` → content plan stage: the ideas agent (`ideaAgentId`, empty = Clickmax AI) proposes ideas with `ideaPrompt` from the line RSS/Atom sources (`studio_lines_feeds_*`, ≤ 10 per line; adding needs "Gerenciar") and the published results, up to `maxOpenIdeas` (default 10) open ideas = line inbox + plan ideas without text, at most every 6 h; `gates.ideaReview` `approval` (default) → ideas wait in the inbox with the line suggested (`sourceKind: agent`, `authorAgentName`, `hook`), `auto` → straight to the plan | `autoProduce` → keeps `minBuffer` (default 5) videos in `production`+`produced` by sending the approved scripts an agent ranks by past performance | `evaluate` → weak topic back to `writing` with the reason | `producer` `person` (+`producerUserId`) or `machines` (rotation between online `studio_lines_machines`) | `autoSchedule` → each rendered video in the next free cadence slot | `reviewNotify` approver/people/none. `visualReview: agent` → agent approves, asks video changes (≤ `reviewMaxRounds`, default 2) or redoes copy, else hands to a person. History marks these `auto`.
- LINE stage owners = the workspace AI agents (`studio_lines_agents`; BYOK, no Clickmax credits; a paused agent or one without a working key fails its first call and the script goes to a person with the reason in the history): `copyAgentId` writes (`studio_scripts_write`) and rewrites rejected copy | `copyReview: agent` + `copyReviewAgentId` + `copyReviewPrompt` → approves or rejects with scene notes; rejected → copy agent rewrites and it is reviewed again, ≤ `copyReviewMaxRounds` (default 2) rejections, then a person reviews (no copy agent → back to the writer in `changes`) | `evaluateAgentId` + `evaluatePrompt` picks/screens for production | `reviewAgentId` does the final review (empty = Clickmax AI). Agent paused/without key → reviews go to a person, and `studio_scripts_write` fails with 422 until the agent is fixed or removed from the line; the production pick falls back to pauta order without screening. Taking a script or video away from an agent ("Revisar eu mesmo") and handing it back are app-only, not MCP tools.
- INBOX idea (`studio_ideas_*`) = no line and no date yet. Promoting it creates a script.
- SCRIPT status: `idea` -> `writing` -> `review` ⇄ `changes` -> `approved` -> `production` -> `produced` -> `scheduled` -> `published`; `discarded`. `published` comes from the social post. Board stages inside `produced`: Revisão final (no `videoApprovedAt`) -> Aprovado (approved, no `videoDriveFileId` yet) -> Renderizado.
- Content is versioned. Editing a script in `review` / `approved` sends it back to `writing`. From `production` on the copy is locked. Every edit needs the `revision` read last (409 = someone saved first: read again).
- Gates per line: `copyReview` / `visualReview` = `approval` (someone with "Gerenciar" approves), `auto` (moves on alone) or `agent` (an AI agent reviews). Approving, asking for changes, approving the video and redo need "Gerenciar" on Pauta do Studio (403 otherwise).
- Final review plays the video in the public-link player (`previewShareId`), WITHOUT rendering. The MP4 is rendered only after `studio_scripts_video_approve`, by the Cx Studio DESKTOP app of the person in `assignedUserId`, only while it is open. The agent queues it; it cannot render. Scheduling needs the MP4 (`videoDriveFileId`).
- AI tools (`studio_scripts_plan`, `studio_scripts_write`, `studio_scripts_hooks`, `studio_ideas_suggest`) spend workspace AI credits on every call, except `studio_scripts_write` on a line with `copyAgentId`: then the line's AI agent writes with the account's own provider key.
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
- Video: `mcp__plugin_clickmax_clickmax__studio_scripts_produce` (`assignedUserId` or `machines: true`, `options`) -> later `mcp__plugin_clickmax_clickmax__studio_scripts_list` (`productionState`: `queued` + `queuePosition` | `rendering`; jump the queue: `mcp__plugin_clickmax_clickmax__studio_scripts_queue_bump`) -> `produced` -> `/public-videos/<previewShareId>` on the API gateway for the person to watch (`mcp__plugin_clickmax_clickmax__studio_scripts_video_url` only after the render) -> `mcp__plugin_clickmax_clickmax__studio_scripts_video_approve`, or `mcp__plugin_clickmax_clickmax__studio_scripts_video_request_changes` (video only) / `mcp__plugin_clickmax_clickmax__studio_scripts_redo` (copy and video); instruction optional on both.
- Schedule: `mcp__plugin_clickmax_clickmax__studio_lines_next_slot` -> `mcp__plugin_clickmax_clickmax__social_posts_create` (draft, the script video = Drive file `videoDriveFileId`, the script caption, the line accounts) -> confirm -> `mcp__plugin_clickmax_clickmax__social_posts_schedule` -> `mcp__plugin_clickmax_clickmax__studio_scripts_link_social_post`.
- Results: `mcp__plugin_clickmax_clickmax__studio_scripts_report` (`brandId`, `month`, optional line). Recreate a hit (`performance.tier` = `outlier`): `mcp__plugin_clickmax_clickmax__studio_scripts_create_similar` then write it.

## Report

- Name scripts by title, with line, planned date and status.
- After planning: one line per idea (date, kind, title, why).
- Results: planned / approved / scheduled / published, `tiers` (outlier/average/flop/smallBase), reach and saves; ranking is by `performance.tier` (CSS = comments+shares+saves vs the account × network 180-day average; `tier` null = account under 30 measured posts), `vsAverage` = reach vs the brand month average.

## Warnings

- `studio_lines_update` replaces `defaults`, `cadence`, `gates`, `rules`, `automation` and `accounts` as a whole: send the full value. `automation` or `visualReview: agent` → needs approve permission and makes the line approve/schedule alone: confirm with the user.
- Cadence advice: `mcp__plugin_clickmax_clickmax__studio_lines_recommended_slots` (best weekday × time of the line, empty = too few published posts). Carousel/image scripts: `mcp__plugin_clickmax_clickmax__studio_scripts_images_list` instead of the video link. Templates a flow can fill per contact: `mcp__plugin_clickmax_clickmax__studio_templates_list`.
- Line `rules` = guard rail (`avoid`, `require`). `studio_scripts_get` → `guardRail.violations` (term + scene, `beatIndex` null = caption): fix before review; with a violation an `auto` copy gate does not approve.
- A video in `production` only moves when the assigned person opens the Cx Studio app.
- Scheduling puts content on a PUBLIC profile: confirm caption, video, accounts and time first.

## Anti-patterns

- Creating every idea from `studio_scripts_plan` without the person choosing.
- Approving copy or video without an explicit yes.
- Asking for changes without a comment saying what to change.
- Promising the video is ready right after `studio_scripts_produce`.

---

Clickmax skill revision: `8720ac6b3017`
