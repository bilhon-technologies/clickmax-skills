---
name: clickmax-insights-dashboards
description: Use when the user wants to build, change, or read a saved Insights dashboard of opportunities BI in Clickmax — assembling widgets, starting from a template, or asking for the numbers of a dashboard they already have.
---

## When this applies

Use this skill to create, edit or read a **saved** Insights dashboard: assembling a board of opportunity/activity/lead widgets, starting from a ready-made template, changing a widget's chart or breakdown, reordering the grid, or answering "what do the numbers on my dashboard say".

Not this skill:

- one-off revenue/lead/funnel KPI questions with no saved board -> `clickmax-analytics`
- what is selling: top offers, order-bump attach, sales trend -> `clickmax-sales-insights`
- payment dashboard browsing and loss/recovery cohorts -> `clickmax-payments-dashboard-analysis`
- moving cards, stages, attendants, pipeline settings -> `clickmax-pipelines`
- per-lead timelines and owner activity aggregates -> `clickmax-leads-activity-analysis`

A dashboard is a saved artifact. If the user just wants a number once, answer with the analytics skills above instead of creating a board.

## Key assumptions

- Widget metrics are a **closed curated catalog**, not free text. A raw `widget` whose `entity/reportType` no engine runs is refused **on write** with 400 listing the valid `reportType`s for that entity — it is not saved as a broken card. Never invent a metric name.
- A parameterized lead segment must carry its parameter **in the same call**: `segmentBy: customField` → `customFieldId`, `segmentBy: tag|cardTag` → `tagIds`. Missing it = 400 on write (`widget_add`/`widget_update`/`compose`), not an empty card to fix later.
- Filters live on the **dashboard**, not on the widget. One filter set applies to every widget at once; there is no per-widget filter. Cuts like open/won/lost are expressed by the widget's own `viewBy`, never by filtering status.
- **`filters` only accepts what the screen can show**: `dateFrom`/`dateTo` (period) plus `pipelineIds`/`attendantIds` (pipeline/owner scope). Every other key — `statuses`, `stageIds`, `temperatures`, `priorities`, `valueMin`, `valueMax`, `origins`, `taskTypes`, `taskStatuses` — is **400 on write** (`create`/`compose`/`settings_update`), and the error names the alternative: negotiation cuts go in `conditions`, open/won/lost goes in the widget's `viewBy`. Reason: saved there, the cut would trim every card with no control on the screen to see or clear it.
- `periodPreset` is resolved **at run time**, on both ends (screen and backend, BRT clock): a dynamic preset (`today`…`currentMonth`) re-resolves the window and OVERWRITES `dateFrom`/`dateTo` on every run; `all`/`month`/`custom` use the saved dates. Sending only the preset (no dates) is the right path — the `run` answers the preset's window, not the whole history.
- The shared filter is one set, but each metric applies only the keys of ITS vocabulary — `taskTypes` means nothing to a deal, `temperatures` nothing to an activity, and `forecast` neutralizes `statuses` on purpose (a `statuses:['open']` would zero out the realized half). The run reports what it dropped per widget in `ignoredFilters`: read it instead of assuming the whole filter landed.
- `pipelineId` on a widget is the one exception, and it is **scope, not filter**: metrics broken down by stage are relative to a single pipeline, so two stage widgets on the same board can look at different pipelines. And it is **required on write**: a stage-broken metric (`progress`, `stagetime`, `stageflow`, `stageConversion`, `lossBreakdown`, or `segmentBy: 'stage'`) without `pipelineId` = 400 on `compose`/`widget_add`/`widget_update`. Without it the card would fall back to the default pipeline (the most recent one) and the number gets read as the whole operation's. List the pipelines first (pipeline tools) or send `pipelineId` in the `compose` body — it covers every stage widget of that assembly.
- `config.conditions` is the **advanced filter** — an opportunity-filter tree, separate from the flat `config.filters`, and flattened via `parentId` (never nested objects). Group node = `field: 'children'` + `operator: childrenAnd|childrenOr`, and it **MUST carry `id`** because leaves link to it by `parentId`; a group without `id` collapses the tree into a flat AND. Scope: applied by `deal:*` and `activity:*`, ignored by `lead:*` → the run flags that widget with `ignoredAdvancedFilter: true`.
- Value encoding per operator, and getting it wrong is **not** a silent no-op — the compiler throws and every widget that applies conditions degrades to `status: 'error'` with a zero:

  |Operator|Column|Encoding|
  |-|-|-|
  |`in` / `notIn`|`valueString`|JSON array string: `'["<uuid>"]'` — **never** a bare `'<uuid>'`|
  |`equals` on enum/text|`valueString`|raw value: `'won'`|
  |date comparisons|`valueDate`|ISO 8601|
  |number comparisons|`valueNumber`|number|
  |boolean|`valueBool`|boolean|

