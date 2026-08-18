---
name: build-model
description: Create or change model variables, dimensions, formulas, and table context. Use when building model logic, checking a formula, or testing a change before saving it.
---

# Build Model

<!-- standards:start -->

Prefer readable names, batch coherent model edits, and dry-run before writing. If neither the request nor the product pins down which scenario or which cells a write lands on, say which you chose.
Give a constant its own named variable when reasonable operators could pick different values or a user might adjust it to test a different outcome; a value the math or the domain fixes stays a literal.

Writing a formula places a rule at an address, inside a regime. Say both before you write: which
cells it addresses, and which time regime it belongs to. A rule naming dates with no regime reaches
actuals only, not every period. After the write, read the echoed address and regime as the fact of
what you stored, then read values back at the grain and period you care about before reporting any
number. An ephemeral test (`try_formulas`) runs as its own all-time rule, so it proves the
expression computes and never proves a saved rule dispatches.

<!-- standards:end -->

## Writing formulas

A value lives at named coordinates (`Payroll$[Department = "Eng", Date.Month = "2026-01"]`), and a
formula is a rule covering a region of those coordinates. The most specific matching rule owns each
cell; whatever it does not claim falls through to broader rules. So the shape of a healthy model is
always **one default plus scoped exceptions**: the default covers every cell, including dimension
values that do not exist yet, and specific rules override where the business differs.
`references/modeling-first-principles.md` builds this picture from zero; read it when a symptom makes no sense.

You state the intent; the tool spells and stores the rule. Every rule answers two questions, and
`edit_variables` `change.set_values` sets them independently.

**Where** it applies: `segments` names coordinates, `grain` pins a Date grain, `condition` takes
the whole address you write yourself. Leave all three out and the rule becomes the variable's
default, covering every cell.

**When** it applies: `period` names one of the model's own regimes — actuals, forecast, or a custom
range — never the dates a table happens to show. Never copy a requested horizon into it.

"When" has no neutral answer, and that is the trap. A rule carrying a Date term that matches more
than one date, but no `period`, is not an all-time rule: the engine confines it to actuals, so in an
all-forecast model it governs nothing. Its cells read zero, the write reports success, and nothing
errors. Only a rule with no Date term at all is genuinely all-time. So:

- write a projection with `period: "forecast"`;
- an all-time assumption with no Date term;
- and a within-regime rule with `condition` plus `period`.

A period merges its window into the Date term at the model's base grain and refuses any other
explicit grain. `grain` alone means "actuals only, at this grain".

`segments` and `condition` are two ways to say where, and they differ in who owns the address.
`segments` names coordinates and lets the tool finish the job: it completes missing dimensions from
the block you are looking at and reports what it filled in. `condition` is the whole condition in the grammar, and it completes nothing.

When you write `condition`, choose its reach by intent. `$[…]` claims a place — exactly this
grain, stopping when the grain changes — so use it for a deliberate cell pin or an address pasted
back from a read. `[…]` claims a shape and keeps applying wherever its named dimensions are broken
out, including added drill-ins. `[]` is the whole-model default; `$[]` is the empty segmentation.
A `segments`, `grain`, or `period` write always stores the exact `$[…]` spelling — `condition` is
the only door to the unsigiled subset shape.

Default to `segments` for cell and coordinate intents — its completion is what makes "make this
cell 5" work while looking at a table. A rule meant to survive drill-ins needs `condition`.
Reach for `condition` in three cases:

- **The term is not a coordinate.** `segments` maps a dimension to one item — `Dim = "item"`, or
  `Dim in any` through the open slot `{"Dim": "*"}`. Sets, ranges, negation, comparisons and
  `where` have no coordinate spelling. Passing any other text as a segment value does not error;
  it addresses an item whose name is that text.
- **The rule must survive drill-ins.** Write an unsigiled condition such as
  `[Department in any, Date.Month in any]` when the rule is about those dimensions wherever they
  are broken out, not only the grain currently in view.
- **The address must be exact.** When you are updating a rule you just read, paste back the
  condition the read echoed. Reads return `Variable[condition] = expression` and a write with
  that same condition updates that rule in place instead of creating a sibling beside it.

