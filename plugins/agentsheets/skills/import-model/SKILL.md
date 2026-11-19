---
name: import-model
description: Rebuilding someone else's model — a workbook, an export, a rendered document — as a native cfo.ai model that keeps its meaning. Use when asked to import, migrate, rebuild, or recreate a model from source files.
---

# Import model

An import rebuilds an external spreadsheet, export, or document as a native, editable cfo.ai model of variables, dimensions, scenarios, formulas, and saved tables (`[[build-model]]`, `[[build-visualizations]]`, `[[connect-data-sources]]`).

- **Preserve business logic over cell coordinates**: Translate spreadsheet formulas into named variables and dimensions using the canonical formula reference in `[[build-model]]`.
- **Default scope**: Unless the user narrows the scope, rebuild every business-relevant visible section, period, scenario, statement, and view while leaving the source files unchanged.
- **Workbooks over connected sources**: When source tabs export a system already connected in cfo.ai, read the data from the live connection (`[[connect-data-sources]]`) and rebuild the workbook's formulas, mappings, and report views over it. Keep export-vs-live comparisons in chat and reconciliation checks unless requested on a page.

## Workbooks Shared in Conversation

- **Question first**: When the person asks about the workbook, answer from the file before rebuilding. Say in the reply that the rebuild started or finished.
- **Lines already in the model**: When the workbook's lines already exist in the model, as after an earlier import or in a workbook exported from this workspace, update those variables instead of adding copies (see "Updating from a newer workbook version").
- **Comparison tables**: Tables that compare results with the workbook read the rebuilt variables, not typed copies of its results.
- **Recorded values**: A file of recorded values, such as a bank export, aging report, timesheet, or statement whose only formulas are subtotals, is source data for `[[connect-data-sources]]`, not a model to rebuild.

## Which sheets a workbook presents

A workbook presents its visible sheets. A rebuild of the workbook covers the visible sheets and the hidden cells they read, and the import's source map names every other hidden sheet as hidden. A request that names a hidden sheet brings it into scope.

## Survey, Source Map, and Navigation

- **Stage claims by evidence**: Before reading the file, describe only your import method. After surveying the file, report its structure and scope. Claim numeric parity only for sections verified by reconciliation.
- **Source map**: Record each sheet and cell range, its purpose, its periods, its attachment ID (`runway-file-…`), and its destination (rebuilt page/variables, ingested CSV source table, or existing connection/query). List sheets in build order: a sheet comes after the sheets it reads.
- **Page and table mapping**:
  - Each in-scope worksheet of assumptions, calculations, or reports becomes a page of the same name, and each labeled section across the full used range becomes a table block.
  - Collapse multiple sheets that repeat an identical layout per item (such as per product or region) into a single page over a shared dimension.

## Separating Data, Calculations, and Scenarios

| Source pattern                                                       | How to rebuild natively in cfo.ai                                                                                                                                                                                                                                   |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Transaction rows & leaf exports (`>200` fixed values)**            | Prepare a CSV at the records' true grain (keeping distinct measures in separate columns and naming source date columns with a business name rather than `Date`), load via `edit_integrations` `ingest_csv`, and map via a saved query (`[[connect-data-sources]]`). |
| **Tunable assumptions**                                              | Create named editable input variables with typed opening/literal cells. Preserve exact stored inputs and dated changes (including zeros).                                                                                                                           |
| **Repeated row groups, per-item tabs, or column bands**              | Model as one variable broken down by a business dimension rather than creating per-item variables.                                                                                                                                                                  |
| **Account/category lookup tables**                                   | Model as dimensional mappings (`[[build-model]]`) rather than hardcoded formula filters.                                                                                                                                                                            |
| **Case selectors, duplicate case tabs, or `FY26 Budget` columns**    | Model as separate scenarios sharing the same variables and `Date` axis (never as an additive dimension). Inspect existing scenarios first.                                                                                                                          |
| **Statement totals, derived subtotals & copied calculation outputs** | Treat as reconciliation targets that native formulas must reproduce—never ingest them as raw data rows or use copied outputs as model inputs. (Preserve parent rows that carry direct postings.)                                                                    |

## Time Boundaries and Formula Translation

- **Source close vs. Last close**: Reconcile against the source's closed period. If the scenario's Last close differs, you may move Last close to match (explaining the change), but never extend `Actuals` across forecast periods just to force parity.
- **Operational vs. annual dates**: Keep operational observations beyond the accounting close distinct from projections, and do not spread annual observations into months without an explicit allocation rule.
- **Translating formulas**:
  - Preserve operands, constants, date windows, and calculation methods while replacing cell coordinates with named variables, dimensions, opening recurrence seeds, and `LAST` stock rollups (`[[build-model]]`).
  - Never silently substitute a different forecasting method. If a source lacks formulas, build defensible native logic with editable inputs and state that the source held values only.
