# Formula grammar

Expression semantics and the patterns worth reaching for. The syntax inventory
itself lives in `grammar-reference.md` beside this file: the grammar productions,
entry-point shapes, reference forms, operators and precedence, literals, and every
function signature. That file is generated from the grammar and verified against
the parser, so it is the authority whenever the two disagree.

## Variable or Dimension references

Formulas reference variables and dimensions by readable name rather than storage URIs.

| Syntax                          | Meaning                                                                                             |
| ------------------------------- | --------------------------------------------------------------------------------------------------- |
| `Revenue`                       | Value from the current segment, shorthand for `this.Revenue`                                        |
| `this.Revenue`                  | Explicit value from the current segment                                                             |
| `Revenue[]`                     | Value from the current segment, equivalent to `this.Revenue`                                        |
| `Revenue$`                      | Absolute value that does not inherit the current segment; the `$` is written after the name         |
| `Revenue[Region = "West"]`      | Current-segment value with `Region` set to a specific value                                         |
| `Revenue$[Region = "West"]`     | Absolute lookup with only the listed dimension predicates                                           |
| `Revenue[Region in any]`        | Current-segment lookup across all `Region` items; aggregate this lookup                             |
| `` `Net Revenue` ``             | Backtick-delimited name for spaces, punctuation, operators, or reserved words                       |
| `Revenue#a3f`                   | Disambiguated variable or dimension name when multiple variables and dimensions are named `Revenue` |
| `Date`                          | Date dimension at the formula's date grain                                                          |
| `Date.Month`                    | Date dimension pinned to month granularity                                                          |
| `` `Fiscal Date`#a3f.Quarter `` | Backticks, disambiguator, and granularity together                                                  |

A **formula condition** is a bracketed group and never appears inside an
expression: `[]` matches every segmentation, `[Dim in any]` any segmentation
containing that dimension, `$[…]` exactly the named set, and `$[]` only the
empty segmentation.

Ext-table references are still URI-shaped because they point at external table
columns, not model variables and dimensions:

```formula
<runway:exttables/UUID/columns/ColumnName/>
```

External-table references are plain only: no `$` sigil, and no brackets of
any kind.

## Dot notation

Access a variable value or dimension item from the current segment context:

```formula
Revenue
```

`Revenue`, `this.Revenue`, and `Revenue[]` are interchangeable
current segment syntax outside `where` filters; `Revenue` is the conventional
short form. Inside a `where` filter, `this.<VariableOrDimension>` reads the current
cell's segment value.

Unprefixed bracket predicates inherit the current segment and override the
named dimensions. Use `Revenue$[Region = "West"]` when the lookup should
include only the listed predicates.

Use `Revenue$` or `Revenue$[...]` only for an absolute lookup that should not
inherit the current segment.

## Segment lookups

Combine multiple dimension predicates in a single lookup:

```formula
Revenue[Region = "NYC", Department = "Sales"]
```

Set membership -- match multiple dimension items:

```formula
Revenue[Department in {"Eng", "Product", "Design"}]
```

The expression above is treated as a `SUM` if no aggregation function is
specified.

Negated set membership -- exclude specific dimension items:

```formula
Revenue[Department not in {"Eng"}]
```

Where predicate -- filter by an expression:

```formula
Revenue[Department where Headcount > 0]
```

Plain variable or dimension names inside a `where` filter are evaluated against each
candidate row for the dimension being filtered. In the example above,
`Headcount` means the candidate row's `Headcount`.

When a `where` filter must compare a candidate value to the current cell's
segment value, use `this.<VariableOrDimension>` for the current value:

```formula
Revenue[Department where Headcount > this.Headcount]
```

Do not write `Headcount > Headcount` for that comparison; both names would refer
to candidate-row values in the `where` scope.

Backtick names inside lookups exactly the same way:

```formula
`Net Revenue`[`Sales Region` = "West"]
```

Use disambiguators when the resolved variable or dimension catalog reports duplicate names:

```formula
Revenue#a3f[Department in any] + Revenue#b41[Department in any]
```

Predicate comparisons use `=`, `<>`, `<`, `<=`, `>`, or `>=`. Set and range
membership use `in` or `not in`; the right side must be a set or range:

