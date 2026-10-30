---
name: clickmax-integrations
description: Use when the user wants to connect or check a messaging channel (WhatsApp, Instagram, Telegram, e-mail), or create, list or submit WhatsApp templates outside an automation.
---

## When this applies

1. **Connect / check a channel** — "quero conectar meu Instagram", "meu WhatsApp está conectado?", "reconectar o WhatsApp".
2. **WhatsApp templates without an automation** — create, list, submit for Meta review, "meus templates foram aprovados?".

Not this skill:

- A template that goes INTO an automation step (`flows_send_whatsapp`) → `clickmax-flows`; this skill only stops at an approved/pending template + the templates screen.
- Mailbox health, signature, history import of an already connected mailbox → `clickmax-email-inbox`.
- "How does X work" with no account action → `clickmax-support`.

## Key assumptions

- Connecting ALWAYS happens on the user's screen (Meta/Instagram OAuth popup, QR code, BotFather token, DNS, app password). No tool connects, disconnects or reconnects a channel → never ask for a password, token or code in chat.
- Status = `mcp__plugin_clickmax_clickmax__channel_instances_list` (archived hidden by default). `channel` → `gupshup` = WhatsApp **API oficial** | `whatsapp_unofficial` = **WhatsApp Web** (QR) | `instagram` | `telegram` | `email` = connected mailbox (IMAP/SMTP).
- `status` → `live` = working | `pending` / `connecting` = connection started, not finished (wizard shows "Conexão incompleta" → **Continuar conexão**) | `awaiting_takeover_confirmation` = account already connected in another Clickmax workspace, waiting the user's **Mover conta para cá** | `error` / `deactivated` = lost access → reconnect the SAME account (conversations, history and automations come back as they were).
- Restricted members only see the accounts assigned to them → empty list ≠ "nothing connected" for a non-owner; say so instead of asserting.
- E-mail has two different things: **sending** (domain + confirmed sender for broadcasts/automations) = `mcp__plugin_clickmax_clickmax__email_sender_signatures_list`, `confirmed: true` = can send | **mailbox** (support e-mails into live chat) = `channel: email` row. No tool returns domain DNS status → `confirmed: false` = "verification not finished on the screen", never a guessed DNS cause.
- Prerequisites, in plain words:
  - WhatsApp API oficial: Facebook account that administers the Business Manager + verified business with a WhatsApp number + access to the number to receive the verification code. Needed for Meta templates, broadcast outside the 24h window, verified badge.
  - WhatsApp Web: just the phone with WhatsApp to read the QR. No Meta approval, no templates. Not endorsed by WhatsApp → excessive sending/spam can get the number banned, and a banned number does not come back.
  - Instagram: **professional** account (Business or Creator, personal is refused) + login to authorize with Meta + permission to manage the account's Directs.
  - Telegram: a bot created in **@BotFather** (`/newbot`) and its access token.
  - E-mail sending: own domain already connected to the workspace + access to its DNS panel. Mailbox: IMAP/SMTP enabled + 2-step verification + an **app password** (the normal password does not work).
- WhatsApp templates exist ONLY on API oficial numbers (`gupshup`). WhatsApp Web sends free text and never uses Meta templates.
- Template `status` → `draft` = Rascunho (local, not at Meta) | `pending` = Em análise (hours to days) | `approved` = Aprovado (sends outside the 24h window) | `rejected` = Reprovado | `failed` = Falha no envio (never reached Meta) | `paused` = Pausado (low quality); `rejected`/`failed`/`paused` → read `reason`. `oldCategory` set = Meta reclassified it (e.g. UTILITY → MARKETING) → report both.
- Authoring rules (variables, sample, buttons, lengths) are enforced by `mcp__plugin_clickmax_clickmax__gupshup_templates_create`, whose error lists everything to fix; wiring into a step → `clickmax-flows`.

## Thought process

1. Which channel / which job? WhatsApp connect without a choice → ask official vs Web in ONE question with the trade-off (templates + broadcast + Meta approval vs QR now + ban risk).
2. Read the real status before saying anything about it.
3. Already `live` → say so with the identity (number, @username, address) and offer the next step; never send the user to reconnect a working account.
4. Missing / broken → prerequisites (only those of the chosen option) → open the screen.
5. Templates: resolve the official number → read → write only after the user confirms the exact content.

## Execute guide

Connect / check (target: 1 read + 1 screen lookup, no long codemode):

