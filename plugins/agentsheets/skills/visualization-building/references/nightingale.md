# Nightingale Chart Guide

## Chart options shape

Nightingale is an enterprise series the injected `Chart` supports out of the box:

```jsx
function Block({ data }) {
  const ds = data.variables || { rows: [], series: [] }; // "variables" = the exact name YOU declared in datasets
  return (
    <Chart
      options={{
        data: ds.rows,
        series: [{ type: "nightingale", angleKey: "date", radiusKey: "value" }],
      }}
    />
  );
}
```
