---
name: clickmax-domains
description: Use when the user wants to connect, check, fix, redirect, or remove a custom domain in Clickmax, or a site/page on their domain is down, not opening, or shows a certificate error.
---

## When this applies

The user wants a domain they own (`www.mystore.com`) serving their Clickmax pages/funnels, a domain stuck in `disconnected`/`connecting`/`securing`, a domain that opens blank/error/someone else's site, the home address `/` of a domain decided, or a domain removed.

Not this skill:

|request|skill|
|-|-|
|buy a new domain|answer from the help center (`clickmax-support`); there is no buy tool|
|e-mail sending on the domain (remetente, e-mail DNS records, `email_domains_*`)|`clickmax-integrations`; the domain must exist here first|
|"is my launch ready" across funnel/checkout/channels|`clickmax-launch-readiness`; it hands domain-hosting problems back here|
|page lives on a site/domain outside Clickmax (Framer, Webflow, own server)|`clickmax-external-pages`|
|page content/layout|`clickmax-pages`; funnel graph → `clickmax-funnels`|
|short links/redirect links on a connected domain|redirect links tools (`redirect_domains_list`)|

## Proactive check

Run the check (below) without waiting to be asked when ANY of these holds:

|signal|examples|
|-|-|
|user asks to check/verify a domain|"verifica meu domínio", "check my domain", "está tudo certo com www.loja.com?"|
|conversation hints at a domain problem|site/page down or not opening, SSL/certificate warning, 404 / "página não encontrada" on the home address, "mexi no DNS", changed provider or set a redirect at the provider, "aparece conectado mas não abre"|
|a domain read shows an inconsistent state|`status: connected` + `cnamePointsToTarget: false` · connected but `hasRootContent: false` (nothing on `/`)|

