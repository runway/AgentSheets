---
name: dimensional-modeling
description: Explain how segments, grains, axes, formulas, rollups, tables, pivots, validity rules, time, scenarios, and comparisons work. Use when designing, changing, debugging, or explaining a model, especially rollup versus recompute behavior or granularity, when numbers look wrong or cells are blank, or before a multi-step model build.
---

# Dimensional modeling

This skill is the conceptual foundation beneath the build procedures in the
`build-model` manual and the statement and forecast playbooks: the laws of
the calculation engine and the table-block view model. Those guides tell you
which tools to call; this one tells you what the system _means_, so your
designs are right before you build. It has two layers: the axioms below
(read them now), and detailed references you load when the
task touches their area.

The laws are facts and hold everywhere; the method and recipes are
guidance. When a request calls for a route the references do not cover,
reason from the laws and be creative.

<!-- standards:start -->

Treat the engine laws as facts and the method, workflow, and recipes as guidance; resolve variable names from `inspect_variables` and dimension names from `inspect_dimensions` in one parallel round; name the grains a table will evaluate and say which formula answers each before writing formulas for anything carrying a breakdown; and check a symptom against the known limitations reference before redesigning a model that is already correct.

<!-- standards:end -->

## The axioms

### The space

1. A workspace's model is an n-dimensional space. **Dimensions** are the axes
   (Department, Region, Date). **Dimension items** are the values along one
   axis (Engineering, Sales). A **segment** is one intersection: a set of
   dimension = item pairs, like `{Department = Engineering, Month = 2026-01}`.
   The empty segment is the whole-model total.
2. **Variables** are functions over that space: Revenue has a value _at every
   segment_, not one value. A **grain** (also called segmentation) is the set
   of dimensions you are currently computing across. The same variable at
   different grains gives different numbers, all correct.
3. Entries (variables and dimensions together) are the workspace's data
   dictionary. They are pure metadata; values come from source data or from
   evaluation. Source data arrives through integrations as external tables;
   each source column becomes an entry.
   3b. A dimension's items usually come from its source column, but they can
   instead be **mapped** from another dimension: a lookup saying which Bucket
   each Account belongs to, which is how GL accounts become P&L buckets. A
   **dimension mapping** — distinct from the value-axis mapping of axiom 11 —
   states a classification once, as structure the whole model can
   address — a judgment that would otherwise repeat across formulas, filters,
   or hand-grouped tables wants to be one; a one-off arrangement does not.
   Mapping inputs are inferred from the distinct nonempty condition grains on
   those dimension formulas; there is no separate root declaration. A
   dimension's authored items come from a direct set formula or from a mapping,
   never both; a source-backed dimension may retain its imported column while
   adding mapping inputs. Several imports spelling one business concept still
   get one dimension — each source's column mapped into it, the way every
   source's dates fold into the system Date. Author a dimension mapping with
   `edit_dimension_mappings` and read it back, grouped by lookup, with
   `inspect_dimension_mappings` (references/14-dimension-mappings.md). A
   mapping is a judgment made on the user's behalf, so show what you matched
   — the pairs themselves, in the reply or in something saved — rather than
   only reporting that a mapping now exists; a classification they cannot
   see is one they cannot correct. Showing the pairs in the reply is enough
   when the mapping is the whole ask; build the block when the user wants
   something modeled on top of it.

### The calculus

4. **Everything is a formula.** Every variable value at every grain is produced
   by evaluating a formula at that segment. A variable with no authored formula
   gets a synthesized default: the sum of its source column, or 0 if it has no
   source. There is no "no formula" state, which also means a bare variable can
   confidently show 0 rather than an error. Read that the other way when
   debugging: a grid of zeros under a correct parent is not data, it is
   dispatch finding no rule at the drilled grain and falling to this floor.
