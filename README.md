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
| `build-model` | Operate the model-authoring surface for variables, dimensions, formulas, and supporting table context. Use when creating, evaluating, validating, or updating model logic. |
| `dimensional-modeling` | The core reference for how modeling works — segments and grains, formula dispatch and recompute-vs-rollup laws, table-block axes and pivoting, validity rules, time and granularity, scenarios and comparisons, and the modeling workflow. Use when designing, restructuring, debugging, or explaining tables, formulas, or pivots; when numbers look wrong or cells are unexpectedly blank; or before any multi-step modeling build. |
| `table-building` | Operate pages and saved table blocks for financial reports. Use when the user wants a table saved as an artifact — creating, validating, inspecting, or updating one — not merely to answer a table-shaped question in chat. |
| `visualization-building` | Operate pages, charts, and code visualizations for saved reports. Use when the user wants a visual saved on a page — creating, validating, inspecting, or updating one — not merely to explain what the data shows in chat. |

## About this repository

Generated from `runway/cfoai` and pushed here by the `sync-agent-sheets`
workflow. Edits made directly to this repository are overwritten on the next
sync — change the skills in the source repository instead.
