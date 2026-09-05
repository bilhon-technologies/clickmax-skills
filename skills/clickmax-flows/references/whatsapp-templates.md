# WhatsApp templates

## Why templates, not free-form text

WhatsApp only lets a business send free-form text inside the **24h customer-care window** — the 24 hours after the contact's own last inbound message. Outside it, only an approved template is delivered.

A flow started by a funnel, tag, schedule, or checkout has no open window, so `flows_send_whatsapp` **rejects `format: 'text'` unless `numberStrategy: 'context'`** (the only strategy that means "the contact started this by messaging us"). This is a deliverability rule, not a cost one: Meta bills per conversation window either way, so never call free-form text the "free" option.

## Choosing a template, cheapest path first

1. `gupshup_templates_list` — an existing `approved` template that already says what the user wants. Nothing to submit, sends today.
2. `gupshup_template_library_list` + `gupshup_template_library_create` — Meta's own pre-vetted library (`UTILITY`/`AUTHENTICATION` only), for generic use cases: OTP, order update, appointment reminder. Near-instant approval. `libraryTemplateName` must come from the list call, never invented; this tool submits to Meta immediately.
3. `gupshup_templates_create` — author one. Saves a local draft and validates it against Meta's rules first. Then show the exact copy to the user, get their confirmation, and `gupshup_templates_submit`.

Wire the step with `format: 'template'`, `gupshupTemplateId`, and a `paramMapping` whose LENGTH matches the template's `positionParams`. `content` in this mode is the template's `elementName`, not prose.

## Authoring rules (validated at create — the error lists everything to fix)

- Body variables are `{{1}}`, `{{2}}`… sequential from 1, no gaps. The body may not START or END with one — Meta requires fixed text around every variable.
- `example` is the body with every variable replaced by a real sample value. An unfilled `{{n}}` there is a rejection.
- `positionParams` is derived from the body — omit it. Passing a length that disagrees with the body is rejected (a short array sends an empty parameter and Meta drops the message with #131008).
- Body max 1024 chars, `header`/`footer` max 60, at most 10 emojis and 2 consecutive line breaks in the body, no emoji/line break/`*_~\`` in a text header.
- `elementName` is normalized to what Meta accepts (lowercase, digits, `_`), 6-24 chars — "Boas Vindas" is stored as `boas_vindas`.
- A text `header` only exists on `templateType: 'TEXT'`. `IMAGE`/`VIDEO`/`DOCUMENT` need `file` (a fetchable url) or an existing `exampleMedia` handle.
- Buttons: label max 25 chars, at most 2 URL buttons, URL must be http(s) with at most one variable at its very end, quick replies grouped together (never interleaved), copy-code only on `MARKETING`.
- `AUTHENTICATION` has a Meta-managed body: empty `content`/`example`, exactly one `OTP` button.
- Never invent copy, button URLs, or sample values the user did not give you.

## Approval timing

Submitting is external and practically irreversible: review takes hours to days, and a rejected or low-quality submission hurts the number's quality rating. A `pending` template can already be wired into the flow step — build the whole automation, then tell the user WhatsApp starts sending once Meta approves, and that the rest of the flow runs immediately.
