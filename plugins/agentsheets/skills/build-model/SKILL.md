---
name: build-model
description: "Use when building, editing, or troubleshooting cfo.ai models: variables, formulas, dimensions, mappings, rollups, tables, scenarios, statements, cash, or BvA."
---

# Build the model

A workspace has one shared model; tables are views over it. Variable names, formulas, and values belong to the shared model; row order, formatting, and breakdowns belong to each table block.

- **Reusing vs. creating:** Pass `new: true` when a report reuses a name for a distinct concept. Plain names reuse an existing variable without overwriting saved settings; `#hex` refs overwrite shared defaults.
- **Deleting shared objects:** Inspect dependents first and offer **Cancel** and **Delete completely**; when Ari can start a new chat, also offer **Keep with Ari** (moving preserved objects to a new page first), else ask for a destination page. System `Date` cannot be deleted.
- **Parallel calls:** Same-turn calls run concurrently—sequence dependent writes across steps or bundle table layout and formulas in one `edit_table_blocks` (`create`).

## How the Model Works

Running example: monthly listening service with tiers (`Lark`, `Chorus`) and territories (`Southgate`, `Eastmere`).

### Cells, Rules, and Ranges

- **Cell & inputs:** One variable at one item per dimension (plus period if dated). Put editable assumptions in input cells and formulas in calculated series (never grids of repeated literals). Single-cell overrides pin one cell; clearing restores the default formula and rollups.
- **`Actuals`, `Forecast`, `Last close`:** A formula's condition sets its `Date` grain (`Month`, `Day`, etc.). `Actuals` and `Forecast` have separate rule slots split at `Last close`—write both when a rule spans the horizon. Put post-close events in `Forecast` and disclosed closed-period estimates in `Actuals` without moving `Last close`.
- **Opening seeds and `[-1]` recurrences:** Seed the opening value in the last `Actuals` month and write the carry-forward rule on both `Actuals` and `Forecast` (typed value wins in its month; advancing `Last close` preserves the rule—on `Forecast` only, newly closed months read `0`):