```formula
Revenue[Region in {"US", "CA"}]
```

Ranges are inclusive; descending endpoints are valid syntax but match nothing.
String endpoints use the same lexical comparison behavior as ordinary `<` and
`>` expressions.

Range endpoints are full expressions. Use literal endpoints for an absolute
range, offset endpoints for a range relative to the current segment, or combine
the two:

```formula
Revenue[Date.Month in "2026-01":"2026-12"]
Revenue[Date.Month in [-12]:[-1]]
Revenue[Date.Month in "2026-01":[-1]]
```

The first range is fixed. The second runs from twelve months before the current
`Date.Month` through the previous month. The third begins at a fixed month and
ends at the month before the current segment.

Within a lookup predicate, an offset can appear anywhere in the right-hand
expression and always uses the nearest predicate root:

```formula
Revenue[Date.Month = startofmonth([-1])]
Revenue[Date.Month where Date.Month >= startofquarter([-1]) AND Date.Month <= [-1]]
```

Offsets are not valid as standalone formulas or in formula conditions.

A predicate may start with a dotted property path. This is shorthand for a
`where` predicate owned by the first (root) dimension:

```formula
Salary[EmployeeId.Department = "Engineering"]
Salary[EmployeeId where EmployeeId.Department = "Engineering"]
```

Those forms are equivalent. A `where` attached to a dotted path is likewise
owned by the root dimension.

## Date granularity in segmented formulas

Write the system date as plain `Date` when reading its value. A date reference
follows the formula's date segmentation, so `Date` is the current period at
whatever grain the table uses — month starts in a monthly table, quarter
starts in a quarterly one. Another datetime dimension follows the view's grain
only when it carries the view's single date-axis descent
([[dimensional-modeling]] axiom 13).

A `Months Live` variable is a typical case (guarded, because `datedif` errors
when the start date is after the end date):

```formula
if(Date < `Launch Date`, 0, datedif(`Launch Date`, Date, "M"))
```

Use the compact offset form to select a nearby period on the system Date
dimension. In a monthly formula, these forms all mean the previous month:

```formula
Revenue[-1]
Revenue[][-1]
Revenue[Date.Month = [-1]]
```

Put an explicit Date grain between the lookup and offset brackets to perform
the lookup at that grain. This example looks up the previous quarter:

```formula
Revenue[]Date.Quarter[-1]
```

Predicates stay attached when an offset suffix is added. These examples select
Engineering at the previous current-grain period and at the previous quarter,
respectively:

```formula
Revenue[Department = "Engineering"][-1]
Revenue[Department = "Engineering"]Date.Quarter[-1]
```

Metric names containing spaces must be backticked before any bracket or
suffix: `` `Cash Balance`[-1] `` parses; `Cash Balance[-1]` does not.
When a reference with a shift is rejected, check the backticks and the
bracket shape before abandoning the relative form; the shift itself is
rarely the problem.

A value reference may pin a specific grain by appending a granularity keyword.
It resolves to the start of its own period even when the table is at a
different grain — the engine converts automatically, so `Date.Quarter` in a
monthly table is the start of the quarter containing that month (equivalent to
`startofquarter(Date)`). Valid suffixes:

- `.Day`
- `.Week`
- `.Month`
- `.Quarter`
- `.Half`
- `.Year`

A Date predicate path — left of its comparison, membership, or `where`
operator — always carries a granularity suffix; plain `Date` is never a
predicate path. Its grain names the bucket shape being addressed:

```formula
Revenue[Date.Month = "2024-01"]
```

```formula
Revenue[Date.Quarter = "2024-Q1"]
```

```formula
Revenue[Date.Year = "2024"]
```

Choose the key grain to match what it addresses:

- **Conditions** segment by the table/date axis grain when configured;
  otherwise default to `.Month` (`Date.Month in any`, `Date.Month = "2024-01"`).
  Beside a `period`, the grain is the model's base grain, not the table's: the
  period merges its window into the Date term at the base grain, and any other
  explicit grain is refused.
- **Absolute (`$`) lookup keys** address the target variable or dimension's own date
  buckets, so match the key grain to the target's data:
  `` `Headcount Payroll`$[Date.Month = this.Date.Month]``.

