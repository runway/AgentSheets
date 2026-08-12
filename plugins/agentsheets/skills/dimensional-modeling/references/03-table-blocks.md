# Table blocks: views into the space

How a grid of numbers is declared. Builds on the axioms in SKILL.md and the
vocabulary of references/01-the-dimensional-universe.md (segment, grain,
item). Recipes are in references/08-recipes.md; the laws about what breaks
are in references/06-validity.md.

## 3.1 A block is a view definition

A table block stores no data. Its config is two ordered trees of **axes**,
one for rows and one for columns, plus block-level settings: name, date
range, granularity (monthly vs quarterly), and comparison state.
An axis is one "break down by \_\_\_" rule carrying one entry. The engine
turns each rule into a fan-out: one row (or column) per dimension item, or a
single row for a variable. Nesting composes fan-outs. Everything the table
shows is derived from config + evaluation; nothing is stored per cell.

Read a config the way you would read its title: the variable plus the chain of
dimensions under it is the table's sentence, "Revenue by Region by Product,
over Months". Titles are generated in exactly that shape.

## 3.2 Four axis kinds

- **Property axis** is the one you author: it points at one entry and
  carries all the knobs below.
- **Formula axis** is the formula lane: a column (or row) whose cells show
  the formula expressions behind the crossing variable, for editing. Every
  persisted table gets one seeded automatically. A table whose only axis is
  the formula lane still counts as empty. It computes nothing.
- **Freeform** axes exist in the schema; never author one. The only live
  freeform is the virtual "comparison" placeholder, and the renderer
  manages it itself.
- **Flattened** axes exist in the schema too; never author one.

One default runs at create: a table whose columns hold no non-formula axis
while its rows carry a native variable silently gains one shared root
system-Date column (an update does this only when the existing block is
blank). A deliberately timeless table therefore needs at least one
non-formula axis — the entity breakdown of a listing — to keep the default
from firing.

## 3.3 The knobs on a property axis

| knob                  | meaning                                                                      | notes                                                                                                                            |
| --------------------- | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| property              | which dictionary entry this rule breaks down by                              | one entry per axis; "X by Y" is expressed by nesting, never by stacking two entries on one node                                  |
| useAs                 | role override: DIMENSION (a segmentation) or VARIABLE (a value-axis mapping) | a dimension in the VARIABLE role shows its item in each cell instead of a measured value; ignored (silently) on native variables |
| item inclusion filter | an allow-list of item values                                                 | empty means all items; exact string match, so stale values silently drop rows                                                    |
| sort                  | ascending, descending, manual ordering, or NONE to clear a sort              | by item identity only; there is no sort-by-cell-value                                                                            |
| item generation       | MATCH (only items present in data) or GENERATE over a range                  | date axes implicitly GENERATE across the table's date range                                                                      |
| date granularity      | per-axis time bucket override                                                | full precedence rules in references/04-time.md                                                                                   |
| children              | nested axes, the drill-in chain                                              | see 3.4                                                                                                                          |
| drill-in path         | breadcrumb of how this node was reached; a step can pin one specific item    | pinning is how "drill into Engineering only" is recorded                                                                         |
| override id           | marks this node as a per-item specialization of a sibling axis               | see 3.5                                                                                                                          |

Sibling order in the config arrays is the primary display order, ahead of
any item sort.

## 3.4 Nesting, and what pivoting really is

Nesting one axis under another multiplies fan-outs and refines the grain.
Rows `Revenue > Region > Product` produce: one Revenue total row, one row per
region, one row per (region, product). Each level's cells are the variable
evaluated at that level's grain, which is why parents are recomputed
aggregates rather than sums of their visible children (references/02-formulas.md §2.4).

