# Line chart

## Options

Declare one dataset per chart. Put its range and granularity in `window`;
`MONTH` is most common. Replace `revenue` with the dataset's declared name:

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

For two or three variables, derive series from the dataset. Multi-variable rows
use `series[i].key`, not `value`:

```jsx
series: ds.series.map((s) => ({
  type: "line",
  xKey: "date",
  yKey: s.key,
  yName: s.label,
}));
```

## Rules

- Single-variable rows expose `value`; multi-variable rows use each `series[i].key`.
- When a chart and table show the same variables, use the same variable or dimension URIs.
- The branded theme supplies colors. Hardcoded colors may fail in other display modes.

## Common mistakes

- Omitting granularity from `window`. The chart will lack a proper time axis.
- Hardcoding variable-name y keys. Read them from `ds.series`.
