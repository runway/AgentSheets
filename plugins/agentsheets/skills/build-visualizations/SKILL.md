---
name: build-visualizations
description: Build model-connected Canvas visualizations and interactive applications. Use when choosing a business representation, making timing or other choices directly manipulable, binding datasets, or reviewing the rendered result.
---

# Build visualizations

A Canvas block is an interactive React application or explanatory visual bound to live model datasets (`[[build-model]]` owns native tables). Match its scope to the user's question: a focused decision gets a focused simulator; a full model build closes with a system-wide simulator showing the primary outcome over time and the few levers that move it most.

## Core Design Principles

- **Requested page content**: Keep application page prose to 1–2 lines on purpose and use. Reports contain the requested figures and narrative, including explicitly requested sources, qualifications, or methods. A report skill's page layout is part of what the person asked for.
- **Authoring notes**: Keep unrequested preparation notes in Ari chat: source coverage, reconciliation status, estimate disclaimers, preview scope, change logs, footnote markers, source lines, and method notes. Delegated writers put these notes in the step report for Ari's chat handoff.
- **Figure labels**: An unrequested qualifier describing where or how Ari produced a number (source system, reconstruction, or basis) is an authoring note. Labels identifying saved, preview, or estimated values, or naming a specific unresolved item beside its figure, stay on the page.
- **Removing notes**: When removing unrequested authoring notes, state their details in your chat reply. Remove that note kind across the page and subsequent edits. Requested report content follows the person's stated removal scope; delegated writers' briefs must name removed note kinds because writers know only their brief.
- **Figure definitions**: A figure named for a metric the person defined must use their definition whenever written. If the data cannot support it, name the figure for what it measures or leave it pending, and explain why in chat.
- **Direct manipulation at the mark**:
  - Combine the primary action and its financial consequence in the opening view. Make the business object its own control: drag an event along the time axis to shift a hire or contract start; drag a segment boundary to change an allocation; drag a trajectory endpoint to adjust a growth rate.
  - Selecting an object opens exact numeric/keyboard entry right beside that mark using its exact currency, unit, and precision. Avoid separate form panels or redundant summary strips that merely repeat the figure.
  - At rest, show one path labeled as both saved and preview; once a user adjusts a lever, keep the saved baseline path visible alongside the diverging preview path and label the difference where it matters most (such as cash low points or threshold crossings).
- **Preview vs. saved model**: Local JavaScript computes responsive preview paths from the model's relationships while historical periods stay fixed. User gestures change local preview state only (`Reset` returns to the saved baseline); saving changes to the financial model happens through modeling tools under user intent.
- **Typography, color, and scale**:
  - Use shared theme tokens, typography roles, tabular numerals, and injected `formatValue` / `formatCompactValue` helpers rather than custom font scales or formatters.
  - Use position and length for quantities, hue for identity (one hue per meaning), and lightness for magnitude. Pair every color cue with a position, shape, line-style, or label cue.
  - Color favorable/unfavorable deltas by business meaning rather than sign, and render zero differences as `"No change"` in neutral style. Keep axis ticks at round business numbers and lock axis scales during drag gestures.

## Designing a Full-Business Simulator ("A Business on One Page")

When building a primary business application, organize the opening view in one clear reading order:

1. **Live headline**: One sentence stating the business question and current outcome in the company's vocabulary (optionally with inline draggable inputs).
2. **Center backbone figure**: The primary outcome the user answers for over time, showing the saved baseline path, key milestones, and any critical threshold (such as minimum cash reserve) labeled at the crossing point.
3. **Interactive levers**: Placed beside or beneath the backbone on a shared time axis or scale, covering the highest-impact revenue, cost, and funding drivers. Next to each lever, plot a small sensitivity curve across its plausible range showing the current value and where the outcome crosses any threshold.
4. **Supporting detail**: Quieter context and methodology below the fold.

## Component Library, HTML Controls, and Host Events

Start by calling `inspect_canvas_library` (`read`) for the target visual components or templates. Its `runtime_contract` and the `source` parameter of `edit_canvas_blocks` define available props, theme tokens, state, and events:

| Capability                               | Rules and contracts                                                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Library components**                   | Use `Timeline` / `DecisionTimeline` for schedules, `EditableChart` for draggable points/bars, `BusinessDriver` for amounts, `EditableAllocation` for fixed-total splits, `InteractiveFunnel` for conversions, and `Chart` for comparisons. Combine with custom HTML/SVG when a single geometry must unify controls and outcomes.                     |
| **Templates (`kind: "template"`)**       | When templates are enabled, call `inspect_canvas_library` `list` (`kind: "template"`) then `read` the match and bind it to real model datasets. When disabled, build a standard chart or KPI.                                                                                                                                                        |
| **Top-level helper components**          | Declare helper React components at the top level outside `Block` and pass data via props (`edit_canvas_blocks` `create` refuses inner component definitions except on `source_template`).                                                                                                                                                            |
| **Standard HTML controls**               | Use native `<button>` (`data-variant="accent" \| "ghost" \| "destructive"`), `<input>`, `<select>`, `<fieldset>` radios (`data-variant="segmented"` for segmented tracks), `<input type="checkbox" role="switch">`, `<input type="range">` + `<output>`, `<textarea>`, and `<input type="file">` (read locally via `File.text()` / `arrayBuffer()`). |
| **In-app navigation (`navigate`)**       | Emit `navigate` from a click handler (never `<a href>`) using a tool-returned `url`, `runway:` reference, or workspace page path (`edit_canvas_blocks` `source`).                                                                                                                                                                                    |
| **Downloading saved files (`download`)** | Emit `download` with a file's `runway-file-` reference from a button's onClick; only the file's owner can download it.                                                                                                                                                                                                                               |
| **Walkthrough anchors**                  | Add stable `data-guide-anchor="<id>"` attributes to interactive controls and outcomes. On toggles that reveal hidden sections/tabs, add `data-guide-reveals="id1 id2"` and keep `aria-expanded` accurate.                                                                                                                                            |
| **Comment anchors**                      | Give meaningful sections, cards, and repeated items a unique `data-comment-id`. Preserve each ID while editing the same logical element. Derive repeated-item IDs from stable record identities, not array positions. Never reuse an ID for unrelated content.                                                                                       |
| **Starting Ari chats (`startAriChat`)**  | Add `data-ari-action` to buttons that trigger Ari work and call `emit({ type: 'startAriChat', message }, event)` synchronously in `onClick` (`edit_canvas_blocks` `source`).                                                                                                                                                                         |

## Canvas Chat Button Messages

- **Visible requests:** Every `startAriChat` message appears verbatim as the person's own words in the action's chat transcript; it has no hidden instruction channel. The message is the short business request plus relevant current inputs within the message's size limits. A request may describe the desired outcome in a few sentences.

- **Repeatable procedures:** When the work has a repeatable procedure, the message also names the saved skill or guide that holds it: "Save my Operations budget with the budget-save skill".

- **Saved instructions:** For a repeatable procedure, tool choices, execution steps, and approval rules live only in the saved skill or guide. Ari can create it before wiring the button. The visible request names that workflow without repeating its instructions.

## Dataset and Chart Binding Contract

Each declared dataset (`data.<name>`) evaluates before `Block` renders, so declare only datasets the active visual uses:

- **Time-series vs. record-list orientation**:
  - **Time-series charts**: Put variables (and breakdowns) in `layout.rows` and `Date` in `layout.columns`. `dataset.rows` holds one entry per time period (`xKey: "date"`), and `dataset.series` holds the value series (`key`, `label`, `variable`, `segments`). If `dataset_evidence` reports `period_count: 0` with dates in `series_names`, swap rows and columns.
  - **Record lists**: Put fields in `layout.rows` (using `useAs: "VARIABLE"` for mapped dimensions) and the record identity dimension matching the source grain in `layout.columns` with `NO_GRANULARITY`, validating mapped fields against source records so coarser keys do not collapse distinct item values. Each `dataset.rows` item is one record (`date` holds the record label, `row[entry.key]` holds each field value).
- **Avoid double-counting**: In `dataset.series`, empty `segments` represent the unsegmented parent variable and populated `segments` represent breakdown items. Never stack a parent total together with its own children.

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

- **Chart types**: Line, bar, and area charts share the `xKey: "date"` row layout; pie/donut charts slice a single row; waterfalls take signed deltas and explicit `totals` flags. Share a line plot only across comparable units and scales.

## Validation, Aesthetic Design Pass, and Updates

1. **Render and layout validation (`validate: true` or `validate_code_blocks`)**:
   - Inspect `readback` grids (`c1`…`cN` are readback-only labels), `plotted` series summaries, and responsive layout checks across phone and desktop viewports (`validate_code_blocks`).
   - Before the design pass, fix text-overlap or overflow warnings (using Grid `minColumnWidth`, wrapping, or intentional horizontal scroll) and rewrite any `"Design:"` warning.
2. **Aesthetic design pass (`visual_review`)**:
   - For interactive applications, request `visual_review` in `validate_code_blocks` with `view: "desktop-opening"` and a `brief` stating the intended hierarchy and user requirements.
   - Inspect the rendered image for visual hierarchy, alignment, direct editor placement on marks, and baseline comparison clarity.
   - If either assessment is unavailable (`visualReview.assessmentError` or no `visualReview.assessment`), stop the design pass and disclose the unavailable assessment in chat. Do not treat it as passed.
   - If `visualReview.assessment.revision` is null, finish the design pass. Optional `visualReview.assessment.refinement` does not trigger further edits or reviews.
   - For a non-null `visualReview.assessment.revision`, apply one repair pass while keeping all levers manipulable. Run `validate_code_blocks` with `visual_review` on the saved repair for exactly one confirming review, using the same target, view, and brief.
   - After that confirmation, stop the design pass regardless of its result. Do not make another repair or request another review. Disclose any remaining blocking defect or unavailable validation in chat; stopping does not mean it passed.
3. **Updating existing blocks (`edit_canvas_blocks` `update`)**:
   - Read the saved block with `inspect_canvas_blocks` first, then apply `source_patches` or a full `source` with `based_on` (`source_hash`).
