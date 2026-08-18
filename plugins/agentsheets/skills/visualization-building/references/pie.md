# Pie chart

## Options

A pie shows one point in time. Prefer a narrow `window` that returns one row.
If several periods remain, use the latest row. The first row may be stale:

```jsx
function Block({ data }) {
  const ds = data.segmented || { rows: [], series: [] }; // Use the name declared in datasets.
  const row = ds.rows[ds.rows.length - 1] || {};
  const slices = ds.series.map((s) => ({
    label: s.label,
    value: row[s.key] || 0,
  }));
  return (
    <Chart
      options={{
        data: slices,
        series: [{ type: "pie", angleKey: "value", legendItemKey: "label" }],
      }}
    />
  );
}
```

## Common mistakes

- Passing a full time series. Pick one snapshot, usually the latest.
