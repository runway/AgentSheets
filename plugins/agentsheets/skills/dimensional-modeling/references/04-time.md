# Time

Time is a special dimension. It has a system-owned axis, a granularity ladder,
the engine's only rollup, and the actuals/forecast boundary. This file builds
on `SKILL.md` and references/02-formulas.md.

## 4.1 The system Date dimension

Every workspace has one system time dimension called Date. Call it “Date” or
“time period” to users. It combines source-table date columns, so one Date axis
can slice CRM revenue, HRIS payroll, and GL lines.

Consequences:

- **Always prefer the system Date for time axes.** Source-specific date
  fields (PAYMENT_DATE, CLOSE_DATE) slice only their own table and fragment
  the model. The listing flags which entry is the system date. When the
  question is explicitly about the other column, a source-specific DATETIME
  axis is the correct shape; the `edit_model_views` `dry_run` warns on a
  non-system-date time axis but does not reject it. On such a view that
  sole date-typed dimension carries the date-axis descent (SKILL.md axiom
  13); rules omitting it descend it like the timeline
  (references/02-formulas.md).
- Date items are **generated, not enumerated**: months exist across the
  table's date range whether or not data exists there. Never try to list the
  Date dimension's items; there is no source column to enumerate, and future
  periods are items by construction.

**Which column dates a variable's rows.** A source-backed variable's cells
are built by bucketing its table's rows onto the Date axis. When the table
carries several date columns (invoices with invoice_date and paid_date),
exactly one column does the dating, and that choice is a per-variable
setting: `preferred_time_property` (`edit_variables` update, by name or id;
read it back as `preferredTimeProperty` on the variable's
`inspect_variables` entry). Unset, rows follow whichever of the table's
date columns joined the system Date first — import order, not business
meaning — and the dictionary does not say which one that is, so probe
ephemerally which column the values follow, then set the preference to pin
it. When the model's meaning depends on the column (billings by invoice
date vs cash by paid date), set it explicitly rather than trusting the
default. The setting exists only for source-backed variables and only
accepts a datetime entry of the variable's own source table. A formula
variable has no rows to date: its cells live on the grid directly and its
time axis is the system Date itself, so the setting does not apply to it —
how its coarser grains summarize its base grain is the rollup
(`edit_variables` `change.set_time_rollup`, §4.4), a different concern.

**What the Date entry carries.** The workspace's time settings hang off the
system Date dimension, not off any block: a default granularity, an explicit
default range (start and end), and Last close (§4.5). `inspect_dimensions`
returns them on the Date entry as `timeSettings` — `defaultTimeGranularity`,
`defaultExplicitRangeStart`, `defaultExplicitRangeEnd`, `lastClose`, and
`lastCloseId`. Only the dictionary read carries them. `edit_dimensions` moves
them: `time_defaults` for the three defaults, `last_close` for the frontier.
The settings are model entities like any other, so scenario resolution
(references/05-scenarios-and-comparisons.md §5.1) applies: reads and writes
bind to the call's scenario, and a scenario that never wrote its own tracks
Main's live.

**Defaults are starting values, not live fallbacks.** A new object copies the
workspace defaults once. Later default changes do not update it. An empty block starts at the workspace
granularity and range; a block created from an authored config takes exactly
what the config says, and the defaults are never consulted (an authored config
that omits granularity lands Raw, §4.2). The builds inherit by the same rule:
an import build takes the workspace grain and range for whichever the call
omits, and the headcount build always takes the grain, collapsing DAY to
month because periodic payroll has no daily semantics. Set the defaults before
you build; to fix a table already on the page, change that block's own
settings (§4.2, §4.3). The one place the stored default keeps mattering after
creation is formula authoring, which takes the base grain for `Date.<Grain>`
spellings from it.

NO_GRANULARITY is a stored value, not an absence, and the two kinds of reader
treat it differently: whoever needs a period unit coerces it to month (formula
authoring, the headcount build), while whoever merely copies it keeps it, so a
seeded block or import build lands on a Raw axis (§4.2).

## 4.2 Granularity

Granularity buckets a time dimension: day, week, month, quarter, half, year.
It applies only to DATETIME entries used as dimensions and does nothing on a
variable.

