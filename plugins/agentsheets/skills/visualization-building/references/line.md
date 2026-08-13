# Line Chart Guide

## Chart options shape

Declare one dataset per chart (granularity and range live in the dataset's
`window`; `MONTH` is the most common granularity). The dataset
key below (`revenue`) is the `name` you declared in `datasets` — substitute yours:

```jsx
function Block({ data }) {
  const ds = data.revenue || { rows: [], series: [] };
  return (
    <Chart
      options={{
        data: ds.rows,
        series: [{ type: "line", xKey: "date", yKey: "value" }],
      }}
    />
  );
}
```

For 2-3 variables in one dataset, derive the series from the dataset instead of
inventing keys — multi-variable rows are keyed by `series[i].key`, not `value`:

```jsx
series: ds.series.map((s) => ({
  type: "line",
  xKey: "date",
  yKey: s.key,
  yName: s.label,
}));
```

## Contract notes

- Single-variable rows expose `value`; multi-variable rows use each `series[i].key`.
- Dataset variable or dimension URIs should match the corresponding table's variable rows when the visual and
  table show the same variables.
- The injected branded theme supplies series colors; hardcoded colors are neither required nor
  portable across display modes.

## Common mistakes

- Forgetting the granularity in the dataset's `window` (no proper time axis)
- Hardcoding variable-name yKeys (`yKey: "revenue"`) — read keys from `ds.series`
