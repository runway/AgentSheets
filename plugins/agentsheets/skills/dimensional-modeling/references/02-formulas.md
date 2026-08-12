# Formulas: the calculus

How every number is computed. Builds on the axioms in SKILL.md and the vocabulary of
references/01-the-dimensional-universe.md (segment, grain, item).

## 2.1 What a formula is

A formula is a triple attached to a target variable:

```
(target property, condition, expression)
```

plus an optional **formula range** (the formula only applies inside a time
window, section 2.7).

The condition says where in the space the formula applies. The expression
says what to compute there. Because they are separate, one variable can
carry a general rule plus any number of targeted overrides without the
rules fighting.

## 2.2 Conditions and dispatch

**Dispatch** is how the engine picks, for every cell, the one formula that
computes it: among the variable's formulas whose conditions match the cell,
the most specific condition wins.

A condition is one bracketed group. Its shape says which grains it claims:

1. `[]` names no dimensions, so it constrains nothing and applies at every
   grain. This is the whole-model default.
2. `[Department = "East", Region in any]` claims a shape. It applies at every
   grain containing {Department, Region}, including richer grains added by
   drill-ins — and within each matching grain, to the cells where Department
   is East: every Region item, and every item of any dimension the condition
   does not name.
3. `$[Department = "East", Region in any]` claims a place. It applies only
   when the grain is exactly {Department, Region}: the named dimensions, no
   more, no fewer — and within it, to the cells where Department is East,
   across every Region item.
4. `$[]` names only the empty segmentation, not the whole model.

Use `[…]` for a rule about named dimensions wherever they are broken out. Use
`$[…]` for a deliberate grain-lock: a cell pin meant to stop when the shape
changes, a condition pasted back from a read, or a structural mapping row.
Existing formulas may use the exact spelling; read them fluently and preserve
the condition an edit addresses. The coordinate bounds (`segments`, `grain`,
`period`) always compose an exact `$[…]` of the named coordinates; a subset
rule that should survive drill-ins is written through `condition`.

Each term constrains one dimension: a literal (`"East"`), a set
(`{"Eng","Product"}` or `NOT {"Eng"}`), the wildcard `ANY`, a predicate
(`Month where Month >= "2025-02-01"`), or a computed value. The constraints
choose items, never grains: `[Department = "East"]` and `[Department in any]`
match exactly the same grains — only the dimension set decides the match. At
a richer matching grain, the dimensions a subset condition does not name
behave as `in any` — the same openness an exact completion writes out
explicitly.

A term naming a date-typed dimension other than the system Date (a signup
cohort, a contract-start month) is granularity-pinned: `Cohort.Month` matches
exactly the views segmented by Cohort at Month, and nothing else. At any other
granularity the condition does not apply, so a formula carrying it goes quiet
when the view re-buckets that dimension. Rollup methods are system-Date-only.
Which date axis a view descends is SKILL.md axiom 13; on a view whose sole
date-typed dimension carries that descent, a rule that omits the dimension
descends it the way a Date-less rule descends the timeline.

**The dispatch order.** When the engine needs a value for (variable, grain),
it gathers candidate formulas and sorts them: term lists beat `[]`; among
equals, more pinned terms beat fewer; tighter constraints beat looser; and
recency breaks any remaining tie. It then walks the list in order. Each
formula claims the _overlap_ between its condition and whatever
space is still unclaimed, and the rest falls to lower-priority formulas. So
a formula pinned to East masks the default exactly on East, and the default
fills everything else.

An **aggregate row** is a cell whose grain omits dimensions shown deeper in
the table (parent and total rows). Its address pins only its own path's
dimensions, and drilling a child dimension in adds no terms to the parent's
condition.

The sigil decides what happens when the grain changes. Drill Product in under
Region and a subset `[Region = "East"]` override keeps applying to the finer
{Region, Product} rows; an exact `$[Region = "East"]` override stops and those
rows fall through. The first is a rule about Region wherever it is broken out;
the second is a pin at exactly the Region grain. If an exact rule should have
followed the drill-in, rewrite that intent as a subset condition and verify
with a data read. If it was a deliberate cell pin, its stopping is correct.

