# The modeling method

Most modeling failures start by building before understanding the data. Use
this method by default, but skip or reorder steps when you already have their
output. Two steps are required: understand before building, and verify against
expected numbers afterward.

## Step 1 — Frame the question

First write the table signature (references/03-table-blocks.md §3.9):
_variable(s) by dimension(s), with or without time_. State the decision it
supports. Discuss this signature, not config JSON. Then choose the output:

- An existing number or a one-off analysis: do not build anything. Evaluate
  ephemerally (references/02-formulas.md §2.8) and answer in chat.
- A named, reusable quantity absent from the model but derivable from supported
  sources: follow [[build-model]] and save one variable with its formula. Do not
  add a table, page, or other artifact unless the user asks for one.
- A ranked or top-N ask ("top 10 customers", "who drove the drop"): an
  `inspect_model_views` `ask.rank` question, never a build. Ask, answer in
  chat; nothing is saved (references/03-table-blocks.md §3.8b).
- A view someone will return to: a block.
- A visual (chart, KPI scorecard, custom layout): a code block, fed by
  table-config datasets. Design each dataset's signature with the same
  care as a table's.
- A statement or report (P&L, board pack): a page of blocks, built after the
  entries and formulas exist.

If the sentence has no variable, you are building an item listing (legal) or
you have not understood the request yet.

## Step 2 — Learn the dictionary

Read workspace entries before assuming they exist. **Read `inspect_variables`
and `inspect_dimensions` in parallel.** Resolve every variable name from the first
and every dimension name from the second; fall back to `resolve` `ask.grammar` only when
the dictionary cannot answer. Absence is not proof of nonexistence: an
`ask.items` page reports `source_items` — whether the source half of the
list landed at all — and coordinates pinned to a block are not listed as items
of any dimension; they are read from the block that shows them.
Hand-created items are not a separate read; they sit in the same list,
tagged. The `build-model` manual carries the full rule (the `resolve` `ask.grammar` fallbacks and
item paging). What matters here is what each entry hands you: its kind,
data type, and _source_ (which integration query made it), plus three
lines to copy from the listing instead of re-deriving:

- **`grammarRef`** is the entry's exact spelling for views and
  readable formulas, backticks and `#hex` disambiguator included. Copy
  it; never derive a disambiguator from a UUID by hand.
- **`items`** (dimensions) is the live item count, with every spelling
  inline for small dimensions and samples for large ones.
- **`slicesBy`** (variables) lists the dimensions this variable can fan out
  over (same source table, plus inheritance through the variable's formula
  references; the system Date spans sources, references/04-time.md §4.1).
  Fan a variable out only over dimensions it lists. Any other slice
  computes, but renders blanks or replicated parents instead of an error.

For variables the build depends on, read their formulas before trusting the
numbers — and read them by layer, not as a list. Classify each formula as the
default, a time window, a grain rule, or a pinned cell, and the variable's
story appears: what it means anywhere, what changes in forecast, and which
grains it actually reaches. A lone rule at the date grain is a variable that
has never been drilled into (references/09-the-layer-model.md §9.8).

This step ends with an inventory: the candidate variables, the dimensions
that can actually slice them, and each dimension's meaning. Every table
you can offer comes from that inventory; without it, "slice by X" and
"pivot by Y" are guesses.

Each extra round-trip costs thinking time; spend it only when one read's
input comes from another's output.

## Step 3 — Survey the blocks

**Reuse before you build.** One `inspect_table_blocks` `ask.list` read is
the block dictionary: every TABLE block's signature, with live `# items:`
counts where counting is cheap, a `signature_status`, and a ready-to-paste
`ref` token. Compare those signatures against your target sentence and
route:

- **A match**: the table exists. Point at it (paste its `ref`) or read its
  data; do not build a twin.
- **A near match**: edit, do not build. Route by what differs. Structure
  or data (variables, breakdowns, filters): `edit_model_views`
  naming the table when the user means _this_ table changed, or a
  `change.configure_table` entry with `copy_from` when they want another view
  with the original kept. Presentation only (rename, column widths,
  hidden axes, window, comparison): one `edit_table_blocks` call on the
  named block — never a view reconfigure
  (references/12-editing-blocks.md §12.5 decides which).