5. A formula is (target variable, condition, expression). The condition says
   _where in the space_ it applies; the expression says _what to compute_. A
   variable can carry many formulas; the most specific matching condition wins
   for each slice, and less specific formulas fill whatever remains. `$[…]`
   claims exactly the named grain, so it stops applying when a drill-in adds a
   dimension. `[…]` claims a shape and matches every grain containing its named
   dimensions, so it follows those drill-ins. `[]` is the whole-model default;
   `$[]` is the empty segmentation. The terms in either nonempty shape choose
   items inside the dimensions it names (references/09-the-layer-model.md).
   The condition also
   fixes _when_: a Date term that matches many dates and names no explicit
   regime is confined to actuals, so a rule that reads like a default over time
   governs nothing in a model whose periods are all forecast. **Address and
   regime must both match before a rule governs a cell.** When a cell reads
   zero, ask both questions, not just the first.
6. **References are relative by default.** Inside a formula, `Revenue` means
   Revenue at the current segment. Brackets re-aim specific dimensions while
   inheriting the rest: `Revenue[Region = "East"]` keeps every other dimension
   of the current segment. Only a `$` written after the name (`Revenue$`) escapes inheritance entirely.
7. **Recompute, not rollup.** A coarser cell is produced by evaluating the
   formula at the coarser grain, never by adding up already-computed child
   cells. This is why a ratio drilled in by a dimension shows per-segment
   ratios and a correct (recomputed) parent, not a sum of ratios. The single
   exception, time rollup, is set at two levels: a variable's aggregation
   function (sum, min, max, first, last, any, count, or do-not-aggregate)
   is its default across every coarse time grain, and `change.set_time_rollup`
   overrides it per grain — quarter, half-year, or year — with sum, last,
   or average, the only place average is reachable. Either way the rollup
   runs through a generated formula chosen by the function, still
   grain-bound; do-not-aggregate variables skip this entirely
   (references/04-time.md). When a block answers about particular members,
   restrict it to them and check its total covers those members rather than
   the whole dataset.
   7b. The function answers one question: what is this
   variable's value for an interval, given the interval's parts? Flows
   (amounts that accumulate) sum. Stocks (states sampled at instants —
   cash, ARR, headcount) take a boundary instant: last for closing, first
   for opening. Extremes take min/max. Presence or a tally take any/count.
   Derived quantities (ratios, rates, statistics) are do-not-aggregate:
   no combination of finer values is correct, so each grain re-derives
   them. Corollary, the collapse-order law: an aggregate wrapped around a
   plain variable reference operates on one already-rolled-up number
   (references/02-formulas.md §2.3).
   7c. Rollup follows the view's single date-axis descent (axiom 13).
   Setting the function carries two
   cautions. Its effect shows only at coarse grains, so a monthly build looks
   the same under every working choice until something reads a quarter
   (axiom 20d). And the setting can silently revert: a source sync can
   clobber an explicit choice back to do-not-aggregate, so read the setting
   back after syncs (references/limitations.md §1).
