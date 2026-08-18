# Stacked area chart

## Options

```jsx
function Block({ data }) {
  const ds = data.revenueByChannel || { rows: [], series: [] }; // Use the name declared in datasets.
  return (
    <Chart
      options={{
        data: ds.rows,
        series: ds.series.map((s) => ({
          type: "area",
          xKey: "date",
          yKey: s.key,
          yName: s.label,
          stacked: true,
        })),
      }}
    />
  );
}
```

## Rules

The stack's upper edge is the period total. Each band is one series' share.

## Common mistakes

- Omitting `dateGranularity`. Stacked areas need a time axis.
- Using one variable. A stack needs at least two.
