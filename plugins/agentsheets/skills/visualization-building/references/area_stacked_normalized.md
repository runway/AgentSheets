# 100% Stacked Area Chart Guide

## Chart options shape

Add `normalizedTo: 100` to stacked area series:

```jsx
function Block({ data }) {
  const ds = data.revenueMix || { rows: [], series: [] }; // "revenueMix" = the name YOU declared in datasets
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
          normalizedTo: 100,
        })),
      }}
    />
  );
}
```

## Contract notes

The full stacked area is normalized to 100% at every period. At least two variables are needed for a
meaningful composition series.

## Common mistakes

- Using with only one variable (always 100%, useless)
- Omitting dateGranularity in the dataset (needs a time axis)
