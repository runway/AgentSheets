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
      "ephemeral_variables": [{ "name": "margin", "expression": "..." }], // WHAT to compute
      "from": "2025-12-01",
      "to": "2026-06-30", // WHICH periods are in scope
      "by": ["Customer", "Date.Month"] // WHAT gets its own number
    }
  }
}
```

The blocks below show the `ask.try_formulas` body alone. Each one goes inside
`{"ask": {"try_formulas": ...}}` the same way.

`by` and `from`/`to` answer different questions and neither substitutes for the other. `by`
decides which numbers come back: add `Date.Month` for one per month, leave every date entry
out to fold the periods together, add a dimension for one per item. `from`/`to` decide what
is in scope at all — they are required because the engine has no default: without a range no
period exists to compute over, even when `by` names no date.

One rule prevents most empty results: **`from`/`to` must cover every period any expression
references**, not just the periods you want to read. A ratio comparing June against a
December–May baseline needs `from: "2025-12-01"`, even if you only read June. If cells that
should have history come back empty, widen the range first.

Each result record carries `ephemeral_variable` (the formula's name), `segments` (the resolved
coordinates, keyed by the names you used in `by`, with the period under `period`), and
`value`. Read identity from those fields. Evaluation doubles as validation: syntax errors
fail the call naming the problem, and semantic issues come back as per-record errors. Fix
and call again.

In new expressions, author power, not-equal, and equal with the Excel spellings `^`, `<>`,
and `=`.

## Pattern: windowed aggregates + ratio, one record per dimension item

The workhorse shape for "compare each X's current value against its own history". Every
number — windows, baselines, ratios — is a formula; `by` fans them out per item; you read
judgments off the records and compute nothing yourself.

```json
{
  "ephemeral_variables": [
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
  "by": ["Customer"]
}
```

- The explicit month set pins the comparison window; compose it from known calendar facts at
  request time. With month references, each set member is that month's bucketed total, so
  `median(...)` is the median **monthly** value.
- An unprefixed lookup inherits the record's coordinates: `by: ["Customer"]` makes every
  expression per-customer with no per-customer spelling. The bracket overrides only what it
  names. That is why one expression is correct for every record.
- `@name` references another entry in `ephemeral_variables`. Every listed entry is calculated and
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
day:

```json
{
  "ephemeral_variables": [
    { "name": "daily_total", "expression": "sum(Variable)" }
  ],
  "from": "2026-06-01",
  "to": "2026-06-30",
  "by": ["Date.Day"]
}
```

The last record with a value is the data watermark. Anchor "how far through the period" to
it. When staleness matters, compare it with the source coverage reported by the delegated
ingestion session described in `[[integrations]]`.

## Pattern: inspect one segment

To decompose a single dimension item (one account, one region) by a second dimension, pin
the first inside the expressions and put the second in `by`:

```json
{
  "ephemeral_variables": [
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
  "by": ["Dimension B", "Date.Month"]
}
```

`count` distinguishes one large entry from many small ones. Use the dimension item exactly
as tool output spells it.

## Pattern: before/after a point in time

Make two calls with the same request and a different `as_of_point`. The delta per record is
what changed between the points. Ingested values resolve as of each point, so "before/after
last night's sync" works. Take boundary timestamps from the delegated ingestion session's
reported job evidence. `[[scenario-management]]` covers the wider diff process.
