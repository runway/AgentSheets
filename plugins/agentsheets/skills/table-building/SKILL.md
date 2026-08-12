---
name: table-building
description: Operate pages and saved table blocks for financial reports. Use
  when creating, validating, inspecting, or updating a report table.
---

# Table building

<!-- standards:start -->

Apply [[presentation-and-voice]]. Persist a table only when the user wants an artifact; use
headless inspection for analysis and preserve the current config on whole-config updates.

<!-- standards:end -->

## Creating table blocks

Use this skill when the user asks for a saved table. Use `edit_model_views` with
`change.configure_table`: it writes to an existing page, and creates the page when `page` names
one that does not exist yet. A markdown table, preview, or description is not a saved result.
Skip the write only when the model has no usable variable for the request.

**Propose first for non-trivial builds.** When this table is part of a larger build or modeling request — building revenue, a model, a forecast, or a report, or anything that takes several steps to work out — follow the plan-approval gate in your core instructions: propose the plan and get the user's approval (their "Build it") before you commit the block. Build directly only for a simple, explicitly requested single table from an existing variable.

The scenario and page in view are the defaults — do not ask the user for them. Pass `scenario` by name to target another scenario, and `page` on an entry to target a different page by id or name — a page that does not exist yet is created, so a new page and its tables land in one call.

A saved block can be shown against other scenarios or against an earlier period — both are `edit_table_blocks` intents, not part of what the table computes; the Time-period comparisons section owns the one-kind-at-a-time rule.

## Where a split belongs: rows or columns

The report layout is fixed: variables band the rows, a dimensional split nests as child
rows under its variable, and time runs across the columns. Author it — a shared `breakdown`
of `"[Date.Month]"` (the system Date), and a create `window` naming a real range and grain —
rather than trusting a default to supply it. A long row does not disqualify the trajectory
read; time leaves the columns only on an explicit layout ask. A split other than time earns the
column axis only on width — small, closed, worth comparing within each period (scenarios, actuals
vs budget, a few named regions). Rows scroll, nest, and drill, so open-ended and hierarchical
splits belong there; crossing one with time on the columns costs the trajectory read too.

A dimensional split goes on the entry's own `breakdown`, as child rows nested under that variable:
``{"variable": "Cost of Events", "breakdown": "[`Event new`, `GL account name`]"}``.
Exactly three shapes are deliberately timeless, and only these take a `NO_GRANULARITY` window: a
mapping table (a dimension entry on the value axis, its keys on the shared breakdown), a database
view listing an entity's items with their attributes at one period (a roster), and an assumptions
list the user asked for as such. [[dimensional-modeling]]'s dimension-mappings reference and its
recipes own the first two shapes. A table whose cells aggregate amounts is a report, demo and
dummy data included: keep `[Date.Month]` and show the flat row.

## When not to use

You should only create a block on a page or update one if it is clear that the user wants to create a new artifact or modify one. If the user or you are simply exploring the data, prefer `inspect_model_views` with `ask.calculate` for table-shaped reads (pass `compare_scenarios` for scenario comparisons), or `inspect_variables` `ask.try_formulas` when the exploration needs formulas that are not saved variables — it takes named formulas with expressions, a required `from`/`to` range, and `by` dimensions, and returns records. The probe shape is spelled out in [[dimensional-modeling:references/02-formulas.md]] — read that file directly; you do not need to load the manual first.
For time-period questions like "how did revenue change vs last quarter?", answer headlessly with `inspect_model_views` (`ask.calculate`) and its `time_comparison` parameter instead of persisting a block — the grid header carries a comparison legend: scenario names, or `time(-N)`. The same tool's other asks: `ask.rank` returns a top-N shortlist from a declared layout (max 50) when the question is which rows lead, and `ask.formatting` is the read half of `change.formatting` — it answers in the same words the write takes, and also surfaces styles stored against rows or columns that no longer exist.

## Workflow

