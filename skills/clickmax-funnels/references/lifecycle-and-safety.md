# Funnel lifecycle and safety

## Lifecycle states

- `draft`
- `published`
- `unpublished_changes`
- `disabled`

## What they mean

- `draft` = editable working state
- `published` = live graph
- `unpublished_changes` = edited after publication; republish required to push changes live
- `disabled` = offline without deletion

## Safety rules

- `funnels_deactivate` is the right tool when the user wants the funnel offline but preserved
- `funnels_delete` is destructive and permanent; confirm first unless deletion was already explicit
- before `funnels_publish`, read fresh structure and validate the graph

## Publish checklist

1. resolve the intended funnel
2. inspect `funnels_structure_get`
3. run `funnels_validate`
4. check for disconnected triggers / orphan nodes / missing pages
5. confirm the user actually wants the funnel live
