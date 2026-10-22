---
name: build-model
description: Build, change, or diagnose a model and model the business in it. Use for variables, formulas, dimensions, mappings, time, rollups, table layouts, comparisons, formatting, and unexpected results; for forecasts, budgets, plans, and scenarios; for headcount, payroll, and rosters; for bookings, pipeline, and deal registers; and for statements, cash and runway, budget versus actuals, and unusual results.
---

# Build the model

A workspace has one model, and every table is a view of it. This skill starts with the picture of how the engine computes a model, told on one running example, then builds that example as the calls that make it, then the
engine's detail for imported records, mappings, and which rule fills a cell. The rest is the business the model simulates: what it earns, what it costs, who does the work, where cash lands, and the statements that read
from all of it.

## The model and its views

A variable's name, formulas, and values are the model, shared by every table that shows it.
Ordering, formatting, and the breakdown a table shows are the view, owned by that table. A change to
an existing table keeps its block identity and edits it in place; a copy on another page is a
separate block with the original left alone.

A readable name identifies an object only together with its identity: two variables can carry one name, and the token an inspection or receipt prints, such as `Headcount#7d3`, names exactly one. A plain name that matches an existing variable shows it and adds only where it holds nothing, and a save that would replace what it holds is refused whole; a row that carries its token writes it, and that rule
replaces the one every table reads. The existing object is the one to reuse when it is the one the request means; a requested report keeps the names the request gives its variables, even when an unrelated object already has such a name (`new: true` on the create), and keeps the shape the request spells out: breakdown dimensions in the order listed, each listed field a column. A different name or shape is a
different deliverable. Saved data, research, supplied numbers, and disclosed assumptions all build, and the missing variables and dimensions the work needs are yours to create.

Variables and dimensions stay on at least one table in each scenario. Updates and reshapes remove
occurrences when another saved table shows the object. Table and page deletion keep objects shown
outside the deleted tables; last-occurrence deletions depend on downstream use. Place objects worth keeping elsewhere first.
Tool calls in one turn run at the same time, so a call that needs another's result, a variable a
create makes or a member an update adds, waits for it, and one create carries a table with its rules.

## The picture

The running example is a listening service. It sells two tiers, Lark and Chorus, in two territories,
Southgate and Eastmere, and is modeled month by month. Five ideas carry the engine.

### A cell

A cell is one variable at one member of each of its dimensions, and at one period when the variable
is dated. Paying listeners for Chorus in Eastmere in March 2027 is one cell. Tier fee for Chorus is
a timeless cell: it has a tier and no month. A table shows some of a variable's cells; the cells
exist wherever the variable's rules and inputs reach.

### A rule and an input

A cell holds a literal input or a formula. An input is a supplied fact or a chosen assumption,
and its cell is the customer's control: an opening count, fee, or growth rate. A formula calculates
from inputs at each cell's coordinates and keeps its dependencies even when today's answer is constant.
Its condition names the Date grain: a Month rule reaches additional business dimensions at Month;
a daily forecast needs matching Day coverage, and time conversions are explicit calculations.
Actuals and Forecast are separate range choices. The saved scope and selected rule explain coverage;
a literal such as months in a year is part of the rule.

A forecast reads earlier cells. A dated input carries forward through an opening cell and a
default formula for later periods, so every later month depends on the assumption. Repeated literals
would create independent overrides. Weekly listeners grow from an opening input:

```formula
`Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory])
```

In the first Forecast month, `Weekly listeners[-1]` is the last Actuals month, so it reads that
month's input, imported value, or Actuals rule; with none of them it reads 0, and every later month
compounds from 0. A model with no connected actuals seeds the opening input in its last Actuals
month. For cash, the opening cell holds the starting amount and the default adds Net Cash Flow to
the prior balance. A recurrence reads its previous period's modeled value: its selected rule, or the
source or zero fallback. An estimated opening stays distinct from observed history, and a legitimate
zero, such as payroll before a hire starts, stays zero.

An intentional one-cell override pins that cell and nothing else; removing it lets the default's
value flow into parents and rollups, so the neighboring periods and parents are what to read
afterward.

An empty read has one rule. An aggregate over no matching cells returns that function's empty
result: `sum` and `product` 0, `min`, `max`, and `average` null. A variable cell with no rule and no
input reads null for a date or text and 0 for any other type, so an End Date nobody entered is null,
and `End Date = null` is true there. A blank imported number is null in the source; a SUM variable
over a record whose rows are all blank reads 0, so `count` over that variable counts the cell, while
`last(<column>)` over the same rows reads null and `count(<column>)` on the raw column counts no
blank. `a = null` asks
whether a value is missing; `a = b` is null when either side is missing, so `if` takes its false
branch, and an unassigned group matches two missing attributes with `(a = null and b = null) or a =
b`. A ratio states its error fallback, ``iferror(`Listening revenue` / `Paying listeners`, NULL)``.

### A read

A formula runs at one cell, and a plain name reads the other variable at that same cell: the same
tier, the same territory, the same month. That is the read to want when both variables share the
cell's coordinates. Listening revenue on the Chorus × Eastmere × March cell reads Chorus × Eastmere
× March paying listeners.

An input with fewer coordinates has no value there. Weekly listeners is saved by Territory and
Month, so the Chorus × Eastmere × March cell asks it for a Chorus value it never had, and the read
gets the generated 0. An absolute reference, `$`, keeps only the coordinates it names, so the read
pins the ones the input carries: `` `Weekly listeners`$[Territory = this.Territory, Date.Month =
this.Date.Month] ``. A timeless input has no month at all, so a monthly rule reads it pinned by its
own keys, `` `Tier fee`$[`Listening tier` = this.`Listening tier`] ``. An absolute read that
identifies a record names every dimension the input carries, and one that drops a dimension combines
whatever records share the ones that remain.

A bracket changes a coordinate and keeps the rest: `` `Listening revenue`[Territory = "Eastmere"] ``
reads Eastmere instead of this cell's territory, and ``sum(`Listening revenue`[Territory in any])``
adds every territory at this cell's tier and month. Outside a `where`, the plain name of a dimension
the cell carries is the cell's own member, so ``[`Employee ID` = `Employee ID`]`` and ``[`Employee
ID` = this.`Employee ID`]`` are the same read.

Inside a `where` clause a plain name reaches only the candidate's own key, so a field the candidate
does not carry is read through that key, and a text variable compared with `this.<Dimension>` there
compares the member's name. A predicate value is an expression, so a bracket can name a computed
date: `Date.Month = startofmonth(`Last Snapshot Month`$)`. A condition on another dimension keeps
the drilled coordinate: `sum(Amount[Account Type = "Income"])` broken down by Account gives each
account its own amount, while `sum(Amount[Account in {"A", "B"}])` broken down by Account repeats
one total on every row, because the bracket replaces the cell's Account. A dimension reference reads
a value, not the cell's grain: with Deal absent from the cell's coordinates, `Deal` can still
resolve a source or mapped value there, so `Deal = null` tests the value and cannot tell a parent
from a record row. A dimension attribute read at a cell with no source row is null. Formulas use
readable names, with the returned `grammarRef` for backticks or disambiguation.

### A breakdown

Nesting dimensions beneath a variable makes a breakdown: Paying listeners by Listening tier, then
Territory. Its leaves are the cells where every broken-down dimension has a member, and they
compute: paying listeners are weekly listeners times the uptake rate. The leaf rule is the
`child_defaults` entry for the deepest dimension, written once for every leaf and saved on the
leaf's full path; with no leaf rule the leaves hold inputs. The table's save then gives each level
above the leaves a default that sums the level beneath it, so each tier adds its territories and the
total adds its tiers, and the write receipt lists them. Where no rule computes a number variable's
leaves, they get a saved rule of 0 that each typed number overrides, so a blank leaf reads 0 and
counts as 0 in the totals. A value set on any leaf's row reaches every total above it.

A level needs a rule of its own only when adding the children is the wrong arithmetic; that rule
takes the place of the sum there. A variable whose `format` is PERCENTAGE is not added up, and its leaves get no rule of 0. A leaf rule on it computes the leaves alone, and the save gives each row above them a default of its own: the leaf formula, read on the row's own inputs, where everything it reads has a value on that row, so a margin there is total profit over total revenue; and a blank in each range where a leaf holds a typed-in rate (one saved with no range counts in Actuals and Forecast), where several formulas compute the leaves, or where the leaf formula reads a variable with no value on that row or takes its own row's item of a dimension that row lacks (`this.Segment` above the segments), since no rate for the whole follows from them without weights. A read that names its item, `Rate[Segment = "Enterprise"]`, or every item, `sum(Revenue[Segment in any])`, needs no Segment from the row; it takes the row's other dimensions with it, and the row gets the formula where the variable has a value in that cell. A percentage written as the row's `formula` is one rule that computes every row, the rows above the leaves included, so the save adds no default there: a typed-in rate it reads is blank above the leaves and counts as 0, so `Profit / Revenue - Rate` shows the margin alone on those rows. A row's `numberFormat` on one table is display only, and the variable keeps its sums. A rule whose inspection shows `writer: "level_default_sync"` belongs to the product and follows the tables, so a layout save can replace it; a save's receipt lists those changes under `level_defaults`, and they are not a change to anything a person authored. Revenue per payer is total revenue over total paying listeners,
not a sum of rates, and a ratio of two variables that already combine is right at every level with
no breakdown rules at all: ``iferror(`Listening revenue` / `Paying listeners`, NULL)``, written as
the row's `formula` with no Date, reads each cell's revenue and paying listeners at that cell, a
quarter's or a year's included. A grouped header column, such as Territory over its months, combines its children as the `summary` on the column beneath it says: `SUM` adds them, and `NONE` keeps the row's own value at the header, so a ratio there reads total revenue over total paying listeners. A drill-down over a record key, weighted pipeline by Stage, Owner,
and Deal, has a leaf that pairs each deal's amount with its probability; the pattern is under
Imported records and sources.

### Time

System Date generates a table's periods inside its window, so forecast periods exist before data
does; a source date shown as a field is not the report calendar.

Rules compute at the model's base grain, usually the month. Day, week, month, quarter, half-year,
and year are distinct grains, and a Date condition matches the grain being calculated: `Date` is the
current period, and predicate paths carry their grain, such as `Date.Month`. A month rule does not
fill quarter, half-year, and year cells; the variable's time rollup does. A number a monthly table
creates gets a `SUM` rollup, right for a flow such as revenue, unless `rollup` on its create row
names another: `LAST` for a balance or a count such as cash, weekly listeners, or headcount,
`AVERAGE` for a mean. A rule with no Date computes at every grain and takes no rollup, so a ratio
written that way stays a ratio in its quarters, half-years, and years. `set_time_rollup` changes a
saved variable's. A balance takes its `LAST` rollup because the `last` source aggregation returns
the latest-dated row anywhere in the coarse period, an earlier month's value when the closing month
is empty.

Last close is the scenario's accounting boundary. Actuals covers the closed periods; Forecast covers
the later ones. A dated rule belongs to one range, so a rule that holds everywhere is written for
both. Last close selects formula ranges; source observations establish facts, and the two can differ
in either direction: a known event after close uses Forecast-range rules, and a requested estimate
in a closed period uses Actuals-range rules. Changing Last close changes other reports; the
requested periods are covered with rules and disclosed estimates, and an estimate is presented as
one.

Comparison values are presentation. When downstream formulas need a difference as model state, make
it a modeled variable with an explicit calendar offset, `Revenue[Date.Month =
dateadd(this.Date.Month, -12, "month")]`; a moving window is one range with inclusive, possibly
relative, endpoints, `average(Revenue[Date.Month in [-3]:[-1]])`.

Proration binds each amount and effective date to its source record, while system Date supplies the
period. The rule for one record in one month, `[Subscription in any, Date.Month in any]`, is written
once for Actuals and once for Forecast from the record's plain fields, `Annual Fee` and its start
and end dates: a nonempty inclusive overlap is `datedif(start, end, "D") + 1`, monthly calendar-day
proration uses `daysin(Date, "month")`, and the first, last, and partial periods and their totals
are the check. Week boundaries follow one reporting calendar. A source-derived date grouping, such
as a signup cohort, can lose older members outside the saved window while its parent total stays
right; widening the window restores them.

## One worked build

The running example as four `edit_table_blocks` creates. Two timeless tables hold the inputs:

```
{"action": "create", "name": "Tier assumptions", "rows": [{"dimension": "Listening tier", "items": ["Lark", "Chorus"]}], "columns": [{"variable": "Tier fee"}, {"variable": "Uptake rate"}], "values": {"columns": ["Tier fee", "Uptake rate"], "rows": [["Lark", 20, 0.04], ["Chorus", 50, 0.01]]}}
{"action": "create", "name": "Territory assumptions", "rows": [{"dimension": "Territory", "items": ["Southgate", "Eastmere"]}], "columns": [{"variable": "Listener growth"}], "values": {"columns": ["Listener growth"], "rows": [["Southgate", 0.02], ["Eastmere", 0.03]]}}
```

Listening is dated: each territory's opening count is an input in the last Actuals month, 2026-08
here, and the Forecast rule grows it; as a count, its quarters, half-years, and years read the last
month:

```
{"action": "create", "name": "Listening", "columns": [{"time": "MONTH"}],
 "rows": [{"variable": "Weekly listeners", "breakdown": ["Territory"], "rollup": "LAST",
   "child_defaults": [{"dimension": "Territory", "forecast": "`Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory])"}]}],
 "values": {"columns": ["2026-08"], "rows": [["Weekly listeners[Southgate]", 100000], ["Weekly listeners[Eastmere]", 60000]]}}
```

Paid listening breaks down by tier then territory. The leaf rules pin the inputs that carry fewer
coordinates, the level defaults add the leaves up to each tier and the total, revenue's quarters and
years add their months, and the ratio, written with no Date, is right at every level and in every
period:

```
{"action": "create", "name": "Paid listening", "columns": [{"time": "MONTH"}], "rows": [
  {"variable": "Paying listeners", "breakdown": ["Listening tier", "Territory"], "rollup": "LAST",
   "child_defaults": [{"dimension": "Territory",
     "actuals": "`Weekly listeners`$[Territory = this.Territory, Date.Month = this.Date.Month] * `Uptake rate`$[`Listening tier` = this.`Listening tier`]",
     "forecast": "`Weekly listeners`$[Territory = this.Territory, Date.Month = this.Date.Month] * `Uptake rate`$[`Listening tier` = this.`Listening tier`]"}]},
  {"variable": "Listening revenue", "breakdown": ["Listening tier", "Territory"], "rollup": "SUM",
   "child_defaults": [{"dimension": "Territory", "actuals": "`Paying listeners` * `Tier fee`$[`Listening tier` = this.`Listening tier`]",
     "forecast": "`Paying listeners` * `Tier fee`$[`Listening tier` = this.`Listening tier`]"}]},
  {"variable": "Revenue per payer", "breakdown": ["Listening tier", "Territory"],
   "formula": "iferror(`Listening revenue` / `Paying listeners`, NULL)"}]}
```

Checks: the first Forecast month grows from the opening input, not from 0; Chorus × Eastmere in
September is 61,800 × 0.01 × 50 = 30,900 of revenue; Chorus's row equals its two territories.

## How a rule reaches a cell

A relative condition claims every cell whose coordinates include all its dimensions, at that depth
and below: `[Stage in any]` runs at every stage, owner, and deal cell. The exact form, `$[...]`,
claims exactly the depth it names. A parent's fan-out that lands where no finer rule claims the cell
runs the parent's own rule there, which reads itself, `#CIRC!`.

When several saved rules could fill a cell, the more specific rule wins, in a fixed order. An exact
condition (`$[...]`) beats a relative one (`[...]`). Among relative rules, the one that names more
dimensions wins: `[Stage in any, Deal in any]` beats `[Stage in any]`, which beats `[]`. Among rules
naming the same number of dimensions, the one that pins more of them to a member wins: `[Stage =
"Won"]` beats `[Stage in any]`; a single member beats a set, and a set beats an expression. Only
when all of that is equal does the most recently saved rule win, so saving either tied rule again
flips the result. Generated fallbacks (the source read, the zero) rank below every saved rule, and a
rule's range limits when it applies without changing its rank.

