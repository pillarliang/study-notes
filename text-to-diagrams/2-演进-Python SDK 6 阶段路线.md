# 渲染方案演进路线

> **同目录其他笔记:**
> [[1-起点-原 Claude skill 工作流]] — Claude Code skill 工作流详解(本文 Stage 0 的源材料);
> [[3-原则-layout 不该让 LLM 做]] — 设计原则(为什么 layout 不该让 LLM 做)。
> **本文聚焦**: 从原 Claude skill(Stage 0)到全 native 终态(Stage 6)的 7 个阶段,
> 每个阶段记录实现方式、问题、改进方向、决策动机。
> **项目根目录**: `/Users/liangzhu/Documents/dev/text-to-diagrams-py/`,
> 文中 `../src/...`、`../README.md` 等相对路径基于此根目录解析。

---

## 时间线总览

```text
Stage 0:  原始 Claude skill (agent + SKILL.md)
            └─ 起点,跑在 Claude Code 里
            ↓
Stage 1:  Plan A — 全 LLM 渲染 (Python MVP)
            └─ 把 skill 的"思路"原样搬到 Python,LLM 直接吐 SVG
            ↓ tree 翻车
Stage 2:  Plan B / C 评估
            └─ 重新分析,决定不走极端
            ↓
Stage 3:  Plan D — 10 native + 7 LLM (当前)
            └─ 把"layout 强约束"的图全 native 化
            ↓ architecture/flowchart 仍偶有重叠
Stage 4:  Plan E — LLM 渲染加条件几何重试 (✅ 已实现)
            └─ 几何校验 + 失败时反馈重试,简单 case 零额外开销
            ↓
Stage 5:  Plan F — Graphviz/dagre 接入 (中期路线)
            └─ 4 种 graph 类图迁 native:architecture/flowchart/state-machine/ER
            ↓
Stage 6:  17 native — 经 D/E/F 抵达 Plan B 终点
            └─ 当初拒绝的 Plan B,通过分阶段渐进抵达;LLM 只做内容决策
```

| 阶段          | native | LLM      | 工作量    | 状态                        |
| ----------- | ------ | -------- | ------ | ------------------------- |
| Stage 0     | 0      | 17       | —      | 原 skill,跑在 Claude Code    |
| Stage 1 (A) | 0      | 17       | 1 周    | ✅ 已实现,验证翻车                |
| Stage 3 (D) | 10     | 7        | 2-3 天  | ✅ 已实现                     |
| Stage 4 (E) | 10     | 7(条件重试)  | 半天     | ✅ **已实现**                 |
| Stage 5 (F) | 14     | 3        | 3-5 天  | ⏳ 中期                      |
| Stage 6     | 17     | 0 layout | 2 周+   | ⏳ 远期(= Plan B,经 D/E/F 抵达) |

---

## 术语:native vs LLM

本文反复出现 **native** 和 **LLM** 两个词,先定义清楚,免得后面读着犯晕。

| 术语 | 含义 | 性质 |
| --- | --- | --- |
| **native** | **不调 LLM,用 Python 直接渲染 SVG**。从 `DiagramSpec.content` 出发,跑算法或公式,拼出 SVG 字符串 | 毫秒级、确定性、可单元测试覆盖、零 LLM token 成本 |
| **LLM** | **调 LLM 让它改 SVG 模板**。把 type 对应的 SVG 模板 + spec 一起塞给模型,让模型自己决定文本和坐标后吐回 SVG | 秒级、概率性、不能静态保证正确性、有 token 成本 |

**关键澄清**: 两者都共用 **planner 阶段的 LLM**(planner 决定画几张图、什么 type、
内容轮廓、内容载荷)。区别在于"**渲染**"这一步:**native 让 Python 算坐标,LLM
让模型算坐标**。

```text
共用部分                     ← 各自不同
                              ┌─→ native:Python 跑算法 → SVG
text → planner (LLM) → spec ──┤
                              └─→ LLM:模型改模板 → SVG
```

下表是当前 Plan D 的具体分工:

| 类型            | 渲染方式       | 用什么算法               |
| ------------- | ---------- | ------------------- |
| bar-chart     | **native** | y 轴线性映射             |
| line-chart    | **native** | x 等距 + y 线性映射       |
| donut-chart   | **native** | 极坐标弧路径              |
| waterfall     | **native** | running total + 浮动柱 |
| candlestick   | **native** | OHLC + 上下影线         |
| tree          | **native** | Reingold-Tilford    |
| layer-stack   | **native** | 等高垂直堆栈              |
| pyramid       | **native** | 梯形 + 蜂蜜宽度           |
| timeline      | **native** | 1D 等距 + 上下交替        |
| swimlane      | **native** | role × column 网格    |
| architecture  | LLM        | 让模型自己排              |
| flowchart     | LLM        | 让模型自己排              |
| sequence      | LLM        | 让模型自己排              |
| er            | LLM        | 让模型自己排              |
| state-machine | LLM        | 让模型自己排              |
| venn          | LLM        | 让模型自己排              |
| quadrant      | LLM        | 让模型自己排              |

