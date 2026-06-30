# `@ai-chart/layout` 壳 API 速查

`@ai-chart/layout` 是注入好的模块（无需安装），每个 renderer **必须**用它（E3 校验）。
本文档是壳的完整 API 参考，从 `src/layout/` 提炼。建/改模板前对照本文档选组件和 frame。

> 配套阅读：[`create-template.md`](./create-template.md)（5 文件契约 + 校验规则）、
> [`refine-template.md`](./refine-template.md)（改已有模板）。

---

## 组件总览

```tsx
import {
  ChartCanvas, Title, Subtitle, Legend, Footnote, ChartArea,
  // 组合原语与布局预设（free frame 内部的多视图/分面）
  LayeredChart, SmallMultiples, Facet, HStack, VStack, LayoutPreset,
} from "@ai-chart/layout";
```

一个最小 renderer 的骨架：

```tsx
<ChartCanvas size={{ width, height }} config={config}>
  <Title ... />
  <Subtitle ... />
  <ChartArea frame="cartesian" cartesian={{...}}>
    {({ rect, scales, palette, clipPathId, toSvgPoint }) => ( ... )}
  </ChartArea>
  <Legend ... />
  <Footnote ... />
</ChartCanvas>
```

子组件靠**静态 layoutRole 自识别**，书写顺序随意；但 `<ChartArea>` 只能有一个。

---

## ChartCanvas —— 画布外壳（必需）

```tsx
interface ChartCanvasProps {
  size: { width: number; height: number };  // 必需，画布像素
  config?: Record<string, unknown>;          // 传整个 config 对象
  padding?: number;                          // 外边距 px，默认 16
  children?: React.ReactNode;                // 下方各 slot 组件
}
```

`ChartCanvas` 负责：画 `<svg>` 根 + 背景矩形（读 `config.backgroundColor`）、
按 padding/title/subtitle/legend/footnote 顺序**分配区域**、把剩余的安全矩形交给 `<ChartArea>`。

它从 `config` 读可见性开关：`showTitle` / `showSubtitle` / `showLegend` / `showFootnote`
（值为 `false` 时对应区域不渲染，即便组件存在）。这就是为什么这些 key 由壳代读、E1/E2 豁免。

调色板 `palette` 从 `config` 的 `primaryColor` / `secondaryColor` / `accentColor` 解析，
传给 `<ChartArea>` 的 render-prop。

---

## Title / Subtitle / Footnote —— 文本区

三者共享同一组 props：

```tsx
interface TextBlockSpec {
  text?: string | null;
  fontSize?: number;
  color?: string;
  align?: "left" | "center" | "right";
  maxLines?: number;
}
```

壳的默认值：
- **Title**：fontSize 18、color `#111827`、字重 700
- **Subtitle**：fontSize 12、color `#6b7280`、字重 400（未设 `align` 时继承 title 的）
- **Footnote**：fontSize 11、color `#6b7280`

```tsx
<Title text={config.title} fontSize={config.titleFontSize} align={config.titleAlign} />
<Subtitle text={config.subtitle} />
<Footnote text={config.note} />
```

> 这些文本内容（title/subtitle/note）及其排版 key 由壳渲染，E1/E2 对它们豁免。
> 但你声明的**图表专属色**（如 primaryColor 用于自定义图例）仍要 renderer 自己读。

---

## Legend —— 图例

两种模式：

**① 分类图例**（最常见）—— 传 `items`，壳自己画色块：

```tsx
interface LegendItem { label: string; color: string; }

<Legend
  position="bottom"                              // "top" | "bottom" | "left" | "right"
  items={[{ label: "营收", color: primaryColor }]}
  fontSize={12}                                  // 可选，默认 12
  color="#374151"                                // 可选，标签色，默认 #374151
/>
```

**② 自定义图例**（如连续渐变条）—— 传 `size` + `children` render-prop：

```tsx
<Legend
  position="bottom"
  size={{ width: 200, height: 24 }}              // 声明精确占地，壳据此预留
>
  {(rect) => (
    // rect 是壳预留给你的安全矩形，画你的渐变条/自定义图例进这里
    <g>
      <rect x={rect.x} y={rect.y} width={rect.width} height={rect.height} fill="url(#grad)" />
      <text x={rect.x + rect.width + 6} y={rect.y + rect.height / 2}>高</text>
    </g>
  )}
</Legend>
```

`visible?: boolean` 可选（默认受 `config.showLegend` 控制）。

---

## ChartArea —— 图表区（核心）

