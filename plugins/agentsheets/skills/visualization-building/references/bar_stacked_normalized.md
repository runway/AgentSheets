# 100% Stacked Bar Chart Guide

## Chart options shape

Add `normalizedTo: 100` to stacked bar series:

```jsx
function Block({ data }) {
  const ds = data.revenueMix || { rows: [], series: [] }; // "revenueMix" = the name YOU declared in datasets
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

## Contract notes

Every bucket is normalized to 100%, so the chart preserves relative composition rather than
absolute magnitude. At least two variables are needed for a meaningful normalized stack.

## Common mistakes

- Using with only one variable (bar is always 100%, useless)
