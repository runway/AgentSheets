---
name: build-model
description: "Use when building, editing, or troubleshooting cfo.ai models: variables, formulas, dimensions, mappings, rollups, tables, comparisons, scenarios, headcount, pipeline, statements, cash, or BvA."
---

# Build the model

A workspace has one shared model; tables are views over it. Variable names, formulas, and values belong to the shared model; row order, formatting, and breakdowns belong to each table block.

- **Reusing vs. creating variables:** Pass `new: true` when a report reuses an existing name for a distinct business concept. Plain names reuse an existing variable without overwriting saved settings, while `#hex` refs overwrite shared defaults.
- **Deleting shared objects:** Inspect dependency impact first and offer **Cancel** and **Delete completely**. When Ari can start a new chat, also offer **Keep with Ari** (which places the preserved object on a new page before removal); otherwise ask for a destination page for safekeeping. System `Date` cannot be deleted.
- **Parallel tool calls:** Calls in one turn run concurrently. Sequence dependent writes across steps, or bundle table layout and formulas in one `edit_table_blocks` (`create`) call.
- **Overriding imported rules:** When the user chooses a rule over an imported one, record the table, chosen rule, reason, and source `runway-file-...` ID on the import summary page so future file updates preserve the choice.

## The picture

Running example: a monthly listening service with tiers (`Lark`, `Chorus`) and territories (`Southgate`, `Eastmere`).

### Cells, rules, and ranges

- **Cell:** One variable at one item of each dimension (plus one period when dated). `Paying listeners` at `Chorus × Eastmere × March 2027` is dated; `Tier fee` at `Chorus` is timeless.
- **Inputs vs. formulas:** Put editable assumptions in literal input cells and use formulas for calculated series rather than grids of repeated literals. Single-cell overrides pin one coordinate; clearing them restores the default formula and updates rollups.
- **`Actuals`, `Forecast`, and `Last close`:** A formula's condition sets its `Date` grain (`Month`, `Day`, etc.). `Actuals` and `Forecast` have separate rule slots split at `Last close`, so write both when a rule spans the horizon. Put known post-close events in `Forecast` and disclosed estimates for closed periods in `Actuals` without moving `Last close`.
- **Opening seeds and `[-1]` recurrences:** Enter an assumption's opening value in the last `Actuals` month and write its carry-forward rule on both `Actuals` and `Forecast`. The typed value wins in its month, and advancing `Last close` preserves the rule (on `Forecast` only, newly closed months read `0` and later months continue from `0`):

