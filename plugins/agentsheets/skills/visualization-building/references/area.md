# Area Chart Guide

## Chart options shape

Single variable (rows carry `value`):

```jsx
function Block({ data }) {
  const ds = data.expenses || { rows: [], series: [] }; // "expenses" = the name YOU declared in datasets
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

For two variables, map the dataset's series instead — multi-variable rows are keyed
by `series[i].key`, so a lone `yKey: "value"` series would render empty:

```jsx
series: ds.series.map((s) => ({
  type: "area",
  xKey: "date",
  yKey: s.key,
  yName: s.label,
}));
```

## Contract notes

Use stacked area series when several inputs are meant to sum to one total. Plain multi-series area
charts overlap rather than add.

## Common mistakes

- Reading `value` from a multi-variable dataset (only single-variable rows carry it)
- Omitting dateGranularity in the dataset settings for time-series data
