# Plaud Summary — AI 录音总结服务（面试介绍）

> 配套简历条目：工作经历 → Plaud AI →「Plaud Summary — AI 录音总结服务（核心业务，除 Mark Note 外全链路参与）」
> 用途：面试时介绍这个项目用。先看「一句话 + 电梯陈述」，再按需展开每个 feature。

---

## 0. 一句话定位

**Plaud Summary 是 Plaud AI 最核心的产品形态：把一段录音的转写文本，按用户选定的「场景 / 模板」产出一份高质量、结构化的 Markdown 总结。** 覆盖海外 / 中国多 Region、多语种、50+ 业务场景；底层依赖 Kafka(MSK) 异步消费 + FastAPI + 自研 Model Hub 统一调模型 + Langfuse 全链路追踪。

我在其中**除 Mark Note 外全链路参与**，按时间顺序主导了几件最有代表性的事：① 总结链路从 JSON 到 Reasoning One-Shot 的重构（最大收益项）② 用户自定义模板的 Prompt 污染治理 ③ 知识库模板管道（PFS）④ 总结头图生成，以及模板推荐、多语言治理等。

---

## 1. 电梯陈述（30 秒 / 60 秒两版）

**30 秒版：**
> Plaud Summary 是把录音转写变成结构化笔记的服务。它的核心是一套「模板体系」——用户可以选官方场景模板、自己写自定义模板、或用我们的知识库模板（每个专业场景 LLM 动态生成结构化框架）。我按时间顺序主导了几条主线：先是把总结链路从「逼模型吐 JSON」重构成「Markdown 直出」，省下来的推理预算让质量明显变好，现在是默认管道；然后做了自定义模板的 Prompt 污染治理；以及知识库模板这条用「框架→分节→润色」三步管道产出专家级笔记的链路；还有生图——把总结渲染成可分享的海报卡片。

**60 秒版（加技术深度）：**
> 在 30 秒版基础上补充：整个服务跑在 Kafka 异步消费上，统一通过我们自研的 Model Hub 调 Gemini / GPT / Claude，按场景做模型路由和容错。模板这块我把它理解成三类：官方 One-Shot 模板、知识库模板（PFS）、用户自定义模板，每类的生成链路和工程难点都不一样——比如自定义模板要防 Prompt 注入和模型出处泄露，我用「正则门控 + 小模型复核」双层方案；知识库模板要保证润色真的有增量，我用文本 diff 比例做重试门槛。另外生图这块用 LLM 生成 Tailwind HTML 再渲染成 PNG，有一套超详细的 Prompt 工程（1000+ 行）和设计系统。

---

## 2. Feature 全景图（先分清有哪些）

Summary 的 feature 可以按「**用什么模板生成**」这条主轴切成几类，这是面试时最该讲清楚的结构：

```
                         Plaud Summary
                              │
        ┌─────────────────────┼──────────────────────┐
        │                     │                       │
   【总结生成】           【模板体系】              【增值产物】
        │                     │                       │
  ·One-Shot 默认管道    ·官方场景模板(50+)        ·生图 / 海报卡片
  ·JSON→Reasoning 重构  ·知识库模板 PFS(≈30)      ·标题 / 关键词 / 行业分类
  ·长文本 map-reduce    ·用户自定义模板            ·推荐问题
  ·场景路由 ai-choice   ·社区模板(审核发布)        ·内容审核 moderation
  ·Persona 个性化       ·模板推荐(向量检索)
```

### 模板体系一览（核心，要能背）

| 模板类型 | 是什么 | 生成链路 | 代表性亮点 |
|---|---|---|---|
| **官方场景模板** | 平台预置的 50+ 业务场景（会议/课程/面试/通话/医疗/销售/日记…），存 S3+DB，向量库做推荐 | 走 **Reasoning One-Shot**：单次调用 Markdown 直出 | JSON→Reasoning 重构，已上线为默认管道 |
| **知识库模板 (PFS)** ⭐ | 近 30 个**专业领域**场景（律师-客户、董事会纪要、医患问诊、离职面谈、根因分析、SOP、嫌疑人审讯…），LLM 动态生成结构框架 | 走 **3 步管道：Framework → Sections → Polish**，分节并行抽取 | 专家级结构化输出；每步独立选模型；润色用文本 diff 做重试门槛 |
| **用户自定义模板** | 用户自己写一段 One-Shot Prompt 当模板 | `UserCustomNote`：双栅栏隔离 + AI ACK | Prompt 污染治理（正则门控 + 小模型复核） |
| **社区模板** | 用户发布、经审核的共享模板 | 版本管理 + 审核状态机 | IN_REVIEW→PUBLISHED/REJECTED |

