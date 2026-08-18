# Valid and invalid blocks

This file explains what the system checks and how failures appear. **Saving is
lenient; calculation is strict.** A config can save and still fail later. It
builds on references/03-table-blocks.md.

## 6.1 Four validation stages

1. **UI construction** stops many bad shapes from being buildable. Nothing an
   agent does passes through this gate, so never assume it protected an edit.
2. **Write-time validation** checks structure (axis types on the right side,
   ids are UUIDs, no query params in property URIs, comparison fields
   well-formed, comparisons mutually exclusive) plus variable placement
   (§6.2 law 4). A `view` resolves every name before
   anything saves: a stale or hallucinated name is a check error first.
   `dry_run` runs the same checks without saving.

3. **Planning** (start of every calculation) enforces the semantic laws.
   Violations kill the whole request.
4. **Execution** contains formula failures: they become error cells (#ERR
   with a trace) and everything else keeps computing. Binding each
   dimension on the view to its source data runs before formulas do, so a
   binding failure is not contained. What such a failure costs is a defect
   rather than a law: it takes the whole request down
   (references/limitations.md §8, temporary).

How failure shows up, loudest to quietest: a rejected write whose error names
the offending field; a read that fails as a whole (planning could not use
the block); the whole-table error overlay ("Update the table config to
recover, or retry"); a per-cell #ERR; a silent blank; a silent _plausible
wrong number_. The last two are the dangerous ones. They motivate the
verify step in references/07-modeling-method.md.

Tool errors: "failed with HTTP 422: …" means the call was refused — what you
sent, or the model it names, which you may not have authored. "failed with
HTTP 424: …" means a service behind the tool failed instead — not your input,
so the same call may be worth repeating with a different breakdown, and
giving up on verifying is the wrong answer to it. The message after the
number carries the rest. When it quotes a second status ("calculation service
returned 500 (internal/unexpected): …"), the engine itself failed; a quoted
422 means it refused the model it was handed instead, and names what to
repair. The code in parentheses says which kind, and the text after it is
your best diagnostic. Read it carefully but literally: it speaks the
engine's internal vocabulary, its ids resolve through no tool you have, and
a name it prints (`Date`, `Amount`) may belong to a different entry than the
one you know by that name.

## 6.2 Required rules

Each item says the rule and what happens when it is broken.

1. **Axis side must match.** Row trees hold only row-typed axes, column trees
   only column-typed, recursively. Violation: rejected at save.
2. **Property URIs carry no query params.** A property URI names the entry
   bare — nothing rides on it as a query string. Violation: rejected at save.
   (Ad hoc evaluation configs may use short readable ids, which come back
   in row addresses, so results decode readably.)
3. **Every property axis names exactly one variable or dimension, non-empty.** The array
   field is single-valued in practice; "X by Y" is nesting, never two URIs on
   one node. Violations differ: an empty list passes save and kills
   calculation; extra URIs are silently ignored (the engine uses the first),
   which looks fine and means something you did not say.
4. **One variable per crossing, on one side only** (references/03-table-blocks.md §3.6). Violation:
   rejected at save on every door; the error names both variables.
5. **Zero variables is legal; formula-only means empty.** An empty config is a
   designed state. Do not mistake a lone formula axis for content.
6. **Freeform axes must never reach the engine.** The one live freeform (the
   comparison placeholder) is stripped before save and calc. Authoring one
   into a saved config passes save and kills calculation.
7. **Every referenced variable or dimension must exist in the target scenario.** For a
   saved block, a miss is treated as data corruption and fails loudly. For
   agent-authored ad hoc reads it is a clean rejection naming the stale or
   hallucinated id. Scenario-comparison calculations instead prune the
   missing axis from that scenario. A view write cannot create a miss: every
   name resolves before anything saves (§6.1). A saved block acquires one
   when what it names is deleted afterwards.
8. **One comparison kind per block.** Scenario and time comparison are
   mutually exclusive; enforced at save, at the tools, and in the engine.
   In `edit_table_blocks`, setting one comparison kind clears the other.
9. **Time comparison needs exactly one date axis** on one side with a real
   granularity (monthly vs quarterly), and an offset count of at least 1
   (references/04-time.md §4.6). `edit_table_blocks` refuses period
   comparison on a block with no date axis up front.
10. **Overrides reference a same-side sibling, acyclically.** Violation:
    rejected at save.
11. **Drill-in breadcrumbs must align with the ancestor chain.** Misalignment
    silently pins wrong items, so the editing helpers hard-fail on it. When
    authoring configs directly, recompute breadcrumbs after any move.
12. **Input drill-ins are single-scenario**, need the variable on rows, and
    cannot combine with any comparison. Violation: the read is rejected.
13. **Dimension items are exact intersections**: at least one dimension, values
    non-null, at most one wildcard term.
14. **Dimension formulas must stay dimensional.** A dimension's domain
    formula may reference only dimensions and source columns, acyclically; a
    cycle there is a hard planning error. (Variable formula cycles, by
    contrast, are legal structurally and only error per-cell when a true
    same-period cycle exists, references/02-formulas.md §2.5.)

## 6.3 Shapes that can exist but cannot calculate

States a saved block can already be in that fail at the next calculation.
Grammar cannot express any of them, so no agent write creates one: they
arrive from UI-authored structure, or from a later deletion elsewhere that
leaves an axis naming something gone. The states planning will kill:

- variable on both sides, or two variables on one path (§6.2 law 4 — every
  save door rejects it, so only a block saved before the check carries it)
- empty property list on a property axis (§6.2 law 3)
- a freeform axis in a saved config (§6.2 law 6)
- a deleted variable or dimension still referenced by an axis (§6.2 law 7)
- a deleted scenario still listed for comparison (skipped by the app;
  rejected if sent directly)

Do not reason a block into this list from a symptom — ask.
`inspect_table_blocks` `ask.problems` checks a saved block and names each
problem, advisory findings marked apart.

It never calculates. A clean answer clears the declaration, not the block —
a symptom that survives it is a calculation failure: §6.5, then §6.6. And
re-declaring the view is not the repair; it rewrites rows and columns and
nothing else. Fix what each problem names.

## 6.4 Valid shapes that can mislead

Some configs calculate but still mislead readers. Tools cannot catch most of these cases.

1. **Ratio or average broken down by a dimension.** Correct per-segment
   recompute means the parent row is _not_ the sum of children. That is
   usually right, and usually surprising. The dry run flags nested
   percentage/average variables by name and format heuristics only; a ratio
   named "Efficiency" is not flagged. When building such tables, say what the
   parent row means.
2. **Dimension as a mapping value.** Cells show one item of that dimension,
   picked arbitrarily. Fine for displaying a text attribute; nonsense for math.
3. **Same dimension on both axes.** Off-diagonal cells cannot exist and
   render blank; no error anywhere.
4. **Do-not-aggregate variables at coarse time granularities.** No time rollup
   is synthesized (references/04-time.md §4.4), so months backed only by a day-scoped formula fall
   through to the synthesized defaults: source sums or zeros that ignore the
   day-level formula, silently.
5. **Variable with no source and no formula shows zeros** everywhere
   (references/01-the-dimensional-universe.md §1.5). Zero is a value, not a warning.
6. **Stale inclusion filters.** Item filters are exact strings; a renamed or
   vanished item just drops its row.
7. **No forecast formula.** Future periods show 0 by default; "the forecast
   is zero" and "there is no forecast" look identical (references/04-time.md §4.5).
8. **Hidden axes still compute.** Hiding is cosmetic; cost and grain
   (each cell's dimension set) are unchanged.
9. **High-cardinality fan-outs.** Nothing caps item enumeration; a customer
   dimension with 50k items will try to render 50k rows. Check cardinality
   first: the item count in `inspect_dimensions`' listing IS that
   check. Never enumerate a dimension just to count it
   (references/07-modeling-method.md).
10. **A saved formula that no longer parses** is silently skipped and the
    variable falls back to defaults (references/02-formulas.md §2.6). Numbers change with no error
    anywhere.
11. **A condition's `$` prefix controls drill-in reach.** An exact `$[…]` override
    stops when adding a dimension changes the grain; a subset `[…]` override
    keeps applying wherever the named dimensions remain present. Use exact for
    a cell pin meant to stop and subset for a named-dimension rule meant to
    survive re-slicing. Coordinate writes through `segments` always store the
    exact `$[…]`; a subset claim needs the `condition` field. Read the echoed
    condition to see which claim was stored (references/02-formulas.md §2.2).

## 6.5 Reading a failure

- **Whole-table overlay, or a saved block's read failing as a whole**: a
  structural law broke at planning (variable placement, missing variable or dimension,
  freeform reached the engine). Fix the block; retrying changes nothing.
  `inspect_table_blocks` `ask.problems` names which law broke, on a saved
  block, without calculating (§6.3).
- **A rejected request naming a field**: the request said something
  invalid (comparison rules, time-comparison axis rules, bad IDs). The
  named field is the offender.
- **#ERR cells**: a formula problem. The trace names the originating formula
  and segment. Errors flow downstream through every dependent cell. Fix
  the origin, not the symptoms.
- **Blank cells with no error**: usually nothing applies there, not a bug:
  no formula in that regime (actuals vs forecast), an empty fan-out for a
  non-additive aggregate, an impossible cell (6.4.3), or a filter that
  matches nothing.
- **A correct parent row over drilled children that are all zero**: a layer
  is missing, not a number. The parent computes at its own grain; the
  children's grain has no matching formula, so they take the floor. Nothing
  errors and nothing is blank, which is why this survives every check above
  it. Compare the two levels before anything else — a variable that is
  really empty reads zero at both (references/09-the-layer-model.md §9.5).
- **Plausible but wrong numbers**: the 6.4 list. Recalculate what the parent
  rows should mean, and check the applicable formula regime before doubting
  the engine.

For the value-shaped cases — blank cells, zeroed children, plausible but
wrong numbers — which one a cell is in can be read rather than guessed.
The page already names the rule behind a quiet cell, or says that no rule
applied and the engine stood in (§6.6 step 2); `formula_origins: "all"` extends
that to cells that did come out with a value. Only these silent cases need that read; the loud modes
above self-report — an overlay or rejection names the broken law, and an
#ERR trace names its formula.

## 6.6 Find the rule that owns a value

After classifying a failure in §6.5, find the responsible rule in this order:

1. **Address the cell.** Read its grain off the grid (rows left of it,
   columns above it — references/09-the-layer-model.md §9.1) and its
   regime: does its period fall before or after Last close _at this
   block's granularity_ (references/04-time.md §4.5)? If the cell is a
   parent or total, jump to step 5 first — totals have their own rules,
   and that check is cheap.
2. **Ask the engine which rule filled it.** Re-read the cell with
   `inspect_model_views` `ask.calculate`; add `formula_origins: "all"` when the
   cell in question came out with a value.
   The page gains a `formulas` section naming the rule behind each cell and
   the order the engine tried that variable's rules in
   (references/03-table-blocks.md §3.10 works through an example). This is
   the owner reported, not inferred — the selection the engine actually
   made. Single-layer only: drop any time or scenario comparison from the
   read first.
3. **Read the order.** The engine stops at the first rule whose condition
   holds. So a rule listed above the one that won was tried and did not
   match, and a rule below it never ran at all. Two things to look for:

   - The rule you expected, sitting _below_ the winner. It is being
     shadowed. Fix the winner — its condition, or the range it is scoped to
     — not the rule you wrote.
   - The rule you expected, absent from the list entirely. Usually that
     means it was never a candidate here: an exact rule's dimensions do not
     equal this cell's grain, or a subset rule names a dimension the grain
     does not contain (§6.4.11; references/09-the-layer-model.md §9.1) —
     written with `segments` and no block or date term, that is
     references/limitations.md §11, and its Do bullet is the remedy. One
     exception: the list is capped (§3.10 — the winner plus eight
     runners-up), so when it runs that long, a rule can be missing because
     it ranked past the end. Confirm with `inspect_variables`
     `ask.saved_formulas` before rewriting anything.

   A line carrying no formula text means no rule applied and the engine
   stood in, and it names the §6.5 case: "engine filled 0" is the confident
   zero — write the missing layer (see Exits); "rolled up from finer cells"
   is a parent computing from its children (step 5); "no formula and no
   fallback covers these cells" is §6.5's blank-with-no-error — nothing can
   fill the cell as addressed.

4. **Derive the owner by hand when the engine cannot report it** — a
   comparison read, or a page whose note says origins are unavailable.
   - _List._ `inspect_variables` `ask.saved_formulas` returns every stored
     rule as a `Variable[condition] = expression` line carrying `range`
     (Actuals, Forecast, or a custom range; empty means unwindowed) and
     `updatedAt`.
   - _Strike._ Remove an exact `$[…]` rule unless its dimensions equal the
     cell's grain; remove a subset `[…]` rule unless all its dimensions occur
     in the cell's grain. What survives: matching rules whose items cover the cell, the
     `[]` default, and the engine's generated floor beneath
     everything, which never appears in the listing
     (references/09-the-layer-model.md §9.2, layer 0).
   - _Pick._ The most specific survivor in the cell's regime wins. Regime
     membership: an Actuals-ranged rule never owns a forecast cell, and an
     unwindowed rule whose system-Date term is non-literal is implicitly
     Actuals-scoped — with two escapes, a Date-less term list spans every
     regime and a literal Date pin escapes the window
     (references/10-deviations.md D2). In a scenario, the returned rules
     already reflect that scenario's layer resolution. The tiebreak: two
     same-grain rules overlapping at equal specificity are owned by
     whichever `updatedAt` is latest — suspect that before anything
     subtler.
5. **Check the carrying rule.** For a parent or total, read the
   variable's aggregation function and the parent-row laws
   (references/13-metric-recipes.md §13.3-§13.4) before _editing_ any
   formula.
6. **Confirm the number.** Compare the owner's expression against what it
   should produce at that address. Naming the owner settles which rule ran;
   it does not settle that the rule is right.

Exits:

- **No candidate at the cell's grain** — write the missing layer via
  `change.set_values` (references/09-the-layer-model.md §9.3), never patch
  cells one by one.
- **An owner that is right with a number that is wrong** — trace its
  inputs (each is a new cell; recurse).
- **The engine appearing to contradict a law** — references/limitations.md
  before doubting the laws: all zeros on a recurrence is the signature of
  limitations §10 (an out-of-window seed), and a reverted aggregation
  setting after a sync is §1 — not §6.5's "nothing applies here."
