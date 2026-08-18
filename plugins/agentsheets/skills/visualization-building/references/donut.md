# Donut chart

## Options

Shape data as for a pie: use the latest row and make one slice per variable.

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
        series: [
          {
            type: "donut",
            angleKey: "value",
            legendItemKey: "label",
            innerRadiusRatio: 0.6,
          },
        ],
      }}
    />
  );
}
```

## Rules

AG Charts accepts `innerLabels` for a value or label in the center.

## Common mistakes

- Passing a full time series. Pick one snapshot; see the pie guide.
