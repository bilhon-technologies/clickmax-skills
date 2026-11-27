---
name: clickmax-scheduling
description: Use when the user wants to create, list, edit or configure a scheduling agenda (booking page), or get its public link or the embed code to show it on an external site.
---

## When this applies

Agenda = a booking page (tipo de evento, screen **Agendas**): name, duration, who attends, booking form, validations, post-booking, public page look, destination pipeline, notifications. A contact picks a free slot on its public link or on the widget embedded in another site, and a meeting is booked.

Asks: create / list / edit / configure an agenda | its public link | "código da agenda", "colocar a agenda no meu site", embed code | a ready prompt for another AI or site builder (Lovable, Cursor, v0, another agent) to place the agenda on a page.

"Where do I get the agenda code" is NOT a location-only question: run this skill and hand over the code itself.

Not this skill:

- one booked meeting (book, reschedule, cancel, recording) -> `clickmax-meetings`
- the team queue of tasks and appointments, the CRM agenda view -> `clickmax-activities`
- tracking script, lead-capture forms or redirects on an external page -> `clickmax-external-pages`
- meeting KPIs (show rate, self booking by agenda) -> `clickmax-insights-dashboards`

## Key assumptions

- Availability belongs to the ATTENDANT (weekly hours + date exceptions, shared by all their agendas), not to the agenda. Weekly hours, exceptions, Google Calendar connection and managed notifications are set on the screen; the tools do not change them. Guide the user and show the agenda screen.
- Who attends: `attendantId` (one attendant, `attendants_list`) | `memberId` (team member without an attendant yet; provisioned on save) | team agenda (`teamId` + `schedulingType: round_robin`) | group call (`schedulingType: group`, sessions + capacity). Without an attendant, member or team the agenda has nobody to offer slots.
- Defaults on create: `duration` 30 min, `isPublic` false (private = no link, no widget), slug generated when omitted.
- The link and the widget only work while `isPublic` is true. The widget renders inside the host page itself (no iframe) and shows the same agenda as the public link.
- Embed = `embedSnippet` from `mcp__plugin_clickmax_clickmax__appointment_schedules_embed_get`, verbatim: a `<div data-cx-booking="…">` where the agenda appears + one `<script … async>`. Optional attributes on the div: `data-cx-locale`, `data-cx-theme` (`light` | `dark` | `auto`), `data-cx-accent` (hex color). Never rebuild the snippet by hand.
- The script is the same Clickmax page script bundle: a page that already has the Clickmax page script (external page set up via `clickmax-external-pages`) needs only the `<div>`.
- Prefill: query params of the HOST page URL fill the widget form, same as on the link (`?name=…&email=…&phone=…`, active form fields only; `+` as `%2B`).

## Thought process

1. EVERY ask in this skill starts with `mcp__plugin_clickmax_clickmax__appointment_schedules_list`, including "create" (avoids duplicates, finds the one the user means).
2. Resolve the agenda:
   - user named one -> match by name; several close matches -> ask.
   - none named + 2 or more agendas -> ask ONE single-choice question listing the agendas by name, plus "Criar uma nova agenda" (a select when the chat has a question tool). Never guess.
   - none named + exactly 1 agenda -> read asks (link, code, prompt, inspect) use it and the reply names it ("usei a agenda X, a única da conta"); edits apply to it and the confirmation names it. An explicit "create" ask still creates a new one.
   - 0 agendas -> offer to create one; on yes, go to Create.
3. Create: ask only what is missing, in one question batch: name, duration, who attends (`attendants_list`). Everything else stays default and is tuned later on the screen or by asking. Goal is link/site/code -> create with `isPublic: true`.
4. Site / external page / "código para colar" -> Embed. "Prompt for another AI" -> Embed, then the prompt.
5. Writes one at a time; never batch-update agendas in parallel.

## Execute guide

