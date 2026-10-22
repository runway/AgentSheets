---
name: build-visualizations
description: "What a page visual is for and how Canvas blocks connect to live model values: choosing the visual, the dataset contract, templates, and how a saved visual is checked and changed."
---

# Build visualizations

A visualization explains a relationship that is harder to see in prose or a table. It earns its
place one of two ways: an **explanatory** visual makes evidence easier to compare; a **simulation**
shows how a controllable assumption changes a modeled outcome.

The question and the relationships the model supports decide which one, whether interaction helps,
and which input is the meaningful one to move. The smallest clear, presentation-ready composition
wins: a person understands the answer, the current state, and the useful next interaction within
five seconds.

Before choosing a form, answer five questions internally: what relationship must become visible
(comparison, change, composition, distribution, correlation, or flow); how many variables matter;
whether the domain is ordered time or categories; how many marks must fit; and whether the person
needs an executive glance, analytical exploration, or operational monitoring. The request and the
model already hold this evidence. Executive views lead with a few outcomes and trends. Explorers
expose filters, cross-focus, and exact detail. Operational views prioritize exceptions, thresholds,
owners, and what needs attention now.

Also decide whether the block is communicating a known message or helping the person process a
population. A message view leads with one memorable conclusion and its proof. A processing view
supports scanning, sorting, comparison, and anomaly finding; it often starts with a compact table,
a semi-graphic table with sparklines or in-cell bars, or aligned small multiples rather than a row
of KPI cards. Match the form to the work.

Match the visual channel to the data. Position and length carry quantities and ordered comparison;
hue identifies categories, series, and states; luminance or a perceptually ordered scale carries
order when order itself is the meaning. A critical distinction has a second carrier besides color.
The strongest positional channels go to the most important variables. When one plot cannot keep
readable baselines and consistent scales, a table or aligned small multiples with shared scales is
the clearer form; several groups that need the same comparison keep one encoding across the set.

A simulation shows a baseline, one input the person can move, and the outcome that follows, side by
side, with whatever history makes the difference legible. Its initial state is the baseline;
adjustments carry labels and can reset. Inside a preview, history and the financial model stay
read-only: an editable mark changes a local future-period preview, and nothing saves unless the
block saves it with `emit`. Editable assumptions and recorded values live in native tables;
starting inputs come from dataset values or editable assumptions.

All new page visuals are Canvas blocks: sandboxed React components connected to live model values. A
requested dashboard or standalone HTML view is a page of Canvas blocks. Native tables belong to the
build-model manual.

Canvas is a product surface. Begin as an extension of the page, not as a card or website inside it.
The page and Canvas host already establish the outer boundary and breathing room, so keep the
component root transparent and unbordered, without a `Card` wrapper or duplicate outer padding. Add
layout directly as the content needs it. Add a surface, border, or padded container only when it
expresses a distinct group, state, elevation, or interactive boundary; do not wrap the block and
then wrap each section again by default.

A visual inherits the design system rather than restyling it, and the same business object keeps the
same color across every chart on a page. Place visuals where they read best; move an existing block
rather than recreating it.

## Templates and the component catalog

Component contracts and composition rules live in `inspect_canvas_library` and in the `source`
field description of the Canvas tools: which components exist, what each is for, how controls and
timelines carry local state, how navigation and chat intents are emitted, and which values come
from theme tokens. Read a component's entry when the visual leans on it.

Controls come from the injected design system: `Button` or `InlineButton` for actions, `TextInput`
or `TextArea` for text, `Select`, `SegmentedControl`, or `RadioGroup` for choices, `Checkbox` or
`Toggle` for on/off values, and `Slider` for ranges. A native input carries date and time values.
Local React components and native HTML/SVG compose layout and marks around the controls. A local
component lives at the top level of the source, outside `Block`, and takes its values as props; one
declared inside `Block` is a new component on every render, so React remounts it and its input
loses focus on each keystroke.

