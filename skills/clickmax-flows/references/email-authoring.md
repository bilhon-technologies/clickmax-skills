# Email step content: picking a mode

`mcp__plugin_clickmax_clickmax__flows_send_email` supports three ways to author the body — pick one per call, never mix:

1. `format: 'text'` — plain text via `content`. Only for an explicit plain-text ask.
2. `format: 'html'` (default) WITHOUT `customHtml` — renders inside a FIXED skeleton (sender logo, eyebrow+title, checkmark illustration, subtitle+body, benefits checklist card, pill CTA button, footer) via the copy slots (`eyebrow`/`title`/`subtitle`/`body`/`benefitsTitle`/`benefits`[]/`ctaLabel`/`ctaHref`/`ctaNote`/`footerNote`) plus a color/font skin (`fontFamily`/`accentColor`/`buttonTextColor`/`titleColor`/`textColor`/`footerTextColor`/`cardBackground`). The skeleton itself (layout, spacing, button shape) never changes no matter the colors — this reads as "the default template, recolored," not as a designed email. The header logo and the footer address are filled from the account automatically (see [Sender identity](#sender-identity)), so leave `footerNote`/`logoUrl` out unless the user dictates them. No `greeting` slot — the skeleton has no opening line; put any greeting inside `body`. Use this mode ONLY when the user explicitly wants the plain default template look, or for a trivial/internal email where visual identity doesn't matter.
3. `format: 'html'` WITH `customHtml` — a genuinely distinct, on-brand email. `customHtml` REPLACES the slot fields entirely when present (they're ignored).

## Always style with a design, never just recolor

Before authoring ANY email whose content the user supplies (not an explicit "plain default" ask): call `page_designs_list` first, pick one curated visual language (whichever fits the content/tone, or pick randomly if none stands out — never ask the user to choose), then use mode 3 (`customHtml`) to actually reproduce that look. Mode 2's color knobs alone are not a substitute for a real design and will look like a bare recolor.

## Authoring `customHtml`

Write real transactional-email-safe HTML, not web-page HTML:

- Table-based layout (`<table role="presentation">`), not flexbox/grid — major email clients strip or ignore those, and this is NOT auto-fixed (see below).
- Every style INLINE (`style="..."` on each element) — no `<style>` block, no external stylesheet; both get stripped by Outlook/Gmail. A leftover `<style>` block or `var(--...)` from a `page_designs_list` design (webpage CSS, not email-safe) IS auto-inlined server-side before sending — but treat that as a safety net for what a hand transcription misses, not a reason to paste page CSS verbatim.
- One clear CTA link per email.
- Same personalization tokens as `content`/the slot fields: `{name}`, `{email}`, `{telephone}`, `{checkout}` / `{checkout:offerId}` — see [step types](step-types.md).
- Keep it light: one focused design reproducing the chosen catalog style, not a multi-section production.

## Sender identity

The email goes out as the USER's business, never as Clickmax. Never put a Clickmax logo, the Clickmax name, or a clickmax.io address in an email body.

- Mode 2 (slots): automatic, and it is the SAME rule the broadcast and the flow email editor apply — the workspace brand logo → the user's profile photo → the sender name as text, plus `footerNote` = business name + registered address. Pass `logoUrl`/`footerNote` only when the user dictates a specific image / different small print — never invent an address.
- Mode 3 (`customHtml`): call `mcp__plugin_clickmax_clickmax__whoami` and use `sender.logoUrl` for the header image and `sender.addressLine` for the footer line. A Clickmax logo copied from a `page_designs_list` design IS swapped for the sender's automatically, but the footer address is yours to write.
- Field null in `whoami`? The account never set it: leave the slot out and TELL the user (logo in `Marca`/profile, address in the account profile), so their next email carries it.

## Sender id

`emailSenderSignatureId` is required the same way as for every other channel — see [step types](step-types.md) for the shared sender-id rule (resolve with `email_sender_signatures_list`, never fabricate a uuid, resolve at creation time).