- WhatsApp / Instagram / Telegram / mailbox: `mcp__plugin_clickmax_clickmax__channel_instances_list` → filter by `channel`. E-mail sending: `mcp__plugin_clickmax_clickmax__email_sender_signatures_list`.
- Open the screen with a screen-lookup tool when available (Max: `find_screen`), never a hardcoded route. Ids: `connections-whatsapp` (both WhatsApp options, the wizard asks "Como você quer conectar o WhatsApp?") | `connections-instagram` | `connections-telegram` | `connections-email` (sending domain + senders) | `email-inbox` (mailbox) | `domains` (domain not connected to the workspace yet).
- User says they finished → read the status again; `live` → confirm + next step (an automation that uses the channel → `clickmax-flows`; Instagram comment/DM automation; first WhatsApp template below).

Templates (API oficial only):

- `mcp__plugin_clickmax_clickmax__channel_instances_list` → `channel: gupshup`, `status: live`. None → say templates need the API oficial, mention WhatsApp Web does not use them if they only have `whatsapp_unofficial`, and go to the connect flow above. Several → ask which number (pass its `id` as `channelInstanceId`).
- List / "foram aprovados?": `mcp__plugin_clickmax_clickmax__gupshup_templates_list` with `channelInstanceId` (+ `status` filter when asked) → real status per template; `rejected` / `paused` → `reason` + offer to fix (`mcp__plugin_clickmax_clickmax__gupshup_templates_update` only on `draft`/`failed`; a rejected one is re-created under a new name).
- Create, cheapest path first:
  1. an existing `approved` template already saying it → reuse, nothing to submit;
  2. generic UTILITY/AUTHENTICATION use (order update, reminder, OTP) → `mcp__plugin_clickmax_clickmax__gupshup_template_library_list` → `mcp__plugin_clickmax_clickmax__gupshup_template_library_create` (submits to Meta immediately → confirm first; `libraryTemplateName` only from the list);
  3. otherwise draft the copy from the user's text (welcome = usually `MARKETING`; `UTILITY` only for transactional) → `mcp__plugin_clickmax_clickmax__gupshup_templates_create` → check `created` (`false` = existing template returned, read its status) → show the exact final copy (name, category, body, sample, buttons) → explicit yes → `mcp__plugin_clickmax_clickmax__gupshup_templates_submit`.
  - Max: the copy goes in ONE `cx-message channel="whatsapp" template="<elementName>"` card (+ `header`/`footer`/`buttons`); category and sample stay in prose. Attribute values come from the user or the tools → escape `&` `"` `<` `>` as `&amp;` `&quot;` `&lt;` `&gt;`; `|` only separates buttons, so a label containing `|` gets `/` instead. User comments arrive as `<conversation_selection_context>` → `mcp__plugin_clickmax_clickmax__gupshup_templates_update` the draft (still `draft`/`failed`) → show the revised card → ask again before submitting.
- EVERY templates reply (list, status, create, submit) ends with the templates screen of that number: `connections-whatsapp` lookup → the `whatsappConnections` entry of the chosen number → its `templatesPath`.

## Report

- Status, one line per account: name/number/@ → Conectado | Conexão incompleta | Aguardando você mover a conta | Perdeu o acesso (reconecte a mesma conta) | Remetente não verificado.
- Not connected → max 3 bullets of prerequisites for the chosen option, then the single screen link.
  - Max: `cx-cta action="open-page"` with the exact `path` from `find_screen` ("Abrir conexões do Instagram"); the wizard steps stay in prose, at most 2–3 lines.
- Already connected → "Seu Instagram @x já está conectado." + one next-step offer, no reconnect link.
- Templates: name → status (+ category, + `reason` when rejected/paused). After submit: "Enviado para a Meta; a análise leva de horas a alguns dias. Enquanto estiver pendente ele já pode entrar numa automação, mas só envia depois de aprovado."
  - Max: every templates reply (list and status answers too, not only after a submit) closes with `cx-cta action="open-page"` to the number's `templatesPath` ("Ver templates do WhatsApp").

## Warnings

- `mcp__plugin_clickmax_clickmax__gupshup_templates_submit` and `mcp__plugin_clickmax_clickmax__gupshup_template_library_create` are external and practically irreversible (review queue, number quality rating) → never without the user confirming the exact content in this conversation.
- Never invent copy, button URLs or sample values the user did not give; ask for the missing piece.
- Moving an account from another workspace or connecting a number already in use disconnects the old connection irreversibly → the user confirms it on the screen; do not push them through it.
- A `find_screen` result `no-permission` → the role cannot open it; suggest asking the workspace owner, no link.

## Anti-patterns

- Telling the user to reconnect when the status read says `live`.
- Answering "está conectado?" from memory or from an automation's config instead of the status read.
- Offering Meta templates for a WhatsApp Web number, or WhatsApp Web as a way to "skip template approval" for broadcasts without the ban warning.
- Hardcoded `/settings/...` routes instead of the screen lookup.
- Asking for passwords, BotFather tokens, QR codes or verification codes in chat.
- Submitting a draft "to see if it gets approved".

---

Clickmax skill revision: `a76ae3e6d3e1`
