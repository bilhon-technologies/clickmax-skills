---
name: clickmax-meetings
description: Use when the user wants to book, reschedule, hand over to another attendant, cancel or look up one meeting (agendamento) of a contact in Clickmax, including finding a meeting that already happened and its recording, transcript or summary.
---

## When this applies

Use for ONE booked meeting at a time: book it for a contact, move it in time, give it to another attendant, close it out (completed / no-show), cancel it, fetch its self-service link, find meetings by who conducts or who booked them, or bring back a meeting that already happened with its recording, transcript and summary.

Not this skill:

- the team queue of tasks and meetings together, workload, or closing a batch -> `clickmax-activities`
- a follow-up reminder with a deadline (a task, not a slot) -> the task operations; card side in `clickmax-pipelines`
- meeting KPIs: show rate, no-show, self booking, who books x who conducts -> `clickmax-insights-dashboards`
- writing the sales method a meeting follows -> `clickmax-playbooks`

## Key assumptions

- A meeting has TWO people: the attendant who CONDUCTS it (`attendantId`) and the one who BOOKED it (`createdByAttendantId` as a filter, `createdByAttendantName` in the row). They are independent: "meetings Helena scheduled" is the booker filter, "Helena's agenda" is the conductor filter. Public booking, automations and calendar sync have no booker.
- Booking is validated, not a blind insert. With an attendant the window must be inside their agenda (or the chosen agenda), their seat role must accept meetings, and it must not overlap another meeting of theirs. Two explicit overrides exist — off-hours (`allowOutsideAvailability`) and overlap (`allowConflict`); use them only when the user asked for that exact time. A booking with NO attendant, agenda or member skips all of it and lands unassigned.
- Rescheduling and handing over are PATCH edits of the same row: new time and/or new `attendantId`. The new time/attendant is re-checked for agenda availability (overridable), but the edit does NOT check overlap with other meetings — look at the new conductor's day first when a clash matters.
- `attendantId` can be changed but never cleared. A handover moves the calendar event to the new attendant's calendar and, when the meeting has a video room, makes them its host. The calendar copy is best-effort: the meeting can be right in Clickmax while the external calendar lags.
- The method the meeting follows is `playbookId`: omitted on booking = the default of the linked card's pipeline (none without a card), `null` = no method, an id = that one. Changing the card later never changes it. Ids come from `playbooks_list`.
- Cancelling keeps the record (status cancelled, reason stored, calendar event removed); it stays in listings. Do not cancel by setting `status` on the patch. `completed`, `no_show` and `confirmed` are set with the patch; a written result goes in `outcomeNote`.
- A plain attendant session only sees and edits its own meetings and those of its pipeline peers.
- A past meeting lives in the same row. Its recording is `recordingStatus` + `recordingUrl`; the content (recording, insights, summary) is blanked for meetings outside the user access to meetings and recordings, so empty fields are not proof nothing was recorded.
- Two kinds of recorded meeting: held in a Clickmax Meet room (`meetRoomCode` set) has a full report with player, transcript, summary and evaluation; a Google Meet recorded by the Clickmax Chrome extension has only the file, kept on the contact and the opportunity card. A Google Meet without the extension leaves nothing in Clickmax.
- The transcript text is not returned by these tools: it is read in the meeting report screen.
- The lead-facing self-service link (`scheduled_calls_manage_link`) lets the CONTACT reschedule, cancel or join; sending it is the user's decision, not a default.

## Thought process

1. Resolve the people first: the contact (`leads_search`) and the attendants (`attendants_list`); never guess ids from names.
2. Classify the ask: book / move in time / hand over / close out / cancel / look up. Only booking needs a contact; the rest need the meeting id from `scheduled_calls_list`.
3. If the ask changes who or when for SEVERAL meetings, do it one by one and confirm the exact list first; there is no bulk edit here.
4. Any change that moves a meeting the contact already knows about (time, person) is worth a confirmation before it is sent.
5. "The meeting of Friday" is a lookup, not a booking: resolve the relative day in the user time zone (the latest past Friday unless they say otherwise) before listing.

## Execute guide

