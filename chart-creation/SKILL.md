---
name: chart-creation
description: Writing Observable Plot chart code for the OP App canvas. Use this skill whenever creating a chart visualization, writing Plot code, or when the user asks for any kind of chart, graph, or visual data representation. This skill provides the rendering environment details, theme constraints, and working patterns — the agent writes JS code and passes it to the create_chart tool (the code is NOT executed here, it runs on the frontend).
---

# Observable Plot Chart Creation

Write JavaScript chart code for the OP App canvas. The code is passed as a string to the `create_chart` tool and executed on the frontend — do NOT attempt to run it yourself.

## Rendering Environment

Your code runs inside: `new Function('Plot', 'd3', 'DATA', 'width', 'height', code)`

| Global | Type | Description |
|--------|------|-------------|
| `Plot` | Observable Plot | Full Plot library (wrapped — enforces min margins, injects CSS class) |
| `d3` | D3.js | Full d3 library for data transforms, formatting, scales |
| `DATA` | `Array<object>` | Rows from the DataFrame specified by `data_variable` |
| `width` | `number` | Container width in pixels (auto-sized to node) |
| `height` | `number` | Container height in pixels (auto-sized to node) |

**Critical:** Your code MUST assign the result to a variable named `chart`:
```javascript
chart = Plot.plot({ ... })
```

Do NOT use `const chart =` or `let chart =` — just bare assignment. The wrapper checks `typeof chart !== 'undefined'` to extract your result.

## Color — use it generously

**Charts must be colorful.** The dark theme is designed to showcase vivid
category colors from Plot's default palette (tableau10). A single-color chart
looks flat and defeats the purpose of visual comparison. **Always set `fill`
(or `stroke` for line marks) to a column in the data** so Plot picks distinct
colors automatically.

The two patterns you will use for almost every chart:

1. **One value per row, coloring by category** — when each row represents a
   distinct category (e.g. `{country: "UK", sales: 42}`), set `fill` to the
   same column as the category axis. Every bar/dot/slice gets its own color.
   ```javascript
   Plot.barY(DATA, { x: "country", y: "sales", fill: "country", tip: true })
   //                                           ^^^^^^^^^^^^^^^^ this is the whole point
   ```

2. **Multi-series / grouped data** — when rows have an explicit series column
   (e.g. `{month: "Jan", region: "EMEA", sales: 42}`), set `fill`/`stroke` to
   the series column so each series gets a distinct color plus a legend.
   ```javascript
   Plot.lineY(DATA, { x: "month", y: "sales", stroke: "region", tip: true })
   ```

**Always add `color: { legend: true }`** to the plot options when you use a
categorical `fill` or `stroke`. The legend makes the color-to-category mapping
explicit, which is the point of color in the first place.

**Only omit `fill` entirely if the chart is genuinely a single series with no
category dimension** (e.g. a single time series with no breakdown). In that
case Plot's default `steelblue` is auto-replaced with the app's accent color —
but prefer a categorical `fill` whenever your data has one.

## Theme Post-processing

