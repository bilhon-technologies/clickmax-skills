---
name: clickmax-launch-readiness
description: Use when the user is about to launch (or just launched) and wants to know if a funnel or the whole Clickmax account is ready, or asks why a funnel/launch "is not working", so every area is checked at once and each problem comes back with its cause and where to fix it.
---

## When this applies

- The user says they are launching ("vou lançar hoje/amanhã", "abro o carrinho às 20h", "I'm launching", "voy a lanzar") or asks if a funnel / the account is ready to sell.
- "Status do meu funil/conta", "checa meu funil", "por que meu funil/lançamento não está funcionando" with no single symptom named.
- Before publishing a funnel the user will push traffic to right away.

Not this skill:

- one named symptom with an obvious owner → that skill directly: WhatsApp/e-mail connection only → `clickmax-integrations`; external script only → `clickmax-external-pages`; declined payments → `clickmax-failure-diagnosis`; wallet/withdrawal only → `clickmax-wallet-receivables`
- open "what should I do now" with no launch → `clickmax-account-suggestions`
- funnel performance numbers (conversion, where people drop) → `clickmax-funnels` / `clickmax-analytics`
- how a feature works → `clickmax-support`
- fixing a finding (connect a trigger, bind an offer, edit a flow) → the owning skill after the report, only on the user's go-ahead

## Key assumptions

- Read-only diagnosis. Nothing is changed while checking; fixes are offered afterwards.
- Scope: funnel named or shown on screen → that funnel | "minha conta", "tudo" → account checks + every funnel with `status` `published`/`unpublished_changes` (more than 5 → ask which ones) | nothing said → the only published funnel, else ask which. Account-wide checks (seller, channels, e-mail) run in every scope: a funnel cannot sell without them.
- A problem only counts as a blocker when the launch path USES the resource. A disconnected WhatsApp number no active automation or scheduled campaign sends from = info, not a blocker.
- Tracking: `pages_tracking_status.lastPageViewAt: null` means no visit was ever recorded. That is also what a page with no traffic yet looks like. Flag it only for an EXTERNAL page (`editorVersion: external`) that is a node of a published funnel, as "script not seen yet", and ask the user to open the live URL once and re-check before calling it "not installed". Builder/Max Design pages get the script automatically; no views there = no traffic, never a script problem.
- `funnels_validate.stepsWithoutNextStep` may be a deliberate last step (a checkout that ends the funnel, an upsell decline that ends). Report it as a question, never as a confirmed error.
- Every link points to the real screen: a screen-lookup tool (screen ids below) or the entity's own link (funnel, automation, page, offer). Never type a route from memory.

## Thought process

1. Resolve the scope (above). Ambiguous funnel → ask once, listing the published funnels.
2. Run the reads in 3 waves; each wave = independent reads in parallel. Do not stop at the first problem.
3. Classify each finding with the severity rules, attach evidence (names, counts, dates) and the fix link.
4. Separate "you can fix" from "looks like a platform bug" (below).
5. Report blockers first, then warnings, then what was checked and is OK, then what could not be verified.

## Execute guide

Wave 1 (account, parallel):

- `mcp__plugin_clickmax_clickmax__sellers_status` · `mcp__plugin_clickmax_clickmax__channel_instances_list` · `mcp__plugin_clickmax_clickmax__channel_instances_alerts_list` · `mcp__plugin_clickmax_clickmax__email_sender_signatures_list` (`perPage: 200`) · `mcp__plugin_clickmax_clickmax__email_domains_list` (`perPage: 200`) · `mcp__plugin_clickmax_clickmax__broadcasts_list` (`perPage: 20`) · `mcp__plugin_clickmax_clickmax__flows_list` (`modes: ["active", "scheduled"]`, plus `funnelId` in funnel scope) · `mcp__plugin_clickmax_clickmax__funnels_list` (account scope or to resolve the funnel).
- Paginated lists (`meta`): read every page of senders, domains, automations and funnels before saying none is broken. Campaigns: read pages until they pass the last 7 days.

