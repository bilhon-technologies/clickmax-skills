# Visual system

## Step 1 — choose a curated design. Always.

`mcp__plugin_clickmax_clickmax__page_designs_list` then `mcp__plugin_clickmax_clickmax__page_designs_get`. This is not optional and not conditional on the user asking for a style.

- **No style request** → pick the curated design that best fits the product and audience. This is what makes a page read as designed instead of as default CSS.
- **A style request exists** (brand, reference URL, or something generic like "dark with a purple accent") → pick the closest **structural** match; dark-first vs light-first is the main filter. Then **adapt the request to the design, not the design to the request**: map what was asked onto the slots the design already has — its accent token, its documented decorative signature, its component set. "Dark with a purple accent" means picking a dark-first design and retinting its accent to purple. It does not mean bolting a new decorative element onto a design that has no such pattern.
- If the request conflicts with an explicit Do/Don't in the chosen design, satisfy the intent through the design's own system rather than breaking its rule.

Type scale, spacing, component patterns, container width, and decorative signature come from the chosen design **verbatim**. Freehanding those is what produces an unreadable type scale, missing gutters, and mismatched contrast.

### When the catalog is empty or nothing fits

Rare, and it is not a licence to inherit the project's neutral default. Build a deliberate fallback:

- Derive one scoped palette: background, elevated surface, chromatic accent, soft accent tint, ink, muted ink, hairline. Declare it as custom properties, not as repeated literals.
- Black, white, and gray alone are valid only when the brief asks for monochrome. Otherwise the primary CTA, the focus state, the section markers, and one signature detail all carry the chromatic accent.
- Pick **one** visual idea tied to the offer — a content calendar, a route, a framework, a scorecard, a stack, a timeline — and express it with light CSS geometry, numbered steps, or typographic composition when there is no real image. Never a generic gradient blob.
- Give sections distinct roles: split heroes, offset grids, tinted surfaces, oversized numerals, editorial dividers, asymmetric alignment. Never one narrow centered column with identical cards top to bottom.
- Keep one color mode and one system. Variety comes from composition, density, scale, and accent tint — never from an unrelated palette in one section.

## Step 2 — map the design onto the injected tokens

The server injects brand tokens before the page's own CSS. Use `var(--cx-...)` with a fallback; a raw hex outside the palette comes back in `warnings`.

|Token|Role|
|-|-|
|`--cx-color-background`|Page and section background|
|`--cx-color-headline`|Headings|
|`--cx-color-subheadline`|Standfirst and section subtitle|
|`--cx-color-content`|Body copy|
|`--cx-color-colored`|Primary action fill|
|`--cx-color-on-colored`|Label inside the primary action|
|`--cx-color-surface`|Card, form, offer box|
|`--cx-color-surfaceElevated`|Modal, highlighted block|
|`--cx-color-inkMuted`|Secondary text on a surface|
|`--cx-color-hairline`|Borders and dividers — never a raw `rgba()`, which disappears in dark mode|
|`--cx-color-link` · `--cx-color-icon` · `--cx-color-overlay`|Links, icons, overlays|
|`--cx-mode`|`light` or `dark` — what the main surface is|
|`--cx-font-headline` · `--cx-font-subheadline` · `--cx-font-content`|Type families|
|`--cx-font-weight-*` · `--cx-line-height-*`|Weight and leading|

```css
color: var(--cx-color-headline, #212529);
background-color: var(--cx-color-surface, #ffffff);
font-family: var(--cx-font-content, system-ui, sans-serif);
```

**The token set is deliberately small — it does not carry a neutral ramp.** A page built only from these tokens reads flat, because there is no step between "surface" and "background" to separate a card from the section it sits in. Declare the missing steps locally, derived from the chosen design's palette, and use them consistently.

## Step 3 — the style-guide trap

The project's style guide is a **fresh default** (commonly light mode, a blue/navy palette, a small type scale) until someone changes it, and no tool changes it.

So when the requested look conflicts with that default, `var(--cx-color-...)` alone is not enough — an inherited token resolves back to the default. In that case:

- Define **one** local, self-contained set of custom properties for background, headline, content, accent, surface, and hairline.
- Apply it **explicitly to every section**.
- Never mix "some elements read guide tokens, some are hardcoded" in the same page. That is exactly what produces the failure where the hand-styled sections look right and everything else reverts to generic blue, gray, and small text.

## Type

