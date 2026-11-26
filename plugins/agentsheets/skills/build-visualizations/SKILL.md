---
name: build-visualizations
description: "Use when building or reviewing Canvas visuals and interactive apps: chart types, direct manipulation, datasets, and visual review."
---

# Build visualizations

A Canvas block is an interactive React visual bound to live model datasets (`[[build-model]]` owns native tables). Scope to the question: a focused decision gets a focused simulator; a full model build ends with a system-wide simulator of the primary outcome over time and its top levers.

## Design Principles

- **Page copy:** Keep app-page prose to 1–2 lines; reports include only requested figures, narrative, sources, qualifications, or methods (report-skill layouts count as requested).
- **Authoring notes:** Keep unrequested notes (coverage, tie-out status, estimate disclaimers, preview scope, change logs, footnotes, provenance) in chat or step reports; keep only saved/preview/estimated labels or unresolved-item flags on the page. If asked to remove unrequested notes, move them to chat, strip that kind across the page and future edits, and list removed kinds in delegated briefs. Requested report content follows the stated removal scope.
- **User-defined metrics:** Use the user's definition; if data cannot support it, rename or defer and explain in chat.
- **Direct manipulation:**
  - Put primary controls and their financial consequence in the opening view, making business objects their own controls (drag events along time, segment boundaries for allocations, endpoints for growth).
  - Selecting a mark opens inline numeric/keyboard entry with exact currency, unit, and precision. Avoid separate form panels or redundant KPI strips.
  - At rest, show one path labeled saved + preview; once adjusted, keep the saved baseline beside preview and label key shifts (cash troughs, threshold crossings).
- **Preview vs. saved:** Local JS computes live preview while history stays fixed; gestures change preview only (`Reset` restores baseline). Model writes happen only via modeling tools under user intent.
- **Theme and color:** Use injected theme tokens, typography roles, tabular numerals, and `formatValue`/`formatCompactValue`. Encode quantity by position/length, identity by hue (1 hue per meaning), magnitude by lightness paired with position/shape/dash/label. Color deltas by business favorability (not sign), show zero as `"No change"`, use round ticks, and lock axis scales while dragging.

## Full-Business Simulator Layout

Order the opening view top to bottom:

1. **Live headline:** One sentence stating the business question and outcome in company terms (optional inline draggable inputs).
2. **Primary outcome chart:** Main outcome over time with saved baseline, milestones, and labeled threshold crossings (such as min cash).
3. **Interactive levers:** Beside or beneath on a shared axis covering top revenue, cost, and funding levers with a small sensitivity curve showing current value and threshold crossing.
4. **Supporting detail:** Lower-priority context below the fold.

## Components, Controls, and Host Events

Inspect target components and templates via `inspect_canvas_library` (`read`) first:

