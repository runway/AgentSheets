# Variable recipes

Use these recipes to build new variables from common requests. They rely on
the rule that a plain variable reference becomes one rolled-up number before
an aggregate sees it (references/02-formulas.md §2.3). Choose the recipe before
the first write to avoid trial and error.

## 13.1 The decision rule

Aggregation answers one question: **what is this variable's value for an
interval, given its values for smaller periods?** The answer chooses a setting:
add the months (SUM); quote the closing month (LAST) or the opening month
(FIRST); quote the extreme (MAX/MIN); tally or test presence (COUNT/ANY);
average the months (AVERAGE, a per-grain override only — see fact 2
below); or go
back to the ingredients and recompute (DO_NOT_AGGREGATE). Flows
(amounts that accumulate during the interval) add. Stocks (states sampled
at an instant: cash, ARR, headcount) have no interval value of their own,
so a boundary value stands in. Derived quantities (ratios, rates,
statistics) have no honest pick or combination of finer values, so each
granularity (monthly vs quarterly; "grain" for short) recomputes them from
the ingredients.

- An **additive quantity** ("revenue", "total spend", "units sold") is a
  source-backed variable with SUM aggregation, usually with no authored
  formula at all: a bare source variable already sums its rows at every
  grain (the bridge law, references/01-the-dimensional-universe.md §1.5).
- A **row-level statistic** ("average deal size", "median order", "largest
  transaction", "how many deals") aggregates the **source column** (or an
  explicit fan-out), never a plain variable reference. A variable
  reference is one already-rolled-up number per cell, so an aggregate
  around it is an identity (references/02-formulas.md §2.3). The column
  reference is the one reference that still sees rows.
- A **derived rate** ("margin %", "revenue per rep", "share of total",
  "growth") is a ratio of variables, DO_NOT_AGGREGATE, recomputed per grain.

Four rules support every recipe:

1. A source-column reference (`<runway:exttables/…/columns/amount/>`) is a
   real row aggregation: the engine aggregates the actual source rows inside
   the cell's slice, at whatever grain the cell is. The URI is already in
   hand: the source variable's default formula reads
   `sum(<runway:exttables/…/columns/amount/>)`, and an `inspect_variables`
   formulas read of that variable returns it. Reuse the URI; change the
   function.
2. The interval answer has two levers, one concept: a default plus
   per-grain exceptions. `aggregation_function` on the `edit_variables`
   create/update operation is the variable-level default across every
   coarse grain (DO_NOT_AGGREGATE, SUM, MAX, MIN, FIRST, LAST, ANY,
   COUNT). `change.set_time_rollup` overrides a single granularity
   (quarter/half-year/year) with SUM, LAST, or AVERAGE — the only place
   AVERAGE exists. There is **no MEDIAN, STDEV, or VAR** at either level;
   for those, a formula is the only construction.
3. Either lever drives the **time-axis rollup only**, through a
   synthesized formula (§4.4's ladder, "Quarter = last of my Months";
   references/02-formulas.md §2.4). It never aggregates across non-time
   dimensions, and its inputs are the variable's own finer-time cells,
   not source rows. A cell your own formula wins is untouched by it: for
   every cell, the most specific matching condition's formula computes it
   (dispatch, §2.2). Set the answer when the variable is created or when
   a view that needs the rollup is built; either works. Both levers survive
   formula saves, each its own way: an explicitly set `aggregation_function`
   is snapshotted and restored when a save re-derives it away (only a failed
   restore warns), while a `set_time_rollup` row is a formula of its own
   that other saves never touch. Source-backed variables have no formula
   saves at all.
4. For non-additive variables the interval question picks the setting. The
   trap is _how_ you apply it. Set it with fact 2's levers, never
   with an authored `[]` formula: the synthesized time rollup is
   an exact condition that outranks `[]` (dispatch, §2.2), so a
   hand-written statistic is masked at every grain and summed back into
   nonsense. A stock's LAST/FIRST also does real work: DO_NOT_AGGREGATE
   re-evaluates the formula at the coarser grain, and a recurrence cannot
   survive that, because no prior period exists there.

## 13.2 Recipes by request