8. Time offsets come in two kinds, and only one needs a floor — an
   earliest cell that resolves without the self-reference. An offset
   LOOKUP reads another variable's prior cell (`Revenue[-1]` inside a
   growth variable): a missing prior is one blank cell, and the result
   degrades gracefully. An offset CHAIN references the variable's own
   prior (a recurrence): each cell is built from the previous one. A
   missing prior counts as zero, so an unfloored chain does not error —
   it quietly starts at zero plus the first change. A floor comes from
   scope: a formula range bounds the chain, and the chain starts from the
   real values across the boundary (ranges are user-defined partitions of
   Date; an actuals/forecast split is just the common convention). Or it
   comes from a seed — one date-pinned cell that gives the chain its
   first value, on the FIRST month of the evaluated window or of the
   chain's range (a later seed leaves the months before it unfloored;
   references/limitations.md §10). A count()/sum() guard on the prior
   period computes and changes none of this; the seed is the taught form
   ([[build-model:references/formula-grammar.md]], recurrence shape 1).
   Self-reference at the same time
   segment is a circular reference and errors only the cells in the
   cycle. Broken formulas break cells (#ERR with a trace), not the whole
   table. That guarantee covers formulas only: binding a dimension to its
   source data happens before any formula runs, and a failed binding's
   blast radius is axiom 12b's.

### The views

9.  A **table block** is a view definition, not data: two ordered trees of
    "break down by \_\_\_" rules called axes, one tree for rows, one for columns.
    One axis node carries one variable or dimension. Nesting a child axis under a parent is
    **drilling in**; the chain reads "Revenue by Region by Product". A figure
    that is part of another belongs nested beneath it, not beside it as a
    peer that reads as something to add on.
10. **Pivoting is placement, nothing else.** Moving an axis between rows and
    columns changes only where its items render. The rule itself (property,
    filter, sort, granularity, children) is untouched. Corollary: the moved
    axis still crosses the content already on its new side — nest them in one
    chain (`[Region, Date.Month]` or `[Date.Month, Region]`), never as fork
    siblings (`[{Region, Date.Month}]`). Sibling branches render each axis
    alone and drop the per-crossing cells, so "regions across the top" of a monthly
    table means region-and-month columns, not region columns beside month
    columns.
    10b. **Transposing.** When the user asks to flip or transpose a whole table, a view says
    so with `transpose: true`, which swaps which half renders as rows and
    changes nothing else — the same view, laid out the other way, so the
    signature does not move. Unprompted, reach for it only on a listing — a
    dimension-mapping table or a database view, where the shared breakdown is the long
    entity axis and the value entries are its few attributes; keyed by a chart
    of accounts, the untransposed mapping is one row and hundreds of columns.
    Any other table transposes only on an explicit ask.
11. **The variable lives on exactly one side.** Along any crossing of a row path
    and a column path there must be at most one variable, and it cannot appear
    on both sides. A cell is that variable evaluated at (row segment × column
    segment). A value-axis entry may instead be a bare dimension held as a
    **value-axis mapping** — an attribute column, distinct from the dimension
    mappings of axiom 3b — whose cells show that dimension's item for the
    crossing rather than a measured value. Dimension-only tables are legal;
    they list items with no values.
12. Structure is declared, values are computed. The config knows which axes
    exist; the engine finds the actual dimension items and computes every
    cell. Filters (item inclusion lists), sorts, granularity (monthly vs
    quarterly), and generation behavior are per-axis knobs. Top-N and
    sort-by-value are not expressible in a saved block; ranked questions
    are `inspect_model_views` `ask.rank` queries.
    12b. **Every axis must earn its place.** A breakdown is not free. To render
    one, the engine has to find the dimension's items and bind every source
    table behind them. That machinery can fail, and when it does the whole
    view fails — even for cells that never read the broken source. That blast
    radius is a defect, not a law (references/limitations.md §8, temporary).

        Pay that cost wherever the values genuinely differ along the axis. For
        time that is most finance work: revenue, headcount, cash, anything that
        moves month to month belongs on a date axis, and putting it there is the
        default rather than a concession.

        What earns a timeless table is what the table is, never what its values
        currently do. Three shapes qualify:

        - a dimension-mapping table;
        - a database view — one row per item of an entity dimension, stating
          each item's attributes and own values at one selected period (a
          roster, a price book; references/08-recipes.md R13);
        - an assumptions list the user asked to see stated as such, its knobs
          set per tier, region, or cohort rather than per period.

        Break those down by the entity or unit they are stated
        per — not by nothing at all: a created table whose columns carry no
        non-formula axis while a native variable sits on rows silently gains a
        shared root Date column, so a timeless table keeps at least one
        non-formula axis (references/03-table-blocks.md §3.2). Dropping a date
        axis there loses no
        information: a block's settings hold its date range and granularity
        whether or not Date sits on an axis.

        A table whose cells aggregate
        anything — sums by category, totals over rows or periods — is a
        report, demo and dummy data included, and a report keeps time on the
        columns. Flat values are a fact about today's data, not proof the
        variable is timeless, and the flat row is the honest picture.

