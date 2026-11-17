---
name: op
description: Drive an Op data canvas — list and inspect projects, run Python, create chart / table / text nodes, edit notebook cells, read live canvas state. Use whenever the user references Op, asks to query their data, create a visualization, or work with their canvas. Triggers on "op", "ask Op", "Op canvas", "show on Op", "add to my canvas", or any data-analysis request that should land on a canvas the user will look at later.
allowed-tools: Bash
argument-hint: [natural-language task — what should land on the canvas?]
---

# Op

You drive the user's Op data canvas through the `op` CLI. Every action you take leaves a persistent artifact on the canvas (table / chart / text / notebook cell) that the user sees in their browser — treat the canvas as your shared workspace, not a scratch pad.

## Setup (once per machine)

If `op` is not on the user's PATH, install it: `npm install -g @opapp/cli`. Verify with `op --version`.

If `op whoami` returns `{logged_in: false}`, run `op login` and tell the user:

> "Op needs you to authorize this CLI. A browser tab should have just opened — click Authorize there (the code to expect is <code>). If nothing opened, go to <url>. Tell me when you're done."

`op login` opens the verification page in the user's default browser
automatically (best-effort) and also prints the URL + code to stderr as
the fallback — relay both. It then blocks polling for up to 10 minutes;
run it in the background or a separate shell so you can keep talking to
the user while it waits.

## Mental model

- `op context` — read-only snapshot of the canvas: nodes (id, type, title, group_id), groups (id, name, member node ids), dataframes (name + shape + columns), and notebook cells (id, position, status, first lines of code). **Always run this first** in any new turn so you know what already exists — every id you pass to the commands below comes from here.
- `op run "<python>"` — execute Python in the project's namespace. No node created. Output goes to stdout. Use for inspection (`df.head()`, `df.dtypes`) or to load/transform data into variables. Variables persist for the rest of the session.
- `op cell create "<python>"` — execute Python *and* create a notebook cell. Use when the code is worth keeping (data load, key transform) — it survives container sleep and the user can re-run it.
- `op cell edit <cell_id> "<python>"` — edit an existing cell and re-execute.
- `op cell delete <cell_id>` — remove a notebook cell. No undo; only on the user's request or to clean up a cell you created.
- `op cells reorder <cell_id> <cell_id> ...` — put cells in this order; cells you leave out keep their relative order after the listed ones. Every id must exist.
- `op chart --title T --var df_name --code "Plot.plot(...)"` — create a chart node. The DataFrame `df_name` must exist in the namespace. Charts use Observable Plot.
- `op table --title T --var df_name` — create a table node from a DataFrame.
- `op text --title T --content "..."` — create a plain-text label on the canvas.
- `op chart edit <node_id> [--title T] [--var df_name] [--code "..."]` — change a chart's title, data source or Plot code in place. Pass only what changes.
- `op table edit <node_id> [--title T] [--var df_name]` — retitle a table or point it at another DataFrame.
- `op node move <node_id> --x X --y Y` / `op node resize <node_id> --width W --height H` — layout tweaks. Both coordinates (or both dimensions) are required.
- `op node group <node_id> <group_name>` — put a node in a group, creating the group if needed; `op node group <node_id> ""` ungroups it.
- `op group create <name>` — create a group (returns the existing one if the name is taken); `op group rename <group_id> <name>`; `op group delete <group_id>` — remove a group, its nodes stay on the canvas ungrouped.
- `op node delete <node_id>` — remove a canvas node (ids from `op context`). Use this to clean up your own mistakes — a chart that errored, a duplicate — instead of leaving broken artifacts on the user's canvas.

Every edit, move, delete and group change shows up live in the user's open browser tab, the same way the in-app agent's actions do.

For multi-line code, pipe via stdin: `op run -` reads stdin.

## How a turn should flow

0. (First turn only) `op whoami` to confirm auth.
1. `op context` — read state.
2. Decide what's missing (data not loaded? chart not yet made?).
3. Use `op run` / `op cell create` to set up data; check stdout to confirm shapes.
4. Use `op chart` / `op table` / `op text` to land the result on the canvas.
5. Show the user a short summary plus the canvas URL (the response from each tool call includes a `state_tail` you can reference).

