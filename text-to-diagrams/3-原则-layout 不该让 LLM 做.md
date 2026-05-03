# 为什么 layout 不该让 LLM 做

> **同目录其他笔记:**
> [[1-起点-原 Claude skill 工作流]] — Claude Code skill 工作流(本文 §"残酷的真相"节里的 agent 作弊证据来自这里);
> [[2-演进-Python SDK 6 阶段路线]] — 6 个阶段的演进路线(本文是其中 Stage 2 → Stage 3 决策的"原则版"沉淀)。
> **本文聚焦**: 一个具体的翻车案例,一条普遍的原则,一份可落地的 17 类型分工表。

## TL;DR

- LLM 擅长**理解内容**和**做创作决策**,不擅长**二维布局求解**。
- 有确定性算法能解决的问题(画 tree、算坐标、算扇形角度),**不该交给 LLM**;
  让 LLM 算这些事不是省工程师的力气,是把可靠性赌掉了。
- 在本项目里,17 种图按这个原则切成 **10 种 native(Python 渲染)+ 7 种 LLM 渲染**。

---

## 起因:一次具体的翻车

我们用 LLM 渲染一张 tree 图 —— 4 个父节点,子节点数 4/3/1/2,共 9 个叶子。
LLM 选定 viewBox `0 0 960 460`,把 4 个父节点等距摆开,然后试图给每个父节点的
子树分配横向空间。结果:

| 现象 | 数据 |
| --- | --- |
| 叶子之间水平重叠 | 8 处,40~88px 不等 |
| 节点出画布 | 最左叶子 `x=-24`,出左界 24px |
| LLM 重试 | 1 次,仍然翻车 |

根因是数学上的:**李振分支要塞 4 个 width=128 的叶子,需要至少 536px 横向空间,
但被分到 216px**。LLM 没做不等宽分配,硬挤,必然重叠。即使再 prompt
"叶子不能重叠"、"rect 必须在 viewBox 内",LLM 也只能解决简单 case —— 因为这是个
布局求解问题,不是文本生成问题。

> 复杂 tree(子节点数不均、叶子多)的布局,1981 年的 Reingold-Tilford 算法就给了
> 解。把它交给 LLM,等于把"用计算器加 235+187"的事交给"心算很厉害的人"。
> 偶尔他能对,但你不该这样运营系统。

---

## 关键问题:为什么原 Claude skill 没翻这种车?

读到这里很自然会问:同样的 17 种图、同样的模板、同样的输入,Claude Code 里跑
`/text-to-diagrams` skill 的时候,产出看起来是对的。差别在哪儿?是不是 agent
系统给了它什么魔法?

### 假说 1: agent 循环 + 自检 → 错了

直觉上,agent 是 `Read → Edit → Read → Edit → ...` 的循环,可以画完看一眼、
发现不对再调。SKILL.md 里 Step 7 也有 "sanity check" 清单。

但**清单内容是这些**:

```text
- 每张图焦点 1-2 个
- 焦点色不与 caption 矛盾
- 箭头端点贴节点边
- 所有坐标 4px 整除
- 没 box-shadow / gradient / emoji
- 应用了 style.md 的颜色
```

**没有"rect 不能重叠"这一项**。所以"agent 自检"是个工程上没强制的心理预期。

### 假说 2: 模型更强 + 思考预算更大 → 部分正确,但不是决定性

Claude Code 里底层是 Opus + extended thinking,一次 skill 运行可能烧 50K+ tokens
边想边画。我们的 renderer 单次调用 ~5K tokens。

但这只是**让 LLM 算 layout 的成功率从 30% 提到 60%**,本质问题没解决 ——
LLM 算法仍然不是"先 measure 再 position"的递归过程,只是更长的 token 序列里
冒出更可能正确的坐标。换句话说,**算力换不来必胜的算法**。

### 真因(决定性): agent 在 layout 困难时**作弊** —— 它编辑数据,不是编辑布局

我对比了原 skill 给 case1 的实际产出 vs 源文档,惊讶地发现:

| | 源文档结构 | 原 skill 渲染的 tree |
| --- | --- | --- |
| 父节点数 | 4 (Tianfeng/Lizhen/Likun/Vertical) | 4 |
| 子节点数 | 4/3/1/2,共 **10 个** | **6 个** |
| 叶子布局 | 应该按子树分组 | 一行均匀 144px 间距 |

它**没**忠实渲染原 hierarchy。它直接编辑了数据,挑了 6 个最重要的元素,塞进
画布能放下的网格里。这是 Claude 的**创作决策**,不是 layout 算法。

换句话说,Claude 看到"4+3+1+2 我画不下,改成 6 个吧"。它绕开了难题。

我们的 pipeline 干的是相反的事:

- planner 忠实输出 spec → renderer 忠实接受 → LLM 算 layout
- planner 不"裁数据",renderer 又算不准 layout
- **两头都是忠诚的,所以两头都失守**

---

## 残酷的真相

**原 skill 的 case1 看起来好,部分原因是它"作弊"了 —— 把 10 个叶子砍成 6 个**。

如果你给原 skill 一个非画 10 个叶子不可的输入(比如组织架构图、文件目录),它一样
会翻车,只是翻车形式可能是字越来越小、节点开始重叠 —— 或者 Claude 索性拒绝画。

agent 系统的好处是能"作弊得优雅"(挑数据、试错、自我修正)。但本质问题没解决:
**LLM 不该负责二维 layout**。

---

## 升华成原则

### 原则 1: 区分"理解"和"执行"

LLM 该做的事:

- **理解** 输入文档讲了什么
- **决策** 用哪种图、哪些是焦点、哪些可以省
- **生成** 文本(标题、caption、节点 label)

