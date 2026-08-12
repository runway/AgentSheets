# The layer model: how a variable's formulas fit together

references/02-formulas.md explains how the engine picks one formula per cell.
This file is about the other direction: which formulas a well-modeled
variable should carry, and in what order to author them. Dispatch is
physics; this is architecture. Read it before building anything a user will
drill into, and read it again when a drilled row shows zeros.

## 9.1 One choice carries this file

`$[…]` claims exactly its named grain. `[…]` claims every grain containing
the dimensions it names. `[]` matches every grain, and drilling in adds a
dimension to the current grain. Put those facts together and you get the law
this file is built on:

> An exact rule stops at a drill-in; a subset rule follows one that retains
> its named dimensions.

Watch it play out. Cash Balance carries a seed and a chain formula, both
written at the month grain:

```
Cash Balance$[Date.Month = "2026-01"] = 250000
Cash Balance$[Date.Month in any] =
  `Cash Balance`[-1] + `Net Cash Flow`
```

A table of Cash Balance by Month works: every cell is at grain
{Date.Month}, the condition's grain, so the formula computes every month.
Now drill in by Department. Each department row is at grain
{Department, Date.Month}. That is a different grain, the condition no
longer matches, and no other formula exists, so the rows fall to the
system floor and show zeros. The parent month row keeps its value, because
it is still computed at its own grain (the recompute law). Nothing
malfunctioned. The `$` made both rules deliberate grain-locks. If the model
instead means one independent chain inside every Department, write the chain
as `[Date.Month in any]` and seed each intended Department at its own starting
coordinate; that subset condition follows any drill-in that retains Date.

The competent move is to predict this before writing, not to diagnose it
after. The rest of this file is the discipline that makes the prediction
automatic.

## 9.2 The five layers

A variable's formulas form a stack. Dispatch reads it from the top (most
specific wins); you author it from the bottom.

| layer             | condition shape                | covers                                          | written with                         |
| ----------------- | ------------------------------ | ----------------------------------------------- | ------------------------------------ |
| 4 exact overrides | `$[…]`, items pinned           | named items of one grain                        | explicit condition or pasted address |
| 3 shape rules     | `[…]`                          | named dimensions at that and every richer grain | explicit condition                   |
| 2 time windows    | date term plus a formula range | a time regime, with reach chosen by the sigil   | `period` and bounds or condition     |
| 1 the default     | `[]`                           | every grain, present and future                 | no bounds                            |
| 0 the floor       | generated per grain            | every grain, always                             | the system                           |

