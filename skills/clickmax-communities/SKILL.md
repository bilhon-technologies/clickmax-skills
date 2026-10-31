---
name: clickmax-communities
description: Use when the user wants to create or set up a community of the Clickmax members area — its categories and channels, look, moderation limits, events — or invite, promote, block or remove community members.
---

## When this applies

Structure and people of ONE community: create it, organize categories → channels, look and moderation, schedule events, manage individual members.

Not this skill:

- enrolling a CRM list/segment/filter in a community → `clickmax-contacts-member-access` (`leads_bulk_member_access`)
- portals, classrooms, courses → `clickmax-members-area`

## Key assumptions

- Structure = community → categories (sidebar groups) → channels (where posts live). A community without channels has nowhere to post.
- Channel `publishRoles`/`commentRoles` default = every role; the owner always can. `viewMode: curso` = course showcase, `externalUrl` = link channel.
- Roles: `member`, `moderator`, `mentor`, `instructor`, `owner` (= "Administrador", full power).
- Member status: `active`, `muted` (reads, cannot post/comment), `blocked` (cannot enter). Reversible; prefer over remove.
- `memberId` in member tools = row id from `communities_members_list`, not the student id.
- Invite e-mails the person unless `sendAccessEmail: false`.
- Images (cover, logos, event cover) must be existing https URLs; uploads happen in the app.

## Thought process

1. Resolve the community: `communities_list` (`search`); create only when the user asked for a new one.
2. "Community for my course with X and Y" → X and Y are usually channels; group them under one or two categories unless the user named categories.
3. Many people from the CRM → bulk skill; a handful of e-mails → invite one by one.

## Execute guide

- New community: `mcp__plugin_clickmax_clickmax__communities_create({ name, description, portalId? })` → `mcp__plugin_clickmax_clickmax__communities_categories_create({ id: communityId, name })` → `mcp__plugin_clickmax_clickmax__communities_channels_create({ id: categoryId, name, ... })` per channel → `mcp__plugin_clickmax_clickmax__communities_categories_list` to confirm the final tree.
- Look: `mcp__plugin_clickmax_clickmax__communities_styling_update`; ranking/messages/sales URL: `mcp__plugin_clickmax_clickmax__communities_update`; limits/moderation: `mcp__plugin_clickmax_clickmax__communities_settings_update`.
- Order: `mcp__plugin_clickmax_clickmax__communities_categories_reorder` / `mcp__plugin_clickmax_clickmax__communities_channels_reorder` with every item and its position.
- Invite: `mcp__plugin_clickmax_clickmax__communities_members_invite({ id, email, name?, role? })` per person. Each invite and each role change asks the user for approval — batch the list in one message so they know what is coming.
- Change one member: `mcp__plugin_clickmax_clickmax__communities_members_list({ id, search })` → `mcp__plugin_clickmax_clickmax__communities_members_set_role` or `mcp__plugin_clickmax_clickmax__communities_members_set_status`.
- Event: `mcp__plugin_clickmax_clickmax__communities_events_create({ communityId, title, locationType, occurrences: [{ startsAt }] })`; keep the returned `id` — there is no admin event list tool.

## Report

- Show the resulting tree (category › channels) after building.
- For invites, list who was added and whether the e-mail went out.

## Warnings

- Deleting a category deletes all its channels; posts disappear. Say how many channels go with it.
- `owner` role hands out admin power — confirm before giving it.
- `communities_events_update` with `occurrences` replaces every date (RSVPs of removed dates are lost).

## Anti-patterns

- Inviting a whole CRM segment one e-mail at a time.
- Removing a member when the user wanted them quiet for a while (`muted`).
- Creating categories without channels and calling the community ready.

---

Clickmax skill revision: `862a8ca93146`
