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
- Leaf node = real `field` (`tagId`, `temperatureStatus`, `email`, `score`, `origin`…) + comparison `operator` (`equals`, `contains`, `startsWith`, `endsWith`, `greaterThan`, `greaterThanOrEqual`, `lessThan`, `lessThanOrEqual`, `in`) + exactly one `value*` slot matching the field type (`tagId` → `valueUuid` = real tag id; `temperatureStatus` → `valueString`).
- `negation: true` on ANY item (group or leaf) inverts it; no separate "not" operator.

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
