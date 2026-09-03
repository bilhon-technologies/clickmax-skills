# Build a quiz: steps, blocks, branching, checkout

Source of truth = the live quiz schema accepted by `mcp__plugin_clickmax_clickmax__forms_*` connector operations. This
reference documents the EXACT product shape so you never guess a block/step field — guessing is what
makes `mcp__plugin_clickmax_clickmax__forms_step_upsert` reject and loop.

## Terminating build flow (anti-loop)

Build a quiz INCREMENTALLY on top of the server-generated seed. Never re-emit a whole hand-authored
schema and re-try on rejection.

1. `mcp__plugin_clickmax_clickmax__forms_create` with `{ name, kind: 'quiz' }` and **NO `schema`** → returns a complete
   valid seed (valid `theme`/`settings` + one step: a `text` block + an `options` block). This is the
   only reliable way to get a valid `theme`/`settings` — do not author them by hand.
2. `mcp__plugin_clickmax_clickmax__forms_get` by the returned `id` → read `schema.steps[].id`, `schema.theme`,
   `schema.settings`, and `updatedAt`. Real step/block ids come from here; you need them for
   `goto`/`next`/`displayRule` targets.
3. Add or edit one step at a time with `mcp__plugin_clickmax_clickmax__forms_step_upsert`:
   - omit `stepId` → creates a step (optional `index` sets position; default append).
   - pass `stepId` → edits that step; `blocks` REPLACES that one step's block list (send the full
     list for that step, not the whole quiz). `name`/`next` optional.
   - the server merges into the freshest saved quiz, so untouched steps are never dropped.
   - pass `expectedUpdatedAt` (from step 2) for lost-update protection.
4. Wholesale theme/settings/qualification edits → `mcp__plugin_clickmax_clickmax__forms_replace_schema` (entire document).
   Prefer step-upsert for structure; use replace ONLY for global fields.
5. Publish with `mcp__plugin_clickmax_clickmax__forms_update` `{ status: 'active' }`. `paused` takes it offline.
   `mcp__plugin_clickmax_clickmax__forms_update` CANNOT touch `schema`, so it can never clobber quiz content.

Rule: one `forms_step_upsert` per step, built on the seed. If a call is rejected, FIX that one
payload against the shapes below — do NOT recreate the quiz or re-emit the whole schema. A second
`forms_create` after an error just leaves a duplicate half-built quiz.

## Step shape

`QuizStep` = `{ id, order, name?, blocks: Block[], next? }`. A step has NO `kind`; its `blocks[]`
define both what renders and how it branches. `next` = default-next step id when no
option/button/loading target applies (linear fall-through). Every block shares
`{ id, displayRule? }` (`displayRule` gates visibility, see Conditions).

`forms_step_upsert` body = `{ stepId?, name?, next?, blocks?, index?, expectedUpdatedAt? }`. It only
carries step-level fields — theme/settings/qualification/formulas are NOT settable here (use
`forms_replace_schema`).

## Blocks that drive branching + capture + checkout

Discriminated union by `type`. Full set (`BLOCK_TYPES`): `text, description, image, video, spacer,
options, scale, number, currency, measure, button, capture, alert, offer, loading, metrics, results,
testimonials, beforeAfter, faq, checklist, pricing, resultChart`. `video` = `{ url, title?,
autoplay? }` — `url` accepts a YouTube/Vimeo page url (rendered as embed) or a direct video file
url; autoplay always starts muted. The load-bearing ones:

### `options` — the question that branches + scores

```json
{
  "id": "b1",
  "type": "options",
  "title": "Qual seu nível?",
  "options": [
    { "id": "o1", "label": "Iniciante", "score": 1, "goto": "step-iniciante" },
    { "id": "o2", "label": "Avançado", "score": 5, "goto": "step-avancado" }
  ],
  "layout": "list",
  "required": true
}
```

- `option.score` (number) → added to the pseudo-field `$score` when chosen.
- `option.goto` (step id) → jumps straight to that step when chosen (per-answer branch).
- `option.value` defaults to `id`; `imageUrl` optional.
- `multiple: true` → multi-select; then advancing REQUIRES a `button` block (selecting doesn't
  auto-advance). `redirectOnButtonOnly: true` → same (selection never auto-advances; a `button` must).
- `layout`: `list | grid2 | grid3`; `disposition`: `text | image-text | image-cover`.
- `mapping` (optional) → writes the chosen value to CRM (see FieldMapping below).

### `scale` — numeric rating/agreement input (adds to `$score`)

```json
{
  "id": "b7",
  "type": "scale",
  "title": "De 1 a 10, o quanto você recomendaria?",
  "min": 1,
  "max": 10,
  "minLabel": "Nada provável",
  "maxLabel": "Extremamente provável",
  "required": true
}
```

- `min` defaults 1, `max` defaults 5. **Never author `max` > 10** — above 10 steps the button row
  wraps/squeezes on mobile (FLOWS-1034). If the user asks for a wider range ("escala de 1 a 20"), cap
  it at 10 and say so — don't silently comply or loop asking. Existing quizzes may already have
  `max` up to 20 from before this cap; leave those alone unless you're editing that exact block.
- Chosen value (1..max) sums into `$score`, same as `option.score` (see Branching by answer).

### `button` — explicit navigation

```json
{
  "id": "b2",
  "type": "button",
  "label": "Ver meu resultado",
  "action": "goto",
  "target": "step-resultado",
  "variant": "solid"
}
```

- `action`: `next` (linear), `goto` (jump to step id in `target`), `url` (redirect to `target`).
- `variant`: `solid | outline | ghost | link`. Needed to advance after a `multiple`/`redirectOnButtonOnly` options block.

