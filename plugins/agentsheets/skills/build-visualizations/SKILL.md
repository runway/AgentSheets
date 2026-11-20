---
name: build-visualizations
description: "Use when building Canvas visuals or interactive apps: choosing chart types, direct manipulation, binding datasets, or visual review."
---

# Build visualizations

A Canvas block is an interactive React visual bound to live model datasets (`[[build-model]]` owns native tables). Scope it to the question: a focused decision gets a focused simulator; a full model build closes with a system-wide simulator showing the primary outcome over time and the few levers that move it most.

## Design Principles

- **Page copy:** Keep application page prose to 1–2 lines on purpose; reports include only requested figures, narrative, sources, qualifications, or methods (report-skill layouts count as requested).
- **Authoring notes:** Keep unrequested notes (coverage, tie-out status, estimate disclaimers, preview scope, change logs, footnotes, provenance) in chat or step reports; keep only saved/preview/estimated labels or unresolved-item flags on the page. When asked to remove unrequested notes, move details to chat, strip that kind across the page and future edits, and list removed kinds in delegated briefs. Requested report content follows the user's stated removal scope.
- **User-defined metrics:** Use the user's definition; if data cannot support it, rename or defer and explain in chat.
- **Direct manipulation on charts:**
  - Put primary controls and their financial consequence in the opening view, making business objects their own controls (drag events along time, drag segment boundaries for allocations, drag endpoints for growth).
  - Selecting a mark opens inline numeric or keyboard entry with exact currency, unit, and precision. Avoid separate form panels or redundant KPI strips.
  - At rest, show one path labeled saved plus preview; once adjusted, keep the saved baseline beside the preview path and label key differences (cash troughs, threshold crossings).
- **Preview vs. saved model:** Local JS computes live preview paths while historical periods stay fixed; gestures change preview state only (`Reset` restores baseline). Model writes happen only via modeling tools under user intent.
- **Theme and color:** Use injected theme tokens, typography roles, tabular numerals, and `formatValue`/`formatCompactValue`. Encode quantity by position/length, identity by hue (1 hue per meaning), and magnitude by lightness, always paired with position, shape, dash, or label. Color deltas by business favorability (not sign), show zero as `"No change"`, use round axis ticks, and lock axis scales while dragging.

## Full-Business Simulator Layout

Organize the opening view in one reading order:

1. **Live headline:** One sentence stating the business question and outcome in company terms (with optional inline draggable inputs).
2. **Primary outcome chart:** Main outcome over time with saved baseline, milestones, and labeled threshold crossings (such as minimum cash).
3. **Interactive levers:** Beside or beneath on a shared axis, covering top revenue, cost, and funding levers with a small sensitivity curve showing current value and threshold crossing.
4. **Supporting detail:** Lower-priority context below the fold.

## Components, Controls, and Host Events

Inspect target components and templates via `inspect_canvas_library` (`read`) first:

- **Library components:** Use `Timeline`/`DecisionTimeline` (schedules), `EditableChart` (draggable points or bars), `BusinessDriver` (amounts), `EditableAllocation` (fixed-total splits), `InteractiveFunnel` (conversions), and `Chart` (comparisons); combine with HTML/SVG when one geometry must unify controls and outcomes.
- **Templates:** When enabled, list `kind: "template"` in `inspect_canvas_library`, read a match, and bind real datasets; otherwise build a standard chart or KPI.
- **Top-level helpers:** Define helper components outside `Block` and pass props (`create` rejects inner definitions except on `source_template`).
- **HTML controls:** Use native `<button>` (`data-variant="accent" | "ghost" | "destructive"`), `<input>`, `<select>`, `<fieldset>` radios (`data-variant="segmented"`), `<input type="checkbox" role="switch">`, `<input type="range">` with `<output>`, `<textarea>`, and `<input type="file">` (local `File.text()`/`arrayBuffer()`).
- **Navigation and downloads:** Emit `navigate` on click (never `<a href>`) with a tool URL, `runway:` ref, or page path. Emit `download` with a `runway-file-` ref on click (file owner only).
- **Guide and comment anchors:** Add stable `data-guide-anchor="<id>"` on controls and outcomes (`data-guide-reveals="id1 id2"` with `aria-expanded` on reveals). Add unique, stable `data-comment-id` on meaningful sections, cards, or items keyed by record ID (never array index; preserve across edits; never reuse for unrelated content).
- **Ari chat buttons (`startAriChat`):** Add `data-ari-action` and call `emit({ type: 'startAriChat', message }, event)` synchronously in `onClick`. Write a concise business request with current inputs (it appears verbatim as the user's prompt); for repeatable workflows, name the saved skill or guide (`"Save my Operations budget with the budget-save skill"`) without pasting procedures or rules.

## Dataset and Chart Binding

Declare only datasets the block actively uses (every declared `data.<name>` evaluates before render):

- **Time-series:** Put variables/breakdowns in `layout.rows` and `Date` in `layout.columns`. `dataset.rows` holds periods (`xKey: "date"`); `dataset.series` holds `{ key, label, variable, segments }`. If `dataset_evidence` shows `period_count: 0` with dates in `series_names`, swap rows and columns.
- **Record lists:** Put fields in `layout.rows` (`useAs: "VARIABLE"` for mapped dimensions) and record-ID dimension at source grain in `layout.columns` with `NO_GRANULARITY` (verify coarser keys do not collapse distinct rows). Each `dataset.rows` entry is one record (`date` is label, `row[entry.key]` is value).
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

- **Chart types:** Line, bar, and area use `xKey: "date"`; pie/donut slice one row; waterfall takes signed deltas and `totals`. Share axes only across comparable units and scales.

## Validation, Visual Review, and Updates

1. **Render validation:** Run `validate: true` or `validate_code_blocks`; check `readback` (`c1`–`cN` are readback-only), `plotted` min/max values, and 360–1400 px warnings. Fix overlaps, overflows (Grid `minColumnWidth`, wrapping, or horizontal scroll), and `"Design:"` warnings before visual review. For hidden tabs/views, pass `visual_review` with `click` set to the tab's `data-guide-anchor` ID and check `visualReview.clickFailure`.
2. **Bounded design pass (`visual_review`):** For interactive apps, call `validate_code_blocks` with `view: "desktop-opening"` and a `brief` stating intended hierarchy and user constraints. Inspect the rendered image for hierarchy, alignment, direct editor placement on marks, and baseline comparison clarity:
   - Missing or errored assessment (`assessmentError` or absent `assessment`): stop and disclose in chat; never treat as passed.
   - `assessment.revision` is null: stop (`assessment.refinement` triggers no extra edits).
   - `assessment.revision` is non-null: apply **one** repair pass keeping all levers manipulable (after data-only edits, fix only defects caused by that change), then run **one** confirming `visual_review` with the same target, view, and brief. Stop after confirmation regardless of outcome and disclose any remaining blocking defect or unavailable check in chat.
3. **Updating blocks:** Read via `inspect_canvas_blocks` first; use `source_patches` with `datasets` for targeted updates and full `source` only for redesigns.
