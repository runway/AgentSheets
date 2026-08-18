---
name: table-building
description: Create, check, review, or update pages and saved financial-report tables. Use when the user wants a saved table, not just an answer in chat.
---

# Table building

<!-- standards:start -->

Decide whether this work is a chat answer, a new artifact, or a revision before writing. Use
headless inspection for analysis.

<!-- standards:end -->

## Creating table blocks

Use `edit_model_views` with `change.configure_table`: it writes to an existing page, and creates
the page when `page` names one that does not exist yet. A markdown table, preview, or description
is not a saved result.

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

For a chat answer rather than an artifact, prefer `inspect_model_views` with `ask.calculate` for table-shaped reads (pass `compare_scenarios` for scenario comparisons), or `inspect_variables` `ask.try_formulas` when the exploration needs formulas that are not saved variables — it takes named `expressions`, a required `from`/`to` range, and an optional `rows_by` dimension, and returns one record per period per item. The probe shape is spelled out in [[dimensional-modeling:references/02-formulas.md]] — read that file directly; you do not need to load the manual first.
For time-period questions like "how did revenue change vs last quarter?", answer headlessly with `inspect_model_views` (`ask.calculate`) and its `time_comparison` parameter instead of persisting a block — the grid header carries a comparison legend: scenario names, or `time(-N)`. The same tool's other asks: `ask.rank` returns a top-N shortlist from a declared layout (max 50) when the question is which rows lead, and `ask.formatting` is the read half of `change.formatting` — it answers in the same words the write takes, and also surfaces styles stored against rows or columns that no longer exist.

## Workflow

1. Call `inspect_variables` and `inspect_dimensions` to see what variables and dimensions the workspace has. Inspect `source`, `type`, and the `isSystemDate` flag before choosing.
2. If the request names an item from an ingested business dimension, call
   `inspect_dimensions` to confirm its spelling, then name it in the view's
   breakdown filter: `[Region in {East, West}]`. Items go in the grammar by
   name; there is no URI to thread.
3. If the user asks to compare against another scenario, pass the scenario by name. Use `inspect_scenarios` when you are unsure what exists, and ask for clarification when a name matches more than one scenario.
4. Finish the table definition, then call `edit_model_views` once with the complete `view`.
   Do not call it early with only a page name or progress message. The writes
   validate themselves, a view that does not check returns its problems, and nothing persists on
   failure. Reach for `dry_run` when a write is risky enough to preflight.

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
   the view plus window, comparison and sort in the words `edit_table_blocks` takes.
   It nests under `ask`; the only top-level fields are `scenario` and `ask`.
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
to it. Use `edit_pages` with a `change.reorder_blocks` block. `place: {block, after_block, to_page}`
moves one block: `after_block` names what it sits below on the page it ends up on (omit = end of
that page, `null` = top), and `to_page` sends it to a different page — a name matching no page
creates it, though not when you also pass `after_block`, which names a block on the destination
and so needs that page to exist already. `block_order` restates one whole page instead. The page the block is on now is named
by `page`, beside `change`.

New blocks land at the end unless you pass `after_block` when creating them —
do that rather than adding and then reordering.

## Arranging pages

Pages nest. A page can hold subpages, which hold their own, and the sidebar shows that tree
indented. `inspect_pages` lists pages in sidebar order with each one's `depth` and `parentId`, so
read it before describing the workspace: a page you report as top-level when it sits under another
is a page the user cannot find from your description.

"Put the department budgets under Budget" is a real instruction, not a figure of speech. Create a
page in place with `change.add_page` and its `parent`; move one that already exists with
`change.move_page` — `parent` names the page it goes under, `null` moves it back out to the top
level, and omitting `parent` only reorders it among the pages it already sits beside. `after_page`
orders it there: omit for last, `null` for first, or name the page it sits below. A page takes its
subpages with it. Never delete and recreate a page to move it — that destroys its blocks.

To reorder several pages at once, pass `move_page` a `page_order` instead of a single page: every
page directly under `parent`, exactly once, top to bottom (`parent` null or omitted is the top
level). It is one write rather than one per page, and like `block_order` a partial list is refused.
It reorders within a level and cannot re-parent — a page arriving from somewhere else is a move.

Reach for a subpage when one page has grown past a single audience and purpose: the supporting
detail behind a summary, or one section of a build with several. Nest what a reader opens _from_
the parent page, not everything that shares its topic — a tree deep enough to hide a page is worse
than a flat list.

## Time-period comparisons

"vs last month", "compared to the prior quarter", "year over year", "MoM/QoQ/YoY" — call `edit_table_blocks` with `comparison: { period_offset: N }` on the named block; this compares the same scenario across time, where `scenarios` compares the same period across scenarios. A block carries one comparison kind at a time, scenarios or time, never both. To answer without changing the block, use the headless `ask.calculate` comparisons under When not to use.

## Variable and dimension selection

- **Prefer variables and dimensions from the same integration source.** Don't mix QuickBooks with Gusto unless the user asks for cross-source analysis.
- **Same-axis dimensions: read the overlap facts, not the item lists.** An unfiltered `inspect_dimensions` listing appends `item_overlaps` — the dimension pairs whose items name the same things (exact matches, abbreviations, spelling variants) — When the entry is present it carries four bounds, each always stated: `more_pairs_not_shown` (zero included), `near_matches_capped` (the near-match comparison hit its ceiling), `not_compared` (dimensions left out because their items could not be read — never scored on a fragment) and `compared_items_capped` (a compared dimension contributed only the spellings that fit under the fetch cap). No entry at all means the comparison ran whole and found no overlap, which is an answer rather than a gap. Judge whether two dimensions are one business axis from those facts rather than eyeballing item lists. The facts ride only on an unfiltered listing — a call that names its dimensions gets none.
- **Exception for time-series:** use the system Date dimension (`isSystemDate: true`) in columns even if its source differs from the variable.
- **Semantic matching** for fuzzy user language:
  - salary ~ payment amount / compensation / total pay
  - employees ~ people / staff / headcount
  - revenue ~ bookings / contract value / deal amount / ARR
  - expenses ~ costs / spend / payments / disbursements
- **Prefer specific over generic.** "Revenue" beats "Amount" when both exist.

For domain-specific variable and dimension guidance by integration source, read
`references/domain-heuristics.md`. When laying out a financial statement, follow the five
formatting tiers in `references/formatting-tiers.md`.

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