- **Nothing close**: design and build (steps 5–6).

Trust `signature_status`. `complete` means the signature is the whole
config; never re-fetch raw config just to check it. `partial (…)` names
what a view cannot express (references/12-editing-blocks.md §12.6).
Existing blocks are the best record of the signatures this workspace
already uses, and rebuilding forks the truth into two near-identical
tables.

Item values come from the same two reads: step 2's `items` lines and the
`# items:` counts here. When a dimension's line shows samples rather than
every spelling, page it with `ask.items` — that is the full enumeration,
merged across both stores. Reach for `resolve` `ask.grammar` when the
dimension is too large to page to the value you need, or the line says
`source items could not be read`. Copy spellings exactly; filters and pins
match exact strings (references/06-validity.md §6.4.6). Hand-created items
come back in the same list, tagged (step 2); coordinates pinned to a block are
not items of any one dimension and are not listed, so for those a missing
item does not prove absence
(references/01-the-dimensional-universe.md §1.4). The
items count is the cardinality check: a dimension with thousands of items
belongs behind a filter, not fanned across columns. If "None" is
present, decide what uncategorized rows mean for this table and whether
to filter or surface them.

## Step 4 — Probe only unanswered questions

The listings answer most probe questions up front: `slicesBy` rules out
wrong-source fan-outs before anything is built, and the write's readback
shows real values one call later (step 7). A probe is not a gate before
every build. Evaluate ephemerally (references/02-formulas.md §2.8; nothing
remains, and recompute works exactly like saved variables) when the answer
lives in the data and the listings are silent:

- Which date column a variable follows, when its source carries several
  (references/04-time.md §4.1).
- Where data ends: read the current month at day granularity. The last day
  with data is the freshness watermark; do not present a half-synced month
  as a full one.
- A slice or intersection the `slicesBy` lines leave genuinely uncertain.

## Step 5 — Design the view

Now, and only now, shape the config (references/03-table-blocks.md, recipes in references/08-recipes.md):

- Variables as root rows with the driver role; each dimension breakdown _nested
  as a chain_ under its variable, not as sibling rows. A root row may instead be
  a bare dimension held as a mapping (`{"dimension": "City"}`) when the cell
  should show that dimension's item rather than a measured value; a mapping
  takes no breakdown and no grain.
- Exactly one date axis for time series, on columns, pointing at the system
  Date, with an explicit granularity (usually month) and an explicit date
  range.
- Item filters where the question names specific items; drill-in pinning
  where the question is about one item's interior.
- Comparison: scenario or time, never both (references/05-scenarios-and-comparisons.md §5.5 decides which).
  The offset counts in granularity units. Treat scenario names like item
  spellings (step 3): list scenarios and copy the display name exactly
  before writing a compare line. Names resolve server-side; a name shared
  by two scenarios fails the parse — surface the ambiguity instead of
  guessing.
- Ratios and averages: decide what the parent row should show (references/06-validity.md §6.4.1), and
  set the aggregation function so time rollup behaves (references/04-time.md §4.4).
- Name the block in the "&lt;Variables&gt; by &lt;Dimensions&gt;" shape unless the user
  named it.

Then, before any formula is written, run the **grain check**
(references/09-the-layer-model.md §9.4). List the grains this design will
actually evaluate — every level of each row path crossed with every
date-carrying column level, the coarser time grains the granularity ladder
will visit, and each of those under both regimes when the variable splits
actuals from forecast — and for each one name the formula that answers it.
The coarser time grains descend the view's one date axis
(references/04-time.md §4.4):

- the system Date, where present;
- else the sole foreign date-typed dimension with a real granularity —
  where first/last/any decline the descent and compute in place;
- several foreign date axes with no system Date descend nothing.

Any grain whose only answer is the floor renders source sums or zeros: either
that is what you intend, or a layer is missing. The check costs no tool calls
and it is the cheapest place to catch drill-in zeros, the blank quarter, and
the zero forecast — all of which are otherwise invisible until someone reads
the built table.