Check = once per turn: resolve the domain (`domains_list` `search`) → `mcp__plugin_clickmax_clickmax__domains_get` → `mcp__plugin_clickmax_clickmax__domains_sync` (read `hasRootContent` from either) → `mcp__plugin_clickmax_clickmax__domains_usages_get` only to name what sits on `/` or what a delete would take down. Then name the probable cause from [broken cases](references/dns-providers.md#diagnose-broken-cases) in plain words, offer the Clickmax-side fix (`domains_root_redirect_update`, page `path: '/'` via `pages_update`, funnel `domainId` via `funnels_update`) and give provider-side steps for the rest. Fixes stay opt-in.

Reading the signals:

- `cnamePointsToTarget: null` = not measured (domain not on the current version, or the check is unavailable) → never conclude broken DNS from it; rely on `status` and what the user sees.
- `cnamePointsToTarget: false` = strong hint, not proof: a failed lookup also reads `false`. Corroborate (status not `connected`, user's provider screen shows other records, A records at the provider IP) before telling the user to change DNS.
- `domains_sync` `queued: true` (v2/v3) = check only queued, the returned state predates it → diagnose from it as "last known", never as the fresh result.
- `domains_sync` `THROTTLED` = checked moments ago → use the returned state, do not call again this turn; `SYNC_FAILED` = say the check could not run now and diagnose from `domains_get`.
- 404 on a subpath (not `/`) = page path/publication problem → `clickmax-pages`/`clickmax-funnels`, not DNS.

## Key assumptions

- Split of work = Clickmax side via tools | DNS provider side (IONOS, GoDaddy, Registro.br, Hostinger, Cloudflare…) by the USER. Never ask for, accept, or store provider login/password/API key/2FA code; if the user pastes one, tell them to remove it and change it. Provider work = step-by-step guidance, or the **Configurar automaticamente** button on the domain screen (provider login happens in that window, not in the chat).
- Registered hostname = one exact name. `www.mystore.com` and `mystore.com` are two different hostnames; only the registered one serves Clickmax content.
- Default = register `www.<domain>`. Bare root (apex, `mystore.com`) as the registered hostname only on `version: v4` AND a provider with CNAME at the root (CNAME flattening / ALIAS / ANAME). Everything else → `www` + provider-side redirect of the root → [DNS providers](references/dns-providers.md).
- Records to create = `domains_get.dnsRecords` (v4): `relativeHost` goes in the provider "name/host" field (`www`, `@`), `host` only when the provider wants the full name, `value` verbatim. Never retype a target from memory. `v3` = no `dnsRecords` in the tool → automatic setup button, or the manual record from `docs_search`.
- Status ladder = `disconnected` → `connecting` (ownership) → `securing` (certificate/CNAME) → `connected`. `cnamePointsToTarget` = live DNS read (v4): `false` = the provider still answers something else for that hostname.
- Home path `/` of the registered hostname answers, in this order: domain `rootRedirectUrl` (overrides everything) → a published page/site/redirect link/live funnel step with `path: '/'` on that domain → nothing ("page not found"). `domains_get.hasRootContent` already evaluates all of these: `false` = nothing on `/`, the domain looks broken to visitors even when `connected`.
- Permissions are enforced by the API: without domain management rights the write tools fail; report it, never retry in a loop.
- `domains_register` consumes custom-domain quota; never register "to test" or register duplicates (`DOMAIN_ALREADY_REGISTERED` = it exists in this or another workspace).

## Thought process

1. Goal first: which address should open what (`www.mystore.com` → sales page? funnel? redirect to Instagram?). Ask only when the hostname or the destination is truly unknown.
2. `domains_list` (`search`) → exists? connected? which version? Never register before this.
3. Missing → resolve project (`projects_list` → `id`) → `domains_register` with the hostname.
4. Not `connected` → `domains_get` → show the record(s) as a table + provider steps for THEIR provider ([DNS providers](references/dns-providers.md)), offer the automatic setup button as the easy path.
5. User says done → `domains_sync` → read status/`cnamePointsToTarget`; diagnose with the [broken cases](references/dns-providers.md#diagnose-broken-cases) before blaming propagation.
6. Connected → make sure `/` has content (see Execute guide) → report.

## Execute guide

- Lookup: `mcp__plugin_clickmax_clickmax__domains_list` with `search` = the bare domain (`mystore`) so both `www.` and apex rows show up.
- Register: `mcp__plugin_clickmax_clickmax__domains_register` with `name` = full hostname (lowercase, no `https://`, no path) + `projectId` (uuid from `mcp__plugin_clickmax_clickmax__projects_list`; default project when the user does not care). The tool picks the version; read `version` + `dnsRecords` from the result.
- DNS instructions: `mcp__plugin_clickmax_clickmax__domains_get` → render `type | name (relativeHost) | value`. Then provider steps from [DNS providers](references/dns-providers.md). Root domain (`mystore.com`) without `www`: read the help page with `mcp__plugin_clickmax_clickmax__docs_pages_get` path `/features/domains/root-domain` ("Como fazer meu domínio abrir sem o www?") and quote its steps. Other provider UI specifics or the v3 manual record → `mcp__plugin_clickmax_clickmax__docs_search` (e.g. "conectar domínio", provider name).
- Recheck: `mcp__plugin_clickmax_clickmax__domains_sync` after the user confirms the record exists. `checked: true` = fresh v4 state | `queued: true` = v2/v3 check queued, confirm with `domains_get` on the user's next message | `THROTTLED` = checked moments ago, wait ~1 min; `SYNC_FAILED` = retry later. `securing` with `cnamePointsToTarget: true` = wait minutes, do not ask the user to change DNS.
- Home path `/`, pick ONE with the user:
  - existing standalone page → `mcp__plugin_clickmax_clickmax__pages_update` `data: { domainId, path: '/' }` → `mcp__plugin_clickmax_clickmax__pages_publish` if it was never published (published pages move by themselves).
  - funnel → new: `mcp__plugin_clickmax_clickmax__funnels_create` with `domainId` | existing: `mcp__plugin_clickmax_clickmax__funnels_update` with `domainId` (`null` = back to the project's public domain; check the current one in `mcp__plugin_clickmax_clickmax__funnels_get` `settings.domain`). Its entry page gets `path: '/'` via `pages_update`. Go live with `mcp__plugin_clickmax_clickmax__funnels_publish`: a live funnel keeps serving on the old domain (`unpublished_changes`) until republished, and its pages with their own `domainId` must match (see `clickmax-funnels`).
  - redirect → `mcp__plugin_clickmax_clickmax__domains_root_redirect_update` with `rootRedirectUrl` (`https://…`). Clear it (`null`) before putting a page on `/`.
  - Check first: `domains_get.hasRootContent` (`true` = something already answers; `rootRedirectUrl` set = it is the redirect) and, to name the page, `mcp__plugin_clickmax_clickmax__domains_usages_get` page items with `path: '/'`. After the change, `domains_get` again → `hasRootContent: true`.
- Delete: `mcp__plugin_clickmax_clickmax__domains_usages_get` → list what goes offline (pages, funnels, forms, short links, e-mail senders) → explicit user confirmation → `mcp__plugin_clickmax_clickmax__domains_delete`. `DOMAIN_RECENTLY_CHECKED` = retry in ~5 min. Remind the user the provider records stay until they remove them.

## Report

- Open with the hostname + status in plain words (`www.mystore.com: conectado e no ar com HTTPS` / `aguardando o registro CNAME`).
- DNS work = a short table (Tipo | Nome | Valor) + numbered provider steps; one provider, the user's.
- End with what opens on `https://<hostname>/` and, when the apex is in play, what the user must still do at the provider.
- Mutations beyond the asked goal (delete, changing a live page's domain, redirecting a domain that already serves pages) = offer, never do unasked.

## Warnings

- Provider "redirect/forwarding" set on `www` replaces the `www` CNAME with the provider's own A records → Clickmax stops serving. Redirect only the root, never `www`.
- Changing `domainId`/`path` of a published page changes its live address immediately; old links break.
- `rootRedirectUrl` silently hides any page on `/` of that hostname.
- A `v2` domain cannot receive pages (`publishable: true` filter hides it).
- Propagation can take up to 48 h, but most failures are a wrong record: check before telling the user to wait.

## Anti-patterns

- Asking for the provider password, API token, or to "log in for them".
- Registering `mystore.com` when the provider cannot do CNAME at the root, or registering both `mystore.com` and `www.` "just in case".
- Inventing the CNAME target or TXT values instead of reading `domains_get`.
- Reporting "domain connected" while `hasRootContent` is `false`.
- Calling `domains_sync` in a loop, or again after `THROTTLED`, instead of waiting for the user to change DNS.
- Telling the user DNS is broken from `cnamePointsToTarget: null`, or from a lone `false` without corroboration.
- Answering a "site down / certificate error" complaint about a custom domain from the help center alone, without reading the domain state.
- Deleting and re-registering a domain to "reset" it.

---

Clickmax skill revision: `0858cc6123fb`
