# tbt-ds pitfalls — side notes

Moved out of SKILL.md (audit 17/07) to keep the main file lean. Same trap → symptom → fix format.

---

## U+2028 / U+2029 in regex literals — use escape form (was SKILL.md §2)

**Trap:** Pasting Unicode line separators (U+2028, U+2029) directly into a JS regex literal: `/U+2028/g`.

**Symptom:** Node ESM parser fails with `SyntaxError: Invalid regular expression: missing /`. The chars look like spaces in editors but the parser treats them as line terminators inside the regex.

**Fix:** Use escape form so source is plain ASCII.
```js
.replace(/ /g, '\\u2028')
.replace(/ /g, '\\u2029')
```
This matters for any "safe JSON for `<script>` tag" helper (e.g. `tbt_page.js`).

---

## URL pattern: internal id, display tranid/entityid (was SKILL.md §13)

**Trap:** Using display IDs (`?id=SO-2026-0001`) in URLs ties navigation to a non-primary key. NetSuite production records use integer internal IDs.

**Fix:** Mock and route as if production:
- Records carry both `id` (integer) and `tranid` / `entityid` (display string)
- URLs use internal id: `/so/form?id=1`
- Page headers / table cells show display id: "SO-2026-0001"
- Foreign-key dropdown `options[].value` = integer internal id
- Record fields like `record.customer = 100` (integer) — not `'C001'`
- List column with `key:'tranid'` + `href: row => formUrl + '?id=' + row.id`

---

## Test running No. + sticky together — manual recipe (was SKILL.md §22)

When both running No. and sticky-left are pinned to column 0, the **easiest verification path** is:
- Hard-refresh page
- Resize Description column wide enough to force horizontal scroll
- Confirm: column "No." stays put with visible numbers; numbers re-sequence on add/delete

If column appears but values are `undefined`, it's almost always SKILL.md §8 (addRow didn't pass rowNo).
