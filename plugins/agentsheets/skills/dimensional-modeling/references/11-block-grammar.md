# The view: table structure as a declaration

A block has one meaning and several renderings (SKILL.md's "one meaning,
many renderings"): the sentence you say, the config JSON, and the **view**
this file defines — the declaration of what the table computes. This file
builds on references/03-table-blocks.md; the pivot law and the crossing
law reappear here, enforced by the tools.

## 11.1 The declaration

A view is variables plus breakdowns:

```json
{
  "variables": [
    {
      "variable": "Revenue",
      "breakdown": "[Department, {Level[Department = \"Engineering\"], Owner}]"
    },
    { "variable": "Headcount" }
  ],
  "breakdown": "[Date.Month]"
}
```

Each `variables[]` entry is one row stack, carrying EXACTLY ONE of two
things. `variable` is the ordinary case: a readable name, resolved like any
formula reference, optionally with its own `breakdown`. `dimension` instead
places a dimension on the value axis as a **mapping**, where each cell shows
that dimension's item for its row and column rather than a measured value:

```json
{ "variables": [{ "dimension": "City" }], "breakdown": "[Name]" }
```

One City per Name. A mapping shows one item per cell, so it takes no
`breakdown` of its own and no grain; naming a variable under `dimension`, or
a dimension under `variable`, is refused either way rather than quietly
reinterpreted. The top-level `breakdown` is shared by every entry and becomes
the columns. Breakdown spelling — comma descends, braces fork, branch
conditions, `in {…}` filters, `.Month` grains — is normative in the
`Table views: breakdowns` section of
[[build-model:references/grammar-reference.md]], where every example
parses against the real grammar. The example below routes a term under
one branch of an ancestor — a branch condition — and branches need not
match: Engineering breaks down by Level, Operations by Owner, and every
other department stays closed.

```formula
[Department, {Level[Department = "Engineering"], Owner[Department = "Operations"]}]
```

## 11.2 The rendering the tools speak

Reads return the view two ways: `view`, the exact shape a write takes, and
`signature`, its text — one line per entry with its breakdown (a mapping
reads `City (mapping)`), a `by` line for the shared one, then `# kind`
comment lines naming anything the view cannot express. Edit the view a read hands back and hand it back.
`inspect_table_blocks` `ask.list` surveys every table in one read (the block
dictionary): each row's `view` and `signature` for what the table computes,
and `presentation` beside them for how it shows. Its `window`, `comparison`
and `sort` are the words `edit_table_blocks` takes, so they edit straight
back; `transposed` states orientation instead, because `change.transpose`
flips rather than sets (§11.3). Every other read describes a table the same
way, page reads included.

Writes go through `edit_model_views` `change.configure_table`: each `tables[]`
entry carries a `view`, and entries apply
independently. A call also performs one operation — it creates, or updates,
or carries one copy — and mixing operations (or batching copies) is
rejected whole.
Naming an existing `table` reconciles the view onto that
block; omitting `table` creates a new one, `page` places it, `window`
(a typed field beside the view, only on create) gives it its date range,
and `copy_from` names a source block. A copy carries the formula lane,
surviving drill-ins, the source's window (the copy path refuses a
`window` param — the source's is kept), its comparison configuration,
and the whole settings bag (column widths included); §11.4 carries the
node-id law a copy runs under. Checking
runs in two layers: view-grammar and model errors reject before anything
persists on every write, and the config checks (the missing-time-column
warning and its kin) ride a create's own result as advisory warnings. On an
update, `dry_run` (at the top level of the call, beside `change`) is how you
see them first — it runs the same checks without persisting
(references/06-validity.md §6.1). Structure a view cannot describe is not agent-writable at all (§11.5).

The pivot law carries over verbatim: two blocks with the same view up to
row/column placement hold the same numbers. The default placement bands the
variables down the rows and runs the shared breakdown across the columns;
`transpose: true` beside the view (or `change.transpose` on
`edit_table_blocks`) swaps the axes whole, which is how "variables across
the top" is built. A variable inside a breakdown is refused; a dimension in
the variables list is §11.1's mapping form, not an error.
Views are therefore the unit of reuse. Compare signatures before building anything new, and prefer
re-declaring a block whose signature already covers the request.

## 11.3 A view says what the table computes — nothing else

This is the law the writes are built on. Window, comparison, sort, column
widths, visibility, the formula lane: none of it is part of a view, so a
view-shaped write **leaves all of it exactly as it was**. There is no line
to restate and no omission that clears — reshaping a table never drops
its window or its comparison. The one exception is orientation, which a
view does not carry: an update that omits `transpose` flips a transposed
non-mapping block back to the default placement, and the swapped axes take
their node ids with them. Restate `transpose: true` on every update to a
block you transposed. To change presentation, use one batched
`edit_table_blocks` call (references/12-editing-blocks.md §12.3 owns the
aspect list). Ranked and top-N
questions are `inspect_model_views` `ask.rank` queries — asked, never saved.

Names resolve like readable formulas: bare words, backticks for spaces
and punctuation, `#hex` to split duplicates, `Date` always the system
Date dimension. Items in conditions and filters are quoted strings;
item text is stored raw, exact string match.

## 11.4 The round-trip law

For every block a view fully describes, the view a read returns is the
whole block, and writing it back reproduces the block byte for byte. A
signature is never a lossy summary — it is the block, in a rendering you
can edit and return. Orientation is the one blind spot: the `view` a read
returns carries no transpose field, so a transposed non-mapping block reads
back untwisted, with residue, and the round trip is not byte for byte there.
Read the orientation from `presentation.transposed` and carry
`transpose: true` yourself on that block's updates (§11.3). What the view
does not carry — node ids, aggregation
functions, sort, the formula lane, surviving segment drill-ins,
presentation state — is reconciled from the block itself: structurally
matched rows and columns keep their axis ids (drill-in anchors and
overrides stay valid) and everything that rides with them. A match is
exact on three things: the side (row vs column), the chain of properties
from the root down to the node, and the node's own drill-in coordinates
— reordering siblings is safe (order never enters the key), but changing
any property in the chain, or a drill step, makes a different node that
matches nothing and starts fresh. A dropped row
takes its settings with it, reported as an explicit warning, never a
silent loss. `copy_from` runs the same reconciliation against its source,
then mints every node id fresh — a copy never shares axis identity with
its source.

To edit a block, hand a view to `change.configure_table` naming the table.
The write reconciles against the block's current config, keeping everything
the view does not describe.

Author time yourself: a table of native variables carries a Date breakdown
(`"breakdown": "[Date.Month]"`) that you write in, with a real window on
the create. An empty readback on a fallback Date axis is a window to fix,
never a Date axis to remove. The fallback: a write that lands
native-variable rows on a block with no other column axis — a create, or
an update to a still-blank block — appends a bare root Date column, and
the block's window drives what it shows:

- the axis renders at the window's grain;
- `NO_GRANULARITY` mints no calendar periods, so the axis shows only
  dates the data itself carries and renders none for formula-only values;
- a missing range falls back to the product's default span.

Leave the breakdown off only for a deliberately time-less shape — a mapping
table, a database view, or an assumptions list the user asked for as such
(references/08-recipes.md R13 owns the database-view shape).

## 11.5 The boundary

Structure a view cannot describe is not agent-writable; a read
names it in the signature's `# kind` lines: generated axes (`generate`),
role overrides other than §11.1's mapping form, multi-property
(flattened) axes, freeform axes, per-item overrides, drill-in paths
beyond a branch condition, segment drill-ins, and the formula lane. A
write to a block holding such structure is refused with a steer rather
than silently rebuilt. Everything else — nearly every block the product
creates — fits inside a view.