A control that opens an app destination emits `navigate` from a click handler, because the host
honors a navigation only after a recent browser-recognized interaction; an anchor's `href` would
navigate the isolated frame itself. The destination is a page's or block's `url` from a tool
result, a `runway:` reference that opens that page and reveals that block, or an app path in the
current workspace: `/<workspace-slug>/pages/<page-id>` with `?scenario=<scenario-id>` and
`#block-anchor-<block-id>`. A relative URL resolves against the app page, and a query-only URL
replaces the current query. Navigation follows the app's permissions and changes no model value.

Canvas templates are reusable building instructions for a business question, such as hiring or
runway. When templates are enabled, run a `list` with `kind: "template"` for the question before
designing, read a match with `read` along with the component contracts the visual needs, and adapt
its defaults to the person's explicit preferences. Selected template instructions and component
context arrive appended to the request. A selected template binds to real model datasets.

When templates are disabled, offer an ordinary chart or KPI. A gated template is a runtime binding,
so generic markup, SVG, or chart options cannot stand in for it.

Give each interactive control, and the value it changes, a stable `data-guide-anchor` id such as
`data-guide-anchor="pricing-slider"`. A saved walkthrough spotlights those ids; without them it can
point only at the whole block. Keep the id when the source changes.

A button that launches a saved guide asks Ari for it in a new chat. Set `ariAction` on the injected
`Button` so a non-icon button gets the standard trailing arrow and all button kinds get subtle Ari
feedback on hover, keyboard focus, and click. Use `ariAction` only for buttons that start Ari work;
do not hand-author another Ari icon or glow. The button calls
`emit({ type: 'startAriChat', message }, event)` synchronously in its
`onClick` and passes that click event; the app opens a new chat in the block's scenario and sends
the message. The message reads as the person's own request and names the guide by its saved name,
such as
`Present the saved guide "pricing-tour"`, so the reply presents it with `present_guide`. An `explain`
guide opens at once; a tour or walkthrough waits for the person to press Start in that chat. A
rendering check leaves the click itself untried, so the person's first click is its first run.

## The dataset and chart contract

A Canvas block reads model values through named datasets, each declaring the same visible shape as
a table: `layout.rows` and `layout.columns`. Ask the engine for the grain the visual needs rather
than thinning finer rows in code. Create only the model inputs or scenario the visual needs, and
report an inaccessible scenario rather than working around it.

Calculated values stay exact; you format only their presentation. Source currency, precision, units,
and the difference between percentage points and relative percent survive. An observed zero is zero;
missing or invalid data gets an honest local fallback. Favorable and unfavorable colors follow
business meaning, not sign.

A dataset inverts the table. Its value rows become `dataset.series`, and its columns (dates or
categories) become `dataset.rows`, one per point. A chart puts the variable, with its breakdown,
in `rows` and the shared time in `columns`.

A record list puts the fields in `rows` and the record's identity dimension in `columns`, with
`NO_GRANULARITY`. Each item in `dataset.rows` is then one record, with a value for each field.
A dimension shown as a value (`useAs: "VARIABLE"`) is read at the record, using the same
field semantics as a native record table in [[build-model]]. For subscriptions, customer, status,
product and created-at are fields beside Subscription ID. Nesting those attributes as breakdowns
asks for a hierarchy with intermediate totals as well as leaves; a record reader only needs the
fields beside each identity. Choose an identity at the source's actual grain: a subscription with
several items needs item-level rows when those items carry different values.

A field mapping must resolve for that identity and source. If the dimension maps a different
source, use an existing source-backed variable at the record grain, or create that variable
through the normal model tools. Check field values against source records before saving the visual.

The field's `dataset.series` entry names its `key` and `variable`; the matching row value is
`row[entry.key]`. The row's `date` is the record identity's display label for this layout. Date fields
are ISO dates, and datetime fields retain the complete UTC instant and microseconds. Those field
values carry the dates used for cohort, ordering and cutoff calculations.

Every declared dataset is calculated before `Block` receives `data`, including datasets read only
by dormant helpers. The block's datasets are the inputs its current visual and controls consume;
shared source code does not make unused declarations free.

A layout with time in `rows` and the variable in `columns` returns one series per period and no
`date` field. The tool's `dataset_evidence` shows it as `period_count: 0` with dates in
`series_names`. That is a layout to fix.

