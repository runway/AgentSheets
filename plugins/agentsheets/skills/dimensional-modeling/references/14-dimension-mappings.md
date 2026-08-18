# Dimension mappings

A mapping answers questions such as “Which Bucket contains each GL account?”
It is a lookup table owned by the target dimension. Define it once, then use it
in every table, formula, and breakdown.

Use a mapping when a classification should be reused: accounts into COGS,
customers into Enterprise, or states into EMEA. Without a mapping, the same
rule gets copied into conditions, formulas, and tables that can drift apart.
**If a classification will be used twice or outlive one answer, make it a mapping.**

Mappings cover any “derive one axis from another” rule: cleaning source items (three
spellings of one vendor becoming one clean item), consolidating one concept
scattered across imports (three sources' event columns all landing in one Event
dimension — §14.10), tiering (customers into Enterprise, Mid, SMB), rollups
(state to region to theater), hierarchies by chaining one mapping through
another (Account to Bucket to statement line).
The user may not say “mapping.” Recognize the relationship.

One neighbor is not a mapping: renaming. When an item's own
label is wrong — not that items need grouping — `edit_dimensions`
`change.rename_item` renames one user-created item everywhere, rewriting the
places that reference it. It takes user-created items only; an imported label
is the source's to change, and canonicalizing imported labels is exactly the
mapping above.

Do not create a mapping for a temporary grouping. Use a view or ranked read,
then create the mapping if the same classification is needed again.

Read this file when grouping one dimension under another, showing or editing a
grouping, or changing an existing mapping's keys.

## 14.1 The lookup, on paper

A mapping from Account to Bucket is this table, and nothing more:

| Account       | Bucket  |
| ------------- | ------- |
| 5001          | COGS    |
| 5002          | COGS    |
| 6001          | Payroll |
| anything else | Other   |

One mapping input has three parts:

- **Keys** — the dimension(s) looked up by; here `[Account]`. The distinct
  condition grains on the target dimension declare these inputs.
- **Rules** — the rows: one key coordinate, one item of the target dimension.
- **Catch-all** — the `anything else` row. Optional, but without it an
  account no rule matches maps to _nothing_: no Bucket at all, not a blank
  that still counts.

Under the hood the engine stores one conditioned formula per row on Bucket,
and the conditions themselves declare the mapping's keys.
`edit_dimension_mappings` writes those formulas for you. Think in the table.

## 14.2 Two keys, and which rule wins

Keys can be several dimensions. "The bucket depends on the account _and_ the
region" is this table:

| Account       | Region    | Bucket        |
| ------------- | --------- | ------------- |
| 7001          | US        | Domestic Opex |
| 7001          | any other | Intl Opex     |
| 5001          | any       | COGS          |
| anything else |           | Other         |

Rows of different precision coexisting is normal. Three laws make it
well-defined:

- **A rule pins some keys and leaves the rest open.** An open key means "every
  item, present and future": the `5001 / any` row is one rule covering every
  region. A key a rule does not name is open; naming it with `"*"` says the
  same thing explicitly.
- **The most specific rule wins where rules overlap.** `7001, US` beats
  `7001, any` exactly at US; the open rule covers every other region; the
  catch-all floors whatever no rule touched. This is the same contest ordinary
  formulas run (axiom 5), because rules _are_ formulas.
- **The lookup runs over key coordinates the data actually has.** If no source
  row carries the pair `7001, APAC`, that pair does not exist to be mapped — no
  rule is owed for it, and the mapping never manufactures the full
  Account × Region grid.

A dimension may have independent mapping inputs for genuinely different
imported vocabularies. For example, Unified Department can have one mapping
table keyed by Department 1 and another keyed by Department 2. An amount from
each import is classified on its own relation before the mapped cells combine.
Two mapping inputs that both apply to the same imported relation, with
neither more specific than the other, have no defined winner: use one
composite key or separate the relations rather than relying on order.

## 14.3 Is the request a mapping at all?

