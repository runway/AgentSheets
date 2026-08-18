# Why the model behaves this way

This file explains the model by comparing it with a spreadsheet. Read it when a
result seems wrong before repeating a correct write. Repeating the write will
not change these rules.

## Spreadsheets store position, not meaning

A revenue model in a spreadsheet: countries and segments nested down the side,
months across the top. You read a highlighted cell instantly as "USA Enterprise
revenue, May 2026: 1,240." The spreadsheet knows none of that. Its entire address
for the cell is row 3, column 6. The country, the segment, the month, even the
fact that it is revenue, live in other cells as display text; the connection
exists only in the reader's head. Position creates meaning.

The value 1,240 needs five coordinates: metric, country, segment, month, and
scenario. A grid answers two, a sheet tab adds one, and the rest are nested in
rows. This is why inserting a country rebuilds the layout, why "what was USA Enterprise
revenue in May" has no machine-answerable form against a sheet, and why three
scenario tabs mean three copies of every formula.

## Model values use named coordinates

The model stores **variables** (Revenue, Payroll,
Headcount), **dimensions** (Department, Region; the ways a variable slices), and
**dimension items** (Engineering, Sales are items of Department). A value does
not live in a box; it lives at an address:

```
Payroll$[Department = "Engineering", Region = "SF", Date.Month = "2026-01"] = 22,000
```

Three consequences, each the reversal of a spreadsheet fact:

- **Order carries nothing.** `$[Department = "Eng", Region = "SF"]` and
  `$[Region = "SF", Department = "Eng"]` are the same set of coordinates, so the
  same cell. In a spreadsheet, order is the address.
- **Each variable has exactly the axes it needs.** Revenue by Country and
  Segment; Headcount by Department alone; a tax-rate assumption by nothing.
- **Tables are views, not stored data.** A table puts dimensions on rows and
  Date on columns. Two tables using the same coordinates show the same numbers
  (the pivot law). Drilling reveals existing coordinates. Deleting a table does
  not delete values.

Scenarios are a coordinate too. What a spreadsheet fakes with copied tabs is one
model here with a scenario axis, so a formula fix lands in every scenario at
once.

## Formulas are rules for coordinate sets

In a spreadsheet, "payroll grows 2% a month" is one formula pasted into 72 cells,
each copy free to drift. Here it is one rule at a partial address:

```
Payroll[Date.Month in any] = Payroll[-1] * 1.02
```

`in any` leaves a slot open: every month. A more specific rule can sit on top:

```
Payroll[Department = "Engineering", Date.Month in any] = Payroll[-1] * 1.05
```

**The most specific matching formula owns each cell; whatever it does not claim
falls through to broader rules.** Engineering grows at 5%, everyone else inherits
2%. This is the whole authoring model: describe the business in a handful of
rules and let the grid, however large, be those rules evaluated at every
coordinate.

The broadest rule of all, the one with no restriction at all, covering the whole
variable, is the **default**. Two engine behaviors attach to it and to nothing
else:

- **It covers rows that do not exist yet.** A rule that names Eng, Sales, and
  Support goes blank the day the data grows a Facilities department. The default
  needs no edit: it matches whatever exists at calculation time. So build models
  as one default plus scoped exceptions, never as one rule per segment. A
  per-segment model silently becomes incomplete as data grows.
- **It is where parent-row math comes from.** In a Spend table, the bold parent
  row shows 89,000 above Eng 40,000, Sales 30,000, Support 10,000, Facilities
  9,000. Why a sum? Every variable carries an aggregation function, and the
  system infers it from the default's expression when the default is saved: a
  default of `sum(Amount)` sets SUM; any other shape, even a plain `0`, sets
  "do not aggregate," which blanks the parent row. Leaf cells right but parents
  blank almost always traces here.

## Time supports previous and next periods

Department has members, not an order; there is no "the department after Eng."
Time has a next and a previous, uniform steps, and coarser grains that nest
(month into quarter into year). The system models it as the built-in **Date**
dimension with a grain, and every variable is a time series at some grain.

So time supports both operations: pin (`Date.Month = "2026-01"`) and step
(`Payroll[-1]`, the previous period at the grain being evaluated). Reach back
**relatively** wherever possible: `mrr[-12]` is month-over-year in one formula,
correct at every grain and in every year. Twelve hardcoded
`mrr[Date.Month = "2025-01"]`-style formulas cover the same ground brittlely and
die outside the modeled year.

Time also carries the model's one **when**. The timeline splits into regimes —
actuals through the org's last close, forecast after it, custom ranges where
defined — and the Date term decides which one a rule lives in. A rule with no
Date term at all spans every regime. A rule whose Date term matches more than
one date but names no `period` is not all-time: the engine confines it to
actuals, so the two growth rules above, saved exactly as displayed, would
govern no forecast month — cells read 0, the write reports success, nothing
errors. A projection therefore ships with `period: "forecast"`, and `grain`
alone means "actuals only, at this grain".

Two facts about time and data that save hours:

- The one shared calendar is the system Date axis. An imported metric is not
  automatically on it: the metric's numbers sit next to their own date column,
  and nothing assumes that column is "when this number happened." The
  **preferred-time binding** (one `edit_variables` update pointing the metric
  at its own date column) is what projects the calendar onto it. Unbound, the
  metric reads back empty on any Date axis.
- A named metric is a sliceable grid; a raw source column is pre-aggregation
  rows. Slice and reference by named metrics. Reach for the raw column only for
  true row statistics (average, median, count), because `average(Revenue)` on
  the named metric averages one already-summed number.

