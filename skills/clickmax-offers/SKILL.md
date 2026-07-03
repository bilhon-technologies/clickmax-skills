---
name: clickmax-offers
description: Use when the user wants to inspect, create, clone, update, approve, archive, unarchive, or delete product offers and checkout variants in Clickmax.
---

## When this applies

Use this skill for offer-level commercial control: pricing/currency/checkout variants, cloning the main offer, readiness diagnostics, approval submission, archive/unarchive, and destructive delete.

Not this skill:

- product catalog identity/archive/delete -> use the direct product tools

## Key assumptions

- the offer is the sellable variant; one main offer is created with the product
- lifecycle is `draft -> pending_approval -> active`
- `offers_get_incomplete_reason` is the preflight diagnostic before approval
- `offers_create` creates secondary offers only; clone-main is the safer variant path
- `offers_update` preserves the same offer id and existing checkout identity
- Offer type matters: internal offers sell through Clickmax checkout; external offers redirect/use `externalUrl`
- Public checkout identity is exposed as `hash`; keep it stable unless the user intentionally creates/clones a different offer
- Prices use the smallest currency unit; recurring behavior is controlled by `isRecurrent` and payment configuration, not product identity alone
- `quantityItems` controls delivered/sold quantity semantics for the offer variant

## Thought process

1. Distinguish read/diagnose from variant creation from lifecycle activation.
2. Prefer cloning the main offer when the user wants a close derivative.
3. Run incomplete-reason diagnostics before approval submission.
4. Confirm destructive delete when intent is not already explicit.

## Execute guide

- Use `mcp__clickmax__offers_get` to inspect one offer by id.
- Use `mcp__clickmax__offers_get_incomplete_reason` before approval to identify every blocking item for that offer id.
- Use `mcp__clickmax__offers_create` to create a secondary offer under the same product, passing the product id, offer name, price fields in cents, currency, and the intended `paymentConfig`.
- Use `mcp__clickmax__offers_clone_main_offer` when the user wants a new variant that inherits the current main-offer setup for the product id.
- Use `mcp__clickmax__offers_update` when the user wants to keep the same offer id and checkout identity while changing commercial fields such as name, price, default installment, currency, or `paymentConfig`.
- Use `mcp__clickmax__offers_send_to_approval` only after blockers are cleared for that offer id.
- Use `mcp__clickmax__offers_archive` or `mcp__clickmax__offers_unarchive` for lifecycle visibility changes without deleting the offer.
- Use `mcp__clickmax__offers_delete` only for explicit permanent removal and expect failure when transaction history prevents deletion.

## Report

- Start with the offer identity and whether it is main or secondary.
- Then report lifecycle state, commercial settings changed or inspected, and any remaining blockers.
- For approval requests, say clearly `ready for approval` or list each blocking item.
- For create/clone/update/archive/unarchive/delete actions, state the action result first and keep follow-up actions opt-in.

## Warnings

- Do not create a new secondary offer when the user really wants to modify the existing offer.
- Approval is not automatic; diagnostics matter.
- Delete can be blocked by transaction history.

## Anti-patterns

- Treating product and offer as the same object.
- Sending to approval without checking blockers.
- Treating `externalUrl` or checkout `hash` as cosmetic fields.