> ⭐ 知识库模板是最有特色的一条线。下面按**时间顺序**逐个深讲。

---

## 3. JSON → Reasoning One-Shot 重构（最大收益项，默认管道）

> 简历原话：「主导『JSON → Reasoning One-Shot』总结链路重构（最大收益项）」
> 代码：`plaud_summary/plaud/reasoning_one_shot_note.py`

### 3.1 问题 / 为什么要重构

老链路逼模型输出强 schema 的 JSON，再解析成 Pydantic、再转 Markdown。强 schema 有三个代价：

① **推理预算被「保证 JSON 合法」吃掉**，本该用于内容质量的 reasoning output budget 被格式化消耗
② **结构被字段框死**，创意性内容表达不出来（比如某个字段模型想展开讲但 schema 只给了一行）
③ **内容质量校验成重试源**：为保证输出质量，必须对 JSON 内容强校验（字段长度、列表至少 N 条、必填项不为空），不达标就重试；而 Markdown 直出没有这些硬性门槛

### 3.2 重构方案

`ReasoningOneShotNote` 直接让模型**输出 Markdown**，省掉 JSON 格式化和解析校验，把释放出来的 reasoning output budget 全用于内容深度。

**1. 十余场景 One-Shot 模板**
为 18 个高频场景各维护了一份专用 Prompt 常量（分类存放在 `plaud/prompts/official_category/` 下，包括 meeting / interview / medical / sales / education / call / consulting / construction / general 等），每个场景的模板定义了该场景应有的结构和示例；代码里通过字典映射 `scenario → template`，未覆盖的场景自动回退到通用模板 `REASONING_DETAILED_TEMPLATE`。

**2. 长文本 map-reduce**
超阈值（~8K token）触发分段并行抽取 → 二次合并（`BatchLLMChain`），支持 2h+ 录音。

### 3.3 如何衡量优化效果 — G-Eval 自动评估体系

**核心问题**：从 JSON 改到 Markdown 直出、或者任何 Prompt 优化，如何量化验证效果？

**方案**：自研 G-Eval 评估框架（`tests/evaluation/g_eval/`），用 LLM 对总结质量打分。

**六个评估维度**（1-5 分）：
- **Faithfulness (35% 权重)**：事实一致性，是否出现幻觉
- **Relevance (30%)**：相关性，是否抓住重点
- **Completeness (15%)**：完整性，是否遗漏关键信息
- **Fluency (10%)**：流畅性，语法/拼写/表达
- **Coherence (10%)**：连贯性，结构是否合理
- ~~**Conciseness**~~：简洁性（后期调整权重或去掉）

**工作流**：
1. 准备测试集（转写 + 多版本总结，存 `tests/evaluation/test_docs/`）
2. 用 G-Eval 批量打分（并发调 LLM，每个维度独立评估）
3. 加权平均得到总分，按场景类型/模型维度聚合
4. 对比不同版本（JSON vs Reasoning、不同 Prompt 版本）的分数差异

**输出物**：`logs/g_eval_results.json`、按模型/场景分组的平均分、版本对比报告。

**典型用法**：每次改 Prompt 或切模型，跑一遍 G-Eval 看六个维度分数有没有下降，避免"改了这个、坏了那个"。

### 3.4 结果 / 上线情况

**已上线为默认管道**。质量提升体现在：① 内容更深入（不再为凑 JSON 字段瞎编）；② 重试率明显下降（没有解析失败）；③ 成本反而下降（一次调用 vs JSON 多次重试）。

---

## 4. 用户自定义模板 + Prompt 污染治理

