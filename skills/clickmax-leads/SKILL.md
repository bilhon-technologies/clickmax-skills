---
name: clickmax-leads
description: Use when the user wants to find, inspect, filter, or compare CRM leads and their commercial context inside Clickmax.
---

## When this applies

Use this skill when the user wants lead discovery or inspection: search/filter contacts, inspect one lead, check whether an email already exists, compare lead-origin patterns, or pull a lead's payments/invoices/products.

Not this skill:

- tagging/classification -> `clickmax-tags`
- manual lists or dynamic segments -> `clickmax-list-segments`
- kanban pipelines/opportunity cards -> `clickmax-pipelines`

## Key assumptions

- `mcp__clickmax__leads_filter_schema` is the source of truth for valid filters/operators; do not invent fields
- `mcp__clickmax__leads_search` is the main entry for cohort discovery
- `mcp__clickmax__leads_get` is enriched commercial context, not just a flat row
- payments, invoices, common products, and origin trees are lead-adjacent projections, not separate core entities
- tags, lists, and segments group leads; they do not replace the lead record itself
- lifecycle and temperature are mutable business signals; report them as current state, not immutable history

## Thought process

1. Decide whether the user needs one lead, a filtered cohort, or supporting aggregates.
2. If the user describes filters vaguely, inspect the filter schema first.
3. Use `leads_search` for cohorts and `leads_get` for one concrete lead.
4. Pull supporting projections only when they materially answer the request.

## Execute guide

- For cohort discovery, inspect `mcp__clickmax__leads_filter_schema` first when the user names filters loosely or when the valid operators are unclear.
- Search cohorts with `mcp__clickmax__leads_search`, passing the needed filters plus paging and sort fields. Use this for discovery, comparison, and broad CRM filtering.
- Inspect one known lead with `mcp__clickmax__leads_get`, passing the lead id. Treat this as the main enriched lead view.
- Add commercial context with `mcp__clickmax__leads_payments`, `mcp__clickmax__leads_invoices`, and `mcp__clickmax__leads_common_products` only when payments, billing status, or bought-product patterns materially change the answer.
- Use `mcp__clickmax__leads_exists_by_email` for duplicate-check questions, not enrichment.
- Use `mcp__clickmax__leads_origins`, `mcp__clickmax__leads_sub_origins`, and `mcp__clickmax__leads_origins_tree` for source taxonomy and breakdown questions.
- Use `mcp__clickmax__leads_payments_utm_autocomplete` when the user needs help discovering UTM values before filtering or diagnosing acquisition patterns.
- Preferred order: cohort question -> `mcp__clickmax__leads_filter_schema` when needed -> `mcp__clickmax__leads_search`; single lead question -> `mcp__clickmax__leads_get` -> supporting projections only if needed; origin or UTM exploration -> origin or UTM helper first -> lead search only when matching contacts are also required.

## Report

- Start with what was inspected: one lead, cohort, or origin/UTM diagnostic.
- For one lead: summarize identity, status/context, and only the relevant commercial facts.
- For cohorts: summarize count + the most relevant breakdowns before dumping rows.
- Cap long result sets and show `+N more` when the cohort is too broad.
- Follow-up actions are opt-in only.

## Warnings

- Do not guess filter fields or operators.
- Do not treat lead payments or invoices as if they were the lead record itself.
- `mcp__clickmax__leads_exists_by_email` answers existence, not ownership or enrichment.

## Anti-patterns

- Asking the user for workspace id.
- Using raw origins/UTM helpers as a substitute for lead search.
- Returning every field when the user only asked for one operational answer.
