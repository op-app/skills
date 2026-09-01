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

- `op context` — read-only snapshot of the canvas: nodes, dataframes (name + shape + columns), cell count. **Always run this first** in any new turn so you know what already exists.
- `op run "<python>"` — execute Python in the project's namespace. No node created. Output goes to stdout. Use for inspection (`df.head()`, `df.dtypes`) or to load/transform data into variables. Variables persist for the rest of the session.
- `op cell create "<python>"` — execute Python *and* create a notebook cell. Use when the code is worth keeping (data load, key transform) — it survives container sleep and the user can re-run it.
- `op cell edit <cell_id> "<python>"` — edit an existing cell and re-execute.
- `op chart --title T --var df_name --code "Plot.plot(...)"` — create a chart node. The DataFrame `df_name` must exist in the namespace. Charts use Observable Plot.
- `op table --title T --var df_name` — create a table node from a DataFrame.
- `op text --title T --content "..."` — create a plain-text label on the canvas.
- `op node delete <node_id>` — remove a canvas node (ids from `op context`). Use this to clean up your own mistakes — a chart that errored, a duplicate — instead of leaving broken artifacts on the user's canvas.

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
| `run_code(code=C)` | `op run C` (or `op run -` from stdin) |

The skills' value is the domain content (chart patterns, analysis recipes, layout conventions) — read them for guidance and translate the call shape.

## Arguments

$ARGUMENTS — Natural-language description of what the user wants to land on the canvas.
