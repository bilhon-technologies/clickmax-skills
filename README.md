# Clickmax Skills

Plugin oficial da Clickmax para o [Claude Code](https://claude.com/claude-code): skills que ensinam o Claude a operar a plataforma Clickmax — CRM, leads, funis, fluxos de automação, produtos, ofertas, pagamentos, área de membros e mais — através do servidor MCP da Clickmax.

> ⚠️ **Repositório gerado automaticamente.** O conteúdo é publicado a partir do monorepo interno da Bilhon. Edições manuais serão sobrescritas na próxima sincronização. Encontrou um problema? Abra uma issue.

## Instalação

No Claude Code:

```
/plugin marketplace add bilhon-technologies/clickmax-skills
/plugin install clickmax@clickmax-skills
```

## Autenticação

O plugin conecta ao servidor MCP da Clickmax (`https://ai.clickmax.io/mcp`) usando um token de API lido da variável de ambiente `CLICKMAX_API_TOKEN`:

```bash
export CLICKMAX_API_TOKEN="seu-token-aqui"
```

Gere seu token de API no painel da Clickmax.

## Skills incluídas

- **clickmax-classrooms** — Use when the user wants to list, inspect, create, update, link content to, copy members between, or delete classrooms inside Clickmax member portals.
- **clickmax-external-pages** — Use when the user wants to connect an external page/site to Clickmax tracking or forms using Clickmax page scripts.
- **clickmax-flows** — Use when the user wants to create, inspect, change, validate, or activate/archive a Clickmax automation flow and its step graph.
- **clickmax-forms-quizzes** — Use when the user wants to safely create, edit, publish, inspect, or analyze Clickmax forms and quizzes.
- **clickmax-funnels** — Use when the user wants to create, inspect, change, publish, deactivate, delete, or analyze a Clickmax funnel graph.
- **clickmax-leads** — Use when the user wants to find, inspect, filter, or compare CRM leads and their commercial context inside Clickmax.
- **clickmax-leads-activity-analysis** — Use when the user wants to inspect CRM activity streams, event timelines, or activity-derived metrics for leads and opportunities.
- **clickmax-list-segments** — Use when the user wants to create, inspect, update, reload, or use manual lists and dynamic segments to group leads in Clickmax.
- **clickmax-members** — Use when the user wants to inspect, create, update, enable, disable, enroll, or remove member users and their access/progress in Clickmax Members.
- **clickmax-modules** — Use when the user wants to list, inspect, create, update, reorder, or delete modules and lesson ordering inside a Clickmax Members course.
- **clickmax-offers** — Use when the user wants to inspect, create, clone, update, approve, archive, unarchive, or delete product offers and checkout variants in Clickmax.
- **clickmax-packs** — Use when the user wants to inspect, create, edit, snapshot, publish (share), or import Clickmax packs — shareable bundles of funnels, pages, flows, and affiliated offers — including funnels4 snapshot/drift/gate handling and import remap resolution.
- **clickmax-payments-dashboard-analysis** — Use when the user wants payment dashboard KPIs, paginated dashboard views, my-sales queries, or filter lookups for seller revenue analysis in Clickmax.
- **clickmax-pipelines** — Use when the user wants to operate CRM pipelines, stages, opportunity cards, attendants, or pipeline analytics in Clickmax.
- **clickmax-portal-enrollments** — Use when the user wants to list, add, bulk add, or remove member enrollments at the portal level in Clickmax Members.
- **clickmax-seller-subscriptions** — Use when the user wants to inspect, chart, cancel, renew, or swap cards on customer subscriptions sold through Clickmax.
- **clickmax-tags** — Use when the user wants to inspect, create, update, delete, clone, or apply CRM tags to leads in Clickmax.
- **clickmax-transaction-operations** — Use when the user wants to inspect transactions or sales, read transaction charts, or refund one or many transactions in Clickmax.
- **clickmax-wallet-receivables** — Use when the user asks if they are ready to sell/receive, about the Clickmax wallet ("Carteira"), bank/receiving account approval, balances, "a receber", receivables, statement/extrato, or withdrawals (saques).
- **clickmax-workspace-plans** — Use when the user wants to inspect, compare, preview cancellation of, or cancel the workspace's own Clickmax SaaS subscription and billing plan.

## Licença

Veja [LICENSE](./LICENSE).