A rule's condition comes from the row it is written on: the members pinned above it, the deepest
open dimension as any, and Date. A member row pins its members, so `child_defaults` on Scoping
writes `[Stage = "Scoping", Owner in any]`, a default for Scoping's owners only. A breakdown axis
row pins no member, so the deal rule written as `child_defaults` on `Total[Stage][Owner]` is saved
on the deal's full path, `[Stage in any, Owner in any, Deal in any, Date.Month in any]`, and claims
every deal cell. A `child_defaults` entry is such a rule, and the exact-depth form is the `$` rule
on the axis row. One table can place a variable twice, each placement with its own breakdown and
`child_defaults`, and a rule both placements declare is saved once.

Each row of a read names the rule that computed it under `formulas`; `inherited_from` names the
owner and condition of a rule saved more broadly or in another table. Diagnose a wrong or zero value
with `formula_origins` and with `include_inputs`, which lists each read and its value. A value the
person, another read, or your own arithmetic disputes gets `recalculate`: a calculation from scratch
that replaces the cached result the page shows and says whether the cache held different values for that same read's calculation request; a read with other options, such as `include_inputs`, can send a different request, whose result the cache holds separately. A formula whose expression does not parse still claims its cells and shows the parse error in each until the expression is fixed.
A formula whose condition does not parse applies to no cell, so the cells it was written for take the next-ranked rule or the source or zero fallback. A same-period cycle fails, and a valid prior-period offset breaks it.

## Imported records and sources

A variable combines its source rows into one value per cell by its aggregation, a setting on the
variable alone. An imported column with no rule of its own reads that aggregation over the record's
observations through a generated fallback, the aggregation's function over the source column:
`sum(<runway:exttables/550e8400-e29b-41d4-a716-446655440000/columns/Amount>)` for an amount that is
added, `last(<column>)` for a field aggregated by its latest observation, such as the current
department on a roster keyed by Employee ID. In a timeless table `last` is the latest observation
overall; in a dated table it is the latest in each period. A formula that names an imported variable
reads the value its aggregation already made, so `last(Amount)` over a SUM import receives the sum
of the record's rows. An absolute read of a restated field gathers all of a record's rows before the
aggregation can choose, which is what multiplies a deal's amount by its snapshot count; the relative
read lets the aggregation pick one. The fallback reads its column through Last close, so Forecast
months are zero unless the variable has a Forecast rule; `default_formulas.forecast` of `last(<its
column>)` reads the source rows dated in each forecast month. A new import's saved `[]` rule,
`sum(<column>)`, has no range and reads every period. A raw external-table column reference carries
the cell's coordinates, Date included: in a dated rule `max(<column>)` sees only the rows dated in
that period, in a timeless rule every row. The later of two dated values is `if(a > b, a, b)`.

A source can carry several date columns: an observation month, a start date, a close date. An
imported record's fields sit on the reporting calendar at its preferred time column
(`preferred_time_property`), the source's first date column until one is set; the date the business
reads the record by, expected close for open pipeline, the observation month for a restated rate, is
the column that setting names. An `edit_table_blocks` `sources` entry sets it once for the whole
source, on every column of the source that the same write does not set by name. `first` and `last`
order the record's rows by that same column, so `last` is the row with the latest preferred date,
and rows on one date keep their source order. A relative read in a period covers the rows whose
preferred date falls in it, so a field placed at a date that never falls in the period reads nothing
there. A predicate on a date column filters within that selection, and an explicit period filter
names the grain on both sides, `ObservedAt.Month = this.Date.Month`. A sync column stamps when an
export ran. An export that restates every record each run carries one state per record under each
stamp. When a record's rows are identical under every stamp, the export restates the record as it
stands now, and a stamp names which records existed at that run; when the rows differ, each stamp
holds that record's state at that run, and the stamps are a history. Which of the two an export is,
is a fact of its rows that the preview shows. A read that filters the sync column against a snapshot
input covers that one export; a read with no such filter covers every export in its selection.

A source dimension placed as a column of a table keyed by the record, `{"dimension": "Owner"}` in
`columns` of a register keyed by Deal, shows each row's member, `last` over the record's rows when
it has several: the latest observation by the source's date column when the source has one, the last
row in source order when it has none. A member is matched by its kind: a text member by its exact
spelling, a date member by `date(y, m, d)` or its text, and a datetime member, whose text carries a
clock such as `"2025-11-01 00:00:00+00"`, by that text, while `ObservedAt.Month` is each member's
month as a date, so `ObservedAt.Month = date(2025, 11, 1)` reaches the November snapshot wherever
it is written.

An entity ID identifies a record; an observation stamp identifies one observation of it. A daily or
monthly export restates the same employee or deal once per sync, while a ledger holds one row per
transaction. Current state is the record's latest observation, and history is the observation whose
stamp falls in the report period, so a record observed repeatedly needs one observation selected per
entity before any total, while a ledger is meant to be summed. A connected feed adds one observation
per record per sync, so a source with one row per record today will not stay that way.

The roster and the monthly table answer different questions. A record table keyed by Employee ID
with no Date is a current-state view: each attribute's latest observation, one row per employee. A
monthly table is a time series: each period reads that period's own observation, bound to the report
Date, so the April column reads the April row and a raise shows up in the month it happened. A read
with no date binding returns one value for every period, which flattens the history the monthly
table exists to show. The plain read answers only for months that have a row; carrying the last
observation into later months is the carry-forward default, disclosed as an assumption, and where
the Actuals and Forecast ranges split changes none of that. A connected roster whose template
declares its record key and observation date lands each numeric field with that default already
written: `[Employee ID in any]` reads the period's latest observation and carries the prior period's
value when the period has none, and `[]` sums the employees' record values, so a monthly table and a
company total need no rule of their own.

A field that changes over time is looked up without the `$`: with a monthly roster,
`sum(Headcount[Employee where Department[Employee = Employee] = this.Department, Department = this.Department])` counts
each employee under the department their row shows in this month, so a transfer moves the person in
the month it happened, while `Department$[Employee = Employee]` applies the latest department to
every month, as a timeless department mapping does. When the label is fixed, the mapping is the
tool; when it moves, a text variable carries the label in force. Department in Force reads
Department in Actuals and carries forward in Forecast, and a department row sums the employees it
held that month:

```formula
sum(Headcount$[`Employee ID` where `Department in Force`$[`Employee ID` = `Employee ID`, Date.Month = this.Date.Month] = this.Department, Date.Month = this.Date.Month])
```

Which employees sit under which department is membership, which belongs in a mapping from Employee
to Department. An employee sits under their mapped department. With no mapping or shared source,
each sits under every department and a rule compares theirs to the row's. Records with no item for a
dimension show as None; an open dimension condition covers that empty member, a catch-all mapping
can classify it, and the parent's inheriting rule reaches it along with every named member.

A drill-down over a record key computes at the record and adds up above it. In weighted pipeline
shown by Stage, then Owner, then Deal, by month, the deal is the line: its formula is written once,
as `child_defaults` on the axis row above the deals, `Total[Stage][Owner]`, and the table's save
gives each owner, each stage, and Total a default that adds the level beneath it, in each range the
deal's formula covers. On a deal's row the formula reads that record's own fields by their plain
names, so `Amount`, `Probability`, and `Expected close` are that deal's values:

```formula
if(startofmonth(`Expected close`) = Date.Month, Amount * Probability, 0)
```

The pairing happens at the record and the adding above it, which is the shape of any sum of
products: a product written on a parent would multiply a sum of amounts by a sum of probabilities.

Nesting one source dimension under another shows the combinations in the source rows, so Anouk sits
under Scoping only where a Scoping deal is hers and an ownerless deal sits under the empty member,
spelled `empty:None` in a selector; an all-empty combination appears when `show_empty_members` is true.
A saved item of a key nests under its mapped item, or under each mapped item when no formula maps it.