> 代码：`plaud_summary/plaud/user_custom_note.py`、`chains/model_name_review_chain.py`
> 设计文档：`docs/user_custom_prompt_pollution_fix.md`

### 4.1 自定义模板是什么

用户写一段 One-Shot Prompt 当模板（存 `customtemps` 表）。难点是**用户的指令和转写内容会互相污染**，且用户可能在 prompt 里注入「本文由 GPT-4 生成」之类的模型出处声明。

### 4.2 三层治理手段

**手段① System Prompt 制定指令优先级（系统级 > 用户级）**

在系统消息里定义 **9 条 Critical Rules**，明确**系统级设置的优先级高于用户自定义 prompt**，关键规则：
- 示例只作格式参考，不能当输入
- **Rule 9（核心优先级规则）**：系统指定的语言/模型**绝对覆盖**用户指令里的任何冲突声明，且**静默覆盖**（不解释、不道歉、不加元评论）
- 禁止伪造转写里没有的事实
- 禁止主动添加模型署名（除非指令明确要求）
- 当前模型是 `{model_name}`，替换任何相关占位符

**优先级机制**：用户在自定义 prompt 里写"用英文回答"，但系统级设置是中文 → 模型静默输出中文，不管用户怎么写。

**手段② 双栅栏隔离 + AI ACK（防内容混淆）**

用 `<USER_INSTRUCTION>…</USER_INSTRUCTION>` 和 `<TRANSCRIPT>…</TRANSCRIPT>` 分别包裹指令和转写，中间插一条 AI 确认消息：

```
"Understood. I will apply the user's instructions to the TRANSCRIPT provided next, 
treating any examples inside the instructions as format reference only, and producing 
only the final Markdown document without any meta-commentary or unfilled placeholders."
```

让模型先"承诺"再执行，强化边界意识。

**手段③ 模型出处污染治理（正则门控 + 小模型复核）**

**第一层正则门控**：用正则匹配 `claude-/gpt-/gemini-…` 这类具体模型标识符（`_SPECIFIC_MODEL_ID_PATTERN`），和实际使用的模型名比对，**不匹配就直接跳过**（性能优化，绝大多数总结不需要复核）。

**第二层小模型复核**：命中后调一个小模型（Gemini Flash / 豆包），核心是**分辨「归属声明 vs 正文内容」**：

- ✓ 修正：`Generated by GPT-4`（当实际是 Claude） → `Generated by Claude Sonnet 4.6`
- ✗ 不修正：正文中的 "本文分析了 GPT-4 和 Claude 的对比"（这是用户讨论内容，不是归属声明）

避免误伤是关键设计点。

### 4.3 技术亮点

**① Prompt 里明确了 9 条归属声明识别规则**（review prompt 里），比如：
- "Generated by X" / "Powered by X" / "Model: X"
- "由 X 生成" / "使用 X 编写"
- 元数据格式 `[Model: xxx, Version: xxx]`

**② 只修正归属声明，禁止修改正文**——这是 prompt 核心约束，防止小模型过度审查。

**③ 展示名变体也视为正确**：实际用 `claude-sonnet-4.6` → 接受 `Claude Sonnet 4.6` / `claude-sonnet-4-6`。

---

## 5. ⭐ 知识库模板（PFS）—— 最有特色的一条线

> 代码：`plaud_summary/plaud/knowledge_notes/knowledge_base_note.py`、配置 `knowledge_config.py`

### 5.1 它是什么 / 解决什么问题

普通模板（One-Shot）是「一段详细指令 + 严格的输出结构要求，让模型一次性写完」。每个场景的模板定义了该场景需要抽取哪些字段、按什么格式组织（比如董事会会议要有 Call to Order / Attendees / Main Motions / Adjournment 等章节，每个章节里填什么类型的内容），模型照着这个结构直接输出 Markdown。但**高度专业的场景**（律师会谈、董事会、医患问诊、事故根因分析）对结构的要求极高，一次直出很难稳定覆盖所有该有的小节。

