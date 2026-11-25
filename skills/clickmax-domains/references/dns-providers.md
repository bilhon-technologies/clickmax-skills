# DNS provider steps

The user does every step here in their provider's panel. Values (target, TXT) always come from `mcp__plugin_clickmax_clickmax__domains_get.dnsRecords`; the examples below use `www.mystore.com` and the usual v4 CNAME target `domains.clickmax.space` only to illustrate.

Menu names differ per provider and change often: describe the step ("open the DNS zone of mystore.com", "add a CNAME record"), and use `mcp__plugin_clickmax_clickmax__docs_search` with the provider name for provider-specific help pages instead of inventing menu paths. The root-domain how-to (`mcp__plugin_clickmax_clickmax__docs_pages_get` path `/features/domains/root-domain`) is the user-facing version of the two setups below — quote it when the user wants a link.

## Pick the setup

|provider can CNAME at the root (flattening / ALIAS / ANAME)?|domain version|register in Clickmax|root `mystore.com`|
|-|-|-|-|
|no (IONOS, most registrars)|any|`www.mystore.com`|provider HTTP redirect of the root → `https://www.mystore.com`|
|yes (e.g. Cloudflare DNS)|`v4`|`www.mystore.com` (default) or `mystore.com` if the user wants the bare address as the main one|direct: CNAME/ALIAS `@` → target from `dnsRecords`|
|yes|`v3`|`www.mystore.com`|provider redirect, as in row 1|

Unsure whether the provider supports CNAME at the root → assume no and use row 1; it works everywhere.

## `www` + root redirect (providers without CNAME at the root, e.g. IONOS)

1. Clickmax: `www.mystore.com` registered (`mcp__plugin_clickmax_clickmax__domains_register`).
2. Provider, DNS zone of `mystore.com`: create (or edit) the record from `dnsRecords` — `CNAME`, name `www`, value = target. If `www` already has A/AAAA records or a "redirect/forwarding" configured, remove them first: a hostname cannot have a CNAME together with other records.
3. Provider, root only (`mystore.com`, sometimes shown as `@`): configure the provider's HTTP redirect / domain forwarding to `https://www.mystore.com` (permanent / 301 when offered).
   - NEVER put the redirect on `www`: the provider swaps the `www` CNAME for its own A records and Clickmax stops serving.
   - Turn on the provider's SSL/HTTPS for the root (often a free certificate option on the domain). Without it, `https://mystore.com` shows a certificate error before redirecting; `http://` works but browsers default to `https://`.
4. Clickmax: user confirms → `mcp__plugin_clickmax_clickmax__domains_sync` → `connected`.
5. Test both: `https://www.mystore.com/` (Clickmax content) and `https://mystore.com` (must land on www).

## Root pointed directly (v4 + provider with flattening)

1. Clickmax: register the hostname the user wants as main (`mystore.com` or `www.mystore.com`).
2. Provider: record from `dnsRecords` with name `@` for the root (the provider flattens it); keep proxy/CDN features of the provider OFF for this record when it has them (DNS only), otherwise the check sees the provider instead of the target.
3. The other hostname (`www` when the root is main, or the reverse) → provider redirect to the main one, or register it too only if the user wants both serving content.

## Diagnose broken cases

|symptom (`domains_get` / user report)|likely cause|fix|
|-|-|-|
|`status: connected` + `cnamePointsToTarget: false`, site stopped loading|record at the provider changed after connecting (new provider, DNS edit, redirect added)|ask what changed; restore the CNAME from `dnsRecords`, then `domains_sync`|
|`cnamePointsToTarget: false`, status stuck `disconnected`/`connecting`|CNAME missing, typo in value, created on the wrong name (`www.mystore.com.mystore.com`)|recreate with `relativeHost` exactly; full `host` only if the provider asks for the full name|
|`www` resolves to the provider's IP / provider "parking" page; `cnamePointsToTarget: false`|provider redirect/forwarding was set on `www`, replacing the CNAME with A records|remove the redirect from `www`, recreate the CNAME, move the redirect to the root only|
|`securing` with `cnamePointsToTarget: true`|certificate being issued|wait a few minutes, `domains_sync` later; no DNS change|
|`connected` but the domain home opens an error/blank page or 404|`hasRootContent: false` (nothing on `/`)|put a page/funnel page on `/` or set `domains_root_redirect_update` (Execute guide in SKILL)|
|`connected`, but `mystore.com` (no www) does not open or shows a certificate error|root redirect missing at the provider, or provider SSL off for the root|configure the root redirect + provider SSL (section above)|
|page opens on the domain but the next funnel step changes address|page and funnel on different domains|align `domainId`: page via `pages_update`, funnel via `funnels_update`, then republish (see `clickmax-funnels`)|
|provider refuses changes / domain suspended|domain blocked at the provider (unpaid invoice, pending data confirmation)|user resolves with the provider; `docs_search` "domínio bloqueado"|
|records right for >1 h and status went back to `disconnected`|automatic checking window ended|`domains_sync` restarts it|
