# Saving formulas: the full procedure

The `build-model` body carries the rules that decide whether a write is correct. This file
carries the mechanics: which phrasing maps to which item shape, what each response field
means, and what to do when a verified cell comes back wrong. Read it when you are about to
write formulas, or when a write landed somewhere you did not intend.

## Step 1: Pick the tool from the intent

| User phrasing                                               | Item                                                                                                    |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| "Set Revenue to Price \* Quantity" (applies everywhere)     | `{variable: "Revenue", expression: "Price * Quantity"}`                                                 |
| "For New York, set Revenue to ..."                          | add `segments: {"Region": "New York"}`, and pass the table as `block`                                   |
| "For New York Engineering, ..."                             | one entry per scoping dimension in the same `segments` map                                              |
| "From April onward" / "for the forecast" / "in the actuals" | add `period: "forecast"`, or `{start, end}`                                                             |
| "For New York in the forecast"                              | `segments` and `period` together                                                                        |
| "Quarters should show the last month"                       | `set_time_rollup: {variable, granularity: "quarter", method: "LAST"}`                                   |
| "For each region ..."                                       | resolve the items, then one `items` entry per value in one call — never one `if(Region = …)` expression |
| "One rule at the region grain, future items included"       | `segments: {"Region": "*"}` — the open slot, a different address from leaving `Region` out              |
| "For each region, including richer drill-ins ..."           | `condition: "[Region in any, Date.Month in any]"` beside its `period` — the condition carries Date      |

Write every formula the current build phase needs in one `change.set_values` `items` array; a
separate call per item spends a model step for nothing. Never send write calls in parallel. An
item also takes an optional `grain` (`month`, `quarter`, and so on) when its expression steps
relative in time, such as a year-over-year ratio (`mrr / mrr[-12]`): pinning the grain keeps a
coarser view from stepping by the wrong period. A `period` pins the grain itself, so the two are
not combined.

Finding the block for a scoped write: a named table resolves via `inspect_pages` `ask: {list: ...}` ->
`ask: {read: ...}`; a segment mentioned without a table name resolves via `resolve` `ask.entities` with
`referenced_by_blocks`. With several plausible blocks, make an educated guess, proceed, and tell
the user which you chose. In a from-scratch build the block does not exist yet: create the page
and its table first, then write the scoped values with that block. Only a wholly unbounded
default is safe to write before its table exists.

## Step 2: Write at the address

Give `change.set_values` the target `block` and only the dimensions the user named. The tool completes
the address against the block (anchoring) and reports what it filled in under
`completed_from_block`, so a partial address cannot silently produce a formula that matches no
cells. Do not pre-assemble the full cell shape into `segments` yourself, and do not check for an
existing formula first: writing the same address twice updates in place rather than stacking
rivals. Anchoring can only finish an address that names at least one dimension, so an item bounded
only by `period` or `grain` has no block coordinates to complete. Every bounds write — `segments`,
`grain`, `period`, alone or together — stores the exact `$[…]` spelling, so such an item claims
the plain Date grain and stops at dimension drill-ins. The unsigiled `[…]` shape, the one rule
form that keeps applying under drill-ins, is authorable only through `condition`. That is one
stranding direction: time bounded, no dimensions, stranded on the dimensionless Date grain.