Wave 2 (per funnel in scope, parallel):

- `mcp__plugin_clickmax_clickmax__funnels_get` (nodes with `pageId`, `config`, domain) · `mcp__plugin_clickmax_clickmax__funnels_structure_get` (labels, `pageType`) · `mcp__plugin_clickmax_clickmax__funnels_validate` · `mcp__plugin_clickmax_clickmax__funnels_automations_list`.
- Unusable senders/numbers from wave 1 → `mcp__plugin_clickmax_clickmax__flows_list` with `emailSenderIds` / `whatsappNumberIds` = those ids and `modes: ["active", "scheduled"]` to know which automations depend on them.

Wave 3 (drill-down, parallel, only where wave 1–2 point):

- `mcp__plugin_clickmax_clickmax__pages_tracking_status` with every `pageId` of the funnel nodes (max 50 per call) → keep `editorVersion: external`.
- Checkout nodes (`pageType: checkout`) → `mcp__plugin_clickmax_clickmax__checkouts_get` → offer → `mcp__plugin_clickmax_clickmax__offers_get` (`status`); not `active` → `mcp__plugin_clickmax_clickmax__offers_get_incomplete_reason`.
- `external_checkout` nodes → `mcp__plugin_clickmax_clickmax__external_checkout_accounts_list` (`setup` not null = integration incomplete).
- Each active/scheduled automation of the funnel (`funnels_automations_list` + `flows_list`; cap 15, say how many were left out) → `mcp__plugin_clickmax_clickmax__flows_validate`; the ones with executions → `mcp__plugin_clickmax_clickmax__flows_executions_list` (`perPage: 1`, read `counts.failed` + `topErrors`).
- Unverified/paused domain that hosts a used sender → `mcp__plugin_clickmax_clickmax__email_domains_dns_get` (which record is missing + value).
- WhatsApp Web number in use → `mcp__plugin_clickmax_clickmax__channel_instances_health_get`.
- Campaign `scheduled` → `mcp__plugin_clickmax_clickmax__broadcasts_get` (sender / template still valid); `failed` in the last 7 days → `mcp__plugin_clickmax_clickmax__broadcasts_get` `dispatchError`.

Checklist (signal → finding):

|Area|Signal|Severity|
|-|-|-|
|Receiving|`sellerStatus` ≠ `enabled` (`reason`)|blocker|
|Receiving|`bankAccountStatus` ≠ `approved`|warning: sells, cannot withdraw|
|Offer|checkout offer `status` ≠ `active` (`pending_approval` = under review; `draft` + incomplete reasons)|blocker|
|Checkout|`checkoutsMissingOffer`|blocker: charges nothing|
|Checkout|`externalCheckoutIssues` or external checkout `setup` not null|blocker|
|Funnel|`nodesMissingPage`, `danglingTriggers`, `workflowsMissingFlow` in a published funnel|blocker|
|Funnel|`stepsWithoutNextStep`|warning, ask: "is this the last step on purpose?"|
|Funnel|`disconnectedTriggers`, `orphanNodeIds`, funnel `unpublished_changes`|warning|
|External page|external node, `lastPageViewAt: null`|warning "script not seen", blocker after the user opened the page and it is still null|
|External page|old `lastPageViewAt` + `recentPageViews: 0`|blocker: script stopped firing|
|WhatsApp|number used by an active automation/scheduled campaign: `status` ≠ `live`, `evolution.connectionState` ≠ `open`, `waha.status` ≠ `WORKING`, `bannedAt`, alert `severity: blocker`|blocker|
|WhatsApp|`quotaPausedAt`, health `band` `critical`/`paused`, `warmup.active` with a large send planned, alert `warning`|warning|
|E-mail|sender used by an active automation/scheduled campaign: `confirmed: false`, `revokedAt` set, its domain `postmarkVerified: false` or `emailPausedAt` set|blocker|
|Automation|`flows_validate` `error` issue in an active/scheduled automation|blocker|
|Automation|funnel automation `mode: closed` (paused), `counts.failed` growing, recurring `topErrors`|warning|
|Campaign|scheduled campaign whose sender/template is unusable|blocker|
|Campaign|`failed` with `dispatchError`, high `bounced`|warning|

