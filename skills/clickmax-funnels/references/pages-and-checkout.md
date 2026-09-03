# Funnel pages and checkout

A funnel `page` node is only a graph slot — it has no content until a page is created and linked to it.

**How a page is authored is in the `clickmax-pages` skill**: the recipe → validate → draft-import pipeline, the visual system, the section spine, and the form, checkout, CTA, motion, and order-bump contracts. Load it whenever a funnel build has to produce page content. This document covers only the seam between a page and its funnel node.

## Two ways to fill a page node

|The user wants|Path|
|-|-|
|A ready-made Clickmax design, no authored content|`mcp__plugin_clickmax_clickmax__pages_templates_list` → `mcp__plugin_clickmax_clickmax__pages_create` with `templateId`|
|A page with real copy, sections, and a working form|The authoring pipeline in `clickmax-pages`, ending in `mcp__plugin_clickmax_clickmax__pages_import_html_draft`|
|A blank page to fill in by hand in the editor|`mcp__plugin_clickmax_clickmax__pages_create` with no `templateId` — only when the user explicitly asked for that|

`pages_create` on its own never passes through recipe validation, because there is no authored HTML to validate. Content written on the spot always goes through the draft import.

## Checkout page from a template

The fastest path when the user explicitly wants a ready-made Clickmax checkout rather than an authored one:

1. `mcp__plugin_clickmax_clickmax__pages_templates_list` with `type: ["checkout"]` — or `["sales"]` for a sales page that embeds a checkout. Pick one whose `canUse` is `true`.
2. `mcp__plugin_clickmax_clickmax__pages_create` with `type: "checkout"`, that `templateId`, the `offerId` to sell, plus `name`, `path`, `projectId`, and `funnelId`. The backend copies the template and **auto-binds the offer** into the page's checkout block — no manual editing.
3. `mcp__plugin_clickmax_clickmax__funnels_node_create` a `page` node, or reuse the existing checkout node, then `mcp__plugin_clickmax_clickmax__funnels_node_connect_page` with the returned `pageId`.
4. `mcp__plugin_clickmax_clickmax__funnels_triggers_connect`, then `mcp__plugin_clickmax_clickmax__funnels_validate`, then publish only when asked.

When the checkout should carry the funnel's own visual identity instead of the template's, author it through `clickmax-pages` with the `checkout` recipe and connect it the same way from step 3.

## Linking and routing

- Link with `mcp__plugin_clickmax_clickmax__funnels_node_connect_page` after the page exists. A page node with no page linked is a slot the visitor cannot reach.
- Route the node's outgoing edges with `mcp__plugin_clickmax_clickmax__funnels_triggers_connect`. Creating the node and stopping is an unfinished build.
- The `form_submit` trigger with no element bound already routes the whole page to the next node. Binding it to a specific element is only needed when the page has more than one form — and a page should not have more than one form.
- A CTA only becomes the "button click" exit trigger when the element carries the CTA marker. Styling never implies intent, so an unmarked button renders correctly and still leaves that trigger unavailable on the canvas.
- A checkout page that needs order bumps gets them after it is linked to its node. The call full-replaces the bump list.
- The offer bound to a page's checkout can be read and changed after the fact with `mcp__plugin_clickmax_clickmax__checkout_get` / `mcp__plugin_clickmax_clickmax__checkout_set` — no re-import needed just to fix an offer.

## A checkout is an offer's checkout

"Page with a checkout" means a page whose checkout block is bound to an `offerId`. Passing the offer at creation (template path) or at import (authored path) is what wires it; without an offer the block renders and charges nothing. One checkout per page — payment is resolved per page.

## Notes

- Visual fine-tuning — dragging blocks around — stays in the editor. The assistant creates the page from a template or authors its HTML, and wires the offer and the links.
- Re-importing against the same `pageId` replaces that page's content. That is how a page is corrected, without creating a second one.