- Every single-agenda operation (get, update, embed) takes `appointmentScheduleId`.
- List: `mcp__plugin_clickmax_clickmax__appointment_schedules_list` -> `{ data: [...] }`, the whole set (no pages). One agenda's full settings: `mcp__plugin_clickmax_clickmax__appointment_schedules_get`.
- Delete: not available here -> point to the **Agendas** screen.
- Create: `mcp__plugin_clickmax_clickmax__appointment_schedules_create` with `name`, `duration`, who attends (`attendantId` | `memberId` | `teamId` + `schedulingType`), `isPublic` per step 3.
- Edit: `mcp__plugin_clickmax_clickmax__appointment_schedules_update` with only the fields to change. The settings tabs map to: Geral (name, description, duration, color, `isPublic`, slug, meeting location) | Formulário (`bookingFields`, `collectFields`) | Validações (`minNoticeMinutes`, `bufferMinutes`, daily and per-slot caps, `maxAdvanceDays`, rolling window, contact code verification, screening) | Pós-agendamento (`confirmationMessage`, `confirmationButtons`, `redirectUrl`, `redirectDelaySeconds`, `pixelId`) | Página pública (first-slot preselect, navigation limit, highlight, confirmation pop-up, subtitle, logo, hide host name) | Pipeline (`pipelineId` + `stageId`, `defaultTemperature`) | Notificações (`emailLayoutTemplateId`). Disponibilidade = attendant hours on the screen.
- After every create or edit: show the agenda next to the chat (Max: `panel_open` kind `schedule`, id = `appointmentScheduleId`), without asking, so the user sees the result and every settings tab. Not kind `agenda`: that one is the CRM tasks/calls view.
- Embed: `mcp__plugin_clickmax_clickmax__appointment_schedules_embed_get` -> `{ appointmentScheduleId, name, slug, isPublic, publicUrl, embedSnippet }`.
  - `publicUrl`/`embedSnippet` null = the workspace has no active project, so there is no address to build them on -> say so and offer to create a project (`clickmax-projects`), then call embed_get again.
  - `isPublic` false -> say the code only works with the link on, offer to turn it on; on yes `mcp__plugin_clickmax_clickmax__appointment_schedules_update` with `isPublic: true`, then call embed_get again and show the panel.
  - Hand over: `embedSnippet` in an `html` code block -> where to paste -> prefill (only when the agenda form matters) -> `publicUrl` as the alternative (a button or link on the site that opens the agenda).
  - Where to paste: in the page HTML, at the exact spot the agenda should appear. Site builders: a custom HTML / embed code block placed in that section. Builder that strips scripts or isolates custom HTML -> use the link instead. Do not promise support for a specific builder by name.
- Prompt for another AI: build it from [embed prompt](references/embed-prompt.md) with the real agenda name, `embedSnippet` and `publicUrl`, in the user's language, in one `text` code block.

## Report

- Open with the agenda used (name) and, for single-agenda assumptions, that it was the only one.
- After create/edit: one line per changed setting (before -> after for edits), the public link if public, and the panel already open. What the tools cannot set (attendant hours, Google Calendar, notifications) -> one line saying it is on the screen beside the chat.
- Embed answer order: code block -> 2–4 short steps -> link alternative. Nothing else.
- Never show agenda ids, slugs as ids, field names or tool names; the slug only inside the link/snippet.
- Making the agenda public, creating, or editing beyond the ask = offered, then done on yes.

## Warnings

- Turning `isPublic` on publishes a link anyone can book on; turning it off breaks every widget and link already shared. Say so before switching.
- Changing the slug changes the link and the `data-cx-booking` value: every site and link with the old one stops working.
- The widget does not use the confirmation pop-up of the public page; that setting only affects the link.

## Anti-patterns

- Answering "onde pego o código da agenda" with only the screen location.
- Writing the snippet from memory or editing the `data-cx-booking` value.
- Picking an agenda on your own when the account has several.
- Handing a snippet of a private agenda without saying it will not show anything.
- Inventing a link or snippet when embed_get returned null for them.
- Trying to set weekly hours or calendar sync through agenda fields.
- Using the CRM agenda view panel instead of the schedule panel.

---

Clickmax skill revision: `0858cc6123fb`
