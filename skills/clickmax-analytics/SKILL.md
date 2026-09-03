---
name: clickmax-analytics
description: Use when the user asks business/revenue/KPI questions like how much they made, how they are performing, top products, lead counts, or funnel performance over a period.
---

## When this applies

Use this skill for business/KPI questions answered by specific analytics cuts over a date range: revenue totals, period-over-period sales, lead/conversion metrics, funnel step performance, messaging engagement, top products, most-accessed sales pages, and recent workspace activity.

## Not this skill

- payment dashboard browsing, headline KPIs, my-sales, or loss/recovery cohorts (`failed`, `canceled`, `refunded`, `chargedBack`, `dispute`, `pending`) -> `clickmax-payments-dashboard-analysis`
- refund/chargeback operations on a transaction -> `clickmax-transaction-operations`
- raw per-lead timelines or owner activity aggregates -> `clickmax-leads-activity-analysis`
- finding/filtering individual leads -> `clickmax-leads`
- building or reading a saved Insights dashboard of opportunities BI -> `clickmax-insights-dashboards`

## Key assumptions

- these are workspace-scoped read-only cuts; do not ask for workspace/owner ids.
- date filters are `startDate` / `endDate` (ISO date strings); backend clamps start to two-years-ago and end to today/tomorrow, so out-of-range dates are silently trimmed.
- comparison tools (`analytics_sales_metrics`, `analytics_leads_metrics`, `analytics_messages_metrics`) take a current window plus a `previousStartDate` / `previousEndDate` window; set both windows explicitly for honest growth reads.
- `projectSlugs`, `funnelIds`, `pageIds` are optional array filters; empty means whole workspace.
- percentage fields are 0..1 fractions (multiply by 100 for display); money fields are already amounts, not cents-strings.
- revenue splits: `analytics_total_sales` returns `totalInSales` (internal/native), `totalExternalSales` (imported platforms, with `externalBreakdown` per platform), and `totalRevenue` (sum). State which one you mean.
- there is no direct "lost/refused revenue" cut in this skill; `previousPeriodGrowth` (in `analytics_sales_metrics`) can be negative but means decline vs the prior window, not lost money.
- `analytics_top_products` `limit` defaults 5 (max 100); `analytics_sales_pages` `limit` defaults 5 (max 20).

## Thought process

1. Map the question to the narrowest tool:
   - "quanto faturei" / "how much did I make" -> `analytics_total_sales` for the range (headline revenue: internal + external + total), or `analytics_sales_metrics` when they also want conversion, top product, or growth vs a prior period.
   - "quantos leads" / lead conversion / lead price -> `analytics_leads_metrics`; lead-engagement overview across lists -> `analytics_leads_overview`.
   - "top produtos" / best sellers -> `analytics_top_products`.
   - "desempenho do funil" -> `analytics_funnel` (step + aggregate stats + sales history).
   - messaging engagement -> `analytics_messages_metrics`; automation reach/executions -> `analytics_flows_overview`.
   - page traffic -> `analytics_sales_pages`; latest workspace movement -> `analytics_recent_activities`.
2. For "quanto estou perdendo" / lost money: this skill has no direct loss field. Decide what the user means and be explicit:
   - money not converted (failed/canceled/refunded/chargeback/pending) -> hand off to `clickmax-payments-dashboard-analysis` (`dashboard_my_sales` with non-paid statuses); that is where loss cohorts live.
   - within this skill you can only approximate the gap: low `salesConversionPercentage` and `totalViews` vs `totalProductsSold` (from `analytics_sales_metrics`) show demand that did not convert, and negative `previousPeriodGrowth` shows revenue decline. Frame these as leakage signals, not a refund/loss total, and say so.
3. For period-over-period questions, always pass an explicit previous window so growth is meaningful.

## Execute guide