## Inclusive overlap proration

Use this shape when one periodic total comes from source rows with an amount,
start date, and optional end date. The source fields must come from the same
rows. If the amount's `slicesBy` lists those dates, treat a monthly, weekly,
quarterly, or yearly total as this case even when the user only says "total."
Do not use `Amount / 12` alone; it ignores when rows start and end.

1. Choose their one shared record dimension from `slicesBy`.
2. With `change.set_values`, write the rule at the record ×
   base-grain shape: `condition: "[Record in any, Date.Month in any]"`,
   spelling the record dimension by its exact `grammarRef` (the subset form
   keeps the rule applying under added drill-ins). Keep the Date term at the
   model's base grain: beside a `period`, any other explicit grain is refused.
   Write two items, same condition, one per regime — `period: "actuals"` and
   `period: "forecast"` — because a Date-term rule with no `period` governs
   actuals only.
3. Pin every source field to the current record as
   `Field$[Record = this.Record]`. The `$` keeps the report month from
   filtering the undated source row first.

This makes one calculation cell for each month and record. The target
variable's `SUM` aggregation then rolls the records into the monthly total.
For a weekly, quarterly, or yearly total, keep the base-grain rule and let
the coarse cells roll it up (`change.set_time_rollup` sets the method when
the default is wrong).

The overlap starts at the later of the row start and period start. It ends at
the earlier of the row end and period end. Add one because
`datedif(..., "D")` excludes the ending day.

```text
sum(
  if(
    `Starts At`$[Subscription = this.Subscription] <= eomonth(Date, 0) AND
      (`Ends At`$[Subscription = this.Subscription] = NULL OR
       `Ends At`$[Subscription = this.Subscription] >= startofmonth(Date)),
    `Annual Fee`$[Subscription = this.Subscription] / 12 *
      (datedif(
        if(
          `Starts At`$[Subscription = this.Subscription] > startofmonth(Date),
          `Starts At`$[Subscription = this.Subscription],
          startofmonth(Date)
        ),
        if(
          `Ends At`$[Subscription = this.Subscription] = NULL OR
            `Ends At`$[Subscription = this.Subscription] > eomonth(Date, 0),
          eomonth(Date, 0),
          `Ends At`$[Subscription = this.Subscription]
        ),
        "D"
      ) + 1) / daysin(Date, "month"),
    0
  )
)
```

Here, `Subscription` is the record dimension. Replace it and the synthetic fields
with resolved `grammarRef` names. Keep the same current-record pin on all three
fields inside one `sum(if(...))`. The outer `sum` makes the contribution
explicit at the record cell; the target's `SUM` aggregation performs the
parent rollup. Read the monthly result back. A nonzero source with zero in
every active month means the Date and record grains were not both declared.
Do not copy source dates or amounts into literals.

## Preferred time binding for imported variables

Use the property marked `isSystemDate: true` as the model's canonical time axis; transaction,
pay-period, effective, and close dates are dimensions and filters, not the model axis. When a
saved formula on the system Date axis reads an imported variable whose own time key is a
different source-date property (transaction date, pay-period date, effective date, close date),
make that source datetime the variable's `preferred_time_property` via an `edit_variables`
update on the imported variable. The calculation engine uses that one property-level binding to
project the current system-Date segment onto the variable's source date, so month cells aggregate
the source rows at month grain directly instead of stacking a day-level rollup of raw rows. Then
keep the source reference relative so it inherits the current report period:

```formula
sum(Revenue[Department = "Enterprise"])
```

Resolve the preferred datetime from the variable's source/query metadata (the ingestion handoff
names the authoritative source datetime). When the source supplies a single datetime the engine
already defaults to it, so the binding is optional. When it supplies more than one, the engine's
pick is arbitrary (map ordering), so always set `preferred_time_property` to the authoritative
source datetime; preserve an existing correct binding, and if the authoritative choice
cannot be determined from metadata or the handoff, ask the user instead of guessing. Extend the
relative lookup with any source dimensions that must be aggregated, for example:

```formula
sum(Opex[Department in any, Vendor in any])
```