13. There is one system **Date** dimension that unifies every source's date
    columns. Time-series tables put it on exactly one column axis. A view
    descends at most one date axis: the system Date wherever it is present;
    otherwise its sole foreign date-typed dimension — foreign meaning any
    date dimension other than the system Date — carrying a real granularity.
    A Raw axis (no granularity set) descends nothing; several foreign date
    axes with no system Date descend nothing. On a foreign axis the
    positional aggregations (first/last/any) decline the descent and compute
    in place; on the system Date they step one grain.
    Granularity (day, week, month, quarter, half, year) is part of a time
    dimension's identity: Month-of-Date and Day-of-Date are different axes.
14. **Last close** is the movable frontier between actuals and forecast. The
    system Actuals and Forecast formula ranges split a variable's formulas
    around it: history usually reads source data, the future is projected. A
    range-scoped formula carries a Date term to hold its window. Written as
    `[Date.<grain> in any]`, that rule reaches every segmentation containing the
    Date grain, including drilled rows; written as `$[Date.<grain> in any]`, it
    stays on the unsegmented time row (references/limitations.md §7). Choose
    the condition by that intent; the coordinate bounds (`segments`, `grain`,
    `period`) always compose an exact `$[…]`, so a drill-in-surviving subset
    rule is written through `condition`.
15. **Scenarios are auto-rebasing branches** of the Main scenario. A scenario
    stores only what it changed and inherits everything else live; merging
    lands changes on Main and closes the scenario. A snapshot is a scenario
    frozen at a point in time, read-only. Pages, blocks, entries, and
    formulas are all scenario-scoped.
16. A block carries at most one comparison: scenario comparison or time
    comparison, never both. Time comparison needs exactly one date axis, and
    its offset counts in units of the table's granularity (year over year on a
    monthly table is 12). Comparison is presentation, owned by
    `edit_table_blocks` (setting one kind clears the other); a view write
    through `edit_model_views` leaves an existing comparison untouched
    (references/12-editing-blocks.md).
17. **Saving is lenient; calculating is strict.** Many broken configs save
    fine and only fail (or silently misrender) at calc time. The strict
    write paths guard this for you — a view is checked before anything
    persists, and every formula write validates each item before applying any
    of it — so skip separate pre-validation steps: write directly and iterate
    on the per-item errors a write reports (references/07-modeling-method.md
    step 6). Formula writes default to `mode: "partial"`, which applies the
    items that pass and reports the rest; pass `mode: "atomic"` when the batch
    has to land whole. `dry_run` answers the same question a write answers
    without persisting, for a write you want to check first
    (references/06-validity.md §6.1). Both are call-scoped — they sit at the
    top level beside `change`, not inside the block. `edit_variables`
    `change.operations` honors `mode` (atomic runs the batch in one
    transaction) but has no `dry_run`, and refuses one by name.

### The correspondence

17b. The axiom groups above describe one object you already know from other
tools. A condition is a box (each term clips one axis; the comma
intersects clips; `[]` is the empty clip — the whole grid). A
view re-presents that grid without touching it (pivoting is placement,
axiom 10). And the grains in play form a lattice whose nodes formulas either
pin exactly or cover upward through richer segmentations
(references/09-the-layer-model.md). Stated once
so the folklore transfers:

    | here | you know it as |
    |---|---|
    | bracket condition | a `WHERE` of per-axis filters |
    | `[]` | no `WHERE` clause |
    | a grain | a `GROUP BY` key set |
    | all grains of a variable | the OLAP `CUBE` lattice |
    | a breakdown chain | a pivot hierarchy |
    | pivoting an axis | a reshape — lossless, value-safe |
    | recompute, not rollup | "never average an average" |
    | drilled rows show zeros | a groupby no formula governs |

    The folklore behind the right column is valid here — except where the
    engine deviates from the clean structure. Before acting on an analogy
    for aggregation choice, scenario overrides, formulas that might overlap,
    offsets, or totals reconciled across grains, load
    references/10-deviations.md; it outranks this table.

### The method

