# Segment Filter Model

purpose = build dynamic segment filters without losing logic during full-tree replacement

## Shape

`filters` (upsert, preview_count, timeseries, categories_metrics) = FLAT ARRAY, not nested objects — no `childrenAnd`/`childrenOr` container key. Each item:

```
{ id, order, operator, field, negation, parentId?, valueBool?/valueDate?/valueNumber?/valueString?/valueUuid?, customFieldId?, formId?, questionKey? }
```

- `id` = short label you choose (`"a"`, `"grp1"`); links items inside this array only. NOT a UUID — the tool converts labels. Editing an existing segment → keep the ids from `segments_get_filters` for unchanged items.
- `parentId` = `id` of this item's parent GROUP item in the same array; omit for top-level. Must match an item's `id` or the call fails.
- Top-level items are ANDed → intersection (tag A AND tag B, tag AND temperature) = sibling top-level leaves, no group needed.
- Group node = `field: 'children'` + `operator: 'childrenAnd'` | `'childrenOr'`; no value; children = items whose `parentId` points to it. Needed only for OR or nesting.
- Leaf node = real `field` (`tagId`, `temperatureScore`, `leadScore`, `email`, `origin`…) + comparison `operator` (`equals`, `contains`, `startsWith`, `endsWith`, `greaterThan`, `greaterThanOrEqual`, `lessThan`, `lessThanOrEqual`, `in`) + exactly one `value*` slot matching the field type (`tagId` → `valueUuid` = real tag id; `temperatureScore`/`leadScore` → `valueNumber`).
- `negation: true` on ANY item (group or leaf) inverts it; no separate "not" operator.
- Boolean contact flags use `operator: equals` + `valueBool`; `suspectedFraud` = possible card testing (`true` flagged, `false` not flagged). Temperature/Score use `isNotEmpty` (+ `negation: true`) for "not calculated yet".
- Block leaves = one item whose whole condition is a JSON object in `valueString`, `operator: childrenAnd`; every key applies to the SAME underlying row; `negation: true` = "has none like this":
  - `emailEngagement` → `{ engagement: "opened" | "clicked" | "notOpened", withinDays?: 1..365, broadcastId?: <campaign id from broadcasts_list> }`. `opened` = opened or clicked; `notOpened` = received and neither; window counts back from now (event time, or send time for `notOpened`). People only: bot/scanner opens and clicks never count, so it can be lower than a campaign's reported opens.
    - `{ engagement: "openedCount", min: 1..lastCampaigns, lastCampaigns: 1..20 }` = opened (same rule) at least `min` of the last `lastCampaigns` campaigns received, newest first; no `withinDays`/`broadcastId`. Automation e-mails and test sends are not campaigns; fewer campaigns received → counted on those. Negated = below the count, including contacts who never got a campaign.
  - `webinar` → `{ webinarId?, sessionId?, attendance?: "attended" | "noShow", watched?: { metric: "minutes" | "percent", gte?, lte? }, reached?: "pitch" | "offer" | "end", clickedOffer?: boolean }`; `{}` = registered for any webinar.
- Filtering OPPORTUNITY cards by contact fields nests this same array, stringified in `valueString` of a `lead` item (`operator: childrenAnd|childrenOr`) of the opportunity filter — see `opportunities_query`.

### Worked example: tag A AND tag B (measure combined audience)

```
[
  { id: "a", order: 0, field: "tagId", operator: "equals", negation: false, valueUuid: "<tag A id>" },
  { id: "b", order: 1, field: "tagId", operator: "equals", negation: false, valueUuid: "<tag B id>" },
]
```

### Worked example: `(A OR B) AND C`

Order doesn't imply nesting — only `parentId` does:

```
[
  { id: "orGrp", order: 0, field: "children", operator: "childrenOr", negation: false },
  { id: "leafA", order: 1, field: "...",      operator: "equals",     negation: false, parentId: "orGrp", valueString: "..." },
  { id: "leafB", order: 2, field: "...",      operator: "equals",     negation: false, parentId: "orGrp", valueString: "..." },
  { id: "leafC", order: 3, field: "...",      operator: "equals",     negation: false, valueString: "..." },   // top-level → ANDed with the group
]
```

### Worked example: received e-mail in the last 30 days but did not engage

```
[
  { id: "got", order: 0, field: "emailEngagement", operator: "childrenAnd", negation: false, valueString: "{\"engagement\":\"notOpened\",\"withinDays\":30}" },
  { id: "eng", order: 1, field: "emailEngagement", operator: "childrenAnd", negation: true,  valueString: "{\"engagement\":\"opened\",\"withinDays\":30}" },
]
```

Only the negated `eng` item = "no engagement in 30 days", which also matches contacts who received nothing.

## Common Patterns

- `did X` = activity/event condition for X
- `did X AND not Y` = two top-level leaves, Y with `negation: true`
- `(A OR B) AND C` = see worked example above
- broad recency = activity condition + date/window condition

## Safe Usage

- `segments_upsert_filters` replaces the whole tree; never send only the branch being changed
- preview broad filters with `segments_preview_count` before upsert
- preserve unrelated existing branches when editing one part of a segment
- after upsert, reload when the user needs fresh membership before using the segment