1. Call `inspect_variables` and `inspect_dimensions` to see what variables and dimensions the workspace has. Inspect `source`, `type`, and the `isSystemDate` flag before choosing.
2. If the request names an item from an ingested business dimension, call
   `inspect_dimensions`. Use its item URI in `dimensionItemInclusionFilter`.
3. If the user asks to compare against another scenario, pass the scenario by name. Use `inspect_scenarios` when you are unsure what exists, and ask for clarification when a name matches more than one scenario.
4. Finish the table definition, then call `edit_model_views` once with the complete `view` or
   `table_config`. Do not call it early with only a page name or progress message. The writes
   validate themselves, a view that does not check returns its problems, and nothing persists on
   failure. Reach for `dry_run` only when a whole-config `table_config` replace of an existing
   table is risky enough to preflight. Omit unknown optional IDs.

### Judge the write's readback

The write's own `readback` is the verification read for the saved table: judge the values it
returns, and treat zeros, blanks, or an empty result as a failed check to diagnose rather than a
table confirmed. On a dated table, diagnose an empty readback at the window first:
`NO_GRANULARITY` mints no calendar periods, so only dates the data itself carries can appear (a
missing range just falls back to the default span); the repair is a real window, never stripping
the Date axis or reshaping until values appear. If the user explicitly asks for a post-save
readback, inspect the returned block after the write.

## Referencing an existing table block

When the user asks to **show**, **display**, **embed**, or **pull up** a table block they already have (rather than create a new one), do not create a copy and do not promise an inline chat preview. Resolve the existing block and answer with its name/location; use `inspect_table_blocks` only when you need to reason about the answer or a follow-up edit.

1. Resolve a fuzzy or partial page/block name before using it.
2. To reason about a table, call `inspect_table_blocks` `{"ask": {"list": {"tables": ["<name>"]}}}`:
   the view plus window, comparison and sort in the words `edit_table_blocks` takes. Ask
   `config` only for a whole-config `table_config` replace, which needs the block's own node ids.
   Both nest under `ask`; the only top-level fields are `scenario` and `ask`.
3. Say one sentence naming the table you found and the page it belongs to when that context is available.

If you cannot find a block matching the user's description in `inspect_pages`, tell the user what tables are available rather than guessing or creating a new one from scratch.

## Updating existing table blocks

When the user asks to change how an existing block is shown — a scenario comparison, its window, widths, visibility, sort order (`change.sort`), a transpose (`change.transpose`), colors and text styles (`change.formatting`), or title — change that block with `edit_table_blocks`; do not create another one and do not restate the table. Formatting combines with rename, columns, visibility, and sort only: apply a window, comparison, or transpose change in its own call first, or the combined call is refused. `change.configure_table` is only for changing which variables and dimensions the table is built from. The tool's schema carries the mechanics, and so does [[dimensional-modeling:references/12-editing-blocks.md]] — read it before reshaping a block you did not build.

If the user asks for a scenario comparison but does not say which scenario(s) to compare against, ask which scenario to use before changing the table. If the requested scenario name matches multiple scenarios, ask a concise clarification and list only the matching scenario names.

Layer IDs are internal implementation details. If a tool result echoes one, do not show it to the user; refer to the scenario by name instead. Say "I found Budget and updated the table to compare against it," not "I used layer_id abc123."

## Arranging a page

`inspect_pages` lists blocks top to bottom as they appear, so "below Revenue" is
the next one listed. Name a block by id or by the name shown on the page.

**Never delete and recreate a block to move it** — that loses its config, its id, and any reference
to it. Use `edit_pages` with a `change.reorder_blocks` block: `place: {block, after_block}` moves
one within its page (`after_block: null` = top), and `block_order` restates the whole page. To send
a block elsewhere, use a `change.move_block` block with `to_page`.

New blocks land at the end unless you pass `after_block` when creating them —
do that rather than adding and then reordering.

## Time-period comparisons

"vs last month", "compared to the prior quarter", "year over year", "MoM/QoQ/YoY" — call `edit_table_blocks` with `comparison: { period_offset: N }` on the named block; this compares the same scenario across time, where `scenarios` compares the same period across scenarios. A block carries one comparison kind at a time, scenarios or time, never both. To answer without changing the block, use the headless `ask.calculate` comparisons under When not to use.

