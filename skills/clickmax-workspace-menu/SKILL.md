---
name: clickmax-workspace-menu
description: Use when the user wants to customize the workspace menu (module rail and side menu) — hide, rename, reorder, change icons, move items between modules, create modules, groups, links or shortcuts — or asks why the menu looks different.
---

## When this applies

Changes to the menu every member of the workspace sees (Settings › Personalization › Menu): the module rail on the left and the side menu each module opens. Also "why don't I see X in the menu?" when an admin may have hidden it.

## Not this skill

- app colors, themes, logo and favicon → the user does it in Settings › Personalization (no tool edits colors)
- who can open a page → permissions (`clickmax-team-members`); hiding a menu entry never removes access

## Key assumptions

- Draft → publish. Edits go to a shared draft the admin also sees on the screen; members only see the change after `workspace_personalization_publish`.
- `nav` is a diff over the default Clickmax menu keyed by native id. A wrong native id is ignored with no error, so the change "succeeds" and nothing happens.
- Native ids come only from `nav.nativeCatalog` in the draft. Clickmax fills it in when an admin opens the app in the browser. If it is missing or lacks a name the user said, ask them to reload Clickmax and try again; until then only created entries can be edited.
- An admin may be editing the same draft on the screen. Whatever the screen saves later wins.

## Execute guide

1. `workspace_personalization_get_draft`. If `hasDraft` is true, there are unpublished changes that may be someone else's: mention them before adding yours.
2. Resolve each name the user said to an id. Match it against the current name (override `label` in `modules`/`groups`/`items`/`custom*`, else the catalog `label`). If two entries share a name, use `kind` and `parent` to tell them apart, or ask.
3. Build the new `nav` from the one you read, changing only what was asked, and send it whole with `workspace_personalization_update_draft`. Leaving a key out resets that part to the default menu.
4. Check the returned draft for your change.
5. Summarize the change and ask before `workspace_personalization_publish`, unless the user already asked to publish. Publishing also publishes any color changes waiting in the draft.

## Report

Describe the result in menu terms the user sees ("Webinar no longer shows in the rail; Contatos is now called Clientes"), not ids or JSON. Say whether it is published or still a draft, and where to review it (Settings › Personalization).

## Warnings

- Custom SVG icons need a file upload from the browser. Tools only take the built-in icon keys listed in the tool description.
- `rail` lists every module once. When reordering, keep all the module ids you read, including created ones.
- Only one nesting level: an item that already has children cannot become a sub-item.
- `workspace_personalization_discard_draft` throws away every pending change, the admin's too. Confirm first.

## Anti-patterns

- Guessing ids such as `contacts` instead of reading the catalog.
- Saying "try again later" or "do it on the screen" when the catalog is missing, instead of asking for a reload.
- Sending only the keys you changed in `nav`.
- Telling the user that hiding a page blocks access to it.
- Publishing right after editing without the user's go-ahead.

---

Clickmax skill revision: `d63a63046a94`
