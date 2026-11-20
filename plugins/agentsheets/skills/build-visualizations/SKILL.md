---
name: build-visualizations
description: "Use when building Canvas visuals or interactive apps: choosing chart types, direct manipulation, binding datasets, or visual review."
---

# Build visualizations

A Canvas block is an interactive React visual bound to live model datasets (`[[build-model]]` owns native tables). Scope it to the question: a focused decision gets a focused simulator; a full model build closes with a system-wide simulator showing the primary outcome over time and the few levers that move it most.

## Design Principles

- **Page copy and authoring notes:** Keep application page prose to 1–2 lines on purpose; reports include only requested figures, narrative, sources, qualifications, or methods (report-skill layouts count as requested). Keep unrequested authoring notes (coverage, tie-out status, estimate disclaimers, preview scope, change logs, footnotes, source/method notes, provenance qualifiers) in chat or step reports — only labels marking saved, preview, or estimated values, or naming an unresolved item, stay on the page. When asked to remove unrequested notes, move details to chat, strip that kind across the page and future edits, and list removed kinds in delegated briefs. Requested report content follows the user's stated removal scope.
- **User-defined metrics:** Use the user's definition; if data cannot support it, rename or defer and explain in chat.
- **Direct manipulation at the mark:**
  - Put primary controls and their financial consequence in the opening view, and make business objects their own controls (drag events along time to shift dates, drag segment boundaries for allocations, drag endpoints for growth).
  - Selecting a mark opens inline numeric or keyboard entry with exact currency, unit, and precision. Avoid separate form panels or redundant KPI strips.
  - At rest, show one path labeled saved plus preview; once adjusted, keep the saved baseline visible beside the preview path and label key differences (cash troughs, threshold crossings).
- **Preview vs. saved model:** Local JS computes live preview paths while historical periods stay fixed; gestures change local preview state only (`Reset` restores baseline). Model writes happen only via modeling tools under user intent.
- **Theme and formatting:** Use injected theme tokens, typography roles, tabular numerals, and `formatValue` or `formatCompactValue`.
- **Color and scale:** Encode quantity by position or length, identity by hue (1 hue per meaning), and magnitude by lightness, pairing every color cue with position, shape, dash, or label. Color deltas by business favorability (not sign), show zero as `"No change"`, use round axis ticks, and lock axis scales while dragging.

## Full-Business Simulator Layout

Organize the opening view in one reading order:

1. **Live headline:** One sentence stating the business question and outcome in company terms (with optional inline draggable inputs).
2. **Backbone figure:** Primary outcome over time with saved baseline, milestones, and labeled threshold crossings (such as minimum cash).
3. **Interactive levers:** Beside or beneath on a shared axis, covering top revenue, cost, and funding levers with a small sensitivity curve showing current value and threshold crossing.
4. **Supporting detail:** Quieter context below the fold.

## Components, Controls, and Host Events

Call `inspect_canvas_library` (`read`) for target components and templates first (`runtime_contract` and `edit_canvas_blocks` `source` define props, tokens, state, and events):

