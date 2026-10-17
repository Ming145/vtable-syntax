---
name: vtable-syntax
description: >
  Authoring guide for Vtable table source code (the Vtable plugin for Obsidian).
  Use when generating or editing ```vtable code blocks: JSON grid schema, merged
  cells, line breaks, inline formatting, task lists, math, images, Excel-style
  formulas, diagonal split headers, per-table style presets, display modes,
  RC sizes, built-in bar/pie/line charts, and combo charts merging any number
  of bar/line segments with a shared y-range. Covers every syntax element with
  copy-paste examples.
---

# Vtable Source Syntax

Vtable stores each table inside a ` ```vtable ` fenced code block. The block
content is **JSON**. When you (the AI) are asked to create or modify a Vtable
table, your output is this JSON source — the plugin renders it and writes edits
back automatically.

```text
```vtable
{"rows":[[{"t":"Name","rs":1,"cs":1},{"t":"Qty","rs":1,"cs":1}],[{"t":"Pen","rs":1,"cs":1},{"t":"3","rs":1,"cs":1}]]}
```
```

## 1. Top-level JSON schema

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `rows` | `Cell[][]` | yes | Full matrix of cells; `rows[r][c]` is the cell at row r, column c (0-based). |
| `mode` | `"OL" \| "HS" \| "RC"` | no | Display mode. Omitted = `OL`. |
| `colWidths` | `number[]` | no | Column widths in px. Only meaningful in `RC` mode. |
| `rowHeights` | `number[]` | no | Row heights in px. Only meaningful in `RC` mode. |
| `noHeaderBg` | `boolean` | no | `true` = suppress the header-row background color for this table. |
| `style` | `"three" \| "noline" \| "hline" \| "off"` | no | Per-table line-style override. |
| `zebra` | `boolean` | no | Per-table striped-rows override (`true` on / `false` off). |

Minimal valid table (a blank 1x1):

```text
```vtable
{"rows":[[{"t":"","rs":1,"cs":1}]]}
```
```

## 2. Cell schema

Every position in the matrix must contain a cell object (full-matrix rule).

| Field | Type | Meaning |
| --- | --- | --- |
| `t` | string | Cell content. Markdown inline syntax and inline HTML; line breaks are `<br>`. |
| `rs` | number | Row span (merge anchor >1). `0` = this cell is covered by a merge and hidden. |
| `cs` | number | Column span (merge anchor >1). `0` = covered/hidden. |
| `a` | `"l" \| "c" \| "r"` | Optional horizontal alignment. Default left. |
| `va` | `"t" \| "m" \| "b"` | Optional vertical alignment. Default top. Not used by diagonal cells. |
| `bg` | string | Optional cell background color — any CSS color value (e.g. `#ffcc00`, `lightblue`). |
| `dg` | string | Presence marks a diagonal-split cell: value = top-right region text, `""` = line only. |

## 3. Header row

The **first row is always the header** (bold, background, styled via CSS
`tr:first-child`). There is no header flag — just put header text in row 0.
To remove only the header background, set `"noHeaderBg": true`.

```text
```vtable
{"rows":[[{"t":"Item","rs":1,"cs":1},{"t":"Price","rs":1,"cs":1}],[{"t":"Pen","rs":1,"cs":1},{"t":"$2","rs":1,"cs":1}]],"noHeaderBg":true}
```
```

## 4. Merged cells (rowspan / colspan)

Merge = an anchor cell with `rs`/`cs` > 1. Every cell position the merge
covers must still exist in the JSON as a placeholder with `"rs":0,"cs":0`.
A 2-column merge anchored at (0,0):

```text
```vtable
{"rows":[[{"t":"Wide","rs":1,"cs":2},{"t":"","rs":0,"cs":0}],[{"t":"A","rs":1,"cs":1},{"t":"B","rs":1,"cs":1}]]}
```
```

A vertical merge spanning 3 rows:

```text
```vtable
{"rows":[[{"t":"Group","rs":3,"cs":1},{"t":"r1","rs":1,"cs":1}],[{"t":"","rs":0,"cs":0},{"t":"r2","rs":1,"cs":1}],[{"t":"","rs":0,"cs":0},{"t":"r3","rs":1,"cs":1}]]}
```
```

Rules:
- Count every covered position, otherwise columns collapse.
- The plugin self-heals orphaned placeholders on save, but always emit them correctly.
- Cell alignment/color set on the anchor applies to the merged block.

