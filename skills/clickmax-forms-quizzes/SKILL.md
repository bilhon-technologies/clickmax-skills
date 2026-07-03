---
name: clickmax-forms-quizzes
description: Use when the user wants to safely create, edit, publish, inspect, or analyze Clickmax forms and quizzes.
---

## When this applies

Use this skill when the user wants form/quiz operations where schema safety matters: editing one step, rebuilding a quiz, branching, scoring, qualification, publish state, submissions, or analytics.

Not this skill:

- lead cohort search from submissions -> use the relevant CRM lead/list/segment skill after identifying the submitted form
- page/funnel placement for a form -> resolve page/funnel work separately

## Key assumptions

- Form/quiz content is persisted as one schema document; full replacement overwrites the current schema.
- Step-level tools exist to avoid re-emitting the whole schema for small edits.
- `expectedUpdatedAt` protects against silently overwriting concurrent edits; use it when replacing schema or editing a stale form.
- `theme` and `settings` are part of the working schema; preserve them unless the user asks to change them.
- Quiz branching, score accumulation, and qualification outcomes must remain coherent after every edit.
- Publish state is controlled by `active`; deleting is not the same as pausing collection.

## Thought process

1. Inspect the current form/quiz first.
2. Choose granular step upsert/delete for one-question or one-step changes.
3. Choose full schema replacement only for broad rebuilds where the whole schema is intentionally regenerated.
4. Before publishing, verify branching targets, scoring rules, required fields, and qualification outcomes are still reachable.

## Execute guide

- Use `mcp__clickmax__forms_list` to find candidate forms/quizzes by kind/status/search.
- Use `mcp__clickmax__forms_get` before edits; keep the returned step ids, schema shape, theme/settings, and update timestamp in mind.
- Use `mcp__clickmax__forms_step_upsert` for a single new/changed step; this is the default safe path for incremental quiz editing.
- Use `mcp__clickmax__forms_step_delete` only when removing that step is explicit, then check whether any branching target pointed to it.
- Use `mcp__clickmax__forms_replace_schema` only when the desired result is a complete replacement; send the full schema and protect stale writes with the known update timestamp when available.
- Use `mcp__clickmax__forms_update` for metadata/status changes, including publishing or pausing through active state.
- Use `mcp__clickmax__forms_submissions_list` for response rows and `mcp__clickmax__forms_analytics` for performance summaries.

## Report

- For edits: report the changed step(s), publish state, and whether branching/scoring still needs review.
- For full rebuilds: state that the schema was replaced, not patched.
- For submissions/analytics: summarize counts, conversion/response pattern, and only the most relevant examples.

## Warnings

- Do not use full schema replacement for a small question edit unless the user explicitly wants a rebuild.
- Do not drop existing theme/settings during schema replacement.
- Do not publish a quiz if branching targets, scoring, or qualification outcomes are incomplete.
- Prefer pausing over deleting when the user only wants to stop accepting responses.

## Anti-patterns

- Guessing schema field names instead of inspecting the current form.
- Recreating an entire quiz when a step-level edit is enough.
- Treating form analytics as lead membership source of truth.
