# Redirect / advance the funnel from an external page (step 3 — "Redirecionamento")

Docs: https://docs.clickmax.io/sdk/tracking/

Prereq: the step-1 head script is already installed. This step annotates elements so the SDK's routing advances the visitor to the next funnel step on interaction. STATIC snippet — paste verbatim in a ```html block, then link the docs.

Cycle: **annotate** the elements → **scan** the page from the funnel builder (discovery) → **connect** the detected element to the next node in the funnel graph. The annotation is what the user pastes; scan + connect happen in Clickmax.

```html
<!-- Botão que avança para a próxima etapa do funil -->
<button class="cm-checkout-button">Comprar agora</button>
<!-- ...ou, sem trocar a classe existente: -->
<button data-cx-track="click">Comprar agora</button>

<!-- Link que avança para a próxima etapa -->
<a href="#" class="cm-next-funnel-node">Continuar</a>
<!-- ...ou: -->
<a href="#" data-cx-track="link">Continuar</a>

<!-- Formulário que captura o contato e avança -->
<form class="cm-form">
  <input name="email" type="email" />
  <button type="submit">Quero participar</button>
</form>
```

Recognized markers:

|Element|Class|Attribute alt|
|-|-|-|
|button → next step|`cm-checkout-button`|`data-cx-track="click"`|
|link → next step|`cm-next-funnel-node`|`data-cx-track="link"`|
|form → capture + advance|`cm-form`|—|

- Add the Clickmax class/attribute ALONGSIDE existing ones — never remove the site's own classes.
- Use `data-cx-redirect-url` on the element only to force a fixed destination locally; otherwise the funnel's connected edge decides where the visitor goes.

Source of truth (keep snippet in sync): `web/funnels4/src/features/external-page/lib/snippet-builders.ts` (`buildRedirectAnnotationSnippet`).