- **Range aggregates**: For a range mapped to one variable, use one aggregate over the matching dimension items and relative or fixed date window. Use a set filter for selected items; whole-dimension or category totals retain their native rollup or mapping filter. Code that translates formulas in bulk groups each range's cells by variable and writes that aggregate.

For a monthly `SUM(C7:N8)` at K11, where the rows select cost lines A and B:

```formula
sum(`Cost`$[`Cost line` in {"A", "B"}, Date.Month in [-8]:[3]])
```

## Building, Tying Out, and Resuming Sheet by Sheet

A first import follows these steps. Its summary page in `Main` holds its plan and record; its saved text names the source file's `runway-file-…` ID.

1. **Plan**: After the survey, save the source map on the summary page and show it in chat before building.
2. **One sheet at a time**: Build sheets in source-map order. Creating tables, pages, and variables waits on one workspace-wide lock, so parallel sheet writes can time out.
3. **Section-by-section reconciliation**: Verify every section of the sheet (including later rows and columns, subtotals, formula boundaries, actuals, forecasts, and scenarios) against the file. If neighboring formulas expose a clear formula bug in the source, report both the source figure and the corrected calculation.
4. **Sheet tie-out**: As the sheet's last write, add all its rows to the `Workbook tie-out` table on the summary page in one table write.
5. **Finished sheets**: A sheet with rows in `Workbook tie-out` is finished; one without is unfinished, even when its page holds tables. Sheets the source map excludes, including a sheet with no cells to check, are not built.
6. **Handoff**: State what was imported and rebuilt, provisional assumptions, corrected source defects, and unmapped items. The final reply lists each sheet with its tie-out: cells checked, and each difference with its workbook value, cfo.ai value, and cause.

### The `Workbook tie-out` table

A timeless table with one row per checked cell, created by the first sheet that ties out. Its values recalculate with the model.

| Row or column             | Holds                                                                      |
| ------------------------- | -------------------------------------------------------------------------- |
| `Workbook cell` dimension | One item per checked cell, named by sheet and cell, such as `P&L!N24`.     |
| `Workbook value`          | The cell's value in the file, typed.                                       |
| `cfo.ai value`            | A formula that reads the rebuilt variable at that cell's period and items. |
| `Tie-out difference`      | `cfo.ai value` minus `Workbook value`.                                     |

Check each rebuilt table's total row, or its last row, at the sheet's first and last periods (once on a sheet without periods). A version update retypes `Workbook value` from the new file.

### Resuming a stopped import

An import stops when its turn ends early, a task step fails, or the person stops it. To resume it in any chat:

1. Read the summary page's source map and `Workbook tie-out`.
2. Rebuild nothing on finished sheets' pages.
3. Continue the first unfinished sheet in source-map order, from the tables its page already holds. A stopped background task resumes with `edit_tasks` `retry`.

### Running as a background task

When the person chose a background build:

1. Complete the survey and source map in this conversation. Put them, the tie-out table, and the finished-sheet rule in the task brief.
2. **Foundation step (`verify: false`)**: Create the `Date` axis, Last close, dimensions, mappings, scenarios, and the summary page with the source map (no sheet tables).
3. **Sheet steps (`verify: false`)**: One step per sheet, each with the step before it in `depends_on`, so sheets build in order; neighboring sheets share a step when a task cannot hold one per sheet. A sheet needing numbers from a sheet not yet built loads the workbook's values as temporary inputs and notes them.
4. **Cross-sheet step (`verify: true`)**: Replace temporary inputs with live variable references, verify cross-sheet links, statement identities, cash recurrence, and scenario behavior, and report each sheet's tie-out.

Only the last step takes a review: a review that sends an earlier step back over values alone would start the steps after it while the repair runs.

## Updating from a newer workbook version

A version update compares each section with the prior attachment in the source map. Added and removed sections, changed tables or formulas, and changed CSV source tables are reflected in the model. Edits in unchanged sections and rules the person deliberately chose over the file stay in place, including in changed sections.

The saved summary keeps the rules the person stated, and its source map names the new attachment ID. Update each source-map entry whose recorded facts changed, and keep the other entries as they were.

- **Checks**: In the update report, list each sheet's checked cells, chosen as for a tie-out, with the workbook value, cfo.ai value, and the cause of any difference: a preserved choice or an unexplained difference. The update adds no check table and no check prose to any page. New check rows go only into the summary page's existing `Workbook tie-out` table.
- **Interrupted update**: Resume by comparing the saved model with the new workbook again; a section that already matches needs no write.
