# Observable Plot API Reference

Detailed mark signatures, scale options, and transforms for the Observable Plot library.

## Table of Contents
- [Plot.plot() Options](#plotplot-options)
- [Marks](#marks)
- [Scales](#scales)
- [Transforms](#transforms)
- [Color Schemes](#color-schemes)
- [Legends](#legends)
- [Faceting](#faceting)

---

## Plot.plot() Options

```javascript
Plot.plot({
  width,                    // number (default: 640)
  height,                   // number (auto-calculated if omitted)
  marginTop, marginRight, marginBottom, marginLeft,  // number (auto from marks)

  title: "Chart Title",     // string — above plot
  subtitle: "More context", // string — below title
  caption: "Source: ...",   // string — below plot

  marks: [],                // array of mark instances (required)

  // Scale configs
  x: { /* scale options */ },
  y: { /* scale options */ },
  fx: { /* facet x scale */ },
  fy: { /* facet y scale */ },
  color: { /* color scale */ },
  opacity: { /* opacity scale */ },
  r: { /* radius scale */ },

  // Shorthands
  grid: false,              // enable grid on both x and y
  inset: 0,                 // pixel inset from edges
  clip: false,              // clip marks to plot area
  className: "",            // CSS class on root SVG
})
```

## Marks

All marks: `MarkType(data, options)` where `data` is an array and `options` maps channels to field names or accessor functions.

### Bar
```
Plot.barY(data, { x, y, fill, stroke, inset, rx, ry, sort, tip })
Plot.barX(data, { x, y, fill, stroke, inset, rx, ry, sort, tip })
```
- `fill: "column"` triggers automatic stacking
- `sort: { x: "-y" }` sorts bars descending by value
- `rx: 2` for rounded corners

### Line
```
Plot.line(data, { x, y, z, stroke, strokeWidth, curve, marker, sort, tip })
Plot.lineY(data, opts)   // shorthand: x defaults to index
Plot.lineX(data, opts)   // shorthand: y defaults to index
```
- `stroke: "series"` for multi-series (also sets z grouping)
- `curve`: "linear", "step", "step-before", "step-after", "basis", "cardinal", "catmull-rom", "monotone-x", "natural"
- `marker`: "circle", "arrow", "dot", or true

### Area
```
Plot.areaY(data, { x, y, y1, y2, fill, curve, offset, sort, tip })
Plot.areaX(data, { y, x, x1, x2, fill, curve, offset, sort, tip })
Plot.area(data, { x1, x2, y1, y2, fill, curve, tip })
```
- `fill: "column"` triggers automatic stacking
- `offset`: null, "normalize", "center", "wiggle" (streamgraph)

### Dot
```
Plot.dot(data, { x, y, r, fill, stroke, symbol, rotate, tip })
Plot.dotX(data, opts)
Plot.dotY(data, opts)
```
- `r: "population"` scales dot size
- `symbol`: "circle", "cross", "diamond", "square", "star", "triangle"
- Default fill: currentColor, default r: 3

### Cell (heatmap)
```
Plot.cell(data, { x, y, fill, stroke, inset, rx, ry, tip })
Plot.cellX(data, opts)
Plot.cellY(data, opts)
```
- x and y are ordinal (band) scales
- `fill: "value"` with a sequential color scheme for heatmaps

### Rect
```
Plot.rect(data, { x1, x2, y1, y2, fill, stroke, inset, rx, ry, tip })
Plot.rectX(data, opts)
Plot.rectY(data, opts)
```
- Use with `Plot.binX`/`Plot.binY` for histograms

### Text
```
Plot.text(data, { x, y, text, fill, fontSize, fontWeight, textAnchor, lineAnchor, dx, dy, rotate, tip })
```
- `textAnchor`: "start", "middle", "end"
- `lineAnchor`: "top", "middle", "bottom"
- `text: d => d.label` or `text: "column_name"`

### Rule
```
Plot.ruleY(data, { y, x1, x2, stroke, strokeWidth, strokeDasharray })
Plot.ruleX(data, { x, y1, y2, stroke, strokeWidth, strokeDasharray })
```
- `Plot.ruleY([0])` — horizontal baseline
- `Plot.ruleX([threshold])` — vertical reference line

### Tip (tooltip)
```
Plot.tip(data, Plot.pointerX({ x, y }))
Plot.tip(data, Plot.pointer({ x, y }))
```
- Most marks support `tip: true` as a shorthand
- Pointer interaction: `Plot.pointerX` (nearest x), `Plot.pointerY` (nearest y), `Plot.pointer` (nearest point)

### Frame
```
Plot.frame({ stroke, strokeWidth, fill, inset, rx, ry })
```
- Draws a border around the plot area

### Box (box-and-whisker)
```
Plot.boxY(data, { x, y, fill, stroke, tip })
Plot.boxX(data, { x, y, fill, stroke, tip })
```

### Arrow
```
Plot.arrow(data, { x1, y1, x2, y2, stroke, strokeWidth, bend, headLength, headAngle })
```

### Image
```
Plot.image(data, { x, y, src, width, height, r, tip })
```

---

## Scales

### Scale Types

| Type | Use | Example |
|------|-----|---------|
| `linear` | Continuous numeric | Default for quantitative |
| `log` | Logarithmic | `y: { type: "log" }` |
| `sqrt` | Square root | Good for area/radius |
| `pow` | Power | With `exponent` option |
| `symlog` | Symmetric log | Handles zero/negative |
| `utc` / `time` | Temporal | Auto-detected for Date |
| `band` | Categorical position | Auto for ordinal x in barY |
| `point` | Categorical points | Auto for ordinal x in dot |
| `ordinal` | Categorical color/symbol | Auto for string fill/stroke |
| `categorical` | Categorical (unordered) | Tableau10 palette |
| `sequential` | Continuous → color | `scheme: "blues"` |
| `diverging` | Two-ended → color | `scheme: "RdBu"` |
| `threshold` | Bucketed | `domain: [0, 50, 100]` |
| `quantile` | Equal-frequency buckets | `n: 5` |

### Common Scale Options
```javascript
{
  type: "linear",
  domain: [min, max],        // input range (auto-inferred from data)
  range: [min, max],         // output range
  reverse: false,
  nice: true,                // round domain to nice values
  zero: true,                // include zero
  clamp: false,              // restrict to domain

  // Axis display
  label: "Axis Label",       // null to hide
  axis: "bottom",            // "top", "bottom", "left", "right", null
  ticks: 5,                  // approximate count or explicit array
  tickFormat: d => d + "k",  // formatter function or d3-format string
  tickRotate: -45,           // degrees
  grid: true,                // show gridlines
  line: false,               // show axis baseline

  // Color scale only
  scheme: "tableau10",       // named scheme
  legend: true,              // show legend
}
```

### Tick Formatting
```javascript
// d3 format strings
tickFormat: ".0%"            // percentage
tickFormat: "$,.0f"          // currency
tickFormat: ".2s"            // SI prefix (1.2M)

// Custom functions
tickFormat: d => d + "k"
tickFormat: d => d3.timeFormat("%b %Y")(d)
tickFormat: d => DATA[d]?.label ?? d  // lookup from data
```

---

## Transforms

Transforms aggregate or reshape data inline:

### Group (categorical aggregation)
```javascript
// Count per category
Plot.barY(DATA, Plot.groupX({ y: "count" }, { x: "species" }))

// Sum per category
Plot.barY(DATA, Plot.groupX({ y: "sum" }, { x: "month", y: "revenue" }))

// Available reducers: "count", "sum", "mean", "median", "min", "max",
//   "first", "last", "mode", "deviation", "variance", "proportion"
```

### Bin (numeric binning)
```javascript
// Histogram
Plot.rectY(DATA, Plot.binX({ y: "count" }, { x: "age" }))

// 2D heatmap
Plot.rect(DATA, Plot.bin({ fill: "count" }, { x: "longitude", y: "latitude" }))

// Options: thresholds (count or array), domain, cumulative
```

### Stack
```javascript
// Implicit: fill/stroke on barY/areaY triggers stacking
Plot.barY(DATA, { x: "month", y: "value", fill: "category" })

// Explicit with offset
Plot.areaY(DATA, Plot.stackY({ offset: "normalize" }, { x: "date", y: "value", fill: "series" }))

// Offsets: null (default), "normalize" (0-1), "center", "wiggle"
// Orders: null, "appearance", "inside-out", "sum", "value", "z", "-z"
```

### Map (running calculations)
```javascript
// Cumulative sum
Plot.lineY(DATA, Plot.mapY("cumsum", { x: "date", y: "revenue" }))

// Available: "cumsum", "first", "last", "rank", window functions
```

### Window (rolling)
```javascript
// 7-day moving average
Plot.lineY(DATA, Plot.windowY(7, { x: "date", y: "value" }))

// Options: k (window size), anchor ("middle", "start", "end"),
//   reduce ("mean", "sum", "min", "max", "median", etc.)
```

### Select (filtering)
```javascript
// First/last point per series
Plot.dot(DATA, Plot.selectFirst({ x: "date", y: "value", z: "series" }))
Plot.dot(DATA, Plot.selectLast({ x: "date", y: "value", z: "series" }))

// Min/max per group
Plot.dot(DATA, Plot.selectMaxY({ x: "date", y: "value" }))
```

### Normalize
```javascript
// Normalize each series to start at 1
Plot.lineY(DATA, Plot.normalizeY({ x: "date", y: "price", stroke: "ticker" }))
```

---

## Color Schemes

### Categorical (for discrete categories)
- **`tableau10`** — default for ordinal. 10 distinct colors. Best for most use cases.
- `observable10`, `category10`, `accent`, `dark2`, `paired`, `pastel1`, `pastel2`, `set1`, `set2`, `set3`

### Sequential (continuous low→high)
- `blues`, `greens`, `greys`, `oranges`, `purples`, `reds`
- `viridis`, `magma`, `inferno`, `plasma`, `cividis`, `turbo`, `warm`, `cool`
- `YlGnBu`, `YlOrRd`, `BuGn`, `BuPu`, `GnBu`, `OrRd`, `PuBu`, `PuRd`, `RdPu`, `YlGn`, `YlOrBr`

### Diverging (two-ended around a pivot)
- `RdBu`, `RdYlBu`, `RdYlGn`, `BrBG`, `PiYG`, `PRGn`, `PuOr`, `spectral`

Usage:
```javascript
color: { scheme: "YlOrRd", legend: true }
color: { type: "diverging", scheme: "RdBu", pivot: 0, legend: true }
```

---

## Legends

Enable with `legend: true` on a scale:
```javascript
color: { legend: true }
opacity: { legend: true }
symbol: { legend: true }
```

When `legend: true` on the color scale, Plot wraps the SVG in a `<figure>` element with the legend as a sibling. The frontend handles measuring the legend and adjusting chart height.

Legend options within the scale:
```javascript
color: {
  legend: true,
  domain: ["A", "B", "C"],     // explicit category order
  label: "Category",            // legend title
  tickFormat: d => d.toUpperCase(),
  columns: 3,                   // swatch columns
}
```

---

## Faceting

Split a chart into panels by a categorical variable:

```javascript
chart = Plot.plot({
  width, height,
  fx: { label: null },
  color: { legend: true },
  marks: [
    Plot.barY(DATA, { x: "product", y: "sales", fill: "product", fx: "region", tip: true }),
    Plot.ruleY([0])
  ]
})
```

- `fx: "column"` — facet horizontally
- `fy: "column"` — facet vertically
- Both can be used together for a grid

Top-level `facet` option:
```javascript
facet: { data: DATA, x: "region", y: "year" }
```
