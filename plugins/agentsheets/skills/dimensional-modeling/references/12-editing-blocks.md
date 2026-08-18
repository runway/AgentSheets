# Editing blocks in place

Use this file when changing an existing table. It builds on the axis model and
signature in references/03-table-blocks.md and the method in
references/07-modeling-method.md.

## 12.1 An edit requires a write

**Saying you changed a table does not change it.** “Filtered to East” requires
an `edit_model_views` call. “Narrowed the window to H1” and “switched it to
quarterly” require `edit_table_blocks`. Until the call returns `applied`,
nothing changed. After every edit, read the echo in the tool
result — the signature for a structural edit, the `readback` for a
presentation edit — and confirm the changed part actually changed. That
echo is the proof, not your intention.

Two tools split the work by what kind of change it is:

- **Structure** — which variables, which dimensions, how nested, what
  filter, which axis on which side — is `edit_model_views`, declaring
  the table's view (§12.2).
- **Presentation** — name, window, granularity, comparison, transpose,
  sort, column widths, visibility — is `edit_table_blocks`, one call
  naming the block, never restating the table (§12.3).

## 12.2 Structural edits: declare the view

`edit_model_views` (`change.configure_table`, naming the existing table) with
`view` declares what the table computes and reconciles it onto the SAME
block. Structure that matches the current config keeps its axis ids,
aggregate functions, and sort. The formula lane and surviving segment
drill-ins carry over, and so does presentation state (window,
comparison, widths, hidden axes, formatting, drill-ins) — a view says
nothing about presentation, so a view write can never drop it, with one
exception: orientation. An update omitting `transpose` flips a transposed
non-mapping block back (references/11-block-grammar.md §11.3); restate
`transpose: true` on such a block's updates. Anything
the reconcile had to drop comes back as a warning. A view that fails
validation returns its problems and saves nothing: the writes validate
themselves, so call directly; reach for `dry_run` when an update is risky
enough to check first (SKILL.md axiom 17). A create reports the config
checks itself, as advisory warnings on the applied item.

Use three steps:

1. Start from the block's current `view` (`inspect_table_blocks` `ask.list`
   — the calculated data read does not carry one). Never write an
   update from memory.
2. Edit the variables and breakdowns that change. The view is only the
   structure, so there is no window or comparison line to restate and
   no omission that clears one.
3. Send it, then check the echoed signature and `signature_status` in
   the result.

This path covers what a view declares: re-slice, re-nest, re-filter,
pivot (move a dimension between a variable's breakdown and the shared
one — always safe, pivoting is placement).

One of those moves is not free. **Nesting a level adds a grain.** Pivoting
and filtering rearrange cells the model already computes, but a new drill-in
asks the engine for a coordinate no one has asked for before. A subset `[…]`
formula follows when the new grain retains its named dimensions; an exact
`$[…]` formula does not (references/09-the-layer-model.md §9.1). So on any
edit that adds or deepens a breakdown, run the grain check on the added levels
(§9.4). Then read the numbers, not just the echoed signature: the signature
confirms the shape you asked for, while the readback grid shows whether a
subset rule followed or an exact rule correctly stopped. A correct parent
above a level of zeros means the new grain has no matching rule; fix the
formula intent rather than editing the block again.

## 12.3 Presentation edits: one call on the named block

`edit_table_blocks` takes ONE named block and batches presentation
aspects in a single call: rename, window
(`{start, end, granularity}`), comparison, transpose, sort, column
widths, visibility. Formatting is the exception — it never rides with a
window, comparison, or transpose change; §12.7 owns its batching rule.
The table is never restated. Given
a monthly block over 2025-01..2026-12 whose signature reads:

```
Revenue [Customer]
by [Date.Month]
```

"narrow this to the first half of 2026" is one `edit_table_blocks` call
setting `window: { start: "2026-01-01", end: "2026-06-30" }`. The result
echoes a computed `readback`; it must show 2026-01..2026-06. If it
still shows the old range, the edit did not happen. Do not tell the
user it did.

