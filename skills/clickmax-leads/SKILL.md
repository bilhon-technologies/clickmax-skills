---
name: clickmax-leads
description: Use when the user wants to create, find, inspect, filter, verify the e-mail of, or compare CRM leads and their commercial context inside Clickmax.
---

## When this applies

Use this skill when the user wants lead discovery/inspection OR to create a lead: search/filter contacts, inspect one lead, check whether an email already exists, verify a contact's e-mail, find contacts with e-mail problems or fraud signals, create a new contact, compare lead-origin patterns, or pull a lead's payments/invoices/products.

Not this skill:

- tagging/classification -> `clickmax-tags`
- manual lists or dynamic segments -> `clickmax-list-segments`
- kanban pipelines/opportunity cards -> `clickmax-pipelines`

## Key assumptions

- `mcp__plugin_clickmax_clickmax__leads_filter_schema` describes the legacy filter model; never use it to build `mcp__plugin_clickmax_clickmax__leads_search` filters
- `mcp__plugin_clickmax_clickmax__leads_search` is the main entry for cohort discovery. `filters` is required (`[]` = all leads), and `perPage` accepts 1-100. Its only `sortBy` values are `opportunities_asc`/`opportunities_desc`; omitting `sortBy` returns leads NEWEST-FIRST by `createdAt`. So "the last N leads" uses `filters: []`, `page: 1`, `perPage: N` (N <= 100) with no `sortBy`
- `mcp__plugin_clickmax_clickmax__leads_get` is enriched commercial context, not just a flat row
- payments, invoices, common products, and origin trees are lead-adjacent projections, not separate core entities
- tags, lists, and segments group leads; they do not replace the lead record itself
- lifecycle and temperature are mutable business signals; report them as current state, not immutable history
- e-mail status lives on the lead (`emailStatus`, `emailValidatedAt`, `emailInvalidReason`) and comes back in BOTH `mcp__plugin_clickmax_clickmax__leads_search` rows and `mcp__plugin_clickmax_clickmax__leads_get`: `pending` = not verified yet | `valid` | `invalid` (bounces; fix or exclude) | `risky` (deliverable but disposable/catch-all/role address; sending is still allowed, the user decides). Reading it costs nothing — do not verify just to read it
- `valid` from the automatic background pass only confirms the DOMAIN; `mcp__plugin_clickmax_clickmax__leads_validate_email` asks for a mailbox-level check. Editing a contact e-mail already resets it to `pending` and triggers a check by itself
- `mcp__plugin_clickmax_clickmax__leads_validate_email` is a PAID provider lookup (1 credit per address), synchronous, limited to 10 calls/min, and repeating it for the same lead within 1 hour just returns the stored verdict. It is for one specific contact, never a base-wide sweep
- `suspectedFraud` (+ `suspectedFraudReasons` `email`/`document`) is a read-time warning of card testing, true only when BOTH the e-mail has no relation to the name AND the document is suspect (empty, all zeros, or not 11/14 digits). Nothing is stored or blocked; fixing the e-mail or document clears it. It is a filter field like any other, so it also works in segments

## Thought process

1. Decide whether the user needs one lead, a filtered cohort, or supporting aggregates.
2. If the user describes filters vaguely, inspect the filter schema first.
3. Use `leads_search` for cohorts and `leads_get` for one concrete lead.
4. Pull supporting projections only when they materially answer the request.

## Execute guide

