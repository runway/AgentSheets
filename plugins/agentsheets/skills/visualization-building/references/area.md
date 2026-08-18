# Area chart

## Options

Single variable (rows carry `value`):

```jsx
function Block({ data }) {
  const ds = data.expenses || { rows: [], series: [] }; // Use the name declared in datasets.
  return (
    <Chart
      options={{
        data: ds.rows,
        series: [{ type: "area", xKey: "date", yKey: "value" }],
      }}
    />
  );
}
```

For two variables, map the dataset's series. Multi-variable rows use
`series[i].key`, so `yKey: "value"` would render empty:

```jsx
series: ds.series.map((s) => ({
  type: "area",
  xKey: "date",
  yKey: s.key,
  yName: s.label,
}));
```

## Rules

Use stacked areas when several inputs add to one total. Plain area series overlap.

## Common mistakes

- Reading `value` from a multi-variable dataset. Only single-variable rows have it.
- Omitting `dateGranularity` for time-series data.