```tsx
interface ChartAreaProps {
  frame?: "cartesian" | "polar" | "free";        // 默认 "free"
  cartesian?: { xCount: number; yDomain: [number, number]; yNice?: boolean };  // frame="cartesian" 时必需
  polar?: { count: number; valueDomain: [number, number] };                    // frame="polar" 时必需
  children?: (props: ChartAreaRenderProps) => React.ReactNode;
}
```

`children` 是一个 **render-prop**，收到这些字段：

```tsx
interface ChartAreaRenderProps {
  rect: Rect;                  // {x,y,width,height} —— 安全绘图区
                               //   cartesian 时已扣除坐标轴 gutter
  clipPathId: string;          // 壳定义的 clipPath id，套在 marks 上防溢出
  palette: { primary; secondary; accent; positive; negative; neutral; text };
                               // 从 config 解析的语义色：
                               //   primary/secondary/accent ← primaryColor/secondaryColor/accentColor
                               //   positive/negative ← positiveColor/negativeColor（locale 相关，cn 涨红跌绿）
                               //   neutral/text    ← neutralColor/textColor
                               // 涨跌色用 palette.positive/negative，别写 #22C55E 字面量（E5 会拦）
  warnings: string[];          // 布局警告（如区域过小）
  scales: CartesianScales | PolarScales | null;   // free 时为 null
  toSvgPoint: (e) => { x, y }; // 鼠标事件 → SVG 用户坐标，做 tooltip 用
}
```

### rect 怎么用

```tsx
<ChartArea frame="cartesian" cartesian={{ xCount: data.length, yDomain: [0, yMax], yNice: true }}>
  {({ rect, scales, palette, clipPathId }) => (
    <g clipPath={`url(#${clipPathId})`}>
      {data.map((d, i) => (
        <rect
          x={scales.x(i) - bw / 2}          // ← 用 scales，别手算
          y={scales.y(d.value)}
          width={bw}
          height={rect.y + rect.height - scales.y(d.value)}  // ← rect 算柱底
          fill={palette.primary}
        />
      ))}
    </g>
  )}
</ChartArea>
```

`rect.y + rect.height` 是绘图区底部（柱子的 baseline）。永远不要手算 `height - val/max*height`。

### toSvgPoint —— tooltip 坐标

```tsx
{({ toSvgPoint }) => {
  const [pt, setPt] = useState(null);
  return <rect onMouseMove={(e) => setPt(toSvgPoint(e))} ... />;
}}
```

它内部用 `getScreenCTM().inverse()`，在 CSS `transform: scale()`、`zoom`、viewBox 不一致时都正确。
**永远用它定位 tooltip，不要 `getBoundingClientRect` 手算。**

---

## 三种 frame 的 scales

### cartesian（柱/线/面积/域图）

```tsx
interface CartesianScales {
  x: (i: number) => number;        // 类别索引 → 带中心 x（已 clamp）
  y: (v: number) => number;        // 数值 → y（反转，越大越靠上）
  xInvert: (px: number) => number; // x 像素 → 最近类别索引
  yInvert: (py: number) => number; // y 像素 → 数据值
  bandwidth: number;               // 单带宽度（柱宽用）
  plotRect: Rect;                  // scales 映射的矩形
}
```

`cartesian={{ xCount, yDomain: [min,max], yNice?: true }}`：
- `xCount` = x 轴类别数；`yDomain` = `[最小值, 最大值]`
- `yNice: true` 会把 **yMax** 向上跳档到 `1/2/2.5/5/10 × 10ⁿ`（**只升不降，按档位上限归档**），
  并在 `yMin > 0` 时把 min 归零（柱子落地）。
  - ⚠️ **跳档是激进的**：传 258 会跳到 **500**（m=2.58 落进 `m≤5` 档 → 5×10²），膨胀近一倍，
    顶部凭空多出一大截空白。
  - ⚠️ **不要叠加 nice**：如果你自己已经 `d3.nice(...)` 取整过 yMax，再传 `yNice:true`，
    shell 会**二次跳档膨胀**（258→300→500）。要么交给 shell nice（传原始 yMax + `yNice:true`），
    要么自己 nice（传已 nice 的域 + **`yNice:false`**），二选一。
  - **何时用哪个**：柱状图/不关心顶部留白 → `yNice:true` 让刻度落整；折线图/需要精确控制顶部
    headroom（如末端标签空间）→ 自己 `d3.nice(0, rawMax*1.1, 5)[1]` 算好 + `yNice:false`。

壳**自动预留坐标轴 gutter**，所以你收到的 `rect` 已是轴安全的——标记不会压到 y 轴标签下。
gutter 大小（来自 `axisGutter`，可调 `axisLabelFontSize`）：**左 ≈ `max(28, yAxisWidth||字号×4)`，
底 ≈ `max(20, 字号×2)`**。你画的 X 轴标签/刻度应放在 `rect` 内部底缘，不要再额外往下挤。

> **刻度数值一致性**：壳的 scale **不提供** `ticks()` API，坐标轴刻度由 renderer 自己画。
> 生成刻度数值时，**传给 `d3.ticks(...)` 的域必须等于传给 `cartesian.yDomain` 的域**
> （且若你 `yNice:false`，域就是你 `d3.nice` 后的值）。否则刻度文字与折线/柱子高度错位。
> 刻度位置用 `scales.y(tickValue)`，不要手算 `height - val/max*height`。

### polar（雷达/极坐标）

```tsx
interface PolarScales {
  angle: (i: number) => number;          // 类别索引 → 弧度（-π/2 起，顺时针）
  radius: (v: number) => number;         // 数值 → 像素半径
  angleInvert: (rad: number) => number;  // 弧度 → 类别索引
  radiusInvert: (r: number) => number;   // 像素半径 → 数据值
  plotRect: Rect;
}
```

`polar={{ count, valueDomain: [min,max] }}`。
`angle`/`radius` 返回的是标量，你要自己用 `Math.cos/sin` 围绕中心（从 `plotRect` 取）转成笛卡尔坐标：

```tsx
<ChartArea frame="polar" polar={{ count: axes.length, valueDomain: [0, max] }}>
  {({ rect, scales, palette }) => {
    const cx = rect.x + rect.width / 2;
    const cy = rect.y + rect.height / 2;
    return axes.map((axis, i) => {
      const a = scales.angle(i);                  // 弧度
      const r = scales.radius(values[i]);         // 像素半径
      return (
        <line
          x1={cx} y1={cy}
          x2={cx + Math.cos(a) * r} y2={cy + Math.sin(a) * r}
          stroke={palette.primary}
        />
      );
    });
  }}
