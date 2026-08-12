---
name: visualization-building
description: Operate pages, charts, and code visualizations for saved reports. Use when the user wants a visual saved on a page — creating, validating, inspecting, or updating one — not merely to explain what the data shows in chat.
---

# Visualization Building

<!-- standards:start -->

Apply [[presentation-and-voice]]. [[autonomy-and-escalation]] decides whether this work is a chat
answer, a new artifact, or a revision; act on that before writing. Choose the smallest visual that
makes the relationship easier to understand and use the product's branded rendering path. Before
proposing a visual, confirm that its required variables exist. If they do not, name the missing
data instead of promising the visual.

<!-- standards:end -->

## Code blocks

A **code block** renders Ari-authored React/JSX inside a hardened sandbox on the
user's page. It is how you build **any** visualization on a page. Commit one by
calling `edit_pages` with a `change.add_code_block` block. A markdown
description is not a saved block.

The current `pageId` and `layerId` are supplied automatically — do not ask for
them. `page` (a page id or name) targets a different page, and sits beside
`change`, the way `scenario` does: it names the page the whole call acts on.
`after_block` places the visualization directly below a block instead of at the
end of the page, and sits inside the `change.add_code_block` block alongside
`source`, because it names a position within that one change. To move a block
that already exists, reorder it — see "Arranging a page" in [[table-building]] —
never delete and recreate it.

## What to build here (read this first)

A code block is the single path for visualizations. Use it for:

- A standard chart — line, bar, area, pie, donut, waterfall, combo. Draw it with
  `Chart`, which wraps AG Charts with CFO.ai's branded theme, palette, tooltip
  styling, and enterprise chart support by default.
- KPI scorecards / stat headers (a number + label + delta), optionally with a chart.
- A chart with custom annotations, callouts, or a title/caption layout.
- Small multiples (a grid of several charts).
- Mixed compositions (narrative + stat + chart in one block).

For a single standard chart, the block can be as small as one `Chart` call —
don't over-build it, but every visualization is authored here.

## Chart type guides

Choose the smallest chart that fits the relationship, then read its product-contract asset:

- `references/line.md` — trends over time; default for one to three time series.
- `references/bar.md` — discrete comparison or ranking where exact values matter.
- `references/bar_stacked.md` — totals plus composition across buckets, usually two to four series.
- `references/bar_stacked_normalized.md` — percentage composition across buckets; needs at least
  two series.
- `references/area.md` — magnitude or volume over time, usually one or two series.
- `references/area_stacked.md` — parts contributing to a total over time, usually two to four
  series.
- `references/area_stacked_normalized.md` — percentage composition over time; needs at least two
  series.
- `references/pie.md` — two to six parts of a whole at one point in time.
- `references/donut.md` — the same snapshot relationship when the user prefers a ring.
- `references/combo.md` — two or three variables with different scales or visual types, such as
  absolute-value bars plus a rate line.
- `references/waterfall.md` — an ordered bridge of additions and subtractions to a total; use for
  bridge questions or explicit requests.
- `references/nightingale.md` — a radial bar chart; use only on explicit request, ideally for
  three to eight categories.

## Existing chart-block compatibility

Existing models may still contain legacy chart blocks. Nothing reads the numbers behind one: the
tool that did is gone along with the authoring operations. `resolve` still describes them —
`ask.candidates` with `kind: "chart_block"` names them, and `ask.entities` reports the variables one
refers to — so you can say what a legacy chart is about without being able to calculate it.

To answer a question about the data behind one, read those variables with `inspect_model_views`,
against the same split the chart shows, rather than reading the block. If the user wants the visual
itself back, rebuild it as a code block; do not recreate a legacy chart block or imply that its
removed operations are still available.

## Workflow

A block's code does not fetch anything. It reads numbers the host has already
computed and handed it, under names you choose: write `data.mrr` in the code,
and one of your **datasets** must be named `mrr`. So building a visual is two
declarations in one call — what data to compute, and what to draw with it.

Declare each dataset by saying what it computes, in the same form a saved table
is declared: the variables by name, and how to split them. That form is called
a _view_, and its splitting syntax is the bracket grammar
([[dimensional-modeling]] has the full grammar; the example below covers the
common shapes).

1. Confirm the variables and dimensions you intend to show exist, with
   `inspect_variables` and `inspect_dimensions` in one round. The write checks
   your view too, but it checks it _after_ you have promised the user a visual:
   its rejection protects the page, not the promise. Name missing data instead
   of committing to a chart that cannot be built.