## 2.3 The expression language

Formulas are authored in readable form (entry names) and stored in a
canonical form (URIs). Names with spaces are backticked; ambiguous names take
a short id suffix (`Revenue#a3f`); a granularity (monthly vs quarterly)
suffix picks a time grain (`Date.Month`). Everything below uses readable form.

Scalars and operators: numbers, strings, NULL, `+ - * / %`, power `^`,
comparisons (`= <> < <= > >=`), `AND OR NOT`, comments (`// ...`, `/* ... */`).
Author the Excel spellings `^`, `<>`, `=`; the legacy `**`, `!=`, and `==`
remain accepted in stored formulas. Power chains are right-associative
(`2 ^ 3 ^ 2` is `2 ^ (3 ^ 2)`), unlike Excel, so parenthesize chained powers
when porting a workbook. A trailing `%` is a percent literal (`50%` evaluates
as 0.5); binary `%` between two operands stays modulo (`Months % 12`).
Dividing by zero (or modulo by zero, or raising 0 to a nonpositive power)
produces a cell error, like Excel's `#DIV/0!`; catch it with `iferror` when
you want a fallback value. A null numerator propagates null instead.
Functions:

- aggregates: `sum, product, min, max, average, count, first, last, median,
stdev, var` (the legacy spellings `avg`, `stddev`, `variance` still parse;
  do not author them). `count` counts non-empty values of any type, like
  Excel COUNTA.
- math: `abs, sqrt`; the two-argument roundings `ceiling(x, significance)`,
  `floor(x, significance)`, and `round(x, digits)` — the second argument is
  required.
- dates: `startOfDay/Week/Month/Quarter/Half/Year`; `dateadd(date, n, "month")`
  with units day, week, month, quarter, half, year (like Power BI's `dateadd`,
  it shifts a single date value, not a date column); `datedif(start, end, "M")`;
  `date(year, month, day)`; `edate(date, n)` and `eomonth(date, n)` with
  Excel semantics; the date parts `year, month, day, quarter, half, weekday,
weeknum, isoweeknum`; `daysin(date, "month")`.
- control: `if(condition, then, else)` (exactly three arguments) and
  `iferror(value, fallback)`. `iferror` catches divide-by-zero and every
  other cell error; an uncaught error propagates through arithmetic and
  aggregation like any other error cell.

This list is the complete callable set. A name outside it is rejected as
"Unknown function"; no other catalog adds functions —
[[build-model:references/grammar-reference.md]] gives the signatures.

### References

**The relativity law: plain references inherit the current segment.** Inside
a formula being evaluated at {Department: Eng, Month = 2026-01}, the reference
`Revenue` means Revenue at exactly that segment. This single rule is why
one formula text is correct at every grain.

Do not read that as the whole formula being correct at every grain. Relativity
governs the expression; the condition decides where the expression runs at all
(§2.2). A source sum, a ratio, or plain arithmetic over other variables
re-derives correctly at whatever grain it is evaluated — but under a term-list
condition its reach depends on the sigil: `$[…]` evaluates only at the named
grain, `[…]` at every grain containing the named dimensions, and `[]`
everywhere. So when a drilled row shows zeros, the expression is usually
already right and only its address is wrong:
check the condition before rewriting the formula (references/09-the-layer-model.md
§9.6 tests which expressions belong in the default).

Brackets re-aim chosen dimensions while inheriting the rest:

```
Revenue[Region = "East"]        this segment, but Region forced to East
Revenue[Region in any]           fan out over every region (wrap in an aggregate)
Revenue[Region in {"E","W"}]     a subset; NOT {...} excludes
Revenue[Region where <pred>]  items passing a predicate; inside it, a bare
                               dimension name means the candidate item's value
Revenue[Dim = <expr>]           a computed coordinate, e.g.
                               Salary[Employee = this.Manager]
```

