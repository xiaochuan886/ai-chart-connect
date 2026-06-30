# API Reference

所有端点的速查。`$BASE` = `http://localhost:3001`（默认）。
所有 POST/PUT 的 body 是 JSON，返回也是 JSON（导出文件下载除外）。

## 健康检查

```
GET $BASE/api/health
→ { "ok": true, "service": "vizkit" }
```

## Workspace 管理

### 列出所有模板
```
GET $BASE/api/workspaces
→ { templates: [ { workspaceId, name, chartType, status, businessPurpose, fields, ... } ] }
```
每个模板带 `businessPurpose`（一句话说明用途）和 `fields`（需要的数据字段），
**一次请求就能看清所有模板的用途和数据需求**，据此选模板：

```json
{
  "name": "财务瀑布图",
  "chartType": "bar",
  "workspaceId": "template-bafpom",
  "status": "ready",
  "businessPurpose": "展示利润表各项的增减变化",
  "fields": [
    { "key": "category", "type": "string", "role": "dimension" },
    { "key": "value", "type": "number", "role": "measure" }
  ]
}
```
- `businessPurpose`：判断这个模板是否适合你的数据场景
- `fields`：每个元素的 `key` 就是你 `data` 里每行要带的字段名；
  `type` 是 `category`/`number`/`date`/`string`；`role` 是 `dimension`/`measure`/`series`

选好模板后，想看长什么样：`GET /api/workspaces/:id/thumbnail.png`。

### 创建空壳（新建模板第一步）
```
POST $BASE/api/workspaces
{ "name": "季度销售柱状图", "chartType": "bar" }
→ { workspace: { workspaceId: "xxx", name, chartType, ... } }
```
- `name`：显示名（用户语言，建议 ≤12 中文字）
- `chartType`：自由字符串（bar/line/gauge/任何词），系统不分支
- 可选 `scenarios` / `industries` / `purposes`：业务分类三轴（`string[]`），
  预填到 manifest。标准 key 见 `create-template.md`。留空时可在后续通过
  PUT manifest.json 或 POST `/api/workspaces/:id/taxonomy` 补填。

### 获取单个模板信息
```
GET $BASE/api/workspaces/:id
```

### 软删除（进回收站）
```
DELETE $BASE/api/workspaces/:id
```

## 文件读写

### 读文件
```
GET $BASE/api/workspaces/:id/file?path=manifest.json
→ { path, content }   // content 是字符串
```

### 写文件（核心端点）
```
PUT $BASE/api/workspaces/:id/file
{
  "path": "renderer.tsx",
  "content": "<字符串或 JSON 对象>",
  "validate": true,              // 可选：写后自动跑预览校验
  "onValidateFail": "rollback"   // 可选：失败时回滚（默认 keep）
}
```

`content`：
- 字符串 → 原样写入（renderer.tsx）
- JSON 对象 → 自动 pretty-print 写入（manifest/schema/data/values）

`validate: true` 只对图表文件（manifest/config.schema/sample-data/config.values/renderer）生效：
- **成功** → `200 { path, bytes, validation: { ok: true, warnings: [...] } }`
- **失败** → `422 { error: { ok:false, stage, message }, written: true }`

`stage` 值：`json`（JSON 解析失败）| `compile`（编译或配置检查失败）| `render`（运行时错误）

详见 [`create-template.md`](./create-template.md) 的"失败自检循环"。

### 列出文件
```
GET $BASE/api/workspaces/:id/files
→ { files: [ "manifest.json", "renderer.tsx", ... ] }
```

## 预览 / 校验

### 渲染预览（返回 SVG，可覆盖数据/配置）
```
POST $BASE/api/workspaces/:id/preview
{
  "data": [...],              // 可选：覆盖 sample-data
  "configOverrides": {...},   // 可选：覆盖 config.values
  "width": 640, "height": 360 // 可选
}
→ 200 { preview: { ok: true, svg: "<svg>...", width, height } }
→ 422 { error: { ok:false, stage, message } }
```
preview 只返回 SVG（不出 PNG）。要 PNG 用下面的 export。

## 导出

### 导出图片（SVG/PNG/整包，可覆盖数据/配置）
```
POST $BASE/api/workspaces/:id/export
{
  "format": "png",            // svg（省略时默认）| png | workspace
  "data": [...],              // 可选：覆盖 sample-data（svg/png）
  "configOverrides": {...},   // 可选：覆盖 config.values（svg/png，浅合并）
  "width": 1280,              // 可选：PNG 像素宽（resvg 缩放）
  "height": 720,              // 可选：PNG 像素高
  "background": "with"        // 可选：with（默认，画背景）| transparent（PNG 保留 alpha）
}
→ 200 { ok: true, path: "preview/exports/chart-xxx.png", bytes, metadata: {...} }
→ 422 { ok: false, error: "preview failed (stage): message" }
```
**注意**：
- `format: "workspace"` 是 ZIP 整包（源文件），**忽略** `data`/`configOverrides`/`width`/`height`/`background`。
- **区域开关**（关标题/图例/副标题/脚注）没有顶层字段，通过 `configOverrides` 实现：
  `configOverrides: { showTitle: false, showLegend: false, ... }`（对应 schema 里的 `show*` key）。
