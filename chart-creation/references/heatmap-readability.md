# Heatmap label readability

Observable Plot does not auto-flip text color against the underlying `Plot.cell` fill. If you add `Plot.text` value labels with a constant `fill` (or a fill mapped to the same color scale as the cell), low-value cells render white-on-white and high-value cells render dark-on-dark. This is the single most common heatmap bug.

The fix: derive the label `fill` from the same value the cell encodes, choosing dark text on pale cells and light text on saturated cells. The threshold depends on the color scheme.

## Sequential schemes (Blues, YlOrRd, Greens, Viridis, etc.)

Pick a value cutoff at the point where the scheme crosses to "dark enough for white text" — usually around 40-50% of the domain. Anchor the cutoff to the explicit `domain`, not the data range, so it stays correct as data changes.

```javascript
const DOMAIN = [0, 100];           // explicit domain
const CUTOFF = 40;                 // dark text below this, light above
chart = Plot.plot({
  width, height,
  color: { scheme: "Blues", legend: true, domain: DOMAIN, label: "Retention %" },
  marks: [
    Plot.cell(DATA, { x: "week", y: "cohort", fill: "retention", tip: true }),
    Plot.text(DATA, { x: "week", y: "cohort",
                     text: d => d.retention.toFixed(1) + "%",
                     fill: d => d.retention < CUTOFF ? "#111" : "#fff" })
  ]
})
```

For `Viridis` use `CUTOFF ≈ 50`; for `YlOrRd` use `CUTOFF ≈ 60` (yellow extends further); for `Greys` use `CUTOFF ≈ 50`.

## Diverging schemes (RdBu, PiYG, BrBG)

The midpoint is pale, both ends are saturated. Use distance from the midpoint:

```javascript
const MID = 0;
Plot.text(DATA, { x: "week", y: "cohort", text: d => d.delta.toFixed(1),
                 fill: d => Math.abs(d.delta - MID) < 5 ? "#111" : "#fff" })
```

## Luminance-based (most robust, costs a d3 import)

If the domain is unknown or the scheme can change, compute luminance from the actual scale:

```javascript
const fillScale = d3.scaleSequential(d3.interpolateBlues).domain([0, 100]);
Plot.text(DATA, { x: "week", y: "cohort", text: d => d.retention.toFixed(0),
                 fill: d => d3.lab(fillScale(d.retention)).l > 65 ? "#111" : "#fff" })
```

Use this when the heatmap is reused across datasets with different distributions.

## Other readability fixes that pair well

- **Always set an explicit `domain`** on the color scale. Without it, the scale auto-fits to data and a tight range will saturate every cell to one shade — labels then need contrast handling but the chart also becomes uninformative.
- **Stroke the cells.** A faint stroke (`stroke: "#222", strokeOpacity: 0.4` on `Plot.cell`) keeps the grid legible when low-value cells fade into a dark canvas background.
- **Skip labels below a noise threshold.** If most cells are near zero and clutter dominates, return `""` from `text:` for those: `text: d => d.retention < 1 ? "" : d.retention.toFixed(1)`.
- **For very small cells**, drop in-cell labels entirely and rely on `tip: true` (hover tooltip) — the labels stop fitting before contrast is the bottleneck.
