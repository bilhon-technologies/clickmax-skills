# Components and contracts

`<script>` is stripped on import. Everything below is CSS-only or a declarative contract the runtime honors.

## Marker contracts

|Marker|What it does|
|-|-|
|`data-cx-section="<id>"`|Names a section so the recipe's structure can be verified. Every required id must appear at least once|
|`data-cx-cta`|Marks the link or button that advances the funnel. Without it the funnel shows "button click" as unavailable on this page even though the CTA renders fine — styling never implies intent|
|`data-cx-ingest-form`|Makes the runtime take over the form and record the lead|
|`data-cx-field="..."`|Maps an input to a CRM field|
|`data-cx-checkout`|Marks where the real checkout block is inserted|
|`data-cx-motion="..."`|Declares a safe entrance animation|

## Capture form

```html
<form data-cx-ingest-form id="lead-form">
  <label for="email">Your best email</label>
  <input id="email" type="email" name="email" data-cx-field="email" required />
  <button type="submit">I want the 15-minute routine</button>
</form>
```

Contract:

- exactly one `<button type="submit">`
- input types limited to `checkbox`, `email`, `hidden`, `tel`, `text` (single-select only)
- no `action`, `method`, or `formaction` attributes
- at least one field marked `data-cx-field="email"` or `data-cx-field="telephone"` (or `name="email"` / `name="telephone"`)
- other supported keys: `data-cx-field="name"` and `data-cx-field="lgpdApproved"`; any other key becomes a CRM custom field
- **one form per page** — two forms record the lead twice and make the funnel's form trigger ambiguous

A plain `<form>` without the marker is rejected. Fix the markup; do not reach for a different tool.

Validation is native and costs nothing: `required`, `minlength`, and an email `pattern` that demands a top-level domain, with the field's `title` carrying the message shown. `type="email"` on its own accepts `ana@wrong`.

A real `<label for="...">` above every field. A placeholder is not a label — it disappears the moment someone types.

## Checkout

```html
<div data-cx-checkout></div>
```

Pass `offerId` on the same import. The server replaces the marker with the real checkout block and binds the offer.

- Never hand-write the payment markup. The runtime looks for specific ids, and a mistake there breaks charging with no visible symptom.
- One checkout per page.
- `offerId` with no marker, or a marker with no `offerId`, both come back as warnings — the block exists but nothing charges, or the offer is bound with nowhere to pay.
- Order bumps are attached after the page exists, and the call full-replaces the list.

## FAQ — `<details>`, never JavaScript

```html
<section data-cx-section="faq">
  <details>
    <summary>Do I need previous experience?</summary>
    <p>No. The first session starts from zero.</p>
  </details>
</section>
```

Native, keyboard-accessible, survives import. Style `summary` with a visible focus state and a rotation marker driven by `details[open]`.

## Modal capture — `:target`, never JavaScript

Use only when there is a value exchange **before** the signup — the person wrote an idea, took a diagnostic, made a choice — so the micro-commitment is already made. For an ordinary lead magnet, the inline form converts better.

```html
<a class="cx-cta" data-cx-cta href="#capture">I want in</a>

<div class="cx-modal" id="capture">
  <a class="cx-modal__backdrop" href="#" aria-label="Close"></a>
  <div class="cx-modal__card" role="dialog" aria-modal="true" aria-labelledby="capture-title">
    <h2 id="capture-title">One step left</h2>
    <!-- the single capture form goes here -->
  </div>
</div>
```

```css
.cx-modal {
  display: none;
}
.cx-modal:target {
  display: flex;
  align-items: center;
  justify-content: center;
  position: fixed;
  inset: 0;
  z-index: 50;
}
.cx-modal__backdrop {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.6);
}
.cx-modal__card {
  position: relative;
  width: min(92vw, 420px);
  border-radius: 14px;
  padding: 28px;
  background-color: var(--cx-color-surfaceElevated, #fff);
}
```

The form stays in the DOM from load, so capture and the funnel's form trigger work normally. Still one form per page, modal included.