```formula
`Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory])
```

- **Pre-seed months & imported recurrences:** In `Actuals`, the formula also runs before the seed (calendar month 1 reads an empty `[-1]`). Return `NULL` when `[-1]` is null to keep earlier months blank (same for opening cash + `Net Cash Flow`):

```formula
if(`Weekly listeners`[-1] = null, NULL, `Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory]))
```

In the first `Forecast` month, `[-1]` reads the last `Actuals` input, imported value, or `Actuals` rule (`0` if absent). Keep imported-variable recurrences on `Forecast` only (an `Actuals` rule replaces source reads across `Actuals`).

<!-- Keep compact 3-column table unpadded to save skill tokens. -->
<!-- prettier-ignore -->
| Empty/null case | Value | Pattern |
| --- | --- | --- |
| Unset date/text | `null` (`End Date = null` is true) | `a = null` |
| Unset numeric | `0` | `iferror(x / y, NULL)` |
| Blank raw column | `null` (`last` = `null`; `count` skips) | `(a = null and b = null) or a = b` |
| `SUM` over all-blank rows | `0` (`count` counts `0`) | Guard zero denominators with `iferror` |

### Reads, Lookups, and Breakdowns

- **Relative vs. absolute (`$`):** Plain names read at current cell coordinates. When an input has fewer dimensions (e.g. `Weekly listeners` by `Territory × Month`, no tier), a relative read at `Chorus × Eastmere × March` returns `0`; pin only the input's coordinates with `$`:
  - Dated 1D: `` `Weekly listeners`$[Territory = this.Territory, Date.Month = this.Date.Month] ``
  - Timeless 1D: `` `Tier fee`$[`Listening tier` = this.`Listening tier`] ``
  - **Parent vs. child lines:** Row formulas also run on breakdown lines with no rule of their own; a coarser `$` read repeats on each child, so a `SUM` parent adds one copy per child.
- **Brackets and `where`:** Brackets override named coordinates and inherit the rest (`` `Listening revenue`[Territory = "Eastmere"] ``, ``sum(`Listening revenue`[Territory in any])``; predicate values accept expressions like ``Date.Month = startofmonth(`Last Snapshot Month`$)``). Outside `where`, bare `Dim` = `this.Dim`; inside `where`, bare `Dim` = candidate record and `this.Dim` = current cell. Filtering a second dimension keeps the row's drilled coordinate; filtering the row's own dimension (`sum(Amount[Account in {"A", "B"}])`) replaces it and repeats the total on every row.
- **`child_defaults` & `PERCENTAGE`:** Use `child_defaults` at the target dimension level so future items inherit standing rules (leaf `child_defaults` generates `level_default_sync` sums up each parent). `PERCENTAGE` breakdowns never sum children or fall back to `0` on leaves: if all leaf inputs exist on the parent (`Profit / Revenue`), the parent evaluates on parent totals; if leaves hold typed rates, mixed rules, or `this.Dim`, write an explicit weighted formula or dateless row `formula`. Set `summary: "NONE"` on a child column to run the row's formula on its grouped column header.

### Time, Rollups, and Offsets

- **System `Date` & rollups:** System `Date` drives the table calendar so forecast periods exist before data lands; match fiscal-year labels (`FY26`) to existing month labels. Dated rules run at base grain (usually `Month`) and roll up (`SUM` flows, `LAST` balances/counts/rates via `rollup` or `set_time_rollup`, `AVERAGE` period means). Write ratios (``iferror(`Listening revenue` / `Paying listeners`, NULL)``) as a dateless row `formula` so every grain evaluates directly.
- **Offsets & proration:** Use `dateadd` or relative windows (`[-3]:[-1]`) for period comparisons; prorate across `[Subscription in any, Date.Month in any]` with `datedif(start, end, "D") + 1` over `daysin(Date.Month, "month")`.

## Worked Build

Two timeless input tables:

```
{"action": "create", "name": "Tier assumptions", "rows": [{"dimension": "Listening tier", "items": ["Lark", "Chorus"]}], "columns": [{"variable": "Tier fee"}, {"variable": "Uptake rate"}], "values": {"columns": ["Tier fee", "Uptake rate"], "rows": [["Lark", 20, 0.04], ["Chorus", 50, 0.01]]}}
{"action": "create", "name": "Territory assumptions", "rows": [{"dimension": "Territory", "items": ["Southgate", "Eastmere"]}], "columns": [{"variable": "Listener growth"}], "values": {"columns": ["Listener growth"], "rows": [["Southgate", 0.02], ["Eastmere", 0.03]]}}
```

Dated count table (seed in last Actuals month `2026-08`; growth rule on both ranges; `rollup: "LAST"`):

```
{"action": "create", "name": "Listening", "columns": [{"time": "MONTH"}],
 "rows": [{"variable": "Weekly listeners", "breakdown": ["Territory"], "rollup": "LAST",
   "child_defaults": [{"dimension": "Territory",
     "actuals": "if(`Weekly listeners`[-1] = null, NULL, `Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory]))",
     "forecast": "`Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory])"}]}],
 "values": {"columns": ["2026-08"], "rows": [["Weekly listeners[Southgate]", 100000], ["Weekly listeners[Eastmere]", 60000]]}}
```

Downstream breakdown (pins coarser inputs with `$`; parent sums auto-generate; dateless ratio works at every level/grain):

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

Verify: first Forecast month grows from seed (not 0); `Chorus × Eastmere` Sep = `61,800 × 0.01 × 50 = 30,900`; `Chorus` = sum of both territories.

## Rule Reach and Precedence

- **Relative (`[...]`) vs. exact (`$[...]`)**: Relative claims named depth and deeper (`[Stage in any]` reaches stage, owner, deal); exact claims only that depth. A parent sum fanning into child cells without a finer rule causes `#CIRC!`.
- **Precedence (highest -> lowest):** (1) Exact `$[...]` > Relative `[...]`; (2) More dimensions > fewer > `[]`; (3) Pinned `= "Won"` > Set `in {...}` > `in any`/`where`; (4) Newest saved rule > older > source/zero.
- **Row-to-condition & debugging:** `child_defaults` on pinned row (`Scoping`) saves `[Stage = "Scoping", Owner in any]`; on open axis (`Total[Stage][Owner]`) saves `[Stage in any, Owner in any, Deal in any, Date.Month in any]`. Debug via `inspect_table_blocks` (`formulas`, `inherited_from`, `formula_origins`, `include_inputs`, `comparison: "none"`). Unparseable expressions -> `#ERROR!`; unparseable conditions match nothing.