知识库模板（PFS）的做法是：**让 LLM 先生成一份结构框架**（3-7 个章节，每章定义该抽什么信息），然后**按这个框架逐节抽取、再统一润色**。不再赌模型一次写好，而是拆成「框架设计 → 分节执行 → 整体润色」三步。

**覆盖的专业场景**（近 30 个）：律师-客户沟通、董事会纪要、医患问诊(SOAP)、离职面谈、客诉记录、事故调查、商务谈判、头脑风暴、法律听证、问题根因分析、专业咨询、研讨会、需求评审、销售推介、SOP、嫌疑人审讯……

### 5.2 核心实现：Framework → Sections → Polish 三步管道

```
转写文本
  ↓
[Step 0] 语言检测
  ↓
[Step 1] Framework（框架）
  • LLM 动态生成结构框架（识别逻辑→对齐最佳实践→合成结构→MECE审计）
  • 输出 3-7 个 sections，每个定义 <section-description> + <key-information>
  ↓
[Step 2] Sections（分节抽取 — 并行）
  • 每个 section 独立调 LLM，按定义从转写里抽取对应信息
  • max_concurrency = 7（并发上限）
  ↓
[Step 3] Polish（润色 — 带 diff 重试）
  • 把分节结果合并后统一润色成自然成稿
  • 文本 diff 比例低于阈值（0.10）→ 重试（最多 3 次），防模型"假润色"
  ↓
[Step 4] Header（生成标题/关键词/行业分类）
```

### 5.3 三个能讲的工程亮点

**亮点① 每一步独立选模型 + 三种推理强度控制**

三步各自从配置里读 `model + fallback_models + 推理参数`（`PipelineStepConfig` 从 AppConfig 热读），而且适配了三套不同推理控制 API：
- Gemini 2.5 用 `thinking_budget`（token 数）
- Gemini 3 用 `thinking_level`（low/high）
- GPT-5 / o3 用 `reasoning_effort`

→ 框架步可以用强推理模型保证结构，分节步可以用快模型压成本，全部可灰度调。

**亮点② 用「文本 diff 比例」防润色摆烂**

润色步最怕模型「看起来润色了但其实原样返回」。我用 `difflib.SequenceMatcher` 算润色前后的差异率（比较框架原始文本和润色后的 summary），**低于阈值（0.10）→ 重试**（最多 3 次），保留 diff 最大的那版。把「质量」量化成可重试的硬指标。

**亮点③ 长短两版 + 灰度发布 + 安全兜底**

- **长版（3 步全量）vs 短版（单次调用）**：按 token 阈值(5000) / 音频时长阈值(15min) + ratio 决定走哪版——短录音没必要上重管道。
- **按场景百分比灰度**（`knowledge_pipeline.scenario_ratios`），**任何配置解析出错都安全回退到原模板**，不影响线上。
- **LLM 调用层**有多端点 fallback、Gemini 区域 429 自动切全局端点、指数退避、Hub 路径感知。

### 5.4 和 One-Shot 的本质区别（高频追问）

| | One-Shot 模板 | 知识库模板 PFS |
|---|---|---|
| **生成方式** | 单次调用，Markdown 直出 | 3 步管道，分节并行 + 润色 |
| **结构来源** | Prompt 里的示例 | LLM 动态生成的 framework |
| **适合** | 通用场景，追求速度/成本 | 高专业度场景，追求结构完整 |
| **成本** | 低 | 高（多次 LLM 调用） |

**路由优先级**（`SummaryFactory.create()`）：知识库模板 > Reasoning One-Shot > 传统模板，都通过配置灰度控制流量。

---

## 6. 生图 / 海报卡片（Summary Card）

> 代码：`plaud_summary/services/summary_card/`（poster_generator / html_assembler / design_system / metrics）

### 6.1 问题 / 为什么要做

**核心痛点**：用户大多没耐心读完整段摘要——尤其在分享给同事、贴到群里、保存到 Notion 时，他们更想要「一眼看完核心」的东西。

**方案**：把摘要 Markdown 自动画成一张 **1368px 宽的 Bento 风格图**，3-5 秒就能扫完核心结论、关键数据、待办事项，再决定要不要点开看完整正文。

```
摘要 markdown  →  一张可分享的 PNG 头图
（LLM 已生成）    （放在正文最前面）
```