**Identity order.** Granularity is part of dimension identity
(references/01-the-dimensional-universe.md §1.1): Month-of-Date and
Day-of-Date are different dimensions, in a fixed identity order (day < week
< month < quarter < half < year).

**The cell-rollup chain.** A separate, shorter fact: a
Year cell may compose from Half, Quarter, or Month cells, a Half from
Quarter or Month, a Quarter from Month — and the chain stops at Month.
Nothing derives Day or Week cells by rollup; those grains read source rows
or their own formulas (§4.4).

A date axis gets its granularity in this order:

1. a granularity pinned in the entry reference itself (formulas do this
   with suffixes like `Date.Month`),
2. an explicit per-axis override,
3. the block's default granularity setting,
4. otherwise none (the UI calls this "Raw"), which is valid for display.
   Features that need a period unit then fall back to defaults (time
   comparison assumes monthly) rather than failing. Raw axes render the raw
   datetime items present in the data, unbucketed.

The workspace default is not checked while rendering. It may only have set
item 3 when the empty block was created (§4.1).

Changing the block-level granularity also snaps the date range to bucket
boundaries and rewrites any explicit per-axis overrides to match. The
block-level control therefore wins over stale per-axis values.

**Source bucketing.** One asymmetry (a separate mechanism from the
cell-rollup chain above): weekly buckets are built from day-level source
rows, but nothing coarser is built from weeks — months are built from days
directly, bypassing weeks (weeks do not nest in months). Weekly display is fine;
weekly values feed nothing coarser. And weekly buckets are keyed to
Monday starts while the Actuals frontier anchors week boundaries to
Saturdays, which is why the week straddling Last close behaves oddly
(references/08-recipes.md R11).

**The override's granularity argument.** `change.set_time_rollup`'s
`granularity` is a different thing from a date-axis bucket: it names the
coarser grain a rollup targets (§4.4).

## 4.3 The date range

The visible time window is block-level config (an absolute start and end).
Within that window the date axis generates its items, which is why forecast
months render before any data exists on them. The generated range widens
backward to cover the earliest formula-range boundary, so the actuals window
is never clipped away. Always set an explicit range on configs you author:
a grammar restate that omits it is refused, and `ask.try_formulas` requires
`from`/`to` because a date dimension generates no members without a range.

A block created empty starts at the workspace default range (§4.1); one you
author carries the range you wrote. Either way, editing the range afterwards
is a block-level change and leaves the workspace default where it is.

## 4.4 Time rollup: the one recompute exception

Normally, a coarser cell reruns its formula instead of summing child cells
(references/02-formulas.md §2.4). Time rollup is the one exception. For a
variable with any aggregation function except do-not-aggregate, the engine
synthesizes a normal formula bound to the coarser granularity. That formula
computes each coarser time bucket from the next finer one, for example
"Month = sum of my Days, per whatever other dimensions are in play".

The rollup method says how a coarser cell combines finer periods. It has one
default plus per-grain overrides.
The default is the variable's `aggregation_function` (`edit_variables`
operations), which picks the verb at every coarse grain:

| aggregation function | time rollup                                                    |
| -------------------- | -------------------------------------------------------------- |
| sum                  | sum of finer periods                                           |
| min / max            | min / max of finer periods                                     |
| first / last         | first / last finer period (balances: use last)                 |
| any                  | first finer period                                             |
| count                | count of days, then sum of counts above day                    |
| do-not-aggregate     | none: coarser cells come only from the variable's own formulas |

The exception is `edit_variables` `change.set_time_rollup`, which overrides the
default at one grain: {variable, granularity quarter / half-year / year,
method SUM / LAST / AVERAGE}. One call writes one subset rule at
`[Date.<grain> in any]`, covering that grain at current and future drilled
grains alike; a `block` argument is accepted and ignored. AVERAGE exists
only as this override — there is no average default — and the no-rollup
choice is the do-not-aggregate default.