## Picking a project

Most users have a default project; omit `--project` and the server uses `selected_project_id`. To target a specific one: `op projects` lists them, then pass `--project <id>` on every command.

## Tiers & limits

The full CLI surface (run, charts, tables, text, cells) works on the FREE
tier — never tell a user they must upgrade to start. Free accounts are
bounded by resource limits instead: a daily compute cap shared with in-app
usage, and a project-count cap. First-time users don't need the web app:
the CLI auto-creates a project on first use. Every mutation response
includes a `canvas_url` — after creating something, show the user that
link so they can open it in the app.

## When things go wrong

- `runtime_limit_reached` (HTTP 429) → daily compute cap spent. Relay the message and `upgrade_url`; do NOT retry — it resets tomorrow.
- `project_limit` (HTTP 403) → project-count cap. Relay `upgrade_url`.
- `missing_variable` → the DataFrame named in `--var` doesn't exist. Load it first with `op run` or `op cell create`.
- `not_a_dataframe` → variable exists but isn't a pandas DataFrame.
- `node_not_found` / `cell_not_found` / `group_not_found` (HTTP 404) → the id is stale or mistyped. Re-run `op context` and use an id from there.
- `node_type_mismatch` → you passed chart fields to a table (or vice versa). Check the node's `type` in `op context`.
- `dataframe_too_large` → chart cap is 5000 rows; aggregate or sample first.
- `backend_unavailable` (503) / `backend_timeout` (504) → project container cold boot; the CLI already retries with backoff (you'll see "waking project container" on stderr). If it still fails, wait ~30s and try once more.
- App-level `500` → a real server bug, not a cold boot. Don't blind-retry; surface the error body to the user.

## Related skills

For domain-specific guidance, use these companion skills when relevant:

- **chart-creation** — Observable Plot patterns for line/bar/scatter/heatmap charts
- **data-analysis** — Python recipes for common pandas operations
- **op-canvas** — layout conventions, group naming, when to use cells vs scratchpad
- **pm-analytics** — product-metric patterns (funnels, cohorts, activation)
- **retention-analysis** — cohort retention specifics

### Translating skill examples to CLI calls

The companion skills above use Python tool-call syntax (e.g. `create_chart(title=..., code=..., data_variable=...)`). Translate to CLI as follows:

| Skill syntax | CLI equivalent |
|---|---|
| `create_chart(title=T, code=C, data_variable=V, group=G)` | `op chart --title T --code C --var V --group G` |
| `create_table(title=T, variable_name=V, group=G)` | `op table --title T --var V --group G` |
| `create_text(title=T, content=C)` | `op text --title T --content C` |
| `create_cell(code=C)` | `op cell create C` (or pipe via `op cell create -`) |
| `edit_cell(cell_id=ID, new_code=C)` | `op cell edit ID C` |
| `delete_cell(cell_id=ID)` | `op cell delete ID` |
| `reorder_cells(cell_order=[A, B])` | `op cells reorder A B` |
| `edit_chart(node_id=ID, code=C, title=T, data_variable=V)` | `op chart edit ID --code C --title T --var V` |
| `edit_table(node_id=ID, variable_name=V, title=T)` | `op table edit ID --var V --title T` |
| `move_node(node_id=ID, x=X, y=Y)` | `op node move ID --x X --y Y` |
| `resize_node(node_id=ID, width=W, height=H)` | `op node resize ID --width W --height H` |
| `assign_node_to_group(node_id=ID, group_name=G)` | `op node group ID G` |
| `upsert_group(name=G)` / `upsert_group(name=G, group_id=ID)` | `op group create G` / `op group rename ID G` |
| `delete_group(group_id=ID)` | `op group delete ID` |
| `delete_node(node_id=ID)` | `op node delete ID` |
| `run_code(code=C)` | `op run C` (or `op run -` from stdin) |

The skills' value is the domain content (chart patterns, analysis recipes, layout conventions) — read them for guidance and translate the call shape.

## Arguments

$ARGUMENTS — Natural-language description of what the user wants to land on the canvas.
