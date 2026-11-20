---
name: import-model
description: Use when importing, migrating, or rebuilding an external spreadsheet, export, or document as a native cfo.ai model.
---

# Import model

Rebuild an external spreadsheet, export, or document as native cfo.ai variables, dimensions, scenarios, formulas, and tables (`[[build-model]]`, `[[build-visualizations]]`, `[[connect-data-sources]]`).

- **Default scope:** Translate cell formulas into named variables and dimensions (`[[build-model]]`) covering every business-relevant visible section, period, scenario, statement, and view without modifying source files unless narrowed.
- **Over connected sources:** When tabs export a system already connected in cfo.ai, read from the live connection (`[[connect-data-sources]]`) and rebuild the workbook's logic over it. Keep export-versus-live comparisons in chat unless requested on a page.

## Workbooks Shared in Conversation

- **Question first:** If asked a question about the workbook, answer from the file before rebuilding, and note whether a rebuild started or finished.
- **Existing lines:** Update existing variables from a prior import or cfo.ai export instead of duplicating them (`## Updating from a Newer Workbook Version`). Comparison tables must read rebuilt variables rather than typed copies.
- **Recorded values:** Raw exports whose only formulas are subtotals (bank exports, aging, timesheets) are source data (`[[connect-data-sources]]`), not models to rebuild.

## Which sheets a workbook presents

A workbook presents its visible sheets. A rebuild of the workbook covers the visible sheets and the hidden cells they read, and the import's source map names every other hidden sheet as hidden. A request that names a hidden sheet brings it into scope.

## Survey, Source Map, and Structure

- **Claim boundaries:** Before reading, describe only your planned method; after survey, report structure and scope; claim parity only for reconciled sections.
- **Source map:** Record each sheet, cell range, purpose, periods, attachment ID (`runway-file-...`), and destination (rebuilt page/variables, ingested CSV, or existing connection/query) in dependency order.
- **Pages and tables:** Each in-scope assumption, calculation, and report sheet becomes a page of the same name, and each labeled section becomes a table block (except raw transaction tabs and repeated per-item tabs mapped below).

## Mapping Patterns to cfo.ai

- **Transaction rows or leaf exports (`>200` fixed values):** Clean CSV at true grain (distinct quantities in separate columns; business date column name, not `Date`), loaded via `edit_integrations` (`ingest_csv`) and mapped by a saved query (`[[connect-data-sources]]`).
- **Tunable assumptions:** Editable input variables with typed cells (preserve exact inputs, dated changes, and zeros).
- **Repeated row groups, tabs, or column bands:** One variable broken down by a shared dimension on one page, never per-item variables.
- **Account or category lookups:** Dimension mappings (`[[build-model]]`), not hardcoded formula filters.
- **Case selectors, duplicate case tabs, or `FY26 Budget` columns:** Separate scenarios sharing variables and `Date` (never an additive dimension); inspect existing scenarios first.
- **Statement totals, subtotals, or copied outputs:** Reconciliation targets reproduced by native formulas; never ingest as raw inputs (keep parent rows carrying direct postings).

## Time Boundaries and Formulas

- **Last close:** Reconcile against the source's close. You may move Last close to match (disclose the change), but never stretch `Actuals` into forecast periods to force parity. Keep post-close operational data distinct from projections, and do not spread annual values into months without an explicit rule.
- **Formula translation:** Preserve operands, constants, date windows, and methods (`[[build-model]]`). Never silently change forecasting methods; if the source holds values only, build editable native logic and disclose it.
- **Range aggregates:** Translate a range mapped to one variable into one aggregate over matching items and a relative or fixed date window (filter selected items; use native rollups or mappings for whole-dimension totals). Group bulk range cells by variable (for example, `SUM(C7:N8)` at current month `K11` for cost lines `A` and `B`):

```formula
sum(`Cost`$[`Cost line` in {"A", "B"}, Date.Month in [-8]:[3]])
```

## Build, Tie Out, and Resume Sheet by Sheet

Keep the import plan and record on a `Main` summary page naming the source `runway-file-...` ID:

1. **Plan:** Save the source map on the summary page and show it in chat before building.
2. **Sequential sheets:** Build one sheet at a time in source-map order (schema writes share a workspace lock, so parallel writes can time out).
3. **Reconcile every section:** Check later rows/columns, subtotals, boundaries, actuals, forecasts, and scenarios. If neighboring formulas expose a workbook bug, report both source and corrected values.
4. **Tie out:** As each sheet's last write, append all checked cells to `Workbook tie-out` on the summary page in one write.
5. **Completion rule:** A sheet with rows in `Workbook tie-out` is finished; one without is unfinished even if tables exist. Excluded or check-free sheets are skipped.
6. **Handoff:** Report built scope, assumptions, corrected workbook bugs, unmapped items, and per-sheet tie-out (cells checked and any difference with workbook value, cfo.ai value, and cause).

### `Workbook tie-out` Table

Timeless table created by the first finished sheet (recalculates live):

- **`Workbook cell` dimension:** One item per checked cell, such as `P&L!N24` (check each table's total or last row at the first and last period, or once if timeless).
- **`Workbook value`:** Typed source cell value (retyped on version update).
- **`cfo.ai value`:** Formula reading the rebuilt variable at that cell's period and items.
- **`Tie-out difference`:** `cfo.ai value` minus `Workbook value`.

### Resuming or Running in Background

- **Resume in any chat:** Read the summary page's source map and `Workbook tie-out`, skip finished sheets, and resume the first unfinished sheet from its existing tables (`edit_tasks` `retry` for a stopped background task).
- **Background build (`edit_tasks`):** Survey and map in chat first, then put the map, tie-out spec, and finished-sheet rule in `brief`:
  1. **Foundation (`verify: false`):** `Date`, Last close, dimensions, mappings, scenarios, summary page, and source map.
  2. **Sheet steps (`verify: false`):** One step per sheet chained via `depends_on` (combine neighbors if over the step cap); use noted temporary inputs for forward cross-sheet references.
  3. **Cross-sheet (`verify: true`):** Replace temporary inputs with live formulas, verify cross-sheet, statement, cash, and scenario tie-outs, and report per-sheet results. Only the final step takes review so earlier steps do not re-run concurrently with dependents.

## Updating from a Newer Workbook Version

1. **Compare and apply:** Diff each section against the prior attachment in the source map. Apply added/removed sections, changed tables/formulas, and changed CSV sources while preserving edits in unchanged sections and explicit user rules even inside changed sections.
2. **Update the record:** Update changed source-map entries and the new attachment ID while keeping unchanged entries and user rules. Append new checks only to `Workbook tie-out` (never add extra check tables or prose to pages).
3. **Report and resume:** Report checked cells per sheet (workbook value, cfo.ai value, and whether any difference is a preserved choice or unexplained). Resume interrupted updates by re-comparing against the new workbook and skipping already-matching sections.
