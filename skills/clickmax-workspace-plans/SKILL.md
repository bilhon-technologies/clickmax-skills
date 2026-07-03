---
name: clickmax-workspace-plans
description: Use when the user wants to inspect, compare, preview cancellation of, or cancel the workspace's own Clickmax SaaS subscription and billing plan.
---

## When this applies

Use this skill for the workspace's own Clickmax SaaS billing plan: inspect current plan/usage, compare upgrades, view invoice or plan transaction history, preview cancellation impact, or cancel the current plan.

Not this skill:

- customer subscriptions sold by the workspace -> `clickmax-seller-subscriptions`

## Key assumptions

- this surface is about the workspace as Clickmax customer
- usage metrics are current billing-period counts
- cancel preview should come before cancel execution when intent is exploratory
- admin cancel tools are especially high-risk because they can affect multiple workspaces or arbitrary plan ids

## Thought process

1. Confirm the user means the workspace's own Clickmax subscription, not customer subscriptions sold by that workspace.
2. Separate read-only questions from cancellation intent.
3. Treat admin cancellation as exceptional because it can target arbitrary subscriptions or multiple workspaces.

## Execute guide

- Use `mcp__clickmax__plans_user_summary`, `mcp__clickmax__plans_user_metrics`, and `mcp__clickmax__plans_invoice_summary` together for current plan overview, billing status, renewal context, and current billing-period usage.
- Use `mcp__clickmax__plans_upgrade_list` when the user wants higher-tier options or a comparison between the current plan and available upgrades.
- Use `mcp__clickmax__plans_transaction_history` for past billing events, renewals, charges, and plan movements.
- Use `mcp__clickmax__plans_pending_changes` to detect already-scheduled plan changes before suggesting or performing another change.
- Use `mcp__clickmax__plans_cancel_preview` with the current subscription id when the user is still evaluating cancellation impact, timing, expiration, or access loss.
- Use `mcp__clickmax__plans_cancel_subscription` only after the user clearly asked to cancel; pass the current subscription id and the user's stated reason.
- Use `mcp__clickmax__plans_admin_cancel_by_workspaces` only for explicit admin-scope requests that target one or more specific workspaces.
- Use `mcp__clickmax__plans_admin_cancel_by_id` only for explicit admin-scope requests that target a specific subscription id.
- Before any cancellation path, check pending changes so you do not stack a new mutation on top of an already-scheduled one.

## Report

- Open with the scope: current plan, usage, billing history, upgrade options, pending changes, or cancellation impact.
- Order the answer: current state -> relevant metrics/history -> next action options.
- For cancellation preview or cancellation result, state scheduled effects such as expiration timing or pending access changes when returned.
- Treat follow-up mutations as opt-in only unless the user explicitly asked to cancel.

## Warnings

- Do not use customer subscription tools here.
- Admin cancel tools are broad and high-risk.

## Anti-patterns

- Canceling without preview when the user is still evaluating options.
- Mixing workspace SaaS billing with seller revenue operations.
