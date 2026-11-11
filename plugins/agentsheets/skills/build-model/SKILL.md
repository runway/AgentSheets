---
name: build-model
description: Build, change, or diagnose a model and model the business in it. Use for variables, formulas, dimensions, mappings, time, rollups, table layouts, comparisons, formatting, and unexpected results; for forecasts, budgets, plans, and scenarios; for headcount, payroll, and rosters; for bookings, pipeline, and deal registers; and for statements, receivables and collections, cash and runway, budget versus actuals, and unusual results.
---

# Build the model

A workspace has one shared model, and tables are views over it. This skill explains calculation with a running example, then covers imported records, mappings, rules, and modeling the business (revenue, costs, staffing, cash, and statements).

## The model and its views

- **Shared variables vs. table views**: Variable names, formulas, and values belong to the shared model; row order, formatting, and breakdowns belong to each table block.
- **Reusing vs. creating variables**: Plain names on `create` reuse an existing variable and write only where empty; `#hex` tokens overwrite shared rules, and `new: true` creates a distinct variable when a requested report reuses an existing name for a different concept.
- **Object deletion options**: Before deleting a shared variable or dimension, inspect dependency impact and offer **Cancel** and **Delete completely** (plus **Keep with Ari** when Ari can start a new chat, which inspects the scenario, asks for a destination, and places the preserved object before removal). System `Date` cannot be deleted.
- **Parallel tool execution**: Tool calls in one turn run concurrently. Sequence dependent calls across steps or bundle the table and its rules into one `edit_table_blocks` `create` call.

When the person chooses a rule over an imported one, the import's summary page records the affected table, the chosen rule, and why it differs from the file. Its saved text names the source file's `runway-file-…` ID. Later file updates read that page to preserve the decision.

## The picture

Consider a monthly model for a listening service selling two tiers (`Lark`, `Chorus`) across two territories (`Southgate`, `Eastmere`). Five concepts govern how the engine calculates it.

### A cell

A cell is one variable at one member of each of its dimensions (plus one period when dated). `Paying listeners` for `Chorus` in `Eastmere` in `March 2027` is one dated cell; `Tier fee` for `Chorus` is a timeless cell. Tables display selected cells, while rules and inputs define cells wherever they reach.

### A rule and an input

- **Inputs vs. formulas**: A cell holds either a literal input (an opening count, fee, or growth rate that the operator controls) or a formula calculated at each cell's coordinates. Typed grid values save one rule per cell, so use formulas for calculated series rather than grids of repeated literals.
- **Date grain & range (`Actuals` vs. `Forecast`)**: A formula's condition sets its `Date` grain (a `Month` rule reaches dimensions at `Month`; a daily forecast requires `Day` coverage). `Actuals` and `Forecast` have separate rule slots; write both when a formula applies across the whole horizon.
- **Opening seeds & recurrences (`[-1]`)**: Seed dated assumptions in the last `Actuals` month and carry them forward with a `Forecast` formula so every future month responds to the assumption:

```formula
`Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory])
```

- **How `[-1]` resolves at the boundary**: In the first `Forecast` month, `Weekly listeners[-1]` reads the last `Actuals` month's input, imported value, or `Actuals` rule (or `0` if none exists). For cash, seed opening cash in the last `Actuals` month and add `Net Cash Flow` to the prior balance in `Forecast`.
- **Single-cell overrides**: A one-cell override pins only that coordinate; removing it restores the default formula and updates parent and period rollups.
- **Empty & null semantics**:
  - Unset variable cells read `null` for date/text types (`End Date = null` is true) and `0` for numeric types.
  - Blank imported numbers are `null` in the raw column (`last(<column>)` is `null`; `count(<column>)` skips blanks), whereas a `SUM` variable over all-blank rows evaluates to `0` (`count` on the variable counts the `0`).
  - `a = null` tests for missing values. Because `a = b` evaluates to `null` (falsy in `if`) when either side is `null`, match two nullable attributes with `(a = null and b = null) or a = b`.
  - Wrap ratios in `iferror` (e.g., ``iferror(`Listening revenue` / `Paying listeners`, NULL)``); see `## Formula syntax reference` (`### Functions`) for `sum`/`product` (`0`) vs. `min`/`max`/`average` (`null`).

