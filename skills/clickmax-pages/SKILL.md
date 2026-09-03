---
name: clickmax-pages
description: Use when the user wants to create, inspect, restyle, rebuild, clone, configure, or publish a native Clickmax-hosted page.
---

## When this applies

Any native Clickmax page job: discovery, inspection, authoring a page with real content, a targeted edit, a clone, configuration, checkout binding, or publication.

Scope boundary — pick the right skill first:

|Situation|Skill|
|-|-|
|Page already lives on a domain NOT connected to Clickmax (Framer, Webflow, own site)|`clickmax-external-pages`|
|Whole connected funnel with several pages and edges|`clickmax-funnels`|
|Offer, price, guarantee, deliverable|`clickmax-offers`|
|Form/quiz behavior, fields, logic|`clickmax-forms-quizzes`|
|One native page — content, visual system, structure, publish|this skill|

## Key assumptions

- A page with real content is authored as **one full HTML document** and imported. `pages_create` alone produces a blank page or a template copy — no copy, no sections, no working form.
- The authoring path is validated by machine, not by taste: recipe capabilities and every `requiredSections` id are checked before import. Guessing the shape wastes a round trip.
- **A curated page design is mandatory input, not a style preference.** Choosing one is what separates a designed page from default CSS. This holds even when the user gave zero style direction.
- The project's style guide is a fresh default (commonly light mode, blue/navy palette, small type scale) until someone changes it, and no tool changes it. A dark/accent request that conflicts with it must be satisfied by an explicit local palette applied to every section — never by half-styled markup that lets bare tags inherit the default.
- `<script>` is stripped on import. Every interaction is CSS-only or it does not exist.
- One capture form per page. Two forms record the lead twice and make the funnel's `form_submit` trigger ambiguous.
- Real facts only. Prices, deadlines, guarantees, testimonials, logos, metrics, and image URLs come from the user or they do not appear.

## Thought process

1. Resolve the target page first — existing page, new page in a project, or a page inside a funnel. Never act on a remembered id without a fresh lookup this turn.
2. Classify the job: read/config · blueprint only · new page with content · targeted edit · explicit full rebuild · clone · publish.
3. Narrow request on an existing page → smallest possible edit. Never regenerate unrelated sections.
4. Resolve product, offer, audience, proof, assets, and constraints only as far as the job needs.
5. Pick the recipe from the page type, then pick the curated design, then decide the section rhythm — in that order. Structure before copy, copy before CSS.
6. Author, validate, fix every error, import as draft.
7. Publication is always a separate, explicitly approved step.

## Execute guide

1. **Pipeline and tool ordering** — recipes, manifest, generation context, validation error codes, target rules, clone, configuration, and publication: [authoring pipeline](references/authoring-pipeline.md).
2. **Visual system** — choosing the curated design, mapping it onto the injected tokens, type scale, color ramp, spacing, container, and breakpoints: [visual system](references/visual-system.md).
3. **Structure and copy** — the mandatory spine, page jobs, section logic, and copy rules: [sections and copy](references/sections-and-copy.md).
4. **Components** — the CSS-only building blocks that survive import, plus the form, checkout, CTA, and motion contracts: [components](references/components.md).

## Report

- Open with the page job, target product/offer, and any material assumption made.
- Blueprint answers: sections in order as `section → job → final copy → visual treatment`; cap long lists with `+N more sections`.
- Build/edit answers: the visible sections, the chosen design, the responsive behavior, and exactly what changed. Do not dump markup unless asked.
- List missing business facts and assets as explicit gaps, named.
- Uncertain or failed save → say so and say what must be re-checked. Never claim success from a call that did not confirm.
- Close by offering a desktop and mobile review. Offer publication only as a separate opt-in.

## Warnings

- Never publish without explicit consent, and never as part of a build step.
- Never replace a non-empty page unless the user asked for a full rebuild.
- Never hand-write checkout markup — the payment runtime looks for specific ids and a mistake breaks charging with no visible symptom.
- Never invent an image URL. A broken image in production is worse than a section without one.
- A validation call that returns `valid: false` means fix and revalidate — not try a different tool.
- Warnings on a successful import are actionable: re-import the same page id with the corrected CSS.

## Anti-patterns

- Freehanding the visual system instead of selecting a curated design.
- A generic "modern and clean" direction with no concrete visual concept.
- Writing copy before deciding the page job, the audience's awareness level, and the objection being answered.
- The same centered card layout in every section.
- Padding the page with schedule, modules, instructors, certificate, and bonus blocks the user never gave data for.
- Regenerating the whole page for a color, spacing, or one-section request.
- Creating a blank page and telling the user to fill it in themselves.