## Imported Records, Sources, and Mappings

- **Profiling warehouse tables:** Use `inspect_source_data` (`profile`) first to get row count, column types, `values_present`, and 5 sample rows in one compact call—avoid `SELECT * LIMIT 100` dumps.

- **External columns & post-close reads:** Imported variables aggregate columns (`sum(<runway:exttables/550e8400-e29b-41d4-a716-446655440000/columns/Amount>)` for additive rows; `last(<column>)` for observations). Relative reads aggregate one observation per period; `$` over a restated source sums all snapshots unless filtered. Un-ranged single-column source reads return `after_last_close` and stop at `Last close`—add `default_formulas.forecast: "sum(<column>)"` to read post-close rows (other `[]` expressions run across all periods). `preferred_time_property` sets calendar placement and `first`/`last` order.
- **Ledgers, snapshots, and staffing:** Sum transaction ledgers. For restated syncs, inspect preview first (identical rows across stamps = repeated current state; differing = history) and filter by snapshot month (`ObservedAt.Month = this.Date.Month` or `date(2025, 11, 1)`). Timeless roster (`Employee ID` rows, no `Date`) shows `last(...)` current state; monthly staffing (`Date` columns) carries per-employee state on `[`Employee ID` in any]` and sums active headcount on `[]`. For changing departments, avoid `$` in Actuals or read dated `Department in Force`:

```formula
sum(Headcount$[`Employee ID` where `Department in Force`$[`Employee ID` = `Employee ID`, Date.Month = this.Date.Month] = this.Department, Date.Month = this.Date.Month])
```

- **Sum of products & sparse nesting:** Multiply at the deal leaf (`child_defaults` on `Total[Stage][Owner]`) so parents sum weighted deals instead of multiplying summed amounts × summed probabilities:

```formula
if(startofmonth(`Expected close`) = Date.Month, Amount * Probability, 0)
```

Nested source dimensions show only present combinations (missing -> `empty:None`).

- **Dimension mappings:** Map source keys to reporting groups (`Account -> Bucket`, multi-source depts -> `Department`) with a `catch_all`. Read over an imported variable via `sum(Amount[Bucket = "COGS"])`, or over an authored variable via key hop (`sum(Amount$[Account.Bucket = this.Bucket, Date.Month = this.Date.Month])`; hops chain: `this.Project.SourceTeam.Team`). Drop outer dims deliberately when reading scoped variables (``sum(`R&D Expense`$[Account in any, Date.Month = this.Date.Month])``).
- **Unsplit mapped classes:** When a table groups by a mapped dimension (e.g. `Class`) and a read/receipt flags formulas saved by another group on the same source key, those groups do not split automatically: fetch omitted groups via `next`, add/update a text block right below (`edit_pages` `add_text` with `after_block`) naming the source and unsplit groups in the same turn, ask once how each splits, and keep saved amounts until answered.

## Tables, Pages, and Scenarios

