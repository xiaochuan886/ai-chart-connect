# 创建图表模板 —— 完整指南

本文档教你（外部 Agent）如何通过 HTTP API 创建一个**可预览、可导出**的图表模板。
你生成 5 个契约文件的内容，逐个写入，写 renderer 时自动校验，失败就修，直到成功。

> 这是**从零建**新模板。如果要**改已存在的模板**（换色/加标题/改交互），
> 读 [`refine-template.md`](./refine-template.md)——改模板有不同的流程（先读现状、防"改了看不见"）。

---

## 第一部分：操作流程（怎么调 API）

### 步骤 0：探活

```
GET $BASE/api/health
→ { "ok": true }
```

### 步骤 1：建空壳

```
POST $BASE/api/workspaces
Content-Type: application/json
{ "name": "季度销售柱状图", "chartType": "bar" }
```

`name` 是模板显示名（建议 ≤12 个中文字符，专业、非占位）。
`chartType` 自由字符串（bar / line / gauge / sankey / 任何词），系统不会据此分支。

返回里取 `workspaceId`（例如 `template-abc123`），后续都用它。

### 步骤 2：逐个写入 5 个文件

用同一个端点，区别只在 `path` 和 `content`：

```
PUT $BASE/api/workspaces/:id/file
Content-Type: application/json
{ "path": "<文件名>", "content": <字符串或对象> }
```

`content` 可以是字符串（renderer.tsx）或 JSON 对象（其余文件，会自动 pretty-print）。

按这个顺序写（后面的依赖前面的字段定义）：

| 顺序 | path | 说明 |
|------|------|------|
| 1 | `manifest.json` | 图表身份 + 数据字段契约 |
| 2 | `config.schema.json` | 用户可调的配置项声明 |
| 3 | `sample-data.json` | 示例数据（rows 的 key 对齐 manifest.fields） |
| 4 | `config.values.json` | 每个配置项的当前值 |
| 5 | `renderer.tsx` | 渲染组件（React/TSX）—— **带 `validate: true`** |

### 步骤 3：写 renderer.tsx 时自动校验（关键）

```
PUT $BASE/api/workspaces/:id/file
Content-Type: application/json
{ "path": "renderer.tsx", "content": "<renderer 源码>", "validate": true }
```

- **成功** → `200 { path, bytes, validation: { ok: true, warnings: [...] } }`
- **失败** → `422 { error: { ok: false, stage, message }, written: true, path }`

`validate:true` 只对图表文件（manifest/config.schema/sample-data/config.values/renderer）生效；
对 README/TODO 等无影响。

### 步骤 4：失败自检循环（核心机制）

收到 `422` 时，读 `error.stage` 判断问题类型：

| stage | 含义 | 怎么修 |
|-------|------|--------|
| `json` | 某个 JSON 文件解析失败 | 检查 manifest/config.schema/sample-data 的 JSON 语法 |
| `compile` | renderer 编译失败 **或** 违反配置使用规则 | 看 message：esbuild 语法错误（带行号）；`renderer must read config.XXX`（E1/E2：声明了配置项却没读，用 `pick(config,"XXX",…)` 读进来）；`renderer hardcodes #XXXXXX … equals the default of config.XXX`（E5：在 `fill=`/`stroke=` 里写了标准/语义色——含 backgroundColor/accentColor/positiveColor/negativeColor/neutralColor/textColor——的默认值字面量。把字面量换成读到的值：`pick(config,"XXX",…)`，或对涨跌色用 `palette.positive`/`palette.negative`（来自 ChartArea render-prop），而非 `fill="#22C55E"`/`fill="#E54B4B"`。注意：涨绿跌红的默认值在 cn/intl 两个 locale 下都会被拦，切 locale 也逃不掉）。 |
| `render` | 组件运行时抛错 | 看 message 里的堆栈，修 renderer 逻辑 |

修好后**重新 PUT 同一文件**（带 `validate: true`），直到返回 `200`。
**默认失败时保留写入**（`written: true`），你直接覆盖重写即可。
若想失败时回滚到写入前，加 `"onValidateFail": "rollback"`（一般不需要）。