## 5. Display modes

| Mode | Name | Behavior |
| --- | --- | --- |
| `OL` | Wrap mode (default) | Cells wrap text; table fits the note width. |
| `HS` | Horizontal stretch | Rows keep single-line height; wide tables scroll horizontally with a draggable scrollbar. |
| `RC` | Free resize | Draggable column widths and row heights, stored in `colWidths` / `rowHeights` (px). |

```text
```vtable
{"rows":[[{"t":"A","rs":1,"cs":1},{"t":"B","rs":1,"cs":1}],[{"t":"1","rs":1,"cs":1},{"t":"2","rs":1,"cs":1}]],"mode":"RC","colWidths":[120,240],"rowHeights":[32,48]}
```
```

## 6. Line breaks

Line breaks inside a cell are stored as the literal tag `<br>`. Raw `\n` in the
JSON string is tolerated (normalized to `<br>` on save), but emitting `<br>`
directly is the canonical form. Never wrap `<br>` in backticks — code spans are
not parsed, so `` `<br>` `` would display as text.

```text
```vtable
{"rows":[[{"t":"First line<br>Second line","rs":1,"cs":1}]]}
```
```

## 7. Inline formatting (Markdown + HTML)

Cell content supports Markdown inline syntax (converted on render) and passes
inline HTML through. Inside `` `code` `` spans nothing is parsed.

| Syntax | Renders as |
| --- | --- |
| `**bold**` / `__bold__` | bold |
| `*italic*` / `_italic_` | italic |
| `***bold italic***` / `___x___` | bold italic |
| `==highlight==` | highlighted `<mark>` |
| `~~strikethrough~~` | strikethrough |
| `` `code` `` | inline code (contents literal) |
| `[text](url)` | link |
| `<b>x</b>`, `<i>`, `<span style="color:red">x</span>` … | raw inline HTML |
| `<br>` | line break |

```text
```vtable
{"rows":[[{"t":"**Total** and *note* and ==KEY==","rs":1,"cs":1},{"t":"<span style=\"color:red\">red</span>","rs":1,"cs":1}]]}
```
```

## 8. Task lists (checkboxes)

A line matching `- [ ]` or `- [x]` renders as a clickable checkbox. Use `<br>`
between items to stack several tasks in one cell. Clicking a checkbox rewrites
the source marker.

```text
```vtable
{"rows":[[{"t":"Todo","rs":1,"cs":1},{"t":"- [x] done item<br>- [ ] pending item","rs":1,"cs":1}]]}
```
```

## 9. Math (MathJax)

`$...$` renders inline math, `$$...$$` display math. Text inside `` `code` ``
is not processed.

```text
```vtable
{"rows":[[{"t":"Formula","rs":1,"cs":1},{"t":"$E=mc^2$","rs":1,"cs":1}],[{"t":"Display","rs":1,"cs":1},{"t":"$$\\int_0^1 x\\,dx = \\frac{1}{2}$$","rs":1,"cs":1}]]}
```
```

## 10. Images

| Syntax | Meaning |
| --- | --- |
| `![alt](url)` | Markdown image |
| `![[filename.png]]` | Obsidian vault embed |
| `![[filename.png|120]]` | Vault embed with width in px |
| `<img src="..." width="120">` | Raw HTML image (also fine) |

```text
```vtable
{"rows":[[{"t":"Photo","rs":1,"cs":1},{"t":"![[photo.png|80]]","rs":1,"cs":1}]]}
```
```

## 11. Formulas

A cell whose content is exactly `=FUNC(range)` is evaluated at render time;
while editing the cell shows the original formula. Ranges use Excel-style
addresses (`A1`, `A1:B3`); column letters then 1-based row numbers.

| Function | Result |
| --- | --- |
| `SUM(A1:B3)` | Sum of numeric cells in range |
| `AVERAGE(A1:A5)` | Arithmetic mean (0 when empty) |
| `COUNT(A1:A5)` | Count of numeric cells |
| `MIN(A1:A5)` | Minimum (0 when empty) |
| `MAX(A1:A5)` | Maximum (0 when empty) |

Notes:
- Only these five functions; no cell arithmetic like `=A1+A2` and no `AVG` alias.
- Hidden merge placeholders and non-numeric cells in the range are skipped.
- Text starting with `=` that does not match `=FUNC(range)` displays literally.