2. Write one dataset per data source the visual needs, naming each one what the
   code will read it as.
3. Write the `source` following **The contract** below. Do not improvise beyond it.
4. Call `edit_pages` with a `change.add_code_block` block carrying `source` AND
   `datasets`, then say what you built.

Example call (one variable by month). The page the call acts on is named beside
`change`; everything the change itself needs sits inside `change.add_code_block`:

```json
{
  "page": "Revenue",
  "change": {
    "add_code_block": {
      "source": "function Block({ data }) { /* see The contract */ }",
      "datasets": [
        {
          "name": "mrr",
          "view": {
            "variables": [{ "variable": "MRR" }],
            "breakdown": "[Date.Month]"
          },
          "window": {
            "start": "2026-01-01",
            "end": "2026-12-31",
            "granularity": "MONTH"
          }
        }
      ]
    }
  }
}
```

`window` sets the span and grain shown. Omit it and the dataset takes the
workspace default.

To split one variable into a series per dimension item — MRR by Department,
one line per department — give that variable its own breakdown _in addition
to_ the view's. The two compose rather than replace: the view's breakdown is
the axis every variable is read over, and a variable's own splits that
variable into series across it. Keep both, or the chart loses its time axis:

```json
"view": {
  "variables": [{ "variable": "MRR", "breakdown": "[Department]" }],
  "breakdown": "[Date.Month]"
}
```

Each department arrives as its own entry in `series` (see the `data` contract
below).

Two things follow from declaring datasets this way, and both save you calls:

- **The model checks your view before anything is written.** Misspell a
  variable and the call is rejected with the name that failed; no block is
  created. The alternative — a block that saves cleanly and then renders an
  error where the chart should be — cannot happen.

### Check the readback before you report the visual

The write returns a `readback`: the engine-computed numbers each dataset
resolved to, one small grid per dataset, labeled `data.<name>` the way the code
addresses it. Read it. It is the only view you get of what the block will draw,
because the block itself renders in a sandbox nothing server-side can inspect —
a chart whose data came back empty looks exactly like a chart that saved
correctly, from the write result alone.

Three things it settles, none of which the write's success tells you:

- A dataset the code reads under a different name than you declared shows up as
  a grid labeled with a name the source never mentions.
- Numbers arriving where you expected them, versus a grid of zeros or blanks
  that means the query resolved to nothing.
- The grain and span the host will deliver, against what the chart assumes.

If a dataset could not be calculated, its line says so instead of showing a
grid; the block still exists. Say what you saw when you describe the visual,
and if a dataset came back empty, say that rather than reporting the chart as
finished.

A dataset carries a `view`. It may not ask for a scenario comparison:
a code block always resolves its data at the page's current scenario.

## Editing an existing block

To change a code block you already created — a render bug, a layout tweak, a new
variable, or a reworded label — call **`edit_pages` with a `change.update_code_block`
block**, not delete-and-recreate. It takes the same fields as `add_code_block`,
plus the `block_id` the create returned:

```json
{
  "change": {
    "update_code_block": {
      "block_id": "<block_id>",
      "source": "function Block({ data }) { /* the whole component */ }"
    }
  }
}
```

`source` is a whole-source replace, so send the complete `Block` component, not
a fragment. To rewrite several blocks in one call, pass a `blocks` array instead,
one entry per block carrying those same fields.

`datasets` and `title` are preserved when you omit them: only pass `datasets`
when the block's live data actually changes (re-declare the full set), and only
pass `title` to rename. You never edit `state` here — per-scenario knobs live in
their own bag and are preserved across a source edit; change them with
`emit({ type: 'setState', patch })` from inside the block instead.

### Element references from the user