## table_config shape

Strict shape. Don't improvise field names — the server validates against this exact schema:

```json
{
  "name": "Payment Amount by Date",
  "prompt": "",
  "didOverrideAIGeneratedName": true,
  "rows": [
    {
      "id": "<uuid-you-generate>",
      "type": "ROW_PROPERTY",
      "drillInPath": [],
      "children": [],
      "inheritFromParent": true,
      "parentId": null,
      "propertyUris": ["runway:properties/<variable-property-id>/"],
      "useAs": "VARIABLE",
      "dimensionItemInclusionFilter": [],
      "overrideId": null
    }
  ],
  "columns": [
    {
      "id": "<uuid-you-generate>",
      "type": "COLUMN_PROPERTY",
      "drillInPath": [],
      "children": [],
      "inheritFromParent": true,
      "parentId": null,
      "propertyUris": ["runway:properties/<date-property-id>/"],
      "dimensionItemInclusionFilter": [],
      "overrideId": null
    }
  ],
  "settings": {
    "dateGranularity": "MONTH",
    "dateRange": {
      "type": "ABSOLUTE",
      "start": "2025-01-01T00:00:00Z",
      "end": "2026-12-31T00:00:00Z"
    }
  }
}
```

### Schema rules

- The `type` discriminator for each row/column must be one of: `ROW_PROPERTY`, `ROW_FORMULA`, `ROW_FREEFORM`, `COLUMN_PROPERTY`, `COLUMN_FORMULA`, `COLUMN_FREEFORM`. For building a table from `inspect_variables`/`inspect_dimensions` results you almost always want `ROW_PROPERTY` / `COLUMN_PROPERTY`.
- `segmentDrillIns` is a valid optional top-level field (saved drill-in expansions). Omit it when creating a table; preserve its existing value on whole-config updates.
- Variables and dimensions are referenced through the internal field `propertyUris: ["runway:properties/<id>/"]` — an array of URI strings, not a bare `propertyId`. The id comes from the `id` field of an `inspect_variables` or `inspect_dimensions` result. Always include the trailing slash.
- Keep variables on one axis. In a monthly table, put variables on rows as `ROW_PROPERTY` with `useAs: "VARIABLE"`. The Date column is not a variable.
- For time, use a `COLUMN_PROPERTY` that points to the system Date dimension.
  Omit `useAs`. Never mark system Date as `VARIABLE` or `DIMENSION`. If you only
  need its ID, request `inspect_dimensions` with
  `{"ask": {"dimensions": {"names": ["system_date"]}}}`.
- Keep required arrays as arrays. Use `[]` when `children`, `drillInPath`, or
  `dimensionItemInclusionFilter` is empty. Never use `null`.
- Generate a fresh UUID for every row/column `id` you are adding. When a full-config update replaces an existing table, keep the existing node ids and mint new ones only for added rows/columns — re-minting an existing row's id severs its saved widths and drill-ins. Leave `parentId` and `overrideId` as `null` unless you know otherwise.
- To filter a dimension to specific items, populate `dimensionItemInclusionFilter` with dimension item URIs from `inspect_dimensions`.
- When the user asks for a variable "by", "broken down by", "pivoted by", "grouped by", or "drilled into" a dimension, interpret that as **Segment by**: make the variable the parent row and the dimension a child row in that variable's `children`. The variable row uses the internal `useAs: "VARIABLE"`; the dimension child uses `useAs: "DIMENSION"` and `parentId` equal to the variable row's `id`. Do not add the variable and dimension as separate top-level sibling rows unless the user asks for them separately. The same pattern holds in columns: supply explicit `children` when the user requests a specific column drill-in, and the server preserves them.

## Variable and dimension selection