### A read

- **Relative reads (same coordinates)**: A plain variable name reads that variable at the current cell's exact coordinates (same tier, territory, and month).
- **Absolute reads (`$`) for coarser inputs**: If an input has fewer dimensions than the current cell (for example, `Weekly listeners` by `Territory` and `Month`, with no `Listening tier`), a relative read at `Chorus × Eastmere × March` finds no `Chorus` value and returns `0`. Pin only the coordinates the input carries with `$`:
  - Dated 1D input: `` `Weekly listeners`$[Territory = this.Territory, Date.Month = this.Date.Month] ``
  - Timeless 1D input: `` `Tier fee`$[`Listening tier` = this.`Listening tier`] ``
  - A row's formula also applies to breakdown lines without their own formula; when the formula does not vary by the breakdown dimension, a coarser absolute read repeats its amount on each line, so N such lines sum to N times that amount for a SUM variable.
- **Coordinate overrides (`[...]`)**: Brackets replace specified coordinates and keep the rest: `` `Listening revenue`[Territory = "Eastmere"] `` reads Eastmere at this cell's tier and month, and ``sum(`Listening revenue`[Territory in any])`` sums all territories at this cell's tier and month. Outside `where`, `Dimension` and `this.Dimension` are equivalent.
- **Inside `where` clauses**: Inside `where`, a plain name refers to the candidate record's key, while `this.<Dimension>` refers to the current cell's member. Predicate values accept expressions such as `Date.Month = startofmonth(`Last Snapshot Month`$)`.
- **Keeping vs. replacing drilled coordinates**: Filtering a second dimension keeps the row's drilled coordinate (``sum(Amount[`Account Type` = "Income"])`` broken down by `Account` shows each account's amount), whereas filtering `Account` itself (`sum(Amount[Account in {"A", "B"}])`) replaces the row's account and repeats the combined sum on every row.

### A breakdown

A request phrased as "if X, then use Y" defines a standing rule. A default formula at the relevant dimension level (`child_defaults`) keeps that rule true for current and future rows, including accounts or vendors added later.

- **Leaf rules & automatic parent sums**: Nesting dimensions under a variable creates a breakdown (`Paying listeners` by `Listening tier`, then `Territory`). Writing `child_defaults` on the deepest dimension defines the leaf formula; saving the table automatically generates `level_default_sync` rules that sum each level into its parent (`Territory` into `Listening tier`, and tiers into the variable total).
- **Percentage breakdowns (`PERCENTAGE`)**: Variables with `format: "PERCENTAGE"` are never summed across children and receive no `0` leaf fallback:
  - When a leaf formula's inputs all exist on the parent row (such as `Profit / Revenue`), the parent row automatically evaluates that formula on the parent's totals.
  - When leaves hold typed rates, multiple formulas, or dimension-specific terms (`this.Segment`) that have no parent value, the parent row stays blank unless you write an explicit weighted formula or a single row-level `formula` (such as ``iferror(`Listening revenue` / `Paying listeners`, NULL)``).
- **Grouped column headers (`summary`)**: When columns group months under a dimension header, the child column's `summary` setting controls the header (`SUM` adds the months; `NONE` evaluates the row's own formula at the header).

### Time

- **System `Date` vs. source dates**: System `Date` generates the table's reporting calendar across the window so forecast periods exist before data lands; a source date column shown as a field does not drive the table calendar.
- **Fiscal-year conventions**: Align fiscal-year labels (`FY26`) with the customer's existing month labels and dated figures so restating those figures under a fiscal-year label targets the same calendar months.
- **Base grain & time rollups (`rollup`)**: Dated rules evaluate at the model's base grain (usually `Month`). Coarser columns (`Quarter`, `Half-year`, `Year`) are filled by the variable's time rollup:
  - `SUM` (default for monthly numbers): Flows such as revenue or expenses.
  - `LAST`: Balances and counts such as cash, weekly listeners, or headcount (update via `set_time_rollup` on existing variables).
    An annual amount, price, rate or share held constant each month rolls up with `LAST`; recurring monthly revenue and expenses use `SUM`.
  - `AVERAGE`: Period means.
  - **Dateless ratio formulas**: A row `formula` written with no `Date` condition evaluates directly at every time grain without a rollup, keeping ratios accurate in quarters and years.
