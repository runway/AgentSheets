# Donut Chart Guide

## Chart options shape

Same data shaping as the pie guide (latest row → one slice per variable), with a
donut series:

```jsx
function Block({ data }) {
  const ds = data.segmented || { rows: [], series: [] }; // "segmented" = the name YOU declared in datasets
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

## Contract notes

Donut charts use the same latest-row-to-slices shaping as pie charts. AG Charts also accepts
`innerLabels` when the requested design includes a value or label in the center.

## Common mistakes

- Feeding a full time series into it (pick one snapshot — see the pie guide)