How coarse grains summarize the base grain is a different question — it takes a
method, not an expression — so it is `edit_variables` `change.set_time_rollup`.

The same picture runs the reads. `ask.try_formulas` narrows a hypothetical the same
way: `from`/`to` bounds time, `rows_by` picks the one dimension whose items get their own number.
Bounds are the one idea under every formula tool — decide where, then say what.

**Reads take as many ask sections as you need; most writes take one.** An `inspect_` call may fill
in several ask sections (the named sub-objects of `ask`, not table blocks): they are questions, so
asking two at once costs one round trip and the answers come back keyed by section name.

A write that changes different things takes one concern per call, because two changes in one call
have an order nobody set: on `edit_variables`, a variable operation and a value write are two
calls. The exception is `edit_table_blocks`, whose aspects are facets of one act on one table and
apply together — renaming a block and changing its window is one call, not two.

Decide whether the user wants a model change or just an answer. Use
[Saving formulas](#saving-formulas) for the former, [Evaluating without saving](#evaluating-without-saving)
(`inspect_variables` `ask.try_formulas`) for the latter. Both jobs start from the shared steps below.

For the underlying laws (segments, granularity, rollups, validity, and time), read
[[dimensional-modeling]]; you need not load the whole manual.

Read its references by symptom: [[dimensional-modeling:references/limitations.md]] and
[[dimensional-modeling:references/06-validity.md]] for blank cells;
[[dimensional-modeling:references/02-formulas.md]] and
[[dimensional-modeling:references/04-time.md]] for dispatch, totals, or periods;
[[dimensional-modeling:references/05-scenarios-and-comparisons.md]] for scenario overrides;
[[dimensional-modeling:references/03-table-blocks.md]] and
[[dimensional-modeling:references/11-block-grammar.md]] for table shape; and
[[dimensional-modeling:references/13-metric-recipes.md]] for a metric-specific recipe. Re-running a correct write will not fix a law.

## Shared steps

### Resolve variable and dimension references

Resolve every variable name the user mentions (target plus references like Revenue/COGS) from `inspect_variables`, and every dimension name from `inspect_dimensions`; the two reads are parallel-safe, so fire both in one round. Each entry binds the name to its entity (`id`, `type`) and carries `grammarRef` (the spelling to paste into _grammar_ — expressions, conditions, table grammar — with backticks and `#hex` where needed; name fields take the plain `name`), a variable's `dataType` and aggregation, and a variable's `slicesBy`.
If a sliced additive variable includes `totalFormula`, use that exact expression when the request needs its parent total. A bare parent can show zero while its child slices contain real values.

Reach for `resolve` `ask.grammar` when the dictionary cannot answer a name: absent from the listing, a listing that reports itself truncated (`showing N of M`), or an approximate spelling you cannot match by eye. A `resolve` `ask.grammar` hit for a variable or dimension carries its own `grammarRef` for grammar text, and its plain `name` for name fields, so it is ready to write with no read-back. A variable create, reuse, or rename echoes the same write-ready dictionary shape, so use its `grammarRef` directly rather than re-reading.

For a dimension item value (a filter or pin like `Department = "Engineering"`), page the dimension's items with `inspect_dimensions` (`ask: {items: {dimension, after}}`); reach for `resolve` `ask.grammar` only when a dimension is too large to page to the item you need, or the item is still absent after paging.

To **create** a value the source data does not carry yet (a new region "Canada", a planned department), call `edit_dimensions` with `change: {add_items: {dimension, values}}`. Do not look the value up first — a value that does not exist yet is the point. The items are model-wide: each shows on every table that slices the dimension, so no table block takes part, and `inspect_dimensions` reads them back in the dimension's item list, tagged `origin: "manual"`. Only a STRING dimension takes added values; a time axis widens by generating more periods (`time_defaults`, or the block's date range) rather than by adding an item. To pin an intersection of two or more dimensions onto one table instead — a single hand-added cell such as Engineering in NA — use `change: {pin_coordinates: {coordinates, table_block}}`, naming the table the way you read it.