- Caps: **25 dashboards** per workspace, **24 widgets** per dashboard — and a section title band spends one of those 24. Both refuse with a permission-style error, not a validation error.
- Filter caps (validation errors): each `filters` list ≤ **100** items (`origins` strings ≤ 255 chars) | `conditions` ≤ **200** nodes, nested ≤ **20** group levels. Filter by the ids that matter instead of pasting the whole workspace.
- Layout is part of the artifact, not decoration: `size` is `{w,h}` on a **12-column** grid (`w` 3..12, `h` 1..8) and a section title band is a widget entry carrying `heading` instead of a chart. Omit `size` and the card falls back to the default for its `chartType` (KPI 3×2, pie/ranking/list 4×4, bar/line 6×4, table 12×5) — which is why a board of three KPIs assembled one by one leaves a quarter of the screen empty. `compose` closes those rows for you.
- Result units are not uniform: monetary measures come back in **cents**, `duration`/`stagetime` in **days**, `conversion`/`completion` as a **0..1 rate**. Convert before showing money or percentages.
- Running a dashboard is **stateless** — the config does not have to be saved. Use it to preview a board before creating it.
- A widget whose selection cannot run **degrades to an empty result** instead of failing the call — and says so: `status: 'error'` on that widget. Trust the field, never the zero: an all-zero widget with `status: 'ok'` is a real "no results for this cut".
- Deleting a dashboard removes only the saved view; opportunity data is untouched.

## Thought process

1. Read the catalog before proposing anything — it is the only source of valid metrics and template ids.
2. Prefer a template when the ask is broad ("um painel pra acompanhar o comercial"). When the user named the metrics, build the board with `compose` in ONE call — widget-by-widget is for editing a board that already exists.
3. Decide whether the user wants an artifact (create/save) or an answer (run only).
4. For edits, address the single widget instead of resending the whole board.
5. Confirm before deleting a dashboard or replacing a whole config.

## Execute guide

- **Always start with** connector operation `mcp__plugin_clickmax_clickmax__analytics_dashboards_catalog`. It returns the valid widget presets (each with a stable id, group, label and the metric selection behind it), the dashboard templates, and the current limits. Everything below depends on ids that come from here.

