---
name: tbt-ds-pitfalls
description: 'Debugging pitfalls and recipes for Teibto Design System (Lit 3 Web Components) and Suitelet body templates — use when something BREAKS or BEFORE writing a risky pattern, not for general page building. Trigger when editing DS internals under components/**, templates/**, netsuite/**, theme/tbt-theme.css; BEFORE writing CSS-in-JS template literals, sticky/overflow CSS, field-grid + dropdown layouts, or view/edit mode refactors; and when seeing symptoms like "popup clipped", "double scrollbar", "ReferenceError: X is not defined", "subtab disappeared", "running No is undefined", "popup at top of viewport", "view mode still editable", "List item โดนทับ". For building new pages/components, layout, theming, component API lookup, or governance/RFC, use sibling skill teibto-ui-component instead.'
---

# Teibto DS pitfalls & recipes

Each entry: **the trap → the symptom → the fix**. Most cost an hour of debugging in a previous session. Read the matching section before writing similar code.

---

## ⚡ First-class rules — recite before writing ANY code in tbt-ds

When opening any file under `components/**`, `templates/**`, `theme/**`, or `netsuite/**`, recite these verbatim before editing. They are the top-3 highest-frequency mistakes in the codebase's history.

1. **NEVER put a backtick `` ` `` inside a CSS comment in a Lit `css\`\`` template literal.** Use plain text or single quotes. Cost of forgetting: component fails to register → entire subtab/section disappears at runtime, with a `ReferenceError` in console that doesn't mention the file. See §1.

2. **For dropdown / popover popups: use `popover="manual"` + `.showPopover()` + `inset: auto` in CSS. Do NOT use `position: absolute` (clipped), do NOT use `position: fixed` alone (overlapped by next row), do NOT portal to `document.body` (loses CSS scope).** Cost of forgetting: 3+ failed approaches and the user explicitly says "โดนทับ". See §6.

3. **In hybrid-render components (Lit `<thead>` + manual `<tbody>` innerHTML), every prop that branches the row-HTML output must appear in the `updated(changed)` hook with a call to `_renderTbody()`.** That includes `readonly`, `stickyLeft`, `stickyRight`, `_colWidths`, and any future flag. Cost of forgetting: view mode keeps showing edit-mode dropdowns, sticky body cells don't follow header changes. See §7.

If any of these conditions appear in the code you're about to write or modify, **stop and read the matching section first.**

Other high-frequency: `tbt-section { overflow: hidden }` clips child popups (§5), components without `readonly` prop are silent no-ops on `setAttribute('readonly')` (§11), `border-collapse: collapse` breaks sticky on `<th>` (§3), and — a *separate* freeze-pane breaker — a **`colspan` cell can't be sticky** so subtotal/total rows bleed over frozen columns (§3.1).

---

## 1. Lit `css\`\`` template literal — never use backticks inside CSS comments

**This pitfall has bitten the same session ≥3 times.** Every time you write a CSS comment that mentions a CSS property name, attribute selector, or value, the urge is to wrap it in backticks for "code style" — DON'T. Use plain text or single quotes.

**Trap:** Backticks inside a CSS comment break the tagged template literal.

```js
// ❌ BREAKS THE TEMPLATE — parser closes the template at the first backtick
static styles = css`
  /* Class `.sticky-cell` is applied via render() */
  .sticky-cell { ... }
`;
```

**Symptom:** `Uncaught ReferenceError: cell is not defined` at static initializer. Component fails to register → all instances in the page render as empty custom elements → **whole subtab / section content disappears**. `npm run build` may succeed but runtime explodes.

**Fix:** Strip backticks from CSS comments — use plain text or single quotes.
```js
static styles = css`
  /* Class .sticky-cell is applied via render() */
  .sticky-cell { ... }
`;
```

Also applies to Rollup parse errors: if you see `(line, column): Expected a semicolon` pointing into a `css\`\`` block, look for a backtick or `${expression}` that shouldn't be there.

---

## 2. U+2028 / U+2029 in regex literals — use escape form

Node ESM parser fails with `SyntaxError: Invalid regular expression: missing /` when U+2028/U+2029 are pasted raw into a JS regex literal (they look like spaces in editors). Matters for any "safe JSON for `<script>` tag" helper (e.g. `tbt_page.js`). Full trap/symptom/fix + escape-form snippet → **`references/notes.md`**.

---

## 3. `position: sticky` on `<th>` requires `border-collapse: separate`

**Trap:** Default `table { border-collapse: collapse }` breaks `position: sticky` on `<th>` in Chrome / Firefox / Safari (a long-standing browser limitation).

**Symptom:** Sticky header row works on `<thead>` element but **per-column sticky `<th>` doesn't stay in place when scrolling horizontally**. Header pin shows but td cells slide away (or vice-versa).

**Fix:** Switch to separate borders.
```css
table {
  border-collapse: separate;
  border-spacing: 0;
}
td { background: var(--tbt-bg-card); }   /* opaque so sticky cells cover scrolled cells */
```

