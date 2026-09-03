# Lead capture on an external page (step 2 — "Configurar captura")

Docs: https://docs.clickmax.io/sdk/forms/

Prereq: the step-1 head script (`pages_get_external_script` -> `headScript`) is already in the page `<head>` — the form/widget below is bound by that SDK. These snippets are STATIC (no per-page data): paste one verbatim in a ```html block, then link the docs. Two options: a **custom form** (user's own markup) or a **widget** (the SDK renders it).

## Option A — custom form (bind the user's own HTML form)

```html
<form data-cx-ingest-form>
  <input data-cx-field="name" type="text" placeholder="Nome" />
  <input data-cx-field="email" type="email" placeholder="E-mail" />
  <input data-cx-field="telephone" type="tel" placeholder="Telefone" />
  <label>
    <input data-cx-field="lgpdApproved" type="checkbox" />
    Concordo em receber comunicações
  </label>
  <input data-cx-field="website" type="text" hidden />
  <button type="submit">Enviar</button>
</form>
```

- `data-cx-ingest-form` on the `<form>` = the SDK captures the submit and ingests the lead.
- `data-cx-field="<field>"` per input — supported: `name`, `email`, `telephone`, `lgpdApproved` (LGPD checkbox), `website`.
- `website` = honeypot: keep it `hidden`, never fill — bots that fill it are dropped.
- Optional `<form>` attributes:

|attr|effect|
|-|-|
|`data-cx-redirect-url="https://…"`|force post-submit destination (else the funnel hints decide)|
|`data-cx-nostyles`|render without Clickmax's default form styles|
|`data-cx-hijack-form`|bind an existing/third-party form instead of this markup|
|`data-cx-success-message="…"`|inline success text after submit|
|`data-cx-locale="pt-BR"`|form/validation locale|

## Option B — widget (SDK renders the form)

Inline (renders where the div is):

```html
<div
  data-cx-form-widget
  data-cx-mode="inline"
  data-cx-theme="light"
  data-cx-title="Receba o material"
  data-cx-submit-label="Quero receber"
  data-cx-fields="name,email,telephone"
  data-cx-required-fields="email"
  data-cx-show-lgpd="true"
  data-cx-lgpd-required="true"
></div>
```

Modal (place the container once, then a trigger button anywhere):

```html
<div
  data-cx-form-widget
  data-cx-widget-id="captura"
  data-cx-mode="modal"
  data-cx-title="Receba o material"
  data-cx-fields="name,email"
  data-cx-required-fields="email"
></div>

<button data-cx-open="captura">Quero participar</button>
```

- `data-cx-fields` / `data-cx-required-fields` = comma-separated field list.
- Modal mode: `data-cx-widget-id` must match the `data-cx-open` value on the trigger.

Source of truth (keep snippets in sync): `web/funnels4/src/features/funnel/update/ui/ingest-link-snippets/snippet-builders.ts`.
