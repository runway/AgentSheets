---
name: build-visualizations
description: Build model-connected Canvas visualizations and interactive applications. Use when choosing a business representation, making timing or other choices directly manipulable, binding datasets, or reviewing the rendered result.
---

# Build visualizations

A Canvas application is a working representation of a business decision. The financial model
describes what happens; the application gives the person a useful way to act on that model and
understand the consequences. An explanatory visual makes evidence easier to compare. A simulation
makes a business choice directly manipulable.

The design bar combines Tufte's clarity of quantitative evidence with Bret Victor's immediacy of
action and consequence. The shared product theme supplies the visual language.

Its form comes from the objects and relationships in the business. A hire begins at a date and
creates recurring cost and capacity. A contract has a service period, billing terms, and
collections. A scarce resource is divided between competing uses. The model determines which
quantities are choices, which are consequences, and what connects them.

Timing is often a particularly revealing choice. Moving a consequential event lets the person see
how a familiar scheduling decision changes several parts of the forecast together. When timing
matters, give the event a visible place in time and make that position adjustable. Its dependencies
retain their business meaning: moving a collection changes cash, while moving a service start can
change when revenue is earned. A point changes date; a span may move while preserving duration or
have an editable end when duration is itself a choice.

The business object is also its control. A date is the position of an event; an allocation is a
boundary between uses; an editable amount belongs to the quantity it represents. The gesture
follows that meaning. Exact entry and keyboard operation express the same choice with precision.

The application is an annotated figure the person can manipulate. Its marks are the business
objects, its geometry explains their relationships, and its annotations give the current answer.
The first useful action and its financial consequence occupy the opening view together. A cash
minimum is labelled at the low point; a changed event carries its date and difference; selecting
that event opens precise editing beside it. Overview, action, comparison, and detail have a common
visual home.

In a time-based figure, the same horizontal position means the same period. Event marks sit on or
beside the financial paths they affect. The person can hold an event, move it, and watch the
response in that picture. The saved position and original path remain visible. Constraints,
plateaus, and delays appear where they affect the choice. Related marks retain their identity
across views. This relationship stays intact as the width changes.

The question determines the information form. A known conclusion needs its convincing proof.
Exploring a population benefits from comparison, focus, and exact detail; tables with embedded
marks and aligned small multiples can make dense evidence readable. Position and length carry
quantities, hue carries identity, and an ordered lightness scale carries magnitude. Comparable
views retain consistent units, scales, and periods. Labels belong to the marks they explain.
Each important distinction has a readable cue in its position, shape, line style, or label as well
as color.

A new preview starts from the saved baseline. An existing preview retains the person's choices and
identifies its changes. Model data supplies the opening values and editable assumptions. Local
JavaScript can calculate a preview from those relationships; unchanged inputs reproduce the saved
result, historical periods stay fixed, and dependent future values respond at their modeled grain.
A gesture changes local preview state. `emit` can persist block-local choices or request an explicit
Ari action; the financial model stays unchanged by the gesture. Reset returns to the current
model-supplied baseline.

All new page visuals are Canvas blocks: sandboxed React components connected to live model values.
A requested dashboard or application can be composed from Canvas blocks. Native tables belong to
[[build-model]], and remain the inspectable basis of the application.

At a narrow width, responsive geometry, focus on a selected object, or readable bounded scrolling
keeps the useful choice and its immediate consequence together. Customer-facing copy describes the
business and actionable data conditions; authoring checks belong in the chat handoff. A quiet
caption identifies a local preview's scope. The host supplies the boundary and inset, so begin
with a transparent, unbordered field. Containers express real groups, states, or interactive
boundaries. The shared design system supplies coherence; the business supplies the distinctive
composition. Move an existing block rather than recreating it.

## Templates and the component catalog

Component contracts and composition rules live in `inspect_canvas_library` and in the `source`
field description of the Canvas tools: which components exist, what each is for, how controls and
timelines carry local state, how navigation and chat intents are emitted, and which values come
from theme tokens. Read a component's entry when the visual leans on it.

Injected components provide reusable interactions: `Timeline` or `DecisionTimeline` for row-based schedules,
`EditableChart` for editable future points or bar endpoints, `BusinessDriver` for an amount,
`EditableAllocation` for a fixed-total boundary, and `InteractiveFunnel` for conversion. Their
catalog entries define the gesture and callback contracts. `Chart` supports comparison and mark
selection; a read-only outcome remains an outcome.