### 步骤 5（可选）：导出 / 整体验证

全部写完后，可以用 `POST /preview` 看整体渲染，或导出图片：

```
POST $BASE/api/workspaces/:id/preview  { }          → 渲染当前状态，返回 SVG
POST $BASE/api/workspaces/:id/export   { "format": "png" }   → 生成图片
GET  $BASE/api/workspaces/:id/export-file?path=<preview/exports/xxx.png>  → 下载
```

或告诉用户打开 `http://localhost:5173`，在画廊里直接看到并编辑这个模板。

---

## 第二部分：5 个文件的精确契约

### 1. `manifest.json` —— 身份与数据字段

```json
{
  "id": "<同 workspaceId>",
  "name": "季度销售柱状图",
  "description": "一句话描述",
  "chartType": "bar",
  "version": "0.1.0",
  "fields": [
    { "key": "label", "label": "类别", "type": "category", "role": "dimension" },
    { "key": "value", "label": "数值", "type": "number", "role": "measure" }
  ],
  "businessPurpose": "这个图回答什么业务问题",
  "scenarios": ["sales-recap"],
  "industries": ["ecommerce"],
  "purposes": ["comparison"]
}
```

规则：
- `id` / 每个 `fields[].key` 必须匹配正则 `/^[A-Za-z0-9_.\-]+$/`（**英文标识符**）。
- `fields` 至少 1 个。
- `fields[].type` 只能是：`category` | `number` | `date` | `string`
- `fields[].role` 可选，只能是：`dimension` | `measure` | `series`
- `chartType` 任意非空字符串（系统不分支，由你的 renderer 定义图表）。
- `name` 用用户语言（默认中文），不要用"未命名模板"这种占位词。
- **`scenarios` / `industries` / `purposes`** 是业务分类三轴（多选数组），
  决定模板在画廊里出现在哪些筛选分类下。强烈建议根据图表用途填写——空数组
  的模板在按场景/行业/目的筛选时不可见。标准 key 见下表，清单外的
  kebab-case 自定义值也合法（画廊会自动聚合到筛选条）：

  | 轴 | 含义 | 标准 key（部分） |
  |---|---|---|
  | `scenarios` | 用在什么场合 | `operations-review` `sales-recap` `finance-analysis` `user-growth` `campaign-review` `kpi-dashboard` … |
  | `industries` | 给哪个行业 | `ecommerce` `finance` `technology` `saas` `education` `healthcare` `manufacturing` … |
  | `purposes` | 呈现什么结论 | `trend` `comparison` `composition` `funnel` `correlation` `ranking` `kpi` … |

### 2. `config.schema.json` —— 配置项声明

```json
{
  "templateId": "<同 workspaceId>",
  "chartType": "bar",
  "properties": [
    { "key": "title", "label": "标题", "type": "string", "default": "季度销售", "group": "text" },
    { "key": "primaryColor", "label": "主色", "type": "color", "default": "#2563eb", "group": "style" }
  ]
}
```

**`type` 只能是这 7 个值**：`string` | `number` | `boolean` | `color` | `select` | `text` | `multiline`
- `number` 可带 `min` / `max` / `step`
- `select` 必须带 `options: string[]`

**`group` 只能是这 9 个英文值**（填中文会被 422 拒绝）：

| group | 用途 | 典型 key |
|-------|------|---------|
| `text` | 文本内容 | title, subtitle |
| `typography` | 排版 | 字号、颜色、对齐 |
| `style` | 样式 | primaryColor 等颜色 |
| `legend` | 图例 | showLegend, legendPosition |
| `labels` | 标签 | showLabels |
| `size` | 尺寸 | width, height |
| `analysis` | 分析 | showChartNote, chartNote |
| `chart` | 图表专属 | nodeWidth, showPercent 等任意图表特有项 |
| `general` | 通用兜底 | 少用 |

`key` 用英文（稳定 id）；`label` / `description` / `default` 文本用用户语言（默认中文）。

