---
name: clickmax-transaction-operations
description: Use when the user wants to inspect transactions or sales, read transaction charts, or refund one or many transactions in Clickmax.
---

## When this applies

Use this skill for seller-side transaction and sale operations: inspect a transaction/sale, chart transaction activity, read sale buyer/UTM details, or refund one or many transactions.

Not this skill:

- recurring subscriptions -> `clickmax-seller-subscriptions`
- SaaS workspace billing/plans/cards -> payment billing skills

## Key assumptions

- transaction and sale are related but distinct records
- external-sale details are gateway/provider-specific and should not be merged into native transaction semantics without saying so
- status, UTM, and provider fields answer different questions: payment state, attribution, and gateway traceability
- single refund is permanent and reasoned
- mass refund is a broad workspace/product-level action and may cancel related subscriptions depending on backend behavior
- mass refund scope should be explicit: usually `productIds`, optionally narrowed further by `leadIds`
- debt is current accrued platform-fee debt, not all-time historical accounting
- `id` means a DIFFERENT id-space per tool, and there is no conversion tool between them: on `transactions_get`/`transactions_refund` it is the TRANSACTION id (route `/transactions/{id}`); on `sales_get_details`/`sales_get_buyer`/`sales_get_utms` it is the SALE id (route `/sales/{id}`); on `sales_get_external_details` it is a THIRD, external-sale id (route `/sales/external/{id}`). Never reuse one id across these tools assuming it's the same entity.

## Thought process

1. Distinguish inspection from refund mutation.
2. Identify whether the user needs transaction-level or sale-level detail.
3. Confirm any refund path unless the intent is already explicit and unambiguous.

## Execute guide

- Use `mcp__plugin_clickmax_clickmax__transactions_list` to list transactions by cohort. Pass pagination plus the exact filters the user asked for, such as `transactionStatus`, `productIds`, and `transactionPeriod.from/to`.
- For a paid-sales slice, prefer `transactionStatus = ['paid']` and restrict the date window explicitly.
- Use `mcp__plugin_clickmax_clickmax__transactions_get` when the user starts from a transaction id and needs the transaction record itself.
- Inspect one sale with `mcp__plugin_clickmax_clickmax__sales_get_details`, then add `mcp__plugin_clickmax_clickmax__sales_get_buyer` for buyer identity/context and `mcp__plugin_clickmax_clickmax__sales_get_utms` for attribution context.
- Use `mcp__plugin_clickmax_clickmax__sales_get_external_details` when the user needs provider-facing or external sale detail beyond the main sale record.
- Read debt with `mcp__plugin_clickmax_clickmax__transactions_debt` when the user asks about current platform-fee debt.
- Read chart trends with `mcp__plugin_clickmax_clickmax__transactions_chart` when the user asks for transaction volume or movement over time.
- Single refund: use `mcp__plugin_clickmax_clickmax__transactions_refund` with the exact transaction id, a valid cancellation reason, and a short description when needed.
- Mass refund: use `mcp__plugin_clickmax_clickmax__transactions_mass_refund` only after the target scope is explicit, typically by product, lead cohort, or both.
- For mass refund, avoid broad workspace-wide input; prefer product-scoped input and add lead narrowing when the user names a specific cohort.

## Report

- For reads: summarize the commercial answer, not raw accounting internals.
- For refunds: report what was refunded, what else was affected, and any failures/skips.

## Warnings

- Do not confuse seller transaction refunds with SaaS billing changes.
- Mass refund is high-impact and can span many records.

## Anti-patterns

- Refunding based on fuzzy product matching without confirming scope.
- Treating sale buyer/UTM helpers as the canonical transaction object.