- **Library components:** `Timeline`/`DecisionTimeline` (schedules), `EditableChart` (draggable points/bars), `BusinessDriver` (amounts), `EditableAllocation` (fixed-total splits), `InteractiveFunnel` (conversions), `Chart` (comparisons); combine with HTML/SVG when one geometry must unify controls and outcomes.
- **Templates:** When enabled, list `kind: "template"` in `inspect_canvas_library`, read a match, and bind real datasets; otherwise build a standard chart/KPI.
- **Top-level helpers:** Define helper components outside `Block` and pass props (`create` rejects inner definitions except on `source_template`).
- **HTML controls:** Native `<button>` (`data-variant="accent" | "ghost" | "destructive"`), `<input>`, `<select>`, `<fieldset>` radios (`data-variant="segmented"`), `<input type="checkbox" role="switch">`, `<input type="range">` + `<output>`, `<textarea>`, `<input type="file">` (local `File.text()`/`arrayBuffer()`).
- **Navigation and downloads:** Emit `navigate` on click (never `<a href>`) with a tool URL, `runway:` ref, or page path. Emit `download` with a `runway-file-` ref on click (file owner only).
- **Guide and comment anchors:** Add stable `data-guide-anchor="<id>"` on controls/outcomes (`data-guide-reveals="id1 id2"` + `aria-expanded` on reveals). Add unique, stable `data-comment-id` on meaningful sections/cards/items keyed by record ID (never array index; preserve across edits; never reuse for unrelated content).
- **Editable text:** In page Canvas blocks (not saved emails or forms), add block-unique `editable="<name>"` on `Text` for headings, titles, and short prose: `<Text variant="heading" editable="headline">Revenue overview</Text>`. Children must be literal source text; page editors click or tap to edit, saving their wording into the source. When someone asks to reword text themselves, mark it editable and tell them to click it.
- **Ari chat buttons (`startAriChat`):** Add `data-ari-action` and call `emit({ type: 'startAriChat', message }, event)` synchronously in `onClick`. Write a concise business prompt with current inputs (appears verbatim as the user's message); for repeatable workflows, name the saved skill/guide without pasting procedures.

## Dataset and Chart Binding

Declare only datasets the block uses (every `data.<name>` evaluates before render):

- **Time-series:** Variables/breakdowns in `layout.rows`, `Date` in `layout.columns`. `dataset.rows` holds periods (`xKey: "date"`); `dataset.series` holds `{ key, label, variable, segments }`. If `dataset_evidence` has `period_count: 0` with dates in `series_names`, swap rows and columns.
- **Record lists:** Fields in `layout.rows` (`useAs: "VARIABLE"` for mapped dimensions), record-ID dimension at source grain in `layout.columns` with `NO_GRANULARITY` (verify coarser keys do not collapse rows). Each `dataset.rows` entry is one record (`date` = label, `row[entry.key]` = value).
- **Audience, export, and double-counting:** The page's Excel download (Download as Excel) includes a values sheet for each saved Canvas dataset the page can show that reader. Each sheet's rows and columns follow the dataset's declared layout. If a block lets the reader pick periods, emit `shownWindows` on first render and when the pick changes (for example, from a `React.useEffect` on the picked period) so the sheet's period columns keep only those periods. Time entries listed before the variables on the columns order that sheet period by period. Restrict sensitive rows via dataset `include` (code filters hide display only). Never stack unsegmented parents (`segments: []`) with segmented children.
- **Chart types:** Line/bar/area use `xKey: "date"`; pie/donut slice one row; waterfall takes signed deltas + `totals`. Share axes only across comparable units/scales. Check `dataset.error` before rendering:

```jsx
function Block({ data }) {
  const ds = data.revenue || { rows: [], series: [] };
  if (ds.error) return <Text variant="soft">{ds.error}</Text>;
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

## Validation, Visual Review, and Updates

1. **Render validation:** Pass `validate: true` or call `validate_code_blocks`; inspect `readback` (`c1`–`cN` are readback-only), `plotted` min/max, and 360–1400 px warnings. Fix overlaps, overflows (Grid `minColumnWidth`, wrapping, horizontal scroll), and `"Design:"` warnings before visual review. For hidden tabs/views, pass `visual_review` with `click` set to the tab's `data-guide-anchor` ID and check `visualReview.clickFailure`.
2. **Bounded design pass (`visual_review`):** For interactive apps, call `validate_code_blocks` with `view: "desktop-opening"` and a `brief` of intended hierarchy and constraints. Check hierarchy, alignment, direct editor placement, and baseline clarity:
   - Missing/errored assessment (`assessmentError` or absent `assessment`): stop and disclose in chat; never treat as passed.
   - `assessment.revision` null: stop (`assessment.refinement` triggers no extra edits).
   - `assessment.revision` non-null: apply **one** repair pass keeping all levers manipulable (after data-only edits, fix only defects caused by that change), then run **one** confirming `visual_review` with the same target, view, and brief. Stop after confirmation regardless of outcome and disclose any remaining blocking defect or unavailable check.
3. **Updates:** Read via `inspect_canvas_blocks` first; use `source_patches` + `datasets` for targeted updates and full `source` only for redesigns.