## Dimensions and mappings

A dimension mapping classifies one source dimension through another: Account to statement Bucket,
several department vocabularies to one reporting Department. The classification is shared model
structure: the mapping holds which accounts make up each statement line, a row's formula names the
line, and the person changes the classification once, in the mapping, with every row that reads it
following. Mapping changes are reversible edits with historical effects worth explaining. Source
columns supply dimensions and variables; manual dimensions carry planning inputs, and a planned
record has its own identity until a source record replaces it. Several sources describing one
business concept share one reporting dimension, each mapped in with its own keys. A lookup's keys
are dimensions, and a mapping rule's condition names every key, each term comparing one key's value
or leaving it open with `in any`.

A mapping is how a statement gets company-specific lines. A one-off visual grouping can stay as table
nesting; renaming a user-created item is an item edit; making several imported labels read as one
item is a mapping. A written mapping carries a catch-all whose stated meaning covers the unmatched
items, and the receipt's `row_membership` is the write's effect on other reports. A timeless mapping
view, source keys on rows and mapped attributes as columns, is where a person edits a classification
and the home of each dimension a map creates. A blank mapping cell is a source item that maps nowhere
and stays visible; a reporting category with no activity is right when nothing happened and wrong
when the mapping never sends anything there, and only the source rows say which. A filter names an
exact member: the source code `DATA` and the mapped label `Data` are different values.

A mapping reads in formulas two ways. For an imported measure, whose source rows carry the mapping's
keys, a mapped dimension in a predicate groups those rows through the mapping:

```formula
sum(Amount[Bucket = "COGS"])
```

An entered or formula-authored variable has cells only where they were written, such as Account ×
Month, so a predicate on Bucket reads it at Account × Bucket × Month, where nothing is written, and
the next matching rule runs there. An authored variable rolls up through its key instead:

```formula
sum(Amount$[Account.Bucket = this.Bucket, Date.Month = this.Date.Month])
```