| Capability                            | Rules                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Library components**                | `Timeline` or `DecisionTimeline` (schedules), `EditableChart` (draggable points or bars), `BusinessDriver` (amounts), `EditableAllocation` (fixed-total splits), `InteractiveFunnel` (conversions), `Chart` (comparisons); combine with HTML or SVG when one geometry must unify controls and outcomes.                                                                            |
| **Templates (`kind: "template"`)**    | When enabled, call `inspect_canvas_library` `list` (`kind: "template"`), read a match, and bind real datasets; otherwise build a standard chart or KPI.                                                                                                                                                                                                                            |
| **Top-level helpers**                 | Define helper components outside `Block` and pass props (`create` refuses inner definitions except on `source_template`).                                                                                                                                                                                                                                                          |
| **HTML controls**                     | Native `<button>` (`data-variant="accent" \| "ghost" \| "destructive"`), `<input>`, `<select>`, `<fieldset>` radios (`data-variant="segmented"`), `<input type="checkbox" role="switch">`, `<input type="range">` with `<output>`, `<textarea>`, and `<input type="file">` (local `File.text()` or `arrayBuffer()`).                                                               |
| **Navigation and downloads**          | Emit `navigate` on click (never `<a href>`) with a tool URL, `runway:` ref, or page path. Emit `download` with a `runway-file-` ref on click (file owner only).                                                                                                                                                                                                                    |
| **Guide and comment anchors**         | Add stable `data-guide-anchor="<id>"` on interactive controls and outcomes (`data-guide-reveals="id1 id2"` with `aria-expanded` on reveals). Add unique, stable `data-comment-id` on meaningful sections, cards, or items keyed by record ID (never array index; preserve across edits; never reuse for unrelated content).                                                        |
| **Ari chat buttons (`startAriChat`)** | Add `data-ari-action` and call `emit({ type: 'startAriChat', message }, event)` synchronously in `onClick`. Because `message` appears verbatim as the user's prompt, write a concise business request with current inputs; for repeatable workflows, name the saved skill or guide (`"Save my Operations budget with the budget-save skill"`) without pasting procedures or rules. |

## Dataset and Chart Binding

Declare only datasets the block actively uses (every declared `data.<name>` evaluates before render):

- **Time-series:** Variables and breakdowns in `layout.rows`, `Date` in `layout.columns`. `dataset.rows` holds periods (`xKey: "date"`); `dataset.series` holds `{ key, label, variable, segments }`. If `dataset_evidence` shows `period_count: 0` with dates in `series_names`, swap rows and columns.
- **Record lists:** Fields in `layout.rows` (`useAs: "VARIABLE"` for mapped dimensions), record-ID dimension at source grain in `layout.columns` with `NO_GRANULARITY` (verify coarser keys do not collapse distinct rows). Each `dataset.rows` entry is one record (`date` is label, `row[entry.key]` is value).
- **Audience and double-counting:** Restrict sensitive rows via dataset `include` rather than code filters (code filters hide display only). Never stack unsegmented parent series (`segments: []`) with their segmented children.

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

- **Chart types:** Line, bar, and area use `xKey: "date"`; pie and donut slice one row; waterfall takes signed deltas and `totals`. Share axes only across comparable units and scales.

## Validation, Aesthetic Review, and Updates

1. **Render validation (`validate: true` or `validate_code_blocks`):** Check `readback` (`c1` to `cN` are readback-only), `plotted` min and max values, and 360 to 1400 px warnings. Fix overlaps, overflows (Grid `minColumnWidth`, wrapping, or intentional horizontal scroll), and `"Design:"` warnings before visual review.
   - **Hidden tabs or views:** Pass `visual_review` with `click` set to the tab's `data-guide-anchor` ID, and check `visualReview.clickFailure` if the click fails.
2. **Bounded design pass (`visual_review`):** For interactive apps, run `validate_code_blocks` with `view: "desktop-opening"` and a `brief` stating intended hierarchy and user constraints. Inspect the rendered image for visual hierarchy, alignment, direct editor placement on marks, and baseline comparison clarity:
   - If the assessment is missing or errored (`assessmentError` or no `assessment`), stop and disclose it in chat; never treat it as passed.
   - If `assessment.revision` is null, stop (`assessment.refinement` triggers no extra edits).
   - If `assessment.revision` is non-null, apply **one** repair pass keeping all levers manipulable (after data-only edits, fix only defects caused by that change), then run **one** confirming `visual_review` with the same target, view, and brief. Stop after that confirmation regardless of outcome and disclose any remaining blocking defect or unavailable check in chat.
3. **Updating blocks (`edit_canvas_blocks` `update`):** Read via `inspect_canvas_blocks` first. Use `source_patches` with `datasets` for data updates, and full `source` only for redesigns.