## Step 6 — Build

Write directly — grammar, formulas, and creates validate themselves and
save nothing on error; pre-flight an update you want to check first,
with `edit_model_views` `dry_run`. One read first when adding a term-list
rule: `ask.saved_formulas` on the target variable. If a rule already
covers that grain and overlaps the new one at equal specificity, the
overlap is owned by whichever saves last and any re-save flips it
(references/10-deviations.md D1) — update the existing rule via `formula_id`
instead of stacking a rival. Order of operations for anything
beyond a single block: entries first, then formulas, then blocks/pages
that reference them. Get plan approval before creating artifacts. Pages
need no pre-step: pass `page` and the page is reused when it exists,
created when it is missing, in the same write. Prefer updating the block
you just made over creating a near-duplicate; for statement builds, create
the page and its blocks in one batch. Three things to get right:

- **An update states the whole structure, never a diff.** Prefer the
  `view` form of `edit_model_views`. It is id-free and
  reconciles in place: matched structure keeps its axis ids and aggregate
  functions; the formula lane, surviving segment drill-ins, and
  presentation state (widths, hidden axes) carry over; drops come back as
  warnings. A view is structure only, so there is no window, granularity or
  comparison line to restate and no omission that clears one — presentation
  survives the write, and changing it is an `edit_table_blocks` call
  (references/12-editing-blocks.md §12.3). The one omission that does change
  the block is orientation: restate `transpose: true` on a transposed
  non-mapping block (references/11-block-grammar.md §11.3). A
  `change.configure_table` entry with `copy_from` gives a reconciled NEW block
  with the original kept (references/11-block-grammar.md).
- **Regime-scoped formulas**: write them with `change.set_values` with `period`;
  it resolves the window by name. Reading and moving Last close:
  references/04-time.md §4.5. Regimes you tile by hand (non-book series,
  references/04-time.md §4.5) must match the break boundary shapes: a
  window's start lowers to `Date > start` (exclusive), its end to
  `Date <= end` (inclusive). Otherwise adjacent regimes double-count or
  gap at the boundary.

- **Every grain the check named needs coverage, not necessarily its own rule.**
  One subset condition can answer every richer grain containing the dimensions
  it names. For a regime rule meant to survive dimension drill-ins, write the
  Date shape explicitly, such as `[Date.Month in any]`, with the `period`; use
  `$[Date.Month in any]` only when it should stop at the unsegmented time row.
  Coordinate bounds always compose an exact `$[…]`, so a drill-in-surviving
  rule needs its shape written in `condition`; read the echoed condition and
  `uncovered_block_grains` rather than fanning the same formula out per
  grain (references/limitations.md §7,
  references/09-the-layer-model.md §9.1).

When a formula targets a slice shown in a specific table, pass that
block's table name as `block`: the tool completes the address against the
block and reports what it filled in, and term order never matters (the
tools canonicalize it). The result then carries `uncovered_block_grains`: the
grains that block evaluates which this variable has no formula for, one entry
per formula still owed, each naming the grain and the range it belongs to. That
list is work remaining, not a warning to acknowledge and move past.

Read an empty list narrowly. It means every grain the block evaluates has a
formula in each range this variable already uses somewhere — and it can say
nothing about a range the variable uses nowhere. A variable with no forecast
formula at all therefore reports nothing here while its forecast months read
zero at every level. That case is uniform down the table rather than a gap at
one level, so the readback grid is what catches it (step 7).

## Step 7 — Verify from the readback

The verification read is already in hand: every successful create, update,
or copy result carries a `readback` section, an engine-computed
depth-1 grid of the block just written. Read the result's `warnings` list
first: an aggregation warning means consult references/13-metric-recipes.md
§13.4 before reporting done, and reconcile drops surface there too. Then
run the checks on the readback, in order: no #ERR cells (fix at the origin
of the trace); no unexplained blanks (references/06-validity.md §6.5);
parent rows mean what you intend (the laws in
references/13-metric-recipes.md §13.3); the date range shows the periods
the user cares about; totals match any probe you ran.