18. **The dictionary comes first.** Do not design a table before you know
    which variables and dimensions exist, what each dimension slices the data
    into, and which source produced each. Every breakdown and pivot you
    can offer is determined by that inventory. If you cannot restate the
    request as the block's sentence — its signature, "&lt;variables&gt; by
    &lt;dimensions&gt; over &lt;time&gt;" (`references/03-table-blocks.md` §3.9) — you
    do not understand it yet. A rendered signature, a config JSON, and the
    spoken title are interchangeable renderings of that sentence: reason
    at the sentence level, and never mistake a rendering's bookkeeping
    (node ids, condition ordering) for the meaning. Where sources overlap,
    name the system of record first — the ledger for money, the roster for
    people and their departments — and look for a shared key before adopting
    one source's labels as the dimension. Overlapping sources also restate
    each other's figures, so check whether one already contains the other
    before adding them together.
19. **Reuse before you build.** Existing blocks record the grains this
    workspace already uses. Survey the block dictionary first: one
    `inspect_table_blocks` `ask.list` read returns each table's signature (formula-lane-only blocks have none).
    Prefer duplicating a close match (`copy_from` on an `edit_model_views`
    `change.configure_table` entry) and re-slicing it (pivot, re-nest,
    re-filter) over building from scratch. Pivot moves are always safe
    (pivoting is placement), so reshaping an existing table is cheap.

        **Entries too, and this is where reuse fails most often.** When a
        correction changes what a variable means, revise that variable — formula,
        name, format — rather than creating a second beside it. Renaming is an
        edit; a new name plus a create is a fork. Three pushbacks should leave one
        right driver, not three generations: `Capacity`, `Ramped CSMs`, and
        `CSM Capacity Linked` are three candidate truths, and the tables
        silently keep reading the oldest. Read the existing entries before
        creating one. To retire a superseded entry you cannot revise, repoint
        whatever still reads it, then delete it — the delete is refused while
        anything references it.

20. **A saved block is the user's deliverable, not scratch space.** To
    debug a formula, sanity-check a build, or explore a shape, use the
    paths that persist nothing: ephemeral evaluation (`inspect_variables` `ask.try_formulas`,
    `references/02-formulas.md` §2.8) runs any formula at any grain
    without saving, and a write's readback returns the real values one
    call later. Change a saved block (its window, its shape) only to
    change what the user sees; never narrow or reshape one as a passing
    convenience and leave it altered. The exception: reshape freely when
    that _is_ the ask, or the block is an acknowledged draft.
    20b. **Run the formula before creating the variables that assume it works.**
    Building a shape you have not built before, evaluate its central formula
    once through `ask.try_formulas` with ephemeral variables. Nothing saves.

        The asymmetry is the point. A formula that errors ephemerally errors the
        same way after you create ten variables to hold it — except now the ten
        variables exist. Variables outlive the design they were made for, so a
        failed build leaves the workspace carrying entries nobody asked for and
        nobody will recognise later. The ephemeral run costs one call.

    20c. **When a read fails and you cannot say why, simplify the read.** Re-run
    the same ask smaller: drop the breakdown, narrow the window, keep one
    variable. Repeat until it succeeds. The last thing you removed is where
    the fault lives, and now you can name it.

        Do not reach for a settings change instead — a time default, a property
        binding — to fix an error you cannot explain. A read is free and leaves
        no trace. A settings change persists whether or not it helped, so an
        abandoned guess stays in the model as a change nobody intended. Change a
        setting only when you can name the broken thing and say why this fixes it.

    20d. **"Applied" means saved, not correct.** Some settings show their effect
    only where you are not looking.

        The aggregation function is the standard case. It acts when a coarser
        grain rolls finer cells up. So every working choice renders the same
        monthly table, and the first cell that can disagree is a quarter or a
        year — reading the months back settles nothing about the choice. Read a
        quarter back instead and check it shows what the intended verb would
        produce — the closing month for a LAST stock, the months' total for a
        SUM flow. A quarter showing the total instead of the closing month is
        a wrong setting; a quarter showing the formula re-run at quarter pace
        is a reverted one (references/limitations.md §1, §5).

        The same holds for a rollup override, and for a range-scoped rule under
        the regime you did not evaluate. Verify where the effect appears: read
        one quarter, one drilled row, the other regime. Or tell the user which
        view you did not check, because an unmentioned unchecked grain reads as a
        claim that it was fine.

### Speaking

