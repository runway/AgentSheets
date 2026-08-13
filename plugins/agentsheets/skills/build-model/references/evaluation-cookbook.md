# Evaluation cookbook

Request patterns for `inspect_variables` `ask.try_formulas`: multi-formula, per-segment analysis
that answers a question without persisting anything. Expression syntax lives in
`references/formula-grammar.md`. Read this when composing an evaluation, not before: the
manual's steps cover the ordinary path.

## The request is the question

A `ask.try_formulas` call is three decisions, and they are the same three decisions every time:

```json
{
  "ask": {
    "try_formulas": {
      "expressions": [{ "name": "margin", "expression": "..." }], // WHAT to compute
      "from": "2025-12-01",
      "to": "2026-06-30", // WHICH periods
      "rows_by": "Customer" // one record per item (optional)
    }
  }
}
```

The blocks below show the `ask.try_formulas` body alone. Each one goes inside
`{"ask": {"try_formulas": ...}}` the same way.

Periods are always the columns, at the model's base grain: one record per period, per
`rows_by` item, per expression — no date field to remember. `from`/`to` choose the periods.
One asymmetry matters: forecast months exist only inside the range (a pin at one outside it
reads 0), while actuals a bracket pins resolve even outside it.

Each result record carries `ephemeral_variable` (the formula's name), `segments` (keyed by
the `rows_by` dimension, with the period under `period`), and `value`. `computed` restates
the grid in one line — check it before reading records. A 0 no formula reached carries a
`note` saying so; a computed 0 does not. Syntax errors fail the call naming the problem;
semantic issues come back as per-record errors. Fix and call again.

In new expressions, author power, not-equal, and equal with the Excel spellings `^`, `<>`,
and `=`.

## Pattern: windowed aggregates + ratio, one record per dimension item

The workhorse shape for "compare each X's current value against its own history". Every
number — windows, baselines, ratios — is a formula; `rows_by` fans them out per item; you
read judgments off the records and compute nothing yourself.

```json
{
  "expressions": [
    {
      "name": "current_total",
      "expression": "sum(Variable[Date.Month = \"2026-06\"])"
    },
    {
      "name": "baseline_median",
      "expression": "median(Variable[Date.Month in {\"2025-12\", \"2026-01\", \"2026-02\", \"2026-03\", \"2026-04\", \"2026-05\"}])"
    },
    {
      "name": "baseline_stdev",
      "expression": "stdev(Variable[Date.Month in {\"2025-12\", \"2026-01\", \"2026-02\", \"2026-03\", \"2026-04\", \"2026-05\"}])"
    },
    {
      "name": "vs_baseline",
      "expression": "if(@baseline_median > 0, @current_total / @baseline_median, NULL)"
    }
  ],
  "from": "2025-12-01",
  "to": "2026-06-30",
  "rows_by": "Customer"
}
```

- The explicit month set pins the comparison window; compose it from known calendar facts at
  request time. With month references, each set member is that month's bucketed total, so
  `median(...)` is the median **monthly** value.
- An unprefixed lookup inherits the record's coordinates: `rows_by: "Customer"` makes every
  expression per-customer with no per-customer spelling. The bracket overrides only what it
  names. That is why one expression is correct for every record.
- `@name` references another entry in `expressions`. Every listed entry is calculated and
  comes back in the results, so an intermediate you do not want to read must be inlined in the
  expression rather than given its own entry.
- Several outputs simply mean several formulas: each gets the same breakdown.

**Restricting to a subset of items**: constrain inside the expressions with a bracket —
`sum(Variable[Customer in {"Acme", "Globex"}])`, spelling items exactly as tool output
prints them — or by a second dimension (`[Type in {"Expense", "Cost of Goods Sold"}]`).
Many items times many formulas can hit `max_rows`; if records come back truncated, drop
formulas you will not read or split the request by a coarser dimension.

## Measure recipes: evidence for trend and anomaly analysis

Composable formulas for richer evidence than a single ratio. `WINDOW` below is an explicit
month set as in the pattern above. `@latest` is the period under test, e.g.
`sum(Variable[Date.Month = "2026-05"])`.

- **Z-score**: how far the latest value sits from its own history, in units of spread:
  `if(@spread > 0, (@latest - @center) / @spread, NULL)`, with `@center` = `average(...)` or
  `median(...)` over WINDOW and `@spread` = `stdev(...)` over WINDOW. Median-centering is
  more robust to one outlier month. With short windows (~6 points) the spread estimate is
  noisy. Treat z as corroborating evidence, not a verdict.
- **Year-over-year**: `@latest` against the same period one cycle earlier (a month literal
  one year back). Use it when the series has an annual rhythm a trailing window cannot see.
- **Trend break**: `average(...)` over the 3 most recent closed months over `average(...)`
  over the 3 before them. It surfaces momentum shifts no single month makes obvious.
- **New extreme**: `if(@latest > max(Variable[Date.Month in WINDOW]), 1, 0)` (or `min`/`<`).
  Crude but interpretable: the first period outside its own historical range.
- **Share of total (mix shift)**: `Variable / sum(Variable[Dimension in any])` per segment.
  It catches a segment quietly taking a larger share while the total looks normal.
- **Volume vs value**: `count(...)` alongside `sum(...)`: counts non-empty values, so it
  distinguishes more transactions from bigger ones.
- **Relationship ratios**: financial anomalies often live in a relationship, not a level
  (margin, payroll per head, spend per unit of output). Define the ratio as its own formula,
  then treat it as the series: `@name` references accept the same bracket constraints as
  variable references, so `stdev(@margin[Date.Month in WINDOW])` scores the ratio's history
  like any variable.

Period-vs-period comparisons are expressions too: `@latest` and `@prior` as two formulas,
delta and percent as two more. Composing them costs four lines and keeps every comparison
window independent, which one table-level offset never could.

## Pattern: how far does the data go (watermark)

Ingested data ends at the last sync, not at today. Before pace math, find the last populated
day. Daily grain is a view read, not a probe — `inspect_model_views` `ask.calculate` with a
daily window:

```json
{
  "ask": {
    "calculate": {
      "view": { "variables": [{ "variable": "Variable" }] },
      "window": {
        "start": "2026-06-01",
        "end": "2026-06-30",
        "granularity": "DAY"
      }
    }
  }
}
```

The last populated day is the data watermark. Anchor "how far through the period" to it.
When staleness matters, compare it with the source coverage reported by the delegated
ingestion session described in `[[integrations]]`.

## Pattern: inspect one segment

To decompose a single dimension item (one account, one region) by a second dimension, pin
the first inside the expressions and put the second in `rows_by`:

```json
{
  "expressions": [
    {
      "name": "seg_total",
      "expression": "sum(Variable[`Dimension A` = \"<item>\"])"
    },
    {
      "name": "seg_count",
      "expression": "count(Variable[`Dimension A` = \"<item>\"])"
    }
  ],
  "from": "2025-12-01",
  "to": "2026-06-30",
  "rows_by": "Dimension B"
}
```

`count` distinguishes one large entry from many small ones. Use the dimension item exactly
as tool output spells it.

## Pattern: before/after a point in time

Make two calls with the same request and a different `as_of_point`. The delta per record is
what changed between the points. Ingested values resolve as of each point, so "before/after
last night's sync" works. Take boundary timestamps from the delegated ingestion session's
reported job evidence. `[[scenario-management]]` covers the wider diff process.