- **Broad ask -> template.** Use `mcp__plugin_clickmax_clickmax__analytics_dashboards_create_from_template` with the `templateId` from the catalog, optionally `name` and `pipelineId` (the pipeline for the template's stage metrics, STAMPED into each widget; absent = the most recent pipeline, stamped the same way — so the board never switches pipeline on its own later). One call produces a coherent board with its widgets already configured — prefer it over assembling widget by widget, which costs many calls and usually lands on a worse selection.

- **Named metrics -> `mcp__plugin_clickmax_clickmax__analytics_dashboards_compose`.** Declare the board in `sections`: each section is `{ heading?, widgets: [{ presetId | widget, title?, pipelineId?, size?, customFieldId?, tagIds? }] }`. One call resolves every preset, inserts the section title bands and sizes the cards so the grid rows close. Leave the first section without a `heading` — that is the KPI band that opens the board. `pipelineId` at the top level is the default scope for stage-broken metrics; an item may override it. `filters`, `periodPreset`, `showGoals` and `conditions` are set in the same call. `autoSize: false` only when the user wants the raw defaults.

- **Escape hatches for creating.** `mcp__plugin_clickmax_clickmax__analytics_dashboards_create` takes a whole `config` (each widget needs a client-generated uuid `id`) — use it to reproduce an exact config you already have. A blank board (`widgets: []`, `filters: {}`) plus `widget_add` calls still works and validates one metric at a time, but it costs N calls and lands on a worse layout than `compose`.

- **Add a widget** with `mcp__plugin_clickmax_clickmax__analytics_dashboards_widget_add`, passing the dashboard id plus **exactly one** of: `presetId` (preferred — the metric selection and the title are seeded from the catalog), a raw `widget` selection for a variation the catalog does not enumerate, or `heading` to insert a section title band. `title` overrides the seeded label, `position` is the 0-based insertion index (default: last), `pipelineId` scopes a stage-broken metric, `size` sets the card on the 12-column grid, and `customFieldId`/`tagIds` fill the parameter of a lead segment (required whenever `segmentBy` is `customField|tag|cardTag` — the call is refused without it).

- **Change one widget** with `mcp__plugin_clickmax_clickmax__analytics_dashboards_widget_update` (dashboard id + widget id): only the keys you send change. This is the way to switch `chartType` between bar, line, pie, number, ranking and list, to retitle, or to change `segmentBy`/`viewBy`/`interval`. Validation runs on the MERGED widget: switching only `reportType` on a lead widget to a deal-only report is a 400, and switching `segmentBy` to a parameterized one requires sending its parameter in the same call. Remove with `mcp__plugin_clickmax_clickmax__analytics_dashboards_widget_remove`, reorder with `mcp__plugin_clickmax_clickmax__analytics_dashboards_widget_move` — the widget order IS the display order in the grid.

- **Board settings** (shared `filters`, `periodPreset`, `showGoals`, advanced `conditions`, `live` TV mode) change with `mcp__plugin_clickmax_clickmax__analytics_dashboards_settings_update` — partial merge, widgets untouched: only the keys you send change.

- **Do not** reach for `mcp__plugin_clickmax_clickmax__analytics_dashboards_update` to touch a widget or a setting: it replaces the whole config and drops any concurrent change. Use it only to rename a dashboard.

- **Read the numbers** with `mcp__plugin_clickmax_clickmax__analytics_dashboards_run`, passing a `config` — take it from `mcp__plugin_clickmax_clickmax__analytics_dashboards_list` for a saved board, or pass a candidate config to preview one before saving. Each widget comes back with `buckets`, `series` and a `summary.total`.

- **Before creating**, check `meta.count` against `meta.limit` from `mcp__plugin_clickmax_clickmax__analytics_dashboards_list` when the workspace looks crowded, so you can offer to delete one instead of hitting the cap mid-flow.

## Report

- After creating: name the dashboard, say which template (if any) it came from, and list the sections and the widgets each one has — do not dump the raw config or the sizes.
- After a widget edit: state what changed on which widget, not the whole board.
- When reporting numbers: convert cents to currency, rates to percentages, and durations to days before showing them. Lead with the takeaway, then the per-widget values.
- A widget with `status: 'error'`: say the selection failed and offer a catalog alternative — never report its zero as a business result. Zeros with `status: 'ok'` are real, even next to populated siblings.
- When a widget lists `ignoredFilters`, say which cut did not reach it ("the period does not apply to overdue activities — it is a snapshot of now"). Silently presenting the number as filtered is how the user ends up trusting the wrong figure.
- Deleting a dashboard is opt-in only; confirm before doing it.

## Warnings

- Never guess a `presetId` or `templateId`; both come from the catalog and nothing else.
- Do not express open/won/lost by filtering status — that is what `viewBy` is for, and the filter would hit every other widget on the board too.
- A stage-broken widget without `pipelineId` falls back to a default pipeline; set it explicitly when the user named a pipeline.
- Money is in cents everywhere in the run result. Reporting the raw integer as currency is off by 100x.

## Anti-patterns

- Assembling ten widgets by hand when a template answers the same ask in one call — or when `compose` builds the named board in one call.
- Building a board of six cards with no section band and no sizes: it renders as a column of unrelated cards with open rows.
- Resending the whole config to change the period or turn goals on.
- Creating a saved dashboard when the user only asked for a number once.
- Resending the whole config to change one chart type.
- Reporting a widget with `status: 'error'` as a real business result — or calling a legitimate zero an invalid selection.
- Presenting a number as filtered when the widget reported that cut in `ignoredFilters`.
- Inventing a metric name instead of reading the catalog — a raw `widget` with a combination no engine runs now fails the call instead of landing as a broken card.
- Writing a cut into `filters` beyond period/pipeline/owner (it is a 400) instead of `conditions` or the widget's `viewBy`.
- Assembling a stage widget without `pipelineId` (it is a 400) — ask which pipeline, or list them first.
- Hand-computing `dateFrom`/`dateTo` when the ask is "this month"/"last 7 days": send `periodPreset` and let the run resolve it.
- Switching a widget to `segmentBy: customField|tag|cardTag` without sending `customFieldId`/`tagIds` in the same call.