The rollup is dispatched like any formula, so an authored formula for the
coarser granularity always beats it. The write warns when a more-specific
formula already governs that Date grain somewhere — it says one exists,
never which — and the new method does not reach that grain until the
formula is removed. Rollup never applies to ratios: their
aggregation should be do-not-aggregate, or their formula simply recomputes.
Rollup follows the view's single date-axis descent — which axis, what a Raw
or several-foreign-axes view descends, and what the positional verbs
decline is SKILL.md axiom 13. One trap:
a do-not-aggregate variable whose only formula
is day-scoped does not show empty months. The synthesized defaults fill the
coarser cells with source sums or zeros that ignore the day-level formula,
silently (references/06-validity.md).

## 4.5 Last close: the actuals/forecast frontier

**Last close** is a workspace date reference, defined by a formula, that marks
where history ends. System formula ranges split every
entry's formulas around it: **Actuals** ends at last close, **Forecast**
starts after it. A range-scoped formula's window is inlined into its
condition, so regime selection is ordinary dispatch. The condition must
therefore carry a Date term for the window to inline against.
A `period` bound injects that term itself, so a windowed default never hits
the date-less rejection (references/02-formulas.md §2.7).

"Ends at last close" is exact only at day granularity: the window truncates
to the last _closed_ period of the evaluating granularity, and the snap
direction depends on that granularity. At WEEK the Actuals end snaps back
to the last Saturday on or before last close, so the straddle week (the
week containing last close) evaluates entirely as Forecast; even its closed
days get re-projected. QUARTER, HALF, and YEAR snap back the same way.
MONTH snaps _forward_ to the containing month's end: a mid-month close
makes the whole month Actuals. So weekly and monthly tables disagree across
the frontier unless last close sits on a completed-week boundary, and "why
is this week's actual wrong" is usually the straddle week.

Actuals sync continuously, so this boundary does not mean “where data exists.”
It means “where the books are trusted.” It
moves manually: forward month by month as books close, or into the current
month when someone wants intra-month numbers treated as actuals. Each
scenario has its own last close.

Read the current value, with its `lastCloseId`, on `inspect_dimensions` —
the Date entry's time settings (§4.1). Move it with `edit_dimensions`
`last_close`, which takes an ISO date or an ISO month.

The system ranges belong to series whose data arrives through the
accounting close (the GL and its derivatives): for those, the close is
where trustworthy data ends. A series that keeps updating regardless of
when books close (an uploaded price history, usage logs) should not be
split on Last close: if its data runs past the close, the Forecast range
overrides real history with projections, silently. For those series, use
plain date-conditioned formulas instead: the source-reading formula
conditioned through the series' own last data point, projections after it.
Everything is a formula; the ranges are a convenience for data that follows
the book close, not a requirement.

"Why is next month zero" is almost always "no forecast formula yet": with no
authored forecast formula the forecast defaults to 0 (references/02-formulas.md §2.7).

## 4.6 Time comparison

Time comparison puts each visible period next to an earlier period, such as
last month or the same month last year.

The rules:

- The table must contain **exactly one date axis**, on one side only. The
  comparison axis is inferred from that; it is never configured. Zero or two
  date axes make "previous period" undefined and the request is rejected.
  Granularity resolves from the axis, then the block default, and falls
  back to month when neither is set.
- The **offset counts in units of the table's granularity**, not calendar
  units: year over year is 12 on a monthly table, 4 on a quarterly table, 1
  on a yearly table.
- Four measures can be shown per period: the value, the comparison value,
  the variance (value minus comparison), and variance %. They render as
  extra columns per period or as row splits; that layout is block
  presentation (references/12-editing-blocks.md).
- Setting one comparison kind clears the other, so switching is a single
  edit (references/05-scenarios-and-comparisons.md §5.4 covers the scenario side).

Time comparison answers "how does each period compare to its shifted
counterpart" with zero formula work. When you instead need the delta as a
_number other formulas can use_, write it as a formula
(`Revenue - Revenue[-1]`), which composes; comparison measures do not.
references/02-formulas.md §2.9 owns the relative-vs-pinned choice for that
reach-back. On an imported variable, both paths read through its
preferred-time binding, so set the binding first ([[build-model:references/formula-grammar.md#Bind imported variables to time]]).
