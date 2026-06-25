# 总结头图（Summary Card）生成

![[summary-card-pipeline-map.png|760]]

---

## 1. 第一步 · 模型生成：让模型直接产出网页

干脆把版式决策权交给 LLM——让它**直接输出带 Tailwind 类名的 HTML 片段**，而不是输出数据再套模板。

代价：**LLM 的输出不可信**（语法会坏、类名会错、布局会崩）。于是必须配一整套质量防线来兜底

### 1.1 用 Prompt 约束模型怎么画

- **便当（Bento）设计理念**：信息像便当盒一样分格摆放——模块化、有层级、留白、构图平衡。
- **四层信息编排**：L1 页面主标题（< 8 词）→ L2 核心维度（2–5 个，逻辑骨架）→ L3 具体证据 → L4 待办（仅在有明确任务时出现）。
- **四阶段生成**：先做内容盘点、布局蓝图、预检，最后才出 HTML。模型被要求先输出 `<!-- PLAN: ... -->` 注释（前三阶段的思考），再输出正式 HTML——这部分注释会在下一步被清洗掉。

> [!tip] 为什么强制"先规划后出图"
> 直接让模型出 HTML，它容易边写边乱、布局失衡。强制它先写计划注释，相当于让它先想清楚再动手，显著降低布局崩坏率。代价是输出里混进了思考文本，所以清洗环节要专门剥离（见 2.1）。

### 1.2 超时

- **超时**：LLM SDK 的原生超时不总可靠，所以用 `ThreadPoolExecutor` 在线程层面强制超时（默认 120s）——`future.result(timeout)` 到点即放弃。
- **底线**：返回结果统一抽取成纯文本；长度 < 50 字符视为无效，直接返回 `None` 进入降级。

---

## 2. 第二步 · 质量防线：不可信输出怎么变可信

承接上一步——模型给的 HTML 片段不能直接用。这一步用**三道递进防线**把它变可信：先清洗、再体检、最后按严重程度分级处置，通过后才组装成完整页面。

![[quality-defense-flow.png|820]]

### 2.1 清洗：剥掉非 HTML 的杂质

`clean_llm_output()` 按序做三件事，对应模型输出的三类杂质：

1. 去掉 Markdown 代码块包裹（````html` / `````）。
2. 去掉 `<!-- PLAN: ... -->` 规划注释（正是 1.1 强制模型写的思考）。
3. 找到第一个以 `<` 开头的行，丢弃它之前的所有文本（模型偶尔会在 HTML 前面写一段闲话）。

### 2.2 校验：三类校验各司其职

`validate_html()` 产出一个 `ValidationResult`（`is_valid` / `errors` / `warnings` / `has_syntax_error`），由三类检查填充：

| 检查类型     | 手段                                     | 抓什么                         |
| -------- | -------------------------------------- | --------------------------- |
| 语法       | `HTMLSyntaxValidator`（继承 `HTMLParser`） | 未闭合标签、标签不匹配、意外闭标签、解析异常      |
| LLM 高频错误 | 正则 + BeautifulSoup                     | 6 类模型常犯错（见下）                |
| 语义       | BeautifulSoup 遍历                       | col-span 总和、未知颜色、空 icon、空内容 |


> [!note] 六类 LLM 高频 HTML 错误（`check_common_html_errors`）
> 这些是从线上反复观察到的模型坏习惯，专门写正则去抓：
>
> 1. `css=` 写成 `class=`（DeepSeek 常见）
> 2. 属性引号未闭合
> 3. 破损闭标签：`div>` 而非 `</div>`
> 4. 不完整开标签：`<div` 缺 `>`
> 5. Tailwind 类名分隔符错误：`col-span+8` 应为 `col-span-8`
> 6. 嵌套 `grid-cols-12` 缺 `col-span-12`，导致内层被压成 1/12 宽（DeepSeek 常见）