- **Prefer variables and dimensions from the same integration source.** Don't mix QuickBooks with Gusto unless the user asks for cross-source analysis.
- **Same-axis dimensions: read the overlap facts, not the item lists.** An unfiltered `inspect_dimensions` listing appends `item_overlaps` — the dimension pairs whose items name the same things (exact matches, abbreviations, spelling variants) — plus `more_pairs_not_shown` (always stated, zero included) and `near_matches_capped` (the near-match comparison hit its ceiling; equal spellings are always found). Judge whether two dimensions are one business axis from those facts rather than eyeballing item lists. The facts ride only on an unfiltered listing — a call that names its dimensions gets none.
- **Exception for time-series:** use the system Date dimension (`isSystemDate: true`) in columns even if its source differs from the variable.
- **Semantic matching** for fuzzy user language:
  - salary ~ payment amount / compensation / total pay
  - employees ~ people / staff / headcount
  - revenue ~ bookings / contract value / deal amount / ARR
  - expenses ~ costs / spend / payments / disbursements
- **Prefer specific over generic.** "Revenue" beats "Amount" when both exist.

For domain-specific variable and dimension guidance, see the Domain heuristics section below.

## Compiled P&L analysis and map validation

The statement playbook owns classification and row design. One verticalized tool, `inspect_pnl`,
supplies the bounded computation and validation, as two concerns.

### Analyze the GL set

Call `inspect_pnl` `ask.gl_materiality` with the resolved amount variable, Account Type
dimension, exact GL-name dimension, and every candidate Class/Department-like dimension in
`extra_dimension_ids`. Do not substitute a raw `inspect_model_views` dump and manually
recreate the aggregation.

The result supplies, per GL, a three-month `monthly_avg` and `percent_of_section` share plus a
materiality label, along with total GL count, small-business mode, dominant-stream/refund signals,
extra dimension items, and truncation state. The fixed three-month lookback is a recent snapshot —
a longer trend will not show. Keep raw GL names exactly as returned, prefixes included, for the map.

If `truncated` is true, warn and do not claim complete coverage. Inspect all extra-dimension
candidates; an empty Class field does not prove Department is empty.

### Validate the map

Call `inspect_pnl` `ask.bucket_map` with the proposed map and the source Account Type and GL
name dimensions. Correct and retry until valid. Within the account types you submitted, the
validator covers: every source GL exactly once, no unknown names, no duplicates, no empty buckets.

The validator only inspects account types present in the submitted map, so a wholly omitted source
section still passes. Before accepting the map, compare its account types against the source
Account Type dimension yourself and confirm no populated section was left out.
The validator consumes raw exact GL values, while the report preview uses clean display labels.

## Compute budget-vs-actuals variances

Call `bva_analysis` with `block_id` and the budget scenario: it returns per-row, per-period
`actual`, `budget`, `variance` (actual − budget), and `variance_pct` against the signed budget,
capped at `max_rows` (default 100) with a `truncated` flag — raise `max_rows` or narrow the block
when it is set. Use it instead of hand-computing variances whenever a budget scenario exists;
re-sign against |plan| afterward only as a presentation choice for negative plan lines.

---

## Domain heuristics for table blocks

Use these heuristics to choose variables and dimensions when the request is ambiguous or the dictionary returns many candidates.

### Accounting (QuickBooks, Xero, NetSuite)

- **Variables:** payment amount, expense amount, total amount, invoice amount
- **Row dimensions:** GL account, vendor, department, expense category
- **Column:** system Date (prefer over payment date)

### HRIS / Payroll (Gusto, BambooHR, Rippling, ADP)

- **Variables:** salary amount, total compensation, headcount, hours worked
- **Row dimensions:** employee, department, location, role, employment type
- **Column:** system Date (prefer over pay period / effective date)

### CRM / Sales (Salesforce, HubSpot, Close)

- **Variables:** contract value, revenue, deal amount, pipeline value, win rate
- **Row dimensions:** customer/account, sales rep, product, region, stage, lead source
- **Column:** system Date (prefer over close date / contract date)

### Generic / Unknown

- Internal `VARIABLE`-type properties are variables; `STRING`/`ENUM` properties are row dimensions.
- When unsure, pick the most amount-like variable for cells and the most category-like dimension for grouping.

