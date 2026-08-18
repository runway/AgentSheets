# 100% stacked area chart

## Options

Add `normalizedTo: 100` to stacked area series:

```jsx
function Block({ data }) {
  const ds = data.revenueMix || { rows: [], series: [] }; // Use the name declared in datasets.
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
          normalizedTo: 100,
        })),
      }}
    />
  );
}
```

## Rules

Each period totals 100%. Use at least two variables.

## Common mistakes

- Using one variable. It will always show 100%.
- Omitting `dateGranularity`. The chart needs a time axis.
