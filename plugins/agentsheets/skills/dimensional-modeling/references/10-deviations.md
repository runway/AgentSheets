# Where the engine deviates from the clean structure

SKILL.md's correspondence table says which outside folklore transfers into
this system. This file is the verified list of places it must not be trusted —
each found by pressing the clean mathematics against the engine's source and
adversarially re-verifying. Load it before acting on an analogy for
aggregation choice, formulas that might overlap, offsets, or totals reconciled
across grains. Each entry: the symptom you'd see, the
mechanism, and the decision it changes. Entries are numbered D1–D7, and other
references cite them by that number.

**D1 — Two formulas tie on one cell; the winner is whoever saved last.**
Precedence is (condition class, arity, pinned count, tightness, recency), and
time-window bounds add nothing — so a period-bound rule and a segment-bound
rule on one node (the same variable or dimension) tie, and any re-save flips
the overlap. Arity sits above pinned count: the two-term
`[A in any, Date in any]` outranks the one-term `[A = "x"]` even though the
latter pins more. Decision: never leave two overlapping same-node rules at
equal specificity; make one strictly more specific — pin an extra axis, or
add an `in any` term, a legitimate way to make a rule strictly more
specific — or merge them.

**D2 — An explicit rollup zeros out when the forecast starts.** A dated model
silently inlines the actuals window into any unlinked formula whose condition
carries a non-literal system-Date term — a rollup's condition always does — so
a hand-written rollup governs actuals only; in the forecast, the generated `0`
default outranks the synthetic rollup. Membership is exact: a Date-less subset
rule spans regimes, and a literal Date pin escapes the window; only the
non-literal Date term pulls it in. Decision: after writing a rollup, read back
a forecast-window coarse cell before trusting it; expect
authored-fine/derived-coarse to fail in forecast until fixed.

**D3 — A stock's quarter turns into a sum (or a blank) after a formula edit.**
Saving or deleting an `[]` formula re-derives the variable's
aggregation function from the expression's syntax: delete resets to `SUM`, a
recurrence derives `DO_NOT_AGGREGATE`. The save path guards itself: it
snapshots explicit settings before the save and restores any the save
re-derived away; only a failed restore warns, naming the reset. Deletes have
no such guard. Decision: after deleting an `[]` formula on a `LAST`/`FIRST`
variable, re-check and if needed re-assert its aggregation function; after a
save, act only when a warning names a failed restore.

**D4 — Monthly and quarterly views disagree about where actuals end.** The break
boundary rounds forward at month grain and backward at every coarser grain, so
one stored date closes July on the monthly view and re-forecasts it on the
quarterly one. Decision: author range boundaries on period edges of the org's
base grain; when reconciling across grains near the boundary, expect the
split to differ by up to one period.

**D5 — "Budget vs actuals, each vs last quarter" cannot be one block.** Scenario
and time comparison are separate stitched pipelines, not axes, and the engine
forbids combining them. Decision: plan two blocks (or a comparison plus a
formula-built delta column); do not spend calls trying to configure the
composition.

**D6 — Any offset means "one step of the current view's grain."** `X[-1]` steps a
month in a monthly table and a quarter in a quarterly one. An unfloored
recurrence is caught by a warning, but a floored ungrained one silently
diverges across grains. Decision: an offset recurrence must have its stepping
grain fixed, and the forecast-side spelling is one `change.set_values` item
with `period` — a period pins the formula's Date step to the model's base grain _and_
carries the window, exactly the pair a recurrence needs, which is why `grain`
and `period` cannot be set together. A `grain`-only item is silently
actuals-scoped (D2 hits any unwindowed term list). A recurrence that must
step at a grain coarser than base cannot be forecast-scoped in one item —
flag that instead of writing a bare grain item.

**D7 — A total row shows one branch of its children's `if()` instead of their sum.** A
rule addressed `[]` owns every level the blocks evaluate, including the collapsed
parent, which binds no item of the dimensions the expression tests. In periods with
no observed source rows every test is false and the cell takes the leftover branch;
in observed periods `this.<Dim>` can bind through the source relation — whichever
stored row comes first decides — so either arm can show. What never shows is the
children's sum: the rule replaces the aggregate that level would otherwise recompute. The `formulas` section attributing the
same formula to the parent and its children confirms it. Decision: per-segment
values are per-segment conditions, one rule per segment; keep dimension tests out
of globally-addressed expressions unless every level shown binds that dimension.
