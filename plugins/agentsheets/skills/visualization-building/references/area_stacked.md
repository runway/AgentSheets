# Stacked Area Chart Guide

## Chart options shape

```jsx
function Block({ data }) {
  const ds = data.revenueByChannel || { rows: [], series: [] }; // "revenueByChannel" = the name YOU declared in datasets
  return (
    <Chart
      options={{
        data: ds.rows,
        series: ds.series.map((s) => ({
          type: "area",
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

The upper edge of the stack is the total across all series for each period; each filled band is one
series' contribution.

## Common mistakes

- Omitting dateGranularity in the dataset (stacked areas need a time axis)
- Using only one variable leaves no visible composition in the stack