```text
```vtable
{"rows":[[{"t":"Item","rs":1,"cs":1},{"t":"Qty","rs":1,"cs":1}],[{"t":"Pen","rs":1,"cs":1},{"t":"3","rs":1,"cs":1}],[{"t":"Book","rs":1,"cs":1},{"t":"4","rs":1,"cs":1}],[{"t":"Total","rs":1,"cs":1},{"t":"=SUM(B2:B3)","rs":1,"cs":1}]]}
```
```

## 12. Diagonal split cells (corner header)

A cell with a `dg` field is drawn with a diagonal line:
`dg` = top-right region text, `t` = bottom-left region text. Both regions are
single-line plain text (no HTML, no `<br>`). `dg: ""` draws the line with an
empty top-right region. Diagonal cells ignore `va`.

```text
```vtable
{"rows":[[{"t":"Item","dg":"Region","rs":1,"cs":1},{"t":"Q1","rs":1,"cs":1},{"t":"Q2","rs":1,"cs":1}],[{"t":"North","rs":1,"cs":1},{"t":"10","rs":1,"cs":1},{"t":"20","rs":1,"cs":1}]]}
```
```

The example's first cell shows "Region" top-right, "Item" bottom-left, with a
diagonal line between them — the classic cross-tab corner.

## 13. Alignment and background

```text
```vtable
{"rows":[[{"t":"Centered","rs":1,"cs":1,"a":"c","va":"m","bg":"#fff3cd"},{"t":"Right","rs":1,"cs":1,"a":"r"}]]}
```
```

`bg` accepts any CSS color: names, hex, `rgb()`. Omitted fields simply fall
back to defaults (left / top / transparent).

## 14. Per-table style presets

| `style` value | Look |
| --- | --- |
| `"three"` | Three-line (booktabs) table: thick top/bottom rules, thin rule under header, no vertical lines |
| `"noline"` | No grid lines at all |
| `"hline"` | Horizontal lines only |
| `"off"` | Explicitly cancel a globally enabled line style for this table |
| omitted | Follow the plugin's global setting |

`"zebra": true/false` toggles striped rows for this table; `"noHeaderBg": true`
removes the header background.

```text
```vtable
{"rows":[[{"t":"A","rs":1,"cs":1},{"t":"B","rs":1,"cs":1}],[{"t":"1","rs":1,"cs":1},{"t":"2","rs":1,"cs":1}]],"style":"three","zebra":true}
```
```

## 15. Built-in charts

A table becomes a chart when **any cell in the first row** carries a chart type
keyword: `vbarchart` (bar), `vline` (line), `vpie` (pie). Data rows are
`[label, value]` pairs in two adjacent columns; values may include non-numeric
decorations (stripped) but must parse to a number.

**Keyword labels are bilingual.** Chinese (`标题` / `x范围` / `y范围`) and
English (`Title` / `x range` / `y range`) variants are interchangeable;
`x`, `y`, `v%` are the same in both. Values `是` / `yes` / `1` all turn the
`v%` flag on (`否` / `no` = off). The examples below use Chinese — either
language works.

**Row order is lenient** for single charts: the `v%` row may sit anywhere
(templates put it last). Minimum row counts: bar ≥ 7 (header rows + at least
one data row), line ≥ 7, pie ≥ 3. The `x范围` row is accepted but ignored
(the x-axis name comes from the `x` row) — you may omit it entirely.

### Bar chart — first cell of row 0 is `vbarchart`

```text
```vtable
{"rows":[
[{"t":"vbarchart","rs":1,"cs":1},{"t":"","rs":1,"cs":1}],
[{"t":"标题","rs":1,"cs":1},{"t":"Sales","rs":1,"cs":1}],
[{"t":"x","rs":1,"cs":1},{"t":"Quarter","rs":1,"cs":1}],
[{"t":"y范围","rs":1,"cs":1},{"t":"0-100","rs":1,"cs":1}],
[{"t":"y","rs":1,"cs":1},{"t":"Units","rs":1,"cs":1}],
[{"t":"Q1","rs":1,"cs":1},{"t":"40","rs":1,"cs":1}],
[{"t":"Q2","rs":1,"cs":1},{"t":"65","rs":1,"cs":1}],
[{"t":"Q3","rs":1,"cs":1},{"t":"80","rs":1,"cs":1}],
[{"t":"v%","rs":1,"cs":1},{"t":"no","rs":1,"cs":1}]
]}
```
```