## Section header

The most reusable block in a long page. Three parts, same order everywhere: a pill badge with the section label, the headline, and a subtitle whose desktop width fixes its line breaks. Repeating it verbatim across sections is what gives a long page a spine; redesigning the heading treatment per section is what makes it read as assembled from parts.

## Persistent conversion furniture

- **Urgency bar** at the very top, only when the deadline is real. Static text, never a fake countdown — a counter needs JavaScript and would not survive import anyway.
- **Fixed action bar** at the bottom with the price and the same CTA label as the hero. Compresses to label + CTA on narrow screens.
- **Progress meter** for a limited batch:

```html
<div
  role="progressbar"
  aria-valuenow="57"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-label="Spots taken in the current batch"
>
  <span style="width: 57%"></span>
</div>
```

The number must be real. A fabricated scarcity meter is the fastest way to lose a buyer who comes back the next day and sees the same percentage.

## Logo belt

Two identical strips side by side, the track translated by `calc(-100% - var(--gap))` so strip two ends exactly where strip one began. The strip must be wider than the viewport, or a gap opens at the end of the cycle. Wrap the animation in `prefers-reduced-motion` — when motion is reduced, show the first set statically.

## Video

An `<iframe>` from YouTube or Vimeo is the only embed that survives import. Give it a `title`, a bounded aspect ratio, and never autoplay with sound.

## Images

- Use an asset the user supplied, or search a real, license-clear stock photo with `mcp__plugin_clickmax_clickmax__pages_images_search`. Never hand-write or guess a photo URL — a broken image in production is worse than a section with none.
- Hotlink the returned URL **verbatim** in `<img src>`: never re-host it, never edit its path or query.
- Search with concrete visual terms in the page's language ("mulher treinando academia"), never marketing abstractions ("sucesso").
- The `openverse` provider requires rendering `credit.author` and `credit.license` on the page. `picsum` returns random placeholders and is only acceptable when the user explicitly asked for placeholders.
- **Silent-fallback trap:** when the stock provider's key is missing or its quota is spent, the server answers with random Lorem Picsum photos instead of failing. If every result carries a Lorem Picsum credit while you asked for a topical search, those images are unrelated to the query — ship the section without a photo or ask for a real asset, and never present them as topical.
- Meaningful `alt` on informative images; empty `alt` plus `aria-hidden="true"` on decorative artwork.
- `sizes` must match the real slot width. A `sizes` smaller than the slot is the most common cause of a blurry page: the browser downloads the small variant and stretches it. The source needs at least twice the slot width.
- Never `loading="lazy"` on an image that enters the viewport by animation. The off-screen copies sit outside the lazy trigger's reach and render as empty boxes mid-cycle — the symptom looks exactly like a broken animation.
- Never bake copy into an image. It cannot be translated, searched, read aloud, or edited.

## Motion

```html
<h1 data-cx-motion="fade-up" data-cx-motion-duration="450" data-cx-motion-delay="80">…</h1>
```

- Presets: `fade-in` · `fade-up` · `slide-left` · `slide-right` · `scale-in`
- `data-cx-motion-duration` and `data-cx-motion-delay` are optional, in milliseconds
- Runs once on entering the viewport, respects reduced-motion, and leaves content visible when unsupported
- **Text is born visible.** Never write a custom reveal toggle; without JavaScript it would hide the content permanently
- Animate a few focal elements, not every block

## Accessibility floor

- One `h1`; heading levels in order after it, with no skipped levels.
- Every decorative element gets `aria-hidden="true"`.
- Visible keyboard focus on every interactive element — an outline with an offset, in a color legible against both the light and dark surfaces used on the page.
- Touch targets 48×48px minimum with 8px of clearance.
- Links and buttons describe their destination or action; never use a non-interactive element as a control.
- Contrast is checked against the section's real background, and information is never carried by color alone.
- `lang` on the document matches the copy's language.
- Anything that moves is wrapped in `prefers-reduced-motion`.