### `loading` — score-gated interstitial that branches

```json
{
  "id": "b3",
  "type": "loading",
  "label": "Analisando…",
  "durationMs": 2500,
  "goto": "step-avancado",
  "displayRule": { "when": [{ "field": "$score", "op": "gte", "value": 5 }] }
}
```

- Renders only if its `displayRule` matches (typically over `$score`); on completion navigates to
  `goto` (or the step's linear `next`). Put several loading blocks in one "router" step, each with a
  different `displayRule` + `goto`, to fan out by score.

### `capture` — collect name/email/phone (lead)

```json
{
  "id": "b4",
  "type": "capture",
  "title": "Pra onde envio o resultado?",
  "fields": [
    {
      "id": "cf1",
      "fieldType": "text",
      "label": "Nome",
      "required": true,
      "mapping": { "scope": "lead", "target": "name" }
    },
    {
      "id": "cf2",
      "fieldType": "email",
      "label": "Email",
      "required": true,
      "mapping": { "scope": "lead", "target": "email" }
    },
    {
      "id": "cf3",
      "fieldType": "phone",
      "label": "Telefone",
      "required": false,
      "mapping": { "scope": "lead", "target": "telephone" }
    }
  ],
  "submitLabel": "Enviar"
}
```

Native lead targets = `name` / `email` / `telephone`. The seed's `capture` block already carries
exactly these three — reuse them.

### `offer` — the checkout CTA in a result step

```json
{
  "id": "b5",
  "type": "offer",
  "offerId": "<real-offer-id>",
  "productName": "Plano Avançado",
  "priceLabel": "R$ 497,00",
  "checkoutUrl": "https://<real-checkout-url>",
  "ctaLabel": "Comprar agora"
}
```

- `checkoutUrl` powers the CTA button — it is what sends the visitor to checkout at the end.
- `offerId` references a real `products` offer (display fields are a builder snapshot; the public
  renderer has no products API). `ctaLabel`/`productName`/`priceLabel`/`imageUrl` optional display.
- NOT to be confused with `pricing` (static display card; no real offer; `ctaUrl` optional).

### `results` — result summary (checkmarked benefits), no checkout

```json
{
  "id": "b6",
  "type": "results",
  "title": "Seu plano",
  "items": [{ "id": "r1", "title": "Resultado em 30 dias", "description": "Sob medida pra você." }]
}
```

### FieldMapping (used by `capture` fields and `options.mapping`)

`{ scope: 'lead' | 'custom' | 'opportunity' | 'none', target?: string }`. `target` = native field key
(`name`/`email`/`telephone`), a custom-field id (`cf.<uuid>`), or an opportunity field key. Use
`scope: 'lead'` for the standard name/email/phone capture.

## Branching by answer

`$score` is the accumulator. Three ways to branch, all resolvable to a step id:

1. `option.score` sums into `$score`; a later `loading.displayRule` (or a block `displayRule`) reads
   it via a Condition, e.g. `{ field: '$score', op: 'gte', value: 5 }`, and its `goto` routes.
2. `option.goto` → immediate per-answer jump (no score needed).
3. `button` with `action: 'goto'` + `target`.

Fall-through when nothing applies = the step's `next`. `scale` inputs also add their chosen value to
`$score`; `number`/`currency`/`measure` do NOT affect `$score`.

## Conditions (for displayRule + qualification)

`Condition` = `{ field, op, value? }`. `field` = a block id (a previous answer) or the pseudo-field
`$score`. `op` = `eq | neq | contains | in | gt | gte | lt | lte | exists` (`value` unused for
`exists`). `DisplayRule` = `{ when: Condition[] }` — a block renders only when ALL `when` match;
absent/empty ⇒ always render.

## Checkout at the end (the case that loops)

A result step holds an `offer` block whose `checkoutUrl` (+ `offerId`, `ctaLabel`) sends the visitor
to checkout. To route "advanced vs beginner checkout by answer":

1. Two result steps (e.g. `step-avancado`, `step-iniciante`), each with its own `offer` block pointing
   at the correct offer's checkout URL.
2. Reach them by branching: `option.goto` / `button goto`, or by `option.score` → `loading.displayRule`
   over `$score` → `goto`.

There is NO magic "checkout node" — checkout is just an `offer` block with a real `checkoutUrl`,
reached by the branching above.

## CRITICAL prerequisite (anti-loop): the offer/checkout must exist

The `offer` block needs a REAL `checkoutUrl`/`offerId`. On a fresh account those may not exist yet.
Before wiring `offer`, one of:

- **Create the target first**: use `clickmax-products` / `clickmax-offers` / `clickmax-funnels` to make
  the product+offer and a checkout page (`pages_create`), then read that page's public URL and use it
  as `checkoutUrl`.
- **Ask the user** which existing checkout/offer to point at (offer id or checkout URL).
- **Or defer the CTA**: if the user only wants to QUALIFY (no checkout ready), build the quiz with a
  `results` + `capture` result step and leave the `offer`/CTA pending — then TERMINATE and report
  exactly what's missing (which product/offer/checkout URL) so the user can supply it.

NEVER loop trying to reference a checkout that doesn't exist. Every path ends in either a wired
`offer`, a question to the user, or a finished quiz with a documented pending CTA.

## qualification[]

Server-side CRM rules evaluated on submit (top-level, set via `forms_replace_schema`). Each =
`{ when: Condition[], applyTagIds?: string[], addToListId?: string }`. When ALL `when` match
(typically over `$score`), apply tags / add to a list. Example:
`{ when: [ { field: '$score', op: 'gte', value: 5 } ], applyTagIds: ['<tagId>'] }`. Ids must be real
(resolve tag/list via the CRM tools); do not invent them.