`Account.Bucket` keeps the read at Account × Month and selects the accounts the mapping puts in this
bucket. A hop is a lookup keyed by the value before it: `Bucket$[Account = this.Account]` reads the
mapping by its key, and `this.Project.SourceTeam.Team` works when Project maps to SourceTeam and
SourceTeam to Team. A measure that already isolates its scope reads while keeping only Month: `R&D
Expense` held for R&D accounts alone feeds the R&D department's Opex rule as ``sum(`R&D
Expense`$[Account in any, Date.Month = this.Date.Month])``, with Department dropped deliberately.

## Tables as views

A time-series table has variables on rows and Date on columns; a record table has records on rows
and fields on columns, its key rows the items of a dimension such as Employee ID. Its fields are
variables such as Salary and dimensions such as City, shown per record; a stable identity keeps two
workers with one readable name distinct. A row or column declared on create takes the same `sort`,
`role`, `grain`, `members`, and `hidden` a saved node takes.

A dimension shown as a field beside each record is read at the cell's address, the dimensions on
the table's axes, which a mapping rule needs as its keys; nesting it beneath a variable makes a
breakdown. Moving an axis between rows and columns changes presentation, not the cell's grain. A
filter narrows the items of a breakdown dimension, and records are selected by their fields through
a measure: active headcount is a measure that counts active employees with Status kept as a field.
A formula conditioned on a classification dimension keeps each drilled member's own coordinate, so
members outside the condition read empty and hide when empty members are hidden.

A read resolves every cell a table shows, so a table that lists every source record with many
columns costs rows times columns, and a mapped column adds a lookup per row. A request for a column
on the records keeps that column, and the rest of the spreadsheet comes with it only when the whole
table is cheap. When the records are too many for every column, the columns the request names and
the ones needed to read them stay, with the row dimension's items filtered to the rows that matter.
A table is the person's report, and a wrong window is fixed on the table itself.

Period-specific inputs and reports use system Date; genuinely timeless mappings, rosters, and
assumptions can omit it. Writing dated values or selecting periods for readback does not set the
table's display window. For a report with a requested start and end, set the table's `window`
explicitly and verify its saved window range as well as its values. A readback of selected months
proves those cells, not the full displayed range. A saved block owns its name, window, granularity,
transpose state, comparison, visibility, and formatting, and a change touches only the aspects
requested. Number format and decimal places on a variable are defaults; row, column, and cell
formats belong to the saved table, styled at the broadest scope that fits, and currency, color, and
variance polarity stay on the variable.

An analysis table is saved model state built from model variables and dimensions. Like every saved
table, a read resolves its cells and everything they read, until it is removed once it stops
being useful. Related edits save together, and a table that exceeds one request grows in coherent
groups that keep the table and its shared variables. A table read with `layout_problem` is edited
against its returned refs.

## Scenarios

Mutually exclusive business futures are scenarios. Additive dimension items coexist in one future
and their aggregate answers a real question; a total across alternative fundraising or growth cases
answers nothing, so alternatives are scenarios and the model's dimensions are the coordinates the
business actually has. A scenario inherits unchanged state from its base. User-authored overrides
are scenario-scoped; imported source records stay shared and read-only.

A write with no `scenario` lands in the scenario the person is viewing, which can change between
turns; a write meant for another scenario names it. A lever that moves only the forecast is a dated
input whose Forecast cells differ in the scenario, or an input only the Forecast rule reads: Upside
overrides Listener growth, which only the Forecast rule of Weekly listeners reads. A timeless value
that every period reads moves the history too, because a scenario can change closed periods as well.
Cases sit side by side in one table through its `comparison` with `scenarios`. Each comparison column identifies a period and a scenario: Main's September 2027 column and Upside's September 2027 column have separate returned refs. Selecting the inspected Upside ref reads that case's value.
A read of the selected scenario with `comparison: "none"` inspects its underlying model values and formulas while keeping the table's saved comparison. On an underlying read, `formula_origins: "all"` adds annotations showing which rules computed its cells. Comparison values can be complete while the selected formula is reported as unavailable.
A Canvas simulator shows one scenario at a time unless cross-scenario Canvas reads are enabled for the workspace.

## What a saved value proves

The calculated readback tells a no-data blank from a numeric zero. A write receipt whose `status` is
`applied`, with its `changed` counts and readback, proves the change, and the receipt states what
the write replaced, tied, or refused, so the next call can be shaped from it. When a receipt cannot
prove the outcome, the saved table is read at meaningful starting and later periods, business
segments, and coarser grains. Verify a saved read on system Date at the requested grain with a
member whose state changes inside the window: a start, an end, a rate change, a different currency.
A member that never changes cannot reveal a wrong read, and an undated total cannot prove the
projection.

A saved value proves that the rule computes at those coordinates; a member whose value follows from
its own record, a salaried member's full month equal to its rate divided by twelve, proves the
amount, and a cell that differs from its record is unfinished work. A result that differs from the
intended model traces to the selected rule and what its references read, and the breakdown stays
while the rule is repaired. A driver test proves how a result responds: after an input change,
compare the affected and the unaffected results at the requested grains. That input change is a
save: in Main it changes the person's numbers until reversed; in a scenario, that scenario alone.

A row blank or zero across the whole horizon is unfinished work unless the business fact is zero.
Say which checks ran and which relationships stay untested. A read that timed out proves nothing,
and the work stays unverified until a read returns. Missing scope, a wrong regime, a source column
placed at the wrong date, and unmapped items are modeling conditions, resolved in the model while
the imported rows stay as they are; the work is finished when the requested cells compute from the
source for the members whose state changes inside the window, and rows the source never contained
are the only evidence gap to report. The `variable` and `dimension` reads list what depends on an
object; a delete without `force` is refused while anything does and deletes at once when nothing
does; `force` is for a deletion the person asked for after hearing them. An edit to past periods is said so.

Build with these rules; check them when calculations slow or time out:

- **Use only necessary time detail.** Prefer monthly or yearly source queries and formulas when finer detail cannot change results.
- **Use general formulas.** Use the cell's system Date or dimension item instead of copies with hardcoded years or items.
- **Share expensive expressions.** Consider helper variables for repeated calculations; measure whether they improve performance.
- **Use source data.** Read available amounts, dates, and items instead of hardcoding values or manual lists. Keep editable assumptions where needed.

Preserve exceptions, event timing, historical observations, dimensions, and Actuals/Forecast coverage. Verify repairs with the same read: unchanged values, missingness, and errors; improved calculation time.

## What a model of the business is

A model is a simulation of the business: its editable drivers explain how revenue, costs, staffing,
and cash change over the horizon, and every calculated output traces to a control an operator can
move. A request to build a model produces the whole simulation, every part of the business that
changes the economics, connected from its levers to cash, with its records, scenarios, pages, and
closing simulator and walkthrough. A request for one plan, forecast, statement, or projection
produces that piece, built so it can join the rest.

A model has a horizon, a reporting currency, the history that is available, and the scenario's Last
close. By default it spans the previous twelve months and the next twenty-four. It keeps recorded
results, commitments, and provisional assumptions apart, and it becomes a budget only when the
company calls it one. A model of the business carries its base case with an upside and a downside
beside it; the three differ in the levers they move, growth, pricing, hiring pace, churn, and a
narrower plan adds a scenario when a decision turns on an alternative future.

The method follows the evidence: trend, cohort, pipeline, usage, contract, or plain assumption.
When several estimates are defensible, recommend one, use it, and say what the alternatives would
change. A modeled goal is an editable assumption; only the company can make it a commitment. A
value that carries a unit is segmented by the dimension that carries it, pay by its currency, before
any total, so the top of a payroll broken down by currency is a converted figure at a stated rate or
the currency rows themselves. Definitions live beside the saved model.

## The coordinates of the business

The dimensions of a model are the coordinates along which the business actually differs: plan,
segment, region, and channel for a subscription business whose customers arrive in cohorts that
age; side, category, and geography for a marketplace; service line, client, and the people who
deliver for a services firm; SKU, channel, and location for a physical product. Each such
coordinate is a dimension, and a variable it changes is segmented by it; a flat company-level
number where the business varies underneath is a total with its explanation missing.

## Records and placeholders

The people, customers, and deals of a business are records: a table keyed by a stable identity, its
fields the facts a formula needs, with time series that read those fields. A customer table keyed
by customer carries plan, segment, region, start date, end date, and price, and revenue, churn, and
expansion per period are calculated from those fields, so an operator extends the business by
adding a row. A record the business does not yet have, a planned hire or an expected customer, is
a row like the rest, planned by its start date and editable fields until evidence makes it actual.

When no roster or customer list exists, placeholder records seed the tables and reconcile to the
totals the person gave: one placeholder employee per row with an ID, department, role, start date,
and pay; customers as cohort records, one per combination of the dimensions the business sells into.
Each is named the way the company names its own people and customers, by role or segment, as in a
`Team roster` holding `Software Engineer 2`, with its scope and currency.

## Controls and pages

Shared inputs live together on one page the operator reads top to bottom: growth, pricing, churn,
hiring pace, unit costs, payment terms. A control sits at the grain
where someone would change it, a churn rate per plan, a hiring start per role, and within a range
that makes business sense to move. Every calculated output traces to these controls, so moving one
changes the whole business; the forecast simulator that closes a build binds to this page, and the
upside and downside scenarios differ in these controls. One page per part of the business, in the
order a reader thinks about it, each page reusing the same variables.

## How the company earns

Revenue is modeled the way the company makes it. A recurring-revenue schedule shows opening
customers, additions, losses, ending customers, and their revenue; growth and churn are the
assumptions, and the customer movements are their calculated effect. Subscription cohorts reconcile
to recurring revenue, and a zero-growth or zero-churn assumption is disclosed as one. Transactions
and marketplaces follow their unit economics. Seasonality is evidence from the company's own
history or an assumption. Contracts, projects, and grants keep their real terms under the
company's recognition policy, so unsigned or conditional revenue stays contingent. Bookings,
recognized revenue, billing, and cash are four events with four timings.

A pipeline is the deals themselves, and a deal is open, won, or lost, with its amount meaning
something different in each state. A won amount is a booking and belongs in the month the deal
closed. An open amount weighted by its probability is an expectation and belongs in the month the
deal is expected to close. A lost amount is its own measure when someone asks. So bookings, open
pipeline, and lost deals are three quantities, each reading the deals in its state; a closed deal
contributes nothing to open pipeline. A deal count counts each deal once, in the forms under `count`
for a total and for a deal's row; deals without an amount are in the counts and out of the totals.
A pipeline model is a time series of bookings and pipeline with the deals beneath it, one row per deal with its stage, owner, amount, and dates, so
the register reconciles to the totals because it is the same deals; weighted open pipeline is the
two-rule drill-down under How a rule reaches a cell. The source's real classifications stay on the
deals, with stage first for open pipeline, and measures carry business names:
`HS_WEIGHTED_PIPELINE_IN_COMPANY_CURRENCY` is Weighted pipeline. The reader learns how many deals
have no owner and which stages were treated as won or open.

## What it costs

A cost is modeled at the scope where it happens and by the driver that moves it. Company accounting
policy decides what is cost of revenue and what is operating expense: cost of revenue delivers the
sold product or service, operating expenses develop, sell, and administer. Shared resources follow
an established allocation or a disclosed provisional split, counted once. Infrastructure and usage
costs follow their real driver, transactions, customers, compute, or storage, with vendor
baselines, tiers, floors, and effective dates kept. A percentage of revenue fits a cost that moves
with revenue; a usage, staffing, or unit driver fits a cost that moves with that thing. Ratios are
grounded in complete company periods, an editable assumption, or a labeled estimate, and staffing
steps, capacity, commitments, and pricing tiers are the reasons margins change.

## Who does the work

A headcount plan connects who is employed, from when to when, and on what pay, to how many people
there are each period and what they cost. It delivers the roster and the monthly table described
under Records and observations.

Headcount counts people: an employee contributes one while employed and zero otherwise, and
headcount adds those contributions, so roster rows are neither extra workers nor the bounds of
anyone's employment. Employee IDs are the identity, so two people with the same name stay two
people. By default a month's headcount counts anyone employed at any point in the month, so the
start month and the termination month both count and a null termination leaves the employment
open; a task that names another convention, such as month-end headcount, sets it. Employment dates
decide eligibility and proration within a month, and a termination date is a fact about the
person's last day whatever the status field says. Hires and departures that are not in the records
are planning assumptions.

Annual pay divided by twelve is a full month at unchanged pay; a monthly rate is a month's pay as it
stands; hourly pay needs a disclosed, editable hours assumption or stays visibly uncovered. Partial
months follow the employment dates and the stated proration basis. Loaded cost, variable pay,
taxes, benefits, comes from records or disclosed assumptions; a known unpaid founder is a real zero
where an unexplained zero is a coverage gap. Contractors, uncategorized departments, and unresolved
compensation stay visible as what they are, and person-level privacy holds in formulas, summaries,
and anything that leaves the workspace.

A bottom-up plan is the people: active employees, confirmed departures, committed starts, planned
hiring, and disclosed attrition, each planned worker with a stable identity and a department. A
narrower plan can hold editable role counts, costs, and dates with no HR connection. A demand-driven
plan separates required headcount, committed headcount, and the gap, paying committed workers and
hypothetical hires alike while keeping them apart. Capacity follows available working time and
allocations while payroll follows paid employment, and the two can differ in timing.

## Where cash lands

Collection and payment timing connect operating activity to cash. Working capital, inventory,
spending, debt, and financing follow commitments and disclosed assumptions, with real payment terms
and material customer concentration kept. A projected balance is an opening balance plus dated
movements; when opening cash is unknown, an editable opening assumption stands in, disclosed, and
the runway that follows is conditional on it. Negative cash stays visible, and committed financing
stays apart from a hypothetical raise.

A cash review answers how long the current plan can fund the business and which decisions could
change that answer. It establishes the cash balance's date, currency, restrictions, and the
forecast scenario, then reads the dated cash projection for the first period that crosses the
company's minimum cash requirement, or zero when the question is exhaustion; a forecast that never
crosses that level supports a statement about its covered horizon. The outlook connects to
collections, hiring, committed spending, and financing, showing which decisions have the largest
supported effect and when their cash impact begins. The delivery is the cash basis, the projected
constraint date or covered horizon, the consequential assumptions, and the decisions requiring
attention; a missing starting balance leaves a conditional answer whose missing inputs are named.

## Statements

A statement is the model's results arranged the way finance reads them. Its rows are measures,
revenue, cost of revenue, operating expenses, profit, assets, cash, each a variable the rest of the
model shares, and the statement shows how they relate for the requested entity, periods, currency,
and accounting basis. History comes from records or supplied facts; a missing historical input
stays unknown in the statement, and an editable assumption completes a forecast or a labeled draft.
Each reporting month is entirely actual or entirely forecast at its close boundary, and
month-to-date pacing lives apart from the statements.

A ledger arrives as accounts, dates, and amounts. The statement adds one fact per account: the
line it belongs to. That fact is a dimension mapped from Account, `Bucket`, kept in a mapping
table, accounts on rows and Bucket as the column, where a person changes a classification once.
One `edit_dimension_mappings` `map` call writes that classification and creates `Bucket` with the
mapped lines as its items when it does not exist yet; the receipt says whether the target was
created or reused. The `edit_table_blocks` create whose rows read `Bucket` declares the same
mapping in its `dimensions` entry (`target`, `keys`, `items`, `catch_all`), saved in the table's
own change before its rows resolve, so the classification and the statement are one call. The
calendar write names the scenario, not a table, so it rides in the same step. A line is one read
through it, `sum(Amount[Bucket = "Revenue"])`, and the same variable broken down by Account is the
line's account detail, each account at its own coordinate. Gross profit
and operating profit are arithmetic over the lines. Every value on the statement is one of those
two formulas, so it moves when the ledger does.

A measure that sums ledger records is a total of accounts, and it carries the ledger's own account
hierarchy beneath it in the same table, keeping a level when it groups. The reader opens a measure
and sees its accounts; the measure's classification keeps each account's coordinate, so an account
outside the measure reads empty and hides. A subtotal such as gross profit is arithmetic over other
measures and has no accounts of its own. Department, class, and other cross-cutting dimensions
belong in a column or a separate view when someone asks.

The imported accounting properties are the statement's starting measures, named for the source's own
classifications; separate classification properties carry account detail beneath them, and a single
amount property with an Account Type dimension supplies the statement groups through that dimension.
Uncertain classifications stay visible and editable in the mapping, and Unclassified imported
amounts are part of the source total. A requested check-in before the build presents the proposed
statement groups and the accounts feeding them and asks only the questions the source leaves open.

The profit relationships are formulas on the statement's rows: with positive expenses, Gross Profit
is Revenue minus Cost of Revenue, and Operating Profit is Gross Profit minus Operating Expenses, so
the statement stays right when source activity or a classification changes. The arithmetic follows
the imported sign convention, which the actual values and source reconciliation establish. Each row
has a rule at the model's base Date grain that covers the range it is meant for, a parent that is
its own formula, and quarter, half-year, and year values that add up the months.

A balance sheet is period-end values by the company's reporting groups, with every account that
carries postings, and `Total assets`, `Total liabilities`, and `Total equity` with their natural and
contra signs; its cash measure is what the cash-flow statement reconciles to. A source row's kind,
Data, Header, Section, says how the provider printed it that period: a Header can carry real
postings, and a Section is a presentation subtotal that never counts again. Total equity includes
every source equity component, Net Income and Retained Earnings properties included. Period-end
balances sum across accounts at the same snapshot date, and quarter, half-year, and year views use
the closing snapshot. Assets equal liabilities plus equity in every period in the report's signs; a
residual has a cause, and the answer names its amount, period, and accounts.

A cash-flow statement explains the change in cash, direct or indirect, and reconciles to the change
in the balance-sheet cash measure. The indirect method is a bridge from net income to cash, and
every other row either joins the bridge or stays out: cash itself and a retained-earnings movement
that repeats net income stay out, while distributions and financing movements count once. A balance
change can mix cash transactions with noncash movements, and the underlying transactions say what
the cash was and whether it was operating, investing, or financing. With only cash purchases of 20
and depreciation of 10, gross fixed assets rise by 20 and net by 10; purchases are the gross
increase, investing cash flow is −20, and the depreciation already in net income is an operating
addback, so deducting it from the gross movement as well would count it twice. The statement
delivers operating, investing, financing, and net cash flow and the opening balance or assumption
behind the first visible change, and its net cash flow equals the balance-sheet cash change.

## Comparisons and unusual results

A comparison puts a plan and the actuals for the same periods side by side and explains the
difference. The plan is the requested budget, saved plan, forecast, or upload; the actuals come from
accounting, CRM, workforce, billing, saved results, or supplied figures. Without an official plan,
budget performance is unknown, and a comparison against a labeled provisional plan is a what-if. The
comparison uses the existing measure definitions and mappings, keeps unmatched categories visible
outside the reconciled totals, and uses the latest complete period both sides cover, with a partial
comparison carrying its cutoff as a label. A plan can sit in Forecast while the actual for the same
Date sits in Actuals. Variance is actual minus plan,
and with a known nonzero plan the percentage is variance over the plan's absolute value; a zero plan
has an absolute variance only. Materiality and favorability are the company's. A saved comparison
is a native view with the customer's budget label, layout, dimensions, and formula-derived
differences.

An anomaly is a claim about calculated financial results against a baseline, and a company history
without an approved budget is a fine baseline. An all-clear is a real finding that covers only the
scope and evidence checked: totals and reconciliations need observed coverage of their periods and
members, and partial evidence supports a narrower finding while the rest stays open. Operating
expense and cost of revenue are the default scope of an unspecified spend check; a named scope is
the scope. Payroll cadence, non-operating entries, intercompany transfers, credits, and eliminations
each mean something different and are read as their own cases. The finding reports observed and
expected amounts, the date the data reaches, supported drivers, and hypotheses labeled as such; a
scheduled briefing compares with the last successful one and saves a dated page only when something
changed.

## What a finished model shows

A working model, saved, with editable assumptions and formula-linked outputs across the horizon and
the business's dimensions, showing the business in every row across the horizon. Its scenarios
differ in inputs. Its record tables count into its totals: additive totals reconcile across one
population, parents compute at their own grain, and gross profit and margin respond to the saved
drivers. Its simulator and walkthrough exist. The roster reads the latest values and the monthly
table per-period values, department and company figures agree with the employee rows in every
period, and every payroll figure has one currency basis. The three deal states are separate
measures, each deal counted once, placed by the date that governs its state, with the register
reconciling to the totals. The requested statements are complete and scoped to the right entities
and periods, every account maps once or is explicitly excluded, and each statement reconciles to
its source or the answer names the remaining residual at the account level. Cash and liquidity
claims keep their opening-balance basis and tell factual, forecast, and hypothetical values apart.

The explanation names the method, the consequential assumptions, the commitments, the evidence
gaps, and the risks: the conventions chosen, who is uncovered, which rows the source never had, and,
for a restated source, that a record's repeated rows are one picture of that record per sync, so a
period's history is the observation stamped in it rather than a count of rows.
The saved result is linked in the turn that built it, and an explicitly requested static snapshot
keeps that form.

The generated reference below is the complete function inventory and the authority for every
accepted spelling; its notes carry the places the engine differs from Excel.

<!-- formula-syntax:start -->
<!-- prettier-ignore-start -->

<!--
Code generated by scripts/generate-formula-functions.py. DO NOT EDIT.
Sources: shared/formula-functions.json, antlr/modelingv3/*Lexer.g4,
shared/grammar-reference-overlay.json. Every example below is parsed by
scripts/verify_grammar_reference.py before this section is written.
-->

## Formula syntax reference

This is the syntax accepted by the parser. The preceding sections explain
model authoring and formula behavior.

### Formula shapes

Each formula string uses one of these entry points. Match it to the field you are filling.

| Shape | Written in | Example |
| --- | --- | --- |
| Expression | formula bodies: variable formulas, defaults, rollups | `Revenue - COGS` |
| Condition | the condition half of a formula write | `[]` |
| Dimension predicate | one predicate in a predicate editor | `Region = "West"` |
| Predicate value | the operator and right side beside an already-chosen dimension | `in {"East", "West"}` |
| Reference | a single variable or dimension reference with no surrounding arithmetic | `Revenue$[Region = "West"]` |
| Formula address | reading and matching one variable's rules; explicit formula writes build this shape from their target, grain, period, and segments | `Revenue[]` |
| Breakdown | readable and internal breakdown text; table and Canvas requests use the typed breakdown array rather than this bracket string | `[Department, {Level[Department = "Engineering"], Owner}]` |
| Predicate list | a field that holds several predicates at once, with no brackets around them | `Region = "West", Department = "Engineering"` |

### Reference forms

How a formula names a variable, a dimension, or an external column.

| Form | Meaning |
| --- | --- |
| `Revenue` | Shorthand for `this.Revenue` or `Revenue[]` |
| `` `Net Revenue` `` | Backtick name, for spaces, punctuation, or a reserved word |
| `Revenue#a3f` | Identity suffix from an inspection, when several properties share one name; any longer prefix of the same identity also resolves |
| `Revenue$` | Absolute reference; does not inherit the current segment |
| `this.Revenue` | Within a `where` clause, performs a lookup or segment access at the current cell's coordinate |
| `Date.Day … Date.Year` | Granularity pin: `.Day` `.Week` `.Month` `.Quarter` `.Half` `.Year`, exact case |
| `Revenue[Region = "West"]` | Bracket lookup: dimension predicates, or a date offset |
| `<runway:properties/UUID>` | Raw property URI. Storage form; read it, but author readable names |
| `<runway:exttables/550e8400-e29b-41d4-a716-446655440000/columns/Amount>` | External-table column. Plain reference only: no postfix `$`, `[…]`, or dot after the closing `>` |
| `// note` | Comments: `//` to end of line, `/* … */` across lines |