**Pivoting an axis (rows to columns or back) changes placement and nothing
else.** The entry, filter, sort, granularity, children, and pinning all
travel with it. The grain of every cell is the union of its row path and
column path dimensions, and unions do not care which side a dimension came
from. "Revenue by Region over Months" and "Revenue by Months over Region"
contain identical numbers, transposed. To flip the whole table, swap its
complete root trees: `rows: Revenue > Region; cols: Date` becomes `rows:
Date; cols: Revenue > Region`. Change the row/column type of every moved node,
preserve each subtree's hierarchy and knobs, and keep every child `parentId`
anchored to its parent in that same subtree. Pivot moves are always safe.

There is no per-move edit vocabulary. A structural edit — including a pivot —
restates the whole view: `edit_model_views` `change.configure_table`, naming the
existing table to reconfigure it in place (references/12-editing-blocks.md
owns the edit rules and the structural-vs-presentation split). The system
re-anchors drill-in breadcrumbs and carries presentation state across
restates; you maintain neither by hand.

## 3.5 The drill-in family

One user verb ("drill in"), several mechanisms. Knowing which is which
matters when editing configs:

1. **Drill in by dimension** adds a child property axis under a node. The
   items appear as nested rows. When drilling an axis that already has
   children, the old children re-parent under the first new level, keeping a
   chain rather than a fan.
2. **Drill in to one item** pins the child to a single dimension item
   (recorded in the breadcrumb or a one-element inclusion filter): "show me
   Engineering only, under this row".
3. **Per-item overrides** specialize one item's subtree. An override node
   references its base sibling, inherits the base's child tree, and appends
   its own children. The base then _excludes_ that item so it appears only
   once. This is how "break Engineering down differently from the other
   departments" is represented. Overrides must stay siblings of their base.
4. **Segment drill-ins** attach a drill-in to a _generated_ header that has
   no config node of its own (a specific month column, one region under a
   breakdown). They live in a side list on the config, anchored by the path
   of the clicked header, and are merged into the tree at read time. Stale
   anchors are pruned automatically on structural edits.
5. **Drill in by input** is not config
   nesting at all: it is a request-time expansion that shows the variables
   referenced by a row's formula as child rows, ending in terminal raw
   source line items. Use it when the data has no dimensional breakdown to
   offer; prefer dimension drill-ins when it does.

A drill-in-by-input source-row listing binds the parent row's pinned segment
exactly: when a pinned value cannot be expressed as a source filter, the
listing declines rather than widen past its cell, and a listing cut at the
row cap reports the truncation. It works single-scenario only and cannot
combine with comparisons.

What the first four share: they change the grain the cells beneath them are
evaluated at. That is a modeling consequence, not a display one — the
variable needs a formula that reaches the new grain, or those rows fall to
the floor and render zeros under a parent that still looks right
(references/09-the-layer-model.md). Decide the drill-ins a block will carry
before writing its formulas, not after.

## 3.6 Cells, and the variable placement law

A cell is the crossing of one row path and one column path: the variable
evaluated at the union of their segments — or, where the value slot holds a
value-axis mapping (a bare dimension, references/11-block-grammar.md §11.1),
that dimension's item for the crossing. Hence the law ("the variable lives on
exactly one side" in SKILL.md): along any crossing there must be exactly one
variable or none, and it cannot sit on both sides. One variable per crossing
means one empty slot in every cell address, which is how results decode back
into "variable + segments".

Corollaries: dimension-only tables are legal item listings with no value
cells. Variable-under-variable nesting, or variables on both sides, saves fine and
fails at calculation (references/06-validity.md). The engine returns one
value per cell; a config that makes a cell ambiguous (the same dimension
crossed with itself) yields silent blanks, not errors.

## 3.7 Block-level state

- **Name and prompt.** Names are AI-generated in the "&lt;Variables&gt; by
  &lt;Dimensions&gt;" shape until a user renames (a flag records the override).
  The prompt preserves what the table was asked to be.
- **Date range and granularity** (references/04-time.md): the visible time window and default
  time bucket, both block-level with per-axis overrides.
- **Comparison state** (references/04-time.md/references/05-scenarios-and-comparisons.md): scenario comparison or time comparison,
  mutually exclusive, plus which measures (value, variance, variance %) are
  visible.