Each series carries `key`, `label`, `variable`, and `segments`. A series with empty `segments` is
the unsegmented variable; populated `segments` name breakdown members. The question decides which
level of a breakdown belongs in the visual. Stacking an additive total with the children it already
includes counts them twice.

```jsx
function Block({ data }) {
  const dataset = data.revenue || { rows: [], series: [] };
  if (dataset.error) return <Text variant="soft">{dataset.error}</Text>;

  return (
    <Chart
      options={{
        data: dataset.rows,
        series: dataset.series.map((entry) => ({
          type: "line",
          xKey: "date",
          yKey: entry.key,
          yName: entry.label,
        })),
      }}
    />
  );
}
```

Here `data.revenue` has Revenue on rows and monthly Date on columns with no business breakdown, so
every returned series belongs in the line chart. A product comparison would select the product level
from a dataset that includes that breakdown.

Line, bar, and area charts share this shape. Pie and donut reshape a chosen row into slices. A
waterfall takes ordered signed changes and explicit beginning or ending `totals`, not absolute
levels. Several line series share one plot only when their units and scale remain comparable, and
irregular dates take a real time axis. A dataset's Last close is the period, at the dataset's
grain, that contains the close date.

## Checking a saved visual

The write validates every declared layout before saving. Its `readback` shows the calculated grid,
or the failure, for each `data.<name>`. That grid is for checking values; its `c1`…`cN` column
names are not keys your code sees.

A save proves that the stored source compiles and that each dataset layout resolves against the
model. The readback grid shows the first rows of each dataset, blank and error cells included.
Rendering evidence comes from `validate: true` on the write or an independent
`validate_code_blocks` call, and belongs to the reported saved revision and to the frame width
its `observed` field names. The light render is also laid out at 360, 600, 760, 900, 1100, and
1400 px wide. A warning such as "At 760 and 900 px wide, text overlaps: td "Acme Ltd" at source
212:8 and td "INV-17" at source 213:8" names each width, element, and source position where text
runs into other text or spills out of its box. Fix it before handing the block over: stack columns
with Grid `minColumnWidth`, let text wrap, or give a wide table a deliberate horizontal scroll with
columns that keep their width. Grid already stacks a column whose content is wider than the column,
but it cannot shrink that content. Use that evidence for the changed block; its warnings can reveal a
wrong variable, breakdown, filter, window, or chart option, and they name the rows where a numeric
series in the whole dataset is blank or any series holds an error code such as `#DIV/0!`. The
block receives null in each of those cells. Its `plotted` list holds the values each chart was
given to draw: each numeric field of every visible series with its point count and its first,
last, lowest, and highest point, named by the chart's title and source position. Those are the
values as supplied. AG Charts draws stacked and normalized series, waterfall running totals, and
histogram bin counts from them, so the drawn shape of those series differs from the list. The check renders and reads; clicks, typing, links, and guide launches stay untested. An unavailable check leaves
rendering untested; a failed check leaves the edit saved.

Text diagnostics settle ordinary checks. Request a `visual_review` of one changed block when seeing
the settled result would change your next decision: a dense dashboard where clipping, overlap,
contrast, or hierarchy matters. It shows the 1200 px frame, so hover and interaction stay unproven, and other widths are
covered only by the width warnings.

Before saving, make one deliberate craft pass on the rendered composition: the eye's entry point,
scan path, grouping, numeric alignment, active and hover states, empty and error behavior, long
labels, narrow width, and overflow. Fix the few largest defects without broadly restyling the block.
The result still reads when color is removed, and every remaining mark or control carries
information or enables a real action.

## Changing an existing visual

The request sets the scope of a change. An element-selection chip identifies a block, an element, a
visible label, a sibling position, and often `@L<number>` in the current source. It can be the target
or an example for a broader request, such as applying its style across a page. A table `cell`, `row`,
or `column` selection refers to the presentation component's matching data.

Preserve block identities and leave unrelated work intact. `inspect_canvas_blocks` returns a block's
saved source, configuration, and reusable reference. The saved model table or its variables supply
the values for an answer. Focused changes are `source_patches`, one call holding every exact
replacement the change needs; a deliberate whole-component rewrite is a new `source`.
