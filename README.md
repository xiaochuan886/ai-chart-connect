# ai-chart-connect

> Connect any coding agent (Claude Code, Codex, Cursor, …) to a [Vizkit](https://github.com/) AI chart workbench over HTTP — create chart templates, render real data, export images, and build illustrated analysis reports. **The agent generates the content and writes it through the API; it does not call the workbench's built-in AI.**

This is an [agent skill](https://skills.sh) that lets an external agent drive a local chart service. The agent writes template files (manifest / config / renderer) itself and validates them through the API, so you get the agent's stronger model capability — and it works even if the workbench has no built-in AI configured.

## Install

```bash
npx skills add xiaochuan886/ai-chart-connect
```

This drops the skill into `~/.agents/skills/ai-chart-connect/` and links it into your agent's skill directory (Claude Code, Codex, Cursor, … — auto-detected). Requires the [`skills` CLI](https://skills.sh).

## Prerequisites

A running chart workbench (the Vizkit project) on `http://localhost:3001`. Start it in the project root:

```bash
npm run dev   # serves API on :3001, gallery on :5173
```

Verify it's up:

```bash
curl http://localhost:3001/api/health
# → { "ok": true, "service": "vizkit" }
```

If the health check fails, the service isn't running — tell the agent to start it first.

## What the agent can do

Once installed, ask your agent in natural language. The skill activates on intents like *"做一个折线图"*, *"replicate this gauge"*, *"用这份数据出图"*, *"生成经营报告"*, *"导出 PNG"*.

| Capability | What it does |
|------------|--------------|
| **Create a chart template** | Generate the 5-file contract (manifest/config.schema/sample-data/config.values/renderer.tsx), write via API with self-validation, fix until it renders |
| **Refine an existing template** | Change colors / add titles / toggle labels / add interaction — editing the right files so the change is visible in preview |
| **Render real data** | Apply your data to a template via `POST /export` with a `data` field — the template files stay untouched |
| **Build analysis reports** | Batch-render multiple charts + your narrative text into a Markdown/HTML report; recipe + cron for periodic digests |

## Quick start (the agent's playbook)

The agent follows this flow (detailed in `references/create-template.md`):

```
1. GET  /api/health                              → probe
2. POST /api/workspaces  { name, chartType }     → get workspaceId
3. PUT  /api/workspaces/:id/file                 → write 5 files in order:
       manifest.json → config.schema.json → sample-data.json
       → config.values.json → renderer.tsx (with validate:true)
4. POST /api/workspaces/:id/export  { format:"png" }
5. GET  /api/workspaces/:id/export-file?path=... → download the image
```

Every write self-checks: PUT a file with `validate:true`, read `error.stage` + `error.message` on failure, fix and rewrite until it returns `200`.

## Documentation

The skill ships with progressive reference docs (the agent reads them on demand):

| Doc | Covers |
|-----|--------|
| [`references/create-template.md`](./references/create-template.md) | The 5-file contract, validation rules (E1–E5), IBCS/Zebra-BI patterns |
| [`references/refine-template.md`](./references/refine-template.md) | Editing existing templates, the "config → renderer visibility" chain |
| [`references/apply-data.md`](./references/apply-data.md) | Rendering real data without mutating the template |
| [`references/build-report.md`](./references/build-report.md) | Multi-chart reports + periodic digest recipes |
| [`references/layout-shell.md`](./references/layout-shell.md) | The `@ai-chart/layout` shell API — ChartCanvas/ChartArea/scales/composition primitives |
| [`references/api-reference.md`](./references/api-reference.md) | Every HTTP endpoint, CLI equivalents |

## Key principle

**The agent never reads/writes the filesystem directly — everything goes through the HTTP API.** Override data (`data`/`configOverrides` on export) only affects that render; the workspace's `sample-data.json` / `config.values.json` are never modified. This is what makes "render my data" safe and repeatable.

## License

MIT — see [LICENSE](./LICENSE).