- **Hidden axes** hide a breakdown's rendering only; hidden axes still
  compute. Formula-column display and column widths are block presentation
  too; row expansion is the only local UI state here.

Presentation — rename, column widths and the formula column, axis
visibility, window, comparison — is edited on the live block through
`edit_table_blocks`; references/12-editing-blocks.md owns that surface.
Number formatting (currency, decimals, variance coloring) lives on the
entry, not the block; an entry with no explicit precision renders at 0
decimal places, so a value that looks rounded is the format default, not
the engine.

## 3.8 What a table block cannot express

Requests will ask for these; the answer is a workaround, not a config field:

- **No top-N or limit-by-value.** Item restriction is only the inclusion
  filter (explicit list) or drill-in pinning.
- **No sort-by-cell-value.** Sorting is by item identity or manual order.
- **No two variables in one cell, no variable acting as a dimension.**
- **No per-axis display rename.** Labels come from entry names.
- **No cross-block references.** Blocks share the space through entries
  and formulas, not by referencing each other's cells.
- **No combined scenario + time comparison.**

No workaround filters an axis by a variable's value; no config mechanism does.
The standard workarounds: show a computed flag or rank variable as its own
stack and let readers scan it; answer analytically with an ephemeral
evaluation, naming the offenders in chat; or snapshot the qualifying items
into an inclusion filter, flagged as a point-in-time list that will go
stale.

## 3.8b Ranked reads

Ranked questions are queries, not blocks. The engine cannot sort a block by
value (no sort-by-cell-value, above), so "top N customers", "bottom 5 by margin", or
"which items drove the drop" must never become a rendered table or an
enumeration in chat. Use `inspect_model_views` `ask.rank`: a view plus a
typed ranking naming the dimension to rank, `n`, and the key. Ranking
runs server-side, returns only the ranked rows, and persists nothing;
n ≤ 50. The keys:

- `by: value` ranks by the variable at the window grain.
- `by: change` ranks by last period minus first. It needs a Date axis
  and a window of at least two periods.
- `by: delta` / `by: percent` read a comparison's own measures, so the
  ask carries a typed `comparison` (`period_offset`) beside the view.
  `by: change` there is rejected; the comparison already computes the
  move. Ranking an ad-hoc view against a named scenario is not
  supported — rank a saved block for that.

Reading a rank response: each row carries `key`, the exact number it was
ranked by. Read the move straight from `key`; never re-derive it from the
period cells or re-query for it. Signed keys (change, delta, percent) rank
by |key| by default, so drops and spikes both surface; explicit `asc` puts
the most negative first (decliners), `desc` the most positive first. Blank
cells rank last as no-data, never as zero. To keep the ranking in a block,
pin the returned items as an explicit `in {…}` filter — a snapshot of
today's ranking that goes stale as data changes, and say so when you pin
it.

## 3.9 The block signature

A block has one meaning and several renderings: the sentence you say
("Gross Margin % by Region and Product, monthly over 2026"), the **view**,
and the config JSON. The view is the rendering the tools speak;
references/11-block-grammar.md defines it, and the breakdown spelling is
normative in [[build-model:references/grammar-reference.md]], the
generated grammar reference.
Reads return its text as `signature` and its write shape as `view`; the
block survey and the update echoes pair it with a `signature_status`
(`complete` means the view is the whole config; `partial (…)` names what
it cannot describe, references/12-editing-blocks.md §12.6). The write
tools accept the same `view` back. They are their own validation, so call
them directly: a view that does not check returns its problems and
persists nothing; there is no separate validator to run first. Design and
reuse conversations happen in this form, not JSON. Derive it by hand only
when no tool has rendered it for you.

The grain of any cell reads off the signature as the union of its row and
column path dimensions. Two blocks with the same signature up to row/column
placement contain the same numbers (the pivot law), so compare signatures
before building anything new.

## 3.10 Reading incrementally: coarse first, then descend

