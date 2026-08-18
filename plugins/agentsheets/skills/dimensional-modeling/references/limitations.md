# Known limitations

Each entry gives a current limitation, its symptom, and the safe response.
Remove **temporary** items after the platform fix ships. **Permanent** items
describe intended behavior.

**Running balances.** Several limits affect values that use their own prior
period, such as cash or customer balances. Use three parts:

1. **Seed** — one starting cell on the **first month** of the table's
   date window: `` `X`$[Date.Month = "2026-01"] = 100000 ``. (`set_values` with a
   `condition` writes this form. If you write the condition by hand, use the
   string form `"2026-01"` — a `date(2026, 1, 1)` condition never matches.)
   The `$` makes the seed one exact place (entry 7's law), so write it at
   the chain's own grain. A seed at a finer or coarser grain than the chain
   silently never fires. Never seed a coarser grain to fix its view; that
   is the anchor's job.
2. **Chain** — a plain prior-period read with no guard. For one
   chain on the unsegmented time row, write
   `` `X`$[Date.Month in any] = X[-1] + `Change` ``, one item per regime the
   window spans: a non-literal Date term (`in any`) with no `period` governs
   actuals only. A literal Date pin escapes that implicit window — which is
   why step 1's seed needs no `period` wherever its month falls.
   The `$` is intentional:
   the chain stops if the view adds a dimension. Use the subset
   `` `X`[Date.Month in any] = X[-1] + `Change` `` only when the business means
   an independent chain in every segmentation containing Date, and seed each
   intended segment or deliberately let its missing prior start at zero.
3. **Anchor** — for each coarser grain a view will show (quarter, year),
   one `edit_variables` `change.set_time_rollup` {variable, granularity,
   method: "LAST"} write. Without it, coarse views are quietly wrong
   (entries 2 and 5); with it, a quarter shows its closing month. The
   anchor has a regime edge: its stored condition names Date
   non-literally, so with no `period` it governs actuals only — read back
   a forecast-window coarse cell before trusting it
   (references/10-deviations.md D2).

Entries 2, 3, 5, and 10 are what you see when a piece of this recipe is
missing or replaced.

## 1. A sync can revert an explicit aggregation function — temporary

A stored LAST-family `aggregation_function` computes correctly at every grain — a LAST
recurrence's quarter is its closing month with no anchor formula. The hazard is on the write
path: a source sync can clobber an explicit LAST back to DO_NOT_AGGREGATE (AI-701), silently
reverting the setting, and `edit_variables` warns when one of these functions is set.

- What you'll see: the setting reads back different from what was written, after a sync.
- Do: for formula-driven point-in-time metrics prefer DO_NOT_AGGREGATE plus a
  `set_time_rollup` {method: "LAST"} anchor (entry 2) — a stored formula rule is immune to the
  sync clobber. If a variable misbehaves after an aggregation change, read the setting back: the
  engine computes whatever is actually stored. The anchor write's warning contract is
  references/09-the-layer-model.md §9.2.

## 2. Balances at quarter or year: use the three-piece recipe — temporary

Cash balance, customer count, anything computed as "last period plus this
period's change". The LAST aggregation setting computes these, but a sync
can silently clobber it (entry 1), so the robust shape is the
running-balance recipe at the top of this file. The pieces that still
matter:

- No seed inside the window → every cell is silently zero (entry 10).
- No anchor → monthly is right but a quarterly view re-runs the chain at
  quarter pace and shows wrong numbers with no error (entry 5).

## 3. A running total built with WHERE only works at one time scale — permanent

`sum(X[Date.Month where Date.Month <= this.Date.Month])` gives a
correct running total in a monthly table. But in a quarterly or yearly
view, each cell only adds up through the FIRST month of its period: a Q3
cell shows the total through July, not through September. That is how
`this.` resolves at a coarse grain — always the period's first month —
and it will not change.

- Do: keep the WHERE formula for the monthly view, and anchor each coarser
  grain with `set_time_rollup` {method: "LAST"}. The anchor makes the
  quarter show its closing month's total, which is the correct quarterly
  running total. (The recipe's chain form works too.)

## 5. A date-pinned seed alone leaves quarterly and yearly views wrong — temporary

A variable seeded with a pinned month — the recipe's own seed piece, or
`if(Date.Month = "2026-01", <first value>, X[-1] + …)` — computes correctly
in a monthly table. But a quarterly or yearly view without an anchor is
quietly wrong: the coarse cell is built from its period's FIRST month, not
its closing month, so a monthly 100/110/120 quarter reads as 100 or 110 —
never the correct 120. There is no error anywhere, which makes it easy to
mistake for a data problem.

- Don't: swap the seed for a `count(X[-1]) = 0` guard expecting different
  quarters; this entry's law applies to it identically — the anchor is
  still required.
- Do: keep the seed and add the missing piece — a `set_time_rollup`
  {method: "LAST"} anchor for each coarse grain a view will show. With the
  anchor, every actuals view is correct; the forecast side carries the
  anchor's regime edge (the recipe's step 3).

## 6. Breaking a variable down by a date-typed grouping dimension shows nothing — temporary

A dimension whose items are dates but whose job is to group rows into
categories — a signup cohort spelled "2025-01", a contract-start month, a
vintage — is stored as DATETIME by ingestion. Break a variable down by it
(`mrr > cohort`) and every item that falls outside the TABLE's date window
is hidden from the result: for 2025 cohorts viewed in a 2026 window, the
breakdown carries no 2025 cohort rows — only the window's own months as
empty rows. The parent variable total
is correct; only the breakdown is missing. The engine finds the real items,
then clips every date-typed axis to the table's window before rendering.

- What you'll see: no per-item rows (or only the few items inside the
  window), under a correct parent total.
