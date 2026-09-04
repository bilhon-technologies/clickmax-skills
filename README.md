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

Na primeira conexão ao servidor MCP da Clickmax (`https://mcp.clickmax.io/mcp`), o Claude Code abre a autorização no navegador:

1. Entre na sua conta Clickmax.
2. Confira o que o cliente poderá fazer e selecione **Permitir acesso**.
3. Volte ao terminal para concluir a conexão.

O acesso é guardado e renovado pelo próprio cliente — não é preciso gerar, copiar nem exportar token.

## Atualizações

As skills mudam junto com a plataforma, e este repositório é republicado com a versão bumpada a cada mudança. O Claude Code busca atualizações de marketplace em segundo plano depois que a sessão inicia, mas **marketplaces de terceiros vêm com o auto-update desligado**. Ligue uma vez:

1. Rode `/plugin`.
2. Vá para a aba **Marketplaces**.
3. Selecione `clickmax-skills`.
4. Escolha **Enable auto-update**.

Quando uma versão nova chega, o Claude Code avisa para rodar `/reload-plugins`; se você não rodar, ela entra no próximo start.

Para atualizar na hora, sem esperar o refresh:

```
/plugin marketplace update clickmax-skills
```

Sem o auto-update ligado, a cópia local fica parada na versão instalada e a IA segue operando com regras antigas da plataforma.

## Skills incluídas

- **clickmax-analytics** — Use when the user asks business/revenue/KPI questions like how much they made, how they are performing, top products, lead counts, or funnel performance over a period.
- **clickmax-classrooms** — Use when the user wants to list, inspect, create, update, link content to, copy members between, or delete classrooms inside Clickmax member portals.
- **clickmax-external-pages** — Use when the user wants to connect an external page/site to Clickmax tracking or forms using Clickmax page scripts.
- **clickmax-failure-diagnosis** — Use when the seller asks why their sales/transactions are failing or being declined, wants failed payments grouped by reason, or wants the recoverable value and next action per failure reason in Clickmax.
- **clickmax-flows** — Use when the user wants to create, inspect, change, validate, or activate/archive a Clickmax automation flow and its step graph — including any request to send/create an email (or SMS/WhatsApp) message to leads, even one mentioning a checkout button or a custom visual/dark style (the flow email step's own template options, never a page).
- **clickmax-forms-quizzes** — Use when the user wants to safely create, edit, publish, inspect, or analyze Clickmax forms and quizzes.
- **clickmax-funnels** — Use when the user wants to create, inspect, change, publish, deactivate, delete, or analyze a Clickmax funnel graph.
- **clickmax-insights-dashboards** — Use when the user wants to build, change, or read a saved Insights dashboard of opportunities BI in Clickmax — assembling widgets, starting from a template, or asking for the numbers of a dashboard they already have.
- **clickmax-leads** — Use when the user wants to create, find, inspect, filter, or compare CRM leads and their commercial context inside Clickmax.
- **clickmax-leads-activity-analysis** — Use when the user wants to inspect CRM activity streams, event timelines, or activity-derived metrics for leads and opportunities.
- **clickmax-list-segments** — Use when the user wants to create, inspect, update, reload, or use manual lists and dynamic segments to group leads in Clickmax.
- **clickmax-members** — Use when the user wants to inspect, create, update, enable, disable, enroll, or remove member users and their access/progress in Clickmax Members.
- **clickmax-members-area** — Use when the user wants to build/create a full members area — a portal with classrooms, courses, modules, lessons, students, and login links — end to end in one flow.
- **clickmax-modules** — Use when the user wants to list, inspect, create, update, reorder, or delete modules and lesson ordering inside a Clickmax Members course.
- **clickmax-offers** — Use when the user wants to inspect, create, clone, update, approve, archive, unarchive, or delete product offers and checkout variants in Clickmax, or make a product sellable (deliverable, support contact, invoice name, approval).
- **clickmax-packs** — Use when the user wants to inspect, create, edit, snapshot, publish (share), or import Clickmax packs — shareable bundles of funnels, pages, flows, and affiliated offers — including funnels4 snapshot/drift/gate handling and import remap resolution.
- **clickmax-pages** — Use when the user wants to create, inspect, restyle, rebuild, clone, configure, or publish a native Clickmax-hosted page.
- **clickmax-payments-dashboard-analysis** — Use when the user wants payment dashboard KPIs, paginated dashboard views, my-sales queries, filter lookups, or a recoverable-revenue reading (how much is failed/canceled/refunded/pending/abandoned to win back) for seller revenue analysis in Clickmax.
- **clickmax-pipelines** — Use when the user wants to operate CRM pipelines, stages, opportunity cards, attendants, or pipeline analytics in Clickmax.
- **clickmax-portal-enrollments** — Use when the user wants to list, add, bulk add, or remove member enrollments at the portal level in Clickmax Members.
- **clickmax-products** — Use when the user wants to create, inspect, list, archive, unarchive, or delete products in the Clickmax catalog, including one-time-payment and subscription/recurring products.
- **clickmax-projects** — Use when the user wants to create, list, inspect, rename, set-default, or delete workspace projects in Clickmax, or when any build needs a target project resolved first.
- **clickmax-sales-insights** — Use when the user wants to know what is selling — top offers/products by revenue and quantity, order-bump attach rate, or period-over-period sales trend (revenue, average ticket, sales count) for seller sales analysis in Clickmax.
- **clickmax-seller-subscriptions** — Use when the user wants to inspect, chart, cancel, or swap cards on customer subscriptions sold through Clickmax.
- **clickmax-support** — Use when the user asks a how-to, troubleshooting, or "why isn't this working" support question about using Clickmax, and the answer should come from the help center.
- **clickmax-tags** — Use when the user wants to inspect, create, update, delete, clone, or apply CRM tags to leads in Clickmax.
- **clickmax-transaction-operations** — Use when the user wants to inspect transactions or sales, read transaction charts, or refund a transaction in Clickmax.
- **clickmax-vturb** — Use when the user asks about VSL / Vturb video performance — play rate, engagement, retention, A/B test winners, whether the video is converting, how many people are watching now, or how to connect the Vturb account.
- **clickmax-wallet-receivables** — Use when the user asks if they are ready to sell/receive, about the Clickmax wallet ("Carteira"), bank/receiving account approval, balances, "a receber", receivables, statement/extrato, or withdrawals (saques).
- **clickmax-workspace-plans** — Use when the user wants to inspect, compare, preview cancellation of, or cancel the workspace's own Clickmax SaaS subscription and billing plan.

## Licença

Veja [LICENSE](./LICENSE).
