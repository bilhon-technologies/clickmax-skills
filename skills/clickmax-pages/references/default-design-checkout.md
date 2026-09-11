# Default design — Minimal Checkout Frame

Start from [the bundled HTML base](../assets/editorial-checkout.html) and reuse the resolved sales-page palette/font. Read [visual system](visual-system.md) and [content discovery](content-discovery.md); these resources ship with the skill and need no catalog lookup.

**This is a structure, not a color system.** A checkout page is one hop away from the sales/capture page that sent the visitor here — it must read as the same funnel, not a new one. Never introduce a fresh palette for it: inherit the funnel's existing brand tokens (`var(--cx-color-...)`, the same `.theme-*` override if the funnel's other pages defined one). If this is the very first page of its funnel (no preceding styled page to inherit from), treat it as its own funnel and pick an accent the same way `default-design-course-launch-editorial.md` describes — but that is the exception, not the common case.

## What the checkout widget already does — and does not do

`<div data-cx-checkout></div>` (plus `offerId` at import) is replaced server-side by an iframe (`payments.clickmax.io/custom/...`) that renders the **offer summary, payment form, and any order bump** — by construction, always exactly that, never more. It has no benefits, guarantee, or FAQ content of its own. Everything around it — why-to-buy reinforcement, trust signals, objection handling — is entirely the page author's job. A checkout page that is just a logo and the iframe is a **valid, minimal, real product pattern** (plenty of live pages look exactly like this) — but it is the floor, not the ideal. This default adds the layer around the iframe that a bare checkout skips.

## Section anatomy (fixed order)

Two sections are required by the `checkout` recipe (`offer-summary`, `checkout`); the rest are optional and should only appear with real content from the user — never invented to "look more complete."

1. **Header strip** — logo/wordmark, optionally one line of context inherited from the funnel (edition, event date, "ao vivo") if the funnel already established one. Thin band, not a hero — no headline, no CTA here, the page has exactly one job.
2. **`data-cx-section="offer-summary"`** — a compact recap of what's being purchased, authored around (not inside) the checkout iframe: product/offer name, a small image if the offer has one, and 3–5 short bullets of what's included. This is what lets someone confirm "yes, this is the thing I meant to buy" before entering payment details — never omit it in favor of letting the iframe's own internal summary carry that alone, the two read differently (this block sells the decision, the iframe's internal summary is transactional).
3. **`data-cx-section="checkout"`** — the `<div data-cx-checkout></div>` marker itself. Single column, centered, nothing beside it at any breakpoint — a checkout page is never a two-column layout with a distraction in the second column.
4. **Trust row** (optional, cheap to always include when real) — payment-method icons (the same set already used elsewhere in the funnel, never invented logos) and a short "secure checkout" line. Placed directly under the checkout block, not the header.
5. **Guarantee** (optional, real terms only) — one short block, same guarantee language already used earlier in the funnel if one exists. Never invent a guarantee here that the sales/capture page didn't already promise.
6. **FAQ** (optional, 3–5 items) — only purchase-hesitation objections: refund/guarantee terms, when access is delivered, how to get support. Not a repeat of the sales page's FAQ — a checkout FAQ answers "should I complete this payment right now," not "should I buy this at all."
7. **Footer** — legal/contact only, no CTA (the checkout above is the only conversion action on this page).

## Do's and Don'ts

### Do

- Reuse the funnel's existing visual system verbatim — same tokens, same button/badge recipe, same type scale as the page that led here.
- Keep the offer-summary block honest and specific: real product name, real price context if shown, real inclusions — this block exists to prevent buyer's-remorse-before-purchase, not to re-sell.
- Ship the two-section minimum (header + checkout) with confidence when the user hasn't given benefit/guarantee/FAQ content — a clean minimal checkout is a legitimate outcome, not an unfinished one.

### Don't

- Don't add a second CTA, a navigation menu, or any link that isn't the checkout's own submit action — every exit from this page except completing or abandoning the purchase is a leak.
- Don't invent a guarantee, refund policy, or delivery timeline that wasn't stated earlier in the funnel or by the user.
- Don't build a two-column layout around the checkout iframe — it is designed to be read as a single vertical flow.
- Don't restate the full sales pitch here — that page already did its job; repeating it dilutes the one-decision focus of a checkout page.

## Known gaps

- This default does not cover order-bump copy/design — the order bump itself is rendered inside the same payments iframe as part of the checkout widget, not authored on the page.
- No color/typography system is defined here on purpose — see the note at the top of this file.