Three constructions answer "group these", and choosing is a question about the
knowledge rather than the wording: how far does the judgment reach, and how
long does it live?

**A mapping**, when it reaches beyond one view and outlives one answer.
"Bucket our GLs into COGS and Opex, then show payroll by bucket" — the second
half is evidence of reach, but reach is often silent: a grouping that three
formulas will each need has the same claim on structure whether or not anyone
says "everywhere".

**Row nesting**, when it is one view's arrangement. "Show accounts grouped
under region" is `[Region, Account]` on that table's breakdown; it says
nothing about the model.

**A ranked read**, when it dies with the answer. "Which accounts are biggest"
is `inspect_model_views` `ask.rank`; nothing is saved.

In an existing model, the signals that a mapping is already owed rather than
newly requested: the same item set repeated across conditions
(`Account in {"5001", "5002"}` in three formulas), an IF chain classifying a
dimension's items, a grouping maintained by hand in a table. Extracting the
mapping usually deletes more than it adds. When genuinely unsure, answer first
and build the structure on the second occurrence — the inverse law above.

## 14.4 Write a mapping

`edit_dimension_mappings` writes one lookup per call, and every call names the
lookup it addresses: the target plus the keys. It reaches no other lookup on
that target, so several sources each mapping into one dimension is a call each.

The single-key table from §14.1, verbatim:

```json
{
  "change": {
    "map": {
      "target_dimension": "Bucket",
      "keys": ["Account"],
      "items": { "5001": "COGS", "5002": "COGS", "6001": "Payroll" }
    }
  }
}
```

`items` is the one-key shorthand: source item to target item. The catch-all is
its own call, because one fallback serves every lookup into the target:
`{"change": {"catch_all": {"target_dimension": "Bucket", "item": "Other"}}}`.

The two-key table from §14.2 needs `rules`, one entry per row:

```json
{
  "change": {
    "map": {
      "target_dimension": "Bucket",
      "keys": ["Account", "Region"],
      "rules": [
        {
          "match": { "Account": "7001", "Region": "US" },
          "item": "Domestic Opex"
        },
        { "match": { "Account": "7001" }, "item": "Intl Opex" },
        { "match": { "Account": "5001" }, "item": "COGS" }
      ]
    }
  }
}
```

A `match` pins keys by item name, no formula quoting; a key it leaves out is
open (§14.2). For a term richer than "equals this item" — a set, a range, a
negation — a rule takes `condition` instead, in the same grammar as a formula
condition, brackets and all:
`{"condition": "$[Account in {\"8001\", \"8002\"}, Region in any]", "item": "Misc"}`.
One difference: a condition must name **every** key, opening one with
`in any` — condition text is stored as written, and a row naming fewer keys
would belong to a different lookup.

`map` writes and retargets; it never removes. The removals are their own
intents, so what a call deletes is what it named:

- `unmap` removes rows: `at` takes coordinates the way `match` does, and
  `conditions` takes the condition text the read returns for rows a coordinate
  cannot address. A coordinate that matches no row comes back under
  `not_found` rather than failing the call.
- `drop_mapping` retires one whole lookup — that source no longer maps —
  naming the keys so it can never mean the whole target.

What the write refuses, so refusals are predictable:

- a direct or transitive dependency cycle. Pick one canonical or reporting
  dimension as the target and map each source dimension into it. Never add the
  inverse lookup; if one already exists, remove it before writing the intended
  direction;
- a rule naming a dimension outside `keys` — a row cannot widen what the
  lookup is keyed by;
- a `condition` that leaves a key out (a `match` may omit keys — they are
  opened for you; a condition opens one with `in any`);
- keys that share some but not all of an existing lookup's keys: where both
  apply neither is more specific, so there is no defined winner;
- the target as its own key;
- a repeated key;
- a variable as the target;
- a mapped item that does not read as the target's data type — it would
  become a null member;
- a `map` given neither `items` nor `rules`;
- a target whose items already come from a set formula.

The overlap refusal is what a misremembered key set produces — the one shape
the engine leaves undefined. Nested keys are allowed: the richer rows win
where both apply, and the echo says so.