Row meanings: `v%` = show-values-as-percentage flag (bar charts only); `y范围`
= y-axis range `"lo-hi"` (hi becomes y-max); `y` = y-axis name; `x` = x-axis
name; then label/value data rows.

### Pie chart — first cell of row 0 is `vpie`

```text
```vtable
{"rows":[
[{"t":"vpie","rs":1,"cs":1},{"t":"","rs":1,"cs":1}],
[{"t":"标题","rs":1,"cs":1},{"t":"Market share","rs":1,"cs":1}],
[{"t":"Alpha","rs":1,"cs":1},{"t":"45","rs":1,"cs":1}],
[{"t":"Beta","rs":1,"cs":1},{"t":"30","rs":1,"cs":1}],
[{"t":"Gamma","rs":1,"cs":1},{"t":"25","rs":1,"cs":1}]
]}
```
```

Values must be positive.

### Line chart — first cell of row 0 is `vline`

```text
```vtable
{"rows":[
[{"t":"vline","rs":1,"cs":1},{"t":"","rs":1,"cs":1}],
[{"t":"标题","rs":1,"cs":1},{"t":"Growth","rs":1,"cs":1}],
[{"t":"x","rs":1,"cs":1},{"t":"Month","rs":1,"cs":1}],
[{"t":"y范围","rs":1,"cs":1},{"t":"0-100","rs":1,"cs":1}],
[{"t":"y","rs":1,"cs":1},{"t":"Index","rs":1,"cs":1}],
[{"t":"Jan","rs":1,"cs":1},{"t":"10","rs":1,"cs":1}],
[{"t":"Feb","rs":1,"cs":1},{"t":"35","rs":1,"cs":1}],
[{"t":"Mar","rs":1,"cs":1},{"t":"60","rs":1,"cs":1}]
]}
```
```

### Combo charts — any number of bar/line segments in one table

Put **any number of bar/line segments** in one wide table; the plugin renders
one fused SVG. Structure rules:

- **Each segment owns two adjacent columns**: its left column holds the type
  marker (row 0), the category labels (data rows) and its config keywords
  (`y范围` / `y`); its right column holds the values.
- **Row 0 declares every segment's type** in its own left column:
  e.g. `vbarchart｜｜vline｜｜vline｜` = bar + line + line. Any mix and count
  works (`vpie` does not join combos — pie has no shared axes).
- **Config rows may be shared side by side**: `y范围` / `y` of different
  segments sit on the same row, each in its own segment's columns. Rows may
  also be staggered — the parser scans each segment's columns independently.
- **Every segment's category labels must be written in its own left column**
  (repeat the label per segment — a segment whose left column is empty has no
  data).
- **Shared y-range**: the axis takes the smallest lower bound and the largest
  upper bound across all segments' `y范围` (e.g. `0-3` and `1-5` → `0-5`).
- **Title and x/y axis names come from the leftmost segment**.
- **Category axis** = the leftmost bar segment's categories; other segments'
  data aligns to matching labels (unmatched labels are ignored). With no bar
  segment, the leftmost segment's categories are used.
- **Multiple bar segments** render side by side within each category slot;
  line segments overlay as polylines aligned to bar centers.
- `v%` is bar-only; **any** bar segment writing `是`/`yes` turns percentage
  display on.
- Put the `v%` row at the very bottom of the table.

Bar + line combo:

```text
```vtable
{"rows":[
[{"t":"vbarchart","rs":1,"cs":1},{"t":"","rs":1,"cs":1},{"t":"vline","rs":1,"cs":1},{"t":"","rs":1,"cs":1}],
[{"t":"标题","rs":1,"cs":1},{"t":"Sales & Profit","rs":1,"cs":1},{"t":"","rs":1,"cs":1},{"t":"","rs":1,"cs":1}],
[{"t":"x","rs":1,"cs":1},{"t":"Month","rs":1,"cs":1},{"t":"","rs":1,"cs":1},{"t":"","rs":1,"cs":1}],
[{"t":"y范围","rs":1,"cs":1},{"t":"0-100","rs":1,"cs":1},{"t":"y范围","rs":1,"cs":1},{"t":"20-80","rs":1,"cs":1}],
[{"t":"y","rs":1,"cs":1},{"t":"Sales","rs":1,"cs":1},{"t":"y","rs":1,"cs":1},{"t":"Profit","rs":1,"cs":1}],
[{"t":"Jan","rs":1,"cs":1},{"t":"30","rs":1,"cs":1},{"t":"Jan","rs":1,"cs":1},{"t":"50","rs":1,"cs":1}],
[{"t":"Feb","rs":1,"cs":1},{"t":"60","rs":1,"cs":1},{"t":"Feb","rs":1,"cs":1},{"t":"70","rs":1,"cs":1}],
[{"t":"Mar","rs":1,"cs":1},{"t":"45","rs":1,"cs":1},{"t":"Mar","rs":1,"cs":1},{"t":"65","rs":1,"cs":1}],
[{"t":"v%","rs":1,"cs":1},{"t":"no","rs":1,"cs":1},{"t":"","rs":1,"cs":1},{"t":"","rs":1,"cs":1}]
]}
```
```