Also: `<th>` and `<td>` that are sticky cells must each have their own opaque background (`background: var(--tbt-bg-card)` or `var(--tbt-bg-hover)` for header), otherwise scrolled content shows through.

---

## 3.1 `colspan` cells can't be sticky — a *second*, independent freeze-pane breaker

**Trap:** §3 is one way sticky columns break (`border-collapse: collapse`). This is a **different** one that survives even after §3 is fixed: **`position: sticky` is unreliable on a `<td>`/`<th>` that has `colspan`.** Chrome leaves the colspanned cell **unpinned** — it scrolls off with the body instead of freezing. (Independent of border-collapse: a table on `collapse` whose cells are per-column and non-colspan can still freeze fine, while a `colspan` cell on the same table won't.)

Real case: a freeze-pane table whose leaf rows used one sticky cell per row-dim column (pinned correctly) but whose **subtotal / grand-total rows used a single `colspan` cell for the label** (never pinned). On horizontal scroll the value cells slid out over the frozen columns and overlapped the labels ("ข้อมูลซ้อนทับ ดูแล้วงง").

**Symptom:** Data rows freeze correctly; **subtotal / group-header / total rows bleed** — the label doesn't pin and scrolled cells show through the span where the colspan cell should have covered.

**Diagnose by MEASURING, not by re-reading CSS.** A pinned sticky cell sits at the scroll container's left edge; an unpinned one scrolls with content:
```js
cell.getBoundingClientRect().left   // pinned → ≈ container left; unpinned → far from it
```
A sticky cell whose computed `left` is `0px` but which *measures* far from the container's left IS the bug. Re-reading `z-index`/`background` won't reveal it — those values already look correct. (And keep the measurement eval + a screenshot on the SAME frame — a stray re-render makes "computed says covered but pixels bleed" look like a paint mystery when it's really two different renders.)

**Fix:** never put a sticky label in a `colspan` cell. Emit **one sticky cell per frozen column** (identical to the working leaf rows); put the label in the first cell and let it **overflow** rightward across the now-empty filler cells:
- label cell: `overflow: visible`, normal sticky z-index
- filler cells: sticky + opaque, **one z-index layer below** the label, so the overflowing label paints over them while they still cover scrolled data

Mind the sibling z-order — see §4 (sticky-cell z-index specificity).

---

## 3.2 One view needs a different number/format? Add an ADDITIVE variant — never mutate the shared formatter

**Trap:** A single table/column needs a new display format (e.g. Excel `#,##0.00`: thousands separators + fixed 2 decimals + no currency symbol) and the shared `formatValue`/`currency` helper doesn't produce it. The urge is to change the shared `currency`/`number` branch. **That silently reformats every other table/KPI/report that shares the formatter.**

**Fix:** add a NEW named format (`num2`, `money2`, …) alongside the existing branches and point only the fields that need it at it. Zero blast radius. Same rule for `formatRaw`/CSV so the export matches. Don't reach for `toFixed(2)` alone (no separators) or `toLocaleString()` alone (locale-default decimals) — Excel `#,##0.00` needs both: `Number(v).toLocaleString(undefined,{minimumFractionDigits:2,maximumFractionDigits:2})`.

---

## 4. CSS specificity wars in sticky-cell z-index

**Trap:** Mixing multiple sticky rules in different specificities.
```css
.lines-wrap.scrollable thead th    { z-index: 2; }   /* specificity 0,2,1 */
th.sticky                          { z-index: 1; }   /* specificity 0,1,1 */
thead th.sticky                    { z-index: 3; }   /* specificity 0,1,2 */
```
Higher specificity wins: row 1 (`0,2,1`) beats row 3 (`0,1,2`) → header sticky-column never reaches z-index: 3 → other sticky body cells overlap it.

**Symptom:** Pin header → header cell looks pinned, but **scrolled body cells stack on top** of the pinned header cell.

**Fix:** Bring the sticky-header rule's selector specificity above the vertical-sticky rule.
```css
.lines-wrap.scrollable thead th.sticky-cell { z-index: 3; }   /* 0,3,1 — now wins */
```

General rule: when adding a new sticky/positioning concern, **count specificity** of every rule touching the same property on the same selector path before assuming yours wins.

---

## 5. `tbt-section { overflow: hidden }` clips child popups

**Trap:** Sections clip everything inside them, including dropdown popovers, tooltips, date-pickers.

**Symptom:** Open a `tbt-dropdown` inside a section → list shows but **gets cut off at the section edge**. Section below visually covers the popup.

**Fix:** Remove `overflow: hidden` from `:host`. Round only the header div (top corners) so hover background still respects radius.
```css
:host {
  border-radius: var(--tbt-radius-lg);
  overflow: visible;     /* was: hidden */
}
header {
  border-radius: var(--tbt-radius-lg) var(--tbt-radius-lg) 0 0;
}
```

This trade-off is acceptable because content overflow is rare in practice; popup clipping is constant.

---

## 6. Searchable dropdowns/popovers — use HTML5 Popover API (top layer), NOT portal-to-body

**Trap:** Three failed approaches in sequence:
1. `position: absolute` — clipped by any ancestor overflow.
2. `position: fixed` — escapes most overflow but inside deeply nested shadow DOM + `<table>` + scroll wrappers, the popup **still gets visually stacked under sibling table rows / next sections**.
3. **Portal to `document.body`** — escapes stacking but **loses all shadow-DOM CSS scoping**. The popup renders as an unstyled bare element (no background, no border, options invisible). The user sees "โดนทับหนักกว่าเดิม และไม่เห็นรายการเลย".

**Symptom A (`position: absolute`):** Popup completely cut off at scroll-container edge.
**Symptom B (`position: fixed` only):** Popup renders styled but next row/section visually overlaps it.
**Symptom C (`document.body` portal):** Popup escapes overlap but is **completely unstyled** — invisible options, no border, no background. Looks worse than B.

**Fix (correct one):** Use the **HTML5 Popover API**. Setting `popover="manual"` + calling `.showPopover()` promotes the element to the browser's **top layer**, which:
- Escapes ALL ancestor stacking contexts (above everything, even other fixed elements)
- Escapes ALL overflow clipping (visible anywhere on the viewport)
- **Stays inside the shadow DOM** — all `static styles = css\`...\`` rules continue to apply
- No portaling, no DOM moving, no orphan cleanup

```js
// In updated() when _open flips true:
const popup = this.shadowRoot.querySelector('.dropdown');
if (!popup.hasAttribute('popover')) popup.setAttribute('popover', 'manual');
try { popup.showPopover(); } catch (_) {}
this._positionPopup();    // set top/left/width from trigger.getBoundingClientRect()
// add scroll/resize reposition listeners

// In updated() when _open flips false:
try { popup.hidePopover(); } catch (_) {}
// remove listeners
```

CSS:
```css
.dropdown {
  display: none;
  position: fixed;
  inset: auto;          /* CRITICAL — see warning below */
  z-index: 9999;
  background: var(--tbt-bg-card);     /* keep all your usual styling */
  border: 1px solid var(--tbt-border);
  border-radius: var(--tbt-radius-md);
  box-shadow: var(--tbt-shadow-md);
  max-height: 240px; overflow-y: auto;
  /* reset browser popover defaults: */
  margin: 0; padding: 0; color: var(--tbt-text-primary);
}
.dropdown:popover-open { display: block; }   /* show only when popover is invoked */
```

**Critical sub-trap — `[popover]` browser default is `inset: 0; margin: auto`**.
This centers the popup in the viewport. Your inline `top: Ypx; left: Xpx` overrides only those two of the four insets — `right: 0` and `bottom: 0` remain, so the popup stretches from your-left/your-top to the viewport's right/bottom corner = giant misplaced rectangle, usually appearing at the top of the page.

**Fix:** add `inset: auto;` to the popup's CSS to clear all four insets at once. Inline `top` / `left` then position it correctly.

**Symptom of forgetting this:** Popup opens at top of viewport instead of below trigger, often as a wide horizontal stripe.

`_positionPopup()` JS:
```js
const rect = this.shadowRoot.querySelector('.trigger').getBoundingClientRect();
const spaceBelow  = window.innerHeight - rect.bottom;
const popupHeight = Math.min(popup.scrollHeight || 240, 240);
const flipUp = spaceBelow < popupHeight + 8 && rect.top > popupHeight;
popup.style.left   = `${rect.left}px`;
popup.style.width  = `${rect.width}px`;
if (flipUp) { popup.style.top = ''; popup.style.bottom = `${innerHeight - rect.top + 4}px`; }
else        { popup.style.bottom = ''; popup.style.top = `${rect.bottom + 4}px`; }
```

**Why Popover API wins:**
- Top layer is a single per-document layer above the normal painting tree. Nothing can stack on top of it.
- Element does not move — DOM hierarchy and CSS scope are preserved.
- `:popover-open` pseudo-class makes show/hide a pure-CSS toggle.
- `popover="manual"` means YOU control open/close (vs `auto` which dismisses on outside-click — your component already has its own outside-click handler).

**Browser support:** Chrome 114+ (May 2023), Firefox 125+ (April 2024), Safari 17+ (Sept 2023). Wrap `showPopover()` / `hidePopover()` in try/catch so older browsers don't throw; they'll just fall back to `position: fixed` behavior (which still works for most cases).

**Anti-pattern: do NOT move popups to `document.body`** even though it sounds like it should work. You will spend an extra hour redebugging missing styles.
```js
_positionPopup() {
  const rect = this.shadowRoot.querySelector('.trigger').getBoundingClientRect();
  const popup = this.shadowRoot.querySelector('.dropdown');
  popup.style.left  = `${rect.left}px`;
  popup.style.width = `${rect.width}px`;
  popup.style.top   = `${rect.bottom + 4}px`;
  // Flip up if not enough room below:
  // const spaceBelow = window.innerHeight - rect.bottom;
  // if (spaceBelow < 240 && rect.top > 240) { use bottom: instead of top: }
}

updated(changed) {
  if (changed.has('_open')) {
    if (this._open) {
      this._positionPopup();
      this._repositionBound = () => this._positionPopup();
      window.addEventListener('scroll', this._repositionBound, true);  // capture
      window.addEventListener('resize', this._repositionBound);
    } else if (this._repositionBound) {
      window.removeEventListener('scroll', this._repositionBound, true);
      window.removeEventListener('resize', this._repositionBound);
      this._repositionBound = null;
    }
  }
}
disconnectedCallback() {
  super.disconnectedCallback();
  if (this._repositionBound) {
    window.removeEventListener('scroll', this._repositionBound, true);
    window.removeEventListener('resize', this._repositionBound);
  }
}
```

---

## 6.1 Click-outside must use `composedPath()`, never `contains(e.target)`

**Trap:** dropdown/multiselect with a document-level click-outside closer:
`if (!host.contains(e.target)) close()`. Picking an option **re-renders the
option list** (checkbox toggles, selected-state repaint) *before* the click
finishes bubbling to `document` — by the time the handler runs, the clicked
node is **detached**, `contains()` returns false, and the panel closes itself
on every pick. Multi-selects break hardest: they're supposed to stay open.

**Symptom:** single-select works; multi-select closes after the first checkbox
click (or any pick that triggers a repaint). Tests that click twice in a row
catch it; a human clicking slowly may not.

**Fix:** the event path is snapshotted at dispatch time and survives
detachment (this is what tbt-ds' own components do):

```js
const inside = e.composedPath ? e.composedPath().includes(host) : host.contains(e.target);
if (open && !inside) close();
```

Bonus for imperative (non-Lit) widgets rebuilt via `innerHTML`: make the
document listener self-pruning — `if (!host.isConnected) { document.removeEventListener('click', onDoc); return; }`
— so wholesale re-renders of the container don't leak dead listeners.
(Proven in Teibto Report Builder's filter combo, 2026-07-12 — jsdom test
caught the multi-select self-closing before it shipped.)

---

## 7. Hybrid render: Lit `<thead>` + manual `<tbody>` innerHTML

**Trap:** `tbt-line-items` renders `<thead>` via Lit but `<tbody>` via `_renderTbody()` (manual `innerHTML` for inline-edit performance). Lit auto re-renders only its template; the tbody stays stale when properties change.

**Symptom:** Change `stickyLeft` → header gets new sticky class but body cells **don't update**. User pins a column → top sticks but rows below ignore it. **Or:** set `readonly = true` after `rows` already assigned → header shows edit-mode column headers but body still has live dropdowns/inputs because tbody was rendered before the prop flipped and never re-rendered.

**Fix:** Add a `updated()` lifecycle hook that mirrors ALL prop changes that affect tbody output — sticky props, column widths, AND `readonly` mode switch.
```js
updated(changed) {
  if (changed.has('stickyLeft')  ||
      changed.has('stickyRight') ||
      changed.has('_colWidths')  ||
      changed.has('readonly')) {     // ← critical — without this, view mode keeps showing edit inputs
    this._renderTbody();
  }
}
```

Same principle applies to running row numbers, computed columns, etc. When in doubt: **every prop that branches the row-HTML output must be in this list**.

---

## 8. Running No. column — pass rowNo to ALL render paths

**Trap:** `_rowHTML(r, rowNo)` accepts a row number, but `addRow()` calls `_rowHTML(r)` without it → cell shows `"undefined"`. Delete also breaks: `btn.closest('tr').remove()` removes one row but leaves others with stale numbers (1, 3, 4).

**Fix:**
```js
addRow() {
  // ...
  tbody.insertAdjacentHTML('beforeend', this._rowHTML(r, this._rows.length));
}

_onClick(e) {
  if (!e.target.closest('.del[data-id]')) return;
  this._rows = this._rows.filter(/* ... */);
  this._renderTbody();   // ALWAYS full re-render — re-sequences No.
}
```

Rule: when a column is computed from index/position, every mutation path (add, remove, reorder, filter) **must full-re-render** that column's source.

---

## 9. Native inputs in flex/grid won't shrink — need `min-width: 0`

**Trap:** Component host has `display: inline-block` + native input has `min-width: 220px` → inside `tbt-field-grid` cell, host won't fill the 1fr column **and** input won't shrink. Result: input overflows visible region.

**Symptom:** Search field bursts out past section edge on standard viewport.

**Fix:** Cascade `min-width: 0` from host down through wrapper to input.
```css
:host    { display: block; width: 100%; max-width: 100%; min-width: 0; }
.wrap    { display: flex;  width: 100%; min-width: 0; }
input    { width: 100%; min-width: 0; box-sizing: border-box; }
```

Default `min-width: auto` (= content size) is the silent killer for flex/grid items.

---

## 10. `box-sizing` inconsistency across input variants

**Trap:** Native `<select>` declares `box-sizing: border-box`; custom `.trigger` div doesn't → at the same `min-height: 38px`, the trigger renders ~16px taller (padding + border are added on top of content height).

**Symptom:** In a row of side-by-side fields, the searchable dropdown is visibly taller than the native dropdown next to it.

**Fix:** Always `box-sizing: border-box` on form-control elements/wrappers.

---

## 11. Components that look "broken" in view mode lack a `readonly` prop

**Trap:** Setting `readonly` attribute on a component that has no `readonly` property is a no-op. User clicks the field and it still opens/edits.

**Affected components historically:** `tbt-dropdown`, `tbt-multiselect`, `tbt-datepicker`, `tbt-date-range`, `tbt-toggle`, `tbt-checkbox`, `tbt-search`.

**Fix:** Each input-like component needs:
```js
static properties = {
  ...,
  readonly: { type: Boolean, reflect: true },
};
```
+ CSS to neutralize interaction:
```css
:host([readonly]) .trigger   { background: var(--tbt-bg-hover); cursor: default; pointer-events: none; }
```
+ guard handlers:
```js
_toggleOpen() {
  if (!this.disabled && !this.readonly) this._open = !this._open;
}
```
+ proper `tabindex`:
```html
tabindex=${this.disabled || this.readonly ? '-1' : '0'}
```

**Audit command** to find missing readonly support:
```bash
for f in tbt-dropdown tbt-datepicker tbt-toggle tbt-checkbox tbt-search; do
  grep -l "readonly:[[:space:]]*\{[[:space:]]*type:[[:space:]]*Boolean" "components/${f}.js"
done
```

---

## 12. View mode shouldn't look like edit mode

**Trap:** Setting `readonly` on every input makes them gray and uneditable — but they **still look like input boxes** with borders. Not how NetSuite native or any ERP renders read-only data.

**Fix:** In view mode, **don't render `<tbt-input>` etc.** Render `<tbt-field label="X" value="Y">` (label + value pair) instead.

For schema-driven forms (`tbt-doc-form`-style), iterate `schema.sections[i].fields` in view mode and emit `<tbt-field>` per field; emit `<tbt-input>`/`<tbt-dropdown>` per field in edit/new mode.

For foreign-key fields, render a link inside `tbt-field`:
```js
if (FK_LINKS[field.name] && val) {
  const a = document.createElement('a');
  a.href = FK_LINKS[field.name](val);   // e.g. /customer/form?id=100
  a.style.color = 'var(--tbt-text-link)';
  a.textContent = displayValue(field, val);   // resolve label from optionLists
  field.appendChild(a);
}
```

---

## 13. URL pattern: internal id, display tranid/entityid

URLs route by integer internal id (`/so/form?id=1`), display strings (`SO-2026-0001` / entityid) are for headers and table cells only — same as NetSuite production. Full convention checklist → **`references/notes.md`**.

---

## 14. `tbt-button` needs `href` + modifier-key handling

**Trap:** Naïvely, `<tbt-button>` has no `href` prop → setting one is silently ignored. Hub menu / "Back" buttons don't navigate.

**Fix:**
```js
static properties = { ..., href: { type: String }, target: { type: String } };

_handleClick(e) {
  if (this.disabled || this.loading) { e.preventDefault(); return; }
  if (this.href) {
    if (e.metaKey || e.ctrlKey || e.shiftKey || this.target === '_blank') {
      window.open(this.href, this.target || '_blank', 'noopener,noreferrer');
    } else {
      window.location.href = this.href;
    }
  }
}
```
Honors Ctrl/Cmd/Shift-click for open-in-new-tab — users will reach for that.

---

## 15. Action icons in tables — `cursor: pointer` doesn't inherit through shadow DOM reliably

**Trap:** Wrapping shadow-DOM custom elements in `.tbt-row-actions { cursor: pointer }` doesn't always pierce into the icon's host. User sees default cursor + no tooltip.

**Fix:** Set inline `style="cursor:pointer"` + `title="Action"` + `aria-label` + `role="button"` + `tabindex="0"` directly on each `<tbt-icon>` HTML string emitted by the helper.
```js
const iconBtn = (action, name, color, label) =>
  `<tbt-icon name="${name}" color="${color}" size="md"
     data-action="${action}"
     title="${label}" aria-label="${label}"
     role="button" tabindex="0"
     style="cursor:pointer"></tbt-icon>`;
```

---

## 16. Section wrapper proliferation — make it optional

**Trap:** `<tbt-lines-block>` always wraps its body in `<tbt-section>` internally. When placed inside `tbt-tabs-panel` (already framed) or under an outer `<tbt-section>`, you get **3 nested frames** stacking borders + padding — visually heavy.

**Fix:** Render the wrapper only when `title` is provided.
```js
render() {
  const hasTitle = this.title && String(this.title).trim();
  const inner = html`<tbt-line-items ...></tbt-line-items><div class="footer">...</div>`;
  return hasTitle
    ? html`<tbt-section .title=${this.title}>${inner}</tbt-section>`
    : inner;
}
```
Backward-compatible: existing consumers with `title="Line items"` keep their frame.

---

## 17. `not-collapsible` for sections that shouldn't toggle

**Trap:** Subtabs section under the Main section in a form: the toggle chevron is wasted UX (no reason to collapse a tab container).

**Fix:** Add boolean prop:
```js
notCollapsible: { type: Boolean, attribute: 'not-collapsible', reflect: true }
```
+ CSS:
```css
:host([not-collapsible]) .chevron     { display: none; }
:host([not-collapsible]) .toggle-btn  { cursor: default; }
:host([not-collapsible]) header:hover { background: transparent; }
```
+ guard in `toggle()`:
```js
toggle() {
  if (this.notCollapsible) return;
  this.collapsed = !this.collapsed;
  // ...
}
```

---

## 18. Sticky column with `pref-key` persistence pattern

When a component supports column resize + sticky, build all three as a kit:
1. **Default** via attribute: `sticky-left="0,1" sticky-right="8"`
2. **Per-user override** via right-click context menu → menu items: Pin to left / Pin to right / Unpin
3. **Persistence** via `pref-key` attribute → `localStorage['tbt-<comp>-sticky-<prefKey>']`
4. **Cleanup** in `disconnectedCallback` for any window listeners attached during open

Same skeleton works for `tbt-table` (list pages) and `tbt-line-items` (form line items).

```js
_loadStickyPref() {
  if (!this.prefKey) return;
  try {
    const raw = localStorage.getItem('tbt-line-items-sticky-' + this.prefKey);
    if (!raw) return;
    const obj = JSON.parse(raw);
    if (typeof obj.left  === 'string') this.stickyLeft  = obj.left;
    if (typeof obj.right === 'string') this.stickyRight = obj.right;
  } catch (_) {}
}
_saveStickyPref() {
  if (!this.prefKey) return;
  try {
    localStorage.setItem('tbt-line-items-sticky-' + this.prefKey, JSON.stringify({
      left: this.stickyLeft || '', right: this.stickyRight || '',
    }));
  } catch (_) {}
}
```

---

## 19. Subtab terminology

In Teibto convention, in-form tabs are called **"Subtabs"** (not "Tabs"). The in-form component is `<tbt-tabs>` with `<tbt-tabs-panel>` children (ARIA-correct) but naming in code comments, prompts, and chat should use **Subtab**. อย่าสับสน: `<tbt-subtab>` + `<tbt-tab>` เป็นอีก component หนึ่ง (tab navigation) — ทั้งสองตัว register อยู่จริงใน tbt-ds; `tbt-tabs`'s panel ถูก rename จาก `tbt-tab` → `tbt-tabs-panel` เพื่อเลี่ยงชนกับของ `tbt-subtab` (verified against source v1.45.1).

---

## 20. Sticky CSS structure — keep blocks ordered & non-overlapping

After hitting issues #3, #4, #6 repeatedly, the working layout is:

```css
/* === Layout === */
.scroll { overflow-x: auto; max-width: 100%; min-width: 0; }
.scroll.scrollable { overflow-y: auto; max-height: ...; }

/* === Table base === */
table { border-collapse: separate; border-spacing: 0; }
table.fixed { table-layout: fixed; }
th { position: relative; background: ...; }   /* anchor for resize handle */
td { background: var(--tbt-bg-card); }         /* opaque for sticky overlap */

/* === Vertical sticky header (only when scrollable) === */
.scroll.scrollable thead th { position: sticky; top: 0; z-index: 2; }

/* === Horizontal sticky columns === */
th.sticky-cell, td.sticky-cell { position: sticky; z-index: 1; }
.sticky-cell.edge-left  { box-shadow:  1px 0 0 var(--tbt-border); }
.sticky-cell.edge-right { box-shadow: -1px 0 0 var(--tbt-border); }
td.sticky-cell { background: var(--tbt-bg-card); }
tbody tr:hover td.sticky-cell { background: var(--tbt-bg-hover); }
.scroll.scrollable thead th.sticky-cell { z-index: 3; }   /* 0,3,1 beats 0,2,1 */
```

Add comments above each block stating **what specificity it has** if it's competing with another rule.

---

## 21. Bundle errors at build pass but explode at runtime

Rollup minification can succeed even when a `css\`\`` template literal would produce invalid JS at runtime. The bundle ships; the browser console fails on `static initializer`.

**Checklist before declaring done:**
1. `npm run build` succeeds
2. `npm run lint` passes
3. **Open ONE page in a browser** and check console for errors before reporting "fixed" — `curl` tests only verify SSR HTML, not client-side execution

---

## 22. Test running No. + sticky together

Manual test recipe (hard-refresh → force horizontal scroll → confirm "No." pins + re-sequences) → **`references/notes.md`**. If the column shows `undefined` values, it's almost always §8.

---

## Quick-reference checklist when something looks broken

| Symptom | Likely cause | Section |
|---|---|---|
| Subtab content disappears entirely | Lit static init error from CSS template | §1 |
| Sticky `<th>` doesn't pin while scrolling | `border-collapse: collapse` | §3 |
| Pinned header z-index wrong | Specificity ordering | §4 |
| Dropdown popup clipped | Section overflow or `position: absolute` | §5, §6 |
| Dropdown popup visible but next row/section overlaps it | Use HTML5 Popover API (`popover="manual"` + `.showPopover()`) | §6 |
| Dropdown options invisible / unstyled after fix | You moved popup to `document.body` (anti-pattern) — use Popover API instead | §6 |
| Popup opens at top of viewport / as wide stripe | Missing `inset: auto` — browser `[popover]` default `inset: 0` centers it | §6 |
| Body cells don't update after sticky change | Manual tbody not re-rendered | §7 |
| "No." cell shows `undefined` | addRow didn't pass rowNo | §8 |
| Search input overflows section | Missing `min-width: 0` cascade | §9 |
| Heights mismatch in field-grid row | `box-sizing` inconsistency | §10 |
| View mode field still editable | Component lacks `readonly` prop | §11 |
| View mode `tbt-lines-block` still shows dropdowns/inputs | Compound component missing `readonly` pass-through OR `_renderTbody()` not triggered on `readonly` change | §7, §11 |
| View mode looks like edit boxes | Should use `tbt-field`, not readonly input | §12 |
| Double scrollbar on page | Body margin not reset (`body { margin: 0; height: 100% }`) — see `tbt_page.js` reset block | (not numbered) |
| `tbt-button href` not navigating | Component missing href handler | §14 |
| Cursor not pointer on action icon | Set inline `style="cursor:pointer"` + `title` | §15 |
| Icon shows as empty colored box, no error | `ti ti-${name}` rendered raw — resolve via `ICON_ALIASES` | §23 |
| Second Save creates a duplicate record | Page ignored the RESTlet response — adopt `{id, tranid, status}` | §24 |
| Section title breaks mid-word on mobile | Wide actions slot crushed the title (fixed: header wraps) | §25 |
| axe `button-name` critical on a section | `<tbt-section>` with no title = chevron-only button | §25 |
| Filter dropdown truncates its placeholder | Flex-item collapse — add `auto-width` | §26 |
| English strings on a Thai page | Component defaults / backend messages — override attrs + Thai at source | §27 |
| Two date formats on one screen | Native date input uses browser locale — pending RFC #29, don't hand-fix | §27 |
| Hamburger overlaps drawer brand | Hide ☰ via `tbt-app-shell[drawer-open]` | §27 |

---

## 23. Icon names must resolve through `ICON_ALIASES` — never `ti ti-${name}` raw

**Trap:** a component renders Tabler webfont classes directly from its attribute:
`<i class="ti ti-${this.icon}">`. ERP alias names (`money`, `payment`) are NOT real
Tabler classes → `::before { content: none }` → an **empty colored box** with no glyph,
no error anywhere. Deceptive detail: some aliases (`invoice`, `receipt`) happen to ALSO
exist as raw Tabler names, so the bug shows only on *some* icons (2 of 4 stat cards).

**Fix (shipped, #25):** `ICON_ALIASES` is exported from `components/tbt-icon.js`.
Any component that renders `ti ti-...` itself must resolve first:
```js
import { ICON_ALIASES } from './tbt-icon.js';
html`<i class="ti ti-${ICON_ALIASES[this.icon] ?? this.icon}"></i>`
```
**Detect:** `grep -rn 'ti ti-\${' components/` — every hit must have the resolve.
In-browser check: `getComputedStyle(iEl, '::before').content === 'none'` = missing glyph.

---

## 24. Save flow must adopt the server's returned identity (duplicate-on-double-save)

**Trap:** form page posts to the RESTlet and ignores the response:
`await rt.post(url, collect('save')); showAlert('ok')`. The record IS created, but the
page still has `v.id = null` → header stays "(ใหม่)", URL has no `id=`, and a second
click of Save **creates a duplicate record**. Proven live on a real sandbox.

**Fix pattern (bill-receipt-form.html `applySaved()`):** backend `save()` returns
`{ id, tranid }` (generate tranid inside save and return it — the client's copy is
empty on first save); RESTlet returns `{ ok, id, tranid, status }`; the page then:
1. `v.id/tranid/status = res...` 2. re-render header + `document.title`
3. `history.replaceState(null,'',href + '&id=' + id)` (guard: only if no `id=` yet)
4. re-evaluate action-button visibility per the new status.

Corollary: "ส่งตรวจรับ" (submit) from a never-saved form must **save-then-submit** in
one handler — the state machine only allows Draft → Submitted, and letting the backend
answer `Cannot submit ... "new"` verbatim is UX failure.

---

## 25. `tbt-section` header with a wide actions slot — and the empty-title section

Two traps in the same header (axe/375px findings):

- **Mid-word title break:** header was `display:flex` with `.toggle-btn { flex:1 }` —
  a wide actions slot (search + dropdown) crushed the title until it broke mid-word
  ("รายการ⏎บิล" around the search box). Fixed in the component with `flex-wrap: wrap`
  + `flex: 1 1 12rem` on the toggle; if you build a custom header, wrap — don't shrink
  the title below its basis.
- **Chevron-only button:** `<tbt-section>` with NO title renders a toggle button whose
  only content is "▾" → axe `button-name` **critical**. Component now falls back to
  `aria-label="Toggle section"` when title is empty. If a page header section doesn't
  need collapsing at all, prefer `not-collapsible` (§17).

---

## 26. Compact filter dropdown in an actions slot → `auto-width`

**Trap:** `tbt-dropdown` defaults to `select { width:100% }` (right for field-grids —
§9 needs shrinkability). But as a **flex item in a section's actions slot** the host
is content-sized → circular sizing collapses to min-content → a 78px box, placeholder
truncated to "ทุกสถ…" at every viewport.

**Fix (shipped):** `<tbt-dropdown auto-width ...>` → `width: max-content` on the
select/trigger. Rule of thumb: field in a grid = default; filter chip in a toolbar =
`auto-width`. Do NOT "fix" it with a `style=` attribute on the page (hard rule #2).

---

## 27. Language + locale traps that pass every unit test

- **Backend messages surface verbatim in the UI alert** — `validate()` and
  `checkTransition()` strings in `*_lib.js` / `*_meta.js` ARE user-facing copy.
  Write them in Thai at the source; don't bolt a translation layer on the client.
- **Component English defaults leak into Thai pages** — `tbt-table` "No data",
  `tbt-audit-log` "No activity yet", `tbt-dropdown` "Select…", `tbt-approval-flow`
  "Approved/Awaiting/Pending". All have override attrs (`empty-message`,
  `placeholder`, per-step `statusLabel`) — every Thai page must set them.
- **Native `<input type=date>` renders the BROWSER's locale** (mm/dd/yyyy) while the
  rest of the page shows ISO — two date formats on one screen. No cheap fix; pending
  RFC #29 for tbt-datepicker display format. Don't hand-roll a per-page fix.
- **Floating mobile hamburger + drawer:** `tbt-app-shell` reflects `drawer-open`;
  page CSS hides the fixed ☰ while open (`body:has(tbt-app-shell[drawer-open])`) —
  without it the button overlaps the drawer brand ("o ERP").

---

## 28. wtr test hangs at `fixture()` ONLY in the full suite — plain-element root + hidden page = rAF never fires

**Symptom:** a test file passes standalone (`npx wtr tests/x.test.js`) but every test
in it fails with `Timeout of 2000ms exceeded` when run with ANY second file. Raising
mocha timeout / Chrome `--disable-*-backgrounding` flags does NOT help. Proven root
cause (2026-07-17, #64): concurrent wtr test pages are `visibilityState: hidden` tabs,
and headless Chrome never fires `requestAnimationFrame` in hidden tabs — open-wc
`fixture()` falls back to `nextFrame` (rAF) whenever the template ROOT is not a Lit
element (e.g. `<div>` wrapping components) → awaits forever.

**Fix:** never `fixture()` a template whose root is a plain element. Either fixture a
single Lit component (its `updateComplete` path avoids rAF), or build the DOM manually:

```js
const wrap = document.createElement('div');
wrap.innerHTML = `...components...`;
document.body.appendChild(wrap);
await Promise.all([...wrap.children].map(el => el.updateComplete));
// + afterEach(() => wrap.remove()) — manual DOM is not auto-cleaned, duplicate ids leak across tests
```

Same trap applies to anything rAF-based in tests (`nextFrame`, scroll/resize rAF loops).

---

## Workflow home: `docs/UI-PLAYBOOK.md` (in-repo, canonical)

End-to-end build → test → release → deploy → QA workflow, definition-of-done checklist
(console, dark, 375/768, axe 0 critical, real-flow save/refresh), and the open
decisions (#28 sidebar, #29 datepicker). Read it before starting any new Suitelet page;
this skill stays the negative-space companion.

---

## Compatible with skill: `teibto-ui-component`

That skill describes the components and standard layout patterns. **This skill** is the negative-space companion: what goes wrong + how to make it not go wrong.

When a fix proves itself across multiple iterations, codify it here so the next AI session inherits the lesson.