## Two common failure cases

Check these two cases before rebuilding or rereading the same result:

**Grain-locked formula.** A `$[…]` rule runs only where its dimension set
exactly equals the cell's. On a table sliced by Department and Region, an exact
formula naming only `Department = "Eng"` stores fine, validates fine, and fills
nothing. No error anywhere. Use `[…]` instead when the rule should follow every
grain containing Department.

For coordinate writes, `change.set_values` with a `block` reads the block,
completes missing dimensions, and reports what it filled in — and stores the
completed address as an exact `$[…]`, so a coordinate write is always a
deliberate grain claim; the unsigiled shape is authorable only through
`condition`. Where the block gives no single answer (the variable placed twice
at different shapes, or a date the tool will not guess), it refuses with
directions instead of writing something plausible and wrong.

**Default zero.** A variable with no default shows real history up to the
org's last-close date and exactly 0 after it. That zero is the engine's
deliberate stand-in ("no forecast logic exists yet"), not a data bug. The fix is
writing forecast logic, not hunting the data. The stand-in also passes source
history through, and it switches off entirely the moment any default exists.
That is why a placeholder default of `0` is the worst formula in the system: it
zeros the forecast, suppresses the history passthrough, and blanks parent rows,
all at once.

## State the intended scope

Name the intent with `segments` and the tool stores the completed coordinates
as an exact `$[…]` — every bounds write claims exactly the grain it names.
Author `condition` when its grammar carries meaning the bounds cannot: a
richer predicate, or the unsigiled subset shape that keeps applying under
added drill-ins.

| Intent                                                          | What you send                                                    |
| --------------------------------------------------------------- | ---------------------------------------------------------------- |
| a value at exactly these coordinates (stores `$[…]`)            | `segments: {"Department": "Eng"}`                                |
| a rule about named dimensions that survives added drill-ins     | `condition: "[Department in any]"`                               |
| a deliberate grain-lock or an address pasted back from a read   | `condition: "$[Department = \"Eng\"]"`                           |
| the default: the rule everything falls back to                  | no bounds at all                                                 |
| a default only inside a time window (actuals, forecast, custom) | `period: "forecast"`                                             |
| how coarser grains summarize the base grain                     | `change.set_time_rollup`, which takes a method not an expression |

A new variable's condition follows the expression's time semantics.
`COGS[] = sum(ISAmount[AccountName in {"Materials", "Freight"}])` has no time
manipulation, so the global `[]` formula correctly re-runs at every grain.
`Cash[Date.Month in any] = Cash[-1] + NetCashFlow` names the monthly Date grain
because the expression steps through time. One caveat: an expression over a
stock input makes the coarse grains ambiguous. `Headcount * 10000` manipulates
no dates, but a quarter can mean two things:

1. **A run rate** — closing headcount × 10000. The global `[]` rule is already
   right: it evaluates at the quarter, reading the stock at its closing month.
2. **A cost to be totaled** — the sum of each month's product. The rule must
   name the Date grain (`[Date.Month in any]`) so months compute and coarser
   cells sum them.

Ask which the user means; the expression cannot say.

Every write answers in the same readable grammar: `bounds`,
`completed_from_block`, `written`, `regime`, `uncovered_block_grains`,
`created`, rejections. What each field
means and what to do about it is references/saving-formulas.md Step 3 ("Read
the response, not just the status").

After writing, read your work back: `inspect_model_views` takes a view, so
you can look at exactly the slice you just wrote and check the numbers.

## Troubleshooting table

Symptom first, because that is the direction troubleshooting runs:

| What the table shows                                     | What is happening                                                                                                                                | The fix                                                                                                                                          |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Real values through last close, then 0 every month after | No rule governs the forecast: none exists (the deliberate stand-in 0), or the rule carries a Date term with no `period`, confining it to actuals | a `period: "forecast"` item                                                                                                                      |
| An imported metric shows 0 everywhere, history included  | A placeholder default is suppressing the source passthrough                                                                                      | an unbounded `sum(<source column>)` and real forecast logic                                                                                      |
| Leaf cells right, parent row blank or wrong              | Aggregation was never established; only saving the default sets it                                                                               | Re-save the default as one aggregation call, or set aggregation on the variable directly                                                         |
| A new dimension value appears with blank cells           | Every rule is segment-scoped; nothing matches the new value                                                                                      | an unbounded rule as the base; keep scoped ones as exceptions                                                                                    |
| A rule you wrote fills nothing, no error                 | Its dimensions do not exactly match the cells (the grain-locked formula)                                                                         | `segments` + the `block` (stores the completed exact address) — or an unsigiled `condition` when it must cover every grain — then read `written` |
| An imported metric reads empty on any time axis          | No preferred-time binding; the metric is not on the calendar                                                                                     | One `edit_variables` update binding it to its own date column                                                                                    |
| "Everything grows 5% except Eng"                         | One formula per segment misses future segments                                                                                                   | an unbounded 5% rule + one `segments` exception; specificity does the rest                                                                       |

The pattern under every row: **write the default first, layer exceptions on
top**, and when a symptom looks impossible, check the walls before spiraling on
verification.

Three of these rows look the same in a grid: the deliberate 0, the suppressed
passthrough, and the grain-locked formula all show zeros and no error. What separates
them is which rule the cell took, and that can be read instead of guessed:
every read names the rule behind a zeroed cell — or says that no rule applied
and the engine filled it. That is what the `formulas` section reports, and it
rides on any read carrying a quiet row; `formula_origins: "all"` extends it to
rows that came out with values.