> **⚠️ 硬必需配置项（漏写直接 422）**：schema 校验会强制要求以下 key 必须声明，
> 否则 `PUT config.schema.json` / commit / publish 全部被拒，报
> `missing required property "xxx"`：
> - **所有 chartType**：`showTitle`（开关标题区）、`backgroundColor`（画布+透明导出）
> - **轴类图表**（bar/line/area 等几乎所有类型）：额外必需 `showLegend`（开关图例区）。
>   （仅少数无轴类型如纯 KPI/饼可豁免 showLegend——拿不准就声明它，宁多勿缺。）
>
> 这些是 layout shell 消费的协议级开关，**不是可选建议**。下面"建议"列表里的其余 key
>（title/primaryColor/width 等）才是可选的（但强烈建议加上，让模板可调）。

#### 建议总是声明的通用配置项

为了让模板响应用户的日常微调，建议 `properties` 里至少包含这些标准 key
（renderer 必须真读它们，见下文强制规则）：

- 文本：`title`, `subtitle`
- 颜色：`primaryColor`, `secondaryColor`, `backgroundColor`, `accentColor`
- 尺寸：`width`, `height`
- 排版：`textScale`(`small`/`standard`/`large`), `textColor`(`auto`或`#RRGGBB`),
  `titleAlign`(`left`/`center`/`right`), `titleFontSize`, `titleColor`,
  `subtitleFontSize`, `subtitleColor`, `axisLabelFontSize`, `axisLabelColor`,
  `valueLabelFontSize`, `valueLabelColor`, `legendFontSize`, `legendColor`,
  `noteFontSize`, `noteColor`
- 分析：`showChartNote`, `chartNote`

### 3. `sample-data.json` —— 示例数据

**顶层是数组**（不是对象），每个元素的 key 对齐 `manifest.fields[].key`：

```json
[
  { "label": "Q1", "value": 120 },
  { "label": "Q2", "value": 180 },
  { "label": "Q3", "value": 95 },
  { "label": "Q4", "value": 240 }
]
```

### 4. `config.values.json` —— 配置当前值

扁平的 `key → value` 映射，key 对齐 `config.schema.properties[].key`：

```json
{
  "title": "季度销售",
  "primaryColor": "#2563eb",
  "secondaryColor": "#94a3b8",
  "width": 640,
  "height": 360,
  "showLabels": true
}
```

### 5. `renderer.tsx` —— 渲染组件（重点）

**每个 renderer 必须用 `@ai-chart/layout` 壳**（硬规则⓪，见下文）。
下面是一个通过全部校验（E1/E2/E3/E4）的完整 cartesian 柱状图示例：

```tsx
import React from "react";
import {
  ChartCanvas,
  Title,
  Subtitle,
  ChartArea,
  Legend,
  Footnote,
} from "@ai-chart/layout";
import * as d3 from "d3";

// pick 是 renderer 内的本地惯用 helper（不是 import）——每个 renderer 自己定义
function pick<T>(values: Record<string, unknown>, key: string, fallback: T): T {
  const v = values[key];
  return v === undefined || v === null ? fallback : (v as T);
}

export function Chart({ data, config }: {
  data: any[];
  config: Record<string, unknown>;
}) {
  // 读你在 schema 声明的配置项（壳会代读 title/subtitle/background，见下文豁免）
  const width = Math.max(320, Number(pick(config, "width", 800)));
  const height = Math.max(240, Number(pick(config, "height", 500)));
  const values = (data || []).map((d) => Number(d.value) || 0);
  const yMax = Math.max(1, ...values);

  return (
    <ChartCanvas size={{ width, height }} config={config}>
      <Title text={pick(config, "title", "")} fontSize={pick(config, "titleFontSize", 18)} />
      <Subtitle text={pick(config, "subtitle", "")} />
      <ChartArea
        frame="cartesian"
        cartesian={{ xCount: (data || []).length, yDomain: [0, yMax], yNice: true }}
      >
        {({ rect, scales, palette, clipPathId }) => (
          <g clipPath={`url(#${clipPathId})`}>
            {(data || []).map((d, i) => {
              const bw = Math.max(2, scales.bandwidth * 0.7);
              const v = Number(d.value) || 0;
              return (
                <rect
                  key={i}
                  x={scales.x(i) - bw / 2}
                  y={scales.y(v)}
                  width={bw}
                  height={rect.y + rect.height - scales.y(v)}
                  fill={palette.primary}
                />
              );
            })}
          </g>
        )}
      </ChartArea>
      <Legend position="bottom" items={[{ label: "数值", color: String(pick(config, "primaryColor", "#3b82f6")) }]} />
      <Footnote text={pick(config, "note", "")} />
    </ChartCanvas>
  );
}

