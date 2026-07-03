# Funnel pages and checkout

A funnel `page` node is just a graph slot — it has no content until an editor3 page is created and linked. You CAN create that page from a Clickmax template (including a page that already has a checkout) and bind an offer to it, without manual visual editing.

## Create a page WITH a checkout (from a template) and attach it to a funnel

1. `pages_templates_list` with `type: ["checkout"]` (or `["sales"]` for a sales page that embeds a checkout). Pick a template whose `canUse` is `true`; keep its `id`.
2. `pages_create` with:
   - `type: "checkout"`
   - `templateId`: the template id from step 1
   - `offerId`: the offer to sell (resolve/create it first via the offers/products tools) — the backend copies the template and AUTO-BINDS this offer into the page's checkout block(s)
   - `name`, `path` (e.g. `checkout`), `projectId`, and `funnelId` (the funnel it belongs to)
3. `funnels_node_create` a `page` node (or reuse an existing checkout node), then `funnels_node_connect_page` with the new `pageId` to link the page to the node.
4. Connect the node's triggers (`funnels_triggers_connect`), `funnels_validate`, then publish when asked.

## Notes

- A checkout in Clickmax is an OFFER's checkout. "Page with a checkout" = a page whose checkout block is bound to an `offerId`. Passing `offerId` on `pages_create` (with `type: "checkout"` + a checkout `templateId`) is what wires it; without an offer the checkout block is unbound.
- For a sales page that needs a checkout/order-bump, use a `sales` template that includes a checkout and pass the `offerId` the same way; order-bump offers are configured on the offer side.
- Page types: `advertorial | capture | quiz | presell | sales | checkout | upsell | downsell | termsPrivacy | thankYou | vsl | webinar`.
- `pages_create` without a `templateId` makes a BLANK page — only use that when the user explicitly wants to start from scratch and design in the editor.
- Fine-grained visual design (moving blocks, copy) still happens in the page editor UI; the AI creates the page from a template and wires the offer/links.
