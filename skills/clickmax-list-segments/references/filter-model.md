# Segment Filter Model

purpose = build dynamic segment filters without losing logic during full-tree replacement

## Shape

- filter tree = root group + nested groups + leaf conditions
- `childrenAnd` = every child must match
- `childrenOr` = any child may match
- leaf condition = field + operator + value slot
- negation = model as an explicit negative operator or a sibling exclusion group when the available operator requires it

## Common Patterns

- `did X` = activity/event condition for X
- `did X AND not Y` = `childrenAnd` with X condition + negative Y condition
- `(A OR B) AND C` = root `childrenAnd` containing one `childrenOr` group for A/B plus leaf C
- broad recency = activity condition + date/window condition

## Safe Usage

- `segments_upsert_filters` replaces the whole tree; never send only the branch being changed
- preview broad filters with `segments_preview_count` before upsert
- preserve unrelated existing branches when editing one part of a segment
- after upsert, reload when the user needs fresh membership before using the segment