- **Layout & field count:** Time series: variables on rows, `Date` on columns. Record tables: record keys on rows, fields on columns (filter large tables). Show requested fields; else all up to 12, or for wider sources <=12 identifying/actionable fields (key, date, counterparty, amount, status, owner) with remaining audit fields on a second keyed table. Set `window` explicitly when dates are named; apply `format_updates` at the broadest scope. Set units, scale and decimal places on the table block so the page and Excel export use the same number formats. Keep the underlying values at source precision.
- **Page placement:**
  - **Deliverables** (forecast, dashboard, tracker, report, schedule, tie-out): own page (create or reuse) with tables beside visuals; extend existing report tables for new periods unless asked for a new report.
  - **Supporting checks:** Reuse a workspace-visible checks page in `Main` (never a deliverable/report/private page), or create/reuse top-level **Ari's working papers** in `Main` (`audience: "workspace"` if enabled). Pass `Main` on every call, name each check's source scenario, preserve other blocks/audiences. Distinguish **internal tie-outs** (`Assets - Liabilities - Equity = 0`, `Cash change - Net Cash Flow = 0`, `Sum of parts - Total = 0`) from **external reasonableness / sense checks** comparing every headline output against independent market benchmarks, peer comparables, or explicit domain judgment ranges rather than internal tautologies.
  - **One-off sketches:** `[[sketch-answers]]` (**Ari's scratchpad**, private) only for temporary sketches, never durable team checks. Create an explicitly requested separate page as asked.
- **Scenarios:** Coexisting options -> switches/dated inputs in one plan; mutually exclusive alternatives -> scenarios; coexisting segments -> dimensions; what-if questions -> temporary scenario unless asked to keep. Put new pages, tables, baselines, reports, and models in `workspace_default` (`Main`) unless the user names a scenario or edits screen/chat scenario assumptions; name target scenario on every write. Live scenarios inherit unchanged base work (imports are shared read-only); fix factual errors in the owning scenario (name and offer to fix overridden copies). Disclose in opening/closing replies when target != screen/chat scenario and note `workspace_default` ripple effects. Write assumption updates to existing inputs preserving formulas, and report resulting values and any moved closed periods; for forecast-only moves, edit dated `Forecast` cells or Forecast-only inputs so timeless inputs don't shift history. Delete temporary/check scenarios (`edit_scenarios` `delete`) and restore temporary shared test inputs before ending the turn.
- **Performance:** Prefer monthly/yearly grain over daily when results match; write general `Date`/`this.Dim` formulas over per-period copies; extract repeated sub-expressions into helper variables; use saved queries (`ingest_csv`) over hundreds of cell rules.

## Checking Sources and Disclosures

- Inspect workbook tabs and connected sources before asking unanswered business questions.
- **Overriding imported rules:** Record table, chosen rule, reason, and `runway-file-...` ID on the import summary page so future file updates preserve it.
- **Contradictions & earlier figures:** Cite saved value, table, variable, segments, period, and corrected value. When a new figure measures the same thing as an earlier chat figure (incl. before restart/rebuild), name the earlier figure and split difference into data/input vs. method changes.
- **QuickBooks:** Internal IDs (`TRANSACTION_ID`, `ID`) don't appear in QBO UI—cite type, date, payee, amount, account, memo, doc number. P&L and BS follow `accounting_basis` from `inspect_integrations` (cash omits unpaid bills; basis changes rewrite history).
- **Verification:** Check `status: "applied"` readbacks across start/mid/end periods on mid-window changes (hire, termination, rate change, FX) + coarse rollups, and sanity-check record cells against source facts (salaried month = annual / 12). When building from attached workbooks, recompute every headline output independently from literal source inputs after identifying formula cells and reconcile against both `inspect_table_blocks` readbacks and recalculated Excel outputs before delivery. When delivering an Excel workbook, run `cfoai_workbook.recalculate_workbook` (passing `baseline` set to the starting workbook path when retained input tabs stay untouched, or omitting `baseline` when standardizing the full workbook); inspect all `[ERROR]` and `[CHECK]` diagnostics, fix genuine defects, and re-run before `deliver_files`.

## Modeling the Business

- **Full vs. partial build:** Full build delivers connected revenue, costs, staffing, and cash (default 12 mo history + 24 mo forecast), Base/Upside/Downside scenarios, one shared controls page, dimensional depth grounded in research/benchmarks, keyed records/placeholders (`Software Engineer 2`), a Canvas simulator, and a `present_guide` walkthrough. Partial build delivers only the requested piece on its own/existing pages (add extra scenarios/simulators/walkthroughs only when asked; offer broader build in chat).
- **Discrete events:** Model hires, cohorts, capex, contracts, and financings as dated records so shifting a date moves downstream lines automatically (never flat smooth curves):

```formula
if(`Build Date` <> null and startofmonth(`Build Date`) = Date.Month, `Project Capex`, 0)
```

- **Revenue & pipeline:** Match business mechanics (SaaS: opening/additions/churn/ending × ARPU; keep bookings, recognized revenue, billings, cash distinct). Split deals into **Bookings** (won, close month), **Open / Weighted pipeline** (open, expected close month, leaf `Amount * Probability`), and **Lost** (if asked); disclose ownerless deals and stage mappings.
- **Costs & headcount:** Separate COGS from OpEx (R&D, S&M, G&A) driven by operational units. Pair a roster with monthly staffing keyed by `Employee ID` or cohort driving `Headcount` (`1`/count while active, else `0`) and prorated monthly pay:

```formula
if(`Start Date` <> null and startofmonth(`Start Date`) <= Date.Month and (`End Date` = null or startofmonth(`End Date`) >= Date.Month), Headcount * (`Annual Pay` / 12) * (datedif(if(`Start Date` > Date.Month, `Start Date`, Date.Month), if(`End Date` <> null and `End Date` < eomonth(Date.Month, 0), `End Date`, eomonth(Date.Month, 0)), "D") + 1) / daysin(Date.Month, "month"), 0)
```

Estimate pay via role-level rates (person-level only from sourced/user facts; disclose estimates), include taxes/benefits, keep contractors, unpaid founders (`0`), uncategorized depts, and currencies explicit. Before reporting payroll, flag any planned hire matching a roster/payroll record by name or role + start month: open the reply with that pair and each monthly pay, state totals after it, and keep both until the user decides.

- **Statements & cash:**
  - **Standard accounting, historical calibration, & precision:** Keep full floating-point precision in intermediate formulas; apply `ROUND(...)` only when explicitly requested and state `"rounded to N decimals"` **only** on cells/sheets using `ROUND(...)` (`"displayed to N decimals; full precision retained in formulas"` elsewhere). Before building, verify exact output definitions, units, period timing (beginning vs. end of period), day-count/stub conventions, and sign conventions from the request, template labels, or existing formulas: for relative variances, inspect whether source costs are stored positive or negative and verify on a line where `Actual / Plan - 1` (signed denominator, matching cfo.ai's built-in comparison) and `(Actual - Plan) / ABS(Plan)` (favorable/unfavorable sign) differ; on Budget vs. Actuals comparisons, reconcile the union of both sides' keys independently to each source total (100% coverage). Whenever the source holds historical values or worked rows, back-test candidate forecast rules and metric definitions (`Actual - Implied = 0`) on a period where competing definitions diverge before locking formulas, then stress-test editable inputs on a disposable copy (driving a divisor to zero and extending a schedule one period past its range) to confirm error-free, un-truncated behavior. For cash waterfalls and debt sweeps, verify priority order from source facts, solve same-period average-balance revolver interest in closed form to prevent `#CIRC!`, and verify cash and revolver invariants every period. Unless source facts or instructions specify otherwise: `Gross Profit = Revenue - COGS` (excluding other operating income); per-share, `ROE`, and parent margins use `Net Income Attributable to Common` and `Common Equity`; `EBITDA` and `FFO` add back all operating D&A lines.
  - **Cash:** Opening cash + dated operating, working-capital, capex, debt, and financing flows. Prefer live accounting open balances over lagging CRM for overdue collections; state exact cash constraint date (or positive across horizon).
  - **P&L:** Map ledger accounts to `Bucket` (keep `Unclassified` visible); compute `Gross Profit`/`Operating Profit` over statement variables and nest `Account` beneath.
  - **Balance Sheet:** `rollup: "LAST"` on `Total assets`/`Total liabilities`/`Total equity` (include provider `Net Income`/`Retained Earnings`, exclude `Section` subtotals). Verify `Assets = Liabilities + Equity` every period; report any residual's amount, period, accounts.
  - **Cash Flow:** Reconcile to BS cash change. Indirect: Net Income + each working-capital, gross capex (never double-count depreciation), debt, and equity movement once (exclude cash and Retained Earnings net-income rollups). Direct: classified bank movements (e.g. `Xero Cash Flow` standard queries once journals sync).
  - **Account-level forecasts & run-rate check:** Forecast on account lines while keeping parent total sums (ask before replacing a total if lines are blank). Before reporting forecasts, compare every variable against its last 3 actual months on the same definition (Q1 booked costs, collections, debtor/creditor days, targeted KPIs) and quantify any step change/gap with months and source (user instruction, recorded change, or none found).
- **BvA & delivery summary:** Align plan and actuals over latest complete shared period using existing mappings (`Variance = Actual - Plan`; use the requested or template variance-% convention, or `Variance / abs(Plan)` when unspecified and Plan != `0`). Distinguish internal consistency from external tie-outs (balance/cash prove arithmetic only; forecast completeness needs account-level forecasts + run-rate check). Link the page and state method, key assumptions, commitments, coverage gaps, and additive vs. restated rows.

Full grammar, operators, and functions: `[[formula-reference]]`.