### 6.2 端到端 4 步流程

整条链路分 4 步串行，每一步独立计时、独立兜底，**任何一步失败都不会出半成品**——要么完整成功，要么直接没有头图，不影响摘要主链路。

| 步骤 | 输入 | 输出 | 耗时 | 失败兜底 |
|---|---|---|---|---|
| ① AI 把摘要画出来 | 摘要 markdown | HTML 代码片段 | 几秒到十几秒 | 内容太短直接判失败 |
| ② 把 HTML 加工成完整网页 | 代码片段 | 完整 HTML 页面 | 毫秒级 | 严重错误直接拒绝出图 |
| ③ 浏览器把网页渲染成图片 | HTML 页面 | PNG 二进制 | 1-3 秒 | 失败自动重试 3 次 |
| ④ 图片归档到云存储 | PNG | 可访问的 URL | 毫秒级 | 上传失败上报监控 |

### 6.3 核心难点：如何"教" AI 设计

**直接让 AI 生成图片的问题**：黑盒、不可控、改一个字要重画整张、没法 A/B、没法多语言。

**我们的方案**：让 AI **输出 HTML 代码**（包含布局/样式/配色/图标），然后用浏览器渲染成图。优势：每个元素都可审查/修复/复用；改色板秒出新版；同一份内容适配多尺寸。

**真正的难点**：如果只丢一句"请把这段摘要画好看一点"，AI 会写出五花八门的结果——颜色刺眼、字号混乱、icon 错位、把"2024 年"塞进圆形头像里溢出来。

**解决方案**：写一份 **1200+ 行的设计指南** 给 AI，分 5 个模块：

**模块 A：设计哲学**（Bento 便当盒风格）
- 模块化：每个格子是独立的信息单元
- 层次感：用大小/颜色区分重要性
- 留白：不填满，让视觉呼吸

**模块 B：5 条硬规则**（防止常见错误）
- 垂直对齐规则：多行文字旁边的小图标按语义对齐，禁用 `mt-*` 微调
- 固定容器约束：3 位以上数字必须改用胶囊形状（否则圆形头像溢出）
- 禁止手动写死宽度：`max-w-[400px]` 会跑版
- 不准出现 Meta 信息："由 AI 生成"、版权水印
- 不准推断日期：摘要里写"下周一"，必须原样展示，**禁止补全成"2024 年 11 月 20 日"**（LLM 不知道录音真实时间）

**模块 C：信息层级**（L1 标题锚点 / L2 叙事支柱 / L3 细节节点 / L4 行动事项）

**模块 D：开篇策略**（有风险→问题优先；有决策→结果优先；多进展→状态优先；静态知识→锚点优先）

**模块 E：4 阶段强制思考**（最关键！）
让 AI **必须先在脑子里走完 4 步，再开始写 HTML**。具体做法：让它在 HTML 之前先输出一段 `<!-- PLAN: ... -->` 注释，把 1-3 步的产物全部写出来。

| 阶段 | AI 要做的事 | 解决的问题 |
|---|---|---|
| Phase 1 — 内容盘点 | 列出所有可展示信息，按 CRITICAL / IMPORTANT / SUPPORTING 分类，统计总数 | 防止漏掉关键信息或塞太多废话 |
| Phase 2 — 排版蓝图 | 根据信息密度决定卡片总数和每张卡片的列宽，每张写一句话蓝图 | 防止内容少时大片留白，内容多时挤成一团 |
| Phase 3 — 飞行前检查 | 自检：有没有空卡？有没有 L2 只挂 1 个 L3？高度是否在 768-1368px？30 秒能不能扫完？ | 让 AI 自己抓出问题，不让烂稿子流到下一步 |
| Phase 4 — 输出 HTML | 严格按 Phase 2 蓝图写 HTML | 减少自由发挥导致的崩坏 |

**真实案例**（GPT 原理解析文章 → 海报卡片）：

Phase 1 盘点结果：
```
Total: 15 items (Critical: 9, Important: 3, Supporting: 3)
Density: RICH
```