</ChartArea>
```

### free（桑基/弦/网络/词云/地图/KPI/饼）—— scales 为 null

```tsx
{({ rect }) => {
  // scales 是 null；你拿原始 rect，用 d3 算几何，自由画
  const layout = sankey().extent([[1, 1], [rect.width - 1, rect.height - 1]]);
  // ... 注意 free frame 左侧标签要预留 gutter 或加翻转 guard（E4）
}}
```

#### frame 决策表

| frame | 用于 | scales 给什么 |
|-------|------|---------------|
| `"cartesian"` | 柱/线/面积/域 | `x(i)` 带中心、`y(v)`、`bandwidth`、反函数 |
| `"polar"` | 雷达/极坐标 | `angle(i)` 弧度、`radius(v)`、反函数 |
| `"free"`（默认） | 桑基/弦/网络/词云/地图/KPI/饼 | `null`——拿原始 `rect` 自由画（d3 算几何） |

> **free frame 注意 E4**：左侧标签（`node.x0 - 8` 这类）必须有 gutter 预留或宽度翻转 guard，
> 否则被 E4 拒绝。详见 create-template.md 硬规则⑤/E4。
>
> **不要把 layout 写进 frame**：`dashboard` / `ibcs-variance` / `small-multiples`
> 是布局预设或组合模式，不是坐标系。用 `frame="free"` 或 `frame="cartesian"`
> 拿到 `rect/scales` 后，再用下方组合原语或 `<LayoutPreset>`。

---

## 组合原语（多视图 / 分面）

这些是独立组件，**在 `<ChartArea>` 的 render-prop 内部**用（它们需要一个 `rect`）。
都通过 render-prop 把切分后的子矩形交给你。

### LayeredChart —— 多图层叠加（组合图：柱+线）

不切分 rect，所有层画在**同一个**区域：

```tsx
<LayeredChart rect={rect} scales={scales} palette={palette}>
  {({ rect, scales, palette }) => (
    <>
      {/* 图层 1：柱 */}
      {data.map(...)}
      {/* 图层 2：线（可从 rect 重建自己的 scale） */}
    </>
  )}
</LayeredChart>
```

### SmallMultiples —— 小多图网格（Zebra BI 式）

```tsx
<SmallMultiples
  rect={rect}
  groups={[rowGroup1, rowGroup2, ...]}   // 每个子图一份数据
  cols={2}
  gap={8}
  syncScales={{ yDomain: "auto" }}        // 可选：所有子图共享 y 轴，可比
  palette={palette}
>
  {({ rect, data, index, yDomain, palette }) => (
    // 每个格子：用 data + 共享的 yDomain 画一个小图
  )}
