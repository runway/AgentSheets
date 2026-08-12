# Recipes

Worked table shapes. Each recipe is a sentence — variables by dimensions
over dimensions, plus knobs — and the laws it exercises. Write the
sentence as a view (references/11-block-grammar.md): variables with
breakdowns, id-free. The config-JSON rendering
(references/03-table-blocks.md §3.9) is the escape hatch for what a view
cannot spell — per-item overrides, multi-entry axes, UI-authored
drill-ins — and its field mechanics live on the tool schema, not here.
Recipes are starting points, not the space of valid tables. When a
request fits no recipe, compose a new shape from the laws instead of
forcing it.

## The rule (memorize once)

An axis is one rule, and a view spells it as a breakdown term: which
dimension, the branch it sits under, the items it keeps, the time grain it
buckets (`.Month`), what descends beneath it. Sort is presentation
(`edit_table_blocks` `change.sort`); a dimension can sit on the value axis
as a mapping (`{"dimension": "City"}`, references/11-block-grammar.md
§11.1), and any other role override exists only in `table_config`.

Building or reshaping a table is id-free: you never call
`generate_uuids`, because a view carries no node ids and the write mints
and reconciles every one. Node ids surface only in `table_config` — the
tool schema carries their mechanics, and `inspect_table_blocks`
`ask.config` hands back a block's own ids to thread on a raw-config edit.

## R1 — A variable over time

"Revenue by month." One variable row, the system Date on columns, nothing else:

```json
{ "variables": [{ "variable": "Revenue" }], "breakdown": "[Date.Month]" }
```

with a `window` of 2026-01..2026-12 monthly beside it on the create.

This is the default shape for any "show me X over time": a variable row,
the system Date across the top, monthly.

## R2 — A variable by a dimension, over time (the canonical table)

"Revenue by Region, monthly." The dimension nests _under_ the variable —
`{"variable": "Revenue", "breakdown": "[Region]"}` on R1's frame.