```formula
`Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory])
```

- **Pre-seed months:** In `Actuals`, the formula also runs before the seed, and the calendar's first month reads an empty `[-1]` (so a running balance adds prior flows). Return `NULL` when `[-1]` is null to keep earlier months blank (same pattern for opening cash plus `Net Cash Flow`):

```formula
if(`Weekly listeners`[-1] = null, NULL, `Weekly listeners`[-1] * (1 + `Listener growth`$[Territory = this.Territory]))
```

- **First `Forecast` month and imported recurrences:** In the first `Forecast` month, `[-1]` reads the last `Actuals` input, imported value, or `Actuals` rule (`0` if absent). For imported variables, keep recurrences on `Forecast` only, because an `Actuals` rule replaces the source read in all `Actuals` months.

| Empty or null case                 | Evaluates to                                    | Formula pattern                                          |
| ---------------------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| Unset date or text cell            | `null` (`End Date = null` is true)              | `a = null` for missing checks                            |
| Unset numeric cell                 | `0`                                             | `iferror(x / y, NULL)` for ratios                        |
| Blank imported raw column          | `null` (`last` is `null`; `count` skips blanks) | `(a = null and b = null) or a = b` for nullable equality |
| `SUM` variable over all-blank rows | `0` (`count` counts the `0`)                    | Guard zero denominators with `iferror`                   |

### Reads, lookups, and breakdowns

- **Relative vs. absolute (`$`):** Plain names read at the current cell's coordinates. When an input has fewer dimensions than the cell (such as `Weekly listeners` by `Territory × Month`, with no tier), a relative read at `Chorus × Eastmere × March` returns `0`; pin only the input's coordinates with `$`:
  - Dated 1D: `` `Weekly listeners`$[Territory = this.Territory, Date.Month = this.Date.Month] ``
  - Timeless 1D: `` `Tier fee`$[`Listening tier` = this.`Listening tier`] ``
  - **Parent vs. child lines:** A row's formula also runs on breakdown lines that have no rule of their own. A coarser `$` read repeats on each line (for example, `Weekly listeners` by `Territory` repeats on both `Lark` and `Chorus` and sums to 2× on a `SUM` parent).
- **Bracket overrides (`[...]`):** Brackets override named coordinates and inherit the rest (`` `Listening revenue`[Territory = "Eastmere"] ``, ``sum(`Listening revenue`[Territory in any])``). Predicate values accept expressions such as ``Date.Month = startofmonth(`Last Snapshot Month`$)``.
- **`where` and dimension filters:** Outside `where`, bare `Dim` equals `this.Dim`; inside `where`, bare `Dim` is the candidate record and `this.Dim` is the current cell. Filtering a second dimension keeps the row's drilled coordinate (``sum(Amount[`Account Type` = "Income"])`` by `Account`), whereas filtering the row's own dimension (`sum(Amount[Account in {"A", "B"}])`) replaces it and repeats the total on every row.
- **Breakdowns and `child_defaults`:** "If X, then use Y" is a standing rule: use `child_defaults` at the target dimension level so future items inherit it. Writing `child_defaults` on the leaf generates `level_default_sync` sums up each parent level.
- **Percentage breakdowns (`PERCENTAGE`):** Never summed across children and never fall back to `0` on leaves. If all leaf inputs exist on the parent (`Profit / Revenue`), the parent evaluates the formula on parent totals; if leaves hold typed rates, mixed rules, or `this.Dim` terms, write an explicit weighted formula or dateless row `formula` (below).
- **Grouped column headers (`summary`):** Set `summary` on the child column to control its parent header (`NONE` runs the row's own formula).

### Time, rollups, and offsets

- **System `Date` and fiscal years:** System `Date` drives the table calendar so forecast periods exist before data lands; source date columns do not. Match fiscal-year labels (`FY26`) to the customer's existing month labels.
- **Time rollups (`rollup`):** Dated rules run at base grain (usually `Month`) and roll up to coarser columns (`Quarter`, `Half`, `Year`):

| Rollup setting     | When to use                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| `SUM` (default)    | Flows (revenue, expenses).                                                                          |
| `LAST`             | Balances, counts, and constant prices, rates, or shares (change existing via `set_time_rollup`).    |
| `AVERAGE`          | Period means.                                                                                       |
| Dateless `formula` | Ratios (`iferror(`Listening revenue`/`Paying listeners`, NULL)`) evaluated directly at every grain. |

- **Offsets and proration:** Use `dateadd` or relative windows (`[-3]:[-1]`) for period comparisons. Prorate across `[Subscription in any, Date.Month in any]` with `datedif(start, end, "D") + 1` and `daysin(Date, "month")`.

## Worked build

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

Downstream breakdown (pins coarser inputs with `$`; parent sums auto-generate; dateless ratio stays accurate at every level and grain):

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

Verify: first Forecast month grows from seed (not 0); `Chorus × Eastmere` in Sep = `61,800 × 0.01 × 50 = 30,900`; `Chorus` equals its two territories.

## Rule reach and precedence

- **Relative (`[...]`) vs. exact (`$[...]`) conditions:** Relative claims the named depth and deeper (`[Stage in any]` reaches stage, owner, and deal); exact claims only that depth. A parent sum fanning into child cells with no finer rule causes `#CIRC!`.

| Priority | Precedence rule    | Higher                                          | Lower                                       |
| -------- | ------------------ | ----------------------------------------------- | ------------------------------------------- |
| 1        | Match mode         | Exact (`$[...]`)                                | Relative (`[...]`)                          |
| 2        | Dimension count    | More dimensions (`[Stage in any, Deal in any]`) | Fewer (`[Stage in any]`, then `[]`)         |
| 3        | Filter specificity | Pinned item (`= "Won"`)                         | Set (`in {...}`), then `in any` or `where`  |
| 4        | Tie-breaker        | Newest saved formula                            | Older formula, then source or zero fallback |

