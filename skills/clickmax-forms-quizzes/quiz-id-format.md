---
title: Quiz block/option/condition id format
---

# id format

rule = `^[a-zA-Z_][a-zA-Z0-9_]*$` — identifier-safe, no hyphens
recommended = `<prefix>_<rand>` via the same shape genId() produces

- block id → `b_<rand>` (e.g. `b_a1z9`)
- option id → `o_<rand>`
- step id → `qs_<rand>`
- result id → `r_<rand>`

why = displayRule uses jsep — `b-gest` is parsed as `b - gest` → undefined → condition silently false
forbidden = hyphens, spaces, dots, any non-identifier char

# displayRule.when[].field

reference = the BLOCK id of the block whose answer drives the condition
do not use = the CRM custom-field key (`mapping.target`); runtime aliases it for backward compat but new content must use the block id

example correct = `{ field: 'b_renal', op: 'eq', value: 'sim' }`
example wrong = `{ field: 'b-renal', op: 'eq', value: 'sim' }`
example wrong = `{ field: 'insuf_renal_hepatica', op: 'eq', value: 'sim' }` (custom field key)