An explicit source-date lookup or a `$` absolute reference bypasses the
engine-owned preferred-time projection and can collapse or zero the monthly
result. The inclusive-overlap recipe above is the exception: it deliberately
declares a Date × record calculation and pins each aligned source field to that
current record before comparing it with system `Date`. Do not replace the
system Date axis or copy monthly values into literals. For ordinary projection,
repair the source variable's preferred-time binding; for interval overlap,
declare both grains. Read the saved formula back at the model grain.

## Functions

Every function's signature, description, aliases and one example are in
`grammar-reference.md`, which is generated from the engine's own function
manifest. The worked examples below add what a signature cannot: when to
reach for one, and how it behaves at the edges.

Conditional:

```formula
if(Revenue > 100, "high", "low")
```

Nested conditional:

```formula
if(Revenue > 100, "high", if(Revenue > 50, "medium", "low"))
```

Error fallback:

```formula
iferror(datedif(StartDate, Date, "D"), 0)
```

Returns the first argument's value, or the second argument when evaluating the
first produces an error, matching Excel `IFERROR`. Here the fallback covers the
error `datedif` produces when `StartDate` is after `Date`. A `NULL` value is
not an error; it passes through unchanged. Division by zero (and modulo by
zero, and 0 raised to a nonpositive power) produces a cell error like Excel's
`#DIV/0!`, which `iferror` catches the same way.

Date shift:

```formula
dateadd(Date, -1, "month")
```

Like Power BI's `dateadd`, this shifts dates by a signed number of time units.
cfo.ai takes a single date value rather than Power BI's date column or calendar.
The quoted unit is `"day"`, `"week"`, `"month"`, `"quarter"`, `"half"`, or
`"year"`.

Date difference:

```formula
datedif(StartDate, Date, "M")
```

Returns the whole number of units between two dates, matching Excel `DATEDIF`.
The third argument is one of Excel's unit codes (case-insensitive): `"Y"`
(complete years), `"M"` (complete months), `"D"` (days), `"MD"` (days ignoring
months and years), `"YM"` (months ignoring years), or `"YD"` (days ignoring
years). Note these unit codes differ from `dateadd`'s unit names. `"MD"` can be
negative near month ends (for example Jan 31 to Mar 1 is `-2`); this matches
Excel's documented `MD` behavior and is intentional. A start date after the
end date is an error, like Excel's `#NUM!`; a `NULL` operand returns `NULL`.
A fixed date operand may be written as a string literal using the same date
strings that work elsewhere in formulas (comparisons and date lookups), e.g.
`"2024-01-15"` or `"2024-01"`.

Date components:

```formula
year(Date)
```

`year(Date)`, `month(Date)`, and `day(Date)` return a date's
components as numbers, matching Excel's `YEAR`, `MONTH`, and `DAY`: the 4-digit
year, the month 1-12, and the day of the month 1-31. `quarter(Date)` (1-4)
and `half(Date)` (1 or 2) extend the family to the quarter and half-year
grains; Excel has no equivalent functions. A granularity-suffixed date reference
resolves to its period start, so `month(Date.Quarter)` is the first month of
the containing quarter. A non-date operand is an error;
a `NULL` operand returns `NULL`. The sole legacy-compatibility exception is
`half(NULL)`, which returns `2`. A fixed date operand may be a string literal,
as in `datedif`.

Week functions:

```formula
weeknum(Date)
```

`weekday(Date, [return_type])` and `weeknum(Date, [return_type])` match
Excel's WEEKDAY and WEEKNUM, including the optional `return_type` codes.
Defaults match Excel: `weekday(Date)` numbers Sunday=1 through Saturday=7,
and `weeknum(Date)` starts weeks on Sunday with week 1 containing Jan 1.
`weekday` accepts codes 1, 2, 3, and 11-17; `weeknum` accepts 1, 2, 11-17,
and 21 (ISO 8601). `isoweeknum(Date)` matches Excel's ISOWEEKNUM and equals
`weeknum(Date, 21)`. Note `weeknum`'s Sunday-based default disagrees with
`startofweek`, which truncates to Monday; use `weeknum(Date, 21)` or
`isoweeknum(Date)` for Monday-based ISO weeks that align with `startofweek`.
The `return_type` must be a number literal.