- **Row-to-condition mapping:** `child_defaults` on a pinned row (`Scoping`) saves `[Stage = "Scoping", Owner in any]`; on an open breakdown axis (`Total[Stage][Owner]`) it saves `[Stage in any, Owner in any, Deal in any, Date.Month in any]`.
- **Debugging:** Inspect via `inspect_table_blocks` (`formulas`, `inherited_from`, `formula_origins`, `include_inputs`, and `comparison: "none"` for underlying formulas on comparison tables). Unparseable expressions show `#ERROR!`; unparseable conditions match nothing.

## Imported records and sources

- **External columns:** Imported variables aggregate source columns (`sum(<runway:exttables/550e8400-e29b-41d4-a716-446655440000/columns/Amount>)` for additive transactions; `last(<column>)` for latest observations). Relative reads let the variable's aggregation pick one observation per period; an absolute `$` read over a restated source sums every snapshot unless filtered.
- **Source reads after `Last close`:** Un-ranged single-column source reads return `after_last_close` in diagnostics and read through `Last close`, evaluating to `0` or `null` afterward. Add `default_formulas.forecast: "sum(<column>)"` to read post-close source rows (any other `[]` expression runs on all periods).
- **Preferred time column (`preferred_time_property`):** Sets the governing date for calendar placement and `first`/`last` ordering; configure via `edit_table_blocks` (`sources`).
- **Ledgers vs. restated snapshots:** Sum transaction ledgers. For restated syncs (one row per entity per run), inspect preview first (identical rows across stamps repeat current state; differing rows preserve history) and filter by snapshot month (`ObservedAt.Month = this.Date.Month` or `date(2025, 11, 1)`).
- **Rosters vs. monthly staffing:** A timeless roster (`Employee ID` rows, no `Date`) shows `last(...)` current state. Monthly staffing (`Date` columns) binds each month's observation: the child rule (`[`Employee ID` in any]`) carries forward each employee's state, and the parent (`[]`) sums active headcount. For changing departments, avoid `$` in Actuals or read a dated `Department in Force` variable:

```formula
sum(Headcount$[`Employee ID` where `Department in Force`$[`Employee ID` = `Employee ID`, Date.Month = this.Date.Month] = this.Department, Date.Month = this.Date.Month])
```

- **Sum of products and sparse nesting:** Write deal-level multiplication (`child_defaults` on `Total[Stage][Owner]`) so parents sum weighted deals instead of multiplying summed amounts by summed probabilities:

```formula
if(startofmonth(`Expected close`) = Date.Month, Amount * Probability, 0)
```

Nested source dimensions display only combinations present in source rows (missing values group under `empty:None`).

## Dimensions and mappings

Map source keys to reporting groups (`Account -> Bucket`, multi-source departments -> `Department`) with a `catch_all` for unmatched items. Read mappings two ways:

1. Over an imported variable (source rows carry the key):

```formula
sum(Amount[Bucket = "COGS"])
```

2. Over an authored variable (no `Bucket` in coordinates): filter via key hop:

```formula
sum(Amount$[Account.Bucket = this.Bucket, Date.Month = this.Date.Month])
```

Hops chain (`this.Project.SourceTeam.Team`). Drop outer dimensions deliberately when reading scoped variables (``sum(`R&D Expense`$[Account in any, Date.Month = this.Date.Month])``).

- **Unsplit mapped classes:** When a table groups by a mapped dimension (such as `Class`) and a read or receipt reports formulas saved by another group from the same source key, those groups do not split across classes automatically.
  - Fetch omitted groups via `next`, add or update a text block right below the table (`edit_pages` `add_text` with `after_block`) naming the class source and unsplit groups in the same turn, ask once how each splits, and keep saved amounts until answered.

## Tables, pages, and scenarios