- Don't: add `.Month` to the dimension, switch layouts, put the dimension
  on columns, or hand-build a formula per item. None of it brings the rows
  back, and chasing it burns the whole turn.
- Do: widen the block's date window so it covers the dimension's own
  dates — `edit_table_blocks` with `change.window` {start, end}. For
  signups in early 2025 viewed in 2026, set the window to 2025-01..2026-12
  and the 2025 cohort rows appear with their values. Tell the user the two
  costs that come with it: the columns span the wider window too, and
  every window month with no dimension item appears as an empty row.

## 7. A bare period-only write reaches only the plain date grain — permanent

A `change.set_values` item whose only bound is `period` (actuals, forecast,
or a custom window) stores `$[Date.<base-grain> in any]`: an exact condition
with one lone Date term, so the windowed formula applies only at the {Date}
grain — the variable's own row over time. Any grain that adds a dimension
(the same variable drilled by Department, Product, anything) is not covered
by it, and those cells fall to the generated regime fallbacks: raw source
sums in actuals, 0 in forecast. The `$` prefix makes a condition exact;
without `$`, a `[…]` condition matches any grain that contains its terms. Every
bounds-composed write stores `$`.

- What you'll see: the variable's top row correct across time, while every
  drilled or segmented row shows 0 in forecast months (and unwindowed
  source sums in actuals) with no error anywhere.
- Don't: rewrite the windowed formula or re-run the same `period` write
  hoping the drills pick it up; they cannot. And don't write one rule per
  drilled grain; one subset rule covers the pile.
- Do: keep the `period` write for the variable's own time row, and add ONE
  item with `condition = "[Department in any, Date.<base-grain> in any]"`
  (each drilled dimension as an `in any` term, Date at the model's base
  grain) plus the same `period` — the subset rule covers every richer grain
  that retains those dimensions, however deep the drilling goes.
  A condition-plus-period write requires exactly that base-grain Date term; an
  off-base grain is rejected rather than stored dead. `segments` naming each
  drilled dimension as an open slot ({"Department": "\*"}) plus `period`
  remains as coordinate sugar — stored exact, one grain. See
  references/04-time.md §4.5.