Three segments (bar + line + line, 6 columns) — same pattern, extend each row
with the extra segment's two columns. A single marker sitting **not in column
A** (e.g. only `vline` in column C) also renders through the combo engine,
degenerating to a plain single chart.

### Chart detection summary

| Row 0 content | Rendered as |
| --- | --- |
| `vbarchart` in column A only | bar chart (single-chart path) |
| `vline` in column A only | line chart (single-chart path) |
| `vpie` in column A only | pie chart |
| two or more markers in row 0, or a single marker not in column A | combo engine |
| `vpie` anywhere in a multi-marker row 0 | pie ignored (renders bar/line combo) |

## 16. Markdown table compatibility

Non-JSON block content falls back to pipe parsing: each line becomes a row,
split on `|`. So a plain Markdown pipe table pasted inside the block still
renders as a table (no merge/alignment, one text cell per pipe field):

```text
```vtable
| A | B |
| 1 | 2 |
```
```

To get full features from existing Markdown/HTML tables, use the plugin's
convert commands (`Convert to vtable`) — the converter strips separator rows
and applies inline formatting. As an AI, prefer emitting native JSON directly.

## 17. Escaping rules

- In JSON strings escape `"` and `\` as usual (`\"`, `\\`).
- Inside pipe-table fallback rows, a literal `|` is written `\|`.
- `&`, `<`, `>` in plain text are handled by the plugin; write text naturally.
- Inline HTML you emit (e.g. `<span style="color:red">`) must be valid within a
  JSON string (escape its quotes).

## 18. Pitfalls checklist

- **Full matrix**: every covered merge position must appear with `"rs":0,"cs":0`.
- **`<br>` never inside backticks** — code spans render literally.
- **`AVERAGE`, not `AVG`**; formulas need the exact `=FUNC(range)` shape.
- **Charts are row-format sensitive**: keyword rows must match the bilingual
  labels exactly (`标题`/`Title`, `x范围`/`x range`, `y范围`/`y range`);
  minimum row counts enforced (bar 7, pie 3, line 7); the `x范围` row is
  optional and ignored.
- **Combo charts**: every segment's category labels must be written in that
  segment's own left column (an empty left column = that segment has no data);
  config rows of different segments may share one row; the `v%` row goes last.
- **Diagonal cells are plain text only** — no formatting in `t`/`dg` there.
- **Mermaid is not supported** in cells; a ` ```mermaid ` block inside a cell is
  treated as an ordinary fenced code block, not a diagram.
- Row/column addresses for formulas are Excel-style over the grid
  (A = first column, 1 = first row).

## 19. Complete example

One table combining header, merge, style preset, alignment, formula, task,
math, and background:

```text
```vtable
{"rows":[
[{"t":"Task","rs":1,"cs":2,"a":"c"},{"t":"","rs":0,"cs":0},{"t":"Score","rs":1,"cs":1}],
[{"t":"alpha","rs":1,"cs":1},{"t":"- [x] draft","rs":1,"cs":1},{"t":"80","rs":1,"cs":1,"bg":"#e8f5e9"}],
[{"t":"beta","rs":1,"cs":1},{"t":"- [ ] review","rs":1,"cs":1},{"t":"90","rs":1,"cs":1,"bg":"#e8f5e9"}],
[{"t":"Average","rs":1,"cs":1,"a":"r"},{"t":"$\\bar{x}$","rs":1,"cs":1},{"t":"=AVERAGE(C2:C3)","rs":1,"cs":1,"a":"c"}]
],"style":"three","zebra":true}
```
```

Keep JSON on a single line per table or pretty-printed — both are accepted;
the plugin re-serializes compactly after the first edit.