- `format` 省略时服务端默认 `svg`（不是 png）。

覆盖数据的详细用法见 [`apply-data.md`](./apply-data.md)。

### 下载导出文件
```
GET $BASE/api/workspaces/:id/export-file?path=preview/exports/chart-xxx.png
→ 图片字节流（Content-Type 对应 svg/png/zip）
```

## CLI 等价命令

`cli/chart-tool.mjs`。`--base URL` 或 `$CHART_BASE` 覆盖服务地址。

| API | CLI |
|-----|-----|
| GET /health | `chart-tool health` |
| POST /workspaces | `chart-tool new "名称" [--type bar]` |
| GET /workspaces | `chart-tool list` |
| GET /workspaces/:id | `chart-tool get <id>` |
| GET /workspaces/:id/files | `chart-tool files <id>` |
| GET /file | `chart-tool read <id> <path>` |
| PUT /file | `chart-tool put <id> <path> <file\|-> [--validate] [--on-fail rollback]` |
| (拉取模板文件到本地) | `chart-tool pull <id> [dir]` |
| (本地改完推回) | `chart-tool push <id> [dir] [--no-validate] [--no-stop]` |
| POST /preview | `chart-tool preview <id> [--data <file>]` |
| POST /preview {} | `chart-tool validate <id>` |
| POST /export | `chart-tool export <id> [--format svg\|png\|workspace] [--data <file>] [--config <file>] [--out <file>] [--background with\|transparent] [--width N] [--height N] [--no-title] [--no-subtitle] [--no-legend] [--no-footer] [--set key=value ...]` |
| DELETE /workspaces/:id | `chart-tool delete <id>` |
| GET /geo/catalog | `chart-tool geocatalog` |
| POST /geo/import | `chart-tool geoimport <id> <name>` |

`export` 的便捷开关（`--no-title`/`--no-subtitle`/`--no-legend`/`--no-footer`）等价于
`configOverrides.showX = false`；`--set key=value`（可重复）按 `key=value` 覆盖任意配置项
（数字/布尔/null 会自动转类型）。`--config` 文件优先级高于 `--set`。

## 扩展端点（生命周期与元数据）

创建/出图之外，这些端点对编排场景（批量管理、版本、报告配方）有用。每个给方法+路径+用途，
完整字段语义见 `server/workspaceRoutes.ts`。

### 元数据与分类
| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/workspaces/:id/rename` | 改模板名 `{ name }`（同步 manifest + DB） |
| POST | `/api/workspaces/:id/taxonomy` | 改业务分类三轴 `{ scenarios?, industries?, purposes? }`，每轴可选、部分更新 |
| GET | `/api/workspaces/:id/thumbnail.png` | 取缩略图（PNG 字节流，选模板/拼报告时预览用） |

### 地图数据
| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/api/geo/catalog` | 列出内置地理边界数据集（如 `china-province`） |
| POST | `/api/workspaces/:id/geo/import` | `{ name }` 把一个内置边界集复制进 workspace 的 `assets/geo.json`，地图 renderer 才能画真实边界 |

### 版本与复制
| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/workspaces/:id/versions` | 建命名版本快照 `{ version?, label? }` |
| GET | `/api/workspaces/:id/versions` | 列版本快照 |
| POST | `/api/workspaces/:id/versions/:version/restore` | 从指定版本恢复文件 |
| POST | `/api/workspaces/:id/save-as` | 另存为新 workspace `{ name, workspaceId? }` |
| POST | `/api/workspaces/import` | 从 ZIP 导入 workspace `{ zipBase64, name? }`（`format:"workspace"` 导出的逆操作） |

### 发布与回收站
| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/workspaces/:id/publish` | 标记 `ready`（跑业务用途+渲染校验，内容变更时建版本） |
| POST | `/api/workspaces/:id/unpublish` | 退回 `draft` |
| GET | `/api/workspaces/trashed` | 列回收站里的已删模板 |
| POST | `/api/workspaces/:id/restore` | 从回收站恢复 |
| POST | `/api/workspaces/:id/purge` | 永久删除（磁盘 + DB，不可恢复） |

### 附件
| 方法 | 路径 | 用途 |
|------|------|------|
| POST | `/api/workspaces/:id/attachments` | 上传 AI 附件 `{ attachments: [{ id, name, mimeType, size, kind, dataUrl }] }`，存到 `attachments/` |

> 这些端点大多没有 CLI 封装——需要时直接发 HTTP。`agent/status`、`agent/skill`、
> `save`/`draft-*`、`files-hash`、`renderer-client-module.js` 等是内置 UI/agent 用的，
> 外部 agent 一般不需要。
