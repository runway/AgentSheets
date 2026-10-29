---
name: import-model
description: Rebuilding someone else's model — a workbook, an export, a rendered document — as a native cfo.ai model that keeps its meaning. Use when asked to import, migrate, rebuild, or recreate a model from source files.
---

# Import model

An import rebuilds the business model that the source files represent as native variables,
dimensions, scenarios, formulas, and saved tables. The source describes relationships and outputs to
preserve; the rebuilt model is what makes them editable.

A ported formula preserves business meaning, not spreadsheet positions. The build-model skill's
generated formula reference is the authority for every accepted spelling, and its notes carry the
places the engine differs from Excel.

A clear import request authorizes the ordinary work within its scope. Without a narrower scope, you
cover every business-relevant section, period, scenario, statement, and view the source presents.
The source files stay as they are.

The build-model skill carries the formula and table mechanics and the statements and headcount
judgment; build-visualizations lends its judgment where a source calls for a visual.

## Which sheets a workbook presents

A workbook presents its visible sheets. A hidden sheet, very hidden ones included, is one its
author keeps out of view: a rebuild of the workbook covers the visible sheets and the hidden cells
they read, and the import's source map, its record of each sheet and where it went, names every
other hidden sheet as hidden. A request that names a hidden sheet brings it into scope.

## What the source decides

How much of a source carries over is a finding of the import, learned from the file. Before the file
is read, what you know is the method you will follow: survey the file, map its sections, rebuild
them natively, and reconcile each against the source. Which sheets the readers open whole, which
sections rebuild, and how closely each matches are what that work finds. The outcome depends on the
file: its size, which formulas have a native form, and which values come from macros or external
links. Each section's fidelity is established by its own reconciliation, and a large workbook is a
staged rebuild that runs well past one reply.

A person deciding whether to hand over their model decides on what you tell them, so your account
of an import tracks what you know at that moment: the method before the file arrives, the survey's
findings once it is read, and parity for the sections reconciliation has proven.

## The source's shape is the model's navigation

Each in-scope worksheet becomes a page with the same name, and each source-labeled financial section
becomes a table named after it, so a reader compares every source section with its saved
counterpart. A section's heading and cell range mark its review boundary, even when several sections
feed one calculation.

When several sheets are copies of one layout repeated per item, they are one page holding a shared
dimensioned variable, not one page per copy.

The whole used extent counts — the sections below and beside the first table too. A page with the
right tab name proves only that the page exists; each section needs its own disposition and
reconciliation before the worksheet is covered. Disclose any reader limit you cannot recover from.

A source map, in the plan or a small saved summary, records each sheet and range, what it is for,
its periods, and where it went: the pages and variables it became, or the data source that holds
it as a source table. A saved summary names the source file by its attachment id, such as
`runway-file-…`, which is how a later version of the same source finds it.

## Data, calculation, and presentation are different things

Transaction exports and report leaf rows are observations. They are what you import, at their real
grain, with stable identities, full parseable dates, currency and signs intact, and the row-level
business dimensions they carry.

Statement totals, copied calculation outputs, and derived subtotal columns are reconciliation
targets — the numbers the rebuilt model has to reproduce, not rows to ingest.

A parent row that carries postings keeps them. A source calling a row a header does not make its
activity disappear. Notes and scratch tallies belong in the model only when labels and formula
dependencies say they feed it.

Repeated row groups, per-item sheets, and recurring column bands are usually one variable segmented
by a business dimension. Recovering that dimension is the import; a separate variable per product,
department, or region is the spreadsheet's limitation copied in.

A classification that belongs to a source observation stays on the imported rows. One derived from
another dimension — statement bucket by account — is a mapping keyed by that dimension. Write and
check it through the model's own operations, so the mapping and everything that reads it are
visible in one place.

Case-selector cells, duplicated case sheets, plan-versus-actual bands, and mutually exclusive case
rows are scenarios. Shared variables and Dates stay shared, and you write only what differs by case
per scenario. A column labeled `FY26 Budget` can carry both a Date and a scenario. A scenario is never
an additive dimension. Inspect the existing scenarios before creating any, and leave the unrelated
ones alone.

Exact stored inputs and their dated changes, zeros included, carry over, and how each set lands
follows from how the calculation treats it. A typed input is a saved rule of its own at its cell's
coordinates, and the calculation evaluates every rule a variable holds, so a grid of typed values
is as many rules as it has cells. A source table is read as data: its rows are records that
variables read through their aggregation, one rule for all of them. A large set of fixed values,
such as a history sheet, a roster, or a list of transactions, anything past a few hundred values,
is therefore the rows of a source table. The workspace prepares it from the workbook as a CSV at
the records' real grain, with the sheet's dates as a column, and `edit_integrations` `ingest_csv`
loads it, inline when it is small and through its upload slot when it is large. The loaded table
reaches the model's variables once a saved query maps it into the model, as connect-data-sources
describes. A small set of tunable
assumptions, the drivers a person changes to move the plan, become named editable variables with
typed inputs. Computed outputs stay targets.

