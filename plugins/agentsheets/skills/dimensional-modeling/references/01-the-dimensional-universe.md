# Dimensional model basics

This file defines the model's core parts and where they come from. It builds on `SKILL.md`.

## 1.1 The space

A segment is one point in the model, written as dimension-item pairs:

```
{Department: Engineering}                          a coarse segment
{Department: Engineering, Region: East}            a finer one
{Department: Engineering, Month: 2026-01}          time is a dimension too
{}                                                 the empty segment = the total
```

The **grain** is the set of dimensions being calculated. Revenue at `{Region}`
gives one value per region. Revenue at `{Region, Month}` gives one per region
per month. The empty grain gives one total. Two rules matter:

- Grains are sets, not lists. {Region, Month} and {Month, Region} are the
  same grain. Row/column placement is presentation, not meaning.
- For time dimensions, granularity (monthly vs quarterly) is part of
  identity. Month-of-Date and Day-of-Date are _different dimensions_ with a
  fixed refinement order (day < week < month < quarter < half < year). This
  is what makes time rollups well-defined; rollup follows the view's single
  date-axis descent (SKILL.md axiom 13).

## 1.2 The dictionary: variables and dimensions

Every data-dictionary entry is a **variable** or **dimension**. Internally,
both are properties, so output may use `property_id`. There are three kinds:

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

Variables and dimensions are metadata; they do not store values. Values come
from source data or formulas. Scenarios can therefore branch without copying
data (references/05-scenarios-and-comparisons.md).

Kind is immutable after creation. An entry's _role in one table_ can be
overridden in one direction only. A dimension can sit on the value axis as a
**value-axis mapping** — an attribute column, distinct from the dimension
mappings of references/14-dimension-mappings.md: its item value fills cells,
aggregated by picking an arbitrary item. Useful for showing a text attribute
in a column. A variable can never act as a dimension.
A config that claims otherwise is silently corrected: variables have no
discrete items to group by.

## 1.3 Where entries come from

Entries come from three places:

1. **Created by a user or Ari**, with a name, kind, data type, and format.
   Names are not unique: two variables
   called "Revenue" can coexist. That is why formula text supports
   disambiguators (references/02-formulas.md).
2. **Synced from source data.** When an integration query loads (accounting,
   HRIS, CRM, warehouse), every column of the resulting external table becomes
   an entry automatically. Numeric columns whose names do not end in "id"
   (case-insensitive suffix — `Paid` counts) become variables; everything
   else becomes dimensions. The kind is immutable after sync (no re-kind
   operation exists), so a numeric code column that should group — an
   account number, a zip code — has exactly one lever: alias it in the
   ingestion query's SQL so the name says what it is, then re-sync. Ari
   cannot edit saved-query SQL, so name the exact alias change and hand it
   to the user to apply to the query; once they have, reach the re-sync
   through `delegate_agent` `change.ingestion`. These entries carry
   provenance (which query, which integration). Identity is (name, source
   query), so re-syncs are idempotent.
3. **System bootstrap.** Every workspace gets the system **Date**
   dimension, the **Last close** date ref, and the **Actuals**/**Forecast**
   formula ranges. System entries are universally readable, unrenamable,
   and undeletable. The Date dimension is special: its items are the
   union of every source table's date columns, kept in sync as integrations
   land. That is why one Date axis can slice data from every source at once.

## 1.4 Dimension items

By default, a dimension's items are the distinct source values (MATCH). Items
can also come from three places:

- **The empty item.** Rows whose dimension value is missing form a real,
  addressable boxed item with typed URI identity `empty:None`, rendered "None".
  It sorts last. Treat it as the "uncategorized" bucket, not as an error.
- **Hand-added items**: values added by hand so a table or formula can address
  what has no data yet (a planned department, a future product). A value added
  to one dimension (`add_items`) is model-wide — it exists on every table that
  slices that dimension — and reads back in the item list beside the source
  spellings, tagged `origin: "manual"` (or `"source+manual"` once the source
  starts carrying it too). Pinning an intersection of several dimensions
  (`pin_coordinates`) is scoped to the one table block that shows it, and is
  not an item of any single dimension, so it is not listed among them.
- **Generated items** (the GENERATE behavior): an axis can be told to
  generate items over a configured range even where no data exists. Date
  axes do this implicitly across the table's date range. That is why
  forecast months exist as columns before any actuals land on them.

`inspect_dimensions` shows an item count and a few samples. To read all items,
page with `ask: {items: {dimension, after}}`.

An unfiltered listing also appends `item_overlaps` facts — which dimensions
name the same things, bounded by `more_pairs_not_shown` and
`near_matches_capped` — the read for "are these two axes the same axis".

Check item count before spreading a dimension across a table. Do not page just
to count. Filter a 50,000-item dimension instead of spreading it across columns
(references/07-modeling-method.md).

## 1.5 How formulas reach source data

Source data lives in external OLAP tables, one per ingestion query. Their
columns map to entries. Formulas read raw columns through external-column
references, usually inside an aggregate. The rule is:

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

There is no separate schema file or modeling layer. In each scenario, the
model is its data dictionary, formulas, source tables, pages, and blocks.
Tables show and edit choices such as variables, breakdowns, and formulas.

To understand a model, read the dictionary, survey the blocks, then inspect
formulas for key variables. See references/07-modeling-method.md.
