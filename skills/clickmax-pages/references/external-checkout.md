# External checkout on a page — Whop, Hotmart and Stripe

A page can sell through the workspace's own **Whop**, **Hotmart** or **Stripe** account instead of a Clickmax offer. The checkout is embedded in the page, the funnel moves on when the payment is approved, and the sale is recorded against the page.

## When to use it

|The user says|Checkout|
|-|-|
|"vende pela Whop", a Whop product/plan, `plan_…`|Whop|
|"checkout da Hotmart", a Hotmart product/offer, an offer code (`off`)|Hotmart|
|"checkout da Stripe", a Stripe product/price, `price_…`|Stripe|
|A Clickmax product/offer, or no platform named|Native checkout (`offerId`) — see [components](components.md#checkout)|

Never mix them: one page has exactly one checkout, either a Clickmax offer or one external account.

## Step 0 — the integration must exist

`mcp__plugin_clickmax_clickmax__external_checkout_accounts_list` with `platform: "whop" | "hotmart" | "stripe"` is ALWAYS the first call, before writing any copy or HTML.

- `setup` is not null → **stop the build** and relay `setup.instructions` in the user's language, with the `setup.docsUrl` link:
  - `not_connected` → no active integration. Tell the user to open **Integrações → Explorar Integrações → <platform>**, create it following the guide, and come back. Offer to continue right after.
  - `incomplete` → the integration exists but fields are empty (`missingCredentials` names them exactly as the form shows, e.g. **Chave publicável**, **Segredo de assinatura do webhook**). Tell the user which fields to fill in **Integrações → <their integration>**.
- Never build a page without the checkout as a workaround, never invent an `incomingId`, and never switch to another platform or to a Clickmax offer without the user asking.
- What each platform needs to sell from a page:

|Platform|Fields in Integrações|Without them|
|-|-|-|
|Whop|**Token da API** (key with the Admin role)|Products/plans cannot be listed; the user can paste the plan id (`plan_…`)|
|Hotmart|**Client ID** + **Client Secret** (production credential)|Products/offers cannot be listed; the user can paste the offer code (`off`)|
|Stripe|**Chave secreta ou restrita**, **Chave publicável**, **Segredo de assinatura do webhook**|The checkout does not charge — the import is rejected until all three are filled|

## Collect the choice from real data

One account → use it without asking. Several → ask which, by name. Then walk the catalog:

- Whop: `mcp__plugin_clickmax_clickmax__whop_products_list` → `mcp__plugin_clickmax_clickmax__whop_plans_list` (`productId`). Each plan is a price, one-time or recurring.
- Hotmart: `mcp__plugin_clickmax_clickmax__hotmart_products_list` → `mcp__plugin_clickmax_clickmax__hotmart_offers_list` (`productId`). `isMainOffer` marks the default.
- Stripe: `mcp__plugin_clickmax_clickmax__stripe_prices_list` — every active price with its `productName`; `type: "recurring"` + `interval` is a subscription.

Several products/plans/offers/prices and the user did not name one → ask with names and prices as choices. Exactly one → use it and say so.

The lists are cursor-paginated with no name search: keep calling with `cursor` = `meta.nextCursor` while `meta.hasNextPage` is true and the item is not found yet.

## Build the page

Same authoring pipeline and the same marker as any checkout — only the binding changes:

```html
<div data-cx-checkout></div>
```

Pass `externalCheckout` on `mcp__plugin_clickmax_clickmax__pages_validate_html` and `mcp__plugin_clickmax_clickmax__pages_import_html_draft` **instead of** `offerId`, copying the values from the list rows:

- Whop: `{ provider: "whop", incomingId, productId, planId, planLabel, price, currency }` — `planLabel` as `"<plan name> · <formattedPrice>"`.
- Hotmart: `{ provider: "hotmart", incomingId, productId, productLabel, offerCode, offerLabel, price, currency }` — `offerCode` is the offer's `code`.
- Stripe: `{ provider: "stripe", incomingId, priceId, priceLabel, price, currency, interval }` — `priceLabel` as `"<productName> · <price>"`, `interval` `null` for one-time.

Optional, only when the user asked:

|Whop|Hotmart|Stripe|
|-|-|-|
|`theme`: `light` or `dark`|`country` (BR, PT, US, ES, MX, AR, CO, CL, PE, FR, IT, DE, GB) — language, currency, payment methods|`optionalItems`: order bumps `[{ id, label, price, currency, interval }]`, max 10, same currency and recurrence as the main price|
|`locale`: `pt`, `en`, `es`, `fr`, `de`, `it`, …|`hideBillet`/`hidePix`/`hidePayPal`/`hideCoupon`: `true` hides it|`promotionCodes: true` — coupon field|
||`split`: max card installments, 1–12|`collectPhone: true` — asks the phone|

The `checkout` recipe accepts an external checkout as its checkout. An inactive account, one from another workspace, or a Stripe integration missing keys is rejected on import — go back to Step 0 instead of retrying.

## What the user must know

- The checkout's look comes from the platform. Hotmart takes colors and fields from the product's checkout settings in Hotmart; Whop only takes the light/dark theme here; Stripe follows the account's branding in Stripe.
- Whop and Stripe tie each sale to the page when the page is **published**. Before publishing, Stripe does not charge at all and Whop purchases are not attributed.
- Revenue comes from the platform's webhook. Hotmart's "change country" button cannot be hidden; boleto and other async methods confirm only when the platform notifies, and the funnel does not advance on them.
- Order bumps and `checkout_set` from the native checkout do not apply; Stripe bumps go in `optionalItems`.
- Fine-tuning (Whop accent/radius/custom colors) is done in the editor: select the block and use its side panel.

## Inside a funnel

A funnel step that sells through Whop/Hotmart/Stripe is an `external_checkout` node, not a `page` node — see the `clickmax-funnels` skill. Build the page with the SAME provider as the node's `config.provider`.