- **`Last close` boundary**: `Last close` separates closed `Actuals` periods from `Forecast` periods across the scenario. Write rules in `Forecast` for known events after close, and write disclosed estimates in `Actuals` when estimating closed periods without shifting `Last close` for other reports.
- **Time offsets & proration**: Use explicit calendar offsets (`dateadd`) or relative windows (`[-3]:[-1]`) for period comparisons, and prorate records across `[Subscription in any, Date.Month in any]` with `datedif(start, end, "D") + 1` and `daysin(Date, "month")`.

## One worked build

The running example as four `edit_table_blocks` creates. Two timeless tables hold the inputs:

```
{"action": "create", "name": "Tier assumptions", "rows": [{"dimension": "Listening tier", "items": ["Lark", "Chorus"]}], "columns": [{"variable": "Tier fee"}, {"variable": "Uptake rate"}], "values": {"columns": ["Tier fee", "Uptake rate"], "rows": [["Lark", 20, 0.04], ["Chorus", 50, 0.01]]}}
{"action": "create", "name": "Territory assumptions", "rows": [{"dimension": "Territory", "items": ["Southgate", "Eastmere"]}], "columns": [{"variable": "Listener growth"}], "values": {"columns": ["Listener growth"], "rows": [["Southgate", 0.02], ["Eastmere", 0.03]]}}
```

Listening is dated: each territory's opening count is an input in the last Actuals month, 2026-08 here, and the Forecast rule grows it; as a count, its quarters, half-years, and years read the last month:

```
{"action": "create", "name": "Listening", "columns": [{"time": "MONTH"}],
 "rows": [{"variable": "Weekly listeners", "breakdown": ["Territory"], "rollup": "LAST",
   "child_defaults": [{"dimension": "Territory", "forecast": "`Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory])"}]}],
 "values": {"columns": ["2026-08"], "rows": [["Weekly listeners[Southgate]", 100000], ["Weekly listeners[Eastmere]", 60000]]}}
```

Paid listening breaks down by tier then territory. The leaf rules pin the inputs that carry fewer coordinates, the level defaults add the leaves up to each tier and the total, revenue's quarters and years add their months, and the ratio, written with no Date, is right at every level and in every period:

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

Checks: the first Forecast month grows from the opening input, not from 0; Chorus × Eastmere in September is 61,800 × 0.01 × 50 = 30,900 of revenue; Chorus's row equals its two territories.

## How a rule reaches a cell

- **Relative (`[...]`) vs. exact (`$[...]`) conditions**: A relative condition (`[Stage in any]`) claims every cell whose coordinates include those dimensions at that depth and below (stage, owner, and deal). An exact condition (`$[...]`) claims only the exact depth named. If a parent sum fans out into child cells where no finer rule exists, it reads itself (`#CIRC!`).
- **Specificity precedence order**: When multiple rules match a cell, the engine selects one in strict order:
  1. Exact condition (`$[...]`) beats relative condition (`[...]`).
  2. More dimensions named beats fewer (`[Stage in any, Deal in any]` beats `[Stage in any]`, which beats `[]`).
  3. More pinned members beats open wildcards (`[Stage = "Won"]` beats a set, which beats `in any` or an expression).
  4. Most recently saved rule breaks remaining ties (all saved rules beat generated source/zero fallbacks).
- **How table rows map to rule conditions**: Writing `child_defaults` on a pinned member row (`Scoping`) saves `[Stage = "Scoping", Owner in any]`. Writing `child_defaults` on an open breakdown axis row (`Total[Stage][Owner]`) saves on the full path `[Stage in any, Owner in any, Deal in any, Date.Month in any]`.
- **Diagnosing unexpected values**: Inspect `formulas`, `inherited_from`, `formula_origins`, and `include_inputs` on `inspect_table_blocks` readback (with `recalculate` when cached values are disputed). Unparseable expressions show `#ERROR!` in their claimed cells, while unparseable conditions match no cells.

