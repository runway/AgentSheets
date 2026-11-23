---
name: import-model
description: Use when rebuilding an external spreadsheet, export, or document as a native cfo.ai model.
---

# Import model

Rebuild an external spreadsheet, export, or document as native cfo.ai variables, dimensions, scenarios, formulas, and tables (`[[build-model]]`, `[[build-visualizations]]`, `[[connect-data-sources]]`).

- **Default scope:** Translate cell formulas into named variables/dimensions covering every business-relevant visible section, period, scenario, statement, and view without modifying source files unless narrowed.
- **Over connected sources:** When tabs export a system already connected in cfo.ai, read from the live connection (`[[connect-data-sources]]`) and rebuild logic over it. Keep export-vs-live comparisons in chat unless requested on a page.
- **Shared in chat:** If asked a question about the workbook, answer from the file before rebuilding and note whether a rebuild started/finished. Update existing variables from a prior import/export instead of duplicating (`## Updating from a Newer Workbook Version`); comparison tables must read rebuilt variables, not typed copies. Raw exports with only subtotals (bank exports, aging, timesheets) are source data (`[[connect-data-sources]]`), not models.

## Which sheets a workbook presents

A workbook presents its visible sheets. A rebuild of the workbook covers the visible sheets and the hidden cells they read, and the import's source map names every other hidden sheet as hidden. A request that names a hidden sheet brings it into scope.

## Survey, Source Map, and Mapping Patterns

- **Claim boundaries & source map:** Describe planned method before reading; report structure/scope after survey; claim parity only for reconciled sections. Record each sheet, range, purpose, periods, attachment ID (`runway-file-...`), and destination (page/variables, CSV, or connection/query) in dependency order. Each in-scope tab becomes a page of the same name and each labeled section becomes a table (except raw transaction tabs and repeated per-item tabs below).
- **Raw rows (`>200` fixed values):** Clean CSV at true grain (distinct quantities in separate columns; business date column, not `Date`), load via `edit_integrations` (`ingest_csv`), map via saved query (`[[connect-data-sources]]`).
- **Structure patterns:** Editable inputs -> typed cells (preserve inputs, dated changes, zeros). Repeated row groups/tabs/column bands -> one variable broken down by a shared dimension on one page. Account/category lookups -> dimension mappings (`[[build-model]]`), not formula filters. Case selectors, case tabs, or `FY26 Budget` columns -> scenarios sharing variables and `Date` (never an additive dimension; inspect existing scenarios first). Subtotals/totals -> native formulas as tie-out targets, never ingested raw inputs (keep parent rows with direct postings).
- **Time boundaries and formulas:** Reconcile against source close (may move Last close and disclose; never stretch `Actuals` into forecast, keep post-close data separate, never spread annuals without a rule). Preserve operands, constants, windows, and methods; if source holds values only, build editable logic and disclose. Translate ranges over one variable into one aggregate over items + relative/fixed window (e.g. `SUM(C7:N8)` at current month `K11` for cost lines `A`/`B`):

```formula
sum(`Cost`$[`Cost line` in {"A", "B"}, Date.Month in [-8]:[3]])
```

## Build, Tie Out, and Resume Sheet by Sheet

Keep the import plan and record on a `Main` summary page naming source `runway-file-...`:

1. **Plan:** Save the source map on the summary page and show it in chat before building.
2. **Sequential sheets:** Build one sheet at a time in source-map order (schema writes share a lock; parallel writes can time out).
3. **Reconcile:** Check later rows/columns, subtotals, boundaries, actuals, forecasts, scenarios. If neighboring formulas expose a workbook bug, report both source and corrected values.
4. **Tie out (`Workbook tie-out`):** As each sheet's last write, append its tie-out cells to timeless table `Workbook tie-out` on the summary page in one write:
   - Tie-out cells: each table's total/last row at first and last period, or once if timeless. Step 3's checks add no rows. When the person asks to check every cell, load the file's values as raw rows (above) that only the check reads.
   - `Workbook cell` dimension (one item per cell, e.g. `P&L!N24`).
   - `Workbook value` (typed; retyped on update), `cfo.ai value` (live formula reading rebuilt variable), `Tie-out difference` (`cfo.ai value - Workbook value`).
5. **Completion & handoff:** A sheet is finished only after its `Workbook value` entries are retyped from the current file; one without current-file tie-out rows is unfinished even if tables exist (skip excluded/check-free sheets). Report built scope, assumptions, corrected bugs, unmapped items, and per-sheet tie-out (cells checked + any difference and cause).

### Resuming or Running in Background

- **Resume in any chat:** Read source map and `Workbook tie-out`, skip finished sheets, resume first unfinished sheet from existing tables (`edit_tasks` `retry` for a stopped task).
- **Background build (`edit_tasks`):** Survey and map in chat first, then put map, tie-out spec, and finished-sheet rule in `brief`: (1) **Foundation (`verify: false`)**: `Date`, Last close, dimensions, mappings, scenarios, summary page, source map; (2) **Sheet steps (`verify: false`)**: 1 step/sheet chained via `depends_on` (combine neighbors if over cap; use noted temporary inputs for forward refs); (3) **Cross-sheet (`verify: true`)**: replace temp inputs with live formulas, verify cross-sheet/statement/cash/scenario tie-outs, report per-sheet results (only final step verifies so earlier steps don't re-run concurrently).

## Updating from a Newer Workbook Version

1. Diff each section against the prior attachment in the source map. Apply added/removed sections, changed tables/formulas, and changed CSVs while preserving edits in unchanged sections and explicit user rules even inside changed sections.
2. Update changed source-map entries and new attachment ID while keeping unchanged entries and user rules. Append new checks only to `Workbook tie-out` (no extra check tables or prose on pages).
3. Report checked cells per sheet (workbook value, cfo.ai value, preserved choice vs. unexplained). Resume interrupted updates by re-comparing and skipping already-matching sections.
