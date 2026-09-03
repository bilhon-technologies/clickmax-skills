---
name: clickmax-support
description: Use when the user asks a how-to, troubleshooting, or "why isn't this working" support question about using Clickmax, and the answer should come from the help center.
---

## When this applies

Use this skill when the user asks a **support/help question about how Clickmax works** — how to do something, why something is not working, or where to find a feature — and the answer should come from the Clickmax help center (product documentation).

Not this skill:

- The user wants Max to _do_ an operation on their account (create a lead, build a funnel, send messages, analyze sales) -> the matching operational skill (`clickmax-leads`, `clickmax-funnels`, etc.). This skill is ANSWER-ONLY; it never performs account actions.
- The user asks about their own workspace data (their leads, their orders, their revenue) -> the operational analysis skills. The help center documents the product, not the user's data.
- The user asks "what changed / what's new" about a release -> use `mcp__plugin_clickmax_clickmax__helpdesk_releases_list` / `mcp__plugin_clickmax_clickmax__helpdesk_release_latest` instead of article search.

## Key assumptions

- The help center is the **single source of truth** for support answers. Answer only from article content actually retrieved this turn — never from general knowledge, and never from the product rules baked into the operational skills.
- `mcp__plugin_clickmax_clickmax__helpdesk_search` matches the term against article title, excerpt and body only (it ignores `keywords`), and returns only PUBLISHED articles whose parent feature and module are also published. A relevant article can therefore be missing from results.
- Search results carry excerpts, not full bodies. Excerpts are for **picking** the right article; the body is for **answering**. Always fetch the body before answering.
- Answering is a two-step retrieval: `mcp__plugin_clickmax_clickmax__helpdesk_search` to find candidates, then `mcp__plugin_clickmax_clickmax__helpdesk_article_get` to read the winner — mirroring `leads_search` -> `leads_get`.
- If the user pasted a help deep-link (a slug), resolve it directly with `mcp__plugin_clickmax_clickmax__helpdesk_article_get_by_slug` instead of searching.

## Thought process

1. Confirm this is a support/help question, not an operational request or a data question. If it is operational or about the user's data, hand off to the right skill — do not answer it here.
2. Decide the retrieval entry point: a concrete question -> `mcp__plugin_clickmax_clickmax__helpdesk_search`; a pasted help link/slug -> `mcp__plugin_clickmax_clickmax__helpdesk_article_get_by_slug`; a vague/browsing question where search terms are unclear -> `mcp__plugin_clickmax_clickmax__helpdesk_tree` to navigate by topic first.
3. Judge relevance honestly. A search hit is only usable if its title/excerpt genuinely matches the user's question. A weak or tangential match is NOT a match — treat it as "no article found."
4. Ground before answering: read the chosen article's full body, then answer strictly from it.
5. If nothing relevant is found, deflect honestly — never stretch an unrelated article to fit.

## Execute guide

- Use `mcp__plugin_clickmax_clickmax__helpdesk_search` with the user's question (or its key terms) as `search`. Prefer the user's own wording; retry once with alternative terms only if the first search is empty and you have a clearly better phrasing.
- Inspect the returned list. Pick the single best match by title + excerpt + module/feature relevance. If two articles look equally right, fetch the more specific one first.
- Use `mcp__plugin_clickmax_clickmax__helpdesk_article_get` with the chosen `id` to read the full markdown body. Use `mcp__plugin_clickmax_clickmax__helpdesk_article_get_by_slug` when resolving a pasted slug.
- Answer the user **only** from that body: summarize the relevant steps faithfully, in the user's language, without adding steps or facts the article does not state.
- Always end the answer with the source: the article title and its help-center reference (slug/link).
- When search returns nothing relevant (empty, or only weak/tangential hits): deflect. Tell the user, honestly, that you could not find this in the help center, and point them to the official support channel to talk to a human. Do NOT invent an answer, and do NOT answer from general knowledge.
- Never call any write/action tool from this skill. It is answer-only.

## Report

- Answer in the user's language (PT by default; match the user if they write in another language).
- Be concise and practical: the concrete steps or explanation the article gives, in order.
- Always cite the source article (title + slug/link) so the answer is traceable.
- On deflection: one honest sentence that you did not find it in the help center, then the official support channel to reach a human. Keep it short and non-defensive.

## Warnings

- Strict grounding is mandatory: if you did not read it in a retrieved article this turn, do not state it as a support answer.
- A published article can be missing from search when its feature/module is unpublished. If the user is sure a feature exists but search is empty, say you could not find documentation for it and deflect — do not reconstruct it from memory.
- Do not confuse the help center (product docs) with the user's workspace data. "Por que meu pagamento está pendente?" about a specific order is a data/operational question, not a docs lookup.

## Anti-patterns

- Answering a support question from general/model knowledge or from the operational product skills instead of a retrieved article.
- Stretching a weakly-matching article to avoid deflecting.
- Answering from a search excerpt without fetching the full article body.
- Silently performing an account action to "fix" the user's problem — this skill never acts.
- Omitting the source citation on a grounded answer.