## Imported records and sources

- **Variable aggregations & external columns**: Each imported variable aggregates source rows using its configured function (`sum(<runway:exttables/550e8400-e29b-41d4-a716-446655440000/columns/Amount>)` for additive amounts, or `last(<column>)` for latest-observation attributes such as roster department). Relative reads let the variable's aggregation pick one observation per period; an absolute read (`$`) over a restated source gathers all snapshots unless filtered.
- **Source reads after `Last close`**: A variable's source read covers months through `Last close`; later months read `0` (null for dates and text). It is an imported variable's generated fallback, or a variable's only `[]` rule when that rule is un-ranged and exactly one aggregate over one source column (`sum(<column>)`). Inspection marks that rule `after_last_close`.
- **Source rows in `Forecast` months**: A `Forecast` rule with the same read (`default_formulas.forecast: "sum(<column>)"`) reads the source rows dated after `Last close`. A `[]` rule with any other expression computes every period.
- **Preferred time column (`preferred_time_property`)**: Places imported records on the reporting calendar by the business date that governs them (such as expected close date for open pipeline or observation month for restated rosters) and orders `first`/`last` aggregations. Set it across a source via `edit_table_blocks` `sources`.
- **Restated snapshots vs. transaction ledgers**: A ledger has one row per transaction and is summed; a restated export or sync stamps one observation per entity per run. Inspect the preview first (identical rows across stamps repeat current state, not history; differing rows preserve per-stamp state), and filter by snapshot month (`ObservedAt.Month = this.Date.Month` or `ObservedAt.Month = date(2025, 11, 1)`).
- **Rosters vs. monthly staffing tables**:
  - **Timeless roster (`Employee ID` on rows, no `Date`)**: Shows each employee's latest current-state attributes (`last` over observations).
  - **Monthly staffing table (`Date` on columns)**: Binds each period to that month's observation and carries the latest observation forward into future months (`[`Employee ID` in any]` carries forward; `[]` sums across employees).
- **Time-varying attributes (`Department in Force`)**: When an employee's department changes over time, look it up without `$` in Actuals or carry it in a dated text variable (`Department in Force`) so transfers move headcount in the month they occur:

```formula
sum(Headcount$[`Employee ID` where `Department in Force`$[`Employee ID` = `Employee ID`, Date.Month = this.Date.Month] = this.Department, Date.Month = this.Date.Month])
```

- **Record-key drill-downs (sum of products)**: To compute weighted pipeline by `Stage`, `Owner`, and `Deal`, write the multiplication rule once on the deal leaf (`child_defaults` on `Total[Stage][Owner]`) so parent rows sum the weighted deal results rather than multiplying summed amounts by summed probabilities:

```formula
if(startofmonth(`Expected close`) = Date.Month, Amount * Probability, 0)
```

- **Sparse dimension nesting & empty members**: Nesting source dimensions displays only combinations present in source rows (with missing values under `empty:None`, or all-empty combinations when `show_empty_members` is true).

## Dimensions and mappings

- **Shared classification structure**: A dimension mapping classifies source keys into reporting groups (such as `Account` to statement `Bucket`, or multiple source department lists into one reporting `Department`). Editing the mapping updates every formula and table that reads it, and its `catch_all` bucket keeps unmatched items visible.
- **Two ways to read a mapping in formulas**:
  1. **Over an imported measure** (whose source rows carry the mapping key): Filter directly on the mapped target dimension:

```formula
sum(Amount[Bucket = "COGS"])
```

2. **Over an authored or entered variable** (written at `Account × Month` without `Bucket` in its coordinates): Filter via a key hop (`Account.Bucket`) so the read stays at `Account × Month`:

```formula
sum(Amount$[Account.Bucket = this.Bucket, Date.Month = this.Date.Month])
```