A relative ask — "widen it by a year", "one more quarter" — is computed
from the window the block has now, never from memory of an old read. Two
places state it in the same `{start, end, granularity}` the write takes:
every write's `showing` line, which says what the block shows and what it
showed before when the call moved it, and each `presentation` in
`inspect_table_blocks` `ask.list`, beside the signature. So a follow-up
right after your own edit needs no read at all, and any other relative ask
needs the survey you would already run to name the block.

Widening, shifting, and granularity changes are the same shape — one
window aspect, and bucket and range can change together. Structure is
untouched, so drill-ins and formulas are too. One rule owns the
granularity case:

- **A granularity change re-routes formula variables through their time
  rollup** at the new coarser granularity (default per variable,
  per-grain override via `edit_variables` `change.set_time_rollup`) —
  confirm each matches the interval question's answer (§13.1's
  when-to-set rule in references/13-metric-recipes.md). On a block
  carrying stock variables (LAST-family aggregation), read the stored
  aggregation back first: a sync can clobber an explicit LAST back to
  DO_NOT_AGGREGATE (references/limitations.md §1).

Because it is this cheap, a narrow looks like a debugging or speed
trick: shrink the range, calculate, then restore. Do not. The window is
what the user sees. A saved block is their deliverable, not scratch
space (a SKILL.md axiom). To check work cheaply, evaluate ephemerally
(§2.8) or read the write's readback — neither touches the block. Narrow
a saved block only when the narrow window is the ask.

## 12.4 Comparison asks are block edits

"Show the change versus last month/quarter/year", MoM/QoQ/YoY, "versus
budget" — these are block edits, not questions to answer in chat. A chat
answer scrolls away. The block remains and keeps showing the
comparison. What a block is shown against is presentation, so it is one
`edit_table_blocks` call and the table itself is never restated. On a
quarterly table, "the change versus the quarter before it" is:

```
comparison: { period_offset: 1, measures: ["CURRENT", "DELTA", "PERCENT"] }
```

The `measures` vocabulary differs by comparison kind, through the one
shared field: a period comparison (`period_offset` > 0) takes `CURRENT`,
`COMPARISON_VALUE`, `DELTA`, `PERCENT`; a scenario comparison
(`scenarios` set) takes `VALUE`, `DELTA`, `PERCENT`; and `layout`
(`COLUMNS`/`ROWS`) belongs to scenario comparisons only — it is refused
on a period comparison.

The offset counts in the table's OWN granularity: one bucket back is `1` at any
granularity. Do not pattern-match the period words to a year offset.
`period_offset: 4` on a quarterly table is year-over-year, and an offset
that reaches before the data starts leaves the comparison blank. "Versus
budget" is the scenario form, `comparison: { scenarios: ["Budget 2026"],
measures: ["VALUE", "DELTA"] }`. Setting one kind clears the other:
scenarios and period offset do not stack. The tool refuses a period
comparison on a block with no date axis, and refuses comparing a block
against its own scenario. Either way the result's `comparing` line
reports what the block shows itself against, and `ask.list`'s
`presentation` reports it before you write. A compared scenario comes
back as the token that names it and nothing else: its name, or
`Name#hex` where another scenario answers to the same name — the same
disambiguator a variable or a table takes. Pass back what the read
printed and it binds to the one scenario you read.

## 12.5 Edit in place or duplicate?

Two write paths reshape an existing table. Pick by what the user means:

| the user means                                                                                               | call                                                                                                                                     |
| ------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| _this_ table, changed — "narrow it", "make it quarterly", "add Region"                                       | name the table: same block, stable axis ids, nothing else appears                                                                        |
| _another view_ of this table — "show me this by product too", "a copy for the board page", keep the original | `copy_from`: the source table plus the edited view; a NEW block with fresh ids, the source untouched, formula lane and drill-ins carried |

Never create a sibling block when the user is pointing at a table they
want changed. That leaves two near-identical blocks and a stale
original. Duplicate lands on the source's page unless `page` targets
another. That makes it the right tool for "same table, other page" too.

## 12.6 Blocks a view cannot fully describe

