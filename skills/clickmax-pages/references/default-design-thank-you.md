# Default design — Confirmation Split

Start from [the bundled HTML base](../assets/editorial-thank-you.html) and reuse the resolved sales-page palette/font. Read [visual system](visual-system.md) and [content discovery](content-discovery.md); these resources ship with the skill and need no catalog lookup.

**This is a structure, not a color system** — same rule as the checkout default. A thank-you page is the last stop of a funnel someone just converted on; it must read as the same funnel, not a new one. Inherit the funnel's existing brand tokens rather than introducing a fresh palette. The one deliberate exception is a semantic **success accent** (a green checkmark badge) — that color communicates "done" universally and should stay distinct from the funnel's own brand accent, the same way a WhatsApp/support affordance elsewhere in this catalog keeps its own green regardless of the page's theme.

## What this recipe forbids — and why the structure below respects it

`thank-you` forbids both lead capture and checkout. That is a real product constraint, not a style choice: no `<form data-cx-ingest-form>`, no `<div data-cx-checkout></div>` anywhere on this page. A bonus/one-time offer is still a legitimate, common thing to put on a thank-you page — but it must be a plain outbound link to a separate checkout page elsewhere, never an embedded checkout widget.

## Section anatomy (fixed order)

Two-column split desktop (confirmation left, action bento right), single stacked column on mobile — collapse order is confirmation first, then the bento, never interleaved.

1. **Brand mark** — logo only, top of the left column. No nav, no other link out of the page except the one required next-step action below.
2. **`data-cx-section="confirmation"`** (left column, required) — a small pill/badge combining a checkmark icon in a colored ring with a short uppercase label ("access unlocked," "you're in," or equivalent — real to the product) in the success accent; then an H1 that states plainly what just happened ("You just secured access to **[the thing]**," one word bold); then one short support paragraph restating the value in the funnel's own words — never a generic "thank you for your purchase" line that could belong to any product.
3. **Two info tiles** (right column, top row, optional but cheap and worth including whenever real facts exist) — compact practical details a buyer needs right after paying: date/time, format (live/recorded/shipping), anything logistic. Real facts only — never assert a delivery platform, channel, or timeline the user hasn't confirmed.
4. **`data-cx-section="next-step"`** (right column, full-width tile, required) — the single required action: join the community/channel where delivery actually happens, download the thing, or whatever the ONE next action is. This tile gets the strongest visual treatment on the page (its own accent-tinted border/background, distinct from the neutral info tiles) — it is the only thing on the page besides the optional bonus offer that is clickable.
5. **Bonus/one-time-offer tile** (right column, full-width, optional — only with real price/product from the user) — a last-chance related offer, framed as exclusive to this page ("only here, only now"). The CTA is a plain outbound `<a>` to a separate checkout page (never `data-cx-checkout` on this page). When no purchase link exists yet, render the same visual treatment as a disabled, non-interactive element (`aria-disabled`, no `href`, not focusable) rather than a dead link or omitting the tile silently — honest-but-incomplete beats fake-but-clickable.
6. **Footer** — legal/contact only, no CTA repeated here (the next-step tile above is the only conversion action worth making).

## Do's and Don'ts

### Do

- Make the next-step action visually unmistakable — it is the entire point of the page. If a visitor converts and then can't find what to do next, the sale still fails to deliver.
- Keep the success badge's color semantic (a checkmark green, or whatever the product already uses for "done") independent of the funnel's brand accent.
- Ship the two-tile minimum (info + next-step) with confidence when there is no bonus offer — a clean confirmation page is complete on its own, not unfinished.
- Reuse the funnel's existing visual system verbatim otherwise — same tokens, same button recipe, same type scale as the page that led here.

### Don't

- Don't add a capture form or an embedded checkout anywhere on this page — both are forbidden by the recipe. A bonus offer links OUT to its own checkout page.
- Don't assert a delivery platform, channel name, or exact timing that the user hasn't confirmed — a wrong guess here breaks trust immediately after a purchase, when the reader is at their most attentive.
- Don't add a real headline pitch or objection-handling copy — that was the sales/capture page's job; this page confirms and directs, it doesn't sell again (except the one narrow, explicitly-labeled bonus offer).
- Don't invent a bonus/OTO offer or price the user never gave you — omit the tile rather than fabricate one.

## Known gaps

- No color/typography system is defined here on purpose — see the note at the top of this file.
- This default assumes a single primary next-step action. A funnel that genuinely needs to direct the visitor to two unrelated next steps should treat that as a style/structure request from the user, not something this default should silently improvise.