`inspect_model_views` returns an aligned text grid with windowing knobs. Never
read a big block in one call. Read coarse first, then descend: take the
default window (top of the hierarchy, one level down) and read the footers.
The footers tell you the shape (`rows 4/4 at depth 1 · Engineering has 6
children, Marketing 4`), which is usually the answer to "what is in this
table". Only then narrow to the branch the question is actually about. A
3,000-row block almost never needs 3,000 rows read.

The four knobs compose in this order:

- **scope** — a row path exactly as a previous page printed it
  (`"Opex > Engineering"`). It narrows to that subtree; the scope row
  itself is included with its values. On an unknown scope the tool answers
  with the nearest existing ancestor and its children as copyable tokens.
- **depth** — levels below scope (default 1; 0 is the scope row alone;
  `"all"` for the whole subtree). Provenance rows from `include_inputs`
  never count as a level.
- **after** — the keyset cursor for paging: rows strictly after that path
  in display order. **The cursor law: only pass paths a previous response
  printed** — the footer's `next: pass after "…"` line, verbatim. Never
  invent or edit a path.
- **limit** — rows per page. Omit it and a window that fits (up to 500 rows,
  within the byte budget) comes back whole, no cursor; larger windows page
  50 at a time. Pass a limit only when you want a specific page size — never
  compute one to dodge a second page. The footer names whichever bound
  tripped (`limit reached`, `byte budget hit at N rows`) and prints the
  exact cursor for the next page.

Two further knobs answer "where did this number come from". Both need a plain
single-layer read: neither can ride a time or scenario comparison.

**include_inputs** adds the values that fed each row, as visible `inputs:`
provenance rows.

A `formulas` section reports which rule filled each cell, and which other rules
were in the running. It arrives without asking on any row that came out zero,
blank, or `#ERR` — the rows whose values cannot explain themselves. That is
**formula_origins: "zeros"**, the default. **"all"** extends it to every row,
for a value that is present but wrong; **"off"** skips it.

A variable usually has several rules. The engine tries them in a fixed order
and stops at the first one whose condition holds. None of that is visible in
a grid — every cell just shows a number — so a cell filled by the wrong rule
looks exactly like a cell filled by the right one. This is what the section
shows:

```
formulas    (each cell takes the first candidate whose condition holds)
  f1  Revenue = Bookings * 0.8    range Actuals
  f2  Revenue — no formula covers these cells, engine filled 0
  f3  Revenue = Users * Price

  Revenue    c1-c3 f1 · c4-c6 f2

  candidates
    Revenue  f1 → f2 → f3
```

Read that as: the first three columns took `f1`, the last three fell to `f2`,
and Revenue's rules were tried in the order `f1`, `f2`, `f3`. Because the
engine stops at the first match, `f3` never ran anywhere on this page. If
`f3` is the forecast rule someone wrote, that is the bug — and the grid alone
would only have shown six ordinary-looking numbers.

A line with no formula text on it is the engine standing in where no rule
applied: "engine filled 0", "rolled up from finer cells", "no formula and no
fallback covers these cells". Those are answers too. They say the cell has no
rule rather than the wrong one, which is a different repair.

The candidate list is not always complete: it shows the winning rule and up
to eight runners-up. A variable carrying more rules than that is cut off at
the end, so on a long list, absence is not proof a rule was excluded.

Leave it off for ordinary reads, and narrow the window before turning it on:
the section describes the rows the page draws, so scope to the cell in
question first (references/06-validity.md §6.6).

Pages of one investigation should share one snapshot: when a footer prints
`as_of_point {"change_log_id": …}`, pass it back with the next page's
`after` so a concurrent edit cannot shear the sequence (a stale cursor
answers with a drop-after-or-pin steer, not data).

The grid itself: hierarchy is indentation, values are raw engine numbers,
`·` is no data (blank is not zero, references/02-formulas.md), `#ERR` cells are detailed in an
errors section, and the refs section maps row paths and `c1..cN` column ids
to `runway:` URIs for follow-up tool calls. When a footer says a fan is
large (`fans out to 250 children`), that is a ranked-read question for
`inspect_model_views` `ask.rank` (§3.8b), not something to page through.