**"average deal size" / "average order value" / "avg X"** —
`edit_variables` create, aggregation_function DO_NOT_AGGREGATE; then
`average(<runway:exttables/…/columns/amount/>)`. NOT `average(Amount)`:
that is an identity over the rollup and displays the sum (§2.9; the
write tool warns). Equivalent ratio when a count variable exists:
``Amount / `Deal Count```, guarded or not. A month with no rows shows
blank, and blank is correct. Never guard an average to 0; 0 reads as "we
closed free deals". Law 1 (§13.3).

**"how many deals" / "count of orders" / "number of transactions"** — new
variable with `count(<…column…>)`, which counts the slice's source rows.
Aggregation SUM is fine: counts are additive. NOT `count(Amount)`: that
displays 1 wherever the slice has a value. The COUNT enum on a
source-backed variable counts its finest **time buckets** with data, which
equals the row count only when each row has its own timestamp; the column
form counts rows. An empty slice counts 0 (count and sum are the two
aggregates that return 0, not blank, on empty). Law 3 (§13.3).

**"largest transaction" / "biggest deal" / "smallest order"** —
`max(<…column…>)` or `min(<…column…>)` on a DO_NOT_AGGREGATE variable: the
true row extreme within each cell's slice. The MIN/MAX enum does
something else: it takes the extreme of the variable's own finer-time
buckets, each already a sum of its rows. That is right for "our best
month this quarter" and wrong for "largest single deal".

**"median order" / "typical deal" / "volatility" / "standard deviation"**
— `median(<…column…>)`, `stdev(<…column…>)`, `var(<…column…>)`. No rollup
method exists for any of these (fact 2), so a formula over the column (or
an explicit fan-out) is the only construction. DO_NOT_AGGREGATE. A
one-row slice has no sample spread: stdev/var show blank there, which is
correct, not a bug.

**"margin %" / "gross margin" / "take rate"** — a ratio of variables:
``(Revenue - `Cost of Revenue`) / Revenue``, DO_NOT_AGGREGATE. Parents
recompute the ratio at their own grain, never a sum or mean of child
ratios. Guard (`if(Revenue = 0, 0, …)`) only when a true 0 should
display; otherwise blank is right. Gallery, §2.9. Law 1.

**"revenue per rep" / "cost per customer" / "X per Y"** — the ratio
`Revenue / Headcount`, DO_NOT_AGGREGATE, guarded if 0 should display. The
parent is total-over-total (the weighted rate), not the mean of per-item
rates; recomputing gives exactly that. When the denominator is "the
number of Ys with data", use `count(Metric[Y in any])` (count counts
non-empty values over a fan-out).

**"share of total" / "% of revenue" / "mix"** — `Revenue / Revenue$`
(§2.9): `$` escapes the slice for the denominator; the plain numerator
keeps each cell its own share. DO_NOT_AGGREGATE.

**"running total" / "cumulative" / "carried forward"** — the guarded
recurrence, §2.5 and the gallery line: copy it, do not re-derive it, at
the grain the user asked for. The guard supplies the starting value (a
missing prior period is null; an unguarded recurrence never starts). No
rollup edit belongs in this build: the levers only combine finer-time
cells, and same-grain cells have nothing finer. One caveat: a view
coarser than the guarded grain re-evaluates the recurrence there without
a starting value, unless the variable's rollup says otherwise — set the
interval answer per §13.1.

**"opening/closing balance" / "value at month end"** — aggregation_function
LAST (FIRST for opening): each coarser time cell becomes the last finer
bucket. Point-in-time levels (cash, headcount, ARR) want LAST, no formula
needed. Law 2 does not apply: a LAST parent equals one child, by design.

**"MoM/QoQ/YoY growth"** — two different asks; pick by the sentence.
"Show the change versus last quarter" on an existing table is an
`edit_table_blocks` comparison, not a variable (references/12-editing-blocks.md §12.4). A
saved growth METRIC is `(Revenue - Revenue[-1]) / Revenue[-1]`,
guarded, DO_NOT_AGGREGATE. `[-1]` counts in the viewing grain; that is
what makes one formula MoM on monthly views and QoQ on quarterly. This is
an offset LOOKUP, not a chain (references/02-formulas.md §2.5): it reads
another variable's prior cell, so it needs no starting value. The first period's growth is blank
because its prior is missing, and that is correct, not a defect to patch.

**the user's name for a variable that only exists as a source column** — the
ask says Revenue; the source has only `amount`. Create the named variable
instead of shipping the raw column: rename the variable, or write the
pass-through `Revenue = amount` and place `Revenue` on the table. The row
label is the address every reader and every later formula uses; the right
numbers under the wrong name answer a different question. "There is no
Revenue variable" is the signal to create one, not to substitute the
column that happens to hold the values.

