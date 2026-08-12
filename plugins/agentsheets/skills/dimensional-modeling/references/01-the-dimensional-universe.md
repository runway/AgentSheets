# The dimensional universe

What the space is, what lives in it, and where it comes from. Builds on the
axioms in SKILL.md.

## 1.1 The space

A segment is one point in the coordinate system, written as a set of
dimension = item pairs:

```
{Department: Engineering}                          a coarse segment
{Department: Engineering, Region: East}            a finer one
{Department: Engineering, Month: 2026-01}          time is a dimension too
{}                                                 the empty segment = the total
```

The **grain** of a computation is the set of dimensions in play. Asking for
Revenue at grain {Region} produces one value per region. Asking at grain
{Region, Month} produces one per region per month. The empty grain produces
the single total. Two properties of grains matter constantly:

- Grains are sets, not lists. {Region, Month} and {Month, Region} are the
  same grain. Row/column placement is presentation, not meaning.
- For time dimensions, granularity (monthly vs quarterly) is part of
  identity. Month-of-Date and Day-of-Date are _different dimensions_ with a
  fixed refinement order (day < week < month < quarter < half < year). This
  is what makes time rollups well-defined; rollup follows the view's single
  date-axis descent (SKILL.md axiom 13).

## 1.2 The dictionary: variables and dimensions

Every entry in the workspace's data dictionary is a **variable** or a
**dimension** (internally a "property" — you may see `property_id` in
output). Three kinds:

- **Variable**: a quantity to compute. Usually FLOAT. Carries display metadata (format such
  as currency or percentage, decimal places) and an **aggregation function**
  (`edit_variables` operations) with per-grain `set_time_rollup` overrides;
  references/04-time.md §4.4 owns the verb table, and SKILL.md axiom 7b
  says which verb fits which variable.
- **Dimension**: an axis of the space. Usually STRING; a DATETIME dimension
  is a time axis (which one carries the date-axis descent is SKILL.md
  axiom 13). Dimensions have items; variables do not.
- **Date ref** (formula-range reference): a named date the model can compute
  with, most importantly "Last close" (references/04-time.md).

A variable and a dimension are pure metadata. Neither stores values. Values
come from source data or from formula evaluation. That separation lets scenarios
fork the whole model without copying any data (references/05-scenarios-and-comparisons.md).

Kind is immutable after creation. An entry's _role in one table_ can be
overridden in one direction only. A dimension can sit on the value axis as a
**value-axis mapping** — an attribute column, distinct from the dimension
mappings of references/14-dimension-mappings.md: its item value fills cells,
aggregated by picking an arbitrary item. Useful for showing a text attribute
in a column. A variable can never act as a dimension.
A config that claims otherwise is silently corrected: variables have no
discrete items to group by.

## 1.3 Where entries come from

Entries are created three ways. The paths explain what you find in a
workspace:

1. **Created deliberately**, by a user or by ARI, with a name, kind, data
   type, format. Names of user-created entries are not unique: two variables
   called "Revenue" can coexist. That is why formula text supports
   disambiguators (references/02-formulas.md).
2. **Synced from source data.** When an integration query lands (accounting,
   HRIS, CRM, warehouse), every column of the resulting external table becomes
   an entry automatically. Numeric columns whose names do not end in "id"
   (case-insensitive suffix — `Paid` counts) become variables; everything
   else becomes dimensions. The kind is immutable after sync (no re-kind
   operation exists), so a numeric code column that should group — an
   account number, a zip code — has exactly one lever: alias it in the
   ingestion query's SQL so the name says what it is, then re-sync. Query
   edits live on the ingestion agent, reached with `delegate_agent`
   `change.ingestion`. These entries carry provenance (which query,
   which integration). Identity is (name, source query), so re-syncs are
   idempotent.
3. **System bootstrap.** Every workspace gets the system **Date**
   dimension, the **Last close** date ref, and the **Actuals**/**Forecast**
   formula ranges. System entries are universally readable, unrenamable,
   and undeletable. The Date dimension is special: its items are the
   union of every source table's date columns, kept in sync as integrations
   land. That is why one Date axis can slice data from every source at once.

## 1.4 Dimension items

A dimension's items are, by default, the distinct values present in its
source data (the MATCH behavior). Three ways items exist beyond the data:

- **The empty item.** Rows whose dimension value is missing form a real,
  addressable boxed item with typed URI identity `empty:None`, rendered "None".
  It sorts last. Treat it as the "uncategorized" bucket, not as an error.
- **Hand-added items**: values added by hand so a table or formula can address
  what has no data yet (a planned department, a future product). A value added
  to one dimension (`add_items`) is model-wide — it exists on every table that
  slices that dimension — and is listed back under `manual_items`. Pinning an
  intersection of several dimensions (`pin_coordinates`) is scoped to the one
  table block that shows it, and is not an item of any single dimension, so it
  is not listed among them.
- **Generated items** (the GENERATE behavior): an axis can be told to
  generate items over a configured range even where no data exists. Date
  axes do this implicitly across the table's date range. That is why
  forecast months exist as columns before any actuals land on them.

A dimension's `items` line on `inspect_dimensions` shows a count plus a few
sample spellings; to read every spelling of a large dimension, page it with
`inspect_dimensions` passing `ask: {items: {dimension, after}}`.

An unfiltered listing also appends `item_overlaps` facts — which dimensions
name the same things, bounded by `more_pairs_not_shown` and
`near_matches_capped` — the read for "are these two axes the same axis".

Check cardinality before fanning a dimension across a table. The `items`
count IS that check; never page a dimension just to count it, and a
50,000-item dimension belongs behind a filter, not fanned across columns
(references/07-modeling-method.md).

## 1.5 Source data and the "everything is a formula" bridge

Actual data lives in an OLAP store as external tables, one per ingestion
query, with columns mapped to entries. Formulas reach raw columns one
way only: an external-column reference, almost always wrapped in an
aggregate. The bridge law:

> A variable with no authored formula evaluates as the sum of its source
> column at the requested grain; a source-backed dimension evaluates as its
> column; a variable with no source and no formula evaluates as 0.

So a freshly synced "Amount" column behaves like a fully modeled variable
immediately: ask for it by Account and by Month and the engine groups and
sums the raw rows for you. The trap: a variable with neither source nor
formula shows confident zeros, not blanks or errors.

Scenario forks always share source data (external tables are not
scenario-scoped). Scenarios diverge through entries, formulas, and
configs only.

## 1.6 What "the model" is

There is no schema file and no separate modeling layer. The model is, per
scenario: the data dictionary, the formulas, the source tables, and the
pages of blocks that present them. Tables are both the presentation and the
place where modeling decisions (which variables, which breakdowns, which
formulas) become visible and editable.

To understand a workspace's model: read its dictionary (which variables and
dimensions exist, and from which source), then survey its blocks (the block
dictionary: which grains people actually look at), then read formulas for
the load-bearing variables. That order becomes a procedure in
references/07-modeling-method.md — one read each for the first two.
