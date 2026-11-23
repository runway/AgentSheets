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
| `build-model` | Use when building, editing, or troubleshooting cfo.ai models: variables, formulas, dimensions, mappings, rollups, tables, scenarios, statements, cash, or BvA. |
| `build-visualizations` | Use when building or reviewing Canvas visuals and interactive apps: chart types, direct manipulation, datasets, and visual review. |
| `formula-reference` | Use when checking cfo.ai formula syntax, operators, date grains, functions, arities, or Excel differences (parser-generated). |
| `import-model` | Use when rebuilding an external spreadsheet, export, or document as a native cfo.ai model. |

## About this repository

Generated from `runway/cfoai` and pushed here by the `sync-agent-sheets`
workflow. Edits made directly to this repository are overwritten on the next
sync — change the skills in the source repository instead.