## 8. A vanished source column fails whole calculations, not just its own cells — temporary

An imported file can be re-imported with fewer columns, or a rename can sever a
column's mapping. Either way the dimension or variable created from that column
survives, still pointing at something that no longer exists. Nothing cleans it
up.

From then on, any calculation that has to bind that entry to its source fails
as a whole. Not just its own cells — the whole request, including formulas that
never mention the entry, its table, or its source.

Nearly every source feeds the Date axis, so the usual shape is this: every read
with a date breakdown fails, the same read without one works, and every formula
in the workspace is correct.

- What you'll see: the read is refused rather than failing, carrying
  `error_kind: "blocked"`. It says it binds a source column an imported table
  no longer provides, and quotes the calculation service:
  `external column <name> is not mapped in table <id>`. Every read carrying the
  affected breakdown gets it, even a brand-new block built from variables you
  just created. The `<id>` is an internal source-table id that no tool
  resolves, and the refusal's own hint names the no-breakdown check below.
- Don't: rewrite your formulas, set `preferred_time_property`, or widen the
  workspace time defaults. The broken entry belongs to an imported source that
  no tool here can repair, so none of those can help.
- Don't trust the column name either. The message can say `Date` while the
  culprit is an ordinary imported column that happens to share that display
  name. It does not mean the system Date axis is broken.
- Do: run the same read once with no breakdown (axiom 20c). If values come
  back, your formulas are sound and a broken source binding is confirmed. Then
  either deliver on an axis the question can honestly use (axiom 12b), or tell
  the user which import needs repair. Say plainly what failed and what you
  changed. There is no workaround inside the model.

## 10. A seed outside the table's window makes the whole chain zero — temporary

A recurrence reads only cells inside the window being calculated. If the
seed sits on a month before the table's start, the chain's first in-window
cell reads an empty prior period, empty counts as zero, and every cell
after it is zero too. There is no error anywhere, and the same chain looks
correct in an evaluate whose window happens to include the seed month —
which makes this easy to misread as a data or caching problem.

- What you'll see: the whole chain 0 in the table, while a spot-check
  evaluate over a wider window shows correct values.
- Don't: rewrite the chain formula; it is fine.
- Do: seed the first month **inside** the table's window (forecast chains:
  seed the first forecast month). Or widen the block's window to include
  the seed month. A closed-form curve avoids seeds entirely.

## 11. Per-segment values written without a block or a date term reach no dated row — temporary

The engine matches a stored `$` condition to a cell by exact dimension set.
`segments` alone — no `block`, `grain`, or `period` — stores exactly that:
`$[Department = "Engineering"] = 30000` at the dateless {Department} grain. A dated
table's department rows live at {Date.Month, Department}, a different set, so the
rule reaches none of them, and those cells fall to the generated regime fallbacks:
raw source sums in actuals, 0 in forecast. A scaffold with no source data is
therefore zero everywhere, parent included, with no error anywhere.

- What you'll see: forecast cells 0 at every level, top row included — not §7's
  shape (top row right, drills 0). Actuals cells show raw source sums, so a
  no-source scaffold is 0 everywhere while a source-backed variable shows
  plausible actuals over a zero forecast. And unlike a genuinely empty variable,
  its saved formulas exist and look right, their conditions naming dimensions
  with no Date term.
- Don't: resave the same segments items hoping they reach the rows; they cannot.
  And don't fold the values into one global `if(Department = …)` expression — that
  reaches every level including the collapsed parent, which binds no item, so the
  parent shows the leftover branch instead of its rollup
  (references/10-deviations.md D7).
- Do: rewrite the same items with the table passed as `block` and `period` naming
  the regime the values belong to — anchoring completes the dimensions you did not
  name, and by diagnosis time the table exists. `grain` in place of `period`
  reaches actuals only and leaves forecast cells at 0. In a from-scratch build,
  create the table before its scoped values.
