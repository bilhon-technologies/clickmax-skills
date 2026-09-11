# AI funnel creation

Use this workflow in any MCP client when the user wants the assistant to research, design, and assemble a complete funnel with connected pages.

**Page authoring lives in the `clickmax-pages` skill.** Load it alongside this one for anything that happens inside a single page: the recipe → validate → draft-import pipeline, the visual system and token mapping, the section spine and copy rules, the form/checkout/CTA/motion contracts, and the missing-proof preview. This document covers only what is funnel-scoped — discovery, the shared brief, the one-design-per-funnel rule, and assembly.

## Discovery modes

Load `clickmax-pages` and follow its **content discovery** reference before authoring. Guided discovery is the default for missing content; default design only delegates visual decisions. Explicit automatic creation or skip-discovery bypasses optional questions, never the need to distinguish facts from assumptions.

Use the client's structured input when available, otherwise ask in normal conversation and wait. Ask only missing facts (especially proof, mentors/assets and real delivery links); reuse every answer across the funnel. No questionnaire UI is required.

Both modes produce one internal brief:

`objective | audience | offer | positioning hypothesis | page sequence | CTA path | visual preset | narrative | motion | verified proof/assets | selected omissions | explicitly requested fictional draft samples | assumptions | factual gaps`

An existing request to create the funnel authorizes continuing draft assembly after discovery; do not request approval again. A blueprint-only request ends at the blueprint. Publishing remains separate.

## Safe inference

Infer reversible creative choices from verified context. Missing proof, fictional sample labels and unavailable delivery destinations follow `clickmax-pages` content discovery. Explicitly requested fictional samples are draft-only and must be replaced/removed before publication. Skipping discovery never authorizes invented testimonials, customers, metrics, certifications, dates, scarcity, guarantees, awards, press mentions or outcome claims.

A positioning hypothesis is copy, not evidence. Avoid turning it into a guaranteed or historical result.

## One design for the whole funnel

The design is chosen **once, before the first page**, and reused consistently on every page after it. This is the funnel-scoped half of the rule; how to choose and apply a design is in `clickmax-pages`.

- Resolve one base through `clickmax-pages` and record its resource name plus chosen palette/variants in the brief. Record a catalog id only when an actual catalog design was selected; never invent an id for a bundled default.
- Vary section composition by page purpose; never redesign the visual system per page.
- Reuse shared typography, palette, spacing and component rules; adapt layout to sales, checkout and thank-you purposes using the matching page-authoring base.
- The server inherits brand tokens through the funnel, but it does **not** control layout. Shared tokens alone do not establish a coherent layout; preserve the same component language while varying composition.
- The motion level chosen in discovery (`subtle | balanced | expressive`) applies funnel-wide: expressive means at most 1-2 focal moments per viewport, not motion on every block.

## Page authoring inside a funnel

Follow the pipeline in `clickmax-pages` for every page. Two things are specific to building inside a funnel:

- A **new** page carries the funnel: `placement.projectId` + `funnelId`, plus a unique name and path and the node's page type. That is what makes it inherit the funnel's brand tokens.
- A **re-import** that must keep the funnel style uses `pageId` + `funnelId` — never a `placement` used only to carry the funnel, because its name, type, and path would be silently ignored.
- Choose the recipe from the **node's** page type: a `checkout` node requires the `checkout` recipe.
- Automatic creation always ends in an unpublished draft, page by page and funnel-wide.

## Assemble

1. Resolve the real project, product, offer, and usable assets; never guess ids.
2. Build one blueprint: funnel family, ordered pages, node types, exits, offer/form contracts, the chosen design, and assumptions.
3. Continue the requested draft after discovery; ask for assembly only if the user requested a blueprint without creation.
4. Call `mcp__plugin_clickmax_clickmax__funnels_create` exactly once; standard family → `mcp__plugin_clickmax_clickmax__funnels_sequence_create`, custom graph → `mcp__plugin_clickmax_clickmax__funnels_node_create`, never both for the same skeleton.
5. Author each planned page through the `clickmax-pages` pipeline, ending in `mcp__plugin_clickmax_clickmax__pages_import_html_draft` with the funnel target described above.
6. Attach every returned page with `mcp__plugin_clickmax_clickmax__funnels_node_connect_page`; route page triggers with `mcp__plugin_clickmax_clickmax__funnels_triggers_connect` and other node families with their dedicated connection tool.
7. Call `mcp__plugin_clickmax_clickmax__funnels_structure_get`, verify every intended edge, then `mcp__plugin_clickmax_clickmax__funnels_validate`. Repair the existing funnel/page in place; never restart by creating duplicates.
8. Report draft ids/paths, mode, assumptions, factual placeholders, visual/motion choices, and validation gaps. Publish only after separate explicit consent and a fresh validation pass.
