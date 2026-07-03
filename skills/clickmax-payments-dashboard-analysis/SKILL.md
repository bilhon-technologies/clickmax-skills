---
name: clickmax-payments-dashboard-analysis
description: Use when the user wants payment dashboard KPIs, paginated dashboard views, my-sales queries, or filter lookups for seller revenue analysis in Clickmax.
---

## When this applies

Use this skill for payment dashboard browsing and seller revenue analysis: headline KPIs, dashboard filter discovery, paginated transactions/subscriptions/affiliate lists, my-sales queries, and external sales charts.

Not this skill:

- detailed refund operations -> `clickmax-transaction-operations`
- customer subscription lifecycle mutation -> `clickmax-seller-subscriptions`

## Key assumptions

- `dashboard_filters` is the discovery surface for valid filter values
- `dashboard_my_sales` supports richer filter bodies than the simpler paginated lists
- list endpoints and KPI endpoints can derive time windows differently from the chosen range
- external sales are a separate surface and should not be merged blindly into native sales conclusions

## Thought process

1. Identify whether the user needs KPIs, paginated rows, filter discovery, or deeper my-sales queries.
2. Prefer the narrowest dashboard surface that answers the question.
3. Use my-sales when the filter logic is more expressive than the lightweight dashboard lists.

## Execute guide

- For KPI snapshots, use `mcp__clickmax__dashboard_get` with the requested range and only the narrow filters that materially change the reading, such as product.
- For available filter values before analysis, use `mcp__clickmax__dashboard_filters` with an empty body once per session, then reuse discovered project/product/client/status values instead of guessing ids or labels.
- Keep filter shape explicit: date ranges, status/project/product/client filters, pagination (`page` / `perPage`), and sorting (`column` / `order`) materially change dashboard rows.
- For paginated dashboard browsing, use the smallest list that matches the request:
  - transactions -> `mcp__clickmax__dashboard_transactions_list`
  - subscriptions -> `mcp__clickmax__dashboard_subscriptions_list`
  - affiliates -> `mcp__clickmax__dashboard_affiliates_list`
- For seller-revenue analysis with richer filter bodies, prefer `mcp__clickmax__dashboard_my_sales`, especially when the question depends on `productIds`, `transactionStatus`, or an explicit `transactionPeriod` window.
- For recovery or loss cohorts, use `mcp__clickmax__dashboard_my_sales` with non-paid statuses such as `failed`, `canceled`, `refunded`, `chargedBack`, `dispute`, and `pending`.
- For external-sales trend reading, use `mcp__clickmax__dashboard_external_sales_chart` with explicit `from`, `to`, and period granularity such as `daily`.
- For external-sales row analysis, use `mcp__clickmax__dashboard_external_sales`; keep those conclusions separate from native dashboard KPIs unless the user explicitly wants a combined comparison.

## Report

- Lead with the business takeaway and the period assumption.
- Order results from highest-signal KPI or cohort insight to supporting rows.
- Cap long row dumps and prefer ranked summaries with `+N more` when needed.
- Treat follow-up actions as opt-in only.

## Warnings

- Do not merge external-sales semantics into native dashboard KPIs without stating it.
- Dashboard ranges and filter bodies matter to interpretation.

## Anti-patterns

- Pulling my-sales for every small dashboard question.
- Dumping raw paginated rows without synthesis.