**Wider context scan (only when needed).**
Beyond the name dictionary, reach for `inspect_pages` (`ask: {list: {...}}` to find pages, then `read` for a named one) or `inspect_model_views` when you need sibling meaning or valid segment values. To read the rules a variable is already defined by, ask `inspect_variables` for its saved formulas: `{"ask": {"saved_formulas": {"variables": ["Revenue"]}}}`, and omit `conditions` to get every formula on it. That is the saved read; `ask.try_formulas` is the separate hypothetical, and it calculates rather than lists.

### Write the expression

Use `references/grammar-reference.md` for the syntax itself: the grammar productions, entry-point
shapes, reference forms, operators and their precedence, and every function signature. It is
generated from the grammar and the function manifest, so it is the authority on spelling. Use
`references/formula-grammar.md` for what the forms mean and which to reach for when constructing
the formula expression.

When a formula on the system Date axis reads an imported variable with several source datetimes,
set the authoritative one as the variable's `preferred_time_property` (an `edit_variables` update)
and keep the reference relative — a `$` absolute or explicit source-date lookup bypasses the time
projection and can collapse or zero the monthly result. Full recipe: **Preferred time binding for
imported variables** in `references/formula-grammar.md`.

### Wire drivers into the model

A driver model is a dependency graph, not a materialized answer. Create or reuse the input
variables first, then make every derived variable reference those inputs. Do not create a driver
variable and then bypass it with hardcoded output values. When a stable formula can express the
relationship, do not expand its calculated result into one literal formula per month.

Forecast drivers that apply across the horizon are timeless `[]` constants, not values pinned to
one month. A month-pinned driver governs only that coordinate, so every other forecast period can
fall to zero even though the dependent formula is correct.

For a ratio whose denominator can be zero, define a separate total variable and wrap its explicit
total-over-total formula in `iferror`. Never rely on the ratio variable's recomputed parent: an
error at that grain leaves the parent unusable even when the child ratios are meaningful.

```formula
iferror(sum(Profit[Department in any]) / sum(Revenue[Department in any]), NULL)
```

Prefer a closed-form ramp when the value is a direct function of time. A recurrence can read only
cells inside the current calculation window, so moving the window past its seed can change the
answer or zero the series; the closed form returns the same value for the same period in any
window.

Until the grammar has a cumulative-sum primitive, build a cumulative balance from an event ledger
with Form 1 below: calculate one source-bound `Net Events` variable per period, use
`Cumulative Balance` as the stock and `Net Events` as its change driver, and put the single seed at
the first period of the supported window or formula range. Keep the event date, status, and amount
bound to the imported source. Set each coarser time grain to roll up with `LAST`, then verify the
seed period, a later period, and one coarser closing balance.

The no-literals rule also governs interactive builds. Transcribing source dates or amounts into an
`if()` chain is a failed build even when the displayed values match; unpivot and re-ingest the
source, or bind the formula to its imported variables instead.

A monthly roll-forward (cash balance, customer count, any running total) is computed as whatever it
was last period plus what changed this period. Before writing one, pick which of the three forms
below fits, from what the user actually said. Pick first; do not discover the form through failed attempts.

Form 1, the default. Use this unless form 2 or 3 clearly applies — an exact seed cell plus an
exact-grain chain:

``Customers[-1] + `Net New Customers` ``

Write both items with `condition` in the same `change.set_values` call: the seed at
`$[Date.Month = "2026-07"]`, using the table's first month and the starting value as its
expression, and the chain at `$[Date.Month in any]` — one chain item per regime the window spans,
same condition, `period: "actuals"` and `period: "forecast"`, because a Date-term rule with no
period governs actuals only. The chain reads the seed, and every later cell builds on the one
before it; a missing prior counts as zero, so an unseeded chain quietly starts at zero plus the
first change. The `$` is intentional: this recipe claims the unsegmented Date grain and stops when
another dimension is added. Only use the subset chain `[Date.Month in any]` when the user means a
separate chain in every segment; then seed each intended segment at its own starting coordinate or
deliberately let its missing prior start at zero.

Form 2: real history exists, and the formula should only project the future. The user has actuals
and wants the calculation to take over where the data ends. Write it with `change.set_values` with
`period: "forecast"` so it applies only to future periods; the last real data point becomes the
starting value on its own. Use a prior-period expression such as
``Customers[-1] + `Net New Customers` ``; do not wrap that prior read in `count()`. Keep the change
driver editable, even when the assumed value is zero. Pass `period` alone because it supplies the
Date grain; combining it with `grain` is invalid.

