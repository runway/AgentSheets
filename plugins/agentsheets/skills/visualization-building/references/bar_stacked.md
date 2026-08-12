# Stacked Bar Chart Guide

## Chart options shape

Each variable in the dataset becomes a stack segment via `stacked: true`:

```jsx
function Block({ data }) {
  const ds = data.revenueByProduct || { rows: [], series: [] }; // "revenueByProduct" = the name YOU declared in datasets
  return (
    <Chart
      options={{
        data: ds.rows,
        series: ds.series.map((s) => ({
          type: "bar",
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

## Contract notes

- Every dataset series becomes one stack segment.
- Adding `normalizedTo: 100` to the stacked series produces the normalized variant.

## Common mistakes

- Using only one variable (no stacking visible — use a plain bar instead)
