# AI funnel creation

Use this workflow in any MCP client when the user wants the assistant to research, design, and assemble a complete funnel with connected pages.

**Page authoring lives in the `clickmax-pages` skill.** Load it alongside this one for anything that happens inside a single page: the recipe → validate → draft-import pipeline, the visual system and token mapping, the section spine and copy rules, the form/checkout/CTA/motion contracts, and the missing-proof preview. This document covers only what is funnel-scoped — discovery, the shared brief, the one-design-per-funnel rule, and assembly.

## Discovery modes

Ask once before mutation:

|Mode|Behavior|Draft authorization|
|-|-|-|
|Guided research|Ask 1-3 concise optional questions per round. Every question accepts an answer or “infer for me”; also offer “skip discovery”.|Show the resulting blueprint and ask before assembly.|
|Automatic creation|Infer reversible structural and creative choices from real account context and the request; do not run the questionnaire.|Choosing automatic creation authorizes assembly of an unpublished draft, never publication.|

Client UI is not assumed: use structured elicitation when available; otherwise present the same choices in normal conversation. Skipped answers delegate a decision, not a fact.

Both modes produce one internal brief:

`objective | audience | offer | positioning hypothesis | page sequence | CTA path | visual preset | narrative | motion | verified proof/assets | assumptions | factual gaps`

Guided research covers only high-impact gaps:

1. Offer, audience, objective, and primary CTA.
2. Visual direction: **Cinematic Event | Premium Platform | Minimal Product | Immersive Storytelling | Direct Conversion**; optional reference URL; motion `subtle | balanced | expressive`.
3. Verified commercial facts/assets: price, dates, guarantee, testimonials, metrics, customers, certifications, media, and checkout offer.

Recommend an answer for creative choices. IF omitted → select from context and report the assumption.

## Safe inference

May infer: benefits, objections, FAQ, section order, CTA hierarchy, and positioning copy derived from verified product context.

Must not invent: testimonials, customer/logo names, sales/result metrics, certifications, dates, scarcity, guarantees, awards, press mentions, or outcome claims. Missing proof/facts → omit the section, or use the draft proof preview described in `clickmax-pages`; never create plausible fake proof.

A positioning hypothesis is copy, not evidence. Avoid turning it into a guaranteed or historical result.

## One design for the whole funnel

The design is chosen **once, before the first page**, and reused unchanged on every page after it. This is the funnel-scoped half of the rule; how to choose and apply a design is in `clickmax-pages`.

- Select one design whose light/dark mode and narrative match the brief, and record its id in the brief so every later page reuses the same one.
- Vary section composition by page purpose; never redesign the visual system per page.
- From the second page on, reuse the first page's CSS block literally and change only the content.
- The server inherits brand tokens through the funnel, but it does **not** control layout. A different hero on page three leaves the funnel visually broken even with the correct palette.
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
3. Guided mode → request assembly approval. Automatic mode → continue from the user's mode selection.
4. Call `mcp__plugin_clickmax_clickmax__funnels_create` exactly once; standard family → `mcp__plugin_clickmax_clickmax__funnels_sequence_create`, custom graph → `mcp__plugin_clickmax_clickmax__funnels_node_create`, never both for the same skeleton.
5. Author each planned page through the `clickmax-pages` pipeline, ending in `mcp__plugin_clickmax_clickmax__pages_import_html_draft` with the funnel target described above.
6. Attach every returned page with `mcp__plugin_clickmax_clickmax__funnels_node_connect_page`; route page triggers with `mcp__plugin_clickmax_clickmax__funnels_triggers_connect` and other node families with their dedicated connection tool.
7. Call `mcp__plugin_clickmax_clickmax__funnels_structure_get`, verify every intended edge, then `mcp__plugin_clickmax_clickmax__funnels_validate`. Repair the existing funnel/page in place; never restart by creating duplicates.
8. Report draft ids/paths, mode, assumptions, factual placeholders, visual/motion choices, and validation gaps. Publish only after separate explicit consent and a fresh validation pass.
