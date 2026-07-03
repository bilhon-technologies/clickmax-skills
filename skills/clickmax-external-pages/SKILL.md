---
name: clickmax-external-pages
description: Use when the user wants to connect an external page/site to Clickmax tracking or forms using Clickmax page scripts.
---

## When this applies

Use this skill for external pages: a page hosted outside the Clickmax builder that needs Clickmax tracking, capture/forms, or script installation guidance.

Not this skill:

- designing/editing builder page content -> use the visual builder workflow
- funnel graph routing -> `clickmax-funnels`

## Key assumptions

- External pages are not builder pages; do not promise builder-only editing, publish, path, or domain controls.
- The external script goes in the external site's `<head>` so tracking/form bindings initialize reliably.
- Clickmax can provide script/bootstrap guidance, but the external site's HTML/CMS must be edited outside Clickmax.
- Redirect behavior and form attributes must be kept consistent with the user's external page flow.

## Thought process

1. Confirm the page is external and resolve/create its Clickmax page record.
2. Retrieve the external script for the exact page.
3. Explain the install location and the minimum form attributes needed.
4. Keep the answer to one concrete install example unless the user asks for variants.

## Execute guide

- Use `mcp__clickmax__pages_list` / `mcp__clickmax__pages_get` to identify existing external page records.
- Use `mcp__clickmax__pages_create_external` when the external site does not yet have a Clickmax page record.
- Use `mcp__clickmax__pages_update` for external page metadata or URL changes.
- Use `mcp__clickmax__pages_get_external_script` for the install payload; tell the user to paste the head script into the external site's `<head>`.
- Use `mcp__clickmax__pages_delete` only when the user explicitly wants to remove the Clickmax page record.

## Report

- State that this is an external page setup, not a Clickmax builder page.
- Return page URL/identity, script install location, and one concise form/tracking instruction.
- If the user needs implementation help, provide the smallest HTML example that matches the returned script guidance.

## Warnings

- Do not tell the user Clickmax can visually edit external site content.
- Do not use builder-page publish/config language for external pages.
- Do not dump multiple script/form variants when one exact install path is enough.

## Anti-patterns

- Treating a funnel node that references a page as proof the external script is installed.
- Returning a generic web-tracking answer without using the page-specific script tool.