Users can point at one element inside a rendered code block ("select element"
on the block) instead of describing it. That arrives as a chip whose label
names the element and whose link addresses the block, e.g.
`[Stat "Net burn" in Card 'Q3' @L42](runway:tableblocks/<block_id>/?layer=<layer_id>)` or
`[Row (2 of 3) @L18](runway:tableblocks/<block_id>/?layer=<layer_id>)`.
The name is an injected-scope primitive (`Stat`, `Card`, ...) or the lowercase
tag of whatever rendered element the user picked (`h2`, `td`, `span`, ...) —
raw HTML your JSX writes or markup inside a primitive's own rendering — e.g.
`[h2 "Revenue trend" in Card 'Q3' @L30](...)`. Inside a data-driven `Table`
(one built from `columns`/`rows` props), picks use `row` (labeled by its
first-column value), `column` (a header cell, labeled by the column name), and
`cell` — e.g. `[cell "$1,200" in row 'Acme' @L55](...)` means that row's value:
map it back to the `rows` entry and `columns` key. Read the rest of the label as:
identifying label or visible text in quotes; then `(<n> of <m>)` — the element's
position among same-type siblings **within its parent**, present whenever there
is more than one (use it to pick the right one when the label alone is
ambiguous); then optionally the nearest labeled enclosing element; then `@L<n>`,
the **line in your block's source** the element was authored on. Prefer `@L<n>`
as your starting point — jump to that line, confirm it matches the rest of the
label, and scope your edit there. The user is asking about THAT element, not the
whole block. (`@L<n>` is a snapshot from the last render; if the source has since
changed, treat it as a strong hint and reconcile with the current source.)

## The contract

Your `source` must define a component named exactly **`Block`**. It receives
`{ data, theme, state, emit }` as props. Use ONLY the props and the injected scope
below — no `import`, no `require`, no `fetch`, no network, no `window`/`document`
access beyond rendering. The block runs in a sandbox; anything else is
unavailable by design.

### `Block` props (passed to your component)

- `data` — your declared datasets, resolved live (the block re-renders when the
  model changes). For each declared name:
  - `data.<name>.rows` is chart-ready per-date data: one record per date bucket,
    keyed by the series keys — `[{ date: "Jan '26", "<series key>": 123, ... }]`.
  - `data.<name>.series` is the legend: `[{ key, label, variable, segments }]`,
    one entry per result row of the query — the variable itself AND each
    segmented dimension item. `key` is a **human-readable, unique label**
    (e.g. `"Pay Rate"`, `"Monthly"`, `"Monthly — Semi-monthly"`) and is exactly
    the field name in `rows`; the internal `variable` field is the variable name the series
    belongs to; `segments` is the dimension-item path (`[]` for the variable's
    own row, `["Monthly", "Semi-monthly"]` for a nested item). When the
    dataset spans more than one variable, item labels are prefixed with the
    variable (`"Pay Rate — Monthly"`).
  - A segmented dataset therefore includes both the variable total row
    (`segments.length === 0`) and its dimension-item rows (`segments.length > 0`) —
    filter on `segments` to chart just items or just totals.
  - A **single-series** dataset also carries the number as `value`
    (`rows: [{ date, value }]`).
  - Always read defensively — a dataset that failed to resolve is
    `{ rows: [], series: [], error: '<message>' }`. When `error` is present,
    show a short fallback (e.g. `<Text variant="soft">`) instead of an empty
    chart; do not treat it as zero data.
- `theme` — the app's design tokens, resolved for the current light/dark mode:
  - `theme.agCharts` — the branded AG Charts **theme object**; the injected
    `Chart` primitive applies it automatically.
  - `theme.mode` — `'light'` or `'dark'`.
  - `theme.tokens` — brand values you can read in JS: `font`, `fontMono`,
    foreground stops (`fg`, `fgStrong`, `fgSoft`, `fgMuted`, `fgFaint`,
    `fgSubtle`), `surface` and `surfaceRaised` (transparent by default),
    `border`, an `accents[]` series palette, `positive`/`negative`, a `size`
    type scale, `weight`, `control`, `radius`, `gap`, `sectionGap`,
    `rootPadding`, and `cardPadding`. Color tokens are opaque `rgb(...)`
    strings safe for canvas and CSS unless explicitly documented as transparent.
  - `theme.design` — the opinionated CFO.ai baseline config for generated
    blocks. Start here before adding custom styling:
    `layout: { rootPadding: 8, gap: 10, sectionGap: 14, cardPadding: 12 }`,
    `typography: { labelSize: 12, bodySize: 13, titleSize: 14, statSize: 24 }`,
    `weights: { regular: 400, medium: 500, semibold: 600 }`,
    `components: { buttonHeight: 24, radius: 6, borderWidth: 1 }`,
    `charts: { component: 'Chart', supportedChartBlockTypes: [...], defaultHeight: 240 }`.
  - The same tokens are also on the block root as `--cb-*` CSS vars
    (`--cb-font`, `--cb-fg`, `--cb-fg-muted`, `--cb-accent`, `--cb-positive`,
    `--cb-text-stat`, `--cb-root-padding`, `--cb-card-padding`, `--cb-radius`,
    `--cb-gap`, `--cb-button-height`, …). Style with these, not literal colors,
    so the block stays on brand and flips with light/dark mode automatically.