Period length:

```formula
Revenue / daysin(Date, "month")
```

`daysin(date, unit)` returns the number of days in the unit-aligned period
containing the date — the proration primitive for converting between grains.
The unit is a string literal from `dateadd`'s vocabulary: `"day"` (1),
`"week"` (7), `"month"` (28-31), `"quarter"` (90-92), `"half"` (181-184), or
`"year"` (365 or 366). This is a cfo.ai function with no Excel equivalent
name; Excel's idiom for the month case, `day(eomonth(Date, 0))`, also works
here and returns the same value. A non-date operand is an error; a `NULL`
operand returns `NULL`.

Month shift and month end:

```formula
eomonth(Date, 0)
```

`edate(date, months)` and `eomonth(date, months)` match Excel's EDATE and
EOMONTH. `edate` shifts by whole months with the day clamped to the target
month's length (`edate("2020-01-31", 1)` is 2020-02-29); it is equivalent to
`dateadd(date, months, "month")`, which accepts the other time units.
`eomonth` returns the last day of the month `months` away, so
`eomonth(Date, 0)` is the end of the current month. Fractional `months`
truncate, as in Excel. A non-date first operand or a non-numeric `months` is
an error; a `NULL` operand returns `NULL`.

Date construction:

```formula
date(year(Date), 12, 31)
```

`date(year, month, day)` builds a date from numbers, matching Excel's DATE
exactly: fractional arguments truncate, months and days outside their ranges
roll into adjacent years and months in both directions (`date(2008, 14, 2)`
is 2009-02-02, `date(2008, 1, -15)` is 2007-12-16), and years 0-1899 have
1900 added (`date(108, 1, 2)` is 2008-01-02), so spreadsheet formulas that
rely on that behavior port unchanged. A year outside 0-9999 or a result
outside Excel's date range (1900-01-01 through 9999-12-31) is an error, like
Excel's `#NUM!`. One known divergence, confined to early 1900: Excel's
calendar contains the nonexistent leap day 1900-02-29, and `date()` uses the
real calendar instead, so `date(1900, 2, 29)` is 1900-03-01, day overflow
across February 1900 lands one day later than Excel, and Excel's
`date(1900, 1, 0)` ("1/0/1900") is an error here.
Non-numeric operands are an error; a `NULL` operand returns `NULL`. Composes
with the component functions and `datedif`, e.g.
`datedif(date(2026, 1, 1), Date, "M")`.

## Common formula patterns

Simple pass-through:

```formula
`Source Revenue`
```

Arithmetic between variables and dimensions:

```formula
Price * Quantity
```

```formula
Revenue - COGS
```

Ratio:

```formula
`Gross Margin` / Revenue
```

Aggregate across one dimension while inheriting the rest of the current segment:

```formula
sum(Revenue[Department in any])
```

Filtered aggregate:

```formula
sum(Revenue[Region in {"US", "CA"}])
```

Exclude specific dimension items:

```formula
sum(Revenue[Region not in {"APAC"}])
```

Conditional formula:

```formula
if(Plan = "Enterprise", Amount * 1.2, Amount)
```

Month-over-month growth rate:

```formula
(Revenue - Revenue[-1]) / Revenue[-1]
```

Running total:

```formula
`Running Total`[-1] + Input
```

A recurrence (a formula that reads its own previous period) treats a
missing prior as zero, so this chain starts at zero plus the first Input
and adds from there; the shapes below carry the rules for starting it
anywhere else.

A recurrence has three shapes. Choose by where the chain should start:

**1. Seed plus chain (default).** Two items in one `change.set_values`
call — a seed cell on the table's first month, and the plain chain:

```
`Cash Balance`$[Date.Month = "2026-01"] = 250000
`Cash Balance`$[Date.Month in any] = `Cash Balance`[-1] + `Net Cash Flow`
```

The seed gives the first cell its value; every later cell builds on the
one before. Four rules keep it working:

1. The seed goes on the FIRST month of the table's date window — a seed
   before the window is never read (the whole chain shows zeros with no
   error, limitations §10), and a seed after the first month leaves the
   months before it chaining from zero.