export default Chart;
```

要点：
- `import React from "react"` 必须是第一行（否则编译报 `React is not defined`）。
- `<ChartCanvas>` 是**强制**的外壳：它负责画布、背景、标题/副标题/图例/脚注的区域分配，并把剩下的安全矩形交给 `<ChartArea>`。手写 `<svg>` 壳会被 E3 拒绝。
- `<ChartArea frame="cartesian">` 的 render-prop 给你 `rect`（已扣除坐标轴 gutter 的安全绘图区）、`scales`（`scales.x(i)` 返回带中心 x、`scales.y(v)` 返回 y、`scales.bandwidth` 是带宽）、`palette`（`primary`/`secondary`/`accent` 三个色）、`clipPathId`。
- 用了壳之后，`title`/`subtitle`/`backgroundColor` 等键由壳代读，E1/E2 对这些键自动豁免（见硬规则⑤的豁免说明）。但你声明的**图表专属色**（如 `primaryColor` 用于 `<Legend>`）仍要自己读。

> 壳的完整 API（所有组件 props、frame 决策、scales/toSvgPoint 细节、polar/free 用法、组合原语）
> 见 [`layout-shell.md`](./layout-shell.md)。本示例只给最小可用骨架。

---

## 第三部分：渲染规则（写出能通过校验的 renderer）

### 渲染范式：选对工具

按复杂度选一档（三档最终都是 React 组件 + `@ai-chart/layout` 的 `<ChartCanvas>` 壳）：

- **Tier 1（d3 有现成布局）**：柱/线/饼/桑基/弦/力/层级/地理 —— 用 d3 算坐标，JSX 渲染。代码最少、最精确。
- **Tier 2（d3 算 + 自定义渲染）**：定制桑基/交互式弦 —— d3 做数学，JSX 做形状/交互。
- **Tier 3（纯 React + 数学）**：仪表盘指针、漏斗、KPI 卡、进度环等 d3 没布局的 —— 用 `Math.sin/cos/PI/atan2` 算，直接写 SVG 元素。

**先试 d3；d3 没合适的布局就用纯 React + 基础数学。永远不要幻想一个不存在的库。**

```tsx
import * as d3 from "d3";            // 主包，算坐标用
import { sankey } from "d3-sankey";  // 子包（d3-sankey 已预装；主包 d3 没有桑基布局）
```

允许的外部库只有：`react`（必需）、`@ai-chart/layout`（壳，必需）、`d3` 主包、已安装的 `d3-*` 子包。**没有别的。**

### 硬规则⓪：必须用 @ai-chart/layout 壳（E3 校验）

**这是最重要的规则。** 每个 renderer 必须把输出包在 `<ChartCanvas>` 里，从 `@ai-chart/layout` 导入：

```tsx
import React from "react";
import {
  ChartCanvas, Title, Subtitle, Legend, Footnote, ChartArea,
} from "@ai-chart/layout";

