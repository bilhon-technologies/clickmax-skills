---
name: clickmax-account-suggestions
description: Use when the user asks an open question about priorities in their Clickmax account ("what should I do now", "where do I start", "what is worth my time") or before proposing next steps on your own, so the answer comes from the account's ranked suggestions and their real numbers.
---

## When this applies

- Open question about priorities with no specific task: "o que vale fazer agora?", "por onde começo?", "o que está me fazendo perder dinheiro?".
- The requested work is done and you are about to suggest what comes next.

Not this skill:

- setup checklist progress ("Primeiros passos") -> `clickmax-getting-started`
- a specific metric the user named (revenue, failed payments, top offers) -> `clickmax-analytics`, `clickmax-failure-diagnosis`, `clickmax-sales-insights`
- executing the action of a suggestion -> the domain skill of its `area`

## Key assumptions

- `mcp__plugin_clickmax_clickmax__suggestions_list` returns the same suggestions the app shows as cards, already ranked by relevance. Do not re-rank by your own criteria.
- The list is per person: filtered by what this user is allowed to see and without what they dismissed or snoozed. A teammate may see a different list; never say a suggestion "does not exist" for the account.
- `kind = opportunity` -> money or risk found in the account's data, with numbers. `kind = trail` -> a getting-started step not done yet; `count` is how many steps remain, no amount, no window.
- `amountCents` is BRL cents (`27000` = R$ 270,00). `null` = the signal is not monetary. `windowDays` = how many days back it was read; `null` = a current state, not a count over time.
- Numbers can be up to ~20 minutes old (cached per account). A count off by one from a live report is expected, not a bug.

## Thought process

1. Open priority question -> call `mcp__plugin_clickmax_clickmax__suggestions_list` before answering; do not compose priorities from other tools first.
2. IF the user named an area (sales, automations, messages, students, contacts, lists, funnels, pages, opportunities) -> pass `area`.
3. IF the list is empty -> say nothing stands out right now; offer to look at a specific area. Do not fill the gap with generic marketing advice presented as account findings.
4. IF `partial = true` -> answer with what came, without implying the list is complete.
5. IF the user picks a suggestion -> switch to the domain skill for its `area` and do the work.

## Execute guide

- `mcp__plugin_clickmax_clickmax__suggestions_list({ limit: 3 })` for an open question; `{ area: 'sales' }` when the user scoped it.
- Lead with the top one or two `phrase` values, rewritten naturally in the user's language, keeping every number exactly as returned.
- Offer to act on the first one. `route` is the app path where the action happens; mention the screen in plain words only if the user wants to do it themselves.

## Report

- Short: the top items with their numbers, then one offer to act. No full dump of the list unless asked.
- Never show `id`, `kind`, `area` keys, `route` paths, or raw evidence fields.

## Warnings

- Quote only numbers the tool returned. Never estimate, round up, extrapolate a window, or add a value the suggestion did not carry.
- Never say "you are losing R$ X" when `amountCents` is null.

## Anti-patterns

- Answering "o que faço agora?" with a generic checklist when the tool has account-specific suggestions.
- Calling analytics tools to recompute a number that the suggestion already brought.

---

Clickmax skill revision: `a96306279f5f`
