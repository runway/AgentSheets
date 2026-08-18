# Waterfall chart

## Options

The injected `Chart` supports the enterprise waterfall series. Use a
single-variable dataset whose `value` rows are already bridge changes, such as
New, Expansion, and Churn. Levels such as 100 and 120 render as +100 and +120,
not 100 then +20, so calculate changes first. Keep steps in order and declare
the ending total with `totals`; otherwise it renders as another change:

```jsx
function Block({ data }) {
  const ds = data.arrBridge || { rows: [], series: [] }; // Use the name declared in datasets.
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

## Rules

- Waterfall rows are ordered changes in one variable. Multi-variable rows have no `value`.
- Positive, negative, and total colors come from the branded theme. If an override is required,
  use the corresponding theme tokens rather than literal colors.

## Common mistakes

- Omitting `totals`. The ending value renders as another change.
- Using unordered steps. Build rows in bridge order.
