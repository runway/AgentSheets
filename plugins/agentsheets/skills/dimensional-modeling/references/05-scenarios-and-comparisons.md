# Scenarios and comparing worlds

This reference covers scenarios (branches of the whole model), snapshots,
and how blocks compare across them. Builds on the axioms in SKILL.md.

## 5.1 Scenarios are auto-rebasing branches

Internally a scenario is a _layer_. Every model entity (entry, formula,
block, page, setting) is resolved through the chain of layers from the
scenario up to Main. The resolution law:

> The closest scenario in the chain that has **any** version of an entity
> wins outright. A deletion wins the same way a version does.

Everything else follows from that one law:

- **Scenarios auto-rebase.** Edits on Main flow into scenarios continuously,
  _except_ the specific fields a scenario changed, which it keeps. Change a
  formula in your scenario and Main's later edit to that formula no longer
  reaches you. Everything you did not touch stays current. Propagation
  works per field, not per whole entity. For typed JSON fields like block
  settings it goes finer still: sub-keys merge individually.
- **A scenario can differ in anything**: formulas, entries, blocks,
  whole pages, even its Last close. A block created in a scenario is
  absent everywhere else, silently. In comparisons its missing rows render
  as blanks that look the same as empty values. In practice schemas stay
  Main-authored and scenarios carry formula and config changes, but
  nothing enforces that.
- **Source data is shared.** External tables are not scenario-scoped. A
  scenario can change formulas and configs. It never holds a private copy
  of the data.

**Main** is the live official model, the default target of every read and
write, and the one scenario that cannot be deleted or merged away. Speak of
it as "Main" or "Main Scenario"; call the others by their names, never by
layer ids.

## 5.2 Lifecycle

Scenarios are created as children of Main. Do the work in the scenario.
When it is ready, **merging applies the changes to Main and closes out the
scenario**. Closed out means deleted: nothing remains to sync later.
Scenarios that never land are simply deleted. Three rules an agent must
honor:

- Reading or comparing another scenario does not require switching into it.
- A write targets a scenario through the tools' top-level `scenario`
  parameter (name or id; a name matching several scenarios is refused
  with the ids). Omitted, the write lands in the scenario in view — so
  "update the budget" while Main is in view needs `scenario` set, not a
  switch.
- Switch or create a scenario only on explicit user intent, and announce it
  plainly ("Switching to Fundraise Options; changes I make will land
  there").

## 5.3 Snapshots

A **snapshot** is a scenario frozen at a point in the change history:
read-only, unmergeable, and kept until someone deletes it. Use snapshots for
board versions, plan freezes, "what we believed in January". You cannot
snapshot a snapshot.
Always call them snapshots to users, never "locked layers".

## 5.3b As-of reads

Related but distinct: the inspect reads (`inspect_variables` including
`ask.try_formulas`, `inspect_dimensions`, `inspect_model_views`,
`inspect_table_blocks`, `inspect_scenarios`, `inspect_history`) take
`as_of_point` — a `change_log_id`, a timestamp, a relative duration, or a
calendar boundary — and answer from the model as it stood then, creating
nothing. Reach for a snapshot when the frozen world must be durable and
shareable (a comparison column, a board version), not merely readable.

## 5.4 Scenario comparison on a block

A block can compare its baseline scenario against others. The comparison is
presentation: set it with `edit_table_blocks`, naming scenarios
(references/12-editing-blocks.md). An ad-hoc comparison question needs no
block at all; use `inspect_model_views` `ask.calculate` with compare_scenarios.
Variance is comparison minus baseline.

What you can rely on:

- Each scenario is calculated independently, then aligned **by row identity,
  never by position**. A row that exists in one scenario and not another gets
  nulls, not misaligned numbers.
- **On an axis:** a variable or dimension that does not exist in a comparison
  scenario is pruned from that scenario's calculation rather than failing the
  whole request.
- A read fails when a scenario named in a saved comparison has since been
  deleted. Re-issue the block's comparison on `edit_table_blocks`, listing
  only the surviving scenarios, and the read works again.
- The baseline is whatever scenario the viewer is in. The comparison list
  excludes it, and its column renders plain and unqualified.
- **For a row variable in the fan-out:** the comparison fans out over every
  row variable, and one with no definition in a comparison scenario, most
  often one created after a snapshot was frozen, has nothing to compare: its
  comparison and variance cells come back blank. Blank is the correct output
  there, not a broken block. Keep the single comparison table and let those
  cells render empty; do not split the newer variable into a second table.

When a request compares one thing against another, the first is the baseline
and the second is the comparison. "Current revenue against the January board
plan" keeps the live plan as the baseline and adds the January snapshot to
the comparison list. Do not build the block inside the snapshot or list the
live scenario as a comparison: if the baseline lands on the frozen scenario,
the plain column reads the old plan and every reader takes the frozen number
for the live one. A table built the wrong way round still shows a variance
of the right size with the two roles swapped, so the size of the number is
no proof the sides are right. Before you trust the rest, confirm the plain
column names the scenario you were asked to analyze.

Scenario comparison and time comparison are mutually exclusive on one block:
setting one clears the other. When someone wants "budget vs actuals over
time", the budget lives in a scenario or snapshot and the comparison is
scenario comparison. The time axis is just the table's ordinary date
columns.

One trap inside a comparison: each scenario carries its own Last close,
so the same period can be actuals on one side and forecast on the other —
the variance is then booked-versus-planned, not plan-versus-plan. Confirm
both scenarios' Last close before explaining a variance
(references/04-time.md §4.5).

## 5.5 Choosing the mechanism

| you want                                                                      | use                                                                     |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| a coherent alternative plan touching many things, maybe landing on Main later | scenario                                                                |
| an unsaved what-if on a formula, nothing persisted                            | `inspect_variables` `ask.try_formulas` (references/02-formulas.md §2.8) |
| this period vs a shifted period on one table                                  | time comparison (references/04-time.md §4.6)                            |
| this world vs that world on one table                                         | scenario comparison                                                     |
| a computed delta other formulas can reference                                 | a formula (references/02-formulas.md)                                   |

Use a scenario for a durable alternative world. Use an `ask.try_formulas` probe
for a temporary formula-level question that should not persist.
