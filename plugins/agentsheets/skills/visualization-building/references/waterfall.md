# Waterfall Chart Guide

## Chart options shape

Waterfall is an enterprise series the injected `Chart` supports out of the box.
Use a single-variable dataset (rows carry `value`) whose values are already the
bridge DELTAS (e.g. New / Expansion / Churn) — a raw level series like
100, 120 renders as +100, +120, not as a start of 100 plus +20; derive
differences first if all you have is levels. Map the rows into ordered steps
and declare the ending total with `totals` — without it every row renders as
another delta, not a total:

```jsx
function Block({ data }) {
  const ds = data.arrBridge || { rows: [], series: [] }; // "arrBridge" = the name YOU declared in datasets
  const steps = ds.rows.map((r) => ({ step: r.date, value: r.value || 0 }));
  return (
    <Chart
      options={{
        data: steps,
        series: [
          {
            type: "waterfall",
            xKey: "step",
            yKey: "value",
            // guard: an empty dataset must not produce totals with index -1
            totals: steps.length
              ? [
                  {
                    totalType: "total",
                    index: steps.length - 1,
                    axisLabel: "Total",
                  },
                ]
              : [],
          },
        ],
      }}
    />
  );
}
```

## Contract notes

- Waterfall rows are ordered deltas in a single-variable dataset; multi-variable rows do not expose
  `value`.
- Positive, negative, and total colors come from the branded theme. If an override is required,
  use the corresponding theme tokens rather than literal colors.

## Common mistakes

- Omitting `totals` (the ending value renders as one more delta instead of a total)
- Unordered steps (the sequence is the story — build the rows in bridge order)