LLM 不该做的事:

- **求解** 布局约束(节点不重叠、对齐 4px 网格、viewBox 自适应)
- **算** 任何有公式的坐标(donut 弧线、bar 柱高、candlestick wick)
- **优化** NP-类问题(graph layout、装箱、调度)

把这两类工作混在一起让 LLM 一锅端,**LLM 第一类做得好的优势,会被第二类的错误
率拖累**,最终用户看到的是"AI 画的图老是错位"。

### 原则 2: 有确定性算法的,优先用算法

如果一个问题:

- 有 50 年的成熟算法(Reingold-Tilford for trees, Sugiyama for layered graphs)
- 输入有限、输出有限,可以测试覆盖
- 失败模式可以静态校验(几何重叠、边界 overflow)

那就**让 Python 跑算法**,不要让 LLM 模拟算法。后者是把可靠性赌掉。

### 原则 3: 把契约画在数据层,而不是产物层

我们 pipeline 现在的契约:

```text
LLM 输出: 结构化 content (Pydantic schema)
   ↓
Python 校验: schema 不符就报错
   ↓
Python 渲染: 用算法算坐标 → 出 SVG
```

而不是:

```text
LLM 输出: SVG 字符串
   ↓
Python 校验: 解析 XML、检查重叠、检查出界(发现就重试,但没有保证)
```

前一种契约下,**只要 LLM 输出符合 schema,产物就一定对**;后一种只能事后验尸。

---

## 这条原则在本项目的具体落地

按"layout 是否有确定性算法"把 17 种图切两组,**总共 10 native + 7 LLM**:

| 组 | 数量 | 共同点 |
| --- | --- | --- |
| **native** —— 公式驱动 | 5 (bar / line / donut / waterfall / candlestick) | 给定 content → SVG 是函数式输出,同输入同输出 |
| **native** —— 强 layout 约束 | 5 (tree / layer-stack / pyramid / timeline / swimlane) | 都有教科书算法 (Reingold-Tilford 等) |
| **LLM** —— 真自由布局 | 7 (architecture / flowchart / sequence / er / state-machine / venn / quadrant) | layout 选择本身是创作的一部分;LLM "讲得通"的概率高,Python 写死反而僵硬 |

每种类型用什么算法 / 为什么这样切 / 当前每种翻车率多少,见 [[2-演进-Python SDK 6 阶段路线#术语:native vs LLM\|evolution §术语]]
和 Stage 2 的"Plan D 决定:扩 C 到 10 种"一节。

LLM 那 7 种**依然会翻车**(节点多时重叠、字小、出界),所以加了**条件几何重试**
(几何校验 + 反馈重试 1 次)。彻底的根治还得 graph layout 工具(Graphviz / dagre /
d3-force)—— 见 [[2-演进-Python SDK 6 阶段路线#Stage 5 (Plan F):Graphviz/dagre 接入 — 中期路线\|evolution Stage 5]]。

---

## 推广:这条原则在哪些 AI 应用里成立?

只要满足以下条件之一,就该考虑把"layout / 计算 / 求解"剥离出 LLM:

1. **任务有数学公式或确定性算法**:坐标计算、概率分布、统计量、SQL 优化
2. **失败模式可以静态校验**:JSON schema、类型系统、几何约束、单元测试
3. **正确性比创意更重要**:财务计算、医疗剂量、法务条款匹配
4. **要批量、可复现**:测试夹具、报表生成、批处理

反过来,以下场景**应该**让 LLM 主导:

1. **创作决策**:取舍、命名、tone、风格
2. **理解 + 提取**:从非结构化文本中找信息
3. **拓扑没有"对的"答案**:架构图节点摆放、UI 排版、文案 A/B 候选
4. **少量、一次性、人会复核**:demo、原型、灵感生成

---

## 反例:什么时候这条原则不适用

我不希望这文档被读成"凡是有算法的事都不该让 LLM 做"。反例:

1. **算法本身比 LLM 还不可靠**:比如解析自然语言里的"上周三"成日期 —— `dateparser`
   会出错,LLM 反而稳。不要为了"用算法"而引入更脆的工具。

2. **算法实现成本远高于 LLM 错误带来的损失**:写一个能完美 layout 17 种图的引擎
   是 1-2 周的活,但如果用户画的图都是"3 个节点的 architecture",LLM 失败一次重画
   就行,Python 引擎是过度工程。

3. **数据形态特别多变**:LLM 适应力强,Python 算法需要预先知道所有 case。比如
   "把任意 Excel 表整理成 markdown 报告"这种,Python 写规则会写死。

判断尺度:**(算法实现成本 + 维护成本) vs (LLM 失败率 × 失败损失 × 调用频率)**。
本项目的 layout 问题是"高频 + 中等损失 + 算法成本可控",所以 native 化是对的。
你的项目可能不一样。

---

## 致谢

这个 insight 是因为我们的 LLM-renderer 在 case1 的 tree 上具体翻车,然后追根
究底"为什么原 Claude skill 没翻这种车"才挖出来的。如果不是用户(本项目作者)
在看到产物时追问"节点为什么耦合在一起",这个根因可能就被一句"LLM 偶尔会出错"
糊弄过去了。

**所有重要的设计 insight 都来自具体的失败,而不是抽象的思考。** 这个文档
本身就是这条规律的例证。

---

## 参考

- [Reingold–Tilford algorithm](https://reingold.co/tidier-drawings.pdf) — 1981 年的论文,
  解决树布局的"漂亮 + 紧凑"问题
- 本项目 `src/text_to_diagrams/rendering/native/tree.py` 的实现
- 本项目 `src/text_to_diagrams/content_schemas.py` —— native 类型的 content 契约
