# Clickmax Skills

Plugin oficial da Clickmax para [Claude Code](https://claude.com/claude-code) e [Codex](https://developers.openai.com/codex): skills que ensinam a IA a operar a plataforma Clickmax — CRM, leads, funis, fluxos de automação, produtos, ofertas, pagamentos, área de membros e mais — através do servidor MCP da Clickmax.

> ⚠️ **Repositório gerado automaticamente.** O conteúdo é publicado a partir do monorepo interno da Bilhon. Edições manuais serão sobrescritas na próxima sincronização. Encontrou um problema? Abra uma issue.

## Instalação

**Claude Code:**

```
/plugin marketplace add bilhon-technologies/clickmax-skills
/plugin install clickmax@clickmax-skills
```

**Codex:**

```bash
codex plugin marketplace add bilhon-technologies/clickmax-skills
codex plugin add clickmax@clickmax-skills
```

## Autenticação

Na primeira conexão ao servidor MCP da Clickmax (`https://mcp.clickmax.io/mcp`), o cliente abre a autorização no navegador (no Codex, rode `codex mcp login clickmax` se não abrir sozinho):

1. Entre na sua conta Clickmax.
2. Confira o que o cliente poderá fazer e selecione **Permitir acesso**.
3. Volte ao cliente de IA para concluir a conexão.

O acesso é guardado e renovado pelo próprio cliente — não é preciso gerar, copiar nem exportar token.

## Atualizações

As skills mudam junto com a plataforma, e este repositório é republicado com a versão bumpada a cada mudança. Nenhum dos dois clientes atualiza sozinho por padrão — puxe a versão nova manualmente de vez em quando.

**Claude Code** busca atualizações de marketplace em segundo plano depois que a sessão inicia, mas **marketplaces de terceiros vêm com o auto-update desligado**. Ligue uma vez:

1. Rode `/plugin`.
2. Vá para a aba **Marketplaces**.
3. Selecione `clickmax-skills`.
4. Escolha **Enable auto-update**.

Quando uma versão nova chega, o Claude Code avisa para rodar `/reload-plugins`; se você não rodar, ela entra no próximo start. Para atualizar na hora, sem esperar o refresh:

```
/plugin marketplace update clickmax-skills
```

**Codex** não tem toggle de auto-update por marketplace — pra conta pessoal, atualize com:

```bash
codex plugin marketplace upgrade clickmax-skills
```

(Workspace Business/Enterprise é exceção: importar este repositório em Admin > Plugins > Import marketplace liga sync diário automático.)

Sem esses passos, a cópia local fica parada na versão instalada e a IA segue operando com regras antigas da plataforma.

## Skills incluídas

- **clickmax-activities** — Use when the user asks about the team's Activities queue (follow-up tasks and booked appointments together), such as what is late, what is scheduled for a day or week, workload per attendant, closing a batch of them, or setting up a multi-step follow-up cadence with automatic WhatsApp steps on an opportunity.
- **clickmax-analytics** — Use when the user asks business/revenue/KPI questions like how much they made, how they are performing, top products, lead counts, funnel performance, or campaign/email open rates over a period.
- **clickmax-campaign-planning** — Use when the user asks for a multi-week or multi-month marketing plan, campaign sequence, launch calendar or nurture strategy that should be built on the account's own history (open rates, base growth, tags, channels, brand voice) and later executed in Clickmax.
- **clickmax-classrooms** — Use when the user wants to list, inspect, create, update, link content to, copy members between, or delete classrooms inside Clickmax member portals.
- **clickmax-contacts-member-access** — Use when the user wants to give a whole group of CRM contacts (a list, a segment, a filtered selection) access to a community and/or classrooms of the members area in Clickmax.
- **clickmax-custom-fields** — Use when the user wants to audit, clean up, or manage the workspace custom fields (campos customizados) of contacts or opportunities — fill rate, usage, health, who filled them, groups, bulk changes, or creating/editing one.
- **clickmax-custom-objects** — Use when the user wants to model, fill, link, filter or clean up CUSTOM OBJECTS (objetos personalizados) in the CRM — entities beyond contacts/opportunities/companies like Imóveis, Veículos, Contratos, Turmas — their records, fields, links (ligações/vínculos), history notes, saved views, bulk changes, or segmenting contacts by linked records.
- **clickmax-drive** — Use when the user wants to find, organize, share, comment on, restore or clean up files and folders in the Clickmax workspace Drive (spaces, trash, quota, Google Drive).
- **clickmax-email-inbox** — Use when the user asks whether a connected company e-mail mailbox is working, wants to change its sender name, signature or Sent-folder sync, or wants to bring the mailbox's recent e-mail history into Conversations.
- **clickmax-external-pages** — Use when the user wants to connect an external page/site to Clickmax tracking or forms using Clickmax page scripts.
- **clickmax-failure-diagnosis** — Use when the seller asks why their sales/transactions are failing or being declined, wants failed payments grouped by reason, or wants the recoverable value and next action per failure reason in Clickmax.
- **clickmax-flows** — Use when the user wants to create, inspect, change, validate, test, debug (executions, failures, retry), run for a list/tag/segment (manual runs), or activate/archive a Clickmax automation flow and its step graph, or asks which automation message a contact is on — including any request to send/create an email (or SMS/WhatsApp) message to leads, even one mentioning a checkout button or a custom visual/dark style (the flow email step's own template options, never a page).
- **clickmax-forms-quizzes** — Use when the user wants to safely create, edit, publish, inspect, or analyze Clickmax forms and quizzes.
- **clickmax-funnels** — Use when the user wants to create, inspect, change, publish, deactivate, delete, or analyze a Clickmax funnel graph.
- **clickmax-getting-started** — Use when the user asks what to do next, how far along the account setup ("Primeiros passos") is, or right after the conversation finished a setup task (product, offer, page, funnel, channel, automation, quiz, lesson, pipeline, opportunity) and the next pending setup task may be offered.
- **clickmax-insights-dashboards** — Use when the user wants to build, change, or read a saved Insights dashboard in Clickmax (opportunities, contacts, activities, meetings, sales, subscriptions, affiliates, visitors, automations, campaigns, messages, live chat) — assembling widgets, starting from a template, asking for the numbers of a dashboard they already have, or opening the records behind a widget number.
- **clickmax-leads** — Use when the user wants to create, find, deduplicate, inspect (including a contact's Raio-X), filter, verify the e-mail of, or compare CRM leads and their commercial context inside Clickmax, or to check whether a contact import/export finished.
- **clickmax-leads-activity-analysis** — Use when the user wants to inspect CRM activity streams, event timelines, or activity-derived metrics for leads and opportunities.
- **clickmax-list-segments** — Use when the user wants to create, inspect, update, reload, or use manual lists and dynamic segments to group leads in Clickmax.
- **clickmax-meetings** — Use when the user wants to book, reschedule, hand over to another attendant, cancel or look up one booked meeting (agendamento) of a contact in Clickmax.
- **clickmax-members** — Use when the user wants to inspect, create, update, enable, disable, enroll, or remove member users and their access/progress in Clickmax Members.
- **clickmax-members-area** — Use when the user wants to build/create a full members area — a portal with classrooms, courses, modules, lessons, students, and login links — end to end in one flow.
- **clickmax-modules** — Use when the user wants to list, inspect, create, update, reorder, or delete modules and lesson ordering inside a Clickmax Members course.
- **clickmax-offers** — Use when the user wants to inspect, create, clone, update, approve, archive, unarchive, or delete product offers and checkout variants in Clickmax, or make a product sellable (deliverable, support contact, invoice name, approval).
- **clickmax-packs** — Use when the user wants to inspect, create, edit, snapshot, publish (share), or import Clickmax packs — shareable bundles of funnels, pages, flows, and affiliated offers — including funnels4 snapshot/drift/gate handling and import remap resolution.
- **clickmax-pages** — Use when the user wants to create, inspect, restyle, rebuild, clone, configure, or publish a native Clickmax-hosted page.
- **clickmax-payments-dashboard-analysis** — Use when the user wants payment dashboard KPIs, paginated dashboard views, my-sales queries, filter lookups, or a recoverable-revenue reading (how much is failed/canceled/refunded/pending/abandoned to win back) for seller revenue analysis in Clickmax.
- **clickmax-pipelines** — Use when the user wants to operate CRM pipelines, stages, opportunity cards, attendants, or pipeline analytics in Clickmax.
- **clickmax-playbooks** — Use when the user wants to create, edit, delete, link to pipelines, preview, or rehearse a CRM playbook (the sales method with stages, questions, tones, keyword triggers and guardrails that guides meetings and calls) in Clickmax.
- **clickmax-portal-enrollments** — Use when the user wants to list, add, bulk add, or remove member enrollments at the portal level in Clickmax Members.
- **clickmax-products** — Use when the user wants to create, inspect, list, archive, unarchive, or delete products in the Clickmax catalog, including one-time-payment and subscription/recurring products.
- **clickmax-projects** — Use when the user wants to create, list, inspect, rename, set-default, or delete workspace projects in Clickmax, or when any build needs a target project resolved first.
- **clickmax-proposals** — Use when the user wants to find, create, edit, version or consolidate Clickmax proposal templates (slide deck or Word document) and the proposals generated for contacts/opportunities.
- **clickmax-sales-insights** — Use when the user wants to know what is selling — top offers/products by revenue and quantity, order-bump attach rate, or period-over-period sales trend (revenue, average ticket, sales count) for seller sales analysis in Clickmax.
- **clickmax-seller-subscriptions** — Use when the user wants to inspect, chart, cancel, or swap cards on customer subscriptions sold through Clickmax.
- **clickmax-support** — Use when the user asks a how-to, troubleshooting, or "why isn't this working" support question about using Clickmax, and the answer should come from the help center.
- **clickmax-tags** — Use when the user wants to inspect, X-ray usage of, create, update, delete, clone, or apply CRM tags to leads in Clickmax.
- **clickmax-team-members** — Use when the user asks who is on the workspace team, how a specific teammate is doing (open opportunities, recent activity, last access), or needs to resolve a person to the right id before acting on their opportunities, tasks, files or shares.
- **clickmax-temperature-score** — Use when the user wants to understand, tune, or audit contact Temperature (behavioural 0-100 engagement) and contact Score (criteria-based fit points) in Clickmax, including why one contact has a given number and the Temperature × Score health of the base.
- **clickmax-transaction-operations** — Use when the user wants to inspect transactions or sales, read transaction charts, or refund a transaction in Clickmax.
- **clickmax-visitors** — Use when the user asks about site visitors, traffic sources, channels or campaigns that brought visits/leads, the visitor conversion funnel, what is happening on the site right now, or what a contact browsed before converting.
- **clickmax-vturb** — Use when the user asks about VSL / Vturb video performance — play rate, engagement, retention, A/B test winners, whether the video is converting, how many people are watching now, or how to connect the Vturb account.
- **clickmax-wallet-receivables** — Use when the user asks if they are ready to sell/receive, about the Clickmax wallet ("Carteira"), bank/receiving account approval, balances, "a receber", receivables, statement/extrato, or withdrawals (saques).
- **clickmax-webchat** — Use when the user wants to create, inspect, edit, enable, or disable a Clickmax webchat bot (the embeddable chat widget channel) and its pre-chat bot flow — greetings, lead capture, clickable questions, AI classification, and handoff to a human attendant.
- **clickmax-workspace-plans** — Use when the user wants to inspect, compare, preview cancellation of, or cancel the workspace's own Clickmax SaaS subscription and billing plan.

## Licença

Veja [LICENSE](./LICENSE).
