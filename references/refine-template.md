# 改已有模板（能力⑤）

本文档教你（外部 Agent）如何**在已存在的模板上做增量修改**——换色、加标题、
显示数值标签、改字号、加图例、调交互。核心 API 还是 `PUT /api/workspaces/:id/file`，
关键在于**改完要让改动真正在预览里可见**。

> 先读 [`create-template.md`](./create-template.md) 的「5 个文件契约」和渲染规则（硬规则⓪-⑥），
> 本文只讲"在已有模板上改"的增量流程和最容易踩的坑。

---

## 第一部分：改模板的工作流（pull → 本地改 → push）

改已有模板的推荐方式是**把文件拉到本地、用你顺手的工具改、再推回去**。
这样你能用编辑器/diff/版本控制，比逐个 HTTP 调用改更顺手，也不容易丢内容。

### 步骤 0：pull —— 把 5 个契约文件拉到本地

```bash
chart-tool pull <id> [dir]        # 默认 dir = ./<id>
```

`pull` 下载这 5 个文件到本地目录（`renderer.tsx` 原样，4 个 `.json` 已 pretty-print 成可读 JSON）：

```
manifest.json   config.schema.json   sample-data.json   config.values.json   renderer.tsx
```

这就是你改一个模板会碰的**全部**文件。

> 不想拉全套？也可以只读单个文件：`chart-tool read <id> renderer.tsx`（`GET /file?path=`）。
> 但改模板时一般 5 个都要看，`pull` 一次拿全更省事。

### 步骤 1：本地改

在本地目录里改对应文件。**先读再改，别盲改**——你面对的是已有模板，直接覆盖会丢内容。
改完用编辑器/diff 自检。

按下方「决策表」确定改哪几个文件。

### 步骤 2：push —— 推回去并自检

```bash
chart-tool push <id> [dir]        # 默认 dir = ./<id>，默认每个文件带 --validate
```

`push` 按**依赖顺序**逐个 PUT：`manifest → config.schema → sample-data → config.values → renderer`（renderer 最后写，校验时前面已就位）。每个文件写完跑 `validate`。

- **全部成功** → 每个文件返回 `{ ok: true, validation: {...} }`
- **某个文件校验失败** → 默认**停在该文件**（`--stop-on-fail`，默认开），报出错的文件名 + `stage` + `message`，已成功的前面的文件已写入。修好那个文件再 `push`（或只 `put` 那一个）。
- 想跳过校验：`--no-validate`；想失败不停继续推：`--no-stop`（不推荐——一个坏 renderer 会让后续校验全失败）。

### 步骤 3：预览/导出确认改动可见

```bash
chart-tool preview <id>           # 渲染当前状态，返回 SVG
chart-tool export <id> --format png --out chart.png
```

**一定要下载看一眼**，确认用户的改动真的出现在图里（见下方头号陷阱）。

### 纯 HTTP 等价写法（不用 CLI 时）

```bash
# 读
GET  $BASE/api/workspaces/:id/file?path=renderer.tsx
# 改完写回（单个文件）
PUT  $BASE/api/workspaces/:id/file  { "path": "renderer.tsx", "content": "...", "validate": true }
```

### 决策表：改哪几个文件

| 情况 | 改什么 |
|------|--------|
| **key 已在 schema 里 且 renderer 已读它** | 只改 `config.values.json`（最省、最安全） |
| **key 不在 schema，或 renderer 没读它** | **三件套同改**：`config.schema.json`（加声明）+ `renderer.tsx`（读并渲染）+ `config.values.json`（设值） |
| **结构/几何/交互改动**（轴布局、柱形、tooltip、hover、点击高亮） | 只改 `renderer.tsx` |
| **数据契约改动**（加字段、改列名） | `manifest.json` + `sample-data.json` + `renderer.tsx` 一起改 |
| **业务分类改动**（"归到金融行业""这是经营汇报用的"） | 只改 `manifest.json` 的 `scenarios`/`industries`/`purposes`，或 `POST /api/workspaces/:id/taxonomy` |

> **⚠️ push 失败时的副作用**：服务端 PUT 文件是**先写后校验**——校验失败时文件**已经写入了**
> （`written: true`）。所以一个语法错的 renderer 推上去后，会让该 workspace **后续所有校验都失败**
> （因为校验要重新编译当前 renderer）。这就是 `push` 默认 `--stop-on-fail` 的原因：坏文件一出现就停，
> 别让后续 PUT 踩着坏 renderer。万一推坏了一个文件，修好本地版本重新 `push`（或单独 `put` 那个文件）即可恢复。

---

## 第二部分：头号陷阱 —— 改了但看不见

**最常见的失败：改了 `config.values.json` 但 renderer 根本没读那个 key，改动对用户不可见。**

```
你设了 config.values.json 里 primaryColor = "#16a34a"
  ← 但 config.schema.json 没声明 primaryColor？
  ← 或 renderer.tsx 没读 config.primaryColor（而是硬编码了颜色）？
  → 改动 INVISIBLE。静态校验可能仍通过（如果 schema 没声明这个 key），但图没变。
```

**改完前，心里过一遍这条链：**
> "我改的这个 config key，renderer 里有没有 `pick(config, "key", ...)` 或
> `config.key` 真去读它，并且渲染成可见的 SVG/HTML 内容？"

如果 renderer 没读 → 你**必须同时改 renderer**。只改 config + schema 不改 renderer，
是改模板时**最容易犯的错**。

### 典型例子

用户："给这个仪表盘加个标题"

