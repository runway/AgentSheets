# Stacked bar chart

## Options

Set `stacked: true` so each variable becomes one segment:

```jsx
function Block({ data }) {
  const ds = data.revenueByProduct || { rows: [], series: [] }; // Use the name declared in datasets.
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

## Rules

- Each dataset series becomes one segment.
- Add `normalizedTo: 100` for a 100% stack.

## Common mistakes

- Using one variable. Use a plain bar instead.