A `signature_status` of `partial (…)` names structure only the config
rendering can touch — generated axes, role overrides, per-item
overrides, multi-entry axes, drill-in paths beyond a branch condition,
the formula lane (references/03-table-blocks.md §3.9). A view update
against those rejects with guidance rather than silently dropping user work.
There is no raw-config path to fall back to: say what the view cannot express
and stop, rather than rebuilding the block by hand.

## 12.7 Formatting: the same styles users set by hand

"Make the EMEA row teal", "bold the total column", "highlight that
cell" — `change.formatting` on `edit_table_blocks` writes the exact
styles a user sets from the table's own menus, so they see and can
clear everything you set. It takes up to 20 `writes`, each a target
plus a `style`:

```
formatting: { writes: [
  { row: "Opex > Engineering", style: { bg: "teal", bold: true } },
  { column: "2026-03", style: { italic: true } },
  { row: "Opex > Engineering", column: "2026-03", style: { bg: "red" } }
] }
```

Name what the grid printed. A row is its path (`"Opex > Engineering"`), a
column is its label — the same words `ask.calculate` shows you, so there is
nothing to look up.

Date columns print as a period START, never as a period name, so the label
rarely looks like the word the user said. Every granularity prints one of
two shapes:

| Granularity | A column prints                            | Also reachable as                             |
| ----------- | ------------------------------------------ | --------------------------------------------- |
| Year        | `2026-01`                                  | `2026`, `FY2026`                              |
| Half        | `2026-01`, `2026-07`                       | `H1 2026`, `H2 2026`                          |
| Quarter     | `2026-01`, `2026-04`, `2026-07`, `2026-10` | `Q1 2026` … `Q4 2026`                         |
| Month       | `2026-03`                                  | `Mar 2026`, `March 2026`, `Mar'26`, `03/2026` |
| Week        | `2026-03-09` (the week's start)            | `Mar 9, 2026`, `9 Mar 2026`                   |
| Day         | `2026-03-09`                               | `Mar 9, 2026`, `9 Mar 2026`                   |

Read the right-hand column as a convenience for when you are working from
the user's words rather than a read — the printed label always works, and a
read is still the way to be sure. A period spelling only works at its OWN
granularity: `"Q3 2026"` on a monthly table would name July alone, so it is
refused rather than styling a third of what was asked for. The same goes the
other way — a month token does not sweep up the weeks inside it.

Row alone styles the whole row (a
member row scopes to that member everywhere it appears), column alone the
whole column, both together that one cell. A `c1..cN` id
does NOT work: it is a position in the grid you happened to read, and a
boundary-period or comparison read numbers the same columns differently,
so the write refuses it rather than risk styling a column you did not
mean. Prefer whole-row and whole-column scopes: a cell write per grid
cell bloats the block's stored settings.

Colors are named — purple, teal, orange, yellow, red, pink, blue, green,
yellow green, cyan, azure, indigo, magenta. Styles have two states, on
and inherited: there is no "off". `null` clears an attribute so the scope
inherits broader styles again; `false` is refused. More specific scopes
win per attribute — a cell's fill beats its row's, but the row's bold
still applies. `indent` is row-only and absolute (0–8; 0 clears).
Formatting combines with rename, column widths, visibility and sort; a
window, comparison or transpose change goes in its own call first —
those move what the grid contains, and formatting names what it saw. Comparison
rows and columns, scenario-variant rows, and source rows take no styles —
the tool names why when it refuses.

The result's `formatted` lines echo what was stored per write — and say
"already as requested" for writes that changed nothing, so a no-op is a
fact you report, not a success you assume.

To see what a table already carries, ask `inspect_model_views`
`ask.formatting` with the table's name. It answers in the same words the
write takes — rows by path, columns by label, colors by name — so what it
returns can be edited straight back. Read it before "make this match that
one" or "clear the highlighting"; the stored styles are not in the
calculated grid. It also lists styles the table stores that no current row
or column matches: formatting written before a rename or a re-slice, which
renders nowhere and is worth clearing.
