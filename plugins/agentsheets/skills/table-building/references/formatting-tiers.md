## Formatting tiers for a financial statement

For a P&L, scannable formatting looks like this:

- **Tier 1 — Detail lines** (e.g. Subscription revenue): indented, no color background, no
  bolding.
- **Tier 2 — Structural totals** (e.g. Total revenue, Total cost of revenue, Total operating
  expenses): don't indent (flush-left), bold, no background.
- **Tier 3 — Mid-tier subtotals**, where one exists (e.g. Total non-headcount expense within the
  broader operating expense section): indented + bold, no color background.
- **Tier 4 — Calculated milestones** (e.g. Gross profit, Operating profit/(loss), Net
  profit/(loss)): flush-left, bold, background color.
- **Tier 5 — % variables** (e.g. Gross margin %, and Op margin % / Net margin % if you add them):
  italic, not bold, with a named fill distinct from tier 4's, aligned flush-left.

Note: within one artifact, all tier 4 lines share one named fill and all tier 5 lines share a
different one. The fill palette is a closed name set with no lightness control.
Styles are written the same way the table's own menus set them; the table-building manual teaches
the formatting write surface and its batching constraints.
