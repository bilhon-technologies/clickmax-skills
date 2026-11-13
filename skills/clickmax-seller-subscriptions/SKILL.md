---
name: clickmax-seller-subscriptions
description: Use when the user wants to inspect, chart, or cancel customer subscriptions sold through Clickmax.
---

## When this applies

Use this skill for seller-side recurring subscriptions: inspect one subscription, filter/chart cohorts, or cancel one subscription. There is no tool to swap the customer's billing card; the customer changes it on the gateway side.

Mutations here are always single-subscription: there is no bulk cancel and no renew. Reactivating a canceled subscription means the customer subscribes again through checkout.

Not this skill:

- workspace Clickmax SaaS plan billing -> use the direct plan and buyer billing tools

## Key assumptions

- this is about end-customer subscriptions the workspace sells, not the workspace's own SaaS plan
- cancel is permanent at the processor and applies to exactly one subscription
- a cohort can be listed and charted, but not mutated in one call; acting on many subscriptions means one call per subscription, each confirmed

## Thought process

1. Confirm which subscription scope the user means: customer subscription vs workspace SaaS plan.
2. For destructive or billing-instrument changes, make the requested action explicit before mutating — name the subscription and the customer, not just the id.

## Execute guide

- Use `mcp__plugin_clickmax_clickmax__subscriptions_get` with `id = subscription id` to inspect one subscription.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_list` with `page`, `perPage`, and the needed filters such as `searchText`, `subscriptionStatus`, `productIds`, `column`, and `order` to find one customer or a cohort.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_chart` with `transactionStatus` and `membershipPeriod.from/to` for counts and trend views inside a billing or membership window.
- Use `mcp__plugin_clickmax_clickmax__subscriptions_cancel` with `id`, `cancellationReason`, and optional `cancellationReasonDescription` to cancel one subscription.

- Default order:
  1. inspect or list first when the user did not provide an exact subscription id
  2. use `subscriptions_chart` for counts/trends, not record-by-record inspection
  3. use `subscriptions_cancel` on one subscription at a time

## Report

- Start with the scope assumed: one subscription or a product-level cohort.
- For inspection, report status, product, customer match, and key lifecycle dates.
- For charts/lists, summarize the cohort first and then show the most relevant statuses or counts.
- For mutations, report the exact action completed and the subscription it affected.
- Treat follow-up actions as opt-in only.

## Warnings

- Do not mix this surface with SaaS workspace plan billing.
- A cohort listing is not an action target: cancelling what a filter returned still requires one confirmed call per subscription.

## Anti-patterns

- Canceling when the user only wants to inspect or pause conceptually.
- Using the SaaS plan tools for customer subscription problems.
- Promising a bulk cancel or a renew; neither exists on this surface.

---

Clickmax skill revision: `a33abcf246fd`