> [!note] 语义校验的级别区分
> 同样不合规，处置不同：
>
> - **Warning（容忍）**：col-span 总和 ≠ 12、颜色不在调色板内——布局可能错位，但不致命。
> - **Error（阻断）**：`<i>` 缺 `data-lucide`（图标不渲染）、内容为空。

### 2.3 分级处置：严重就拒绝，轻微就修复

校验结果决定命运，这是这一步的收口逻辑：

```text
has_syntax_error = True  → 直接拒绝，返回空 HTML（语法错不可靠，不冒险修）
is_valid = False         → 自动修复后继续（fix_html，非致命错）
is_valid = True          → 直接组装
```

`fix_html()` 只做两件保守的修复：给缺 `data-lucide` 的 `<i>` 补默认图标 `circle`；给有 `rounded-card` 但缺 `flex` 的卡片补 `flex flex-col`。修不了的（语法错）一律走拒绝。

### 2.4 组装：把片段拼成完整页面

体检通过后，`assemble_html()` 把片段塞进 `HTML_SKELETON` 骨架，补齐三样外部依赖与一份配置：

- **Tailwind CDN + 设计系统配置**：`generate_tailwind_config()` 生成的 JS，让模型写的自定义类名生效（这是第 3 节的主角）。
- **Lucide Icons**：页面加载后由 `lucide.createIcons()` 把 `<i>` 替换成 SVG。
- **Google Fonts**：按语言只加载对应字体（中文 Noto Sans SC、日文 JP……，未知语言加载全部）。
- **行为兜底 CSS**：flex 防压缩、长文本自然换行。

---

## 3. 第三步 · 设计系统：人与模型共享的词汇表

> 这一节正面回答了代码里 `tailwind_config = generate_tailwind_config()` 那一步在做什么。

**矛盾**：要让模型产出风格统一的卡片，就得让它用一套**固定的样式词汇**；可模型只会写类名（`text-display-2xl`、`bg-glacier-blue-200`、`text-primary`、`rounded-card`），这些类名在标准 Tailwind 里并不存在。

**解法**：用 `design_system.py` 把全部设计 token 定义在 Python 一侧（9 套调色板 × 9 色阶、10 级字号、文字色、语义色、圆角/边框），再由 `generate_tailwind_config()` 把它们**翻译成一段 Tailwind 配置 JS**，在组装时注入页面。于是模型写的那些类名就都有了定义。

![[design-system-dataflow.png|820]]

它具体生成这几样东西：


| 配置                                       | 由什么 token 翻译                | 让哪些类名生效                                                                   |
| ---------------------------------------- | --------------------------- | ------------------------------------------------------------------------- |
| `colors`                                 | 文字色 + 9×9 调色板 + 9×3 语义色     | `text-primary`、`bg-glacier-blue-200`、`text-honey-tea-semantic-success` …… |
| `fontSize`（**完全覆盖** Tailwind 默认）         | 10 级排版 token，每级带 size/行高/字重 | `text-md`（=18px 而非 Tailwind 默认）、`text-display-2xl` ……                     |
| `borderRadius.card` / `borderWidth.card` | 圆角 5px / 边框 1px             | `rounded-card`、`border-card`                                              |


### 3.1 配色方案怎么落地：三段分工

配色是最容易误解的一环——很多人以为"`generate_tailwind_config()` 针对模型用到的类名生成样式"。实际不是。它落地分三段，由三个角色各管一段：