- **Table layout:** Put variables on rows and `Date` on columns for time series, or record keys on rows and fields on columns for record tables (filter large tables).
- **Field count:** Show requested fields. When none are named, show all fields up to 12; for wider sources, show up to 12 identifying or actionable fields (key, date, counterparty, amount, status, owner) and put remaining audit fields on a second keyed table.
- **Windows and formatting:** Set `window` explicitly when a report names dates, and apply `format_updates` at the broadest scope.
- **Page placement:**

  - **Deliverables** (forecast, dashboard, tracker, report, schedule, interactive tie-out): Use its own page (create or reuse) with tables beside visuals; extend existing report tables for new periods unless asked for a separate report.
  - **Supporting checks and evidence:** Reuse a workspace-visible checks page in `Main` (never a deliverable, report, or private page), or create and reuse one top-level **Ari's working papers** in `Main` (`audience: "workspace"` if enabled). Pass `Main` on every working-papers call, identify each check's source scenario, and preserve other blocks and audiences.
  - **One-off sketches:** Use `[[sketch-answers]]` (**Ari's scratchpad**, private) only for temporary visual sketches, never durable team checks. Create an explicitly requested separate page as asked.

- **Scenarios:**
  - **Choices vs. scenarios:** Model coexisting options as switches or dated inputs within a plan; use scenarios for mutually exclusive side-by-side alternatives and dimensions for coexisting segments. Answer what-if questions in a temporary scenario unless asked to keep it.
  - **Target scenario:** Put new pages, tables, baselines, reports, and models in `workspace_default` (`Main`) unless the user names a scenario or edits the screen or chat scenario's assumptions (keep planned hires, spend, funding, and supporting tables in that scenario). Name the chosen scenario explicitly on every write.
  - **Inheritance and corrections:** Live scenarios inherit unchanged work from their base (imported records are shared read-only); work in another scenario is absent from `Main`. Fix factual errors in the scenario owning the original (if another scenario holds its own version, name it and offer to fix it too).
  - **Target disclosure:** Disclose in opening and closing replies when the target differs from the screen or chat scenario, and explain how `workspace_default` input changes ripple to derived scenarios (except where a scenario keeps its own version).
  - **Assumptions and forecast-only edits:** Write user-given assumption values to the existing input while keeping downstream formulas (report resulting values and any moved closed periods). For forecast-only moves, override dated `Forecast` cells or Forecast-only inputs, because timeless inputs read by all periods move history too.
  - **Cleanup before handoff:** Keep requested and full-build scenarios; delete temporary or scratch check scenarios (`edit_scenarios` `delete`) before ending the turn, and restore any temporary test inputs in shared scenarios.
- **Performance:** Prefer monthly or yearly grain over daily when results match. Write general `Date` and `this.Dim` formulas rather than per-period copies, extract repeated expensive sub-expressions into helper variables, and use a saved query for warehouse rows (or `edit_integrations` `ingest_csv` for standalone CSVs) so one aggregate replaces hundreds of cell rules.

## Checking sources and disclosures

- Check workbook tabs and connected sources before asking the user; ask only for unanswered business questions.
- **Contradictions and earlier figures:** When a finding contradicts a saved value, name the saved value, table, variable, segments, period, and corrected value. When a new figure measures the same thing as an earlier figure in chat (including before a restart or rebuild), name the earlier figure and split the difference into data or input changes versus method changes.
- **QuickBooks:** QuickBooks internal IDs (`TRANSACTION_ID`, `ID`) do not appear in QuickBooks forms or reports; list each referenced transaction with type, date, payee, amount, account, memo, and document number. P&L and Balance Sheet follow `accounting_basis` from `inspect_integrations` (cash basis omits unpaid bills, and basis changes rewrite synced history).
- **Verification:** Check `status: "applied"` readbacks across start, mid, and end periods on an item that changes mid-window (hire, termination, rate change, FX) plus coarse rollups, and sanity-check record cells against source facts (such as salaried month = annual / 12).

## Modeling the business

- **Full build** (new model or full rebuild): Deliver connected revenue, costs, staffing, and cash (default 12 months history plus 24 months forecast), Base/Upside/Downside scenarios, one shared controls page, full dimensional depth where economics or timing differ (grounded in research and benchmarks), keyed records or placeholders (`Software Engineer 2`), a Canvas simulator, and a `present_guide` walkthrough.
- **Partial build** (a requested piece): Deliver only the requested piece at requested size on its own or existing pages. Add separate scenarios, pages, simulators, or walkthroughs only when asked, and offer a broader build in chat.
- **Discrete events:** Model hires, cohorts, capex projects, contract starts, and financings as dated records or items so shifting a date moves downstream payroll, capex, revenue, and cash automatically (never flat smooth curves):

```formula
if(`Build Date` <> null and startofmonth(`Build Date`) = Date.Month, `Project Capex`, 0)
```

- **Revenue:** Match business mechanics: SaaS tracks opening, additions, churn, and ending customers × ARPU; transaction or contract models track unit economics and recognition. Keep bookings, recognized revenue, billings, and cash distinct.
- **Pipeline:** Split deals into **Bookings** (won, close month), **Open / Weighted pipeline** (open, expected close month, `Amount * Probability` at deal leaf), and **Lost** (when requested); disclose ownerless deals and stage mappings.
- **Costs and headcount:** Separate Cost of Revenue from OpEx (R&D, S&M, G&A) driven by operational units (users, volume, compute, headcount). Deliver both a current roster and a monthly staffing table keyed by `Employee ID` or cohort, driving `Headcount` (`1` or cohort count while active, else `0`) and prorated monthly pay from start and end dates:

```formula
if(`Start Date` <> null and startofmonth(`Start Date`) <= Date.Month and (`End Date` = null or startofmonth(`End Date`) >= Date.Month), Headcount * (`Annual Pay` / 12) * (datedif(if(`Start Date` > Date.Month, `Start Date`, Date.Month), if(`End Date` <> null and `End Date` < eomonth(Date.Month, 0), `End Date`, eomonth(Date.Month, 0)), "D") + 1) / daysin(Date.Month, "month"), 0)
```

- **Pay estimates and duplicate hires:**
  - Estimate pay only via role-level rates (person-level only from sourced or user facts; disclose estimates), include taxes and benefits, and keep contractors, unpaid founders (`0`), uncategorized departments, and currencies explicit.
  - Before reporting payroll, flag any planned hire matching a roster or payroll record by name or by role and start month. Open the reply with that pair and each record's monthly pay, state totals after it, and keep both records until the user decides which to keep.
- **Statements and cash:**
  - **Cash:** Opening cash plus dated operating, working-capital, capex, debt, and financing flows. Use live accounting open balances over lagging CRM for overdue collections, and state the exact cash constraint date (or that cash stays positive across the horizon).
  - **P&L:** Map ledger accounts to `Bucket` (`edit_dimension_mappings` or inline `dimensions`), keeping `Unclassified` visible; write `Gross Profit` and `Operating Profit` over statement variables and nest `Account` beneath.
  - **Balance Sheet:** Use `rollup: "LAST"` across `Total assets`, `Total liabilities`, and `Total equity` (include provider `Net Income` and `Retained Earnings`, exclude `Section` subtotals). Verify `Assets = Liabilities + Equity` every period and report any residual's amount, period, and accounts.
  - **Cash Flow:** Reconcile to Balance Sheet cash change. Indirect: start from Net Income, include each working-capital, gross capex (never double-count depreciation), debt, and equity movement once, excluding cash and Retained Earnings net-income rollups. Direct: sum classified bank movements (such as `Xero Cash Flow` installed via `edit_integrations` standard queries once journals sync).
  - **Account-level forecasts:** Forecast on account lines, preserving generated parent total sums (ask before replacing a total if account lines are blank).
  - **Run-rate check:** Before reporting forecasts, compare every variable against its last 3 actual months on the same definition (booked cost lines for the first forecast quarter, collections, debtor and creditor days, and targeted KPIs versus targets). Quantify every step change or gap with its months and source (user instruction, recorded change, or none found).
- **BvA:** Align plan and actuals over the latest complete shared period using existing variable mappings (`Variance = Actual - Plan`; `Variance % = Variance / abs(Plan)`, omitted if Plan is `0`).
- **Checks and delivery summary:** Distinguish internal consistency from external source tie-outs (balance and cash checks prove arithmetic only; forecast completeness requires every account line forecast and the run-rate comparison). Link the page and state the method, key assumptions, commitments, coverage gaps, and additive versus restated source rows.

For the complete parser grammar, reference syntax, operators, and callable function inventory (including where the engine differs from Excel), see `[[formula-reference]]`.
