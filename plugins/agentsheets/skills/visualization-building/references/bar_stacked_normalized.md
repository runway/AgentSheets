# 100% stacked bar chart

## Options

Add `normalizedTo: 100` to stacked bar series:

```jsx
function Block({ data }) {
  const ds = data.revenueMix || { rows: [], series: [] }; // Use the name declared in datasets.
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
          normalizedTo: 100,
        })),
      }}
    />
  );
}
```

## Rules

Each bucket totals 100%, so the chart shows relative composition, not magnitude. Use at least two
variables.

## Common mistakes

- Using one variable. It will always show 100%.
