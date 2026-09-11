# Default design — Course Launch Editorial Grid

Read and adapt [editorial-sales.html](../assets/editorial-sales.html). Its HTML/CSS is the executable visual reference: paper canvas, Inter hierarchy, generous gutters, object-like deliverable illustrations, alternating composition, a perforated offer ticket and asymmetric native FAQ. Describing these patterns from memory is not equivalent to using the base.

## Resolve the brief first

Follow [content discovery](content-discovery.md). Dates, scarcity, prices, guarantee, instructor, testimonials, logos, curriculum, certificate and delivery must come from verified context/user answers. The sample bootcamp content demonstrates slots, not defaults for another business. Sample certificate artwork is visibly illustrative, not a credential. Never infer seats sold from capacity.

## Adapt without cloning

1. Retain one visual family across the funnel: neutral paper/ink, font, CTA treatment, hairlines and corner scale.
2. Set one product-specific accent using `.ce-page`'s `--ce-accent`, `--ce-accent-dark`, `--ce-tint`. Green is an example, not the required platform color. Keep contrast legible.
3. Choose hero composition: default centered with a four-object outcome row; add `ce-hero-split` to the page wrapper for a split hero with a 2×2 outcome grid. Both stack on mobile.
4. Choose content composition: default three module cards; add `ce-editorial` for numbered editorial rows. Mix dense/quiet sections; do not make every section identical cards.
5. Keep sections only when relevant and supported by the brief. Proof/mentors/certificate missing → ask whether the user can provide them, wants labeled draft samples, or prefers omission. Do not silently omit before asking unless the user explicitly skipped discovery.
6. Replace `data-cx-slot` content and example copy. Replace sample sheet/chart/document illustrations with real product assets when supplied. Without assets, adapt the illustration concept to the offer; never portray an illustrative dashboard as measured customer results. Preserve image aspect ratios and mobile fit.
7. Route every purchase CTA to the actual connected checkout URL; the sample anchors are draft scaffolding. Preserve `data-cx-cta` and recipe section markers.

## Recipe mapping

- Sales: preserve `hero`, `problem`, `solution`, `offer`, `cta`; these are roles, not rigid copy labels. Optional modules, schedule, certificate, FAQ and proof depend on answers.
- Capture: reuse composition and palette; remove the purchase ticket/price when irrelevant, include `benefits` and one `lead-form` following [components](components.md), and point CTAs to that form. Do not import unchanged sales content as capture.
- Checkout and thank-you use their companion HTML bases and the exact same chosen palette/font/button system. Do not copy the long sales pitch into them.

## Import-compatible behavior

The bundled baseline deliberately works with no author JavaScript: native details/summary, CSS hover/active states and explicit media-query declarations. The ticket uses opposing cutouts; illustrations are decorative CSS with meaningful visible labels. No unsupported canvas dependency or blank scroll-hidden content.

For richer motion, use only the documented declarative contract or a CSS enhancement with a visible static fallback. A synchronized sticky image gallery, cursor-reactive canvas, countdown or dynamically changing scarcity is not included in this base; never claim it works by adding stripped scripts. Static module imagery remains legible without these effects.

## Review before completion

Inspect the actual imported draft at 1440px, 1024px and 390px. Compare hero proportions, font family/weight, gutters, ticket and FAQ against the base; check horizontal overflow, visible text, loaded assets and correct destinations. Save/reopen in the editor before checking again when editing is part of the task.

Before publication, remove the demo banner and replace/remove every `data-cx-draft-placeholder`. Labeled fictional proof requires replacement/removal, even if the user approved it for a draft. Real illustrative product artwork may remain when clearly labeled and not implying customer evidence. A ready layout is not a verified payment or delivery flow.