**"prior period" / "same month last year"** — `Revenue[-n]` for
grain-relative meaning;
`Revenue[Date.Month = dateadd(this.Date.Month, -12, "month")]` to pin
the calendar meaning across re-graining. §2.9 covers the trap and the
choice. Choose deliberately, not by habit. And when the ask is a
comparison shown beside each period (value, comparison, variance), do
neither: set the block's time comparison (references/04-time.md §4.6),
offset 12 on a monthly table, 4 quarterly, 1 yearly. Comparison-measure
cells exist only when the block carries a time comparison; a hand-built
"Last Year" variable never fills them.

**"NRR" / "retention by cohort"** — three decisions, then one formula.

1. **The cohort dimension's data type.** If the signup cohort is stored
   as DATETIME (its values are dates like "2025-01"), a `mrr > cohort`
   breakdown comes back empty unless the block's date window covers the
   cohort's own dates (references/limitations.md §6 has the window
   workaround). The recipe below is how to build it once the cohort is a
   categorical dimension; for a DATETIME-typed cohort, widen the window
   to cover the signup dates and warn the user about the extra empty rows
   and wider columns.
2. **The time binding.** Bind the transaction date as the source
   variable's preferred time, so the `month` column drives `mrr` on the
   Date axis (the preferred-time binding; the "Preferred time binding for
   imported variables" section of
   [[build-model:references/formula-grammar.md]] teaches it). A variable
   with no preferred time cannot sit on the Date axis at all.
3. **The cohort's placement.** Keep the signup month (`cohort`) a plain
   dimension on the rows: it labels each row with one cohort and is never
   the Date axis, even when its values are date-shaped (`2025-01`). Two
   spelling rules (references/01-the-dimensional-universe.md §1.5):
   address the cohort by its own stored items with no granularity suffix,
   and filter with literal stored spellings
   (`cohort in {2025-01, 2025-02, 2025-03}`), never datetime literals.
   Writing `cohort.Month`, or letting the block treat the key as time,
   hands its rows to the date window: the labels come back as the report
   months and every cell reads zero, because no stored cohort value
   matches a generated report month.

Then the formula. Make the ratio DO_NOT_AGGREGATE, like every
ratio in this file, and write one subset rule per regime. The bare
`[Date.Month in any]` claims the Month shape wherever it appears
(references/09-the-layer-model.md §9.2): the blended parent's own month row
and every cohort row, present and future — cohort goes unnamed, so its slot
stays open at richer grains. Naming the month grain is also what keeps
`[-12]` stepping by month. Two items, same condition, one per regime — a
Date-term rule with no `period` governs actuals only
(references/10-deviations.md D2), so a single unperioded rule would show
history and exactly 0 for every forecast month:

```
set_values(items: [
  {variable = "NRR", condition = "[Date.Month in any]", period = "actuals",
   expression = "if(mrr[-12] > 0, mrr / mrr[-12], NULL)"},
  {variable = "NRR", condition = "[Date.Month in any]", period = "forecast",
   expression = "if(mrr[-12] > 0, mrr / mrr[-12], NULL)"}
])
```

Relative `mrr` inherits whatever cohort the current row slices to, and
`[-12]` reaches that same cohort twelve months back through the
preferred-time projection. Write the reach-back this way, not as a stack
of pinned absolute months (`mrr[Date.Month = "2025-03"]`, one per period):
the relative form is a single formula that holds at every grain and year,
while the pinned stack is a dozen brittle copies. Both forms need
decision 2's binding set first.

What the parent means: the same formula gets the all-cohorts parent right
mechanically — at the parent, `mrr` pools across cohorts before dividing,
so the blended line is pooled over pooled, not an average of the
per-cohort ratios; those differ whenever cohorts are unequal. One caution
on the label: the pooled numerator includes cohorts younger than twelve
months, so the parent is total-MRR growth. If the user means retention
strictly (new cohorts excluded), constrain the numerator to the cohorts
the denominator sees, and say which one you built.

When the ask also wants the same month last year shown beside each
period, set the block's time comparison and leave it in the build (the
"same month last year" note above), rather than a separate `MRR last year`
variable row: a hand-built row cannot fill the comparison cells, and a second
variable on the same crossing is rejected.