21. To users: say variable (never metric, driver, or property as the entity
    name), dimension item, scenario (never layer), workspace (never org),
    formula range (never formula break), drill in (never drill down),
    snapshot. Never show UUIDs, layer ids, raw `runway:` URIs, or system-date
    internals. Users may say nest, pivot by, group by, or break down by; those
    all mean drill in.
22. Compute numbers with the calculation tools; never do model arithmetic in
    prose.

## References (load by task)

Load only the file matching the task below.

| When the task involves...                                                                                                                                                                                                                                                         | Read                                         |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| what variables, dimensions, items, and segments are; where data comes from                                                                                                                                                                                                        | `references/01-the-dimensional-universe.md`  |
| writing, editing, or explaining formulas; debugging a number                                                                                                                                                                                                                      | `references/02-formulas.md`                  |
| designing or editing a block; axes, nesting, drill-ins; the block signature; reading a large block window by window                                                                                                                                                               | `references/03-table-blocks.md`              |
| dates, granularity, actuals vs forecast, period comparisons                                                                                                                                                                                                                       | `references/04-time.md`                      |
| scenarios, snapshots, budget vs actuals, as-of                                                                                                                                                                                                                                    | `references/05-scenarios-and-comparisons.md` |
| a config that errors, renders blank, or shows misleading numbers                                                                                                                                                                                                                  | `references/06-validity.md`                  |
| the step-by-step procedure for any modeling request                                                                                                                                                                                                                               | `references/07-modeling-method.md`           |
| a concrete config shape to copy (with JSON)                                                                                                                                                                                                                                       | `references/08-recipes.md`                   |
| before writing formulas for a table carrying breakdowns; after adding a drill-in; which layer a formula belongs at; why a drilled row shows zeros; authoring order and consolidating hand-typed plans                                                                             | `references/09-the-layer-model.md`           |
| acting on a groupby/pivot/SQL analogy near aggregation, scenario overrides, overlapping formulas, offsets, or cross-grain totals                                                                                                                                                  | `references/10-deviations.md`                |
| declaring or reading table structure as a view; the shape the tools speak                                                                                                                                                                                                         | `references/11-block-grammar.md`             |
| changing an existing block — window narrows/widens, reslices, edit vs duplicate                                                                                                                                                                                                   | `references/12-editing-blocks.md`            |
| creating a variable that averages, counts, rates, or accumulates                                                                                                                                                                                                                  | `references/13-metric-recipes.md`            |
| dimension mappings: grouping one dimension's items under another; the table that shows one; formulas that read a mapped item; changing what a mapping is keyed by; removing rows or retiring one source's lookup (the value-axis mapping is `references/03-table-blocks.md` §3.6) | `references/14-dimension-mappings.md`        |
| a symptom that matches a known platform defect rather than your design                                                                                                                                                                                                            | `references/limitations.md`                  |

Name resolution: fall back to `resolve` `ask.grammar` only when the two dictionary reads
(`inspect_variables`, `inspect_dimensions`) cannot answer, and never
re-search a name they already answered. Absence
is not proof an entry is gone: a listing drops what you cannot read, and a
truncated one says `showing N of M`. The
`build-model` manual carries the full rule (the `resolve` `ask.grammar` fallbacks, how
dimension items page, and the grammar tokens each read hands back).

For build procedures and tool call sequences, follow the `build-model`
manual (this skill explains the laws those procedures rely on). The write
path sets which part applies:

| Write path                                                                                                         | Where the procedure lives                                                                        |
| ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| block builds and edits through a view (`view` on any `edit_model_views` write)                                     | this skill plus the tool schemas are enough                                                      |
| presentation edits on one named block — rename, column widths, hide/show, window, comparison (`edit_table_blocks`) | `references/12-editing-blocks.md`                                                                |
| ranked / top-N / who-moved questions (`inspect_model_views` `ask.rank`)                                            | a query, not a build: ask, don't build; nothing persists (`references/03-table-blocks.md` §3.8b) |
| formula writes (`edit_variables` operations, set_values, set_time_rollup)                                          | `build-model`                                                                                    |