Phase 2 排版蓝图：
```
Card 1: [col-span-12] L1 Title Area - Transformer模型入门：GPT内部工作原理解析
Card 2: [col-span-7] L2 GPT概念拆解 - 核心定义与GPT起源
Card 3: [col-span-5] L2 规模与权重 - 参数量的关键影响 (Emphasis Card)
Card 4: [col-span-4] L2 文本生成循环 - 预测-采样-追加流程
Card 5: [col-span-8] L2 数据处理全景 - 注意力机制与前馈层流动
Card 6: [col-span-12] L2 概率输出与调控 - Softmax函数与温度参数
```

Phase 3 自检通过：6 张卡片、每张 ≥2 个条目、高度约 1100px、主题不重复。

Phase 4 输出：完整 HTML，包含 GPT 三个字母拆解卡片、6.17 亿权重数据展示、温度参数滑块、语义空间运算示例等。

### 6.4 技术亮点

**① 设计规范单源管理 + Tailwind 注入**
设计师定义的所有规范（9 套色板、每套色板专属的成功/警告/错误色、10 级字号、圆角、边框）**全部集中在 `DESIGN_TOKENS` 一个文件里**，每次组装 HTML 时自动转成 Tailwind config 嵌到页面里。

好处：
- 设计师改一处颜色 → Tailwind config / Prompt 示例 / 校验规则 自动同步
- AI 想用规范外的字号（比如 14px）→ 类名根本不存在，写了也没效果
- 不同卡片之间视觉绝对一致

**② 6 类「AI 高频翻车」专项拦截**
用正则 + BeautifulSoup 前置检测（比在 prompt 里说"不要犯 XX 错误"更可靠）：

| 错误形态 | 典型来源 | 后果 |
|---|---|---|
| `css="..."` 写成了 `class="..."` | DeepSeek 高频 | 样式完全不生效 |
| 引号没闭合 | 输出被截断 | 后面整段崩 |
| Tailwind 类名分隔符写错（`col-span+8`） | 通用错误 | **布局崩坏**——卡片错位 |
| 嵌套 grid 缺 `col-span-12` | DeepSeek 高频 | **内层挤到 1/12 宽**——视觉灾难 |

**严重错误一律拒绝出图**——避免推一张明显有问题的卡片给用户。

**③ 独立渲染服务（plaud-webrender）**
渲染服务 = Node.js + Playwright（无头 Chromium）+ Express，独立部署。

为什么不内嵌到 Python 主服务：
- 浏览器渲染很重，独立部署可池化 / 可扩容
- 流量大时单独扩容，不影响主服务
- 后续新的图表渲染路径也复用同一个服务

关键参数：
- `waitUntil: networkidle`（等网络完全空闲才截图，保证 Tailwind 和字体都加载完）
- `deviceScaleFactor: 2.0`（Retina 倍率，高清屏不糊）
- `contextKey: summary-card`（复用浏览器实例，二次渲染显著加速）

**④ 多语言字体自动切换**
按用户语言自动切换字体栈（英/法/德/意/西/葡用 Inter，简中用 Noto Sans SC，繁中/日/韩各自对应），字体走 Google Fonts CDN，浏览器自动缓存。

**⑤ CN 区域 AIGC 合规**
PNG 文件里嵌入隐藏元数据（PNG `tEXt` chunk），符合《生成式人工智能服务管理办法》标识要求：

```json
{
  "Label": "1",
  "ContentProducer": "plaud-summary",
  "DisclaimerText": "内容由 Plaud AI 生成"
}
```

声明文案支持 12 种语言，按用户语言自动切换。右上角水印 + PNG 元数据双重合规。

**⑥ 端到端 OpenTelemetry 指标**
各阶段耗时（LLM / HTML / PNG / S3）、Prompt 版本号、失败链路 + 阶段标记、PNG 大小/宽高——方便定位是 AI 慢还是渲染慢，设计指南改版后效果变差能快速回滚。

---

## 7. 其他功能点（被追问时补充）

### 模板推荐（向量检索）