- **Every heading and paragraph declares its own `font-size` AND `font-weight`.** A bare `h1`/`h2`/`p` inherits the project default, which can be far smaller and lighter than intended. Inheriting the base silently shrinks the text, fits more words per line, and rewrites every line break — the defect is invisible in review and obvious in a screenshot.
- Take the scale from the chosen design. Without one: h1 40px · h2 26px · h3 18px · body 16–18px · support 13–14px; h1 drops to 30–32px on mobile.
- **Write line-height in percent or as a unitless ratio, never in px.** Percentage leading follows the font-size down on mobile on its own; a px leading does not, and the mismatch is what makes a rescaled headline look broken.
- Fix a subtitle's line breaks with an explicit desktop width plus `max-width: 100%` — never with `<br>`, which breaks at every other viewport.
- Negative letter-spacing on large display type only; body and UI text run at normal tracking.

## Color and surface rhythm

- **One color mode for the whole page.** If the request is dark, every section keeps a dark background. No section flips to the opposite mode "for variety".
- **Contrast comes from surface steps inside that one mode**, not from mode flips: a stage section, then a tonal section, then a card plane. Each section owns its own `background-color` — do not rely on one global background that sections punch holes in.
- When a genuinely inverted block is wanted (a dark hero on a light page), use the ready-made `.cx-invert` class on the `<section>`. It repaints heading, paragraph, list item, label, and link together. Do not hand-recolor inside an inverted block — that is how an illegible gray subtitle on a dark background gets made.
- `.cx-surface` and `.cx-surface-elevated` exist for cards and forms.
- **Every `p`, `li`, `label`, and `span` needs an explicit color checked against its actual section background.** An inherited default can resolve to a muted token that is invisible on some surfaces.
- One chromatic accent, plus one soft tint derived from it for backgrounds and markers. Black, white, and gray alone only when the brief asks for monochrome.
- A second accent reserved for a single meaning (urgency, a deadline, a status) keeps its signal only if nothing else uses it.

## Spacing, container, breakpoints

- Spacing in multiples of 8. Vertical rhythm between sections: 56–80px desktop, 40–56px mobile.
- Container width and gutters come from the chosen design's own layout section — not from a universal recipe. Applying the same max-width to every page is what makes different products look identical.
- Write the container in the hard-to-break form:

```css
max-width: 1120px; /* the chosen design's value */
margin-inline: auto;
padding-inline: clamp(20px, 5vw, 48px);
```

Avoid freehanding a multi-argument `min()` on `width` (for example `width: min(74rem, calc(100% - 2.5rem))`). One malformed argument makes the whole declaration invalid, the browser drops it **silently**, and the page renders edge-to-edge with zero gutter. If a multi-argument function is unavoidable, re-read it before shipping — an invalid `clamp()` does not error, it just vanishes.

- Breakpoints, when the chosen design does not specify its own:

|Range|Layout|
|-|-|
|≤430px|Single column, abbreviated labels, compressed persistent bars|
|≤809px|Single column, centered, reduced type scale|
|810–1199px|Centered composition — side-by-side hero splits are unsafe in this band|
|1200–1439px|Side by side, reduced composition|
|≥1440px|Side by side, full composition|

The 810–1439px band is where marketing pages break most often: a hero composition positioned for 1440px leaves the viewport and covers the copy. Treat it as its own design, not as a scaled-down desktop.

- **Mobile is a redesign, not a shrink**: stack columns, reorder only where meaning survives, simplify decoration, enlarge tap targets, and eliminate horizontal overflow.
- If a box is responsive, its contents must be too. Layered artwork should derive every measurement from one proportion variable; when only the outer box shrinks and inner layers keep desktop pixels, the artwork draws outside its box and overflows the viewport.

## Section rhythm

- Alternate composition: split, offset grid, tonal surface, oversized numeral, editorial divider. Never repeat identical centered cards from top to bottom.
- Three cards per row maximum; never five or more across.
- Alternate density deliberately — dense blocks for the offer and the schedule, quiet blocks for the explanation and the proof.
- Decoration must carry a concept tied to the offer (a calendar, a route, a framework, a scorecard, a stack, a sequence) built from light CSS geometry, numerals, or typographic composition. Avoid gradient blobs, glass effects, and complexity without hierarchy. A decorative element stays 24px clear of faces and focal points, sits below the content in stacking order, and uses `pointer-events: none`.

## CSS discipline

- Scope selectors to the section's own id or class. A style-only request changes declarations, never the markup.
- Prefer custom properties within the generated scope over repeated literals.
- No universal resets, no broad element selectors that reach outside the page, no external font imports.
- Preserve unrelated global CSS and assets.