1. **选色在 LLM 侧**：用哪套调色板不是代码定的，而是 Prompt 把 9 套色板名（模板变量 `{{palettes}}`）和一套选择规则交给模型——按内容主题挑 1 套主色（占 70%+）、可选 1 套辅色（~30%），最多 3 套。模型据此写出带具体色板的类名，如 `bg-glacier-blue-200`、`border-glacier-blue-400`。
2. `**generate_tailwind_config()` 全量注册**：它**不知道**模型选了哪套，而是把 9 套 × 9 色阶**无条件全部**塞进 `theme.extend.colors`，产出的只是一张"类名 → 色值"的**字典**，并不是 CSS 规则。
3. **Tailwind CDN 运行时按需生成**：骨架里的 `cdn.tailwindcss.com`（JIT 引擎）扫描页面里实际出现的类名，去第 2 段那张字典查色值，**只为用到的类名**现场生成 CSS（如 `.bg-glacier-blue-200{background-color:#ECF5F9}`）。没被选中的 8 套色板，一条规则都不生成。

> [!tip] 一句话厘清
> "按用到的类名生成样式"确实发生，但在**第 3 段的运行时**，不在 `generate_tailwind_config()`。后者的职责只是把整套色值**提前搬到浏览器端当字典**，让模型随便挑、运行时都查得到——这也是它被称为"模型与校验共用的类名基准"的原因（2.2 的颜色校验同样拿这份调色板判合法性）。

> [!important] 没有这一步会怎样
> `generate_tailwind_config()` 是连接"模型词汇"和"实际渲染"的桥。少了它，注入页面的 `tailwind.config` 就没有这些自定义键，模型写的 `text-display-2xl`、`bg-glacier-blue-200` 全部变成**无定义的空类**——卡片会以浏览器默认样式"裸奔"，配色、字号、圆角全部失效。
>
> 所以这一步本质是：**把设计系统从 Python 端搬运到浏览器端，作为模型与校验器共用的同一份"类名基准"**（2.2 的颜色校验也是拿调色板做基准）。

> [!note] 设计系统速查
>
> - **调色板（9 套）**：honey-tea / almond-cream / willow-bud / sea-mist / glacier-blue / orchid-grey / lavender-mist / sakura-pink / soft-rose，每套 100–900 共 9 阶（浅→背景，深→文字/图标），另带 success/error/warning 语义色。
> - **文字色**：primary `#000` / secondary `#3D3D3D` / tertiary `#7A7A7A`（仅水印）/ error / success / warning。
> - **字号**：display-2xl(72) → sm(16)，正文默认 `text-md`(18px)，最小 `text-sm`(16px) 仅用于 badge/tag。
> - **图标**：Lucide，`<i data-lucide="...">` + `lucide.createIcons()`。

---

## 4. 第四步 · 落地成像：渲染 → 存储 → 插回正文

完整 HTML 到手后，这一步把它落成正文里的一张图，分三段。

**① 渲染成图**（`_convert_html_to_png`）：POST 到外部无头浏览器服务的 `/api/render`，等 `networkidle`（确保字体、图标、CDN 全加载完）后截图。两层超时保护——单次请求 30s、含重试总超时 60s；对 5xx 自动重试 3 次（指数退避 0.5→1→2s）。尺寸直接读 PNG 的 IHDR chunk，不解码整图。

**② 存储**（`_upload_to_s3`）：上传 S3，返回 storage_key（不是预签名 URL，由客户端自行换可访问 URL）。存多久取决于用户隐私开关 PPC：


| PPC 状态   | 存储类型                | 过期   |
| -------- | ------------------- | ---- |
| ENABLED  | `permanent`         | 永久   |
| DISABLED | `temporary_30_days` | 30 天 |


路径结构：`{permanent|temporary_30_days}/{user_id}/summary_poster/card_{summary_id}_{时间戳}_{uuid8}.png`。

**③ 插回正文**（`insert_card_to_markdown`，静态方法）：以 `![PLAUD NOTE](storage_key)` 形式，插在第一个 H1 标题之后（无 H1 则插在开头），前后补空行。对应的 `remove_card_from_markdown()` 负责审核失败时撤回，并智能清理多余空行。

---

## 5. 第五步 · 合规与兜底：CN 特化 + 内容审核

旁路也不能产出违规内容。这一步有两条独立的合规链，分别处理"标识"和"内容"。