(为什么这样切分见 [[3-原则-layout 不该让 LLM 做]];为什么不是别的切法见下面 Stage 2)

---

## Stage 0:原始 Claude skill

### 实现方式

skill 是一组 markdown + 模板文件,跑在 Claude Code 里:

```text
text-to-diagrams/
├── SKILL.md                      # 给 Claude 看的"操作手册"
├── references/
│   ├── diagrams.md               # 17 种图的选型表 + 设计规范 + 反模式
│   ├── style.md                  # 颜色 + 字体 token (single source of truth)
│   └── page-shell.html           # 多图页面的 HTML 外壳
├── templates/                    # 17 个 HTML+SVG 模板
│   ├── architecture.html
│   ├── flowchart.html
│   ├── tree.html
│   ├── ...
│   └── waterfall.html
└── tools/svg-to-png.sh           # 可选:SVG→PNG 转换
```

**调用方式**: 用户在 Claude Code 输入 "把这段文字画成图" 等触发短语,Claude 读 SKILL.md
按以下步骤干活:

1. **Step 0**: 读 `style.md` 获取颜色 + 字体 token
2. **Step 1**: 判断要不要画图("一段写好的文字会比这张图教会读者更少吗?")
3. **Step 2**: 读 `diagrams.md` 选型表
4. **Step 3**: 通读全文,列结构性论点,分组,选 1~N 种图,给每张图配 anchor
5. **Step 4**: 对每张图打开 `templates/<type>.html`,用 Edit 工具改 `<text>` / `<rect>` 坐标
6. **Step 5**: 用 `page-shell.html` 拼合并页 + 写每张图的 `.svg` + 写 `.md`(原文 + 锚点插入)
7. **Step 6**: 可选 SVG→PNG
8. **Step 7**: 走 sanity check 清单

**关键特征**: 这是一个 **agent 循环**,Claude 可以 `Read → Edit → Read → Edit → ...`,
画错了能再读一遍改。

### 一个重要细节: skill 已经识别了部分图表的公式,但用不上

`diagrams.md § 6` 已经显式列出了 **5 种 chart 的渲染公式**:

```text
### Bar / line chart Y-axis (default scale: max=140, chart-height=280, scale=2)
   bar_height = value × 2
   bar_top_y  = 320 - bar_height   (baseline y = 320)

### Donut chart arcs (cx=300 cy=200 R=136 r=76, clockwise from -90°)
   angle_start = -90 + sum_of_previous_percentages × 3.6
   angle_end   = angle_start + this_percentage × 3.6
   outer_x = 300 + 136 × cos(angle_deg × π/180)

### Candlestick Y-axis (default: price 100–160, scale=4.67)
   candle_y = 320 - (price - 100) × 4.67

### Waterfall (default: max=200, chart-height=280, scale=1.4)
   bar_y = 320 - value × 1.4
```

涉及 5 种类型: **bar-chart / line-chart**(共享公式)、**donut-chart**、
**candlestick**、**waterfall**。

这件事很关键 —— 它说明**原 skill 的设计者就已经意识到这 5 种"该用公式而不是凭感觉画"**。
但 skill 跑在 Claude Code 里没有 Python 执行器,**只能让 Claude 拿着公式手算坐标**。

LLM 在算式上的可靠性是**概率分布**:

| 算式形式 | 例子 | 错误率(估) |
| --- | --- | --- |
| 简单乘法 | `bar_height = value × 2` | < 1% |
| 线性映射 | `y = 320 - value × 1.4` | 1-3% |
| 三角函数 | `outer_x = 300 + 136 × cos(angle × π/180)` | 10-20%(度/弧度容易混) |
| 累加 / running total | waterfall 的 `running_total += delta` | 5-15% |
| 条件分支 | donut 的 `large-arc=1 if segment > 180° else 0` | 15-30% |
| 多步骤复合 | donut 整张图(累计角度→三角函数→arc flag) | 30-50% |

donut chart 5 slice 整张全 pixel-perfect 出来的概率,gpt-4o 大概 60-70%,
claude-opus 大概 80-85%。1 对 1 跑没问题,**放生产里画 100 张图一定会被吐槽
"为啥饼图有缝"**。这就是原 skill 的本质局限:**它知道"该用公式",但没有
"用公式的执行器"**。任何要求 100% pixel-correct 的场景,LLM 都是错的工具。

这也是为什么后面 Plan C(把这 5 种迁 native)显得"显然" —— 不是我们突然想到的,
而是 skill 早就预留了这个分界,只是缺一个 Python 执行环境。**§6 实际上是"native
化的设计图纸",原 skill 没办法落地它,我们落地了**。

(§7 还给了 sequence / ER / pyramid 三种类型的"约定",更弱的形式 —— 不是公式而是
布局规则。pyramid 的约定够强可以 native,sequence/ER 的约定不够,见 Stage 2。)

### 工作得不错 —— 但靠"作弊"