The stranding runs in both directions. With no block, an item bounded only by `segments` stores at that
dateless dimension grain — a grain no dated row shares — and on a dated table every cell of the
variable floors to the regime fallback (source sums in actuals, 0 in forecast) with no error
anywhere ([[dimensional-modeling:references/limitations.md#§11]]).

Folding the per-segment values into one globally-addressed `if(Dimension = …)` expression does
reach every level, but it also owns the collapsed parent, which binds no item: each test is false
there and the parent shows the leftover branch instead of the aggregate it would otherwise
recompute ([[dimensional-modeling:references/10-deviations.md#D7]]).

Write `condition` when coordinates cannot carry a term or when the sigil is part of the intent, and
give it every term including Date. Use `[…]` for a named shape that should survive richer grains and
`$[…]` for a place that should stop when the grain changes. Nothing is completed for a condition,
which is the point of choosing one. When pasting from a read, the field takes the bracketed half
alone: reads echo `Variable[condition] = expression`, and a pasted whole address is refused with a
steer to its bracketed half. Three combinations are refused outright: `condition` beside `segments`
(both say where; pass one), `condition` beside `grain` (write the Date term in the condition
instead), and `condition: "[]"` beside a `period` (`[]` names no Date term for the window to merge
into).

A written condition has four spellings, and the `$` is load-bearing: `[]` is the whole-model
default; `$[terms]` claims exactly the dimensions it names; `$[]` claims the empty segmentation;
and the unsigiled `[terms]` claims every segmentation containing the dimensions it names.
A `$` set with no Date term is the same stranding described above, reached by hand: the dateless
grain, read by no dated cell. And a Date term matching many months with no `period` is confined
to actuals. A per-item value that should appear under date columns is therefore `segments` plus
`period` — one item per regime when the window spans actuals and forecast — or a condition
carrying its own Date term beside a `period`. That Date term sits at the model's base grain
(`Date.Month in any` in a monthly model): a period merges its window into it, so any other
explicit grain is refused.

With no `block` and no explicit dimensions from tool output, prefer an unbounded default over
guessing scope, and say so.

`change.set_values` validates as it writes. `dry_run: true` checks every item and writes nothing.
`mode: "atomic"` rejects the whole batch if any item is invalid; prefer the default partial mode
for a large batch — valid items land and you resubmit only the rejects — and use atomic only when
the items must apply together or not at all. When rival rules already sit at one address,
`formula_id` on an item updates the one you name.

## Step 3: Read the response, not just the status

`applied: true` only says the write landed somewhere. Where it landed is the part worth checking,
because a bound you did not mean to send is applied exactly as willingly as one you did. Every
write answers in the readable grammar; the leading fields are one sentence each about where, and
one about when:

- `bounds` restates your own narrowing in words: `Region = West, within forecast`, or `everywhere`
  when you sent none. An authored `condition` is echoed verbatim as the first bound; the global
  `[]` narrows nothing and falls through to `everywhere`. Read this against what the user asked
  for. If it names a bound the user never mentioned, you filled in a field you did not want, and
  this is where you catch it.
- `completed_from_block` names terms the block supplied that you did not write. Anchoring adds
  these so a partial address still claims real cells. If one is wrong, correct it now, before
  anything is built on it.
- `written` is the stored rule, `Variable[condition] = expression`. It is the truth, so it can be
  wider than `bounds`: it carries the Date term a `period` needs and anything anchoring completed.
- `regime` answers when, on every item: the window's name when a `period` resolved, `every period`
  for a rule with no open Date term, or `Actuals only, because a rule matching many dates with no
period is scoped there`. It is the direct read-back for the silent-actuals trap — a projection
  whose regime echoes actuals-only governs no forecast month, whatever `applied` says.
- **The sigil-downgrade trap.** An update matches across the sigil: `[terms]` and `$[terms]`
  naming the same terms are one signature (only `[]` and `$[]` never unify), and the update keeps
  the STORED spelling. So authoring `condition: "[X in any, Date.Month in any]"` over a legacy
  `$[…]` rule silently stores `$[…]`, gains no drill-in coverage, and still reports
  `applied: true` — read `written`'s sigil, not just its terms.
- `uncovered_block_grains` lists grains the named block evaluates that no exact, subset, or default
  formula covers. One subset rule may cover several richer grains, so treat each entry as a coverage
  gap rather than instructions to copy the same formula per grain. Date-typed breakdowns (a cohort
  month, a contract-start month) are reported apart from this list: rollup methods are
  system-Date-only, so they owe no rollup rules. The write succeeded; this is work
  left. What an empty list promises: `07-modeling-method.md` step 6.

Then the bookkeeping: `window` is the window a `period` resolved to, by name. Dates reuse the window
whose bounds match exactly, and a `name` you send alongside them is ignored, because a formula write
never renames shared config. So `window` is where you find out that `{start, end}` landed in a
window called something else entirely. `created: false` means a rule already at that address was
updated rather than a second one added. A rejection names what owns the
intent you expressed; follow the pointer instead of retyping the same input.

## Step 4: Verify calculated state before a dependent build or a reported result

A valid write proves the formula was accepted, not that it computes the intended values. Verify
before anything downstream depends on them and before you report a number or call the build done —
with a small read, not the whole table: one formula fills many cells, so a wrong formula is wrong
everywhere it applies, and the user already sees every cell on the saved page. Check the cells
where a different rule takes over:

- where one named range ends and the next begins (for example, actuals then forecast);
- the first and last periods of the model;
- one rollup parent and one drilled segment, to prove aggregation and scoped rules.

Read those in one `inspect_model_views` call on the saved table or a minimal `view`:
`periods: "boundaries"` picks the period columns; pick rows that reach the parent and the
segment (depth or scope). If a checked cell is wrong, read wider there and fix it first.

If a checked cell is blank or zero, read once more in case it was still calculating, then stop
re-reading — saving the same formula again will not change it. Diagnose instead: the read already
names a blank or zeroed cell's rule and the rules it beat; correct the rule it
implicates — the winner's scope, or your rule's grain — and verify. Each pass must test new
evidence (no spelling variants, no enumerating dimension values in place of a working rollup),
within the owning playbook's diagnose-fix-verify limit. When evidence runs out or a blocker
appears, check the limitations and metric-recipe references in [[dimensional-modeling]], report the
exact unresolved problem, and leave the result visibly unreconciled with its plan item incomplete.