### 5.1 CN 区域合规标识（仅 `config.is_cn_region()`）

中国法规要求 AIGC 内容可见可溯源，于是在两个时机各加一层标识：

- **组装期 · 水印**：画布右上角加"内容由 Plaud AI 生成"（按语言取文案，见 `WATERMARK_TEXT_MAP`），颜色用 tertiary `#7A7A7A`。
- **上传期 · AIGC 元数据**：用 Pillow 把合规字段（Label、ContentProducer、ProduceID、传播者/传播 ID、免责声明等）写进 PNG 的 tEXt chunk。失败则返回原图，不阻断。

### 5.2 内容审核兜底

**矛盾**：头图文字是模型生成的，可能含违规内容；而图片渲染后人眼不便审核。

**解法**：渲染前就用 BeautifulSoup 从 HTML body 抽出纯文本（`extract_text_from_html`），随总结 Markdown **一起送审**。审核不通过则执行三步撤回：

![[moderation-fallback-flow.png|760]]

```text
delete_from_s3(storage_key)              # 删 S3 图片
remove_card_from_markdown(md, key)       # 移除正文引用
result["markdown"] = ""                  # 审核失败不返回任何内容
```

> [!note] 元数据清理
> 无论审核是否通过，最后都会 `del result["_extra_moderation"]`，避免审核用的中间字段（card_text、storage_key）泄露给调用方。

---

## 6. 可观测性（贯穿全程）

旁路要"坏得明明白白"，所以观测覆盖每一步，三套机制互补：

- **Prometheus 指标**（`metrics.py`）：各阶段 latency（LLM/HTML/PNG/S3/总）、PNG 大小与宽高、Markdown 长度、HTML 校验错误数（按错误类型打标签）。直方图桶按业务实际分布设定。
- **事件埋点**（`tracking/events.py`）：
  - `track_card`（非 PII）——成功/失败都发，带 status、scenario、各阶段耗时、`failed_stage`、`error_message` 等，是线上排障主入口。
  - `track_pii_card_content`（含 PII）——记录模型原始 HTML，仅用于调试输出质量。
- **Langfuse 追踪**：记录 LLM 调用的输入/输出/耗时/token，关联 summary_id 与 scenario。

---

## 附录

> [!note] 核心文件清单
>
>
> | 文件                                          | 职责                                                  |
> | ------------------------------------------- | --------------------------------------------------- |
> | `services/summary_card/poster_generator.py` | 主生成器 `SummaryCardGenerator`，串联全流程，含 Fallback Prompt |
> | `services/summary_card/html_assembler.py`   | 清洗 / 校验 / 修复 / 组装                                   |
> | `services/summary_card/design_system.py`    | 设计系统 token 与 `generate_tailwind_config()`           |
> | `services/summary_card/metrics.py`          | Prometheus 指标                                       |
> | `plaud/basic_runnable_summary.py`           | 入口：调用生成器 + 插入 Markdown                              |
> | `summary.py`                                | 内容审核与失败清理                                           |
> | `config.py`                                 | 全部配置项                                               |
>

> [!note] 主路径关键配置项（`config.py`，AWS AppConfig 管理）
>
>
> | 配置                                                    | 默认               | 说明            |
> | ----------------------------------------------------- | ---------------- | ------------- |
> | `summary_card.llm_model`                              | `gemini-2.5-pro` | 模型            |
> | `summary_card.llm_timeout`                            | `120`            | LLM 超时（秒）     |
> | `summary_card.thinking_level` / `reasoning_effort`    | `low` / `medium` | 模型推理强度        |
> | `summary_card.html_to_png_timeout` / `_total_timeout` | `30` / `60`      | 渲染单次 / 总超时（秒） |
> | `summary_card.max_dimension`                          | `1920`           | 画布最大尺寸        |
> | `summary_card.device_scale_factor`                    | `2.0`            | DPI 缩放        |
>

