# Bar Chart Guide

## Chart options shape

Dataset rows always expose the x values under `date` — keep `xKey: "date"`
even when the axis reads as categories (e.g. quarters):

```jsx
function Block({ data }) {
  const ds = data.revenue || { rows: [], series: [] }; // "revenue" = the name YOU declared in datasets
  return (
    <Chart
      options={{
        data: ds.rows,
        series: [{ type: "bar", xKey: "date", yKey: "value" }],
      }}
    />
  );
}
```

For a multi-variable dataset, derive the bars from `ds.series` (rows are keyed
by `series[i].key`, not `value`):

```jsx
series: ds.series.map((s) => ({
  type: "bar",
  xKey: "date",
  yKey: s.key,
  yName: s.label,
})),
```

Pick the comparison buckets with the dataset's
`table_config.settings.dateGranularity` (e.g. QUARTER for "by quarter"); for a
single-point categorical comparison use a narrow
`table_config.settings.dateRange` instead of fine granularity.

## Contract notes

- One monthly multi-series dataset can represent a variable broken out by category over time.
- Stacked bars use the same dataset shape when composition within each bucket is required.

## Common mistakes

- Renaming `xKey` to something other than `date` — rows don't carry other x fields