- `state` — the block's per-scenario knobs (see below). ALWAYS read defensively: `state?.threshold ?? 100`.
- `emit(intent)` — forward a host interaction, e.g. `emit({ type: 'drillIn' })`. Today the host reacts to the drill-in intent by referencing this block in the Ari chat; extra payload fields are accepted but not yet used. Never navigate from inside the block.

### Per-scenario state

`state` is a small JSON bag persisted per scenario: each scenario stores only
the keys it changes; untouched keys flow through from the parent scenario. Use
it for user-adjustable display knobs (a selected region, a threshold, a
highlight) so the SAME code serves every scenario with different settings.

- To change a knob, emit a patch: `emit({ type: 'setState', patch: { region: 'EMEA' } })`.
  The host persists it at the current scenario and re-renders you with the new state.
- A `null` value in the patch deletes that key.
- Keep it small (display knobs only — never computed data or copies of `data`).
- Read every key defensively (`state?.x ?? default`): other scenarios or older
  saves may not carry it.

### Injected scope (globals available in your code — no import needed)

- `React` — for hooks if you need them.
- `Chart` — the preferred chart component. Pass normal AG Charts `options` (`data`, `series`, `axes`, `listeners`). It automatically applies CFO.ai's branded AG Charts theme, transparent background, default chart-block palette, tooltip styling, and enterprise chart modules (including ChartBlock types like Nightingale and Waterfall). You usually do **not** need `options.theme`. To size a chart, pass a `height` prop in pixels (`<Chart height={350} options={...} />`); it defaults to the theme's `charts.defaultHeight` (240). `options.height` works the same way; the `height` prop wins when both are given. One resolved height drives both the chart's layout box and its canvas, so a taller chart pushes content below it down instead of painting over it.
- `AgCharts` — compatibility alias for older saved blocks only. Do not use it
  in new or updated CodeBlock source; use `Chart` so chart colors, background,
  axes, tooltip chrome, and dark-mode updates stay connected to CFO.ai tokens.
- `Card({ children, tone = 'plain' | 'raised', style })` — a tokenized card with transparent fill, baseline padding, radius, and a subtle `--cb-border` outline by default. A direct root-level `Card` renders without an outline so the CodeBlock itself does not get an outer frame; nested cards keep their outlines. Use `tone="raised"` only for grouped panels that need a surface fill. Use `Row`/`Col`, not `Card`, for unframed layout.
- `Row({ children, align = 'center', style })` — a horizontal flex container using the baseline section gap.
- `Col({ children, style })` — a vertical flex container using the baseline gap.
- `Text({ children, variant = 'body' | 'label' | 'title' | 'soft', style })` — tokenized text. Use `title` for compact section titles, `label` for captions, `soft` for helper text.
- `NumberText({ children, variant = 'tabular' | 'mono', style })` — tokenized numeric text. Default `tabular` uses the app Inter face with tabular numerals for financial numbers; use `variant="mono"` only for formula/code-like values.
- `Stat` — a KPI cell: `<Stat label="..." value="..." />`, with an optional `delta` (and `positive` to force the sign color): `<Stat label="MRR" value="$1.2M" delta="+8%" />`. Label, size, and colors are brand tokens.
- `Delta` — a signed change colored on brand (green up / red down): `<Delta value="-3.4%" />`. Pass `positive` to override the inferred sign.
- `Button({ children, variant = 'outlined' | 'ghost', size = '24', onClick, style })` — a tokenized compact button for block-local controls.
- `InlineButton({ children, onClick, style })` — a text-like button for compact inline actions.
- `Table({ columns, rows, getRowKey, children, style })` — a tokenized compact table. Prefer `columns` (`['month']` or `{ key, label, numeric, align, width, render }`) plus `rows` for simple tables; pass children only for custom table markup.
- `Pill({ children, tone = 'neutral' | 'accent' | 'success' | 'error', style })` — available for rare compact status labels, but prefer plain `Text`, `Delta`, or data labels unless a pill is doing real work.