Form 3: the user ties the first value to an exact calendar date. For example "our opening balance
in January 2026 was 250,000". Only then is a date test like `if(Date.Month = "2026-01", …)` the
right start. Watch the distinction: "we start with 100 customers" names an amount, not a date, and
that is form 1. A date-pinned start computes correctly at its own grain; for quarterly or yearly
views add a `change.set_time_rollup` {method: "LAST"} anchor per grain, because a coarse cell is
otherwise built from its period's first month ([[dimensional-modeling:references/limitations.md#§5]]).

Write `Net New Customers` and `Revenue per Customer` as their own assumptions so a later change
flows through the customer and revenue forecast, and compute revenue at the same monthly grain with
``Customers * `Revenue per Customer` ``. Use the same pattern for beginning/ending balances, volume
growth, headcount, and other stock roll-forwards. Worked examples for all three forms live in
`references/formula-grammar.md`; its recurrence shape 1 also covers the `count()` guard, which
changes none of the laws. If the required upstream driver does not exist or
cannot be grounded, create or resolve that input first rather than encoding its implied outputs downstream.

Assumption constants hide most easily in tiered logic: nested ifs enumerating `<= 100, <= 200, …`
restate one assumption ("100 accounts per rep") once per branch, while
``ceiling(`Customers` / `Accounts per CSM`, 1)`` states it once, editable. Group the assumption
variables a build creates in one assumptions table on the model page, apart from the calculation
tables (the inputs section of an Excel model), so the user reads and tunes every lever in one place.

## Saving formulas

Read `references/saving-formulas.md` before you write: which phrasing maps to
which item shape, `segments` versus `condition`, every field the response returns, `dry_run` and
atomic-versus-partial batches, the boundary-read recipe, and the diagnose-fix-verify loop.

These decide whether a write is correct:

- **One call, never parallel.** Every formula the current build phase needs goes in one
  `change.set_values` `items` array.
- **Name only the dimensions the user named.** Do not pre-assemble the full cell shape, and do not
  check for an existing formula first. With no block and no explicit dimensions from tool output,
  prefer an unbounded default over guessing scope, and say so.
- **A `$` rule reaches only the exact dimension set its stored condition names** — that exactness is
  what the sigil declares; an unsigiled `[…]` instead reaches every segmentation containing its terms.
- **A scoped item states its time term itself**: `period` for the regime the values belong to, or
  `grain` for actuals only. The block never supplies it, so `segments` alone stores at a dateless
  grain no dated row shares, and every dated cell falls to the regime fallback — 0 in the forecast —
  with no error; with a `block` named, the same miss is refused loudly instead. In a from-scratch
  build, create the page and table before the scoped values so the writes have a block to anchor
  against (create both together with `edit_model_views` `change.configure_table`, not `edit_pages`).
- **Per-segment values are per-segment items**, never one global `if(Dimension = …)` expression: a
  global rule also owns the collapsed parent, which binds no item, so the parent shows the
  leftover branch instead of its rollup.
- **`applied: true` does not tell you where it landed.** A bound you did not mean to send is
  applied exactly as willingly as one you did, so read the response against what the user asked for.
- **A valid write is not a correct value.** Verify before anything depends on the numbers, and
  before you report one or call the build done. If a checked cell is blank or wrong, diagnose rather
  than resaving.
- **Do not enumerate the formulas in your response text.** The UI already renders each write, and
  each `inspect_variables` `ask.saved_formulas` result, as a card carrying the variable, its
  condition pills, and the expression. Send a one-sentence headline, any educated guesses you made
  so the user can correct them, and a clarifying question if you need one.

## Evaluating without saving

`inspect_variables` `ask.try_formulas` computes expressions that are not in the model, saving
nothing. The request is three decisions — the same three every time:

<!-- prettier-ignore -->
```json
{
  "ask": {
    "try_formulas": {
      "expressions": [{"name": "margin", "expression": "(Revenue - COGS) / Revenue"}],
      "from": "2026-01-01", "to": "2026-12-31",
      "rows_by": "Region"
    }
  }
}
```