The last refusal is a boundary between mechanisms: a direct set-member
domain and condition-inferred mapping inputs are separate
dimension-definition mechanisms. A plain source-backed dimension is fine;
conditioned formulas may supplement or override its imported values.

The echo counts what the transaction did, never what the request asked for:

```json
{
  "success": true,
  "change": "map",
  "target_dimension": "Bucket",
  "keys": ["Account", "Region"],
  "rows_created": 3,
  "rows_updated": 0,
  "rows_removed": 0,
  "removed": [],
  "other_inputs": [{ "keys": ["Class"], "row_count": 22 }],
  "target_items": ["COGS", "Domestic Opex", "Intl Opex"],
  "catch_all": "Other",
  "tables": [{ "block_id": "…", "name": "Bucket mapping", "page_id": "…" }]
}
```

Read it before moving on. `rows_removed: 0` says the write removed nothing.
`other_inputs` is the target's other lookups, untouched — the check that a
write reached only its own. An empty `tables` means nothing shows this mapping
yet: the cue for §14.5.

## 14.5 The table users see

A mapping is formulas, and nobody reads formulas. The table is how a user sees
one — and it is a live view of the rules, not a copy, so it cannot drift and
there is nothing to keep in sync.

Build it with `edit_model_views` `change.configure_table`, omitting `table` so
the entry creates one rather than reconfiguring an existing block:

```json
{
  "view": { "variables": [{ "dimension": "Bucket" }], "breakdown": "[Account]" }
}
```

That renders §14.1's table live: accounts down the rows, each one's Bucket
beside it. The dimension sits on the value axis (the position a variable
normally occupies), the keys on the breakdown. A mapping-only view lays itself
out this way — transposed — without being asked; `transpose: false` overrules
it, and `change.transpose` on `edit_table_blocks` flips a block that already
exists.

The timeless, transposed shape is the mapping table's own, and it stops
there. A table that _uses_ the mapping — spend by bucket, ARR by tier — is an
ordinary report: the measured variable on the value axis, the mapped
dimension nested under it as child rows, and time across the columns as the
system Date with a real window (§14.10 step 4 states the same rule for
consolidations). Never carry the mapping table's keys-on-the-shared-breakdown
layout or its missing date axis onto a table of measured values — a mapping
demo is one timeless lookup table beside timeseries usage tables, not a page
of timeless tables.

A demo's seeded values must land at the dated grain the usage table
evaluates. Seed the measured variable per source item with `segments` plus
`period`, one item per regime the window spans. A per-item write
with no Date term (`$[Account = "5001"]`) stores at the dateless grain instead,
and every dated cell floors to the regime fallback — source sums in actuals,
zeros in forecast, no error anywhere (references/limitations.md §11).

Use one table per mapping input. Unified Department keyed by Department 1 and
Department 2 therefore has two tables, each showing the vocabulary it maps.
The write's `tables` field says which blocks already place the mapped dimension
on the value axis; reuse a block with the matching breakdown, and create the
missing input-specific block only when needed.

The table is also the verification. The echo reports what was _declared_; the
table shows what _resolves_ against the coordinates the data actually has, and
nothing knows those until the table calculates. A blank Bucket cell is an
account nobody bucketed: surface those and offer a `change.catch_all` or new
rules rather than reporting the mapping done.

To verify without creating a block, run the same `view` through
`inspect_model_views` `ask.calculate`: it computes the same table headlessly
and saves nothing.

## 14.6 Changing the keys

Widening one input from `[Account]` to `[Account, Region]` restates that input's
conditions at the composite grain — the keys are the conditions (§14.1),
and nothing else declares them.

It is two calls, and the order matters. **Write the new lookup first**, then
retire the old one:

1. Read the lookup you are widening. Its `pairs` paste straight back.
2. `map` at the new keys, carrying the old rows forward with the new key open:
   `5001 → COGS` becomes `{"match": {"Account": "5001"}, "item": "COGS"}`,
   where the unnamed Region key is open (`"*"` spells it explicitly). Then add
   the rows that motivated the widening, like
   `{"match": {"Account": "7001", "Region": "US"}, "item": "Domestic Opex"}`;
   they win over the open rows exactly where they pin more (§14.2). The echo
   warns that the new lookup nests inside the old one.
3. `drop_mapping` on `["Account"]` retires the old lookup. **Retiring is a
   choice, not a default:** nested lookups have a defined winner, so the
   narrow one may stay — it then catches imports that carry no Region, and
   acts as a per-account fallback where the wide table is silent. Drop it when
   every import carries the new key and misses should fall to the catch-all.

The order is what makes a crash between the calls harmless: both lookups
coexisting is a defined state, while dropping first would leave the source
mapping to nothing.

One behaviour to expect. A row storing `Region in any` applies where the data
carries a Region. An import with an Account column and no Region column matched
the old bare `[Account = "5001"]` row and does not match the widened one, so it
falls to the catch-all — keep the narrow lookup if that data still needs an
answer.

Update the same table's breakdown to `[Account, Region]`. Still one table.

## 14.7 Reading one back

`inspect_dimension_mappings` returns one entry per lookup. Single-key
equalities come back as `pairs` (`{"5001": "COGS"}`), which paste straight
into `change.map`. Everything else — sets, negation, `where`, every row of a
multi-key lookup — comes back as `rows`, each a `condition` plus an `item`;
`change.unmap` takes the condition verbatim. An empty answer means the
dimension has no mapping — its items come from its source or from added items
— and it is trustworthy, because generated defaults are filtered out.

The read carries `tables` too, the write echo's list. Rules present but
`tables` empty means the mapping works and nobody can see it: build §14.5's
table.

## 14.8 What to watch for

**A bare dimension column can acquire a mapping marker.** A top-level
dimension column with no `useAs` gets `VARIABLE` written onto it, which trips
the rule that values live on one axis. The view path does not do this, so a
block reaching that state came from UI-authored structure.

**Mapping cells compute but are not yet click-editable.** Users read the table;
changes still go through `edit_dimension_mappings`.

**A lookup probe that returns the catch-all for every key is reporting where
it asked, not what the mapping says.** Probe `Bucket[Account = "5001"]`
through `try_formulas` and the answer is "Other" — for every account,
including ones with rules. Neither the probe nor the mapping is broken. A
bracket read runs at the address of the cell that contains it, `try_formulas`
cells always carry a period, and the mapping's rules live at the bare key
coordinate — an Account and nothing else — so a read that also carries a date
matches none of them and falls through to the catch-all. That signature — the
catch-all, or one identical item, at every key — means the question was asked
somewhere the rules do not cover. It never means the rules are wrong: do not
rewrite or delete rules because of it. Verify with §14.5's table, its
headless read, or a hop probe (`this.Account.Bucket` with `by: ["Account"]`),
which looks each account up at its own coordinate and returns the right item.

**A variable whose values come from formulas, not imported rows, shows one
account's value per bucket instead of a sum.** The mechanism and the fix —
the reverse-lookup filter — are §14.9's.

## 14.9 Using a mapping in formulas

Once the mapping exists, formulas read it wherever they can look anything up.
With §14.1's table in place:

| You write                             | You get                                                                                             |
| ------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `this.Account.Bucket`                 | the cell's account looked up through the mapping — `COGS` at an account-5001 cell                   |
| `this.Bucket`                         | the current segment's bucket, where the segment carries the keys                                    |
| `sum(Amount[Bucket = "COGS"])`        | imported Amount summed over every account the mapping puts in COGS                                  |
| `Units[Account.Bucket = this.Bucket]` | on Bucket rows: Units summed over each bucket's accounts — right even with no imported rows (§14.9) |
| `if(this.Bucket = "COGS", Amount, 0)` | branch on the mapped item inside another formula                                                    |

