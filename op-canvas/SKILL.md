---
name: op-canvas
description: Creating and managing canvas nodes in OP App — tables, charts, and text. Use this skill when you need to place data on the canvas, create visualizations, add annotations, or manage canvas layout and positioning.
---

# OP App Canvas Operations

Create interactive visualizations and content on the OP App canvas using MCP tools.

## Available Tools

### `create_table(variable_name, title)`
Show a DataFrame as an interactive table on the canvas.

- `variable_name`: name of a DataFrame in the current namespace
- `title`: display title for the table header
- Position is auto-calculated below existing nodes

**Workflow:**
1. Use `run_code` to create/transform a DataFrame
2. Use `create_table` to show it on canvas

```
# Example: show a summary table
run_code: sales_by_region = df.groupby('region')['revenue'].sum().reset_index()
create_table: variable_name="sales_by_region", title="Revenue by Region"
```

### `create_chart(code, title, data_variable)`
Create an Observable Plot chart visualization. See the `chart-creation` skill for detailed patterns and API reference.

- `code`: JavaScript code that assigns to a `chart` variable (runs on frontend, not here)
- `title`: display title for the chart header
- `data_variable`: name of a DataFrame — its rows become the `DATA` array in JS

### `create_text(content, title)`
Add a plain text node for labels, annotations, summaries, and insights. Can overlap other nodes to act as a label.

- `content`: plain text (no markdown)
- `title`: display title

```
# Example: label for a chart
create_text: title="Key Finding", content="Revenue is concentrating in 3 accounts\nTop 3 represent 47% of total revenue"
```

### `create_cell(code)`
Create a visible notebook cell with Python code and execute it.

- `code`: Python code to execute
- Creates a ProposedAction the user can accept or reject
- Use for clean, final code the user should keep

### `edit_cell(cell_id, new_code)`
Edit an existing notebook cell with new code and re-execute it.

### `run_code(code)`
Execute Python code for exploration only. Output returns to you — no cell created, nothing visible to user. Variables persist for subsequent tool calls.

### `list_dataframes()`
List all DataFrames in the namespace with shape, columns, dtypes, and a 3-row sample.

### `get_canvas_info()`
Compact summary: total node count, counts by type, bounding rect, viewport. Use when you just need "what's roughly here". For specifics, use `find_canvas_nodes` / `get_canvas_node`.

### `find_canvas_nodes(query, node_type?, limit?)`
Search the canvas. Substring-matches against title, text content, chart code, table dataframe name, and table column names. Returns id, type, title, position, plus a snippet showing the match. Empty `query` browses; `node_type` ('table' | 'chart' | 'text') filters.

```
# "What charts are about regions?"
find_canvas_nodes: query="region", node_type="chart"

# "Show me all tables"
find_canvas_nodes: query="", node_type="table", limit=20
```

### `get_canvas_node(node_id)`
Full details for one node — including chart code, table columns, or text content. Pair with `find_canvas_nodes` when you need to read or reference an existing artifact.

## Positioning

Canvas uses stable absolute coordinates. The user's pan does NOT shift the coordinate system — `(x, y)` always refers to the same spot.

**Default:** omit `x` / `y` on `create_table` / `create_chart` / `create_text`. Nodes auto-stack below existing content with the right gaps.

Pass explicit coordinates only when you're laying out side-by-side dashboards and need a reference frame — call `find_canvas_nodes` first to read the relevant nearby nodes' positions.

Default node sizes: Table = 400×300px, Chart = 640×400px, Text = 320×auto.

## Layout Principles

Arrange artifacts so the canvas reads like a narrative — left-to-right or top-to-bottom.

- Text nodes are signposts: place them near or overlapping the artifacts they explain
- Label text nodes can overlap the top edge of a chart/table to title it
- ~40px gap between data artifacts (tables, charts)
- New artifacts go below existing ones with a 50px vertical gap
- Keep things in a reasonable area — don't sprawl

**How many artifacts:**
- Single question, simple answer → 1 chart or 1 table + a text label
- "Analyze this data" → table + chart + insight text guiding the narrative
- Comparison → 2+ charts with label text per chart
- Deep dive → multiple chart types showing different facets + connecting narrative text

## Text Insights

Plain text only (no markdown). Multiple small text nodes beat one long one.

**Brevity:**
- Labels: 1 line. A phrase or sentence.
- Insights: 2–3 lines max. One finding + one comparison number.

**Writing style:**
- Lead with the finding, not the methodology
- Always include one comparison (%, vs prior period, vs benchmark) — a number without context isn't an insight
- Write like a chart annotation, not a paragraph

## Best Practices

### 1. Explore Before Creating
```
# Always inspect data before creating visible artifacts
run_code: print(df.shape); print(df.columns.tolist()); print(df.head())
```

### 2. Create New Variables — Don't Overwrite Sources
```
# Good: new variable
run_code: monthly = df.groupby('month')['sales'].sum().reset_index()
create_table: variable_name="monthly", title="Monthly Sales"

# Bad: overwriting source
run_code: df = df.groupby('month')['sales'].sum()
```

### 3. Use Descriptive Names
```
# Good
run_code: top_customers = df.nlargest(10, 'revenue')

# Avoid
run_code: temp = df.nlargest(10, 'revenue')
```

### 4. Combine Artifacts for a Complete Analysis
Tables for detailed data, charts for patterns/trends, text for narrative insights. A typical analysis creates 1-2 tables, 1-2 charts, and a text summary.