A coordinated application can also compose its own HTML/SVG marks and interaction handlers when
one geometry must own both the adjustable objects and their consequences. One time scale can place
events, cash paths, reference lines, and the active date guide in the same picture. Reuse injected
components where their form fits, and use theme values and source-aware formatting throughout the
custom representation. The application is free to choose the form that makes its business clear.

An SVG viewBox scales text and pointer targets along with geometry. Derive geometry from the
available container width, or preserve a readable scale within bounded scrolling. Labels and hit
regions retain their useful physical size. The event's hit region owns its drag; the surrounding
figure retains normal touch scrolling. A small visible mark can have a larger interaction target.

Conventional controls fit choices without a useful spatial representation or provide precise
alternatives within that representation: `Button` and `InlineButton` for actions, `TextInput` and
`TextArea` for text, `Select`, `SegmentedControl`, and `RadioGroup` for choices, `Checkbox` and
`Toggle` for on/off values, and `Slider` for abstract ranges. A native input carries date and time
values. Local React components and native HTML/SVG compose layout and specialized marks around
these controls. A local component lives at the top level of the source, outside `Block`, and takes
its values as props; one declared inside `Block` remounts on each render and loses input focus.

A control that opens an app destination emits `navigate` from a click handler, because the host
honors a navigation only after a recent browser-recognized interaction; an anchor's `href` would
navigate the isolated frame itself. The destination is a page's or block's `url` from a tool
result, a `runway:` reference that opens that page and reveals that block, or an app path in the
current workspace: `/<workspace-slug>/pages/<page-id>` with `?scenario=<scenario-id>` and
`#block-anchor-<block-id>`. A relative URL resolves against the app page, and a query-only URL
replaces the current query. Navigation follows the app's permissions and changes no model value.

Canvas templates are business briefs for a question, such as hiring or runway. When templates are enabled, run a `list` with `kind: "template"` for the question before
designing, read a match with `read` along with the component contracts the visual needs, and adapt
its defaults to the person's explicit preferences. Selected template instructions and component
context arrive appended to the request. A selected template binds to real model datasets.

When templates are disabled, offer an ordinary chart or KPI. A gated template is a runtime binding,
so generic markup, SVG, or chart options cannot stand in for it.

Give each interactive control, and the value it changes, a stable `data-guide-anchor` id such as
`data-guide-anchor="hire-start"`. A saved walkthrough spotlights those ids; without them it can
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

## Aesthetic design pass

A working first render is a design draft. Before handing over an interactive application, give it
a separate aesthetic pass: typography and typesetting, visual hierarchy, colors, balance, and
especially composition. Rendering checks establish execution and layout facts; this pass judges
whether the picture makes the business decision tangible.

Request a `visual_review` with `view: "desktop-opening"` and inspect the rendered application as a
designer. This is a fixed Canvas area visible before scrolling. Its independent assessment is a fresh
reader's judgment of the visible evidence; concrete gaps between that reading and the intended
experience are design work. Keep the same opening through revisions so the comparison tests the
composition. A measured `viewport` explores a different available area;
the full image shows secondary content below the opening. `framing` reports what the image actually
shows. Choose a deliberate
composition for its particular objects and relationships. The opening view puts the meaningful
action and its financial consequence in one working picture. Controls belong on the things they
change; exact entry belongs at the selected object; current values and comparisons belong at the
marks they explain. Use the available space to make that relationship immediately understandable
on both desktop and a narrow screen. The person's attention can focus on one object while the
rest retain enough context to choose another.

The existing data and calculation behavior are the basis of this pass. Preserve baseline
comparison, reset, keyboard and exact entry, and model inspection. Keep the shared type and theme
tokens; improve their use through sizing, weight, alignment, line breaks, contrast, and restraint.
Revise the composition's largest weakness, then inspect the rendered result again. The design
critique belongs in the chat handoff; the application itself explains the business.

The block image does not show the surrounding page, and a static render does not exercise a gesture. Use
available interaction evidence to check the changed date or amount, its dependent values, reset,
and the path into the model. Describe untested behavior accurately. The saved page's reading order
must give the working application its strongest position rather than delaying it with repeated
explanation.

## Changing an existing visual

The request sets the scope of a change. An element-selection chip identifies a block, an element, a
visible label, a sibling position, and often `@L<number>` in the current source. It can be the target
or an example for a broader request, such as applying its style across a page. A table `cell`, `row`,
or `column` selection refers to the presentation component's matching data.

Preserve block identities and leave unrelated work intact. `inspect_canvas_blocks` returns a block's
saved source, configuration, and reusable reference. The saved model table or its variables supply
the values for an answer. Focused changes are `source_patches`, one call holding every exact
replacement the change needs; a deliberate whole-component rewrite is a new `source`.