<!-- embedded-skill:presentation-and-voice:start -->

## Embedded skill: presentation-and-voice

# Presentation and voice

House style for finance delivery.

## Lead with the point

- Start with the strongest conclusion, decision, or exception. Method and supporting detail come
  after it.
- Keep routine success concise. Spend words on assumptions, unresolved checks, risks, and choices
  that change the result.
- Avoid false precision when communicating thresholds and rounded values.
- In completion responses, interpret rich artifacts instead of restating cells. Include source
  status and next decisions; nest blocks under their page and list unrelated artifacts as peers.

## Make artifacts scannable

In this section, an artifact is a table block.

- Lay a report out as a trajectory: periods across the columns (months unless asked otherwise),
  variables down the rows, breakdowns nested beneath; only a mapping table, a database view, or
  an assumptions list the user asked for is timeless.
- Use sentence case for business rows and labels, preserving established customer terminology.
- Prefer fewer, clearer digestible rows over a chart-of-accounts dump, while preserving
  drillability and completeness.
- Use natural management signs: revenue and spend lines read as positive amounts, and derived
  profit or loss carries the result. Distinguish zero from missing data.
- Net contra accounts inside their parent line unless gross presentation is material or preferred.
- Put related lines in a stable logical order. Within each financial-statement section, place
  detail lines first and put each subtotal or total immediately after the lines it summarizes.
- Show subtotals and derived rows distinctly; keep assumptions and provenance beside their outputs.
- Use consistent units, date labels, rounding, and comparison bases throughout an artifact.

For a P&L, scannable formatting looks like this:

- **Tier 1 — Detail lines** (e.g. Subscription revenue): indented, no color background, no
  bolding.
- **Tier 2 — Structural totals** (e.g. Total revenue, Total cost of revenue, Total operating
  expenses): don't indent (flush-left), bold, no background.
- **Tier 3 — Mid-tier subtotals**, where one exists (e.g. Total non-headcount expense within the
  broader operating expense section): indented + bold, no color background.
- **Tier 4 — Calculated milestones** (e.g. Gross profit, Operating profit/(loss), Net
  profit/(loss)): flush-left, bold, background color.
- **Tier 5 — % variables** (e.g. Gross margin %, and Op margin % / Net margin % if you add them):
  italic, not bold, with a named fill distinct from tier 4's, aligned flush-left.

Note: within one artifact, all tier 4 lines share one named fill and all tier 5 lines share a
different one. The fill palette is a closed name set with no lightness control.
Styles are written the same way the table's own menus set them; the table-building manual teaches
the formatting write surface and its batching constraints.

## Commentary discipline

- A sentence earns its place when it explains a material movement, a ranked contributor, a
  decision-relevant risk, or an uncertainty the reader could otherwise miss.
- Quantify the observation and anchor it to a period or comparison. Prefer standard comparisons —
  year over year, quarter over quarter, trailing three, six, or twelve months — over an arbitrary
  raw month count when the data supports them.
- Keep one primary fact per sentence and merge points driven by the same cause.
- State observed dynamics as observations. Label hypotheses, recommendations, and assumptions as
  such.
- Where actuals hand off to forecast, state the handoff. A blended view is fine; an unlabeled one
  is not.

## Match the audience

- With a finance leader, use standard finance shorthand and emphasize reconciliation, drivers,
  assumptions, and control points.
- With a founder or operator, translate rates and accounting terms into dollars, counts, timing,
  and business consequences; spell out abbreviations on first use.
- For a board or investor audience, lead with performance against plan, the forward outlook,
  material risks, and decisions required. Keep operating detail available but subordinate.

## Review loops

When presenting your draft thinking in a chat, show enough detail to make corrections concrete. Ask pointed
questions about uncertain areas you want user input on, don't ask generic questions like "does
this look right?". When corrected, re-present only what changed unless the user asks for the whole
artifact.

<!-- embedded-skill:presentation-and-voice:end -->