Fix link per finding (screen id for a screen-lookup tool, or the entity's own link):

- seller / KYC → `seller-account` (or `wallet`) | bank → `bank-details`
- offer not active → the offer itself | checkout without offer → the funnel (bind the offer on the checkout page)
- funnel steps / triggers / pages → the funnel itself | external checkout integration → `setup.openPath`
- external page script → give `headScript` from `mcp__plugin_clickmax_clickmax__pages_get_external_script` inline (paste before `</head>`) + the page; button that does not advance → the redirect markers of `clickmax-external-pages` (`cm-checkout-button`, `cm-next-funnel-node`, or `data-cx-redirect-url` = the next step's public URL from `mcp__plugin_clickmax_clickmax__pages_get`)
- WhatsApp number / alerts → `connections-whatsapp` | template paused or rejected → `message-templates`
- sender / domain DNS → `connections-email` (records from `email_domains_dns_get`) | domain not in the workspace → `domains`
- automation → the automation itself | campaign → `broadcasts`

User-fixable vs platform bug:

- User-fixable = the cause is configuration (disconnected, unverified, missing, not approved, not connected). Give the fix.
- Probable bug = no configuration cause explains it: `topErrors`/`dispatchError` with an internal or unexplained error, a check that answers a server error twice, external page with recent views but the funnel never advances although markers are installed, state that contradicts itself (sender `confirmed` and verified domain, still failing). Say plainly it looks like a platform problem, include the evidence, and offer to open a support ticket with it through the support-escalation operation (`support_escalate`) when the connector has it; otherwise point to the official support channel (`clickmax-support`). Escalate only after the user agrees.

## Report

- Opening line: scope + verdict + counts ("Funil Lançamento X: 2 bloqueios e 3 avisos antes de abrir o carrinho"). No blockers → say it is ready, still list the warnings.
- Blockers first, then warnings; inside each, the order a buyer walks (receiving → offer/checkout → funnel/pages → messages). One item = what (resource name) → why (evidence in plain words, with the real date/count) → how to fix (one action) → link. Cap 8 items; more → "+N avisos menores" on request.
- Questions the user must answer (deliberate last step, did you open the external page) go with the item, not at the end.
- One line "Conferido e OK: …" naming the areas with no finding, so silence is not ambiguous.
- "Não deu para verificar": what was skipped or capped (automations beyond 15, funnels not chosen, checks that failed) and why.
- Close with ONE offer: fix what can be fixed here (bind an offer, connect a step, send the offer to approval after the offer skill's questions, hand over the script) or escalate a probable bug. Actions only after the user accepts.
- Never show ids, field names or raw statuses; translate (`pending_approval` → "oferta em análise").

## Warnings

- Do not mark a resource as broken from one ambiguous signal: no traffic ≠ script missing; `stepsWithoutNextStep` ≠ broken; an unused disconnected number ≠ blocker.
- Restricted members only see what they are allowed to; an empty channel or funnel list may be a permission limit. Say so instead of reporting "nothing connected".
- `email_domains_dns_get` reflects the provider's last check; a record created minutes ago may still read missing until the user clicks verify.
- Re-check after the user says they fixed something: run only the reads of that finding again.

## Anti-patterns

- Checking one area, finding a problem and stopping there.
- Reporting "script not installed" for a page that simply had no visits.
- Dumping `funnels_validate` arrays or tool output instead of named steps and causes.
- Giving a generic "go to settings" instead of the exact screen or entity link.
- Fixing things during the check without the user's go-ahead.
- Calling a configuration problem a bug, or escalating before offering the fix the user can do.

---

Clickmax skill revision: `53fac6a70402`