The dark theme is applied automatically to the SVG after rendering:
- **Axis text** → foreground (~88% white)
- **Grid/axis lines** → border color (~18% grey)
- **`steelblue` fills/strokes** (Plot's single-series default) → app's accent
  color (`--chart-1`, vivid blue). Fallback only — prefer categorical colors.
- **Background** → transparent (inherits the dark node wrapper)
- **Font size** → 12px for readability
- **All other colors are preserved** — tableau10, scheme colors, and any
  explicit colors you set are kept as-is.

**What this means for your code:**
- Don't set `background` — it gets overridden anyway
- Don't worry about text color — it gets post-processed
- Don't avoid colors to "match the theme" — colors are the theme

## Margin Handling

The frontend enforces minimum margins: `marginTop ≥ 25`, `marginLeft ≥ 55`. Your values are used if larger, but won't go below these minimums. This prevents axis clipping at 12px font size.

When using legends (`color: { legend: true }`), the frontend automatically measures legend height and re-renders with reduced chart height so nothing clips.

## Common Patterns

### Vertical Bar Chart (colored by category)
```javascript
chart = Plot.plot({
  width, height,
  x: { label: null },  // tick rotation is automatic — see Tips
  y: { label: "Revenue ($k)", grid: true },
  color: { legend: true },
  marks: [
    // fill: "category" gives each bar a distinct color from tableau10
    Plot.barY(DATA, { x: "category", y: "value", fill: "category", tip: true }),
    Plot.ruleY([0])
  ]
})
```

### Grouped Bar Chart (multi-series)
```javascript
chart = Plot.plot({
  width, height,
  fx: { label: null },   // group axis (e.g. month)
  x: { axis: null },     // series axis inside each group — hide it, the legend explains it
  y: { label: "Count", grid: true },
  color: { legend: true },
  marks: [
    Plot.barY(DATA, { fx: "month", x: "series", y: "value", fill: "series", tip: true }),
    Plot.ruleY([0])
  ]
})
```

Never encode the same column as both `x` and `fx` — that prints every
group label twice and the copies overprint each other. The group goes on
`fx`, the series on `x` (hidden via `axis: null`). Facet (`fx`) labels
are **not** auto-rotated by the canvas, so keep them short — abbreviate
with `fx: { tickFormat: d => String(d).slice(0, 3) }` when needed.

### Stacked Bar Chart
```javascript
chart = Plot.plot({
  width, height,
  x: { label: null },
  y: { label: "Total", grid: true },
  color: { legend: true },
  marks: [
    Plot.barY(DATA, { x: "month", y: "value", fill: "category", tip: true }),
    Plot.ruleY([0])
  ]
})
```

### Single Line Chart
```javascript
// Only use this form when there's genuinely no category/series dimension
// (e.g. one metric over time). For anything else, prefer the multi-series form.
chart = Plot.plot({
  width, height,
  x: { type: "time", label: null },
  y: { label: "Price ($)", grid: true },
  marks: [
    Plot.lineY(DATA, { x: "date", y: "price", tip: true }),
    Plot.ruleY([0])
  ]
})
```

### Multi-Series Line Chart
```javascript
chart = Plot.plot({
  width, height,
  x: { type: "time", label: null },
  y: { label: "Value", grid: true },
  color: { legend: true },
  marks: [
    Plot.lineY(DATA, { x: "date", y: "value", stroke: "series", tip: true })
  ]
})
```

### Area Chart (stacked)
```javascript
chart = Plot.plot({
  width, height,
  x: { type: "time", label: null },
  y: { label: "Revenue", grid: true },
  color: { legend: true },
  marks: [
    Plot.areaY(DATA, { x: "date", y: "value", fill: "category", tip: true }),
    Plot.ruleY([0])
  ]
})
```

### Scatter Plot
```javascript
chart = Plot.plot({
  width, height,
  x: { label: "Weight (kg)" },
  y: { label: "Height (cm)" },
  color: { legend: true },
  marks: [
    Plot.dot(DATA, { x: "weight", y: "height", fill: "species", r: 4, tip: true }),
  ]
})
```

### Heatmap
Plot does NOT auto-flip text color against cell fill — pale cells with white labels become unreadable. If you draw value labels, set their `fill` based on the cell's value (or its luminance). See `references/heatmap-readability.md` for variants (sequential, diverging, custom domain).

```javascript
chart = Plot.plot({
  width, height,
  x: { label: null },
  y: { label: null },
  color: { scheme: "YlOrRd", legend: true, label: "Intensity", domain: [0, 100] },
  marks: [
    Plot.cell(DATA, { x: "column", y: "row", fill: "value", tip: true }),
    // Pick text color from the same domain as the fill: dark on pale, light on saturated.
    Plot.text(DATA, { x: "column", y: "row", text: d => d.value.toFixed(0) + "%",
                     fill: d => d.value < 40 ? "#111" : "#fff" })
  ]
})
```

### Horizontal Bar Chart (sorted, colored by name)
```javascript
chart = Plot.plot({
  width, height,
  x: { label: "Sales ($)", grid: true },
  y: { label: null },
  color: { legend: true },
  marks: [
    Plot.barX(DATA, {
      y: "name", x: "sales",
      fill: "name",        // each bar gets its own color
      sort: { y: "-x" },
      tip: true
    }),
    Plot.ruleX([0])
  ]
})
```

### Histogram (binning)
```javascript
// Histograms are genuinely single-color (each bar = a bin count, not a category).
// The steelblue default is acceptable here and gets auto-themed to the accent color.
chart = Plot.plot({
  width, height,
  x: { label: "Age" },
  y: { label: "Count", grid: true },
  marks: [
    Plot.rectY(DATA, Plot.binX({ y: "count" }, { x: "age", tip: true })),
    Plot.ruleY([0])
  ]
})
```

## Key API Reference

For detailed mark signatures, scale types, transforms, and advanced patterns, read `references/plot-api.md`.

## Tips

- **Use color** — every bar chart, horizontal bar, scatter, pie, dot plot should have a `fill` (or `stroke` for lines) set to a categorical column. Monochromatic charts are rarely right.
- **Always add `color: { legend: true }`** when you set a categorical `fill`/`stroke` — the legend is what makes color meaningful.
- **Always add `tip: true`** to your primary mark for interactive tooltips on hover
- **Use `Plot.ruleY([0])`** on most bar/area charts for a baseline
- **Use `grid: true`** on the value axis (usually y) for readability
- **Set `label: null`** on axes where the meaning is obvious from context
- **Tick rotation is automatic** — the canvas measures labels after Plot renders and re-renders rotated (-45°, escalating to -90° for very long labels with many ticks) when they would overlap. Don't set `tickRotate` yourself; the canvas overrides it. If the chart still looks crowded, switch to `Plot.barX` (horizontal bars) instead.
- **Use `sort: { y: "-x" }`** (or `{ x: "-y" }`) to sort bars by value
- **Date columns:** if your data has date strings, use `x: { type: "time" }` — Plot handles ISO date strings
- **Formatting ticks:** use `tickFormat: d => d + "k"` or d3 formatters like `tickFormat: d3.format(",.0f")`
- **Don't hardcode dimensions** — always use the provided `width` and `height` globals
- **Empty data is handled** — if your filter/aggregation yields 0 rows, the canvas shows "No rows to plot" automatically. You don't need to guard for empty arrays in your chart code.

## Self-review checklist (run after every create_chart)

After every successful `create_chart` call, immediately re-read your own
chart code against the checklist below and call `edit_chart` once if any
item fails. Stay silent unless you actually fix something — in which case
briefly note what you changed.

The chart is already visible to the user during this review, so don't
delay creation to think about it first — render fast, then verify.

1. **Axis ordering** — Time/date axes must be chronological, not
   alphabetical. If x is a date/month/quarter/year, sort the underlying
   data ascending by that column before charting, or pass `domain` /
   `sort: {x: 'x'}` to enforce order. Months as strings (`'Jan'`, `'Feb'`)
   sort alphabetically by default — almost always wrong.
2. **Ordered categoricals** — Ranked or sized categories (small/medium/large,
   low/mid/high, decile labels) need an explicit `domain` to preserve order.
3. **Bar chart sorting** — For non-ordinal bars, sort by value descending
   (`sort: {x: '-y'}` or `sort: {y: '-x'}` for `barX`) unless there's a
   natural order. Unsorted bars are almost always a bug.
4. **Label density** — Check the category axis for colliding or
   overprinted labels. The canvas auto-rotates plain x-axis tick labels,
   but **not** facet (`fx`) labels: never map the same column to both
   `x` and `fx`, and keep `fx` labels short (abbreviate via
   `fx: { tickFormat: ... }`). If a categorical axis is still crowded
   (many categories, long names), switch to `Plot.barX` (horizontal).
5. **Encoding fit** — Continuous data → line/area/scatter; categorical
   → bar; parts of a whole → stacked bar. Pie charts are not supported
   by Plot — use a horizontal bar instead.
6. **Required options** — `tip: true` and `grid: true` present.
7. **Bare assignment** — `chart = Plot.plot(...)`, NOT `const chart`.
8. **No fixed width/height** in `Plot.plot()` — the container sizes it.
9. **Data volume** — If the data going into the chart has >2,000 rows,
   you should have aggregated/sampled first.
10. **Color semantics** — Don't override theme colors unless the chart
    encodes a meaningful category by color (and then use a colorblind-safe
    scheme like `scheme: 'tableau10'`).

If you call `edit_chart` to fix something, do it once. Don't loop.
