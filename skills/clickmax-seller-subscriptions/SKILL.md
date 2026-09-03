---
name: clickmax-seller-subscriptions
description: Use when the user wants to inspect, chart, cancel, renew, or swap cards on customer subscriptions sold through Clickmax.
---

## When this applies

Use this skill for seller-side recurring subscriptions: inspect one subscription, filter/chart cohorts, swap the customer's billing card, cancel one subscription, or mass cancel/renew by product.

Not this skill:

- workspace Clickmax SaaS plan billing -> use the direct plan and buyer billing tools

## Key assumptions

- this is about end-customer subscriptions the workspace sells, not the workspace's own SaaS plan
- update-card is idempotent for the same card choice, but it is not a low-stakes edit: it repoints a live recurring charge, so the next renewal bills whoever owns the new card. Treat it as a money action and confirm the target subscription and card before calling
- cancel is permanent at the processor
- mass manage is broad and product-scoped, optionally narrowed by lead ids

## Thought process

1. Confirm which subscription scope the user means: customer subscription vs workspace SaaS plan.
2. Distinguish one-subscription mutation from bulk product-level management.
3. For destructive, broad, or billing-instrument changes, make the requested action explicit before mutating — name the subscription and the customer, not just the id.

## Execute guide

- Use `mcp__plugin_clickmax_clickmax__subscriptions_get` with `id = subscription id` to inspect one subscription.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_list` with `page`, `perPage`, and the needed filters such as `searchText`, `subscriptionStatus`, `productIds`, `column`, and `order` to find one customer or a cohort.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_chart` with `transactionStatus` and `membershipPeriod.from/to` for counts and trend views inside a billing or membership window.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_update_card` with `id` and `cardId` to swap the billing card for one subscription, after the user confirmed which subscription and which card.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_cancel` with `id`, `cancellationReason`, and optional `cancellationReasonDescription` to cancel one subscription.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_mass_manage` with `productId`, `operation`, and optional `leadIds` only for product-scoped bulk cancel or renew actions.

- Default order:
  1. inspect or list first when the user did not provide an exact subscription id
  2. use `subscriptions_chart` for counts/trends, not record-by-record inspection
  3. use `subscriptions_update_card` only for a single subscription
  4. use `subscriptions_mass_manage` only when the user clearly wants a product-scoped bulk action

## Report

- Start with the scope assumed: one subscription or a product-level cohort.
- For inspection, report status, product, customer match, and key lifecycle dates.
- For charts/lists, summarize the cohort first and then show the most relevant statuses or counts.
- For mutations, report the exact action completed and any affected scope or count.
- Treat follow-up actions as opt-in only.

## Warnings

- Do not mix this surface with SaaS workspace plan billing.
- Mass manage can affect many customers at once.

## Anti-patterns

- Canceling when the user only wants to inspect or pause conceptually.
- Using the SaaS plan tools for customer subscription problems.