- ❌ 只在 config.schema 加 `title`、config.values 设 `title: "销售达成"` → renderer 没读 → 图上没标题。
- ✅ 三件套：schema 加 `title`、renderer 里 `const title = pick(config, "title", "")` 并画一个 `<text>`、values 设值。

---

## 第三部分：常见改动 → 改哪个 key

| 用户说 | 改哪个 config key | 备注 |
|--------|------------------|------|
| 换成科技蓝 / 蓝色 | `primaryColor` | 确认 renderer 读它（多 series 要派生，见下） |
| 绿色 / 次要色 | `secondaryColor` | |
| 隐藏/显示图例 | `showLegend` | 壳消费 |
| 显示/隐藏数值标签 | `showLabels` | renderer 要读 |
| 标题改成 X | `title` | 确保 renderer 读并渲染 |
| 副标题 | `subtitle` | |
| 整体字号大小 | `textScale`（`small`/`standard`/`large`） | |
| 标题字号 | `titleFontSize` | |
| 标题对齐 | `titleAlign`（`left`/`center`/`right`） | |
| 文字颜色 | `textColor`（`auto` 或 `#RRGGBB`） | |
| 坐标轴字号/颜色 | `axisLabelFontSize` / `axisLabelColor` | |
| 数值标签字号/颜色 | `valueLabelFontSize` / `valueLabelColor` | |

> 用了 `@ai-chart/layout` 壳的模板：title/subtitle/background 等由壳代读，
> 改这些 key 改 `config.values.json` 就生效。但**图表专属 key**（primaryColor 用于自定义图例、
> showLabels 等）仍要 renderer 自己读——改前先读 renderer 确认。

---

## 第四部分：多 series 配色 —— 派生，不要逐 series 声明

如果你碰到一个多 series 图用了 `lineColor1`/`lineColor2`/`seriesColor3`/`nodeColor`
这种**逐 series 的私有 key**，那是 bug：一键主题预设只改 4 个标准色
（`primaryColor`/`secondaryColor`/`backgroundColor`/`accentColor`），
逐 series key 会让换色静默失效。

正确做法：从 `primaryColor` 用 d3.hsl 色相旋转派生整个调色板：

```tsx
import * as d3 from "d3";
function derivePalette(baseHex: string, count: number): string[] {
  const n = Math.max(1, count);
  const h = d3.hsl(baseHex);
  return Array.from({ length: n }, (_, i) => {
    const hue = (((h.h ?? 0) + 360) % 360 + (i * 360) / n) % 360;
    return d3.hsl(hue, h.s ?? 0.6, h.l ?? 0.45).formatHex();
  });
}
const seriesColors = derivePalette(String(config.primaryColor ?? "#3b82f6"), rows.length);
```

改的时候：renderer 换成派生逻辑，schema/config.values 删掉那些私有 key。

---

## 第五部分：业务分类（taxonomy）改动的快捷方式

用户说"归到金融行业""这是经营汇报用的""改成趋势分析"——只动业务分类三轴，
**不需要改 renderer 或 config**。两种方式：

**方式 A：PUT manifest.json**（整块覆盖，先读再改）：
```
GET /file?path=manifest.json   → 拿到现有 manifest
（本地把 scenarios/industries/purposes 改好）
PUT /file  { path: "manifest.json", content: <改后的 manifest> }
```

**方式 B：POST /taxonomy**（部分更新，每轴可选，推荐）：
```
POST $BASE/api/workspaces/:id/taxonomy
{ "scenarios": ["finance-analysis"], "industries": ["finance"], "purposes": ["trend"] }
```
每轴可选——只传你想改的轴，其余保持不变。标准 key 见 create-template.md 的分类表。

---

## 第六部分：发布前的一个坑（business purpose gate）

如果你改完要 `POST /api/workspaces/:id/save`（保存）或 `/publish`（标记 ready），注意：
**save 和 publish 都要求 `business-notes.json` 的 `purpose` 非空**，否则返回 422：

```json
{ "error": "请先在「业务说明」里填写用途（purpose），它是模板的身份说明，保存/发布前必填。", "field": "business-purpose" }
```

`purpose` 是模板的身份说明（mirror 进 `manifest.businessPurpose`）。如果模板还没填，先写：

```
PUT $BASE/api/workspaces/:id/file
{ "path": "business-notes.json", "content": { "purpose": "这张图回答什么业务问题", "audience": "管理层", "notes": "" } }
```

`business-notes.json` 的**正式 schema 是严格三字段**（`purpose` 必填非空，`audience`/`notes` 可空），
多余字段在 import/解析路径会被 `.strict()` 拒。但注意：通过 `PUT /file` 直接写时，
`business-notes.json` **不在 chart-file 白名单**（5 个白名单文件才走 `validate`），
所以多余的 key 会**先落盘成功**——但后续保存/发布会被 purpose gate 拦住，或在 import 时报错。
所以一开始就只写这三字段，别塞 `chartName`/`keyMetrics` 之类的多余结构。
只是改样式/出图**不需要** save/publish——preview/export 随时可用。

---

## 提交前自检清单

- [ ] 我改的每个 config key，renderer 都有读取并渲染（不是只改了 config.values）
- [ ] 我在 schema 新加的每个 property，renderer 都读取了
- [ ] schema 声明了 `backgroundColor`/`accentColor` → renderer 真读了（没硬编码绕过）
- [ ] 多 series 配色从 `primaryColor` 派生，没有 `lineColor1` 这种私有 key
- [ ] 新加/改动的 `label`/`description`/默认文案是用户语言（默认简体中文），没混语言
- [ ] PUT renderer/config.schema 带 `validate:true` 返回 200
- [ ] 下载预览/导出图**亲眼确认**改动可见