- Headline revenue for a window: use `mcp__plugin_clickmax_clickmax__analytics_total_sales` with `startDate` and `endDate` covering the window, optionally scoped by `projectSlugs`; read `totalRevenue` plus the `totalInSales` / `totalExternalSales` split.
- Sales performance with comparison: use `mcp__plugin_clickmax_clickmax__analytics_sales_metrics` with `startDate`/`endDate` for the current window and `previousStartDate`/`previousEndDate` for the prior window; read `totalAmount`, `salesConversionPercentage`, `totalProductsSold`, `products`, `topConversionProduct`, and `previousPeriodGrowth`.
- Lead metrics: use `mcp__plugin_clickmax_clickmax__analytics_leads_metrics` with the same window/previous-window pattern plus optional `funnelIds` / `pageIds`; read `totalLeads`, `leadConversionPercentage`, `leadAveragePrice`, and `leadsPerMonth`.
- Lead-engagement overview: use `mcp__plugin_clickmax_clickmax__analytics_leads_overview` with `startDate`/`endDate` for engaged contacts, active flows, and per-list engagement.
- Top products: use `mcp__plugin_clickmax_clickmax__analytics_top_products` with `startDate`/`endDate`, optional `funnelIds` or `productId`, and `limit` for how many to rank.
- Funnel performance: use `mcp__plugin_clickmax_clickmax__analytics_funnel` with `funnelIds` and/or `projectSlugs` plus the date window; read `steps`, `stats`, and `salesHistory`.
- Messaging engagement: use `mcp__plugin_clickmax_clickmax__analytics_messages_metrics` with window + previous window; read `totalMessagesSent` and per-channel (`mail`, `whatsApp`, `telegram`) engaged percentages.
- Automation reach: use `mcp__plugin_clickmax_clickmax__analytics_flows_overview` with `projectSlugs` and the window for `activeFlows`, `totalExecutions`, and `topFlows`.
- Sales-page traffic: use `mcp__plugin_clickmax_clickmax__analytics_sales_pages` with optional `projectSlugs` / `funnelIds` and `limit`.
- Recent activity: use `mcp__plugin_clickmax_clickmax__analytics_recent_activities` with `startDate`/`endDate` and optional `projectIds`, `funnelIds`, `categories`.
- Showcase prompt "quanto faturei nos últimos 15 dias e quanto estou perdendo":
  1. Compute the 15-day window (`endDate` = today, `startDate` = today − 15d) and the immediately prior 15-day window for `previousStartDate` / `previousEndDate`.
  2. Call `mcp__plugin_clickmax_clickmax__analytics_total_sales` for headline revenue and `mcp__plugin_clickmax_clickmax__analytics_sales_metrics` for conversion, top product, and `previousPeriodGrowth` over the same 15 days.
  3. There is no lost-revenue cut here: report the demand-vs-conversion gap (views vs products sold, low `salesConversionPercentage`, negative growth) as leakage signals, and hand off to `clickmax-payments-dashboard-analysis` for the actual failed/refunded/abandoned amount. State that the "loss" figure is partial unless that skill is used.
- Keep the same date window and filters across tools in one answer so numbers stay comparable.

## Report

- Open with the period assumed and the workspace scope (all projects unless the user narrowed it).
- Answer in plain business language with formatted currency and percentages (fractions ×100); never surface UUIDs, slugs, or raw payloads.
- Lead with the headline number the user asked for, then the strongest supporting cut (growth vs prior period, top product, conversion).
- The runtime renders presentation cards: emit a `cx-metric` for each headline KPI (revenue, conversion, leads) and a `cx-ranking` for top-products / top-flows lists instead of long inline tables.
- Cap ranked lists and summarize the tail as `+N more`.
- When "loss" was requested, be explicit that this skill covers revenue/conversion and that failed/refunded/abandoned amounts require the payments dashboard skill; give the partial signal you do have rather than inventing a loss total.
- Treat follow-up actions as opt-in only.

## Warnings

- Do not present `previousPeriodGrowth` or a negative trend as "money lost"; it is a period-over-period delta.
- Do not merge `totalExternalSales` into native-sales conclusions without labeling it; external is imported-platform revenue.
- Comparison tools need both windows; a missing previous window makes growth meaningless.
- Do not invent refund, chargeback, or abandoned-cart totals — those fields are not in these tools.
- Backend clamps dates to the last two years; flag it if the user asked for older data.

## Anti-patterns

- Answering "quanto estou perdendo" with a fabricated loss number instead of routing to the payments dashboard skill.
- Dumping every product/page/activity row instead of a ranked, capped summary.
- Reporting fractions as if they were already percentages.
- Reusing this skill for per-lead timelines or transaction refund operations.
- Asking the user for workspace or owner ids.