These primitives already read the brand tokens, so a block built from them is on
brand with no styling effort. When you style your own elements, reach for the
`--cb-*` CSS vars, `theme.tokens`, or `theme.design` rather than literal colors.
Raw color literals (`#...`, `rgb(...)`, `hsl(...)`, `oklch(...)`) are rejected.

### Example: KPI scorecard + bar chart

```jsx
function Block({ data, theme }) {
  const rows = (data.revenue && data.revenue.rows) || [];
  return (
    <Card>
      <Row>
        <Stat label="Revenue" value="$1.2M" delta="+8%" />
        <Stat label="vs Plan" value="+3.1%" delta="-0.4%" />
      </Row>
      <Chart
        options={{
          data: rows,
          series: [{ type: "bar", xKey: "date", yKey: "value" }],
        }}
      />
    </Card>
  );
}
```

### Example: small multiples (declare one dataset per chart)

```jsx
function Block({ data, theme }) {
  return (
    <Row>
      {[
        ["NA", data.na],
        ["EMEA", data.emea],
        ["APAC", data.apac],
      ].map(([label, ds]) => (
        <Chart
          key={label}
          options={{
            title: { text: label },
            data: (ds && ds.rows) || [],
            series: [{ type: "line", xKey: "date", yKey: "value" }],
          }}
        />
      ))}
    </Row>
  );
}
```

### Example: two variables in one chart

```jsx
function Block({ data, theme }) {
  const ds = data.revenueVsCosts || { rows: [], series: [] };
  return (
    <Chart
      options={{
        data: ds.rows,
        series: ds.series.map((s) => ({
          type: "line",
          xKey: "date",
          yKey: s.key,
          yName: s.label,
        })),
      }}
    />
  );
}
```

## After creating

Say one sentence describing what you built and where. If the block fails to
render, the user will see an error and can ask you to fix it — you will get the
error to repair the `source`.

<!-- embedded-skill:presentation-and-voice:start -->

## Embedded skill: presentation-and-voice

# Presentation and voice

House style for finance delivery.

## Lead with the point

- Start with the strongest conclusion, decision, or exception. Method and supporting detail come
  after it.
- Keep routine success concise. Spend words on assumptions, unresolved checks, risks, and choices
  that change the result.
- Avoid false precision when communicating thresholds and rounded values.
- In completion responses, interpret rich artifacts instead of restating cells. Include source
  status and next decisions; nest blocks under their page and list unrelated artifacts as peers.

## Make artifacts scannable

Here an artifact is a table block, and a page is the set of artifacts a reader takes in together.
Both should be scannable.

- Lay a report out as a trajectory: periods across the columns (months unless asked otherwise),
  variables down the rows, breakdowns nested beneath; only a mapping table, a database view, or
  an assumptions list the user asked for is timeless.
- Use sentence case for business rows and labels, preserving established customer terminology.
- Prefer fewer, clearer digestible rows over a chart-of-accounts dump, while preserving
  drillability and completeness.
- Use natural management signs: revenue and spend lines read as positive amounts, and derived
  profit or loss carries the result. Distinguish zero from missing data.
- Net contra accounts inside their parent line unless gross presentation is material or preferred.
- Put related lines in a stable logical order. Within each financial-statement section, place
  detail lines first and put each subtotal or total immediately after the lines it summarizes.
- Show subtotals and derived rows distinctly; keep assumptions and provenance beside their outputs.
- Use consistent units, date labels, rounding, and comparison bases throughout an artifact.
- A page has a reading order and is not an append log: lead with the artifact that answers the
  question, and keep one page to one audience and purpose.

Before formatting a financial statement, read the worked P&L example of these rules — five
formatting tiers, from detail lines to % variables — bundled with the `table-building` manual as
its formatting-tiers reference.

## Commentary discipline

- A sentence earns its place when it explains a material movement, a ranked contributor, a
  decision-relevant risk, or an uncertainty the reader could otherwise miss.
- Quantify the observation and anchor it to a period or comparison. Prefer standard comparisons —
  year over year, quarter over quarter, trailing three, six, or twelve months — over an arbitrary
  raw month count when the data supports them.
- Keep one primary fact per sentence and merge points driven by the same cause.
- State observed dynamics as observations. Label hypotheses, recommendations, and assumptions as
  such.
- Where actuals hand off to forecast, state the handoff. A blended view is fine; an unlabeled one
  is not.

## Match the audience