- **`expressions`** is what to compute; every entry is calculated and comes back in the
  results. Reference one from another as `@name`.
- **`from`/`to`** is which periods. Periods are always the columns, at the model's base
  grain — one record per period, automatically. Forecast months exist only inside the range;
  actuals a bracket pins resolve even outside it.
- **`rows_by`** (optional) is one dimension: one record per item, per period. A cross of
  dimensions is a table — read it with `inspect_model_views` instead.

Each record carries `ephemeral_variable`, `segments` (keyed by the `rows_by` dimension, period included), and
`value`. Evaluation is itself validation — syntax errors fail the call with the problem, and
semantic issues come back as per-record errors. For the analysis shapes — windowed baselines
and ratios per item, watermarks, single-segment inspection, before/after `as_of_point` — load
`references/evaluation-cookbook.md` when composing the evaluation.

No cards render and nothing was saved, so never imply a formula was added to the model. If
the user wants to keep what you found, switch to [Saving formulas](#saving-formulas) — the
expression carries over, and the dimension you narrowed `rows_by` with becomes a bound of
the `change.set_values` item.

## Compiled headcount build compatibility

`headcount_build_tables` implements the roster and summary in [[headcount-planning]]. Use it until generic tools offer the same atomic write and readback.

Before calling it, resolve exact source variable and dimension IDs for:

- employee name, department, pay rate, start date, and termination date (required);
- job title, employment type/status, pay period/currency, location, and country (optional).

The build resolves the workspace Last close itself — you pass no id for it. Last close is the system Date dimension's actuals/forecast boundary, set with `edit_dimensions` `last_close`; the build reads whatever is set and fails with a clear message if it is unset. So setting the actuals cutoff (once, workspace-wide) is a precondition, not a per-build input — never `resolve` `ask.grammar` for a Last close id. Read the current value (and its `lastCloseId`) on `inspect_dimensions` (the Date entry's time settings) to decide whether it needs setting first.

If pay period exists, inspect its dimension items. Map known labels to months per pay period: annual 12, semiannual 6, quarterly 3, monthly 1, semimonthly 0.5, biweekly 0.4615, and weekly 0.2308. Omit unknown labels. With no mapping, omit the argument and disclose annual pay divided by the periods per year at the workspace grain (periods per year: 12 monthly, 52 weekly, 4 quarterly, 2 half-yearly, 1 yearly).

The created Payroll column divides pay rate by the mapped months: ``pay_rate / `Pay Period Months`$[`Pay Period` = this.`Pay Period`]``.
`Pay Period Months` is a hidden variable created from the mapping. The `$` keeps the lookup keyed only by Pay Period, so it does not inherit the employee row segment. Preserve that scope when repairing the formula or reproducing it through generic edits.

The call creates wrapper/calculated variables, formulas, a page, employee detail, and department summary in one operation. Pass optional internal property IDs only when resolved; never fabricate them. After the call, inspect:

- page and both table identities;
- created Payroll and Headcount variables;
- formula count;
- omitted optional columns;
- the resulting active employee and department totals.

Use generic model edits instead only when they can preserve the same idempotency, atomic behavior,
formula-break handling, and resulting-state readback. This is an obsoletion boundary, not a reason
to duplicate the compiled logic in the playbook.

---

## Important Rules

1. **Backticks are grammar, not spelling.** A tool field is one of two kinds, and the kind decides whether a name is quoted:

   | kind    | fields                                                                                               | how to write a name                                                               |
   | ------- | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
   | a name  | `variable`, `block`, `dimension`, `names`, `scenario`, `page`, and the keys and values of `segments` | exactly as you read it. `Total Assets`, no backticks                              |
   | grammar | `expression`, `condition`, `conditions`, and `breakdown` strings inside a `view`                     | backtick a name with a space so it parses as one identifier: `` `Total Assets` `` |

   In `change.set_values`, `expression` and `condition` are the grammar fields: the variable, the bounds and the segments are all data, and the tool assembles the address from them. So you never escape a name to say _which_ variable to write, only to reference one _inside_ a formula. Both spellings are accepted in a name field, so a name copied out of a formula still resolves; write the plain one. Backticks are the only part you drop — a repeated name still needs its `#a3f`, in a name field as much as in a formula, and the refusal hands you the prefix to use.

