# Bar chart

## Options

Dataset rows put x values under `date`. Keep `xKey: "date"` even when the
axis shows categories such as quarters:

```jsx
function Block({ data }) {
  const ds = data.revenue || { rows: [], series: [] }; // Use the name declared in datasets.
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

For several variables, derive bars from `ds.series`. Those rows use
`series[i].key`, not `value`:

```jsx
series: ds.series.map((s) => ({
  type: "bar",
  xKey: "date",
  yKey: s.key,
  yName: s.label,
})),
```

Set the comparison buckets with the dataset's `window` granularity, such as
`QUARTER` for “by quarter.” For one categorical snapshot, use a narrow window.

## Rules

- One monthly multi-series dataset can show a category breakdown over time.
- Stacked bars use the same data shape to show each bucket's composition.

## Common mistakes

- Using an `xKey` other than `date`. Rows have no other x field.