### Table layouts and breakdowns

Tables and Canvas datasets declare visible `rows` and `columns` through the tool input schemas. The bracketed forms below are readable and internal breakdown notation; table requests use typed layout fields. Window and comparison remain separate fields.

| Form | Meaning |
| --- | --- |
| `[Department]` | Split by one dimension: one row per Department item |
| `` [`Spend Type`] `` | Backticks quote a name with spaces; written bare, the parse stops at the second word |
| `[Department, Level]` | A comma descends: Levels nest under each Department |
| `[Department, {Level, Owner}]` | Braces fork: Level and Owner sit side by side under each Department, and whatever follows the brace applies to every branch |
| `Level[Department = "Engineering"]` | A branch condition routes a term under ONE branch of a dimension above it — Engineering gets Levels while other branches do not |
| `Region in {East, West}` | An item filter keeps only the listed items of the term's own dimension |
| `[Date.Month]` | A grain buckets a date dimension; Day, Week, Month, Quarter, Half, or Year |
| `{Level, [Owner, Date.Month]}` | A branch that descends is bracketed: Owner splits by month while Level does not |

### Keywords

Case is exact where noted; the rest are case-insensitive.

| Keyword | Case | Where it is legal |
| --- | --- | --- |
| `this` | insensitive | Left hand side of a period. Pins a reference to a variable or dimension to the current cell's coordinate. Useful for disambiguation in `where` clauses. |
| `any` | insensitive | Wildcard predicate: `Dim in any` spans that one dimension |
| `where` | insensitive | Filters a predicate or a reference by a full expression |
| `null` | insensitive | The empty value |