2. The seed's condition must name the chain's own grain — a seed at a
   finer or coarser grain silently never fires.
3. For quarterly or yearly views, add one `change.set_time_rollup`
   {method: "LAST"} anchor per grain; DO_NOT_AGGREGATE (the
   aggregation-function setting,
   [[dimensional-modeling:references/04-time.md#§4.4]]) plus the anchor
   shows the right closing balance at every time scale.
4. The chain names Date across many months, and such a rule with no
   `period` governs actuals only — so write one chain item per regime the
   window spans. As shown above it stops at Last close. The seed escapes
   this: a literal Date pin identifies one month wherever it falls
   ([[dimensional-modeling:references/limitations.md]], the
   running-balance recipe).

A `count()` guard on the prior read computes; it changes none of the laws —
the chain still needs an in-window prior read and the coarse-grain LAST
anchor — so the seed stays the taught form.

The `$` on both conditions is intentional: this recipe is one chain on the
unsegmented Date grain and stops at a dimension drill-in; to run the chain
in drilled grains too, one unsigiled subset chain beside its `period`
covers them all — repeating the rule per grain is retired (limitations §7).
Use an unsigiled chain `[Date.Month in any]` only when the user means a
separate chain in every segmentation containing Date, then seed every
intended segment at its own starting coordinate.

A running total can also be written as
`sum(X[Date.Month where Date.Month <= this.Date.Month])`, but that version
is only right in a monthly table: a quarterly view of it adds up through
the FIRST month of each quarter, not the last — rule 3's LAST anchor is
what shows the closing month.

**2. Start at the actuals/forecast frontier (the range shape).** For "grow
from what actually happened": two `change.set_values` items on the same
variable, each naming its own `period` — one windowed default per regime.
The actuals window takes the imported column:

```formula
sum(<runway:exttables/UUID/columns/subscribers/>)
```

and the forecast window grows from the previous cell:

```formula
Customers[-1] * (1 + `Growth Rate`)
```

Both write the same variable's cells, and `[-1]` reads cell values
regardless of which window produced them. So the first forecast cell
starts from the last actuals cell. The shipped ranges split at Last
close, so this fits only when the user's boundary IS that frontier. For
any other boundary, define the window first with `edit_dimensions` ranges
(ranges must stay contiguous, so a new one is carved from its neighbors), then
pass its name as `period`. A `{start, end}` period only reuses a range
whose bounds already match; when none does, a dated `period` declines
and names the existing windows rather than minting one.

**3. The starting value folded into the formula (the date-pin branch).**
Shape 1 keeps the seed as its own cell. This shape carries it inside one
formula instead, which reads better when the seed month also receives its
first change:

```formula
if(this.Date.Month = date(2026, 1, 1), 250000 + `Net Cash Flow`, `Cash Balance`[-1] + `Net Cash Flow`)
```

A date pin states the seed month declaratively instead of inferring it
from an empty prior read. Shape 1's window and anchor rules apply here too: the pinned
month must sit inside the table's window (and be its first month), and
coarse views need their `change.set_time_rollup` anchors. The
condition-grain rule does not: this pin lives in the expression, where it
follows the view's grain instead of silently missing — which is exactly
why the coarse views need anchors. When the named month is simply where
the table begins, prefer shape 1 (the form-choosing test lives in
SKILL.md).

Two rules when you do use a date test: reference the system `Date`, never
an uploaded table's own date column. And know which spelling works where.
Inside an expression — shape 3 above — both `this.Date.Month =
date(2026, 1, 1)` and `Date.Month = "2026-01"` compare correctly. In the
`condition` FIELD of a write, only the string form works: a
`date(2026, 1, 1)` condition parses and then silently never matches. If
the numbers look wrong, the cause is almost never the comparison, so
check the seed's window coverage and the coarse grains' anchors
(limitations §10 and §5) before redesigning the test.

Backtick-delimited names:

```formula
`Net Revenue` - `Cost of Revenue`
```

Duplicate-name disambiguators:

```formula
Revenue#a3f - Revenue#b41
```

Constants:

```formula
100
```

```formula
"fixed_label"
```

When the user does not specify conditions:

```formula
Revenue
```