- For cohort discovery, follow the `mcp__plugin_clickmax_clickmax__leads_search` input contract; do not translate fields from the legacy filter-schema operation.
- Search cohorts with `mcp__plugin_clickmax_clickmax__leads_search`, passing the required `filters` array (`[]` when unfiltered) plus optional paging and sort fields. Use this for discovery, comparison, and broad CRM filtering.
- Inspect one known lead with `mcp__plugin_clickmax_clickmax__leads_get`, passing the lead id. Treat this as the main enriched lead view.
- Add commercial context with `mcp__plugin_clickmax_clickmax__leads_payments`, `mcp__plugin_clickmax_clickmax__leads_invoices`, and `mcp__plugin_clickmax_clickmax__leads_common_products` only when payments, billing status, or bought-product patterns materially change the answer. Unlike its siblings, `mcp__plugin_clickmax_clickmax__leads_common_products` requires `filter` (not optional) — always pass a filter, even a broad one.
- Use `mcp__plugin_clickmax_clickmax__leads_exists_by_email` for duplicate-check questions, not enrichment.
- E-mail health of a cohort: filter `mcp__plugin_clickmax_clickmax__leads_search` with a filter item on field `emailStatus`, operator `equals`, `valueString` = `invalid` | `risky` | `pending` | `valid` (add `negation: true` for "anything but"). Fraud signals: a filter item on field `suspectedFraud`, operator `equals`, `valueBool` = true. Report the count (`meta.countItens`) before rows.
- Verify one contact's e-mail with `mcp__plugin_clickmax_clickmax__leads_validate_email`, passing the lead id: it returns the new `emailStatus` right away. A `pending` result after a failure (provider error / inconclusive) is not a verdict — the automatic job retries later; do not retry in a loop. A lead without e-mail cannot be verified.
- "How many contacts do I have in total?" -> `mcp__plugin_clickmax_clickmax__analytics_resource_counts` (`contacts`, merged contacts excluded) is one cheap call; use `leads_search` `meta.countItens` when the count must respect filters.
- Create one contact with `mcp__plugin_clickmax_clickmax__leads_create` — only `name` is required; pass `email`/`telephone` when known and check `mcp__plugin_clickmax_clickmax__leads_exists_by_email` first to avoid duplicates. It also accepts `tagIds`/`customFieldValues` inline, so a lead can be created pre-tagged/pre-classified in the SAME call instead of a separate tagging step afterward. It returns the new lead id. To seed a pipeline, create each contact here then add them as opportunity cards via `clickmax-pipelines` (`cards_create` needs the returned lead ids). For several contacts, call `leads_create` once per contact.
- Use `mcp__plugin_clickmax_clickmax__leads_origins`, `mcp__plugin_clickmax_clickmax__leads_sub_origins`, and `mcp__plugin_clickmax_clickmax__leads_origins_tree` for source taxonomy and breakdown questions.
- Use `mcp__plugin_clickmax_clickmax__leads_payments_utm_autocomplete` when the user needs help discovering UTM values before filtering or diagnosing acquisition patterns.
- Preferred order: cohort question -> `mcp__plugin_clickmax_clickmax__leads_search`; single lead question -> `mcp__plugin_clickmax_clickmax__leads_get` -> supporting projections only if needed; origin or UTM exploration -> origin or UTM helper first -> lead search only when matching contacts are also required.

## Report

- Start with what was inspected: one lead, cohort, or origin/UTM diagnostic.
- For one lead: summarize identity, status/context, and only the relevant commercial facts.
- For cohorts: summarize count + the most relevant breakdowns before dumping rows.
- Cap long result sets and show `+N more` when the cohort is too broad.
- Follow-up actions are opt-in only.

## Warnings

- Do not guess filter fields or operators.
- Do not treat lead payments or invoices as if they were the lead record itself.
- `mcp__plugin_clickmax_clickmax__leads_exists_by_email` answers existence, not ownership or enrichment.
- Do not present `suspectedFraud` as a verdict on the person: it is a heuristic signal, and a missing document alone does not trigger it.

## Anti-patterns

- Asking the user for workspace id.
- Using raw origins/UTM helpers as a substitute for lead search.
- Returning every field when the user only asked for one operational answer.
- Calling `mcp__plugin_clickmax_clickmax__leads_validate_email` on many leads to "clean the base" (paid, rate-limited) or just to read a status that `leads_get` already returns.

---

Clickmax skill revision: `2f946ae45dc1`