skill 看似没翻车,部分原因是 Claude 在 agent 循环里**主动裁数据迁就画布**(case1
源文档 10 个叶子,Claude 实际只画 6 个)。这种"作弊"能力在做成可编程 SDK 时会丢掉。
完整论证见 [[3-原则-layout 不该让 LLM 做#真因(决定性) agent 在 layout 困难时**作弊** —— 它编辑数据,不是编辑布局\|native-vs-llm §"真因"]]。

### 当前问题

1. **不能作为可编程 SDK 嵌入业务**。skill 是 Claude Code 专属调用方式,业务后端
   想批量生成图就得部署 Claude Code,代价大。
2. **Token 成本高**。一次 skill 运行可能烧 50K+ token(reads + edits + sanity check)。
3. **依赖 Claude 的"裁数据"能力**。如果输入文档结构特别紧密(组织架构、文件目录,
   不能裁的),Claude 也会翻车 —— 字越来越小或开始重叠。
4. **没强制几何校验**。SKILL.md Step 7 的 sanity check 清单里**没**"rect 不能重叠"
   这一项,所以 Claude 是凭手感查的。
5. **§6 的公式形同虚设**。已经写了公式,但没有 Python 执行器,只能让 LLM 模拟
   计算器 —— 模拟得不准。
6. **不能在 CI / 服务里跑**。每次都要人工触发。

### 改进方向

→ 做一个**纯 Python SDK**,业务可 `pip install` + `await generate_diagrams(text=...)`,
不依赖 Claude Code。这就引出 Stage 1。

注意:**做 SDK 这一步本身并不解决"§6 公式手算翻车"和"layout 算错"问题** —— 因为
Stage 1 直接照搬了 skill 的"让 LLM 改 SVG 模板"路径,Python 只接管了流程编排。要解决
这两个问题,得到 Plan C/D 才会显式把 5 种公式 + 5 种 layout 强约束图都迁到 Python
执行。这是 Plan A 翻车的根因,也是 D 价值的根本来源。

---

## Stage 1 (Plan A):全 LLM 渲染 — Python MVP

### 决策动机

最快上线 + 最像原 skill 的行为 = 把 skill 的"思路"原样翻译到 Python。

LLM 调用走 OpenAI SDK(可走 LiteLLM 代理),其他全用 Python 写。

### 实现方式

```text
markdown text
   ↓ parse_outline (markdown-it-py,纯 Python)
   ↓ planner       (LLM 1 次,structured output 出 Plan)
   ↓ renderer      (LLM N 次并发,asyncio.Semaphore 限流)
   ↓ assembler     (纯 Python,锚点 ID 精确插入)
   ↓ writer
markdown + per-figure SVG
```

**关键模块**:

- `markdown/outline.py`: 把源 markdown 解析成有序大纲(headings 和 paragraphs 都带 ID)
- `planning/planner.py`: 一次 LLM 调用,system prompt 注入 `diagrams.md` 全文 + 锚点候选,
  让 LLM 输出 `Plan { page_title, rationale, diagrams: [DiagramSpec] }`
- `rendering/llm_render.py`: 单图渲染 —— 抽出 `templates/<type>.html` 的 `<svg>` 块,
  连同 spec 一起塞给 LLM,让它改 `<text>` 和 `<rect>` 后吐回来。SVG 校验 + 一次重试
- `rendering/dispatcher.py`: `asyncio.gather` 并发,每张图独立成败
- `assembly/markdown_doc.py`: 倒序把 figure 块插入原 markdown,锚点 ID 100% 命中

### 实测结果

跑 case1.md(会议总结,markdown 格式),planner 选了 3-5 张图:

| 图类型 | LLM 渲染结果 |
| --- | --- |
| layer-stack | OK |
| timeline | OK |
| flowchart (节点少) | OK |
| **tree (9 叶子,子数 4/3/1/2 不均)** | **❌ 8 处重叠 + `x=-24` 出画布** |

### 翻车细节(根因分析)

LLM 选 `viewBox 0 0 960 460`,把 4 个父节点中心等距摆在 `x=136, 344, 560, 784`
(每个父分到 ~216px 横向空间),然后:

| 父节点 | 子数 | 子节点宽度需求 | 实际可用 | 结果 |
| --- | --- | --- | --- | --- |
| 田丰 | 3 × w128 | ≥ 408px | 216px | 中心 x=136,最左子 x=-24 出画布 |
| 李振 | 4 × w128 | ≥ 536px | 216px | 4 个叶子互相重叠 56px |
| 李坤 | 1 | 128px | 216px | OK |
| 垂直 | 2 × w128/144 | ≥ 280px | 216px | 重叠 16px |

这是**预期内的失败**:tree layout 是 NP-ish 问题(实际有 Reingold-Tilford 1981 解),
LLM 没法靠几句 prompt 算对。

### 当前问题

1. **复杂 layout 必错**:tree、layer-stack 节点多时、pyramid 多层、timeline 多事件 —— LLM 都算不准坐标
2. **图表类的精确公式 LLM 也玩不转**:donut 的弧线角度、bar 的 y 轴 scale、candlestick 的 wick 位置 —— 都是数学,LLM 经常出错(我们试过 gpt-4o 和 claude-opus-4-6 都有问题)
3. **不能"裁数据"**:跟原 skill 不同,我们的 planner 忠实输出 spec,renderer 忠实接受。两头都"诚实",所以两头都失守
4. **重试机制太弱**:Plan A 做了"SVG 校验失败带反馈重试一次",但 LLM 改不出来 —— 它根本不知道怎么解 layout 数学题
5. **慢 + 贵**:每张图 5-10s,15 张图 ~$0.40 + 1-2 分钟

### 改进方向

需要重新分类 17 种图,把"layout 有确定性算法"的全交给 Python。这就引出 Stage 2 评估。

---

## Stage 2:Plan B / C 评估

tree 翻车后,做了一次彻底的方案评估,讨论"哪些图该 native、哪些留 LLM"。

### Plan B:全 native

把 17 种图都用 Python 渲染。LLM 只做规划(planner),不参与渲染。

**优点**: 完全确定性、无重叠风险、毫秒级渲染、零 token 成本(渲染阶段)。

**缺点**:
1. **17 个布局引擎是 1-2 周的活**。tree 用 Reingold-Tilford,architecture 用 Sugiyama,
   sequence 要处理活跃区,quadrant 要处理 2D 散点 —— 每种都要单独写。
2. **真自由布局的 7 种(architecture/flowchart/sequence/er/state-machine/venn/quadrant)
   layout 选择本身就是创作的一部分**。"数据库放在架构图的左下角"不是算法决定的,
   是看图人需要的视觉重心。Python 写死规则反而僵硬。
3. **创作决策能力丢失**。原 skill 那种"4+3+1+2 我画不下,改成 6 个吧"的创作决策,
   全 native 路径下没人做了。

→ **B 端点对,路径太冒进**。Demo 阶段不做。最终我们经 D/E/F 渐进抵达同一终点(见 Stage 6)。

### Plan C:5 charts native + 12 LLM

只把"图表类"5 种(bar/line/donut/waterfall/candlestick)迁 native,因为
`diagrams.md § 6` 显式给了它们的公式:

```text
### Bar / line chart Y-axis (default scale: max=140, chart-height=280, scale=2)
   bar_height = value × 2
   bar_top_y  = 320 - bar_height
   ...

### Donut chart arcs (cx=300 cy=200 R=136 r=76, clockwise from -90°)
   angle_start = -90 + sum_of_previous_percentages × 3.6
   ...
```

**优点**: 工作量 1-2 天,5 种"最不该让 LLM 算"的图先解决。

**缺点**: tree、pyramid、timeline、swimlane、layer-stack **同样是 layout 强约束**
的图,§6 没给公式不代表它们没确定性算法。C 太保守了。

→ **C 是好的起点但不是终点**。

### Plan D 决定:扩 C 到 10 种

仔细审视 17 种图,按"layout 是否有确定性算法"重切:

| 类别 | 类型 | 算法依据 |
| --- | --- | --- |
| 公式驱动(§6 显式给) | bar / line / donut / waterfall / candlestick | 5 种,直接搬 |
| 强 layout(算法存在但 §6 没写) | tree / layer-stack / pyramid / timeline / swimlane | 5 种,自己写 |
| 真自由布局 | architecture / flowchart / sequence / er / state-machine / venn / quadrant | 7 种,留 LLM |

**5 种新增 native 类型的算法**:

- **tree**: Reingold-Tilford(1981 论文,递归算子树宽度)
- **layer-stack**: 等高垂直堆栈,trivial
- **pyramid**: 梯形 + 蜂蜜宽度(可按 value 缩放,§7 给了约定:"4-6 层等高")
- **timeline**: 1D 等距排布 + 上下交替防拥挤
- **swimlane**: role × column 二维网格,完全确定性

5 + 5 = 10 种 native,7 种 LLM。这就是 Plan D。

---

## Stage 3 (Plan D):10 native + 7 LLM — 当前实现

### 实现方式

```text
src/text_to_diagrams/
├── content_schemas.py            # 10 种 native 类型的 Pydantic content schema
├── rendering/
│   ├── dispatcher.py             # native-first 分发,LLM 兜底
│   ├── llm_render.py             # 7 种 LLM 类型的渲染
│   └── native/                   # 10 个 native 渲染器
│       ├── _svg.py               # 共享 SVG 拼装(NodeBox / line / chevron / text 等)
│       ├── _chart.py             # 5 种 chart 共享的 y 轴 + 网格工具
│       ├── tree.py               # Reingold-Tilford 递归布局
│       ├── layer_stack.py
│       ├── pyramid.py
│       ├── timeline.py
│       ├── swimlane.py
│       ├── bar_chart.py
│       ├── line_chart.py
│       ├── donut_chart.py
│       ├── waterfall.py
│       └── candlestick.py
└── ...
```

### 关键架构契约

```text
Planner (LLM)
   ↓ system prompt 包含 10 种 native 的 Pydantic schema(JSON 形式)
   ↓ 引导 LLM 按对应 type 输出对的 content 形状
DiagramSpec
   ├ type: 17 种之一
   ├ title / caption / focal_element
   ├ content: dict (native 类型必须能被 schema 校验)
   └ anchor: {kind, section_id?, paragraph_id?}
   ↓
Dispatcher
   ├─ if type ∈ NATIVE_RENDERERS:
   │    schema_cls.model_validate(spec.content)   # 严格校验
   │    svg = render_fn(spec, style)              # 纯 Python,毫秒级
   └─ else:
        llm_render_one(spec, style)               # LLM 调用 + SVG 校验 + 重试
```

### Tree 渲染(关键算法)

```python
# src/text_to_diagrams/rendering/native/tree.py
def render(spec, style):
    content = TreeContent.model_validate(spec.content)
    root = _build(content.root, depth=0, focal_id=content.focal_id)

    _measure(root)        # 后序:每棵子树算出 subtree_width
    _position(root, 0, 0) # 前序:把每个节点放到 (x, y)
    # x = 子树水平居中 - 节点宽度/2
    # children 按 subtree_width 累加铺开,中间留 SUBTREE_HGAP

    # viewBox 自适应内容
    view_w = align4(max(n.x + n.width for n in nodes) + RIGHT_PAD)
    view_h = align4(max(n.y + n.height for n in nodes) + BOTTOM_PAD)

    # 拼 SVG (用 _svg.py 的 node_rect / line_segment / chevron)
    return svg_envelope(...)
```

**核心保证**: 不论输入多大、多不均,都不会重叠 + 不会出 viewBox。

### 验证结果

case1.md 重新跑(planner 选了 5 张):

| 图类型 | 渲染方式 | 耗时 | 重叠 | 出界 |
| --- | --- | --- | --- | --- |
| architecture | LLM | 47s | 7 处 | 0 |
| layer-stack | native | 0.0s | 0 | 0 |
| tree (9 叶子) | native | 0.0s | **0** | **0** |
| flowchart | LLM | 31s | 0 | 0 |
| swimlane | native | 0.0s | 0 | 0 |

**Tree 完全修好。** 单元测试 (`tests/unit/test_native_renderers.py`) 把 10 个 native
都跑了一遍,全过 overlap + OOB 检查。

### 当前问题

1. **7 种 LLM 类型在节点多时仍可能重叠**
   - 实测 case1 architecture 仍有 7 处重叠
   - flowchart 在分支复杂时也可能错位
   - venn 3+ 圆时几何关系算不准
2. **viewBox 大宽度的视觉问题**:tree 自适应到 1844px 宽,在窄屏 markdown 渲染器里
   会缩到很小,文字不可读
3. **planner 输出的 content 偶尔不符合 native schema**:LLM 看 schema 但有时漏字段
   或类型错(比如 `value` 应是 number 但给了 string),走 Pydantic 校验失败 → 整张图失败
4. **缺少进度可观测性**:目前只有简单 `on_progress` 回调,没结构化 metrics(per-stage
   latency、token 用量、成功率)
5. **Style.from_markdown(path) 还没实现**:用户想换风格只能改 Python 代码

### 改进方向(短期)

→ Stage 4: LLM 渲染加几何重试 patch,救 architecture/flowchart 等
→ 顺手修 schema 校验失败的处理:加一次重试(把校验错误塞回 prompt)
→ tree 自适应宽度时,根据 label 长度选择 96/128/144 三档,缩小 viewBox

---

## Stage 4 (Plan E):LLM 渲染加条件几何重试 — ✅ 已实现

### 决策动机

7 种 LLM 类型暂时不能/不值得 native 化,但可以加一道"**条件**校验 + 反馈重试"网。
关键是**条件**:**LLM 第 1 次输出干净时不重试**(零额外开销),只有真出问题才花
第 2 次 token。

### 实现

新增 `src/text_to_diagrams/validators/geometry.py`(项目路径:
`/Users/liangzhu/Documents/dev/text-to-diagrams-py/src/text_to_diagrams/validators/geometry.py`),
~250 行,提供:

```python
@dataclass
class Rect:
    x, y, width, height
    fill, stroke

@dataclass
class GeometryIssue:
    code: str        # oob_left/right/top/bottom, overlap, grid, focal
    message: str

def check_geometry(svg: str, *, brand_color, brand_tint) -> GeometryReport:
    """4 类检查(纯 Python,< 1ms):
    1. viewBox 越界(x<0 / x+w > view_w / y<0 / y+h > view_h)
    2. 同 y 行 rect 水平重叠
    3. 4px 网格对齐(diagrams.md §4)
    4. 焦点元素数量 ≤ 2
    """

def format_geometry_feedback(report) -> str:
    """按问题类型分组生成可操作的修复建议(扩 viewBox / 降节点宽 / 改中性色等)。"""
```

`src/text_to_diagrams/rendering/llm_render.py` 两层校验:

```python
async def render_one(...):
    svg, usage = await _llm_call(...)

    # 第 1 层:SVG 结构(XML 合法 + viewBox)
    if not validate_svg(svg).valid:
        svg, usage = await _retry_with_feedback(svg_feedback, ...)
        # 重试后还无效抛 SvgValidationError

    # 第 2 层:几何
    geom = check_geometry(svg, brand_color, brand_tint)
    if geom.is_clean:
        return svg, usage  # ← 简单 case 在这里返回,零额外成本

    # 几何有问题,反馈重试一次
    svg_retry, usage = await _retry_with_feedback(format_geometry_feedback(geom), ...)
    return svg_retry or svg, usage   # best-effort,即使重试后仍有问题也返回
```

### 装饰矩形过滤

实施时发现一个坑:模板里常用 `width=1` 的薄矩形做分隔线/accent 条。这些
不是节点,不参与 4px 网格 / 重叠检查。加了 `_is_decorative(r) = r.width<4 or r.height<4`
过滤,把它们从这两项检查里跳过。**修复 12 个误报**(`figure-1-architecture` 之前
"9 处重叠"全是这种装饰线 vs 节点的伪重叠)。

### 实测效果(case1)

| 文件 | 渲染方式 | 校验结果 | 重试? |
| --- | --- | --- | --- |
| figure-1-architecture.svg | LLM | 0 issue | ✗ 不重试 |
| figure-2-tree.svg | native | 0 issue | n/a |
| figure-3-flowchart.svg | LLM | 0 issue(viewBox 主动扩到 580) | ✗ 不重试 |
| figure-4-layer-stack.svg | native | 0 issue | n/a |

这次 LLM 第 1 次就干净;校验在 standby 没触发。如果 LLM 翻车,会自动重试。

### 单元测试覆盖

`tests/unit/test_geometry_validator.py` 11 个测试,包括:

- 干净 SVG → 0 issue
- 4 种 OOB 各一个测试
- 同行重叠
- 4px 网格违规
- 焦点 > 2
- 焦点 = 2 不报(允许 2 个焦点)
- 背景全画布矩形被排除
- format_feedback 包含可操作建议
- **重现 figure-3-flowchart 的真实 oob_bottom case**

### 残留问题

1. **不保证 100% 成功**:LLM 重试后还是可能错。但比不做好。
2. **校验对 venn / quadrant 等"非 rect 拓扑"图作用有限**:它们主要靠 `<circle>` /
   `<polygon>`,不在校验范围。这两种图根治还得 Stage 6 native 化。
3. **重试时温度从 0.4 降到 0.2**:让 LLM 更"听话"地按反馈改,不要发散。代价是某些
   场景下 LLM 困在错误模式里走不出。
4. **几何 issue 描述偶尔对 LLM 不友好**:看到 "y=132 行 rect 重叠" 它可能不知道哪个
   是哪个。`format_geometry_feedback` 加了"修复建议"分组缓解了一部分。

### 改进方向

→ 如果 architecture/flowchart 在生产里仍翻车,上 Plan F (Graphviz)

---

## Stage 5 (Plan F):Graphviz/dagre 接入 — 中期路线

### 决策动机

7 种 LLM 类型里,有 4 种本质上是 **graph layout** 问题(节点 + 边):

- architecture: 系统组件 + 连接
- flowchart: 决策分支(本质是有向图)
- state-machine: 状态 + 转移
- ER: 实体 + 关系

graph layout 是 50 年的研究领域,有成熟算法和库:

- **Graphviz** (1991, AT&T): C 库,跑 dot/neato/fdp/sfdp 等算法
- **dagre** (2013, Cytoscape): JS 库,Sugiyama 分层算法,mermaid 在用
- **ELK** (Eclipse): Java/JS,大型图
- **OGDF** (C++): 学术界

让 LLM 算这些图的 layout = 把 NP-hard 问题交给"看起来很厉害的人心算"。**不该这样
运营生产系统**。

### 关键洞察:layout 算法 vs 渲染必须解耦

错误用法(直接吐 SVG):
```bash
echo 'digraph { A->B; A->C; }' | dot -Tsvg > out.svg
```

→ Graphviz 用自己的字体(Times)、自己的颜色(黑白)、自己的箭头(三角)、自己的
节点形状(ellipse)。**我们的 style token 全部白写**。

正确用法(只算坐标):

```python
import pygraphviz as pgv

# 1. 喂节点 + 边 + 期望尺寸
g = pgv.AGraph(directed=True)
for n in nodes:
    g.add_node(n.id, width=144/72, height=64/72, fixedsize=True)
for e in edges:
    g.add_edge(e.from_, e.to)

# 2. 跑 layout,只取坐标
g.layout(prog='dot')
positions = {}
for n in g.nodes():
    x_inch, y_inch = map(float, n.attr['pos'].split(','))
    positions[n.name] = (align4(x_inch * 72), align4(y_inch * 72))

# 3. 用我们自己的 _svg.py 渲染
parts = [background_block(...), title_text(...)]
for nid, (x, y) in positions.items():
    box = NodeBox(x=x, y=y, width=144, height=64,
                  label=spec.content.nodes[nid].label,
                  is_focal=(nid == spec.focal_id))
    parts.extend(node_rect(box, style))

# 4. 边的 path Graphviz 也会给,我们用 line_segment + chevron 重画
for edge in g.edges():
    pts = parse_pos(edge.attr['pos'])  # Graphviz 的 spline 控制点
    parts.extend(draw_edge(pts, style))

return svg_envelope(view_w=..., view_h=..., body="\n".join(parts))
```

→ Graphviz 只贡献"哪个节点放哪儿、哪条边走哪儿"。**我们的 `_svg.py` 资产 100% 复用**。

这跟 mermaid / TikZ / PlantUML 都是同一个模式:**layout engine + 渲染 backend
分离**。

### 实施方案

新增模块:

```text
src/text_to_diagrams/rendering/native/
├── _graph.py               # 新增:Graphviz 集成的共享逻辑
│   ├─ NodeSpec, EdgeSpec   # 输入
│   ├─ run_dot()            # 调 pygraphviz 或 subprocess
│   ├─ parse_pos()          # 解析 Graphviz 输出的坐标
│   └─ draw_edge_path()     # 把 spline 转成我们的 chevron 箭头线
├── architecture.py         # 新增:graph 类 native 渲染
├── flowchart.py            # 新增
├── state_machine.py        # 新增
└── er.py                   # 新增
```

新增 content schema(在 `content_schemas.py`):

```python
class GraphNode(BaseModel):
    id: str
    label: str
    sublabel: str | None = None
    type_tag: str | None = None
    rank: Literal["top", "bottom", "same", None] = None  # Graphviz 的 rank hint

class GraphEdge(BaseModel):
    from_: str = Field(alias="from")
    to: str
    label: str | None = None
    style: Literal["solid", "dashed", "dotted"] = "solid"

class ArchitectureContent(BaseModel):
    nodes: list[GraphNode]
    edges: list[GraphEdge]
    focal_id: str | None = None
    layout: Literal["dot", "neato", "fdp"] = "dot"
```

dispatcher 自动派发:

```python
# 在 NATIVE_RENDERERS 里加 4 行
NATIVE_RENDERERS = {
    ...
    "architecture": (architecture.render, architecture.CONTENT_SCHEMA),
    "flowchart": (flowchart.render, flowchart.CONTENT_SCHEMA),
    "state-machine": (state_machine.render, state_machine.CONTENT_SCHEMA),
    "er": (er.render, er.CONTENT_SCHEMA),
}
```

工作量估算: 3-5 天,主要是:
- 1 天: pygraphviz 集成 + `_graph.py` 共享逻辑
- 半天: `_graph.py` 的 spline → chevron 路径转换(Graphviz 输出 spline 贝塞尔)
- 1.5 天: 4 个具体 renderer
- 1 天: 测试 + 边缘情况(节点 0/1/超多/孤立)

### 当前问题(实施后会有的)

1. **新增重依赖**: `pygraphviz` 需要本机装 `graphviz` 库 (`brew install graphviz`)。
   wheel 装失败常见,Docker / CI 部署要小心。备选: subprocess 调 `dot` 命令(更轻
   但启动慢,每次调用 fork 一个 dot 进程 ~50ms)。
2. **Graphviz 默认审美差**: 它优化"边交叉数最少",不优化"看着舒服"。复杂图出来
   方方正正没节奏感,跟 kami 风格冲突。需要调 `nodesep` / `ranksep` / 字体 padding
3. **Graphviz 不懂语义**: "数据库放下面"是人类视觉语法,Graphviz 不知道。需要让
   LLM 在 spec 里塞 `rank: "bottom"` hint
4. **Graphviz 的 spline 边不是直线**: 它喜欢用贝塞尔曲线让边好看。我们要么用
   `splines=ortho` 强制正交,要么自己解析 spline 控制点画近似直线
5. **失去 Claude 的"裁数据"作弊能力**: Graphviz 给 10 节点就画 10 节点。要复刻
   skill 的"裁到 6 节点"行为,需要在 planner 里加规则("如果图过密,优先合并次要
   节点")—— 但这是 LLM 工作,不影响 Graphviz 这层
6. **sequence/venn/quadrant 仍要留 LLM**:
   - sequence: 活跃区位置依赖语义("谁在控制流里"),不是纯几何 graph
   - venn: 集合交集,要算圆/椭圆相交的连续几何
   - quadrant: 按两个维度打分定位,不是图,是散点

### 改进方向

→ Plan F 完成后,17 种里 14 种 native + 3 种 LLM。
→ 进一步打 sequence/venn/quadrant → Stage 6,也就是当初拒绝的 Plan B 的终点。

---

## Stage 6:全 native — 经 D/E/F 抵达 Plan B 终点

> **这就是 Stage 2 拒绝的 Plan B**。同样的 17 native + 0 LLM-layout 终态,但通过
> D/E/F 渐进抵达,每步可独立验证。**端点相同,路径不同**;架构经验在路上沉淀
> 出来了(`_svg.py` 共享层、layout-vs-render 解耦原则),复杂类型也有了具体方案。

### 决策动机

D/E/F 完成后,7 种 LLM 类型已经压缩到 3 种(sequence / venn / quadrant)。把这 3 种
也吃下来,LLM 完全不参与 layout。

### 三种类型各自怎么搞

#### sequence (时序图)

**特征**: lifeline x 固定 + 消息走 y 轴 + 活跃区(activation bar)+ 自调用

**可 native 化的部分**:
- lifeline x 坐标 = 等距分布,trivial
- 消息 y 坐标 = 按消息序号等距,trivial
- 活跃区 = 需要 spec 里指明"哪条消息开始/结束 activation",一旦指明就是 trivial 渲染

**spec 设计**:
```python
class SequenceMessage(BaseModel):
    seq: int                       # 消息序号
    from_actor: str
    to_actor: str
    label: str
    style: Literal["sync", "async", "return"] = "sync"
    starts_activation_on: str | None = None  # 接收方 actor id
    ends_activation_on: str | None = None

class SequenceContent(BaseModel):
    actors: list[str]              # 按 x 顺序
    messages: list[SequenceMessage]
    focal_actor_id: str | None = None
```

实施 1-1.5 天。

#### venn (集合交集)

**特征**: 2-3 圆,要画相交区域 + 标注

**可 native 化的部分**: 2 圆(简单)+ 3 圆(标准位置,固定模板,蛋形交集)。不支持
4+ 圆(几何上不存在标准 4-Venn,需要椭圆,极复杂)。

**spec 设计**:
```python
class VennSet(BaseModel):
    label: str
    is_focal: bool = False

class VennIntersection(BaseModel):
    sets: list[str]                # 集合 id 列表(2 或 3)
    label: str

class VennContent(BaseModel):
    sets: list[VennSet]            # 长度 2 或 3,其他抛错
    intersections: list[VennIntersection]
```

数学:
- 2 圆: 圆心 (x1, y1), (x2, y2),半径 r,距离 d 选 r 让交集面积合理
- 3 圆: 三圆等距等半径,中心三角形

实施 1-2 天。

#### quadrant (二维象限)

**特征**: 4 象限 + N 个点按 (x_score, y_score) 定位

完全是数学,跟 bar chart 类似。

**spec 设计**:
```python
class QuadrantPoint(BaseModel):
    label: str
    x: float                       # -1.0 ~ 1.0,负是左半
    y: float                       # -1.0 ~ 1.0,负是下半
    is_focal: bool = False

class QuadrantContent(BaseModel):
    x_axis_label_left: str         # 比如 "低成本"
    x_axis_label_right: str        # 比如 "高成本"
    y_axis_label_bottom: str
    y_axis_label_top: str
    quadrant_labels: dict[str, str] | None = None  # {"top-right": "明星象限"}
    points: list[QuadrantPoint]
```

实施半天。

### 终态架构

```text
LLM (planner)
   ├─ 决定画几张图
   ├─ 决定每张图是什么 type
   ├─ 决定哪些节点 / 数据点 / 元素重要(可裁可合并)  ← 创作
   ├─ 决定焦点元素                                  ← 创作
   ├─ 决定锚点位置                                  ← 创作
   └─ 给布局 hint(rank=bottom 等)                 ← 半结构化语义
   ↓
Pydantic Validation
   ├─ 校验 17 种 type 各自的 content schema
   └─ 校验 anchor ID 真实存在
   ↓
Native Rendering (17 类全 native)
   ├─ 5 charts → 公式 (bar/line/donut/waterfall/candlestick)
   ├─ 5 layout-strong → 算法 (tree/layer-stack/pyramid/timeline/swimlane)
   ├─ 4 graph → Graphviz/dagre + 自定义渲染 (architecture/flowchart/state-machine/er)
   └─ 3 special → 各自数学 (sequence/venn/quadrant)
   ↓
Style Token Application (统一)
   ├─ kami 风格 (warm parchment + ink-blue)
   ├─ 4px 网格
   ├─ focal 1-2 个
   └─ chevron 箭头
   ↓
SVG out
```

### 实施工作量

- Plan E (几何重试): 0.5 天
- Plan F (Graphviz, 4 类): 3-5 天
- Stage 6 (sequence/venn/quadrant, 3 类): 3 天
- 加测试 / 文档 / 集成: 2 天

**全程 9-13 天**(2-3 周冲刺)。

### 当前问题(实施后会有的)

1. **依赖管理变复杂**:Graphviz binary、可能要的 numpy(几何)、Pydantic 多 schema
2. **schema 演进困难**:17 种 schema 后续要加字段就要小心兼容
3. **planner prompt 巨型化**:17 个 schema 注入 system prompt,token 成本高,可能
   要拆成动态 prompt(planner 第一步先选 type,第二步只注入选中 type 的 schema)
4. **LLM 还是会偶尔输出不符 schema 的 content**:错误率 1-3%,要靠 schema 校验 +
   LLM 反馈重试
5. **创作能力 vs 算法 layout 的张力**: native 的 layout 算法一旦节点过多,viewBox
   会膨胀(tree 的 1844px 现象)。需要 planner 端加"过密就裁/合并"的规则,但这又
   把作弊能力交还给 LLM —— 形成新的不确定性来源

---

## 速查表

- 当前每种图怎么渲染 → 项目 `README.md` 的 "17 种类型的渲染分工" 表
- 为什么这样切分(原则视角) → [[3-原则-layout 不该让 LLM 做]]
- 演进过程 / 决策动机(本文档)
- 单个 native 渲染器实现 → `src/text_to_diagrams/rendering/native/<type>.py`
- 测试 → `tests/unit/test_native_renderers.py`