- **Chained hops & scoped measures**: Lookup hops chain across mappings (`this.Project.SourceTeam.Team`). When reading a measure that is already isolated to one scope (such as `R&D Expense` stored by `Account × Month`), drop the outer `Department` coordinate deliberately: ``sum(`R&D Expense`$[Account in any, Date.Month = this.Date.Month])``.

## Tables as views

- **Time-series vs. record tables**: Time-series tables put variables on rows and `Date` on columns; record tables put a record key (`Employee ID`, `Deal`) on rows and variables/dimensions as field columns. Keep large record tables fast by displaying only requested columns and filtering the row dimension to relevant records.
- **Display windows & formatting**: Set `window` explicitly on the table when a report specifies a start and end date, and apply table-specific `format_updates` at the broadest applicable scope (variable-level format acts as the model default).

## Pages and supporting work

| Work                                                                                                  | Placement                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A forecast, dashboard, tracker, report, schedule, or interactive tie-out the person asks you to build | This is the deliverable, even without the word "page". Create or reuse its own page. Keep tables created for it together with its visuals. New periods extend the existing report's tables unless the person or reporting workflow asks for a separate report. |
| Supporting checks and evidence from doing that work                                                   | Reuse a workspace-visible page in `Main` that already holds the same kind of checks or evidence. A report or other deliverable page is not a destination for supporting work.                                                                                  |
| Supporting work with no suitable checks page                                                          | Create one top-level **Ari's working papers** in `Main`, with audience `workspace` when page audiences are enabled. Reuse it across tasks.                                                                                                                     |
| A separate page explicitly requested by the person                                                    | Create that page with the requested content.                                                                                                                                                                                                                   |

- **Working-papers scope**: Pass `Main` explicitly on every page or block call targeting working papers, including destination lookup. Identify each check's source scenario. Preserve other blocks and existing audiences; never repurpose a private page as shared working papers.
- **Private sketches**: `[[sketch-answers]]` uses the person's private **Ari's scratchpad** for a visual answer to one question. Save durable live checks and team evidence on working papers, not the scratchpad.

## Scenarios

- **Decisions within a plan**: Model funding, launches, vendor changes, and hires that can happen together as switches or dated inputs within that plan.
- **What-if questions**: A question about a possible change leaves saved plans unchanged. Calculate it in a temporary scenario, answer, and delete that scenario, unless the person asks to keep it.
- **Scenarios vs. dimensions**: Use scenarios for mutually exclusive alternatives the person wants to keep side by side. Use dimensions for segments that coexist and add up within one plan (regions, tiers, departments).
- **Target scenario selection**: New pages, tables, baselines, reports, and models belong in `workspace_default` (Main) unless the person names a scenario or requests a change to the assumptions of the scenario on screen or the chat's scenario. Every write names the chosen scenario explicitly, including writes to inputs.
- **Planned changes**: Adding planned hires, spending, funding, or other amounts that change a scenario's results changes that scenario's assumptions. Keep the accompanying pages, tables, and inputs in the scenario on screen or the chat's scenario. Building a business baseline or assembling a model from existing schedules follows the default placement rule.
- **Scenario inheritance**: Live scenarios inherit unchanged work from their base; imported source records remain shared and read-only. Work created in another scenario is absent from Main.
- **Factual corrections**: Correct the scenario holding the object's original version. A scenario holding its own version keeps it; name that scenario and offer to correct it too.
- **Placement disclosure**: The opening acknowledgement and final reply name destinations that differ from the screen or chat. When new inputs change amounts in `workspace_default`, explain their effect on every live scenario based on it, except where a scenario keeps its own version. Include this in the opening acknowledgement whenever the screen or chat uses another scenario.
- **Requested assumption values**: Write a value the person gives for an assumption to its existing input in the requested scenario, keeping the formulas that read it. If another lever scales the input or closed periods read it, still write it and report the resulting value and moved periods. Changes the person limits to later periods use the forecast-only rule below.
- **Forecast-only levers & scenario comparisons**: To move only the forecast in a scenario, override dated `Forecast` cells or an input read only by `Forecast` rules (timeless inputs read by all periods move history too). Use `comparison: "none"` with `formula_origins: "all"` to inspect comparison rules.

