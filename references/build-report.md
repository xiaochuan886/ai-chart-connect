# 生成分析报告 / 定期周报（能力③④）

本文档教你（外部 Agent）如何把**多张图表 + 分析文字**组装成一份图文并茂的报告，
以及如何用配方文件 + 定时任务实现制式的定期报告（周报/月报）。

核心思想：**报告 = ① 选模板 + ② 批量套数据出图 + 你的分析文字**。
不需要新的后端能力，全部用你已经会的 API 编排。
你拥有充分的自主性——报告的结构、图表数量、分析深度、输出格式都由你根据场景决定。

> 先读 [`apply-data.md`](./apply-data.md) 确保你掌握了"用 data 覆盖出图"。
> 本文档只讲"怎么把多张图 + 文字拼成报告"。

---

## 第一部分：分析报告（能力③）

### 你拥有的积木

| 要做什么 | 用什么 API |
|---------|-----------|
| 看有哪些模板、各自适合什么数据 | `GET /workspaces`（带 businessPurpose + fields） |
| 看某个模板长什么样 | `GET /workspaces/:id/thumbnail.png` |
| 用真实数据渲染某张图 | `POST /workspaces/:id/export { data, format }` |
| 下载渲染好的图片 | `GET /workspaces/:id/export-file?path=...` |

### 报告生成的工作流（参考，不是死步骤）

一份报告通常经过这几个阶段。**具体的顺序、图表数量、分析方式由你自主决定**——
下面是一个典型的、验证过的流程，你可以根据数据特点调整：

**1. 理解数据，规划图表**
分析用户给你的数据（或描述），决定：
- 这份数据值得用几张图？
- 每张图讲什么故事？（趋势？对比？结构？达成？）
- 各自适合什么图表类型？（折线看趋势、柱状看对比、饼图看结构、仪表盘看达成……）

**2. 为每张图选模板**
`GET /workspaces` 一次看清所有模板的 `businessPurpose`（用途）和 `fields`（需要什么字段）。
根据每张图要讲的故事 + 数据字段，匹配最合适的模板。不确定时可以 `GET /thumbnail.png` 看长什么样。

**3. 批量出图**
对每张图，把对应的数据通过 `POST /export { data }` 套进选中的模板，再 `GET /export-file` 下载 PNG。
这就是一个循环——你有几张图就循环几次。

**4. 写分析文字**
这是你的强项。看着每张图（你可以从 SVG 内容或数据本身推断），写出洞察：
- 这张图说明了什么？
- 有什么值得注意的趋势 / 异常 / 对比？
- 对读者有什么建议？

**5. 组装报告**
把图表和分析文字拼成最终产物。两种格式用户都可能需要：

**Markdown 报告**（日常分析、可继续编辑）：
```markdown
# 2024 年 Q3 经营分析

## 利润结构
本季度营业收入 1200 万，成本控制良好，营业利润率达 47.5%。

![季度利润瀑布图](charts/chart1-profit.png)

## 关键指标达成
营收超额完成（120%），但利润率略低于目标（90%），需关注成本。

![KPI 达成仪表盘](charts/chart2-kpi.png)
```
图片用**相对路径**（图片文件和 .md 放在同一目录或子目录），保持报告可移植。

**HTML 报告**（自包含单文件、排版精美、可直接分享）：
生成一个 `.html` 文件，图片用 **base64 内联**（`<img src="data:image/png;base64,...">`），
这样单文件就能打开，不依赖外部图片。可以加简单 CSS 让排版好看。

一个把 PNG 转成 base64 data URI 的方法（Agent 执行）：
```bash
# macOS/Linux
base64 -i chart.png | tr -d '\n' | awk '{print "data:image/png;base64,"$0}'
```

### 真实的批量出图示例（已验证）

下面是两步真实跑通的出图——一份经营分析报告的两张图：

```bash
BASE=http://localhost:3001
mkdir -p report/charts

# 图1：财务瀑布图 ← 季度利润数据（category/value/type 匹配模板 fields）
EXPORT1=$(curl -s -X POST "$BASE/api/workspaces/template-bafpom/export" \
  -H "Content-Type: application/json" \
  -d '{"format":"png","data":[
    {"category":"营业收入","value":1200,"type":"income"},
    {"category":"营业成本","value":-480,"type":"cost"},
    {"category":"销售费用","value":-150,"type":"cost"},
    {"category":"营业利润","value":570,"type":"profit"}
  ]}')
PATH1=$(echo "$EXPORT1" | python3 -c "import sys,json; print(json.load(sys.stdin)['path'])")
curl -s "$BASE/api/workspaces/template-bafpom/export-file?path=$PATH1" -o report/charts/chart1-profit.png

# 图2：KPI 仪表盘 ← 指标达成数据（category/target/actual/rate 匹配模板 fields）
EXPORT2=$(curl -s -X POST "$BASE/api/workspaces/template-upfm2j/export" \
  -H "Content-Type: application/json" \
  -d '{"format":"png","data":[
    {"category":"营收","target":1000,"actual":1200,"rate":1.2},
    {"category":"利润率","target":20,"actual":18,"rate":0.9}
  ]}')
PATH2=$(echo "$EXPORT2" | python3 -c "import sys,json; print(json.load(sys.stdin)['path'])")
curl -s "$BASE/api/workspaces/template-upfm2j/export-file?path=$PATH2" -o report/charts/chart2-kpi.png
```

