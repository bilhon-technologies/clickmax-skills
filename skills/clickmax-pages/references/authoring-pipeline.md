# Authoring pipeline

## The only path that produces a page with real content

`recipe → curated design → one manifest → generation context → author HTML → validate → draft import → (separate) publish`

Every step feeds the next with the **same** manifest object. Changing it mid-flight is the most common cause of a validation that passes and an import that fails.

1. `mcp__plugin_clickmax_clickmax__pages_recipes_list` — pick the one recipe compatible with the target page type.
2. `mcp__plugin_clickmax_clickmax__page_designs_list`, then `mcp__plugin_clickmax_clickmax__page_designs_get` on the chosen one. Mandatory, even with no style request from the user.
3. Build the manifest: `recipeId` plus the optional `designIdOrSlug` of the design chosen in step 2. Reuse it unchanged from here on.
4. `mcp__plugin_clickmax_clickmax__pages_generation_context_get` with the manifest and the target — returns the token CSS and token names the page must use.
5. Author one complete responsive HTML document.
6. `mcp__plugin_clickmax_clickmax__pages_validate_html` with the same target, manifest, HTML, and `offerId` when there is a checkout. Writes nothing.
7. `valid: false` → fix **every** entry in `errors` and revalidate. Read `warnings` too; they are actionable.
8. `mcp__plugin_clickmax_clickmax__pages_import_html_draft` with the exact validated artifact. Always lands unpublished.
9. Publication only after separate explicit consent, via `mcp__plugin_clickmax_clickmax__pages_publish`.

`mcp__plugin_clickmax_clickmax__pages_import_html` is the legacy-compatible entry point: it defaults to publishing and only enforces the recipe when a manifest is supplied. Prefer the draft import for anything generated.

## Target

Send **exactly one** of:

- `placement` — creates a new page. Needs the project, a unique name and path, and the page type.
- `pageId` — replaces the content of an existing page. This is how a page is corrected; it never creates a duplicate.

Both together is rejected. A re-import that should inherit a funnel's brand style uses `pageId` + `funnelId` — never a `placement` used only to carry the funnel, because its name, type, and path would be silently ignored.

A new-page import that fails partway is rolled back, so the same `path` can be retried.

## Recipes

|Recipe|Page types|Required sections|Optional sections|Lead capture|Checkout|CTA|
|-|-|-|-|-|-|-|
|`capture`|capture|`hero` `benefits` `lead-form`|`social-proof` `faq`|required|forbidden|required|
|`sales`|sales, advertorial|`hero` `problem` `solution` `offer` `cta`|`social-proof` `guarantee` `faq`|optional|optional|required|
|`checkout`|checkout|`offer-summary` `checkout`|`benefits` `guarantee` `faq`|optional|required|optional|
|`thank-you`|thankYou|`confirmation` `next-step`|`social-links` `support`|forbidden|forbidden|optional|
|`upsell`|upsell|`offer` `benefits` `accept-cta` `decline-cta`|`social-proof` `guarantee`|forbidden|optional|required|
|`downsell`|downsell|`offer` `benefits` `accept-cta` `decline-cta`|`comparison` `guarantee`|forbidden|optional|required|

Every required id must appear at least once as `data-cx-section="<id>"` on the section that plays that role. Optional sections use the same attribute when present.

Page types: `advertorial` · `capture` · `quiz` · `presell` · `sales` · `checkout` · `upsell` · `downsell` · `termsPrivacy` · `thankYou` · `vsl` · `webinar`.

`quiz` and `webinar` have no import path — build those through their own flows. The types with no recipe of their own (`presell`, `vsl`, `termsPrivacy`) are authored under the closest compatible recipe.

## Validation errors and what they mean

|Code|Fix|
|-|-|
|`REQUIRED_SECTION_MISSING`|Add `data-cx-section="<id>"` to the section that plays that role|
|`REQUIRED_LEAD_CAPTURE_MISSING`|The recipe needs a capture form that follows the form contract|
|`FORBIDDEN_LEAD_CAPTURE_PRESENT`|Remove the form — this page type must not capture|
|`REQUIRED_CHECKOUT_MISSING`|Add `<div data-cx-checkout></div>` and pass `offerId`|
|`FORBIDDEN_CHECKOUT_PRESENT`|Remove the checkout marker|
|`REQUIRED_CTA_MISSING`|Mark the primary action with `data-cx-cta`|
|`INVALID_LEAD_FORM`|The form violates the contract — fix the markup, do not try another tool|
|`INVALID_HTML`|The document did not parse|
|`RECIPE_PAGE_TYPE_MISMATCH`|Wrong recipe for this page type — change the recipe, not the HTML|
|`UNSUPPORTED_TYPE_FOR_IMPORT`|This page type has no import path|

The first eight are fixed by rewriting the HTML. The last two are fixed by changing the recipe or the target — rewriting the same HTML will fail again.

## Templates and clones

- `mcp__plugin_clickmax_clickmax__pages_templates_list` → a Clickmax-designed starting point. Filter by type; only use one whose `canUse` is true.
- `mcp__plugin_clickmax_clickmax__pages_create` with `templateId` copies that template into a new page. With `type: "checkout"` + a checkout template + `offerId`, the offer is bound into the checkout block automatically — no manual editing.
- `mcp__plugin_clickmax_clickmax__pages_clone` duplicates an existing page, content included. Use it for A/B variants of a page that already works.
- Duplicated variants that carry the same copy should be kept out of search indexing until they diverge — two URLs with identical text compete with each other.

## Configuration and publication

- `mcp__plugin_clickmax_clickmax__pages_update` — name, path, type.
- `mcp__plugin_clickmax_clickmax__pages_update_config` — page-level settings.
- `mcp__plugin_clickmax_clickmax__pages_publish` / `mcp__plugin_clickmax_clickmax__pages_unpublish` — always a separate, explicitly approved action.
- `mcp__plugin_clickmax_clickmax__checkout_get` — reads which offer this page's checkout is bound to (`null` when none).
- `mcp__plugin_clickmax_clickmax__checkout_set` — binds this page's checkout to an offer, or clears it with `offerId: null`. This is how the offer is corrected **after** the page exists, without re-importing; it also seeds a default checkout config. The offer must belong to the same workspace.
- `mcp__plugin_clickmax_clickmax__checkouts_set_order_bump` — order bumps for a checkout page.

### Order bumps

A bump **is** an offer — there is no separate bump entity, so what gets attached to a checkout page is a list of offer ids.

1. The bump offers must exist first: `mcp__plugin_clickmax_clickmax__products_create` for a new product plus its offer, or `mcp__plugin_clickmax_clickmax__offers_create` for an extra offer on a product that already exists. Keep the offer ids.
2. The checkout page must exist and, ideally, already be linked to wherever it belongs.
3. `mcp__plugin_clickmax_clickmax__checkouts_set_order_bump` with `pageId` + `orderBumpIds`. It **full-replaces** the list — send the complete set every call, and `[]` removes every bump. A non-empty list turns the bump section on automatically.
4. `mainOfferId` is optional: omit it to keep the offer bound at page creation, pass an id to switch the main product, or `null` to clear it.

Bumps are keyed by the page, so they are set after the page exists. Only the offer linkage changes — style and copy are untouched.

## One visual system per funnel

Decide the visual system once, on the first page of a funnel, and reuse the same CSS block on the following pages, changing only content. The server inherits brand tokens through the funnel, but it does not control layout — a different hero on page three leaves the funnel visually broken even with the right palette.
