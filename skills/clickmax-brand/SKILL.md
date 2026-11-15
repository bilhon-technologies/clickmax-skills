---
name: clickmax-brand
description: Use when the user wants to create, fill, rewrite or review a brandbook (brand guide, visual identity, "manual da marca") in Clickmax, including building one from a website or reference material.
---

## When this applies

A brandbook in Clickmax IS a brand record (Settings › Brandbooks): business data, voice, visual tokens (colors, fonts, logo) and a book of chapters, one HTML fragment per chapter. Every generator in the account (pages, posts, ads, emails, Cx Studio) reads it.

Not this skill:

- writing copy in an existing brand voice -> read `mcp__plugin_clickmax_clickmax__brand_get` inside the skill of what is being written
- saving only the tone during a campaign plan -> `clickmax-campaign-planning`
- who speaks for the brand (producer profile) -> `mcp__plugin_clickmax_clickmax__producers_create` / Producers page

## Key assumptions

- "cria um brandbook" = a brand record with chapters, never a standalone HTML document or report. A document is not listed in Settings › Brandbooks and no generator reads it
- `mcp__plugin_clickmax_clickmax__brand_save_draft` without `id`: creates the default brand only when the workspace has none. Brands already exist and the user wants a new one -> `newBrand: true`. Improving one that already exists -> its `id` from `brandChoices`
- several brands and the request does not say which -> ask which one (or a new one) before writing; never pick by name match
- tokens (colors, fonts, logo) are brand fields saved with `mcp__plugin_clickmax_clickmax__brand_save_draft`; chapters only reference them through `var(--b-*)` and never contain a literal color
- `actor: user` chapter = edited by the person: rewrite it only when asked for that chapter

## Thought process

- reference given (site, PDF, image, text) -> extract the facts first: name, what the business does, audience, promise, palette with roles, fonts, logo rules, voice. Use only what the material says
- order: brand fields + tokens -> chapters. Chapters read the tokens, so they come first
- chapters: `core` ones from the catalog first (essence, logo, colors, typography, voice, ads, posts, kit), then optional ones the material supports (graphics, people, pages, video, email, conversations, webinar). Skip a chapter when the material has nothing for it rather than inventing it
- missing information that changes the result (which brand, the business name) -> ask once; everything else -> build and say what was assumed

## Execute guide

1. `mcp__plugin_clickmax_clickmax__brand_get` -> `brandChoices` and the current brand
2. `mcp__plugin_clickmax_clickmax__brand_save_draft` with `id`, `newBrand: true` or neither (see Key assumptions) and every field the material supports: `name`, `businessName`, audience, USP, `voiceTone`, vocabulary, samples, visual tokens. Keep the returned `id`
3. `mcp__plugin_clickmax_clickmax__brand_chapters_get` with that `brandId` -> `guide`, `catalog` and the current `actor` per chapter
4. one `mcp__plugin_clickmax_clickmax__brand_chapter_render` per chapter, following `guide`. Lint failure = nothing saved: fix the listed problems and call again for the same chapter
5. publish with `mcp__plugin_clickmax_clickmax__brand_mark_ready` only when the user asks and `businessName` is set

## Report

- one or two sentences: which brandbook (new or updated), which chapters were written, what was assumed or skipped
- point to the brandbook in Settings › Brandbooks; never paste the chapters' content or the palette as a list in the chat

## Warnings

- `mcp__plugin_clickmax_clickmax__brand_save_draft` with `newBrand: true` creates a record every call: use it once, then pass the returned `id`
- external image URLs are refused by the chapter lint: only the brand's own file URLs (logo and files from `mcp__plugin_clickmax_clickmax__brand_get`)
- changing tokens recolors every chapter and the account's generators: confirm before replacing tokens the person set

## Anti-patterns

- answering "create a brandbook" with an HTML document, a board or a long chat text
- filling the default brand when the user asked for a new brandbook
- one giant chapter with everything instead of one fragment per catalog `kind`
- writing chapters before the tokens exist (the book renders in default colors)

---

Clickmax skill revision: `61fff086819e`