**Layer 0, the floor.** For every grain a request touches, the engine
synthesizes what is missing: the bare default (source-column sum, or with
no source the aggregation function's identity — 0 for the usual sum), the
actuals and forecast fallbacks (source sum up to last close, 0 after it),
and the time rollup the aggregation function implies. This is why every
cell always computes and why the system's failure voice is a confident
zero for summed variables (a non-additive aggregation's identity can
render blank, references/06-validity.md). When you see zeros, some layer
you meant to write is missing and the floor is speaking.

**Layer 1, the default.** The one `[]` formula is the variable's
identity: what it means anywhere. A subset rule follows drill-ins (§9.1), but
only `[]` covers every grain — including one with none of the subset's named
dimensions, tomorrow's new axis, and any dimension item that arrives with
next month's sync. A variable with a good default survives arbitrary
reshaping; a variable without one can show floor values outside the shapes
its subset rules claim.

**Layer 2, time windows.** A default scoped to a regime: actuals compute one
way, forecast another. A window lives on a Date term in the condition. A
subset `[Date.Month in any]` window reaches every segmentation containing
Date.Month, while an exact `$[Date.Month in any]` window reaches only the
unsegmented month row. Use the subset spelling for a windowed default that
should survive dimension drill-ins; use the exact spelling when the regime
rule deliberately belongs only at that grain. Bounds always store exact:
`segments`, `grain`, and `period` compose a `$[…]` condition, every time,
so the `condition` field is the only way to author a subset rule. The
time rollup a coarse grain needs is also a stored rule: one
`change.set_time_rollup` item stores one subset rule that covers every
richer grain containing its Date grain — no block needed — and it warns
when a more-specific formula may still override the new method at its own
grain. The warning never names the rows; read the variable's formulas to
find and remove them.

**Layers 3 and 4, shape rules and exact overrides.**
`[Department in any, Date.Month in any]` is a rule about the Department ×
Month shape wherever those dimensions remain present, including richer
drill-ins. `$[Department = "Eng", Date.Month = "2026-03"]` pins one cell at
exactly that grain and stops when the grain changes. The constraints choose
items; the sigil chooses reach. Use the subset for a named-dimension business
rule and the exact form for a place that must remain pinned.

**The regime trap.** A rule whose condition carries a non-literal
system-Date term (`Date.Month in any`, a range — anything but one literal
date) and no `period` is silently scoped to Actuals
(references/10-deviations.md D2). Membership is exact, with two escapes as
a pair: a Date-less subset rule spans regimes untouched, and a literal
Date pin (`Date.Month = "2026-01"`) escapes the window; only the
non-literal Date term pulls it in. A layer-3 rule meant to shape the
forecast pairs its `segments` with `period`, which pins the base Date
grain and carries the window in one move (`grain` and `period` cannot be
combined; a period already pins the Date grain). Its reach across other
dimensions still comes from the stored condition's sigil.

## 9.3 Author at the broadest layer that carries the rule

Business statements name their own layer. Listen for it:

- "Revenue is price times quantity" is the variable's meaning: layer 1.
- "Forecast grows 5% a month" is a regime: layer 2.
- "Allocate rent to departments by headcount share" only means something
  at the department grain: layer 3.
- "West is different, use 8%" pins an item: layer 4.
- "SF's rent doubles in March 2027" is one cell: layer 4.

Write the rule at the broadest layer that can say it, and let narrower
layers carry only the exceptions. The healthy shape for a modeled
variable is one default, a window or two, and a handful of exceptions.
The unhealthy shape is dozens of cell overrides and no default: that is a
plan typed in by hand, cell by cell, and it breaks the first time anyone
drills or re-slices. When you inherit one of those, consolidating the pile
upward into one rule at the right layer is usually the most valuable edit
available.

## 9.4 The grain check, before any write

Before writing formulas for a table, list the grains the table will
actually evaluate. The grains are crossings: a cell's grain is one row
level joined with one column level, so enumerate the levels of both axes
and cross them:

1. every level of each row path: the leaf and each parent above it (each
   level drops the dimensions nested below it),
2. every date-carrying level of the column axis: a drilled column has a
   collapsed parent level too, and that collapsed state is the view a
   reader opens first. Sibling axes on one side tile the table next to
   each other and never cross, so two sibling column groups contribute
   separate levels, not a combined one,
3. the coarser time grains the granularity ladder will visit (a monthly
   table a user flips to quarterly evaluates quarter cells),
   - and when a view's only date axis is a foreign datetime dimension,
     that axis has a ladder of its own: an axis-less rule descends it to
     the finest grain and rolls back up (sole-foreign-axis descent, on by
     default), except that First/Last/Any are positional, decline on a
     foreign axis, and compute in place at the asked grain,
4. and each of those under both regimes when the variable splits actuals
   from forecast.

For each grain, say which layer answers. Any grain whose only answer is
the floor will show source sums or zeros; either that is what you intend,
or a layer is missing. This check is cheap, it needs no tool calls, and
it catches the drill-in zeros, the blank quarter, and the zero forecast
before they are built.

For the exact cash recipe above, the check reads: {Date.Month} has the
recurrence; {Department, Date.Month} intentionally has nothing and falls to the
floor. If the business meaning is one chain per Department, change the chain to
the subset `[Date.Month in any]` and seed each Department rather than copying
the same chain rule to every grain. {Date.Quarter} rolls up, so pick the verb a
balance needs (the closing month, not a sum) per references/04-time.md §4.4;
§9.2's layer-2 note carries the rollup write and its warning contract.

## 9.5 A parent cell has two jobs

Every parent cell answers two separate questions, and keeping them
separate is most of debugging:

1. **Who computes me?** Dispatch at my own grain. A dimension parent is
   pure recompute: the formula runs again with fewer dimensions in play.
2. **How do finer periods summarize into me?** Only the time axis has
   this second job: the rollup verb the aggregation function picks
   (references/04-time.md §4.4).

The symptoms sort themselves by which question failed. Parent right,
drilled children zero: question one has no answer at the children's grain,
so a layer is missing below. Children right, parent wrong on the time
axis: question two has the wrong verb, or the variable is a derived
quantity that must recompute instead of roll up (do-not-aggregate).
Ratio parent that is not the sum of its children: nothing failed, that is
recompute working as designed; say what the parent means instead of
"fixing" it.

## 9.6 Defaults must travel

Layer 1's power has a price: the default evaluates at every grain,
including grains with no Date and grains carrying dimensions you never
considered. Before promoting an expression to the default, check that it
travels:

- Date logic at a date-less grain does not error; it evaluates against an
  arbitrary period and returns something plausible and wrong. A
  recurrence, `[-1]`, or an `if` on Date.Month belongs in a time window
  or a grain rule, not in the default.
- A bracket lookup on a dimension the grain lacks returns null; make sure
  null is an acceptable answer there.
- Ratios and rates travel perfectly, because recompute re-derives them at
  every grain. Source sums travel. Plain arithmetic over other variables
  travels.

When the natural formula does not travel, split the variable's story
across layers: the traveling meaning in the default, the time-bound logic
in a window, the grain-bound logic in grain rules.

## 9.7 The address you write is table-bounded

You reason inside the table you are looking at. The engine has no such
context: a condition names dimensions absolutely, then its sigil decides
whether it lands only there or also at richer grains (§9.1). The `block`
parameter bridges the two, and only for `segments`: anchored to a block by
name, `segments` completes your address against that block's segmentation,
filling the dimensions you omitted as open slots.
`segments: {"Country": "USA"}` with `block` naming a Country-by-State table
persists at the grain the table actually computes, {Country, State}, as
`$[Country = "USA", State in any]`. The echo's `written` is the full stored
condition; `completed_from_block` names the terms the block injected. A
hand-written `condition` is never completed — you own every term, Date
included. Passing `block` also fills `uncovered_block_grains`, the levels
that table evaluates that no formula on the variable reaches: the coverage
read-back for any bounded write.

Know the boundary that comes with it: §9.1's law applies to the completed
condition. An exact anchored formula belongs only to that block's grain; a
subset anchored formula also reaches richer grains that contain every
completed dimension, but not a different shape missing one of them.
Anything that should hold everywhere is layer 1's job.

## 9.8 Reading a model by its layers

The layer model also reads in reverse. One `inspect_variables` `ask.saved_formulas`
read on a variable, then classify each formula by layer, and the
variable's story appears: a default plus two windows is a deliberately
modeled variable; fifty pinned cells and no default is a hand-typed plan;
a lone exact month-grain recurrence is the cash table of §9.1 and deliberately
stops at its first drill-in. Explain models to users in these terms (the default, the
forecast rule, the exceptions), propose consolidations when the shape is
bottom-heavy, and run the grain check of §9.4 before extending a shape
you did not build.