export function Chart({ data, config }) {
  return (
    <ChartCanvas size={{ width: config.width ?? 800, height: config.height ?? 500 }} config={config}>
      <Title ... />
      <Subtitle ... />
      <ChartArea frame="..."> {({ rect, scales, ... }) => (...)} </ChartArea>
      <Legend ... />
      <Footnote ... />
    </ChartCanvas>
  );
}
```

手写自己的 `<svg>` 壳（自己画背景、自己定位标题/图例）会被 **E3 校验** 以 `stage: compile` 拒绝。
壳替你解决三个最常见的 bug：标签溢出画布、图例位置错乱、透明背景导出破损。

#### frame 决策（选 `<ChartArea>` 的 frame）

| frame | 用于 | scales 给你什么 |
|-------|------|----------------|
| `"cartesian"` | 柱/线/面积/域 | `scales.x(i)`（带中心）、`scales.y(v)`、`scales.bandwidth`、反函数 |
| `"polar"` | 雷达/极坐标 | `scales.angle(i)`（弧度）、`scales.radius(v)` |
| `"free"`（默认） | 桑基/弦/网络/词云/地图/KPI/饼 | `scales` 是 `null`——你拿原始 `rect` 自由画（d3 算几何） |

cartesian 传 `cartesian={{ xCount, yDomain: [min,max], yNice?: true }}`；
壳会自动在 `rect` 内预留坐标轴 gutter，所以你收到的 `rect` 已经是轴安全的——标记不会压到 y 轴标签下。

> **`yNice` 用法（极易踩坑）**：`yNice:true` 把 yMax **跳档**到 `1/2/2.5/5/10×10ⁿ`（只升不降），
> 传 258 会跳到 **500**，顶部多一大截空白。规则：
> - 柱状图/不在意顶部留白 → 传原始 yMax + `yNice:true`，让刻度落整
> - 折线图/要给末端标签留 headroom → 自己 `d3.nice(0, rawMax*1.1, 5)[1]` 算精确域 + **`yNice:false`**
> - **绝不叠加**：别自己 nice 一遍又开 `yNice:true`，会二次膨胀（258→300→500）
>
> **刻度数值**：壳不提供 `ticks()`，刻度由 renderer 画。用 `d3.ticks(yMin, yMax, 5)` 生成数值，
> **域必须等于 `cartesian.yDomain`**，位置用 `scales.y(tickValue)`。详见 layout-shell.md。

> free frame 时，**你必须自己在 rect 内预留左侧 gutter**，否则左侧标签会溢出画布（见硬规则④/E4）。

壳的完整参考（所有组件 props、render-prop 字段、组合原语）：
[`layout-shell.md`](./layout-shell.md)。

### 硬规则①：绝不手写 path 坐标字符串

这是 AI 最常犯的错（几何不准 + token 爆炸）。用**声明式 JSX + 计算值**：

```tsx
// ✅ 对：坐标是 d3 算出来的变量
<rect x={x(d.label)} y={y(d.value)} width={x.bandwidth()} height={innerH - y(d.value)} />
// ✅ 对：path 字符串来自生成器
<path d={sankeyLinkHorizontal()(link)} />
// ❌ 错：手写一串数字坐标（会猜错位置、超长、数据一变就错）
<path d="M123.4,56.7 C200,80 300,150 380,200..." />
```

### 硬规则②：d3 只用于计算，渲染走 JSX

```tsx
// ✅ 用 d3 的 scale / layout / 生成器
const x = d3.scaleBand().domain(...).range(...);
const pathStr = d3.arc()(...);   // 返回字符串
// ❌ 禁止 d3.select(node).call(axis) 这类 DOM 操作
//    原因：导出用 renderToStaticMarkup，不触发 ref，d3 写的 DOM 会丢失
```

### 硬规则③：生成器输出可能是中心线，不是闭合形状

来自 `sankeyLinkHorizontal()` 的 path 是开放曲线（没有 `Z`），`fill` 看不见 → 必须用 `stroke`。
只有闭合形状（`d3.arc`、`d3.ribbon`）才用 `fill`。
判断：path 字符串以 `M` 开头、`Z` 结尾吗？没有 `Z` 就是中心线 → 用 stroke。

### 硬规则④：几何值都要有正向下限

```tsx
const radius = Math.max(20, computedRadius);   // 防止负半径
const w = Math.max(1, computedWidth);           // 防止负宽高，否则 d3 抛错
```

### 硬规则⑤：rendererConfigCheck 四层校验（最常导致 422 的坑，务必遵守）

校验阶段会**静态检查** renderer。违反任意一层都会被 `422 stage:compile` 拒绝：

**E1 — 必读色 key**：声明了 `backgroundColor` 或 `accentColor` → renderer 必须读它
（`pick(config, "backgroundColor", "#fff")` 或 `config.backgroundColor` 或解构）。
硬编码 `const bg = "#ffffff"` 而忽略声明的配置项 → **被拒**。

**E2 — 排版契约**：读了 `title` → 也必须读 `titleColor` / `titleFontSize` / `titleAlign`（若 schema 声明了它们）；
读了 `subtitle` → 也必须读 `subtitleColor` / `subtitleFontSize`。

**E3 — 必须用壳**（见硬规则⓪）：renderer 必须从 `@ai-chart/layout` 导入并用 `<ChartCanvas>`。
手写 `<svg>` 壳 → **被拒**。

**E4 — 左侧标签溢出保护**（针对 `frame="free"` 的桑基/网络/树等）：
当标签 x 是 `node.x0 - 8` 这类「节点坐标减偏移」时，标签会延伸到节点左侧，
若没有保护就会溢出画布左边缘 → **被拒**。必须满足下面**任一**合法 guard：

- **Guard A — 预留左侧 gutter**（节点根本不靠近边缘）：
  ```tsx
  const leftGutter = 60;  // ≥ 最长标签宽
  const layout = sankey().extent([[leftGutter, top], [innerW, innerH]]);
  ```
- **Guard B — 宽度感知翻转**（估算标签宽，会溢出就翻进节点内部）：
  ```tsx
  const labelW = estimateTextWidth(labelText, fs);
  const labelRight = node.x0 - 6;
  if (labelRight - labelW < 0) {        // 会顶出左边缘
    lx = node.x0 + 4; la = "start";     // 翻进节点内部
  } else {
    lx = labelRight; la = "end";
  }
  ```
  E4 认得这种 guard 的信号：一个宽度估算变量（`labelW =` / `needW =` / `textWidth =` …）
  配一个左边界比较（`x - labelW < …`、`leftEdge >= …`）。裸的 `x0 - 8` 两者皆无 → 被拒。

> **壳豁免（SHELL_HANDLED_KEYS）**：当你用了 `@ai-chart/layout` 壳，`backgroundColor` /
> `title` / `titleColor` / `titleFontSize` / `titleAlign` / `subtitle*` / `footnote*` /
> `note*` / `legend*` 这些键由壳内部代读，E1/E2 对它们自动豁免——你不用在 renderer 里重复读。
> 但 `accentColor` 以及你的**图表专属 key**（如 `primaryColor` 用于自定义图例）仍要自己读。

> 自检技巧：收到 `renderer must read config.XXX` → E1/E2 拦你（把那个 key 读进来）；
> 收到 `renderer must use @ai-chart/layout` → E3（改用壳）；
> 收到 `no overflow guard` → E4（加 gutter 或宽度翻转）。message 里都会写清楚。
> 收到 `renderer hardcodes #XXXXXX … equals the default of config.XXX` → E5：你在
> `fill="…"` / `stroke="…"` 里写了某个标准/语义色（primaryColor/secondaryColor/backgroundColor/
> accentColor/positiveColor/negativeColor/neutralColor/textColor）的默认值字面量。把它替换成
> 你读到的 config 变量或 palette 字段：背景 `fill={bg}`、强调色 `fill={accent}`、
> 涨跌色 `fill={palette.positive}` / `palette.negative`（来自 ChartArea render-prop），
> 别再写 `fill="#22C55E"` / `fill="#ffffff"`。这是**最高频复发**的失误——只读一次 config
> 满足 E1、却继续用字面量画，会让主题预设和样式面板完全失效。现在 E5 是硬错误，会直接拦住
> preview，必须改掉。涨跌色默认值在 cn（涨红跌绿）/intl（涨绿跌红）两个 locale 下都纳入匹配，
> 切 locale 也逃不掉；多系列颜色用 `deriveSeriesColors(primary, n, config.paletteMode)` 派生，
> 不要逐 series 声明颜色键。

