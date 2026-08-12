# Pie Chart Guide

## Chart options shape

A pie shows one point in time. Prefer a dataset that resolves to a single row
(narrow dateRange in `table_config.settings`); if the dataset still carries
multiple periods, take the LATEST row — silently charting the first row shows
stale composition:

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
        series: [{ type: "pie", angleKey: "value", legendItemKey: "label" }],
      }}
    />
  );
}
```

## Common mistakes

- Feeding a full time series into a pie (pick one snapshot — usually the latest)
