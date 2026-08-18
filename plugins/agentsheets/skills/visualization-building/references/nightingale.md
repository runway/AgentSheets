# Nightingale chart

## Options

The injected `Chart` supports the enterprise Nightingale series:

```jsx
function Block({ data }) {
  const ds = data.variables || { rows: [], series: [] }; // Use the exact name declared in datasets.
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