**The collapse-order law: a plain reference is one already-rolled-up
number.** By the time a formula evaluates, a plain variable reference has
collapsed to the slice's rollup — the rows are gone. An aggregate wrapped
around it therefore operates on a single value: `sum` passes it through
harmlessly, `average`/`min`/`max` return it unchanged, `count` returns 1
for a non-null value and 0 for a blank.
Row-level aggregation lives in the variable's aggregation setting, in a raw
source-column reference (the one reference kind that still sees rows), or
in an explicit fan-out (a bracket with ANY or WHERE names the rows) — never
in an aggregate around a plain variable reference (anti-example in section
2.9).

`this.X` reads the current segment's own item on dimension X
(the month we are in, the department this row is about). A dotted chain hops
the space: `this.Employee.Department` is "the department of this
segment's employee".

`$`, written after the name, makes a reference **absolute**: `Revenue$` is the
whole-model total no matter where it is used; `Revenue$[Region = "East"]` binds
exactly the listed dimensions and inherits nothing. Use `$` when you genuinely
want to escape the current context (a share-of-total denominator, a global
threshold).

Time shorthand: a leading bracket term with no dimension name binds the
system Date dimension at the current grain. `Revenue[-1]` is the previous
period (whatever the grain's granularity is); `Revenue["2025-01"]` pins a
month. If the current grain has no time dimension, these evaluate to null
rather than erroring.

An outer `where` filters a fanned-out aggregate by a per-item predicate:

```
sum(Sales$[Employee in any] where Salary$[Employee = Employee] > 100000)
```

Two hard restrictions: references to raw source columns accept no bracket
constraints (slice through dimensions instead), and a dimension cannot
constrain itself (`M[M = ...]` is rejected).

Missing segments are nulls, not errors: looking up a segment that does not
exist yields null, and `sum`/`product`/`count` over an empty fan-out give 0
while every other aggregate gives null.

## 2.4 Evaluation: one text, many grains

The same expression text is _re-bound at every grain it is asked at_. At
binding time the relativity law expands (`Revenue` becomes "Revenue at this
segment's coordinates"), offsets pick their unit from the grain's
granularity, and the implicit date term resolves. This is what enforces the
recompute law from the axioms in SKILL.md:

> A coarser value is the formula evaluated at the coarser grain. Nothing ever
> adds up child cells that were already computed for display.

Concretely: `Margin = Profit / Revenue` at the Region grain divides
Region-grain Profit by Region-grain Revenue. Drill Margin in by Product and
each product row is its own ratio; the Region parent stays the recomputed
Region ratio. There is no code path that averages the children.

The one exception is the time axis. For a variable with any aggregation
function except do-not-aggregate (sum, min, max, first, last, any, count),
the engine synthesizes a _formula_ such as "Month = sum of my Days" and
dispatches it like any other formula. It is still grain-bound evaluation,
it never applies to ratios, and it is skipped for do-not-aggregate variables
entirely (their coarser time cells come from their own formulas, or from
the synthesized defaults). Details in references/04-time.md.

## 2.5 Recurrence and cycles

A formula may reference its own variable through a time offset:

```
`Running Total` = `Running Total`[-1] + Input
```

That is a recurrence, evaluated cell by cell along the time axis. It is the
standard way to build running totals, balances, and cohort carryovers. A
missing prior period follows §2.3 — the lookup yields null — and a null
prior enters the chain's addition as zero, so the chain starts at zero
plus the first change. To start it anywhere else, write one exact seed cell
(`set_values` with a condition such as `$[Date.Month = "2026-01"]`) on the
first month the chain evaluates: the first month of its scoped range for a
windowed chain (a forecast chain seeds the first forecast month), or of the
table's window for an unscoped one. For one chain on the unsegmented time row,
write its rule at `$[Date.Month in any]`; the exact sigil intentionally stops
that chain when a drill-in adds a dimension (references/limitations.md §7). Use `[Date.Month in any]` only
when the model means an independent chain in every segmentation containing
Date, and seed each intended segment at its own starting coordinate or
deliberately let its missing prior start at zero. A `count()` guard on the
prior period computes and changes none of these laws
([[build-model:references/formula-grammar.md]], recurrence shape 1).
Mutual references between variables are also fine as long as some time offset
breaks the loop. A genuine cycle at the same time period has no defined
value: those cells become "Circular reference" errors while the rest of the
table computes normally.

## 2.6 Errors

Three failure tiers, from author-time to run-time:

1. **Syntax errors** reject the formula at validation ("missing '(' at ...").
2. **Semantic errors** reject specific constructs ("Unknown function", "IF
   requires exactly 3 arguments", "Reflexive dot-notation chain is not supported").
3. **Evaluation errors** poison cells, not tables. An erroring cell carries
   its message plus a trace back through the dependency chain to the formula
   and segment that caused it, and the error short-circuits through any
   expression that consumes the cell. Users see #ERR with a tooltip.

One trap shows no error at all: a variable quietly starts showing raw-sum or
zero behavior. That is what a _persisted_ formula that no longer parses
looks like — it is silently skipped at planning time and the variable falls
back to its defaults. Check whether the formula still parses; the known
cause is a condition that predates the bracketed notation, such as the
`<uri>: ANY` spelling — rewrite that formula at the right layer. Conditions
are bracketed now: `[]`, `[terms]`, `$[terms]`, `$[]` and nothing else.

## 2.7 Formula ranges: actuals vs forecast

A **formula range** is a named time window that scopes formulas. The system
ships two, split at the **Last close** date ref: **Actuals** (up to last
close) and **Forecast** (after it). Attaching a formula to a range inlines
the window into its condition, so range dispatch is ordinary formula
dispatch over date-sliced conditions. The window inlines against the
condition's Date term, so a range-scoped formula must carry one — `Date = …` or
`Date.Month = …`. On a date-less condition (a plain `[]`, which is
complete on its own, or one segmented only by non-Date dimensions), the range
write is rejected outright; a `period` bound avoids this by injecting the
Date term itself. The usual pattern for a modeled variable is one
date-scoped formula per window:

```
Actuals:   sum of the source column        (what happened)
Forecast:  a projection you author         (what you predict)
```

Defaults when the author has not written one: the Actuals window falls back
to the source-column sum (or 0 with no source); the Forecast window defaults
to 0. That default is why new variables read as zero in future months until
someone writes a forecast. Last close is workspace-global, manually moved,
and each scenario has its own (references/04-time.md and
references/05-scenarios-and-comparisons.md).

## 2.8 Ephemeral formulas

An agent can evaluate formulas without persisting anything: define named
request-local variables and reference them as `@name` inside another request
formula's expression — the only place `@name` means anything. Ephemeral
formulas behave exactly like saved variables (they recompute per grain, they
can reference each other and take bracket constraints like
`stdev(@margin[Date.Month in {...}])`), but they accept only the
`[]` condition — structurally: an `expressions` entry has exactly
`name` and `expression`, no condition field to pass. To pre-test a
grain-, period-, or segment-scoped rule, use `change.set_values` with
`dry_run` instead: it runs the real write's full validation (condition
lowering, window resolution, semantic check) and persists nothing — but
computes no values. Nothing computes an unsaved rule at a non-`[]`
condition, so the loop is: dry-run for validity, write, verify values
from the readback, and repair by `formula_id` if wrong.
This is the preferred tool for exploration and for any computed answer: run
the engine rather than doing arithmetic on returned cells.

The tool is `inspect_variables` `ask.try_formulas`. It takes `expressions`, a
list of `{name, expression}` for what the model does not hold; `from`/`to`,
the periods in scope (both required); optionally `rows_by`, ONE dimension
whose items each get their own number — never Date, which is refused because
the periods are already the columns — written bare, `Spend Type`, not
`` `Spend Type` ``, because a field holding one name has no parser to satisfy;
and `max_rows`. Periods are always the columns, at the model's base grain,
so the answer comes back per period without asking. No layout: the question is
enough. The expression is ordinary formula text: reference a saved variable by
its plain name (`Revenue`, never `Revenue()` — there are no zero-arg calls),
and another request formula by `@name`. To read a saved variable as the model
computes it, use `inspect_model_views` `ask.calculate` with a `window` — an
expression that merely mentions it runs as one all-time rule recomputing
everywhere, a fair calculation but not what the model says, and the tool
refuses that shape. To compute a margin the model does not hold, saving
nothing:

```
expressions: [{ name = "margin", expression = "Revenue - `Cost of Revenue`" }]
from: 2026-01-01
to:   2026-12-31
```

The engine mints `@margin` as a throwaway do-not-aggregate variable, computes
the table (so `margin` recomputes at each grain, never a cell-sum), and
discards it when the request ends. Each record carries `ephemeral_variable`
and resolved `segments`; `computed` restates the grid, and a 0 no formula
reached carries a `note` saying so.

One instrument that does not exist: a formula delete. `change.operations`
`delete` removes the entire variable (and is refused while anything
references it); a blank expression is rejected per item, never stored. To
neutralize an override, update it via `formula_id` to compute what the
layer beneath would have produced — or tell the user a true delete is a
product action, where deleting a variable's `[]` formula also
resets its aggregation to SUM (references/10-deviations.md D3).

## 2.9 Pattern gallery

```
Gross Margin        Revenue - `Cost of Revenue`
Gross Margin %      (Revenue - `Cost of Revenue`) / Revenue
MoM growth          (Revenue - Revenue[-1]) / Revenue[-1]
Prior year          Revenue[Date.Month = dateadd(this.Date.Month, -12, "month")]
Subtotal over dim   sum(Headcount[Department in any])
Filtered aggregate  sum(Revenue[Region not in {"Internal"}])
Share of total      Revenue / Revenue$
Month from days     average(Amount[Date.Day where startOfMonth(Date.Day) = this.Date.Month])
Running total       `Running Total`[-1] + Input    (seed the first month; see §2.5)
Guarded ratio       if(Revenue = 0, 0, Costs / Revenue)
```

The most tempting mistake: `average(Amount)` for an "Average Deal Size"
variable. The cell silently displays the rollup — the sum — under an
"average" name, because by the collapse-order law (section 2.3) the
reference is already the slice total. The correct forms: an aggregate
over the raw source column (`average(<source column>)` — source-column
references still see rows), the "Month from days" fan-out above, or a ratio
the engine recomputes per grain (``Amount / `Deal Count```). There is no AVG
aggregation setting; min/max/count/first/last have settings and need no
formula at all. Recipes for every variable shape, indexed by the ask:
references/13-metric-recipes.md.

Choosing between the two time self-references in a persisted formula:
`[-n]` counts in the evaluating grain's granularity, so a saved
`Revenue[-12]` silently means twelve quarters back once a view rolls up to
quarters. Pin the calendar meaning with an explicit offset
(`dateadd(this.Date.Month, -12, "month")`) whenever it must survive
re-graining; keep `[-n]` for genuinely grain-relative logic.

Authoring discipline for the write tools: author in readable form and let
the write tool lower both expression and condition. Canonical condition
order is property-UUID order with granularity in the URI query; never write
it by hand. Let `segments` plus the block derive coordinate writes; the
bounds always compose an exact `$[…]`. Write `condition` yourself when the
sigil carries intent: `[…]` for a shape that survives richer grains, `$[…]`
for an exact place. Omitting formula_id is the normal case: a write matches the formula
already at that address — same variable, predicates, and window, regardless
of sigil for a nonempty predicate list — and updates it at its stored
condition, creating one only when nothing matches. `[]` and `$[]` remain
different zero-dimension addresses. So writing the same
place twice updates in place rather than stacking rivals, and formula_id is
the repair case, for when a variable already holds two formulas at one
address and you mean a specific one.

Batch mode defaults to 'partial': valid formulas write, invalid
ones skip, and a ledger is returned. Pass mode='atomic' for
dependency-chained writes; otherwise a ratio can persist while its base was
rejected, and the bare-variable default fills in zeros that look right.
Atomic cuts the other way too: one malformed formula rejects its valid
siblings, so a batch reject is not a verdict on each member. Isolate the
suspect and resubmit the rest before abandoning a form that parsed fine
on its own. Batch strategy and every write-response field:
[[build-model:references/saving-formulas.md]].
