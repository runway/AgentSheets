# AgentSheets

Model in [cfo.ai](https://cfo.ai) from your agent. This plugin bundles cfo.ai's
modeling skills with the cfo.ai MCP server, so an agent can build and edit real
financial models rather than describe how it would.

## Install

```bash
codex plugin marketplace add https://github.com/runway/AgentSheets.git --ref main \
  --sparse '.agents/plugins' --sparse 'plugins'
codex plugin install agentsheets --source agentsheets
```

The bundled MCP server is `https://api.cfo.ai/mcp`. It authenticates with OAuth on first
use, and every tool call is scoped to the permissions your cfo.ai account
already has. This is the default connection URL. If your agent keeps using stale
tool definitions, refresh its tool catalog or start a fresh conversation.

## Skills

| Skill | When it applies |
| ----- | --------------- |
| `build-model` | Build, change, or diagnose a model and model the business in it. Use for variables, formulas, dimensions, mappings, time, rollups, table layouts, comparisons, formatting, and unexpected results; for forecasts, budgets, plans, and scenarios; for headcount, payroll, and rosters; for bookings, pipeline, and deal registers; and for statements, cash and runway, budget versus actuals, and unusual results. |
| `build-visualizations` | What a page visual is for and how Canvas blocks connect to live model values: choosing the visual, the dataset contract, templates, and how a saved visual is checked and changed. |
| `import-model` | Rebuilding someone else's model — a workbook, an export, a rendered document — as a native cfo.ai model that keeps its meaning. Use when asked to import, migrate, rebuild, or recreate a model from source files. |

## About this repository

Generated from `runway/cfoai` and pushed here by the `sync-agent-sheets`
workflow. Edits made directly to this repository are overwritten on the next
sync — change the skills in the source repository instead.