## Checking sources and disclosing corrections

Answers to open items and exceptions usually live in the tabs of attached workbooks and in connected source records. An item needs the customer's input only when those sources leave a specific business question unanswered.

When a finding contradicts a saved model value, the answer names that value, its table, variable, any dimension items, period (when dated), and the corrected value. This identifies the remaining model change, including when another chat saved the original.

- **QuickBooks transaction identity**: QuickBooks record ids (`TRANSACTION_ID` and QuickBooks tables' `ID` columns) are internal; forms and reports do not show them. A correction request lists each referenced transaction separately, with its type, date, payee, amount, account, memo, and document number as QuickBooks stores it, when present.
- **QuickBooks report basis**: QuickBooks P&L and balance-sheet tables follow the connection's `accounting_basis`, reported by `inspect_integrations`. Cash-basis reports omit unpaid bills. After a basis change, syncs rewrite earlier months on the new basis.
- **Readback verification**: A write receipt with `status: "applied"` proves what changed, replaced, tied, or failed. Verify dated calculations on system `Date` across starting, middle, and ending periods using a member whose state changes inside the window (a hire start, termination, rate change, or FX conversion) plus coarse period rollups.
- **Tracing discrepancies & driver sensitivity**: Compare calculated record cells against source facts (for example, a salaried employee's full month equals annual rate divided by 12).

New scenarios are saved and listed for teammates immediately. Edits in Main or another shared scenario stay live for everyone until restored.

| Saved work                                                                                               | Before handoff                                                                                                       |
| -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| A separate version, a requested side-by-side alternative, or alternatives included in a full model build | Keep the scenario and its planning assumptions after verification.                                                   |
| A scenario created solely to check a build, with no role in the delivered plan                           | Delete it with `edit_scenarios` `delete` once the check finishes, before ending the turn. The receipt supports undo. |
| Temporary test inputs in Main or another shared scenario                                                 | Restore the original inputs.                                                                                         |

- **Performance & scale rules**:
  - **Use only necessary time grain**: Prefer monthly or yearly queries and formulas when daily grain does not change the answer.
  - **Write general formulas**: Reference `Date` and `this.<Dimension>` rather than duplicating rules per year or item.
  - **Share expensive expressions**: Extract repeated sub-expressions into helper variables when doing so reduces calculation time.
  - **Large record sets as sources**: Connected warehouse rows use a saved query on that connection and refresh with each sync. `edit_integrations` `ingest_csv` loads CSV files whose records no connection holds as warehouse rows; source changes require another upload. One aggregate formula over a source replaces hundreds of cell rules.

## Modeling the business

### Request scope

- **Full model builds**: The person asks for a new model, with or without connected data, or a rebuild of the whole model. Apply the full simulation recipe below.
- **Pieces of an existing model**: Build the requested piece at the requested size in the scenario receiving the work (see **Target scenario selection**; usually `Main`). Keep separate scenarios and pages only when requested. Include a simulator or walkthrough when requested.
- **Placement for pieces**: Keep the piece's inputs and calculations on its page or existing pages. A larger build can be offered in your reply.

### Structure, coordinates, and controls

- **Simulation scope and horizon**: A full model build connects revenue, costs, staffing, and cash across the horizon (defaulting to 12 months of history and 24 months of forecast) with Base, Upside, and Downside scenarios, pages, a closing simulator, and a walkthrough.
- **Dimensional depth and from-scratch builds**: Segment variables across the dimensions where unit economics, pricing, margins, or timing differ (plan, segment, region, channel, SKU, department, currency). When building from scratch without connected sources, deliver this same full dimensional and operational simulation grounded in company research and industry benchmarks rather than a simplified overview.
- **Lumpy, time-based events over smooth curves**: Where reasonable across all model builds, model discrete, time-based events with adjustable dates and quantities as keyed records or scheduled items rather than approximating flows with smooth curves or flat averages. Shifting an event's date input (an individual hire or hiring cohort start date, a facility or data center build-out, an equipment purchase, a contract start, or a financing tranche) must automatically shift the resulting payroll, capex, revenue, and cash flows across periods.
- **Records and placeholders**: Represent people, hiring cohorts, customers, contracts, deals, and major capital commitments as keyed records grounded in connected data, files, or cited company evidence, filling gaps with role-named placeholders (`Software Engineer 2`), hiring cohorts, or customer cohorts reconciled to stated totals.
- **Shared controls**: For a full model build, place shared editable levers (event dates, growth, pricing, churn, hiring timing, payment terms) on one controls page that drives downstream outputs and the closing simulator.

### Revenue and pipeline

- **Revenue schedules**: Model revenue by how the business earns it: recurring-revenue schedules track opening customers, additions, churn, ending customers, and ARPU; transaction, marketplace, and contract models follow unit economics and recognition terms; commercial step changes (enterprise contract starts, pricing changes, product or market launches) trigger from their scheduled dates. Keep bookings, recognized revenue, billing, and cash collection distinct.
- **Pipeline & deal registers**: Separate deals into three distinct measures by state and governing date:
  - **Bookings (won deals)**: Placed in the month the deal closed.
  - **Open pipeline / Weighted pipeline (open deals)**: Placed in the expected close month (weighted at the deal leaf via `Amount * Probability`, summed above).
  - **Lost deals**: Tracked separately when requested.
  - Disclose how many deals lack an owner and which source stages map to won vs. open.

### Costs and staffing

- **Operating costs & lumpy capital commitments**: Separate Cost of Revenue (delivering the sold product/service) from Operating Expenses (R&D, S&M, G&A) per company policy, driving variable costs by their operational unit (transactions, users, compute, active capacity, headcount) rather than flat revenue percentages. Where capital expenditures or major operating outlays occur in discrete steps (such as data center or facility build-outs, hardware clusters, tooling runs, or major vendor contracts), model each project or phase as a dated item with an adjustable date and cost schedule rather than smoothing capex into a flat average:

```formula
if(`Build Date` <> null and startofmonth(`Build Date`) = Date.Month, `Project Capex`, 0)
```

- **Headcount & payroll plans**: Deliver both a current-state roster and a monthly staffing table keyed by `Employee ID` (or hiring cohort):
  - **Timing-driven headcount and payroll**: Drive changes in headcount and payroll costs from the editable start and termination dates of individual employees, planned hires, or hiring cohorts—never by applying a smooth percentage growth curve to aggregate payroll. Each row's active `Headcount` is `1` (or the cohort headcount on a cohort row) in months employed (by default, any month between start and termination inclusive) and `0` otherwise, and monthly pay scales by `Headcount` and prorates partial start or termination months by active days so moving a hire date immediately shifts headcount and payroll cost:

```formula
if(`Start Date` <> null and startofmonth(`Start Date`) <= Date.Month and (`End Date` = null or startofmonth(`End Date`) >= Date.Month), Headcount * (`Annual Pay` / 12) * (datedif(if(`Start Date` > Date.Month, `Start Date`, Date.Month), if(`End Date` <> null and `End Date` < eomonth(Date.Month, 0), `End Date`, eomonth(Date.Month, 0)), "D") + 1) / daysin(Date.Month, "month"), 0)
```

- **Compensation**: Estimate pay only as a role-level rate each employee row reads, reserving person-level pay for sourced or user-supplied facts and explaining estimates in your reply. Convert annual (`Annual Pay / 12`), monthly, or hourly rates (with disclosed hours), prorating partial months, adding taxes/benefits, and keeping contractors, unpaid founders (`0`), uncategorized departments, and currencies explicit.

### Cash and financial statements

- **Cash & runway**: Project ending cash as opening cash (seeded from books or a disclosed assumption) plus dated operating, working-capital, capex, debt, and financing flows. Use live accounting open balances over lagging CRM statuses for overdue collections, and state the exact cash constraint date (or report that cash stays positive across the horizon).
- **Income Statement (P&L)**: Map source ledger accounts into statement lines (`Bucket`) via `edit_dimension_mappings` `map` (or inline via `dimensions` on `edit_table_blocks` `create`), keeping `Unclassified` visible in reconciliation totals. Write `Gross Profit` and `Operating Profit` as row formulas over the statement measures (`Revenue - Cost of Revenue`), and nest `Account` beneath each measure to show account detail.
- **Balance Sheet**: Show period-end balances (`rollup: "LAST"`) across `Total assets`, `Total liabilities`, and `Total equity` (including provider-generated `Net Income` and `Retained Earnings` rows, while excluding `Section` presentation subtotals). Verify `Assets = Liabilities + Equity` in every period and name the exact amount, period, and accounts of any residual.
- **Cash-Flow Statement**: Reconcile operating, investing, and financing cash flows to the change in Balance Sheet cash. Under the indirect method, start from Net Income, include each working-capital, capex (using gross fixed-asset additions so depreciation addbacks are not double-counted), debt, and equity cash movement once, and exclude cash and retained-earnings net-income rollups.
- **Statement forecasts**: Write P&L and balance-sheet forecasts on account lines; keep each total's generated sum of those lines. A saved total formula replaces that sum for its months, so the total stops showing its account lines. If those account lines are blank, report that and ask before replacing the total's formula.
- **Forecast run-rate**: Before reporting forecasts, compare every driver, including chosen assumptions, with its last three actual months on the same definition: cost account lines (first forecast quarter, booked amounts, not typed totals), collections, debtor and creditor days, and targeted KPIs (also against targets). Quantify each step change or target gap, its months and source: user instructions, a recorded change, or none found.

### Comparisons, anomalies, and final delivery

- **Budget vs. Actuals comparisons**: Align plan and actuals over the latest complete shared period using existing measure mappings. Compute `Variance = Actual - Plan` and `Variance % = Variance / abs(Plan)` (omitting percentage when Plan is zero).
- **Anomaly & tie-out checks**: Distinguish internal model consistency (`0` difference between two views reading the same saved rows) from external source tie-outs (comparing model totals against independent source P&L, trial balance, or bank totals). Balance and cash checks prove arithmetic only; forecast completeness needs every account line forecast and the run-rate comparison.
- **Delivery summary**: Link the saved page and state the method, key assumptions, commitments, coverage gaps, and whether source rows represent restated snapshots or additive transactions.
- **Earlier figures**: When a new figure measures the same thing as one stated earlier in the chat, including one from before a restart or rebuild, the reply names the earlier figure and splits the difference into input or source data changes and method changes.

The generated reference below is the complete function inventory and the authority for every accepted spelling; its notes carry the places the engine differs from Excel.

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

- `%`: between two values `%` is modulo (`Months % 12`, `100 % 3`, `Months % -1`); a number with `%` is one percent value (`50%` is `0.5`, `2 ^ 50%` is `2 ^ 0.5`, and `75% - Fee` is `0.75 - Fee`); a trailing `%` after another expression is its percent (`(a + b)%`)
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
- **Dimension items.** `$[Location = "East Coast"].Location` is a dimension-item literal that pins one dimension to a string and reads that item back. Prefer it over a bare string in comparisons (`this.Location = …`) or filter values, and include `$` so row dimensions do not leak in. `$[Location = "East Coast"].Revenue` reads `Revenue$[Location = "East Coast"]`.
- **Mapped dimensions read like values.** `Bucket$[Account = "5001"]` evaluates to that account's mapped bucket, and `this.Bucket` or `Account.Bucket` (`Bucket$[Account = this.Account]`) follows mapping keys across dot hops without inheriting unrelated dimensions like Month. In predicates, `sum(Amount[Bucket = "COGS"])` groups source rows by mapped bucket, while authored variables roll up through their key (`Amount$[Account.Bucket = this.Bucket]`).

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
- `count(value: any, value2...: any) -> number`: Takes 1 to 10 arguments. Counts non-empty values (like Excel COUNTA). A bracketed condition keeps filtered dimensions and replaces the row item: `count(Deal[Region in {"East", "West"}])` counts Deal × Region pairs, `count(Deal$[Deal where count(Deal[Deal = Deal, Region in {"East", "West"}]) > 0])` counts distinct deals, and `if(count(...) > 0, 1, 0)` flags a deal.
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
