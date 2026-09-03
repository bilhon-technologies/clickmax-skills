# Segment Filter Model

purpose = build dynamic segment filters without losing logic during full-tree replacement

## Shape

`segments_upsert_filters`'s `filters` is a FLAT ARRAY, not nested objects — there is no `childrenAnd`/`childrenOr` container key anywhere in the schema. Each item:

```
{ id, order, operator, field, negation, parentId?, valueBool?/valueDate?/valueNumber?/valueString?/valueUuid?, customFieldId?, formId?, questionKey? }
```

- `id` — client-generated UUID for THIS item (you invent it; it has no meaning outside this array).
- `parentId` — the `id` of this item's parent GROUP item in the same array; omit/null for a top-level item.
- Group node = `field: 'children'` + `operator: 'childrenAnd'` (every child must match) or `'childrenOr'` (any child may match). A group has no direct value; its children are the OTHER array items whose `parentId` points back to it.
- Leaf node = a real `field` (e.g. `email`, `score`, `origin`) + a comparison `operator` (`equals`, `contains`, `startsWith`, `endsWith`, `greaterThan`, `greaterThanOrEqual`, `lessThan`, `lessThanOrEqual`, `in`) + exactly one populated `value*` slot matching the field's type.
- `negation` — boolean on ANY item (group or leaf) to invert it; there is no separate "not" operator.

### Worked example: `(A OR B) AND C`

4 array items, flat, order doesn't imply nesting — only `parentId` does:

```
[
  { id: "root",  field: "children", operator: "childrenAnd", negation: false },                    // top-level group, no parentId
  { id: "orGrp", field: "children", operator: "childrenOr",  negation: false, parentId: "root" },   // A/B group, child of root
  { id: "leafA", field: "...",      operator: "equals",      negation: false, parentId: "orGrp", valueString: "..." },
  { id: "leafB", field: "...",      operator: "equals",      negation: false, parentId: "orGrp", valueString: "..." },
  { id: "leafC", field: "...",      operator: "equals",      negation: false, parentId: "root",  valueString: "..." },
]
```

## Common Patterns

- `did X` = activity/event condition for X
- `did X AND not Y` = a `childrenAnd` group with X as one child leaf and Y as a sibling leaf with `negation: true` (not a separate "not" operator)
- `(A OR B) AND C` = see worked example above
- broad recency = activity condition + date/window condition

## Safe Usage

- `segments_upsert_filters` replaces the whole tree; never send only the branch being changed
- preview broad filters with `segments_preview_count` before upsert
- preserve unrelated existing branches when editing one part of a segment
- after upsert, reload when the user needs fresh membership before using the segment