The rendered grid: a Revenue total row recomputed at the {Month} grain (a
cell's grain is its granularity plus its dimensions), one child row per
region ({Region, Month} grain), months across the top.
Deeper breakdowns descend by comma: `"[Region, Product]"` puts products
under each region, each later dimension one level below the previous.
Fork siblings under one variable are almost never what anyone means.

## R3 — The pivot

"Regions across the top instead." Move Region to the other tree: from
Revenue's own `breakdown` to the shared one — `"[Region, Date.Month]"`
or `"[Date.Month, Region]"`, whichever grouping should dominate on
top. Numbers do not
change; only placement does (references/03-table-blocks.md §3.4). Pivot
requests never require rethinking the model.

## R4 — A ratio, done honestly

"Gross margin % by department." Formulas first (references/02-formulas.md):

```
Gross Margin %   =   (Revenue - `Cost of Revenue`) / Revenue
```

Set the variable's aggregation function to do-not-aggregate: a summed ratio
is nonsense, and do-not-aggregate disables time rollup (references/04-time.md §4.4).
A department with zero or null Revenue mints a division-by-zero cell error;
wrap the ratio as ``iferror((Revenue - `Cost of Revenue`) / Revenue, NULL)``
when a blank reads better than an error cell.
Then an R2-shaped table with Gross Margin % as the variable and Department
nested. Expect and explain: the parent row is the ratio recomputed at the
coarser grain, not any combination of the child rows. If the user wants
the total row to read 100%, that is a share-of-total variable
(`Revenue / Revenue$`), a different formula.

## R5 — Specific items only, and one item opened up

"Just Engineering and Sales, and break Engineering down by level."
One breakdown spells both, an item filter and a branch condition:

```formula
[Department in {Engineering, Sales}, Level[Department = "Engineering"]]
```

Sales stays closed and only Engineering opens, because the branch
condition routes Level under that one branch. Filters match exact
strings, and stale values silently drop rows, so copy values from item
enumeration (references/07-modeling-method.md step 3 owns what each
listing carries). Coordinates pinned to a block and generated items appear
in no listing; their authoritative spellings are the computed grid
itself — the write's readback or `inspect_model_views`.

## R6 — Year over year

R1 or R2, plus one comparison: pair each period with the one twelve
grain steps earlier. What a block is compared against is presentation,
so on an existing block it is one `edit_table_blocks` call
(references/12-editing-blocks.md §12.4):

```jsonc
"comparison": { "period_offset": 12, "measures": ["CURRENT", "DELTA", "PERCENT"] }
```

Twelve because the table is monthly; the offset counts grain steps, not
years (references/04-time.md §4.6) — year over year on a quarterly table
is 4. Requires exactly one date axis. Trim the measures to what the user
asked for; the full quadruple (current, comparison value, delta,
percent) quadruples the value columns. A view never carries a
comparison — `edit_table_blocks` owns it on new and old blocks alike.

## R7 — Budget vs live

The budget lives in a scenario or snapshot named e.g. "Budget 2026" (references/05-scenarios-and-comparisons.md).
On the block viewed from the baseline (block writes land in the viewed
scenario, references/05-scenarios-and-comparisons.md §5.2), the sentence is
"compare against Budget 2026, value and variance, side by side" — again
one `edit_table_blocks` call:

```jsonc
"comparison": { "scenarios": ["Budget 2026"], "measures": ["VALUE", "DELTA"] }
```

Comparison is by scenario name; speak names to the user too. Setting a
scenario comparison clears a period one and vice versa — the two never
stack, and the tool keeps that true.
Rows align by identity across scenarios; a row missing in the budget
shows nulls, which is correct, not a bug (references/05-scenarios-and-comparisons.md §5.4).

Recipe numbers are stable anchors; there is no R8.

## R9 — Running total

Formula, then table. On a `Cumulative Revenue` variable:

```
`Cumulative Revenue`[-1] + Revenue
```

A legal self-reference through the time offset (references/02-formulas.md §2.5). Show it with R1's
shape. The same pattern carries balances and cohort rollforwards. A missing
prior period counts as zero, so this starts at the first Revenue month; to
start from a different value, seed one in-window month with `set_values`.
A `count()` guard on the prior period computes but obeys the same
seed-window and anchor laws (the running-balance recipe at the top of
references/limitations.md); the seed states the intent declaratively.

## R10 — Planning what does not exist yet

"Add a Platform team to the headcount plan" when no data has that item.
Add the value to the dimension with `edit_dimensions`
`change: {add_items: {dimension: "Department", values: ["Platform"]}}`; it is
model-wide (references/01-the-dimensional-universe.md §1.4).
Formulas pinned to it (`Department = "Platform"` conditions) then supply
planned values, and the item appears in tables like any other. For date
ranges (plan the next 24 months), set the axis's item generation to
GENERATE over a date range instead. The added value is listed back under
`manual_items`; generated date items are not enumerated at all
(references/07-modeling-method.md).

## R11 — The Actuals/Forecast pair, weekly

The standard regime split (P&L, cash flow), at a granularity where the
frontier rule matters (references/04-time.md §4.5). Write one windowed
default per regime with `change.set_values` and a `period`, by window name: Actuals
("actuals") reads the source, Forecast ("forecast") is the projection. Show it with
R1's shape at a weekly granularity and date range. Weekly Actuals end at
the last Saturday on or before Last close. The week that straddles the
close evaluates entirely as Forecast, so the weekly table will not match
its monthly sibling across the frontier unless the close lands on a
completed week.

## R12 — Headcount

Headcount is a balance-like variable: its aggregation function is
last, so coarser periods show the period-end count, never a sum (references/04-time.md §4.4).
"Reuse before you build" covers prebuilt builders too: a dedicated
headcount build tool creates the whole deliverable, Employee Detail and
Department Rollup blocks with the payroll formulas wired. It resolves the
workspace Last close itself (the system Date dimension's actuals/forecast
boundary, references/04-time.md §4.5) — you pass no id for it. The build reads
whatever is set and fails with a clear message if it is unset, so read and, if
needed, set the actuals cutoff first.

## R13 — The database view (record table)

"Show me the roster." One row per item of an entity dimension with its
attributes and own values beside it — a roster, a price book, a customer
list. The headcount playbook's employee detail table is this recipe, and
a mapping table (references/14-dimension-mappings.md §14.5) is its
one-attribute case.

The entity goes on the shared breakdown, and every attribute is a
value-axis entry — a dimension entry per mapped attribute, a variable
entry for each per-item value. Set `transpose: true` so the entities run
down the rows; a view carrying any variable entry does not transpose
itself:

```json
{
  "view": {
    "variables": [
      { "dimension": "Department" },
      { "dimension": "Job Title" },
      { "variable": "Pay Rate" }
    ],
    "breakdown": "[Employee]",
    "transpose": true
  }
}
```

Time enters as the selected period, never as columns: the window narrows
to the one period the listing reads — the current month, or the latest
close — at `NO_GRANULARITY` (the product shows it as Raw), and each cell
states that item's field at that period. Attributes change over time
(references/04-time.md), so a listing asked to show history re-windows
to the other period; it does not grow month columns. Anything
aggregated — a department total, payroll over the year — is a report
(R2, R12), built beside the listing rather than into it.

## Not a table: the ephemeral probe

For step-4 probes and one-off analysis, skip layout entirely: a probe is
a bounds question, not a table. `inspect_variables` `ask.try_formulas`
(references/02-formulas.md §2.8) takes named formulas, a period window,
and the dimensions that get their own number:

```
ephemeral_variables: [{ name = "margin", expression = "(Revenue - `Cost of Revenue`) / Revenue" }]
from:     2026-01-01
to:       2026-12-31
by:       ["Region", "Date.Month"]
max_rows: 200
```

`from` and `to` are both required — a date dimension generates no members
without a range. `Date.<grain>` in `by` sets the reported grain; `by` never
substitutes for the range. Expressions reference saved variables by
plain name and each other as `@name`. Results come back as `ephemeral_variable`
and resolved `segments` per row; no address parsing needed.
