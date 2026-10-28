---
name: build-visualizations
description: Build model-connected Canvas visualizations and interactive applications. Use when choosing a business representation, making timing or other choices directly manipulable, binding datasets, or reviewing the rendered result.
---

# Build visualizations

A Canvas application is a working representation of a business: the outcomes a person answers
for, the levers that move them, and the choices they make. The financial model describes what
happens; the application gives the person a useful way to act on that model and understand the
consequences. Its scope follows its reader and their question. A question about one choice gets an
application about that choice. A first picture of a business, such as the application that closes
an onboarding build, shows the business as a system: its outcome over time and the few levers that
move it most. An explanatory visual makes evidence easier to compare. A simulation makes business
choices directly manipulable.

The design bar is a finely made print graphic that the reader can handle: Tufte's clarity of
quantitative evidence, the composition and typesetting of an editorial infographic, and Bret
Victor's immediacy of action and consequence. The shared product theme supplies the visual language.

The person came to read a number or move a lever, so the block is first a working product of its
category. Its controls are the ones the person already knows, the same control looks the same
everywhere in the block, and motion marks a state change in under a quarter of a second and then
settles. The workspace brand lends type, palette, density, and one signature move; it never
supplies the layout or the controls. Brand lives in precise details.

Its form comes from the objects and relationships in the business. A hire begins at a date and
creates recurring cost and capacity. A contract has a service period, billing terms, and
collections. A scarce resource is divided between competing uses. The model determines which
quantities are choices, which are consequences, and what connects them.

When timing matters, moving a consequential event lets the person see how a familiar scheduling
decision changes several parts of the forecast together. Give the event a visible place in time and
make that position adjustable. Its dependencies
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
visual home. Those annotations are the summary: a separate summary row earns its place only by
stating what the figure does not already show, in smaller and later type. Selecting an object
reveals its exact editor at that object, so one business choice has one home. The editor's
settled fields show the formatted value — the same currency, unit, and precision as the mark
they edit — and hold a raw number only while the person is typing. Each distinct
choice in the object has its editor at its own mark: a fee at its amount, a lag at the interval
it creates, a start at its event. A form that collects those choices elsewhere repeats the
figure instead of completing it. A suggested
alternative or optimizer result stays subordinate to the saved plan the person is exploring.

In a time-based figure, the same horizontal position means the same period. Event marks sit on or
beside the financial paths they affect. The person can hold an event, move it, and watch the
response in that picture. The saved position and original path remain visible. Constraints,
plateaus, and delays appear where they affect the choice. Related marks retain their identity
across views. This relationship stays intact as the width changes. The same principle holds
without time: an amount is edited at its drawn value, a conversion at the edge between its
stages, a fixed-total share at the boundary between its segments, and a future quantity at its
own point. In a composed figure, that boundary is itself the draggable handle, with each
segment's amount and share labeled at the segment. In each case the gesture, the saved state, and the responding consequence share one
picture.

The question determines the information form. A known conclusion needs its convincing proof.
Exploring a population benefits from comparison, focus, and exact detail; tables with embedded
marks and aligned small multiples can make dense evidence readable. Position and length carry
quantities, hue carries identity, and an ordered lightness scale carries magnitude. Comparable
views retain consistent units, scales, and periods. Labels belong to the marks they explain.
Each important distinction has a readable cue in its position, shape, line style, or label as well
as color. One hue keeps one meaning inside a block.

The opening reads as one composed picture. A short business question or claim uses the shared
heading treatment, the working figure carries the strongest visual weight, and supporting detail
stays quieter. Use the shared type roles and the injected formatValue and formatCompactValue
helpers rather than inventing local font-size scales or number formatters. Direct labels, exact
editors, and tooltips use exact source formats; axes may use compact values. Comparable numbers
align with tabular numerals. A zero difference reads "No change" in neutral treatment. Mixed
tradeoffs use neutral signed values; favorable and unfavorable colors express an explicit business
judgment rather than numeric sign.

Axis ticks are deliberate round values in business units, and scales hold steady through a gesture
so movement reads as business change. Discrete periods use linear or step segments; smooth
interpolation belongs to a meaningful continuous relationship. Direct labels name saved and
current values at the important endpoint, low, crossing, or constraint. At rest, the figure
draws the saved path alone or one path labeled as both saved and current. When the paths
diverge, the difference itself is drawn and labeled where it matters most. When a choice leaves
its outcome unchanged, the relationship that holds it is drawn, such as demand staying below
capacity.

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

