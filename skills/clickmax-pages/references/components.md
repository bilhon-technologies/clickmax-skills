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

- **Urgency bar** at the very top, only when the deadline is real. Use the real countdown timer (see Interactive components below) — never fake the number as static text.
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

## Interactive components

Six components hydrate via the platform's runtime bundle, already inlined into every published page — same classes/attributes the visual editor emits, no `<script>` needed. `pages_validate_html` catches the common mistakes below as warnings, never hard errors.

### Countdown/timer

```html
<div class="cm-stopwatch" id="promo-timer" timer-type="countdown" data-end-date="2026-12-31" data-end-time="23:59">
  <div class="cm-stopwatch-item visible-block" data-unit="hours"><div class="cm-stopwatch-value">00</div></div>
  <div class="cm-stopwatch-item visible-block" data-unit="minutes"><div class="cm-stopwatch-value">00</div></div>
</div>
```

- `id` unique on the page — the runtime keys its instance map by it
- `timer-type="countdown"` (or omitted) needs `data-end-date="YYYY-MM-DD"` + `data-end-time="HH:MM"`, or it starts at zero/expired
- only units carrying `visible-block` are shown; the import derives the editor's `data-show-<unit>` flags from them, so the timer keeps working after the page is opened and saved in the visual editor
- `data-unit`: `hours` \| `minutes` \| `seconds` \| `days` \| `weeks` \| `months` \| `years`

### Accordion

```html
<div class="cm-accordion-item">
  <div class="cm-accordion-header">Do I need previous experience?</div>
  <div class="cm-accordion-content">No. The first session starts from zero.</div>
</div>
```

Toggled purely by class; header and content must share the same parent. No companion CSS needed.

### Popup

```html
<button class="cm-button-open-popup">See the bonus</button>

<div class="cm-popup not-visible">
  <div class="cm-popup-overlay"></div>
  <div class="cm-popup-modal">
    <div class="cm-popup-close-button">✕</div>
    <div class="bonus-card">...</div>
  </div>
</div>
```

```css
/* visual on the modal or a wrapper inside it — never on .cm-popup */
.cm-popup-modal {
  width: min(92vw, 480px);
  border-radius: 16px;
  background: #fff;
}
.cm-popup-overlay {
  background: rgba(0, 0, 0, 0.55);
} /* optional; keep it semi-transparent */
```

- contract: `div.cm-popup.not-visible > div.cm-popup-modal`, **only one `.cm-popup` per page** — a second one is silently inert
- **never style `.cm-popup`** — no CSS rule, no `style` attribute. The runtime already makes the root a transparent fixed full-screen layer; any background or size there covers the whole page when it opens (`pages_validate_html` warns `POPUP_ROOT_STYLED`)
- all visual styling goes on `.cm-popup-modal` or a wrapper inside it
- backdrop: optional `.cm-popup-overlay` before the modal, semi-transparent by default; if restyled, keep an `rgba` background
- keep `not-visible` on the root or it opens on load (skip only when `on-load="show"` is set on `.cm-popup-modal` on purpose)
- close: `.cm-popup-close-button` inside the modal
- optional attributes on `.cm-popup-modal`: `on-load="show"`, `on-see="#targetId"`, `when-click="#targetId"`, `close-page="show"`

### Order-bump toggle

```html
<input class="cm-checkmark-checkbox" type="checkbox" data-orderbump-id="<offerId>" />
```

Or `<button class="cm-checkmark-button" data-orderbump-id="<offerId>">`. `offerId` must match one of the ids passed to the checkout's order-bump list, or the toggle has nothing to attach to.

### Icon action

```html
<span class="cm-icon" action-type="open-link" action-link="https://example.com"></span>
```

`action-type` needs its matching attribute:

|`action-type`|Required attribute|
|-|-|
|`open-link`|`action-link`|
|`scroll-to`|`scroll-element` (an id)|
|`show-or-hide`|`data-show` and/or `data-hide` (comma-separated ids)|
|`open-popup` / `close-popup`|`popup-id`|
|`mark-complete`|`complete-element` (an id)|
|`nothing-happens`|none|

### Navigation

```html
<nav class="cm-navigation">
  <a class="cm-navigation-item" href="#pricing">Pricing</a>
</nav>
```

- in-page links: `href="#<id>"` pointing at an element with that exact, unique `id` (e.g. `<section id="pricing">`); the runtime scrolls to it smoothly, also inside the editor canvas and preview
- dropdown: `<div class="cm-has-dropdown">Label <div class="cm-submenu"><a class="cm-navigation-item" href="#faq">FAQ</a></div></div>`
- the editor turns these links into its own menu on the first load, so they stay editable in **Gerenciar Menu** and survive every save
- the mobile toggle button self-creates — never hand-write it

## Logo belt

Two identical strips side by side, the track translated by `calc(-100% - var(--gap))` so strip two ends exactly where strip one began. The strip must be wider than the viewport, or a gap opens at the end of the cycle. Wrap the animation in `prefers-reduced-motion` — when motion is reduced, show the first set statically.

## Video

### YouTube / Vimeo

An `<iframe>` from YouTube or Vimeo is the only plain embed that survives import. Give it a `title`, a bounded aspect ratio, and never autoplay with sound.

### VTurb

```html
<div data-cx-vturb="<videoId>"></div>
```

- server resolves the marker into the real `<vturb-smartplayer>`; the published page loads its player script on its own
- `videoId` must already exist as a VTurb player on this workspace
- a hand-written `<script>` for VTurb is always stripped — it never boots the player; the marker is the only working path

### Watch-gate (optional)

Reveal or hide other elements once a timer elapses. Works on a plain YouTube/Panda iframe or on a VTurb marker:

```html
<iframe data-provider="youtube" id="unique-id" data-video-id="..." data-timer="00:05:00" data-cx-show="ctaId"></iframe>
<div data-cx-vturb="<videoId>" data-vturb-timer="300" data-cx-show="ctaId" data-cx-hide="lockId"></div>
```

- `data-cx-show` / `data-cx-hide`: comma-separated element ids to reveal/hide once the timer elapses
- the iframe path additionally needs `data-provider="youtube|panda"` + `id` + `data-video-id` + `data-timer="HH:MM:SS"`
- the VTurb path additionally needs `data-vturb-timer="<seconds>"` on the same marker (plain seconds, e.g. `300` = 5 min); the marker stays as the player wrapper, so every attribute on it (id, class, gate) survives
- prefer `data-cx-show`/`data-cx-hide`; the legacy `component-to-show`/`component-to-hide` still work but are kept only for pages built by hand in the editor

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