</SmallMultiples>
```

`syncScales.yDomain`：省略/null = 各自定义域；`"auto"` = 跨组算共享域；`[min,max]` = 显式。

### Facet —— 按字段自动分面

```tsx
<Facet
  rect={rect}
  data={allRows}
  field="region"                          // 该字段的每个不同值 → 一个子图
  cols={3}
  syncScales={{ yDomain: "auto" }}
  palette={palette}
>
  {({ rect, value, data, index, yDomain, palette }) => (
    // value 是该面的字段值（如 "华东"），data 是该面的行
  )}
</Facet>
```

SmallMultiples 的数据驱动版本；分组保持首次出现顺序。

### HStack / VStack —— 多视图并排/堆叠（仪表盘）

```tsx
<HStack rect={rect} count={2} gap={8}>
  {({ rect, index, palette }) => (
    // index 0..count-1，每个 rect 是切出来的一块
  )}
</HStack>
```

VStack 同理，纵向切分。适合仪表盘里把一张图画区分成几个独立视图。

### LayoutPreset —— HTML 设计稿中的 8 种布局案例

`LayoutPreset` 是 `docs/chart-layout-primitives.html` 中 8 种 layout case 的代码化入口。
它返回语义化 slots；renderer 仍负责真正画 marks。它不替代 `frame`，而是在
`ChartArea` 的 render-prop 内使用。

支持的 `kind`：

```ts
"combo" | "small-multiples" | "facet" | "kpi" |
"radar" | "dashboard" | "sankey" | "ibcs-variance"
```

示例：Dashboard（hconcat）

```tsx
<ChartArea frame="free">
  {({ rect, palette }) => (
    <LayoutPreset rect={rect} kind="dashboard" count={3} gap={12} palette={palette}>
      {({ slots }) => slots.map((slot) => (
        // slot.role === "cell"; 在 slot.rect 内画一个子视图
      ))}
    </LayoutPreset>
  )}
</ChartArea>
```

示例：IBCS 方差图（cartesian + 三层 mark + annotation）

```tsx
<ChartArea frame="free">
  {({ rect }) => (
    <LayoutPreset rect={rect} kind="ibcs-variance" axisWidth={44}>
      {({ slots }) => {
        const plot = slots.find((s) => s.role === "plot");
        const layers = slots.filter((s) => s.role === "layer");
        const annotation = slots.find((s) => s.role === "annotation");
        // 在 plot/layers/annotation 的 rect 中画实际值、方差、基准线和标注
      }}
    </LayoutPreset>
  )}
</ChartArea>
```

---

## 工具函数（从 `@ai-chart/layout` 直接 import）

壳除了组件，还导出一批**纯函数**——遇到几何/文本/域计算，**先用这些，不要手算**：

```tsx
import {
  estimateTextWidth,    // (text, fontSize) → 估算渲染宽度（px），按字符类别精确算
                        //   CJK 全宽、标点窄、大小写不同。用于标签碰撞/翻边判断。
  wrapText,             // (text, {maxWidth, fontSize, maxLines}) → 截断/折行字符串数组
  computeChartLayout,   // (input) → 完整布局（titleRect/chartArea/legend...），自己重排时用
  createCartesianScales,// (rect, opts) → 不经 ChartArea 直接造 scale（自定义子图时用）
  createPolarScales,
  buildLayoutPreset,    // (rect, {kind}) → 布局预设 slots
  unionDomain,          // (groups) → 多组数据合并 y 域（small multiples 同刻度用）
} from "@ai-chart/layout";
```

**最高频**：`estimateTextWidth(text, fontSize)`。任何"标签会不会超出/重叠"的判断，
都用它量宽，**不要写 `text.length * 0.6`** 这种手算（中英文混排、标点宽度都估错）。
E4 的宽度感知翻边 guard 也认 `labelW = estimateTextWidth(...)` 这个写法。

---

## 辅助：Rect 类型

```tsx
interface Rect { x: number; y: number; width: number; height: number; }
```

所有 `rect`（ChartArea、Legend render-prop、组合原语）都是这个形状，单位是 SVG 用户空间像素。

---

## 常见坑

1. **cartesian 别手算坐标**——用 `scales.x(i)` / `scales.y(v)` / `scales.bandwidth`，柱底用 `rect.y + rect.height`。
2. **polar 要自己 cos/sin**——`scales.angle`/`radius` 返回标量，围绕 `rect` 中心转笛卡尔。
3. **free frame 没有 scales**——拿 `rect` 自己用 d3 算，注意 E4 左侧标签 guard。
4. **tooltip 用 toSvgPoint**——别 `getBoundingClientRect`，CSS 变换下会算错。
5. **`clipPathId` 套在 marks 上**——`<g clipPath={...}>` 防止标记溢出绘图区。