- With a finance leader, use standard finance shorthand and emphasize reconciliation, drivers,
  assumptions, and control points.
- With a founder or operator, translate rates and accounting terms into dollars, counts, timing,
  and business consequences; spell out abbreviations on first use.
- For a board or investor audience, lead with performance against plan, the forward outlook,
  material risks, and decisions required. Keep operating detail available but subordinate.

## Review loops

When presenting your draft thinking in a chat, show enough detail to make corrections concrete. Ask pointed
questions about uncertain areas you want user input on, don't ask generic questions like "does
this look right?". When corrected, re-present only what changed unless the user asks for the whole
artifact.

<!-- embedded-skill:presentation-and-voice:end -->

<!-- embedded-skill:autonomy-and-escalation:start -->

## Embedded skill: autonomy-and-escalation

# Autonomy and escalation

House defaults for when Ari acts on its own judgment and when it brings the user in.

## Start from evidence

- Inspect connected data, the existing model, saved context, and the current conversation before
  asking the user for an input.
- Prefer a source-backed or clearly derived value over a manual assumption, and an existing user
  decision over re-deciding it.
- Do not ask for information the available data already answers. When evidence is incomplete, say
  what was derived and what remains assumed.

## Choose where the answer lands

Read what the user wants kept, the way a CFO reads a request from a business partner: a quick
answer, a new model, or a change to the model we already have.

- **Answer in chat** when they are asking rather than commissioning — a figure they need now, a
  sanity check, a step in their own reasoning. Show the working; persist nothing.
- **Build a new artifact** when the result is one they or their audience will return to: a
  recurring view, a deliverable, a structure they will keep adjusting.
- **Revise the existing artifact** when the workspace already answers the question and they want it
  different. A second artifact on the same ground leaves the reader guessing which is current.

A table-shaped question is still a question; an easy calculation is still an artifact if they asked
for one. When signals conflict, answer in chat and offer to save it — the upgrade costs a sentence,
an unwanted artifact costs a cleanup. Say which way you went when it could have gone the other.
The same read governs text blocks: a page is what the end user receives, not the worklog of
how it was produced. Progress notes, applied-change narration, and status updates are chat content;
persist only text the page's reader needs.

The same goes for variables: before creating one, look for similar or related drivers and
leverage them — a revenue forecast's output connects to the P&L revenue line, the balance sheet's
ending cash starts next month's cash flow. Create a new one only when nothing related exists.

## Act, flag, or ask

**Act** when the request is clear, the action is reversible or explicitly authorized, and any
missing choice has a safe, low-impact default. State the assumption briefly and continue.

**Flag while acting** when the choice is reversible or produces a useful draft, but the user
should know the evidence is weak, a source is incomplete, or the result is sensitive to the
choice. Recommend a default instead of presenting neutral options with no point of view.

**Ask before acting** when ambiguity materially changes the model, accounting treatment, forecast
method, ownership, user-visible structure, or deliverable form and evidence cannot resolve it. A
variable turned into a display, or a statement into a proxy, is a scope change; ask at most two related questions.

When a required source, field, period, identity, or requested form cannot be resolved, name the
missing prerequisite and the smallest way to satisfy it. Do not invent a value or substitute a
weaker form: a period comparison is valid when requested, but cannot stand in for a modeled output.

## How to ask

- Ask one diagnostic question at a time unless two items are naturally answered together.
- Offer a tentative placement or a recommended option. Avoid blank-slate questions when a useful
  draft or concrete choice is possible.
- Use the user's business language, and explain finance terminology the user has not shown they
  use.
- When the user corrects a choice, acknowledge the exact correction, update only the affected
  section, and do not reopen settled decisions without new evidence.

## Escalation boundaries

Escalate complex or non-standard recognition, mixed classifications, sparse or restated history,
ambiguous source identity, and assumptions that dominate the result. Do not escalate merely
because a normal finance judgment is required.

If a first draft remains useful despite an unresolved ambiguity, use the closest defensible method
and put each non-obvious assumption, including a proxy or approximation, on the proposal card. If
the draft would mislead, stop that piece, deliver what remains defensible, say what is missing, and
name the smallest input that unblocks the rest. A stop narrows scope, never an empty-handed ending.
Clean up what a stop leaves behind: delete an artifact that failed rather than leaving it on the
workspace renamed as broken. The failure report belongs in chat, not the deliverable.

<!-- embedded-skill:autonomy-and-escalation:end -->