- Book: `mcp__plugin_clickmax_clickmax__scheduled_calls_create` with `leadId`, `startAt`, `endAt` and one of `attendantId` | `appointmentScheduleId` | `memberId`; add `opportunityCardId` when the meeting belongs to a deal. A refusal about slot or overlap is information for the user (that time is not free), not something to retry with an override on your own.
- Look up: `mcp__plugin_clickmax_clickmax__scheduled_calls_list` with `attendantId` (conducts), `createdByAttendantId` (booked), `status`, a `startDate`/`endDate` window; `mcp__plugin_clickmax_clickmax__scheduled_calls_get` for one row with booker, agenda, method, card/pipeline and, once processed, recording insights and summary.
- Reschedule: list the conductor's meetings around the new window, then `mcp__plugin_clickmax_clickmax__scheduled_calls_update` with `id`, `startAt`, `endAt` (send both).
- Hand over: check the new attendant is operational and free (`attendants_list`, `scheduled_calls_list` with their `attendantId` on that day), then `mcp__plugin_clickmax_clickmax__scheduled_calls_update` with `id` + `attendantId`. Send `allowOutsideAvailability` only if the user explicitly accepts an off-hours handover.
- Close out: `mcp__plugin_clickmax_clickmax__scheduled_calls_update` with `status: "completed"` or `"no_show"` and, when the user dictates a result, `outcomeNote`.
- Cancel: `mcp__plugin_clickmax_clickmax__scheduled_calls_cancel` with an optional `reason`.
- Past meeting and its recording: `mcp__plugin_clickmax_clickmax__scheduled_calls_list` with `startDate`/`endDate` covering that whole day (add `leadId` or `attendantId` when the user named someone; past meetings may still be `scheduled` if nobody closed them out, so do not filter by status; pass `includeInactiveCards: true`, because a lost deal still has its recording). One match → `mcp__plugin_clickmax_clickmax__scheduled_calls_get`; several → list them (time, contact, conductor) and ask which. Then, by `recordingStatus`:
  - `ready` → give `recordingUrl` and the `meetSummary` when present.
  - `recording` / `processing` → it is still being processed; say so and offer to check again.
  - `failed` → say it did not arrive, with `recordingFailureReason`.
  - null → no recording reached Clickmax, or the user has no access to it; say both possibilities instead of claiming it was never recorded.
- Show the screen: when your chat can open a Clickmax screen next to the conversation, open it right after finding the meeting, without asking. Meeting with `meetRoomCode` → the meeting report (Max: panel kind `meeting`, id = the scheduled call id), where the user plays the recording and reads the transcript. Recording without a Meet room → the contact page (Max: panel kind `lead`, id = `leadId`).

## Report

- After booking: contact, day/time (with time zone as the user knows it), conductor, agenda/method if set, whether a calendar sync or confirmation e-mail went out, and any override that was used.
- After a move or handover: before -> after (time or person) and whether the external calendar was updated or is lagging.
- Past meeting: day/time, contact, conductor, recording status, the link or the reason there is none, the summary in two or three lines, and where the user finds it themselves: Clickmax Meet → Insights, Meetings mode, then the meeting row (report with recording and transcript); extension recording → the contact page or the opportunity card.
- For lookups: a short list (day, time, contact, conductor, booker, status), up to 10 rows unless the user asked for everything (then list all); when rows are left out, say how many (`+N more`) and offer to continue.
- Mutations beyond what the user asked (sending the self-service link, cancelling, overriding a slot) are offered, never done.

## Warnings

- "No slot / clash" answers are the system protecting the agenda; overrides book on top of real commitments.
- A patch never checks overlap: a successful handover or reschedule does not prove the new time is free.
- Meetings of a lost or deleted card are hidden from listings unless explicitly included.

## Anti-patterns

- Filtering by conductor to answer "who booked" (or the reverse).
- Booking without naming any attendant, agenda or member.
- Saying a meeting has no recording because `recordingUrl` is empty, without checking `recordingStatus` and access.
- Asking the user for the meeting id when a date or a contact name is enough to list it.
- Setting `status: "cancelled"` on the patch instead of cancelling.
- Retrying a refused booking with `allowConflict` / `allowOutsideAvailability` the user never approved.
- Looping over a batch without showing the list first.

---

Clickmax skill revision: `5d568af210b8`