Canvas authoring starts with an `inspect_canvas_library` `read` of a visual component or template. Its `runtime_contract` defines
the sandbox's data, theme, state, and event interfaces once beside the requested component props.
The source field keeps the component signature and common layout props available throughout the work.

At a narrow width, responsive geometry, focus on a selected object, or readable bounded scrolling
keeps the useful choice and its immediate consequence together. The words on a block are its
business question, the labels at its marks, the values, and how to use its controls. Everything
said about the work itself lives in the Ari chat session: estimate and forecast disclosures, their
sources and assumptions, authoring checks, a preview's scope, and what a number is not. A delegated
writer returns those details to Ari in its report.
The host supplies the boundary and inset, so begin
with a transparent, unbordered field. Containers express real groups, states, or interactive
boundaries. The shared design system supplies coherence; the business supplies the distinctive
composition. Move an existing block rather than recreating it.

## A business on one page

A first picture of a business is one composition with one reading order. The parts below are the
ones that usually carry the relationship between an outcome and its levers; the business decides
the exact form each one takes. A short headline states
the business and the state it is in, in the company's own words. The backbone figure owns the
center: the outcome the reader answers for, drawn over time with its saved path, its milestones,
and the point where it crosses a threshold that matters to this business, labeled at the mark.
The levers sit beside or beneath it as a set of smaller figures that share its time axis or its
scale, each drawn as the business object it is: a launch as a dated span, a price as a point on its
demand curve, a funnel stage as the edge between two volumes, a team as its headcount steps. A rate
is the shape of the path it produces: a growth rate is the bend of that future path, so the person
drags the path's end and the rate follows, with the exact rate shown at the mark. Each
lever is changed where it is drawn, and every change redraws the backbone in place, with the
saved path still visible and the difference drawn. Detail and method come last, visibly quieter.

The levers are the inputs that move the outcome most in this business, drawn from every part of
it that moves the outcome: what brings revenue in, what sends cash out, and how the business is
funded. Within each part, the model shows which inputs carry the largest amounts, and the block's
local preview confirms the choice, because it computes the outcome for any lever value.

Each lever also shows its whole range. Beside the lever, a small curve plots the outcome across
the lever's plausible range, computed with the same preview function, with the current value
marked and, when the model holds a threshold such as a cash reserve, the value where the outcome
crosses it labeled. The reader sees where the
edge is before reaching it, which a single preview value cannot show. The headline sentence can be
live too: a sentence that states the current answer, whose input values the reader can drag in
place, keeps the words and the figure in agreement.

The craft is that of a finely made print graphic. The business's own objects, units, and
vocabulary give the composition its identity, so a fleet's vehicles, a marketplace's bookings, and
a retailer's stores each look like themselves. Annotations sit where the eye lands, joined to their
marks by short leaders. Generous margins, aligned edges, one accent for the current selection, and
quiet axes let the evidence carry the page. The workspace brand's heading face and accent arrive
through the theme tokens.

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
alternatives within that representation. They are ordinary HTML: `<button>` for actions, `<input>`
for text, numbers, dates, times, and months, `<select>` or radio inputs in a `<fieldset>` for choices, a
checkbox for an on/off value (with `role="switch"` it draws as a switch), `<input type="range">`
beside an `<output>` for an abstract range, and `<textarea>` for longer text. A file input lets the
person choose a local file, which the block reads in that browser with `File.text()` or
`arrayBuffer()`; Ari receives only what the block passes on, such as text in a `startAriChat`
message, which holds at most 4,000 characters. The Canvas stylesheet draws each element in the
product's type, colors, corners, focus ring, and disabled state, in light and dark mode, so a
`<label>` and the element make a finished control. An unmarked button is outlined; `data-variant`
of `accent`, `ghost`, or `destructive` sets its emphasis, and an element with
`data-variant="segmented"` draws the radio labels that are its direct children as one segmented track. A style or class on an element takes
precedence over the stylesheet. Local React components and HTML/SVG compose layout and specialized
marks around these controls. A local component lives at the top level of the source, outside
`Block`, and takes its values as props. One declared inside `Block` is a new component on each
render, so React remounts it: its chart redraws and loses hover state, and its input loses focus.
A Canvas save refuses a source that declares one there and renders it as JSX. A create from
`source_template` saves the published code as it is, and the block's next source change takes
that check.

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
point only at the whole block. Keep the id when the source changes. An element in a part that
starts closed, such as a disclosure or a tab, isn't drawn until the person opens it. Name its ids
on the control that opens it, as `data-guide-reveals="liquidity holdings"`, so a walkthrough step
on one of them opens the part first. Keep a disclosure toggle's `aria-expanded` current, so the
step never closes a part that is already open.

