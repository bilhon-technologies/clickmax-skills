# External checkout on a page — Whop, Hotmart, Stripe and Pagar.me

A page can sell through the workspace's own **Whop**, **Hotmart**, **Stripe** or **Pagar.me** account instead of a Clickmax offer. The checkout is embedded in the page, the funnel moves on when the payment is approved, and the sale is recorded against the page.

## When to use it

|The user says|Checkout|
|-|-|
|"vende pela Whop", a Whop product/plan, `plan_…`|Whop|
|"checkout da Hotmart", a Hotmart product/offer, an offer code (`off`)|Hotmart|
|"checkout da Stripe", a Stripe product/price, `price_…`|Stripe|
|"checkout da Pagar.me", "checkout transparente da Pagar.me"|Pagar.me|
|A Clickmax product/offer, or no platform named|Native checkout (`offerId`) — see [components](components.md#checkout)|

Never mix them: one page has exactly one checkout, either a Clickmax offer or one external account.

Which platforms you may name: only these four, the ones Clickmax embeds. Offer one only when the user already has an active integration for it, or when the user asks about selling through another platform ("you can connect Whop, Hotmart, Stripe or Pagar.me if you prefer"). Never name or suggest other gateways (PayPal, Mercado Pago, PagSeguro, Kiwify, Eduzz…) as a page checkout — say they are not available and offer the native checkout or these four.

## Step 0 — the integration must exist

`mcp__plugin_clickmax_clickmax__external_checkout_accounts_list` with `platform: "whop" | "hotmart" | "stripe" | "pagarme"` is ALWAYS the first call, before writing any copy or HTML.

- `setup` is not null → **stop the build** and guide the setup, in the user's language:
  1. Say in one line what is missing: `not_connected` = no active integration; `incomplete` = the integration lacks the fields in `missingCredentials` (named exactly as the Integrations form shows, e.g. **Chave publicável**).
  2. Relay `setup.steps` in order as a short numbered list — what to click in Clickmax and what to do on the platform's site. Keep menu and field names exactly as given; do not invent or reorder steps.
  3. Close with one `cx-cta` `action="open-page"` whose `path` is `setup.openPath` (opens the platform's integration form) and the full guide link `setup.docsUrl`. Offer to continue the page once it is connected.
  - An existing integration cannot be edited: an `incomplete` one is fixed by creating it again with every field. Whop/Hotmart can also go on right away: ask for the offer's payment link instead of a raw code — Hotmart: `pay.hotmart.com/…?off=kjl7fk5t` → `offerCode` is the `off=` value; Whop: `whop.com/checkout/plan_…` → `planId` is the `plan_…` segment. Extract it yourself; never ask the user for a raw code.
- Never build a page without the checkout as a workaround, never invent an `incomingId`, and never switch to another platform or to a Clickmax offer without the user asking.
- What each platform needs to sell from a page:

|Platform|Fields in Integrações|Without them|
|-|-|-|
|Whop|**Token da API** (key with the Admin role)|Products/plans cannot be listed; the user can paste the plan's checkout link (`whop.com/checkout/plan_…`)|
|Hotmart|**Client ID** + **Client Secret** (production credential)|Products/offers cannot be listed; the user can paste the offer's payment link (`pay.hotmart.com/…?off=…`) — found in Hotmart under the product, **Precificação e ofertas**|
|Stripe|**Chave secreta ou restrita**, **Chave publicável**, **Segredo de assinatura do webhook**|The checkout does not charge — the import is rejected until all three are filled|
|Pagar.me|**Chave secreta (sk\_)**, **Chave pública (pk\_)**, **Senha do webhook**, plus the page domain released in Pagar.me (**Configurações > Conta > Domínios**)|The checkout does not charge — the import accepts the page with a warning, so treat it like Stripe: do not build until complete|

## Collect the choice from real data

One account → use it without asking. Several → ask which, by name. Then walk the catalog:

- Whop: `mcp__plugin_clickmax_clickmax__whop_products_list` → `mcp__plugin_clickmax_clickmax__whop_plans_list` (`productId`). Each plan is a price, one-time or recurring.
- Hotmart: `mcp__plugin_clickmax_clickmax__hotmart_products_list` → `mcp__plugin_clickmax_clickmax__hotmart_offers_list` (`productId`). `isMainOffer` marks the default.
- Stripe: `mcp__plugin_clickmax_clickmax__stripe_prices_list` — every active price with its `productName`; `type: "recurring"` + `interval` is a subscription.
- Pagar.me (transparent checkout, no product catalog) — never assume the kind:
  1. Call `mcp__plugin_clickmax_clickmax__pagarme_plans_list` FIRST (plans with `amount` in cents and `interval`).
  2. Ask in one question whether the page sells a **one-time product** or a **subscription**, listing the existing plans by name and price as the subscription options. No plans → say only one-time is available (plans are created in the Pagar.me dashboard).
  3. One-time → the product is defined inline from what the user tells you: name and price. Ask for any you do not have; never invent a price. Subscription → the chosen plan.

Several products/plans/offers/prices and the user did not name one → ask with names and prices as choices. Exactly one → use it and say so.

Whop/Hotmart/Stripe lists are cursor-paginated with no name search: keep calling with `cursor` = `meta.nextCursor` while `meta.hasNextPage` is true and the item is not found yet.

## Build the page

Same authoring pipeline and the same marker as any checkout — only the binding changes:

```html
<div data-cx-checkout></div>
```

Pass `externalCheckout` on `mcp__plugin_clickmax_clickmax__pages_validate_html` and `mcp__plugin_clickmax_clickmax__pages_import_html_draft` **instead of** `offerId`, copying the values from the list rows:

- Whop: `{ provider: "whop", incomingId, productId, planId, planLabel, price, currency }` — `planLabel` as `"<plan name> · <formattedPrice>"`.
- Hotmart: `{ provider: "hotmart", incomingId, productId, productLabel, offerCode, offerLabel, price, currency }` — `offerCode` is the offer's `code`.
- Stripe: `{ provider: "stripe", incomingId, priceId, priceLabel, price, currency, interval }` — `priceLabel` as `"<productName> · <price>"`, `interval` `null` for one-time.
- Pagar.me: `{ provider: "pagarme", incomingId, pagarmeOffer }` with `pagarmeOffer`:
  - one-time: `{ kind: "one_time", product: { code, name, amount } }`, where `code` is a short slug of the name (letters, digits, `._-`) and `amount` is in cents (R$ 97 = `9700`); optional `bumps` (same shape, max 5);
  - subscription: `{ kind: "subscription", plan: { id, name } }`, from `pagarme_plans_list`; value and period come from the plan.

  Optional, only when asked: `paymentMethods` (`credit_card`, `pix`, `boleto` — Pix is dropped on subscriptions), `maxInstallments` (1–12), `statementDescriptor` (≤13 chars), `buttonLabel`, `accentColor` (`#RRGGBB`), `fontFamily`. One-click purchase after an approved card sale on the same domain is on by default (`oneClick`).

Optional, only when the user asked:

|Whop|Hotmart|Stripe|
|-|-|-|
|`theme`: `light` or `dark`|`country` (BR, PT, US, ES, MX, AR, CO, CL, PE, FR, IT, DE, GB) — language, currency, payment methods|`optionalItems`: order bumps `[{ id, label, price, currency, interval }]`, max 10, same currency and recurrence as the main price|
|`locale`: `pt`, `en`, `es`, `fr`, `de`, `it`, …|`hideBillet`/`hidePix`/`hidePayPal`/`hideCoupon`: `true` hides it|`promotionCodes: true` — coupon field|
||`split`: max card installments, 1–12|`collectPhone: true` — asks the phone|

The `checkout` recipe accepts an external checkout as its checkout. An inactive account, one from another workspace, or a Stripe integration missing keys is rejected on import (a Pagar.me one missing keys is accepted with a warning but does not sell) — go back to Step 0 instead of retrying.

## What the user must know

- The checkout's look comes from the platform. Hotmart takes colors and fields from the product's checkout settings in Hotmart; Whop only takes the light/dark theme here; Stripe follows the account's branding in Stripe. Pagar.me is a transparent checkout drawn by Clickmax in Pagar.me's style (accent color, font and button text can be set).
- Whop, Stripe and Pagar.me tie each sale to the page when the page is **published**. Before publishing, Stripe and Pagar.me do not charge at all and Whop purchases are not attributed.
- Pagar.me: the card fails until the page's domain is released in Pagar.me (**Configurações > Conta > Domínios**), and each payment method must be active in the Pagar.me account.
- Revenue comes from the platform's webhook. Hotmart's "change country" button cannot be hidden; boleto and other async methods confirm only when the platform notifies, and the funnel does not advance on them.
- Order bumps and `checkout_set` from the native checkout do not apply; Stripe bumps go in `optionalItems`, Pagar.me bumps in `pagarmeOffer.bumps`.
- Fine-tuning (Whop accent/radius/custom colors) is done in the editor: select the block and use its side panel.

## Change the checkout later

To change a checkout already on a page (payment methods, installments, bumps, product/plan, coupon, theme…):

- Page open in the editor → mounted `editor` `codemode.updateExternalCheckout({ options })` (add `componentId` only when the page has more than one checkout block). It writes the block the same way its side panel does, saves, and the canvas shows it at once. Pass only the keys to change, e.g. "só Pix" on Pagar.me = `{ paymentMethods: ["pix"] }`; Hotmart "sem boleto" = `{ hideBillet: true }`; Stripe "com cupom" = `{ promotionCodes: true }`.
- Never re-import a page that is open in the editor to change its checkout: the mounted editor keeps the old content, reading it back shows no change, and its next save can overwrite the import.
- Page not open → re-import the same `pageId` with the new `externalCheckout`, or open it in the editor and use the method above.
- The account cannot be changed this way; switching account is done in the block's side panel.

## Inside a funnel

A funnel step that sells through Whop/Hotmart/Stripe/Pagar.me is an `external_checkout` node, not a `page` node — see the `clickmax-funnels` skill. Build the page with the SAME provider as the node's `config.provider`.
