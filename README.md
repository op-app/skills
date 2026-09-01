# Op Agent Skills

[Op](https://opapp.io) is a data canvas: charts, tables, notebook cells and notes that live in your browser and persist between sessions. These skills let an agent (Claude Code, or anything that reads [Agent Skills](https://skills.sh)) drive your canvas through the `op` CLI.

## Install

```bash
npx skills add op-app/skills
```

Then, in a new Claude Code session, ask for something to land on your canvas — the `op` skill takes over, installs the CLI (`npm install -g @opapp/cli`) if needed, and walks you through `op login`.

## Skills

| Skill | What it does |
| --- | --- |
| `op` | The main skill: inspect projects, run Python, create chart / table / text nodes, edit notebook cells |
| `chart-creation` | Observable Plot patterns, color rules and readability references for charts on the canvas |
| `data-analysis` | Exploratory-analysis workflow that ends in canvas artifacts |
| `op-canvas` | Layout, positioning and text-insight guidance for canvas artifacts |
| `pm-analytics` | Funnels, retention cohorts, A/B readouts, feature adoption — pandas recipes plus canvas layout |
| `retention-analysis` | Cohort retention analysis end to end |

## This repo is a read-only mirror

The source of truth is the `skills/` directory of the main Op repository; a GitHub Action mirrors it here on every change. **Please open issues and pull requests on the main repo, not here** — changes pushed to this repo are overwritten by the next sync.