Distinct measured quantities keep separate columns — melting them into a name-and-value pair creates
the wrong entries. A source datetime column gets a business name, not `Date`. An annual observation
does not spread into months without an explicit allocation decision.

## Source time

The source has a closed period of its own, and reconciliation happens against it. When it differs
from the scenario's Last close, Last close can move to serve the rebuild; explain the effect. Never
extend Actuals across the horizon just to force parity.

An imported variable that shows zero after Last close has real forecast logic in the source to
rebuild, or the source's own timeline to follow.

A source observation beyond the accounting close can be genuine on its operational timeline. Keep
that distinction rather than relabeling projections as actuals.

Each imported measure has one authoritative source date, aligned on the shared Date axis.

## A faithful formula

A healthy source formula translates by keeping its operands, constants, method, date windows, and
dependencies, with spreadsheet positions replaced by named variables and dimensions.

One rule covers many months or items. How far it reaches comes from what the spreadsheet meant, not
from how many cells repeated it.

Lookup tables become mappings, selector logic becomes scenarios, and time windows become native
expressions. A running balance starts from its own actual historical value or an exact opening cell.
Quarterly or yearly stock figures are closing values, not sums.

A different forecasting method is a redesign, not an import, and never a silent substitution.

A source without formulas cannot prove its relationships. Rebuild the most defensible native logic,
make missing inputs editable assumptions, and claim none of it as source-formula parity. Read a
source formula before you claim to reproduce it. Source outputs stay parity evidence, not
dependencies of the new model.

## What reconciliation proves

Compare every section in the source map with the saved content, later rows and columns included.

Equal totals can hide offsetting errors, so reconciliation covers identities and values as well as
results, on both sides of meaningful formula boundaries, for actuals, forecast values, material
business slices, and scenario-specific results. A discrepancy says where to look next, and an
upstream change reopens what it touches.

A source error that healthy neighboring formulas expose can be corrected; report both the source
result and the corrected logic. Stale caches, dead external links, unsupported source claims, and
disclosed redesigns are each a different kind of mismatch from an unexplained one.

A driver test shows how a rebuilt relationship responds at its periods and dimensions. When the
relationships support extending past the source horizon, check that continuation too.

Partial work resumes from the saved model, the source dispositions, the mappings, and the open
discrepancies. Apply a proven correction to every earlier surface it affects.

When the rebuild runs as a task, do the survey and the decisions above in the conversation, and
put in the brief what they settled: the source map, the boundary and units, the scenario plan,
the mappings, and the target shape in the model's terms, which sets are dimensions and which
measures they segment. The brief is findings; what a step must do goes in that step's
instruction. The shared step builds only what every area reads: the Date axis, Last close, the
dimensions, the mappings, the scenarios, and the saved summary that holds the source map, and no
sheet's tables. One step per business area follows, and each depends on the shared step only.
Where an area needs another area's numbers, it loads the workbook's values as inputs and names
them in its report; a dependency between two areas runs them one at a time. Not every worksheet
is the same size. One that is far larger than the rest, in rows, in periods, or in formulas that
read other sheets, is too much for one step: give it one step per section of the sheet, each
depending on the one before it, so they write into the page named after the sheet one at a time.
Each step's criteria include reconciling its own sheets to the workbook, later rows and columns
included. The last step checks what crosses areas: it replaces the inputs loaded from other
areas with references, then confirms the relationships the source map links between areas, the
statement identities, the cash recurrence, and the scenario responses, not a second read of
every sheet. Each step names the sheets, sections, and formulas it works from and sets `verify:
true`: a rebuilt section is worth checking at any effort level.

## A newer version of the source

A connected spreadsheet arrives again as a new attachment each time it changes, and the version
the model was imported from stays attached beside it. Updating the import compares the two
versions section by section, guided by the source map. A changed section's tables, formulas, and
inputs follow the new version, and a section loaded as a source table reloads into the connection
that holds it. A new section gets a disposition like any other, and a removed
section leaves the model or stays for a named reason. Sections that did not change keep what the
model holds, including edits people made in cfo.ai. Reconciliation covers every changed section
against the new version, and the source map then names the new version's attachment id.

## What a finished import shows

Every in-scope section and its periods are imported, rebuilt, intentionally excluded, or blocked for
a named external reason. Unread ranges are incomplete.

The shared variables, business dimensions, visible axes, scenario identities, source-date bindings,
and mappings are present and verified. Imported values and forecast assumptions stay
distinguishable, and copied outputs never stand in for editable logic.

Saved statements and views reconcile to their supported source values, or list the exact
discrepancies that remain.

The handoff names what you imported, what you rebuilt, which assumptions are provisional, which
source defects you corrected, what is unmapped, and what is still open, and it keeps person-level
source privacy. Source analysis alone is not an import.
