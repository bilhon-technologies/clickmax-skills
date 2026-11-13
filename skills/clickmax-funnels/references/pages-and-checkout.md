# Funnel pages and checkout

A funnel `page` node is only a graph slot — it has no content until a page is created and linked to it.

**How a page is authored is in the `clickmax-pages` skill**: the recipe → validate → draft-import pipeline, the visual system, the section spine, and the form, checkout, CTA, motion, and order-bump contracts. Load it whenever a funnel build has to produce page content. This document covers only the seam between a page and its funnel node.

## Two ways to fill a page node

|The user wants|Path|
|-|-|
|Authored content while Max Design is available in the workspace|[Max Design site pages](#max-design-site-pages) (default over the draft import below)|
|A ready-made Clickmax design, no authored content|`mcp__plugin_clickmax_clickmax__pages_templates_list` → `mcp__plugin_clickmax_clickmax__pages_create` with `templateId`|
|A page with real copy, sections, and a working form|The authoring pipeline in `clickmax-pages`, ending in `mcp__plugin_clickmax_clickmax__pages_import_html_draft`|
|A blank page to fill in by hand in the editor|`mcp__plugin_clickmax_clickmax__pages_create` with no `templateId` — only when the user explicitly asked for that|
|A route of a Max Design site (code, not builder)|[Max Design site pages](#max-design-site-pages)|

`pages_create` on its own never passes through recipe validation, because there is no authored HTML to validate. Content written on the spot always goes through the draft import.

## Checkout page from a template

The fastest path when the user explicitly wants a ready-made Clickmax checkout rather than an authored one:

1. `mcp__plugin_clickmax_clickmax__pages_templates_list` with `type: ["checkout"]` — or `["sales"]` for a sales page that embeds a checkout. Pick one whose `canUse` is `true`.
2. `mcp__plugin_clickmax_clickmax__pages_create` with `type: "checkout"`, that `templateId`, the `offerId` to sell, plus `name`, `path`, `projectId`, and `funnelId`. The backend copies the template and **auto-binds the offer** into the page's checkout block — no manual editing.
3. `mcp__plugin_clickmax_clickmax__funnels_nodes_create` a `page` node, or reuse the existing checkout node, then `mcp__plugin_clickmax_clickmax__funnels_nodes_connect_page` with the returned `pageId`.
4. `mcp__plugin_clickmax_clickmax__funnels_triggers_connect`, then `mcp__plugin_clickmax_clickmax__funnels_validate`, then publish only when asked.

When the checkout should carry the funnel's own visual identity instead of the template's, author it through `clickmax-pages` with the `checkout` recipe and connect it the same way from step 3.

## Whop, Hotmart, Stripe, Pagar.me or PagBank checkout step

When the step sells through the workspace's Whop, Hotmart, Stripe, Pagar.me or PagBank account (not a Clickmax offer):

0. `mcp__plugin_clickmax_clickmax__external_checkout_accounts_list` for that platform FIRST. `setup` not null → relay `setup.instructions` (connect or complete the integration in Integrações) and stop: no node, no page.
1. `mcp__plugin_clickmax_clickmax__funnels_nodes_create` with `type: "external_checkout"` and `config: { provider: "whop" | "hotmart" | "stripe" | "pagarme" | "pagbank" }` — always pass the provider. Omit `triggers`: the node gets `purchase_approved` (+ accept fallbacks). Whop/Hotmart/Stripe never emit `purchase_declined`, `cart_abandoned`, pix or boleto; Pagar.me and PagBank can also take `card_declined`, `pix_generated` and `boleto_generated` (add them with `mcp__plugin_clickmax_clickmax__funnels_nodes_triggers_update` when the user wants those paths). Its triggers cannot go to a `workflow` node.
2. Author the page in `clickmax-pages` (its external-checkout reference): catalog walk → `mcp__plugin_clickmax_clickmax__pages_import_html_draft` with `externalCheckout` of the SAME provider and the funnel's `projectId`/`funnelId`.
3. `mcp__plugin_clickmax_clickmax__funnels_nodes_connect_page` on that node, then route `purchase_approved` to the next step (thank-you/upsell) with `mcp__plugin_clickmax_clickmax__funnels_triggers_connect`.
4. Tell the user Whop, Stripe, Pagar.me and PagBank sales are tied to the page once the funnel/page is published (Stripe, Pagar.me and PagBank do not charge before that); revenue shows in the funnel's results.

Never fall back to a page node without checkout, or to another platform, when the integration is missing.

## Max Design site pages

When the funnel's pages are routes of a Max Design site. Default for a NEW funnel whose pages are authored on the spot whenever Max Design is available in the workspace (the client has the Max Design project tools); the builder draft import (`mcp__plugin_clickmax_clickmax__pages_import_html_draft`) is then only for the user explicitly asking for a builder page:

1. Create the funnel and its nodes first, one `page` node per route, `triggers` omitted. A route's triggers come from its code, never from the node: never give a route node `button_click`/`link_click`/`form_submit` (the runtime has no button marker, so the funnel flags them as missing from the page and blocks publishing). Linking the route's page (step 4) removes the node-type defaults a site never emits (scroll, time on page, button/link click, cart abandoned), connected or not, and lists them in `removedTriggers`: never connect those before linking. A CTA click the funnel should count (e.g. "Comprar") = `<CxTrigger id="clicou-comprar" label="Comprar">` fired in that button's `onClick` (`max-design-components` → triggers reference); after publish it appears on the node by its `label`, and that is the trigger to connect.
2. Build every route of the funnel in ONE Max Design project that belongs to the funnel's Clickmax project (`projectId`): `mcp__plugin_clickmax_clickmax__funnels_nodes_connect_page` refuses a page from another project. Project = the one the user names; otherwise create one for this funnel, turn design mode on for it (the build resumes when design mode is ready) and reuse it for every page. Design each route with `max-design-page-craft` instead of the `clickmax-pages` pipeline; content-discovery and one-design-per-funnel rules still apply. A/B version B = its own route (e.g. `/v2`) on its own page node, never `mcp__plugin_clickmax_clickmax__pages_clone`. Capture → thank-you = `data-cx-redirect-url` on the capture form (step 4).
3. `design_checkpoint` returns `routePages`: one draft Clickmax page per static route (`published: false` until the site goes live; dynamic routes get theirs only on publish). Draft build (no publish asked) still links them: the funnel shows those steps as draft and publishing the site turns the same pages live, no relinking.
4. `mcp__plugin_clickmax_clickmax__funnels_nodes_connect_page` each route's page on its OWN node, then validate as below. Trigger arrows wait for the publication that lists the route's triggers (`mcp__plugin_clickmax_clickmax__funnels_structure_get` after activation), then connect them. Publish the site only when the user approves it. Outgoing arrows from these pages are diagram-only: navigation belongs to the site's `<Link>`, `navigate`, handlers or `data-cx-redirect-url` after capture. CTA markers and `form_submit` do not navigate by themselves. Never `mcp__plugin_clickmax_clickmax__pages_create` or `mcp__plugin_clickmax_clickmax__pages_create_external` for a route.
5. Checkout INSIDE the site = the default for a Max Design funnel that sells (sales → checkout → upsell → downsell → thank-you in one site), when the project's plugin has `runtime/checkout` (design mode: `max-design-components` → checkout reference). Per selling route:
   - Node: `mcp__plugin_clickmax_clickmax__funnels_nodes_create` `type: "page"` with `pageType` `checkout`, `upsell` or `downsell` matching the route's component, `triggers` omitted.
   - Offer: each selling route sells its own offer, named in the site's code by the offer `hash` (route loader). Publishing records that offer on the route's page and its funnel node (automation scope); an offer from another account fails the publish (`invalid_capabilities`). Selling another offer = new hash in the code + republish; `mcp__plugin_clickmax_clickmax__checkouts_set` and funnel-side product changes do not apply to these routes. No offer yet → write the route without selling and report the missing offer; the hash goes in once it exists.
   - Order bumps: declared in the same loader (`bumps`: bump offer hashes, `required` or optional, `bumpTitle`), never with the funnel-side bump call below, which does not apply to these routes. Bumps must be active offers of the same account: a refused bump fails the build and the publish. Changing bumps = code change + republish.
   - One click: upsell/downsell routes load with `mode: "upsell"`/`"downsell"` in the code; without it the one-click interface stays off. One click needs the card saved by the previous purchase, so Pix/boleto buyers fill the form.
   - Triggers the route emits: checkout `purchase_approved`, `purchase_declined`, `card_declined`, `pix_generated`, `boleto_generated` (the default node set lacks `card_declined`/`pix_generated`: add them with `mcp__plugin_clickmax_clickmax__funnels_nodes_triggers_update` when the user wants those paths; its `cart_abandoned` is removed when the route's page is linked, since a site never emits it); upsell `upsell_accepted`/`upsell_declined`; downsell `downsell_accepted`/`downsell_declined`. At most one purchase component per route.
   - Arrows (`purchase_approved` → upsell, `upsell_declined` → downsell, accepted/declined → thank-you) are the diagram of what the site's callbacks already do.
   - Automations ("comprou", "recusou", pix/boleto reminders): workflow node with its flow linked first (`mcp__plugin_clickmax_clickmax__funnels_workflow_flow_set`), then `mcp__plugin_clickmax_clickmax__funnels_triggers_connect` the route trigger into it; the trigger keeps its page arrow. They fire only for purchases of the route's offer. `upsell_declined`/`downsell_declined` count in the funnel but start NO automation: never connect them to a workflow; say so when the user asks for one.
   - Price change of the same offer or of a bump offer: the site republishes by itself after the user confirms; nothing to change in code or funnel.

   Checkout outside the site = the user prefers it, the plugin has no `runtime/checkout`, or the step sells through Whop/Hotmart/Stripe/Pagar.me/PagBank: the checkout stays a Clickmax checkout page (above) or an external checkout step, and the site's purchase buttons link to its URL.

## Linking and routing

- Link with `mcp__plugin_clickmax_clickmax__funnels_nodes_connect_page` after the page exists. A page node with no page linked is a slot the visitor cannot reach.
- Route the node's outgoing edges with `mcp__plugin_clickmax_clickmax__funnels_triggers_connect`. Creating the node and stopping is an unfinished build.
- On classic editor and external pages using legacy SDK routing, `form_submit` with no element bound can route the page to the next node. Binding a specific element selects one form when several exist; prefer one capture form per page. Max Design navigation follows the site code, not these edges (above).
- A CTA only becomes the "button click" exit trigger when the element carries the CTA marker. Styling never implies intent, so an unmarked button renders correctly and still leaves that trigger unavailable on the canvas.
- A Clickmax checkout page (outside a Max Design site) that needs order bumps gets them after it is linked to its node. The call full-replaces the bump list.
- The offer bound to a page's checkout can be read and changed after the fact with `mcp__plugin_clickmax_clickmax__checkouts_get` / `mcp__plugin_clickmax_clickmax__checkouts_set` — no re-import needed just to fix an offer.

## A checkout is an offer's checkout

"Page with a checkout" means a page whose checkout block is bound to an `offerId` — or, for a Whop/Hotmart/Stripe/Pagar.me/PagBank step, to the account's plan/offer/price via `externalCheckout`. Passing the offer at creation (template path) or at import (authored path) is what wires it; without an offer the block renders and charges nothing. One checkout per page — payment is resolved per page.

## Notes

- Visual fine-tuning — dragging blocks around — stays in the editor. The assistant creates the page from a template or authors its HTML, and wires the offer and the links.
- Re-importing against the same `pageId` replaces that page's content. That is how a page is corrected, without creating a second one.
