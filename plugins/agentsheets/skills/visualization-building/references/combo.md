# Combo Chart Guide

## Chart options shape

Give each series its own `type` — this is what makes it a combo. Derive the
yKeys from `ds.series` (dataset rows are keyed by generated series keys, NOT by
variable names, so hardcoded keys like `yKey: "revenue"` render empty):

```jsx
function Block({ data }) {
  const ds = data.revenueVsMargin || { rows: [], series: [] }; // "revenueVsMargin" = the name YOU declared in datasets
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

## Contract notes

- The example assigns the first dataset variable or dimension to bars and later variables and dimensions to lines. Control
  that mapping through variable or dimension order or by selecting on `s.label`.
- AG Charts `axes` with `keys` can place differently scaled series on separate axes.

## Common mistakes

- Hardcoding variable-name yKeys instead of reading `ds.series[i].key`
- Giving every series the same type (that's just a line or bar chart — defeats the purpose)