**"average of the monthly averages"** — ambiguous; pick a reading before
writing. (a) The true row average over the longer period is the
`average(<…column…>)` variable recomputed at the coarser grain
(DO_NOT_AGGREGATE, no override): a yearly cell averages ALL the year's
rows. (b) The literal mean of the monthly values is a `change.set_time_rollup`
override on that same variable, method AVERAGE at the asked granularity:
the coarse cell averages the finer cells, not the rows. They differ
whenever the buckets hold unequal row counts. Ask, or state which one you
built.

**"distinct customers" / "unique products"** — when the entity is a
dimension (string source columns sync as dimensions,
references/01-the-dimensional-universe.md §1.3):
`count(Revenue[Customer in any])` counts the items with a non-empty value
in the slice. A distinct count over values that are **not** a dimension
cannot be expressed; there is no distinct(). Say so instead of
approximating silently.

## 13.3 Check the readback

After any variable write, check the readback (references/07-modeling-method.md
Step 7) against the aggregation's shape. Each law takes one glance:

1. An **average** parent sits WITHIN its children's range — never above the
   largest child or below the smallest.
2. A **sum** parent is ≥ its largest child (positive values).
3. A **count** parent is a count — never 1 while many children are visible.

The production failure this file exists for: an "average" parent of
24,000 above children ranging 10–14k. Law 1 broken, visible in one
glance, no recomputation needed. A violated law means the construction is
one of §13.2's anti-patterns. Go back to the recipe and rewrite the
construction; do not patch the number.

### Symptom index

When a formula variable's values are wrong, find the symptom below and
check its listed cause first. The rollup levers are only ever a
coarser-grain suspect (§13.1 fact 3), never a cell your formula wins.

| Symptom                                                        | Check first                                                                                                                                                                       |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zero from the first period of a recurrence                     | the starting value — is the seed on a month inside the table's window? (references/limitations.md §10)                                                                            |
| Wrong at the grain you built the formula for                   | the formula, its references, or the data — never aggregation                                                                                                                      |
| Wrong by a huge factor yet saved as valid                      | a date predicate enumerating raw stored dates, not buckets                                                                                                                        |
| Wrong only at a coarser grain                                  | the rollup: the aggregation_function default or a change.set_time_rollup override                                                                                                 |
| Every cell blank or zero right after an aggregation change     | the stored setting, not your formula — a sync can revert an explicit aggregation function, so read it back (references/limitations.md §1)                                         |
| Only the first month shows a value; every later month is blank | the variable builds on its own previous value and its aggregation is LAST — the synthesized rollup masks the chain (§13.1 fact 4). Set DO_NOT_AGGREGATE; do not touch the formula |
| Wrong (no error) only at a coarser grain of a recurrence       | a missing `set_time_rollup` anchor for that grain (references/limitations.md §5)                                                                                                  |

If a symptom matches an entry in `references/limitations.md`, stop trying
to fix it. Do what the entry says instead, and tell the user.

## 13.4 When the write tool warns

A formula write returns a top-level `warnings` list covering six families.
The write still succeeds; each warning names a shape with a known fix, so
route by family:

- **A non-sum aggregate over a no-fan-out reference** — the identity-over-
  rollup anti-pattern. Match the ask to §13.2 and aggregate the source
  column, not the variable.
- **An unfloored recurrence** (nothing supplies the first value) — seed it:
  the running-balance recipe at the top of references/limitations.md.
- **A recurrence started from a pinned date** — fine monthly, wrong at
  coarser grains. The warning's suggested `count()` guard obeys the same
  laws as a seed; either way, add the `set_time_rollup` LAST anchor
  (references/limitations.md §5).
- **Date logic on an uploaded date column** — uploaded date columns are not
  the Date axis: compared directly, the column's dates meet generated
  report months, match none, and read zero. Compare against system `Date`
  period bounds, keeping the column inside an aggregate only for
  row-level proration.
- **A dated source collapsed with none of its date fields** — classify
  first: a timeless per-record conversion is fine as written; a periodic
  total needs the per-record overlap construction the warning spells out.
- **An aggregation-restore failure** (§13.1 fact 3) — re-assert the named
  setting with `edit_variables`.

Rewrite the construction, and only then report done. Never report a
warned write as done on the strength of its `applied` line.