### 数据来源（由你灵活处理）

报告的数据从哪来？**这完全由你和用户约定**，常见方式：
- **用户直接粘贴**：用户把数据贴在对话里，你解析后用
- **读取本地文件**：用户指向一个 CSV/JSON 文件，你读取后转换成 API 要的数组格式
- **数据库查询**：如果环境允许，你执行 SQL 查询拿到数据

**字段对齐是关键**：不管数据从哪来，你都要把它整理成模板 `fields` 要求的形状。
比如模板要 `category`+`value`，你的数据每行就得有这两个 key。
`GET /workspaces` 返回的 `fields` 就是这个契约。

### CLI 批量出图（不想手写 curl 时）

```bash
# 准备每张图的数据文件
echo '[{"category":"营收","target":1000,"actual":1200,"rate":1.2}]' > kpi-data.json
npm run chart-tool -- export template-upfm2j --format png --data kpi-data.json --out report/charts/kpi.png
```

---

## 第二部分：定期周报（能力④）

定期周报 = **固定的报告配方 + 定时触发**。
本质是能力③的"参数化 + 自动化"版本：报告的结构、用哪些模板是固定的，
每次跑只是换数据 + 生成。

### 配方文件格式（建议，非强制）

配方描述"这份周报长什么样"。格式由你决定，下面是一个清晰的 Markdown frontmatter 方案，
你可以按需调整：

```markdown
---
report: 经营周报
schedule: "0 9 * * 1"           # cron 表达式：每周一 9 点（仅供参考，实际 cron 在外部配）
output: reports/weekly-{date}.md  # {date} 跑时替换成当天日期
charts:
  - name: 利润结构
    template: template-bafpom     # workspaceId（从 gallery「复制 ID」按钮拿到）
    data: data/profit-{date}.csv  # 数据文件（每次跑前由其他脚本准备好）
    format: png
  - name: KPI 达成
    template: template-upfm2j
    data: data/kpi-{date}.csv
    format: png
analysis: |
  本周营收 {营收实际}，达成率 {达成率}。{你根据数据生成的洞察}
---

# 经营周报 ({date})

## 利润结构
![利润结构](charts/0-profit.png)

## KPI 达成
![KPI 达成](charts/1-kpi.png)

## 本周分析
（Agent 读数据后填充）
```

**关键约定**：
- `{date}` 等占位符在跑时替换
- `data:` 指向数据文件——**这些文件由用户的 ETL / 导出脚本提前准备好**，Agent 只负责读
- `template:` 是 workspaceId（用户在 gallery 点「复制 ID」拿到）
- 配方只是给 Agent 的参考，**Agent 保留自主性**：可以根据当次数据的特点调整图表选择和分析内容

### 定时触发：外部 cron

定时调度在用户的系统层面做（cron / launchd / 任何调度器），不在本项目后端。
一个 cron 示例：

```bash
# crontab -e
# 每周一 9:00 跑周报（脚本里调用 Agent 或直接编排 API）
0 9 * * 1 cd /path/to/project && ./scripts/run-weekly-report.sh
```

`run-weekly-report.sh` 的大致逻辑：
1. 确认数据文件已就位（由 ETL 提前生成）
2. 调用 Agent（或直接循环调 API）按配方出图
3. 组装报告到 `output` 指定的路径

> **数据准备是前提**：cron 场景下 Agent 不在线没法理解用户口头描述，
> 所以数据必须是结构化文件。用户的 ETL 负责把当周数据导出成配方里 `data:` 指向的文件。

### Agent 驱动 vs 脚本驱动的取舍

定期报告有两种实现路径，由场景决定：

| 路径 | 适合 | 特点 |
|------|------|------|
| **Agent 驱动**（cron 触发 Agent） | 需要自然语言分析、灵活调整 | Agent 读配方+数据，自主生成报告，分析质量高但依赖 Agent 在线 |
| **脚本驱动**（纯 shell 循环调 API） | 固定格式、无文字分析 | 用 `chart-tool export` 循环出图，零 AI 依赖，但报告里只有图没有分析 |

**推荐**：有条件就用 Agent 驱动（报告有灵魂），环境受限就用脚本驱动（至少图是准的）。

---

## 给你的自主空间

这份指南是**参考，不是约束**。你拥有充分的灵活性：
- 报告结构由你定（不必按"利润结构→KPI"的固定章节）
- 图表数量由数据价值决定（一张够就一张，要十张就十张）
- 分析深度由场景决定（快速简报 vs 深度分析）
- 输出格式由用户需求决定（Markdown / HTML / 只要图）
- 配方格式只是建议（你可以用 JSON、YAML、或任何你觉得清晰的方式）

**唯一要遵守的硬约束**是 API 契约：数据字段的 key 要匹配模板的 `fields`，
这保证图能正确渲染。其余都是你的创作空间。