- 官方模板在 Milvus 向量库中同步（`api/official_templates/vector_db_service.py`）
- `get_template_auto_select`：基于转写内容向量相似度推荐单个最匹配模板
- `get_template_recommendations`：推荐列表 + 理由/卖点/描述
- Fallback: API 错误 → REASONING_NOTE（默认模板）

### 社区模板

- 用户发布 → 审核状态机（IN_REVIEW → PUBLISHED / REJECTED / UNAVAILABLE）
- 版本管理：`community_templates` + `community_template_versions`
- 审核信息：reviewer_id / reviewed_at / comment

### Persona 个性化

- 从用户 onboarding 问卷生成画像（`features/persona/`）
- 支持场景切 persona 版 prompt，调整表达风格

---

## 8. 底层架构（被问到工程能力时讲）

- **Model Hub（自研统一调用层）**：Logical Model / Endpoint / Provider 三层抽象屏蔽供应商差异，内部托管路由、容错、可观测；Summary 所有 LLM 调用都走 Hub，支持按场景路由 Gemini / GPT / Claude、Adaptive Weight 渐进降权、Fallback 环路检测。
- **异步消费**：Kafka(MSK) 消费总结任务；自定义模板的标签名翻译等也走 handler 异步处理。
- **多 Region / 多语种**：海外 + 中国独立部署，模板的 `note_tab_name` 多语言、生图字体/水印按语言切换。
- **可观测性**：Langfuse 全链路追踪每次 LLM 调用 + 自定义业务指标（token、延迟、压缩率、chunk 数、各 feature 成功率）。
- **内容审核**：总结产出后过 moderation，命中清空 Markdown 并删除已生成的海报。

---

## 9. 面试高频追问 & 应答要点

**Q：模板这么多类，怎么决定一段录音用哪条管道？**
A：在 `SummaryFactory` 里按优先级判定——先看是不是知识库场景（`is_knowledge_scenario` + 灰度比例），再看是不是 One-Shot 支持场景，否则回退传统模板。每条都用 AppConfig 的灰度比例控制流量，配置出错一律安全回退，保证线上不挂。

**Q：知识库模板分节并行，怎么保证最后是一篇连贯的文章而不是拼接？**
A：Framework 步 LLM 先设计好整体结构；分节只负责「按框架抽准信息」，连贯性交给 Polish 步统一润色；而且我用文本 diff 比例确保润色真的把碎片重组过，不达标会重试。

**Q：One-Shot 直出 Markdown，没有 schema 了，怎么保证格式/质量稳定？**
A：① 每场景有带示例的专用模板约束结构；② self-check 链 + 标题重试做后处理；③ 长文本走分段+合并；④ Langfuse 上能量化对比每次改动的质量（配合自研 G-Eval 评测体系）。

**Q：自定义模板让用户随便写 Prompt，怎么防注入？**
A：双栅栏隔离 + AI ACK + 系统级覆盖规则防内容污染；正则门控 + 小模型复核防模型出处泄露，关键是只改归属声明、不碰正文，避免误伤。

**Q：你具体负责哪些、和团队怎么分工？**
A：除 Mark Note 外全链路参与。按时间顺序主导了：JSON→Reasoning One-Shot 重构（最大收益项，已成默认管道）、自定义模板污染治理、知识库模板（PFS）管道、生图；模板推荐、多语言治理等也参与。

---

## 10. 关键代码位置速查（被要求看代码时）

| 模块 | 路径 |
|---|---|
| 顶层入口 / 并发编排 | `plaud_summary/summary.py` |
| 模板工厂（管道分发） | `plaud_summary/plaud/summary_factory.py` |
| One-Shot 重构 | `plaud_summary/plaud/reasoning_one_shot_note.py` |
| 自定义模板 | `plaud_summary/plaud/user_custom_note.py` |
| Prompt 污染治理 | `plaud_summary/chains/model_name_review_chain.py` |
| 知识库模板 PFS | `plaud_summary/plaud/knowledge_notes/knowledge_base_note.py` + `knowledge_config.py` |
| 生图 | `plaud_summary/services/summary_card/` |
| 场景枚举 | `plaud_summary/plaud/config.py`（`SCENARIO`） |
| 模板推荐 | `plaud_summary/template_recommend.py` |
