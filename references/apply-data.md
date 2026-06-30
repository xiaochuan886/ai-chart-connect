# 应用数据出图（能力②）

本文档教你（外部 Agent）如何**用真实数据套用已有模板，渲染出图**——
这是数据分析报告、批量出图的基础。

核心 API：`POST /api/workspaces/:id/export`，body 加 `data` 和/或 `configOverrides` 字段。
**workspace 的文件永远不会被修改**——覆盖只影响本次渲染。

---

## 一句话流程

```
你有一份数据 + 一个已存在的模板 workspaceId
  ↓
POST /api/workspaces/:id/export  { format: "png", data: <你的数据> }
  ↓
返回 { ok: true, path: "preview/exports/chart-xxx.png", metadata: {...} }
  ↓
GET /api/workspaces/:id/export-file?path=<上面的 path>  →  下载图片
```

## data 覆盖：用你的数据渲染

`data` 是一个 JSON 数组，每个元素是一行记录，key 对齐 `manifest.fields[].key`：

```
POST $BASE/api/workspaces/:id/export
Content-Type: application/json
{
  "format": "png",
  "data": [
    { "label": "一月", "value": 500 },
    { "label": "二月", "value": 800 },
    { "label": "三月", "value": 300 }
  ]
}
```

**关键语义**：
- 模板按你的 data 渲染，不读磁盘上的 `sample-data.json`
- `sample-data.json` 文件**不会被修改**（它是模板的示例数据，保持原样）
- 返回的 `metadata.dataHash` 反映的是**你的 data**，不是 sample-data

怎么知道有哪些模板、各自要什么字段？**先 list 一次看清全部**：

```
GET $BASE/api/workspaces
→ { templates: [
     { name, workspaceId, businessPurpose, fields: [{key,type,role}, ...], ... },
     ...
   ] }
```
`businessPurpose` 告诉你模板的用途，`fields` 告诉你 data 每行要带哪些 key。
据此选出适合你数据的模板，再按它的 fields 组织 data。

查单个模板的字段也可以直接读 manifest：

```
GET $BASE/api/workspaces/:id/file?path=manifest.json
→ { fields: [ { key: "label", type: "category", ... }, { key: "value", type: "number", ... } ] }
```

你的 data 每行就要有这些 key（如 `label` + `value`）。

## configOverrides：同时覆盖配置

想出图时顺便改标题、配色等配置？加 `configOverrides`（浅合并，覆盖优先）：

```
POST $BASE/api/workspaces/:id/export
Content-Type: application/json
{
  "format": "png",
  "data": [...],
  "configOverrides": { "title": "2024 Q1 销售", "primaryColor": "#dc2626" }
}
```

可覆盖哪些 key？读 `config.schema.json` 的 `properties`：

```
GET $BASE/api/workspaces/:id/file?path=config.schema.json
→ { properties: [ { key: "title", ... }, { key: "primaryColor", ... }, ... ] }
```

`config.values.json` 文件同样**不会被修改**。

## 完整 curl 示例（已验证）

```bash
# 1. 用数据渲染并导出 SVG
EXPORT=$(curl -s -X POST "$BASE/api/workspaces/abc123/export" \
  -H "Content-Type: application/json" \
  -d '{"format":"svg","data":[{"label":"一月","value":500},{"label":"二月","value":800}]}')
# → { "ok": true, "path": "preview/exports/chart-xxx.svg", "metadata": {...} }

# 2. 下载文件
curl -s "$BASE/api/workspaces/abc123/export-file?path=preview/exports/chart-xxx.svg" -o chart.svg
```

## CLI 示例（已验证）

```bash
# data 放文件里
echo '[{"label":"周一","value":300},{"label":"周二","value":450}]' > data.json
npm run chart-tool -- export abc123 --format svg --data data.json --out chart.svg

# 同时覆盖配置
echo '{"title":"周报","primaryColor":"#dc2626"}' > config.json
npm run chart-tool -- export abc123 --format png --data data.json --config config.json --out chart.png

# 便捷开关：套数据出图时临时关掉标题/图例/副标题/脚注（等价 showX=false）
npm run chart-tool -- export abc123 --format png --data data.json --no-title --no-legend --out chart.png

# --set 单项覆盖（可重复，数字/布尔/null 自动转类型）；优先级低于 --config 文件
npm run chart-tool -- export abc123 --format png --data data.json \
  --set primaryColor=#16a34a --set titleFontSize=22 --out chart.png

# 透明背景 PNG（alpha 保留，便于叠到任意底色）
npm run chart-tool -- export abc123 --format png --data data.json --background transparent --out chart.png
```

## format 选项

| format | 输出 | 用途 |
|--------|------|------|
| `svg` | 矢量图 | 嵌网页、再编辑、无损 |
| `png` | 位图 | 插入文档/报告、分享 |
| `workspace` | ZIP 整包 | **不接受 data 覆盖**（打包源文件，不渲染） |

PNG 可加 `width` 控制像素宽（图按模板原尺寸渲染后 resvg 缩放）：
```json
{ "format": "png", "width": 1280, "data": [...] }
```

## 失败处理

导出失败返回 `422`：

```json
{ "ok": false, "error": "preview failed (compile): ..." }
```

常见原因：
- `data` 的 key 和 manifest.fields 不匹配 → renderer 读不到值，可能渲染空图（但不报错）
- renderer 本身坏了（如果模板就是坏的）→ stage: `compile` 或 `render`
- data 不是数组 → 静默当成空数组（渲染空图，不报错）

**注意**：data 不匹配字段不会报错（renderer 自己决定怎么用 data），
所以出图后最好下载看一眼，确认图里有你的数据。

## 用这个能力做什么

- **数据分析报告**：一个模板 + 多份数据 → 批量出多张图，拼进报告
- **定期周报**：固定模板 + 当周数据 → 每周同样的图，数据不同
- **数据探索**：快速把一份数据可视化，看趋势/分布

要把多张图 + 分析文字组装成完整报告（或做定期周报），
见 [`build-report.md`](./build-report.md)——那里有报告工作流、配方格式、cron 示例。