### 硬规则⑥：多 series 颜色从 primaryColor 派生，不要逐 series 声明

一键主题预设只改 4 个标准色（`primaryColor`/`secondaryColor`/`backgroundColor`/`accentColor`）。
逐 series 的 key 会让换色静默失效。用 `primaryColor` 做色相旋转派生整个调色板：

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

// 组件内：
const primary = String(config.primaryColor ?? "#3b82f6");
const seriesColors = derivePalette(primary, rows.length);
```

### 硬规则⑦：几何/文本/域计算先用 layout 提供的工具函数，别手算

`@ai-chart/layout` 除了组件，还导出一批纯函数（完整列表见 layout-shell.md「工具函数」）。
遇到对应计算，**先用现成的**——手算不仅易错（中英文混排宽度、CJK 全角），还会绕开壳的布局逻辑
导致和 scale/rect 对不齐：

```tsx
import { estimateTextWidth } from "@ai-chart/layout";

// ✅ 标签会不会超出/重叠：用 estimateTextWidth 量宽
const labelW = estimateTextWidth(text, fontSize);
if (x + labelW > rect.x + rect.width) { /* 翻边 */ }

// ❌ 别手算字符宽度（中英文/标点/CJK 宽度都不同，估错）
const labelW = text.length * fontSize * 0.6;
```

高频函数：`estimateTextWidth`（标签宽/碰撞/E4 guard）、`wrapText`（折行截断）、
`computeChartLayout`（自己重排）、`createCartesianScales`（自定义子图造 scale）、
`unionDomain`（多组共享 y 域）。坐标值用 `scales.x/y`，文本宽用 `estimateTextWidth`，
**永远不要 `height - val/max*height` 或 `len*0.6` 手算**。

### 交互

需要 tooltip / hover / 点击高亮：用真正的 React state + 事件
（`useState` + `onMouseEnter` / `onMouseMove` / `onClick`）。不要只靠 SVG `<title>`。

### 语言规则

- 英文（协议标识符）：`key`、`group`、`templateId`、`chartType`、`fields[].key`
- 用户语言（默认简体中文）：`label`、`description`、`name`、`default` 文本、`chartNote`
- 一个模板内不要混语言（别中文 name 配英文 label）。

---

## 第四部分：提交前的自检清单

写完 5 个文件、renderer 通过 `validate` 后，逐条核对：

- [ ] manifest.fields 至少 1 个；type 是 4 个合法值之一
- [ ] manifest 的 scenarios / industries / purposes 已按图表用途填写（空数组的模板在场景/行业/目的筛选下不可见）
- [ ] config.schema 的每个 type 是 7 个合法值之一；group 是 9 个英文值之一
- [ ] config.values 的每个 key 都在 config.schema 里声明过
- [ ] **renderer 用 `@ai-chart/layout` 的 `<ChartCanvas>` 壳，没手写 `<svg>` 壳（E3）**
- [ ] renderer 真读了 schema 声明的图表专属色（`accentColor` 等）；title/subtitle/background 由壳代读可豁免（E1/E2）
- [ ] **若用 `frame="free"`：左侧标签有 gutter 预留 或 宽度感知翻转 guard（E4）**
- [ ] 没有手写 path 坐标字符串（都是 d3 生成器或计算值）
- [ ] 多 series 颜色从 `primaryColor` 派生，没有 `lineColor1` 这种私有 key
- [ ] **没有手算坐标/文本宽**：坐标用 `scales.x/y`，标签宽用 `estimateTextWidth`，没写 `len*0.6` 或 `height-val/max*height`（硬规则⑦）
- [ ] **`yNice` 没有双重膨胀**：要么 `yNice:true` 传原始 yMax，要么自己 nice + `yNice:false`，不叠加
- [ ] **刻度域 == cartesian 域**：`d3.ticks(yMin, yMax, n)` 的域与 `cartesian.yDomain` 完全一致
- [ ] label / description / name / 默认文案是中文（或匹配用户语言）
- [ ] `PUT renderer.tsx` 带 `validate:true` 返回 200

全绿后，模板就创建好了。