### Literals

| Literal | Forms |
| --- | --- |
| `42, 3.14, .5` | Numbers; a leading dot is legal |
| `"text" or 'text'` | Either quote style; escapes `\\` `\'` `\"` `` \` `` `\n` `\t` `\r` `\uXXXX` |

### Operators

Use the preferred spelling. Accepted spellings still parse in stored formulas.

| Group | Preferred | Accepted, do not write |
| --- | --- | --- |
| Arithmetic | `+` `-` `*` `/` `%` `^` | `**` |
| Comparison | `=` `<>` `<` `<=` `>` `>=` | `==` `!=` |
| Logical | `and` `or` `not` |  |
| Membership | `in` |  |

- `%`: between two values `%` is modulo: `Months % 12`, `100 % 3`, `Months % -1`; after a number, `% -` is the number's percent and then a subtraction: `75% - Fee` is `0.75 - Fee`; a number's percent is one value: `50%` is `0.5`, `2 ^ 50%` is `2 ^ 0.5`; any other value followed by `%` and nothing else is its percent: `(a + b)%`
- `in`: negate as `not in`

Precedence, highest first: `( ), f( ), 5%; -x; ^ (right-associative, unlike Excel); x%; * / %; + -; < <= > >= = <>; in, not in; not; and; or`.

### Grammar

The productions the parser actually uses, renamed for reading. Terminals are shown as the text you write.

```ebnf
expression         = value
                   | "not" expression
                   | expression "and" expression
                   | expression "or" expression
value              = "(" expression ")"
                   | function "(" (expression ("," expression)*)? ")"
                   | "-" value
                   | value "^" value
                   | value "%"
                   | value ("%" | "*" | "/") value
                   | value ("+" | "-") value
                   | value ( "<" | "<=" | ">" | ">=" | "=" | "<>" ) value
                   | value "not"? "in" membership
                   | "this" "." name (dot-chain ("where" expression)?)?
                   | "$"? "[" predicate ("," predicate)* "]" dot-chain
                   | offset
                   | reference
                   | number "%"
                   | number
                   | "null"
                   | string
                   | set
reference          = name "$"? (dot-chain ("where" expression)?)?
                   | "<runway:exttables/" table-id "/columns/" column-name ">"
                   | name "$"? lookup-suffix dot-chain? ( "where" expression )?
lookup-suffix      = "[" predicates? "]" (name? offset)?
                   | offset
condition          = "$"? "[" (predicate ("," predicate)*)? "]"
formula-address    = name condition
predicates         = predicate ("," predicate)*
predicate          = name dot-chain? predicate-value
predicate-value    = ( "=" | "<>" | "<" | "<=" | ">" | ">=" ) value
                   | "not"? "in" "any"
                   | "not"? "in" membership
                   | "where" expression
membership         = range
                   | set
range              = value ":" value
offset             = "[" expression "]"
dot-chain          = "." name ("." name)*
set                = "{" "}"
                   | "{" expression ("," expression)* "}"
name               = (identifier | backtick-name | granularity) "#hex"?
breakdown          = "[" breakdown-sequence "]"
breakdown-sequence = breakdown-node ("," breakdown-node)*
breakdown-node     = breakdown-term
                   | fork
fork               = "{" branch ("," branch)+ "}"
branch             = breakdown-term
                   | "[" breakdown-term ("," breakdown-node)* "]"
breakdown-term     = name dot-chain? branch-condition? item-filter?
branch-condition   = "[" predicate "]"
item-filter        = "not"? "in" membership
                   | "where" expression