2. **Use readable variable and dimension names in formulas** -- write `Revenue`, `Revenue[Department = "Engineering"]`, `Revenue$`, or `` `Net Revenue` `` instead of property URIs. Plain top-level names and unprefixed bracket predicates inherit the current segment; the bracketed form overrides only the named dimensions. Write `$` after a reference's name (`Revenue$`) only when it must be absolute and must not inherit the current segment. If a name contains spaces, operators, punctuation, or a reserved word, wrap it in backticks. If two variables or dimensions share a name, use the minimal lowercase UUID-hex disambiguator from tool output, e.g. `Revenue#a3f`. For date granularity, append the keyword suffix to the reference, e.g. `Date.Month` or `` `Fiscal Date`#a3f.Quarter ``.
3. **Aggregation defaults to `sum()` for multi-value lookups** -- when a lookup can return multiple values (e.g., `[DIM in any]`, set membership, or `where` predicates that can return multiple rows), wrap it in an aggregation function such as `sum()`, `count()`, `average()`, `min()`, or `max()`; if you omit the wrapper, `sum()` is applied.
4. **Current segment inheritance is implicit** -- omit `this` in normal current segment references. Use `this.<Property>` inside nested expressions, such as `where` filters, when they must compare against the current cell's value.
5. **Conditions and expressions are separate** -- the condition determines _when_ a formula applies; the expression determines _what_ it computes. Don't conflate them.
6. **Use the scenario the user is working in** -- formulas are scenario-scoped. Omitting `scenario` writes to the scenario in view, which is almost always what you want. To write elsewhere, pass `scenario`: prefer the name the user says ("Budget 2026"), and pass the id when the user gives you one or when a name matches more than one scenario. A duplicate name is refused with the candidate ids, so pick the intended id from that list rather than guessing.

<!-- embedded-skill:project-management:start -->

## Embedded skill: project-management

# Project Management

<!-- standards:start -->

Keep proposals and plans short, specific, and accurate.

Before you propose work, inspect enough of the current state to name real sources and assumptions.
Do not change the model during this inspection.

Use `propose_plan` when the work changes model logic, creates several artifacts,
or belongs to a broader project. Build a single artifact from existing model
state directly.

Creating or changing model logic and then displaying it is a multi-step model build, even for one table. Unless the user explicitly says to build now, propose that combined build and wait for approval; the single-artifact exception applies only when the model state already exists.

The proposal records user consent. It is not a skill list. Read
[the proposal-card contract](references/proposal-card.md), call `propose_plan`,
and wait for approval.

After approval, call `write_plan` before execution with the full top-level task
list: mark the first item `in_progress` and the rest `pending`. Whenever a
top-level item finishes, call `write_plan` again in the same step as the next
item's first real tool call; never spend a step on `write_plan` alone. After
the final item's result exists, publish the all-completed plan before the final
response; only that terminal update may stand alone. Never defer or batch
multiple transitions into a later plan update. Complete an item only after its
result exists and has been read back when needed. Add discoveries without
rewriting settled work. Keep tool-level substeps out of the user-visible plan.

Approval settles every non-blocking choice recorded in the proposal; do not ask again. For an empty-org operating model, a manual-driver scaffold is a safe non-blocking default. Put any conservative starting driver values needed for a useful first draft in the proposal as clearly labeled chosen assumptions.
After approval, write exactly those approved values into named input-assumption variables and make each displayed model line formula-driven from those inputs or other lines. Never choose new business values after approval. Create the requested table and disclose which inputs were assumed. An unanswered blocking assumption remains a blocker.

Preserve proposal cardinality: each approved deliverable maps to one persisted artifact unless the proposal explicitly names more. Do not split one promised model into detail and summary blocks. For a page containing one table, create the page and table in one batch operation instead of creating the container and block separately.

<!-- standards:end -->

### On-demand files owned by embedded skill `project-management`

Read these when you need them; they ship alongside this skill:
- `references/proposal-card.md`

<!-- embedded-skill:project-management:end -->
