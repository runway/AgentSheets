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
already has.

## Skills

| Skill | When it applies |
| ----- | --------------- |
| `build-model` | Create or change model variables, dimensions, formulas, and table context. Use when building model logic, checking a formula, or testing a change before saving it. |
| `dimensional-modeling` | Explain how segments, grains, axes, formulas, rollups, tables, pivots, validity rules, time, scenarios, and comparisons work. Use when designing, changing, debugging, or explaining a model, especially rollup versus recompute behavior or granularity, when numbers look wrong or cells are blank, or before a multi-step model build. |
| `table-building` | Create, check, review, or update pages and saved financial-report tables. Use when the user wants a saved table, not just an answer in chat. |
| `visualization-building` | Create, check, review, or update pages, charts, and custom visuals. Use when the user wants a visual saved on a page, not just an answer in chat. |

## About this repository

Generated from `runway/cfoai` and pushed here by the `sync-agent-sheets`
workflow. Edits made directly to this repository are overwritten on the next
sync — change the skills in the source repository instead.
