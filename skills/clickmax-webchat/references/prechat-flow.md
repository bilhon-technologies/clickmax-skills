# Webchat pre-chat flow: exact step shapes and branching

Source of truth = `preChatFlow` accepted by `mcp__plugin_clickmax_clickmax__webchat_channel_update` /
`mcp__plugin_clickmax_clickmax__webchat_channel_create` (`WebChatPreChatFlowSchema` in the platform models). Guessing a
field is what makes the update reject; use these shapes verbatim.

## Document

```json
{ "enabled": true, "steps": [ ...Step ] }
```

- `enabled: false`, `steps: []` or `preChatFlow: null` → the widget opens straight into live chat.
- Max 40 steps. Array order is the DEFAULT path: a step without `next` falls through to the next
  step in the array. `next` overrides that with another step `id`, `"handoff"` (human takes over)
  or `"end"` (closes the conversation).
- Every `id` is unique, 1–64 chars. Use readable ids (`s_welcome`, `q_goal`, `h_sales`).

## Steps (discriminated by `type`)

- `message` — bot bubble. `{ id, type: "message", text, next? }` (`text` 1–2000 chars).
- `capture` — collects lead data and creates/enriches the CRM lead.
  `{ id, type: "capture", text?, fields: [{ key: "name" | "email" | "phone", label?, required }], next? }`
  — 1 to 3 fields, `required` is mandatory on each. Put it BEFORE the qualifying questions so
  every later answer lands on a real lead.
- `question` (clickable options, default) — `{ id, type: "question", text, options: [{ id, label, next? }], answerField? }`
  — 1 to 6 options, `label` ≤120 chars. Each option branches by its own `next`; an option
  without `next` falls through to the next step in the array. No `next` on the step itself.
- `question` (free text) — `{ id, type: "question", text, answerMode: "text", options: [], next?, answerField? }`
  — the visitor types; single exit through `next`.
- `answerField` (either question mode) maps the answer to a lead field: a native key (`name`,
  `email`, `telephone`, `city`, …) or `cf.<customFieldId>` for a custom field. Omit when in doubt.
- `condition` — IF over the last choice or a captured field.
  `{ id, type: "condition", variable: "choice" | "name" | "email" | "phone", op: "contains" | "equals" | "exists", value?, next?, elseNext? }`
  — `value` unused with `exists`; `next` = true branch, `elseNext` = false branch.
- `ai` — free-text answer classified into branches by the model.
  `{ id, type: "ai", text, instruction?, branches: [{ id, label, description?, next? }], fallbackNext? }`
  — 1 to 8 branches; `label`/`description` are what the model matches on; `fallbackNext` when
  nothing matches. Prefer a clickable `question` when the options are few and known.
- `handoff` — terminal; sends the conversation to a human in LiveChat.
  `{ id, type: "handoff", text?, assignTo? }` — `assignTo` is an attendant user id; omit it so the
  conversation lands in the open queue.
- `end` — terminal; closes the conversation. `{ id, type: "end", text? }`.

## Canonical flow (sales qualification)

```json
{
  "enabled": true,
  "steps": [
    { "id": "s_welcome", "type": "message", "text": "Oi! Posso te ajudar a encontrar o plano certo." },
    {
      "id": "s_capture",
      "type": "capture",
      "text": "Antes, me diz seu nome e e-mail:",
      "fields": [
        { "key": "name", "required": true },
        { "key": "email", "required": true },
        { "key": "phone", "required": false }
      ]
    },
    {
      "id": "q_size",
      "type": "question",
      "text": "Quantas pessoas tem no seu time?",
      "options": [
        { "id": "o_solo", "label": "Só eu", "next": "e_selfserve" },
        { "id": "o_small", "label": "2 a 10", "next": "q_goal" },
        { "id": "o_big", "label": "Mais de 10", "next": "h_sales" }
      ]
    },
    {
      "id": "q_goal",
      "type": "question",
      "text": "O que você quer resolver primeiro?",
      "options": [
        { "id": "o_leads", "label": "Gerar leads", "next": "h_sales" },
        { "id": "o_support", "label": "Atender clientes", "next": "h_support" }
      ]
    },
    { "id": "h_sales", "type": "handoff", "text": "Perfeito, vou te passar pra um especialista." },
    { "id": "h_support", "type": "handoff", "text": "Vou chamar alguém do atendimento." },
    {
      "id": "e_selfserve",
      "type": "end",
      "text": "Você consegue começar sozinho pelo plano gratuito. Qualquer dúvida, é só chamar!"
    }
  ]
}
```

Checklist before sending: every `next` points to an existing `id`, `"handoff"` or `"end"`; no path
falls off the end of the array without a terminal; `capture` comes before the questions; each
clickable `question` has ≥1 option.
