---
name: clickmax-ai-sdr-observability
description: Use when the user asks why the AI SDR agent answered, stayed silent, handed off or failed with a contact, or how the AI SDR performed in a period, on a number or in a deployment.
---

## When this applies

Questions about what the AI SDR (the AI agent that answers contacts on WhatsApp/webchat numbers) DID: one contact ("why didn't it answer João?") or a period ("how was the SDR this week?", "which number fails most?"). Data = the decision log behind Settings → Meus agentes → Observabilidade.

Not this skill:

- the conversation messages themselves → messaging/inbox skills
- the contact record, tags, fields → `clickmax-leads`
- Flows automations that are not the AI SDR block → `clickmax-flows`

## Key assumptions

- 1 decision = 1 time the agent was triggered. `trigger`: `inbound` = contact message | `follow_up` = scheduled follow-up | `flow_opening` = Flows block opened the conversation.
- `outcome` = what it did; `reasonCode` = why. Pairs (UI label in quotes):

  |outcome|meaning|common reasonCode|
  |-|-|-|
  |`spoke` "Respondeu"|message reached the contact|`replied`|
  |`suggested` "Sugeriu ao atendente"|deployment in suggest mode; text left for the attendant|`suggest_mode`|
  |`not_sent` "Redigiu, não enviou"|wrote a reply, it did not go out|`window_closed` (WhatsApp 24h window), `deployment_paused_midturn`, `silenced_midturn` (human stepped in while it wrote), `flow_ended_midturn`, `send_refused` (channel refused)|
  |`closed` "Encerrou a conversa"|agent ended the conversation|`model_closed`, `max_turns`|
  |`handed_off` "Passou para a equipe"|sent to the team|`model_escalated`, `max_turns`, `provider_failure`, `reply_after_exit`|
  |`flow_ended` "Encerrou a sessão do Flow"|Flows block session ended on an exit|`qualified`, `meeting_scheduled`, `disqualified`, `human_requested`, `low_confidence`, `limit_reached`, `ai_error`, `no_reply`, `opted_out`, `human_took_over`, `stop_word`, `max_messages`, `repeated_question`|
  |`silenced` "Ficou calado"|did not even try: something took the floor from it|`ai_paused`, `human_attendant`, `awaiting_human`, `session_closed`, `opted_out`, and audience rules: `meeting_held`, `meeting_upcoming`, `opportunity_won`, `opportunity_owned`, `human_elsewhere`|
  |`skipped` "Nada a fazer"|nothing to do|`outside_business_hours`, `live_session_missing`, `empty_history`, `unknown_channel`, `no_reply_text`|
  |`deferred` "Juntou com outro turno"|a turn was already running; message merged into it|`turn_busy`|
  |`error` "Falhou"|the turn broke and nobody took over|`turn_failed`|

- Failures = `error` + `handed_off/provider_failure` + `flow_ended/ai_error` — count all three, never `totals.errors` alone.
- `silenced` and `skipped` are usually CORRECT behavior (a human is attending, outside business hours, the contact is already a client), not bugs. Audience reasons (`meeting_*`, `opportunity_*`, `human_elsewhere`) come from the deployment's "Com quem o agente conversa" settings.
- `delivered` = some message reached the contact; `waitMs` = contact message → delivered reply (only when it replied).
- Window `days` default 7, max 90, cut in whole days of `timeZone`. No decision in the window ≠ never triggered.
- "Atendimento" (deployment) = one agent placed on one number/channel; one agent can have several.

## Thought process

1. One contact or a period? Contact → decisions list; period/number/agent → overview first.
2. Contact by name → `leads_search` first; homonyms → ask which one before reading decisions.
3. "Didn't answer" → look at the contact's latest `inbound` decisions: the `outcome` + `reasonCode` of the message the user means is the answer.

## Execute guide

- One contact: `mcp__plugin_clickmax_clickmax__leads_search` → `mcp__plugin_clickmax_clickmax__ai_sdr_decisions_list({ leadId, days: 7 })`. Empty → retry with `days: 30`, then `90`, before concluding the agent was never triggered for them (then: the contact may not be on a number with an active deployment).
- Period health: `mcp__plugin_clickmax_clickmax__ai_sdr_overview({ days })` → compare `totals` with `previous`, read `byReason` top rows, `placements` per number.
- Drill into a reason: take `agentId`/`deploymentId`/`channelInstanceId` from `placements[]`, then `mcp__plugin_clickmax_clickmax__ai_sdr_decisions_list({ outcome, reasonCode, ... })`.
- Tools that broke inside a turn: overview `tools[]` (`error`, `topErrorCode`); per decision `toolKeys` + `toolErrors`.
- Follow-ups stuck: overview `followUps` (`overdue`, `failed`, `reasons`).
- Pass `timeZone` (e.g. `America/Sao_Paulo`) when the user talks about "today", "yesterday" or days of the week.

## Report

- Contact question: lead with the answer in plain words ("Ficou calado porque havia um atendente humano na conversa"), then when it happened, on which number/deployment, and what would make it speak again.
- Use the UI labels (quoted in the table), never raw codes — unless the user asks for the code.
- Period question: decisions, % delivered, top 3 reasons, failures (the three-reason sum) and the change vs the previous window.
- Link the dashboard (`openPath`) when the user wants to see the charts or the hour-of-day heatmap.

## Warnings

- Do not call `silenced`/`skipped` a malfunction without reading the reason.
- `not_sent/window_closed` is a WhatsApp rule (24h without a contact message): only an approved template reopens it.
- Contact names in the log are personal data: answer only the user who asked; never paste the full list unprompted.

## Anti-patterns

- Answering "why didn't it answer" from the overview totals instead of the contact's own decisions.
- Searching decisions by name — the tool takes only `leadId`.
- Reporting "no decisions" after only the default 7 days.

---

Clickmax skill revision: `b154a7f2c55e`
