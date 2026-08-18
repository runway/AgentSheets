# Combo chart

## Options

Give each series its own `type`. Read y keys from `ds.series`; rows use generated
series keys, not variable names. A hardcoded key such as `yKey: "revenue"`
renders empty:

```jsx
function Block({ data }) {
  const ds = data.revenueVsMargin || { rows: [], series: [] }; // Use the name declared in datasets.
  return (
    <Chart
      options={{
        data: ds.rows,
        series: ds.series.map((s, i) => ({
          type: i === 0 ? "bar" : "line", // declare the bar variable first in the dataset
          xKey: "date",
          yKey: s.key,
          yName: s.label,
        })),
      }}
    />
  );
}
```

## Rules

- The example uses a bar for the first variable or dimension and lines for the rest. Control this
  with dataset order or `s.label`.
- AG Charts `axes` with `keys` can place differently scaled series on separate axes.

## Common mistakes

- Hardcoding variable-name y keys instead of using `ds.series[i].key`.
- Giving every series the same type. That is a plain line or bar chart.