A button that launches a saved guide asks Ari for it in a new chat. A `data-ari-action` attribute
marks a button that starts Ari work: the stylesheet gives it the standard trailing arrow and Ari's
moving edge on hover and keyboard focus. The button calls
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
but it cannot shrink that content. A warning that begins with "Design:" names an element wearing a
generated interface's default: a container inside a container, a stripe or a glow standing in for
emphasis, a tracked uppercase label above a heading, text below reading size or below contrast, a
size off the type scale, motion that marks no change, or an emoji where an icon belongs. It names
the element, its source position when it has one, what was measured, and the move. An element with that shape was
assembled, not decided. Rewrite it as the thing the business needs; a thinner border or a smaller
label keeps the default and loses its clarity. The person's explicit request for one of those
forms is the one thing that earns it. Use that evidence for the changed block; its warnings can reveal a
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
whether the picture makes the business tangible.

Request a `visual_review` with `view: "desktop-opening"` and inspect the rendered application as a
designer. This is a fixed Canvas area visible before scrolling. Its independent assessment is a fresh
reader's judgment of the visible evidence; concrete gaps between that reading and the intended
experience are design work. Its `brief` carries that intended experience in your words: what the
opening shows and the choices already settled for it, including the hierarchy, scope, and
requirements the person or a review set, so the assessment judges the design you were asked for. Keep the same opening through revisions so the comparison tests the
composition. A measured `viewport` explores a different available area;
the full image shows secondary content below the opening. `framing` reports what the image actually
shows. Choose a deliberate
composition for its particular objects and relationships. The opening view puts the meaningful
action and its financial consequence in one working picture. Controls belong on the things they
change; exact entry belongs at the selected object; current values and comparisons belong at the
marks they explain. Read the opening as its first reader: what the summaries repeat, where the
editor sits relative to its object, whether the saved comparison can actually be seen, and
whether the scales and colors mean one thing are composition findings; revise the largest
finding first. A revision keeps every lever the picture offers: a lever held in a separate
editor moves onto the object it changes, so the reader can still handle it. Then read it at a squint: one headline that states the business question with
confidence, one figure that owns the center, and support that is visibly quieter. Hierarchy
comes from size, weight, position, and contrast; summaries sit as open columns under light
rules, and a border encloses only a group that needs one. The company's identity lives in the
composition itself — its objects, units, and relationships — while the shared tokens keep it
recognizably part of the product. Use the available space to make that relationship immediately understandable
on both desktop and a narrow screen. The person's attention can focus on one object while the
rest retain enough context to choose another.

Each non-null `revision` names a material defect and drives another largest-first composition pass
and a new review of the same opening. Handoff follows `revision: null`, which means no material
visual defect remains under the assessment rubric. `refinement` is an optional improvement that
leaves the reading intact; it is yours to take or leave within the brief. A finding left unresolved at the existing turn budget stays open in the handoff; it
is not an aesthetic pass.

The existing data and calculation behavior are the basis of this pass. Preserve baseline
comparison, reset, keyboard and exact entry, and model inspection. Keep the shared type and theme
tokens; improve their use through sizing, weight, alignment, line breaks, contrast, and restraint.
A row of identical cards, a label over every heading, or a summary strip restating the figure is
a default of the same kind, one the visual review judges rather than the render check measures:
assembled rather than decided unless the business earns the form. Rewrite such an element as the
one the business needs. Revise the composition's largest weakness, then inspect the rendered
result again. The design critique belongs in the chat handoff; the application itself
explains the business.

The block image does not show the surrounding page, and a static render does not exercise a gesture.
Use available interaction evidence to check the changed date or amount, its dependent values, reset,
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
replacement the change needs; a deliberate whole-component rewrite is a new `source`, with the
`source_hash` of the read or write it was written from as `based_on`.