The hop resolves through the mapping at the key's own grain — the bare key
coordinate, no date attached — so it is correct at any cell that carries the
key, even when a date rides along. A bracket read of the dimension
(`Bucket[Account = "5001"]`, `Bucket[Account = this.Account]`) is a different
read: it dispatches the rules at the cell's full address — date included —
where no rule matches, so it lands on the catch-all (§14.8). Spell lookups as
hops.

Each dot hop is a lookup keyed by the previous value, so hops keep going where
mappings chain: with `Super Bucket` keyed by `Bucket`, one more `.` walks the
second lookup. And a predicate on the mapped dimension runs the lookup in
reverse — the engine gathers the source rows whose keys map into the named
item, which is why `sum(Amount[Bucket = "COGS"])` needs no list of accounts.
That reverse form works when the variable's values come from imported rows
carrying the key. A variable whose values come only from formulas needs the
explicit reverse-lookup filter spelling from the table above instead. Take
two variables in §14.1's model. `Amount` is an imported ledger column: its
values arrive as transaction rows, each tagged with an Account. Broken down
by Bucket, the engine groups those rows by each account's bucket and adds
them up — the COGS row is the total of every COGS-mapped account's rows.
`Units` has no imported rows: its values are produced by formulas, say 80
at account 5001 and 200 at account 5002, with both accounts mapped to
COGS. With no rows to group, the engine fills the COGS cell from only one
of the two accounts — it shows 80, not 280, and raises no error. For a
variable like Units, write the rollup as the reverse-lookup filter
(`Units[Account.Bucket = this.Bucket]`), which sums every account the
mapping puts in the bucket.

A mapping row always maps to a fixed item. The engine can also hold a
computed form — a single expression at the key condition grain, deriving the
item from the key's own value:

```formula
Bucket[Account in any] = if(this.Account = "5001", "COGS", if(this.Account = "6001", "Payroll", "Other"))
```

Computed forms are read-only: no write tool authors one — a mapping row maps
to a fixed item, never an expression, and `change.set_values` writes variables
only. Reads still echo one from models that carry it. An if-chain like this
loses nothing to that limit — it is exactly two rows and a catch-all.

The one computation rows cannot spell is the key's own label. The
**same-name rule**

```formula
Event[Class in any] = this.Class
```

maps every label that already matches — present and future, with no items
read — and explicit rows override it exactly where a label differs, by
§14.2's specificity contest. Being an expression, it falls under the
read-only limit above: a model that already carries one keeps its
self-mapping behavior, and a consolidation (§14.10) authors the
equivalent by enumerating the matching labels as rows.

## 14.10 Consolidating several sources into one dimension

Several imports often describe the same real-world concept under different
vocabularies — an accounting system's classes, an HRIS's event names, a
warehouse's event column, all naming the same events. This is §14.2's
independent-inputs case, and its tell is the concept appearing once per
source; no request will say "mapping".

1. **Create the target dimension by hand.** Name it for the business concept
   (`Event`), never after any one source — the source dimensions keep their
   source-specific names — and never create a dimension under a name an
   existing dimension already carries. The mapping write registers every item
   its rules declare as a manual item in the same transaction, so pre-adding
   items is only for canonical items no rule produces yet.
2. **One mapping input per source.** Read the source's items and write one
   `map` call per source: an `items` pair per already-matching
   label (`"Renewal": "Renewal"`) and explicit pairs where a label differs.
   A fallback is a second call, `change.catch_all` — add one only when truly
   meant, since it hides unmatched keys. The same-name rule would cover
   matching labels unenumerated,
   but it is read-only (§14.9), so a label the source adds later maps
   nothing until the pairs are re-written; the mapping table's blank cells
   reveal any key still unmatched (§14.5).
3. **One table per input** (§14.5), so each vocabulary has its editable
   surface.
4. **Report by the target, on system time.** Break amounts down by the target
   dimension, and put time on the columns as the system Date — never a
   source's own date column.
5. **Formulas per §14.9** — amounts with imported rows group directly; a line
   computed by formulas takes the reverse-lookup filter.
