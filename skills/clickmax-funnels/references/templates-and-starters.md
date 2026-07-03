# Funnel templates and starters

## Sequence templates

Use `funnels_sequence_create` when the user wants a standard starter graph.

Supported starters:

- `sales`
- `lead-magnet`
- `webinar`
- `tripwire`
- `vsl-auto`
- `upsell-downsell`

## When to prefer a template

Use a template when the user describes a common funnel family and does not need unusual routing logic.

Examples:

- sales page + checkout + thank you -> `sales`
- opt-in for a free asset -> `lead-magnet`
- webinar registration / reminder / replay shape -> `webinar`
- low-ticket front-end into upsell path -> `tripwire`
- VSL-first automated sales journey -> `vsl-auto`
- checkout plus one-click upsell/downsell structure -> `upsell-downsell`

## When to build manually

Build manually when:

- the user describes custom pages and custom trigger routing
- you need conditional branches or traffic-source outputs beyond the standard starter
- you need to attach existing editor pages in a non-standard sequence

## Funnel sequence templates vs page templates

These are different things:

- `funnels_sequence_create` templates (above) scaffold the GRAPH — the nodes and their trigger connections. They do NOT create page content; the page nodes start empty.
- Clickmax PAGE templates (`pages_templates_list`) are ready-made page designs (including checkout pages). Use them to give a node real content. To create a page with a checkout bound to an offer, see [pages and checkout](pages-and-checkout.md).
