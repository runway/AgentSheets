---
name: visualization-building
description: Create, check, review, or update pages, charts, and custom visuals. Use when the user wants a visual saved on a page, not just an answer in chat.
---

# Visualization Building

<!-- standards:start -->

Decide whether this work is a chat answer, a new artifact, or a revision. Choose the smallest visual
that makes the relationship clearer, and use the product's branded display system. Before offering
a visual, confirm that the required variables exist. If they do not, name the missing data.

Treat the first saved version as presentation-ready, not as a wireframe. Apply this polish pass to
every new or substantially revised code block before writing it:

- Match a supplied reference or the page's strongest existing visual. Preserve established cards,
  ordering, spacing, and formats unless the user asks for a redesign.
- Make each independently movable visual its own code block. Combine visuals only when they form one
  deliberate composition; give peers equal widths and heights and align their plot areas.
- Use one compact title and, only when it adds context, one short subtitle. Do not repeat the same
  heading in the block, a card, and the chart. Keep header chrome from crowding the plot.
- Start a normal chart at 320 pixels high. Give waterfalls, angled category labels, and dense charts
  more height. Center a deliberately narrow visual; otherwise use the available width and remove
  dead space around or below it.
- Put direct value labels on bars, columns, waterfalls, and other sparse marks when the labels remain
  legible. Prefer labels above positive bars and inside only when there is enough room. Let dense
  charts rely on a clean axis and tooltip rather than overlapping labels.
- Format every visible number for its meaning: currency with a symbol and thousands separators,
  negatives in parentheses, percentages as percentages, and counts without useless decimals. Use
  the same format in axes, data labels, KPI cards, tables, and totals.
- Keep axes complete but quiet. Show readable category or date ticks and formatted value ticks. Add
  an axis title only when the heading and tick labels do not already explain it, and add a visible
  zero reference line when crossing zero changes the meaning. Never emit an empty title.
- Remove non-data clutter before rendering: null or `None` buckets, zero-only series when zero is not
  meaningful, empty cards, duplicate labels, and default legend entries that explain chart mechanics
  instead of business data. Keep a clear fallback for a genuinely empty or failed dataset.
- Use the branded palette. Give peer series distinct colors, keep the same metric the same color
  across related charts, and reserve positive/negative colors for financial meaning.
- Reconcile a bridge before drawing it: beginning total plus signed changes must equal the ending
  total. Show the beginning and ending totals as totals, omit immaterial zero steps, and format the
  value axis and direct labels consistently.

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

Declaring datasets this way means **the model checks your view before anything
is written**. Misspell a variable and the call is rejected with the name that
failed; no block is created. The alternative — a block that saves cleanly and
then renders an error where the chart should be — cannot happen. The other
thing the declaration buys you is the readback, below.

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
- `Chart` — the preferred chart component. Pass normal AG Charts `options` (`data`, `series`, `axes`, `listeners`). It automatically applies CFO.ai's branded AG Charts theme, transparent background, default chart-block palette, tooltip styling, and enterprise chart modules (including ChartBlock types like Nightingale and Waterfall). You usually do **not** need `options.theme`. To size a chart, pass a `height` prop in pixels (`<Chart height={350} options={...} />`); it defaults to the theme's `charts.defaultHeight` (240). `options.height` works the same way; the `height` prop wins when both are given. One resolved height drives both the chart's layout box and its canvas, so a taller chart pushes content below it down instead of painting over it. Apply the axis-title judgment in the standards above. Never pass a `title` object without meaningful `text`: AG Charts auto-enables it and renders a literal "Axis Title" placeholder.
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
  const money = (value) =>
    "$" + Math.round(Number(value) || 0).toLocaleString();
  return (
    <Card>
      <Row>
        <Stat label="Revenue" value="$1.2M" delta="+8%" />
        <Stat label="vs Plan" value="+3.1%" delta="-0.4%" />
      </Row>
      <Chart
        height={320}
        options={{
          data: rows,
          series: [
            {
              type: "bar",
              xKey: "date",
              yKey: "value",
              label: { enabled: true, formatter: ({ value }) => money(value) },
            },
          ],
          axes: [
            { type: "category", position: "bottom" },
            {
              type: "number",
              position: "left",
              label: { formatter: ({ value }) => money(value) },
            },
          ],
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
    <Row align="stretch">
      {[
        ["NA", data.na],
        ["EMEA", data.emea],
        ["APAC", data.apac],
      ].map(([label, ds]) => (
        <Col key={label} style={{ flex: 1, minWidth: 0 }}>
          <Text variant="title">{label}</Text>
          <Chart
            height={300}
            options={{
              data: (ds && ds.rows) || [],
              series: [{ type: "line", xKey: "date", yKey: "value" }],
            }}
          />
        </Col>
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