Behind those checks is one rule: decide what the cells should be before you
trust them, worked out from the source you inspected, then read the grid
back and reconcile. A value that does not match the number you expected is
the model built wrong, not the data. The gap tells you which part to fix,
and it tells you to keep going rather than settle for what came back. Five readings catch
the mistakes that hide behind a grid that looks reasonable:

- A correct parent row above drilled children that are all zero is the model,
  not the data: no formula answers at the children's grain, so they fall to
  the floor while the parent computes at its own grain
  (references/09-the-layer-model.md §9.5). The asymmetry is the tell — a
  variable that is genuinely empty is zero at _both_ levels. But so is a
  stranded one: saved formulas whose conditions name dimensions with no Date
  term sit at a grain no shown row shares and floor every cell
  (references/limitations.md §11) — read `saved_formulas` before taking
  uniform zeros as emptiness. Nothing errors
  and nothing is blank, so this passes every other check on this list. Write
  the missing rule at the drilled grain; never report done over it, and never
  explain it to the user as a source gap.
- A parent equal to one of its children's values — or to one branch of an
  `if` — while the children differ is not a rollup: a globally-addressed rule
  owns the collapsed parent, where the dimension tests bind nothing and the
  leftover branch replaces the recomputed aggregate.
  the `formulas` section attributing the same formula to the parent and
  its children confirms it (references/10-deviations.md D7).
- A source-backed variable returns a confident zero, never a blank, for a row
  whose segment has no matching source rows
  (references/01-the-dimensional-universe.md §1.5). Once the drilled grain
  has a rule, all-zero breakdown rows mean the row addresses items the data
  does not carry, most often a text dimension pushed onto the date axis (a
  signup month read as `cohort.Month` instead of `cohort`), not that the
  source is empty.
- A comparison whose variance is the size you expected still proves nothing
  about which side is which. Confirm the plain column names the thing you
  were asked to analyze before you trust the variance
  (references/05-scenarios-and-comparisons.md §5.4).
- A value that looks like a date is not always a date. Read the dimension's
  data type and items before you treat a column as a time axis; a
  categorical key spelled like a month stays categorical
  (references/01-the-dimensional-universe.md §1.2).

The readback is bounded, not total: a separate `inspect_model_views` call
is needed only for deeper slices (a scope below depth 1, provenance
inputs, more rows), or when the result says `readback unavailable`.

The other writers echo differently, and the discipline follows the echo.
`change.set_values` returns no computed values — its echo says _where_
the write landed: read `written` to confirm the stored condition and
grain, `bounds`/`window` to confirm the regime, and
`completed_from_block` for terms the anchor injected; then values, if
they matter, come from the block's readback or `inspect_model_views`.
`edit_table_blocks` computes a readback only for the aspects that move
values — window and comparison; a rename, sort, widths, or visibility
change echoes none because none is needed.

Only then report done, in one sentence, without re-rendering the table
in chat.

## Missing instruments

Gaps in the tooling, with the working substitute.

| gap                                                   | substitute                                                                                  |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| no full enumeration of generated / block-pinned items | read the block configs that reference them; remember absence ≠ nonexistence                 |
| variable value stats (min/max, null rate, freshness)  | ephemeral probes; the day-granularity watermark read                                        |
| deleting a formula                                    | overwrite it; there is no agent-side formula delete, so a blank or overwrite is the removal |

## Six rules

1. Understand before building; batch independent reads into one round
   (the inspect tools are parallel-safe); probe only what the listings
   cannot answer.
2. Compute with the engine; never do model arithmetic in prose.
3. Copy names and item values from tool output; never from memory.
4. Write grammar, formulas, and creates directly (they validate
   themselves); every formula of one intent goes in ONE `change.set_values`
   call — a subset condition covers the richer grains containing it, so
   items exist per intent, not per grain; pre-flight an update you want to
   check first, with `edit_model_views` `dry_run`.
5. Updates state the whole structure as a view; comparison and window
   changes are `edit_table_blocks`, not a restate.
6. Verify from the write's readback, warnings first; reconcile against the
   numbers you expected, and read a mismatch as the model, not the data.
   Report in one sentence.