```

`name` also takes the granularity suffix (`Date.Month`) shown under Reference forms.

### Expression composition

The Grammar productions are the whole truth about nesting; these are the
consequences that are easy to read past.

- Function arguments are full expressions, so lookups, nested calls, and bare `and`/`or` are all legal inside `f( … )`.
- **Bracket fallthrough.** A bracket that matches no predicate shape still parses as a date offset. `Revenue[Flag = (Active and Open)]` is a predicate lookup, while `Revenue[Flag = Active and Open]` becomes an offset expression. A clean parse does not prove the bracket means what you intended.
- A formula address is a variable followed by one matching condition, such as `Revenue[]` or `Revenue$[Date.Month in any]`. It may cover one cell or many. Preserve the condition's matching mode, range, and variant when editing a saved rule. The condition carries its own brackets; a second bracket group is a parse error, not a second term group.
- **Dimension items.** `$[Location = "East Coast"].Location` is a dimension-item literal: it pins one dimension to a string and looks that dimension back up, evaluating to the pinned value. Prefer this notation over a bare string whenever it could be ambiguous whether a dimension item is intended (comparisons like `this.Location = …`, values fed to filters); a bare `"East Coast"` is just text, while the literal names its dimension. Write it with the `$`: without it the lookup also inherits the current row's other dimensions and can return a different item depending on the row. Looking up a different property (`$[Location = "East Coast"].Revenue`) evaluates as that property's lookup at the pinned segment, like `Revenue$[Location = "East Coast"]`.
- **Mapped dimensions read like values.** When a dimension's items come from a mapping (a lookup keyed by other dimensions), a lookup of it evaluates to its mapped item: `Bucket$[Account = "5001"]` is that account's bucket. Dot access follows the mapping keys — `this.Bucket` is the current segment's mapped item when the segment carries the keys. A field column reads the dimension at the table's axes the same way: beside Deal rows, a `Status` mapped from Stage has no rule and shows its catch-all on every row, and a text variable `Status$[Stage = Stage]` reads each deal's status through its own stage. Each dot hop desugars to a lookup keyed by the previous value, so `Account.Bucket` is `Bucket$[Account = this.Account]`, and hops keep going where mappings chain. The $ prevents unrelated dimensions such as Month from being inherited into this lookup. In a predicate the mapping runs the other way: `sum(Amount[Bucket = "COGS"])` gathers every source row whose account the mapping puts in COGS. A predicate on Bucket reads an authored variable at a grain that includes Bucket rather than filtering it; roll such a variable up through its key, `Amount$[Account.Bucket = this.Bucket]`.

| Formula | Composes |
| --- | --- |
| `sum(Revenue[Department in any])` | lookup inside a call |
| `Revenue / sum(Revenue$[Date.Month = this.Date.Month, Region in any])` | share of total for the current Month: absolute read retains Month and opens Region |
| `(Revenue - Revenue[-1]) / Revenue[-1]` | date offsets in arithmetic |
| `if(Tier in 1:3, 1, 0)` | range membership |
| `average(Revenue[Date.Month in [-3]:[-1]])` | trailing window: relative date range + aggregate |
| `sum(Amount[Date.Day where weekday(Date.Day, 2) <= 5])` | inner where with a transformed candidate value |
| `` `Cash Balance`[-1] + `Net Cash Flow` `` | recurrence: prior-period reference and backticks |

### Functions

Use the preferred name. Accepted aliases still parse in stored formulas.

#### Aggregation

- `sum(value: number, value2...: number) -> number`: Takes 1 to 10 arguments. Empty fan-out: 0.
- `product(value: number, value2...: number) -> number`: Takes 1 to 10 arguments. Empty fan-out: 0.
- `min(value: number | date, value2...: number) -> number | date | null`: Takes 1 to 10 arguments. Empty input: null. One date or datetime argument returns the earliest; several arguments must be numbers.
- `max(value: number | date, value2...: number) -> number | date | null`: Takes 1 to 10 arguments. Empty input: null. One date or datetime argument returns the latest; several arguments must be numbers.
- `average(value: number, value2...: number) -> number | null`: Takes 1 to 10 arguments. Empty input: null. *(accepted: `avg`)*
- `count(value: any, value2...: any) -> number`: Takes 1 to 10 arguments. Counts non-empty values in one or more expressions. It works like Excel COUNTA, not Excel COUNT. A bracketed read keeps each dimension it filters on, and its condition on a dimension takes the place of the row's item of that dimension. So `count(Deal[Region in {"East", "West"}])` counts each deal once for every one of those Region items its records carry, and on a deal's row it counts that deal's items among them. `count(Deal$[Deal where count(Deal[Deal = Deal, Region in {"East", "West"}]) > 0])` counts each deal once, and its condition on Deal gives every deal's row that same total. On a deal's row, `if(count(Deal[Region in {"East", "West"}]) > 0, 1, 0)` is 1 for a deal with records in East or West and 0 for any other.
- `first(value: T, source2...: T) -> T | null`: Returns the first nonnull value in series order. With multiple arguments, reduces each argument with FIRST and selects the first nonnull result from left to right. Dates order values within each argument only. Errors are not null.
- `last(value: T, source2...: T) -> T | null`: Returns the last nonnull value in series order. With multiple arguments, reduces each argument with LAST and selects the first nonnull result from right to left. Dates order values within each argument only. Errors are not null.
- `stdev(value: number, value2...: number) -> number | null`: Takes 1 to 10 arguments. Returns sample standard deviation (ddof=1) pooled across one or more numeric expressions, or null with fewer than two values. *(accepted: `stddev`)*
- `var(value: number, value2...: number) -> number | null`: Takes 1 to 10 arguments. Returns sample variance (ddof=1) pooled across one or more numeric expressions, or null with fewer than two values. *(accepted: `variance`)*
- `median(value: number) -> number | null`: Empty input: null.

#### Math

- `abs(value: number) -> number`
- `sqrt(value: number) -> number`
- `ceiling(number: number, significance: number) -> number`: Excel CEILING semantics. `ceiling(UnitPrice, 0.05)`
- `floor(number: number, significance: number) -> number`: Excel FLOOR semantics. `floor(UnitPrice, 0.05)`
- `round(number: number, num_digits: number) -> number`: Excel ROUND semantics. `round(Forecast, 2)`

#### Conditional

- `if(condition: any, whenTrue: T, whenFalse: U) -> T | U`: Uses condition truthiness. `if(Revenue > 100, "high", "low")`
- `iferror(value: T, value_if_error: U) -> T | U`: Catches cell errors, including division by zero. Null is not an error and stays null. `iferror(datedif(StartDate, Date, "D"), 0)`

#### Date parts

- `year(date: date) -> number`: Excel YEAR semantics.
- `month(date: date) -> number`: 1–12; Excel MONTH semantics.
- `day(date: date) -> number`: 1–31; Excel DAY semantics.
- `quarter(date: date) -> number`: Calendar quarter, 1–4.
- `half(date: date) -> number`: 1 for January–June; 2 for July–December; 2 for null.
- `weekday(date: date, return_type?: number) -> number`: Excel WEEKDAY semantics, Sunday-based by default.
- `weeknum(date: date, return_type?: number) -> number`: Excel WEEKNUM semantics, Sunday-based by default.
- `isoweeknum(date: date) -> number`: ISO 8601; Excel ISOWEEKNUM, equivalent to WEEKNUM(date, 21).

#### Date truncation

- `startofday(date: date) -> date`
- `startofweek(date: date) -> date`: Monday-based.
- `startofmonth(date: date) -> date`
- `startofquarter(date: date) -> date`
- `startofhalf(date: date) -> date`
- `startofyear(date: date) -> date`

#### Date arithmetic and construction

- `dateadd(date: date, amount: number, unit: string) -> date`: Shifts a date like Power BI's dateadd, with additional half-year support and quoted unit names. `dateadd(Date.Month, -1, "month")`
- `datedif(start_date: date, end_date: date, unit: string) -> number`: Returns the difference between two dates in the requested unit, matching Excel's DATEDIF. Excludes the end day, so an inclusive overlap adds one. `datedif(StartDate, Date, "M")`
- `daysin(date: date, unit: string) -> number`: Returns the number of days in the unit-aligned period containing the date. `daysin(Date, "month")`
- `date(year: number, month: number, day: number) -> date`: Builds a date from year, month, and day numbers, matching Excel's DATE including month and day rollover, on the real calendar: 1900-02-29 does not exist. `date(2026, 1, 31)`
- `edate(date: date, months: number) -> date`: Shifts a date by whole months with the day clamped to the target month's length, matching Excel's EDATE. `edate(Date, -1)`
- `eomonth(date: date, months: number) -> date`: Returns the last day of the month a signed number of months from a date, matching Excel's EOMONTH. `eomonth(Date, 0)`

<!-- prettier-ignore-end -->
<!-- formula-syntax:end -->
