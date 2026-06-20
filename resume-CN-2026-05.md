# 梁 柱

13485492580 | [pillarliang21@gmail.com](mailto:pillarliang21@gmail.com) | [github.com/pillarliang](https://github.com/pillarliang) | Blog: [pillarliang.github.io](https://pillarliang.github.io/)

---

## 教育背景


| 学校                | 专业 / 学位     | 时间                |
| ----------------- | ----------- | ----------------- |
| **新加坡国立大学 (NUS)** | 人工智能系统 · 硕士 | 2023.08 – 2024.10 |
| **太原理工大学**        | 软件工程 · 本科   | 2016.09 – 2020.07 |


---

## 工作经历

### Plaud AI | LLM 应用算法工程师 *(2024.10 – 至今)*

负责 Plaud AI 背后的 AI 服务，覆盖 **录音总结、RAG 问答、项目级摘要、MCP / Agent 集成** 四条业务线，纵向贯穿「Model Hub 底层路由 / 容错 / 可观测性 → 上层 Multi-Agent 业务编排」全栈。

#### Plaud Project Summary — Multi-Agent 项目级摘要服务（独立负责,端到端）

**从 0 到 1 独立交付并上线**：聚合一个项目下的多份录音与文档，产出结构化项目报告，支持总结 / 对比 / 进度跟踪三种策略。

- **Agent 工作流编排 (DeepAgents + LangChain)**：按 token 量静态路由 ONE-PASS / MAP-REDUCE；主链「分析文件关系 → 生成大纲 + 抽 RAG 查询 → 检索补全 → 合成 → 审阅润色」串成工具流；大内容委托 map-reduce 子 Agent(context 隔离)，先按文件并行抽取 section、再按 section 维度并行合并(维度转换实现去重)，最后组装；并发写 state 用 reducer 合并防数据竞争
- **Agent 健壮性工程**：System prompt 定序 + 工具限流防死循环，review 后锁写防覆盖；文件预处理先生成 file_summary，Agent 看摘要而非全文，从源头压低 context
- **Temporal 外部服务接入 + K8s 优雅关停**：上游 Workflow HTTP 触发即返 202，后台 asyncio 算完用 Temporal Signal 异步回传；SIGTERM 后先摘流(健康检查转 503)、排空在途任务再退出，保证长任务不被腰斩
- **Citation 引用溯源**：LLM 内短 ID 省 token，出口三级模糊匹配还原(精确 → endswith → contains)并映射回文件名，纯 Python 确定性处理，不过 LLM
- **自研评测体系 (G-Eval)**：三种策略各配加权维度，每维度多采样(N=8)取均值，LLM-as-Judge(gpt-5.1)让 prompt 改动可量化对比

#### Plaud Model Hub — 统一 LLM 调用基础设施（核心贡献者）

公司所有 LLM 调用的底座，统一接入 OpenAI / Anthropic / Gemini / 火山 / DashScope / Bedrock 等多厂商。解决配额限制、成本优化、可用性 SLA 三重约束。支撑 Plaud-Summary / Plaud-Insight 等所有业务线，**日调用量 50 万次，P99 延迟 <800ms，可用性 99.9%**。

- **PassThrough 架构（核心设计）**：单一入口 `invoke()` 支持统一模式（Hub 标准字段翻译成各家 SDK）、同风格透传（零转换）、跨风格透传（Adapter 矩阵翻译 OpenAI ↔ Anthropic / Gemini / Bedrock），业务方无感切换厂商，三种模式共享同一套容错基础设施（路由 / 重试 / 熔断 / 插件）
- **自适应权重 + 权重重分配（调度算法核心）**：按 429 错误率渐进降权替代二值切流，保留 10% 下限作探测流量；权重重分配让应急 Endpoint 从 8% 升至 45% 真正承接流量；recovery_rate 限速恢复防流量瞬涌再次 429。**GSU(80%) + PayGo(20%) 配置下流量平滑转移，单月节省成本 $1200+**
- **三层容错 + 健康管理闭环**：Retry（429 按 `Retry-After` 退避 / 5xx 指数退避）→ Failover（换同模型 Endpoint）→ Fallback（换备选模型保留 passthrough 字段）；熔断器（5xx/超时三状态机）+ 限流器（429 解析 `Retry-After`）+ 自适应权重（429 降权）三种健康信号分工协同；每次调用成功/失败回写滑窗，下次路由前重新计算排除集 + 权重
- **可观测性与工程化**：插件系统两层钩子（请求级 + 尝试级）；Langfuse 全链路追踪（token / 耗时 / 错误率 / 成本）；自适应权重告警（multiplier 跌破阈值触发 WARNING / CRITICAL）；YAML 配置驱动 + AWS AppConfig 热加载，业务方改配额/切模型零代码改动

#### Plaud Summary — AI 录音总结服务（核心业务,除 Mark Note 外全链路参与）

把录音转写变成结构化 Markdown 笔记。覆盖海外 / CN 多 Region、多语种、50+ 业务场景；底层依赖 Kafka (MSK) 异步消费 + FastAPI + Model Hub + Langfuse 全链路追踪。

- **「JSON → Reasoning One-Shot」总结链路重构（最大收益项）**：Markdown 直出释放被强 schema 吃掉的 reasoning output budget；配合长文本 map-reduce、十余场景 One-Shot 模板，**已上线为默认管道**；自研 G-Eval 评估体系（六维度加权 + LLM-as-Judge）让 Prompt 改动可量化对比，**评测分从 30% 提升至 70+%**
- **知识库模板（PFS）三步管道（最有特色）**：近 30 个专业场景走 **Framework → Sections → Polish** 产出专家级笔记；框架步 LLM 动态生成结构并自检，分节步并行抽取（max_concurrency=7），润色步用文本 diff 比例（阈值 0.10）防摆烂并重试；每步独立选模型 + fallback，**评测分从 70+% 提升至 80+%**
- **总结头图生成（Summary Card）**：LLM 生成 HTML（非黑盒生图）→ 浏览器渲染 → S3 归档；1200+ 行设计指南 + 前置拦截 6 类 AI 翻车（分隔符错误 → 布局崩坏、嵌套 grid 缺 col-span-12）；独立渲染服务（Node.js + Playwright）可扩容，多语言字体自动切换 + CN 区域 AIGC 合规
- **工程化与生态**：User-Custom Prompt 污染治理（正则门控 + 小模型复核防出处泄露）；模板推荐（Milvus 向量相似度）+ 审核状态机；Persona 个性化（onboarding 问卷生成画像调整表达风格）

#### Plaud Ask — RAG 问答

**独立从 0 到 1 交付 Integration Skills**(Gmail + Google Docs + Slack)：LLM 起草可编辑卡片、用户审阅后经 MCP 投递；覆盖 API → Handler → Checkpoint → MCP Client 全栈；同时长期负责推荐问题、多语言治理、Embedding 工程化。

- **卡片不污染 LLM 上下文**：分「展示」与「LLM 推理」两通道，避免模型把自己草稿当用户发言反向引用
- **配置驱动 Handler + 双状态机**：注册表 + builder 替代 if/elif，新技能只追加配置；按 MCP 工具数路由三模式；卡片态 / 投递态分离，部分失败仍标成功
- **Email 链 + MCP Client**：`转写 → Markdown → PDF → 投递` PDF 失败非阻断；S3 key 换 presigned URL 让远端 LLM 读图；1500+ 行单测覆盖原子性 / OAuth / 部分失败

---

### 微软（中国）| 算法研究实习生（Excel Copilot） *(2024.01 – 2024.02)*

- 主导 **LLM4Causality** 研究：梳理文献构建基准因果图，设计元数据解析提升预处理效率
- 设计专业推理 Prompts，量化评估 LLM 在复杂因果识别上的优越性
- 用 Excel Lambda 实现 ML 概念验证(One-Hot / 线性 / 岭回归)，证明 Excel 做高级分析可行

---

### 腾讯（深圳）| 前端开发工程师（腾讯文档） *(2020.06 – 2022.06)*

- **域代码格式标准化**：制定数据规范，重构数据 / 排版 / 业务三层架构 100% 兼容 OOXML；首次实现排版引擎域代码自动断行(行业领先)
- **腾讯文档 Drive 研发**：设计视图 / 控制 / 状态 / 网络四层架构，打通移动端文件云端导入 / 存储 / 在线编辑
- **研发效能建设**：封装 Word 引擎 API、升级团队自动化测试框架，从 0 编写性能测试用例(首屏 / 大文档 / 协同)；团队荣获**公司研发效能奖**

---

## 专业技能


| 类别              | 技术栈                                                                              |
| --------------- | -------------------------------------------------------------------------------- |
| **语言**          | Python, TypeScript / JavaScript, SQL, C#                                         |
| **LLM / Agent** | LangChain, LangGraph, DeepAgents, LlamaIndex, MCP, Langfuse, Ragas               |
| **模型与推理**       | Gemini (Vertex AI), GPT (OpenAI), Claude (Bedrock), 豆包 / 通义 (Model Hub)          |
| **后端框架**        | FastAPI, Flask, Gunicorn / Uvicorn, NiceGUI, Gradio                              |
| **数据 / 存储**     | AWS S3, OpenSearch, Milvus, pgvector, Redis, MySQL, MongoDB, Snowflake, Redshift |
| **基础设施**        | Docker, K8s, Supervisor, Temporal, Kafka (MSK), AWS AppConfig                    |
| **可观测性**        | OpenTelemetry, Langfuse, 自定义业务指标                                                 |
| **工程工具**        | uv, Ruff, MyPy, Pre-commit, Git, Linux                                           |


---

## 开源 & 个人项目

### text-to-diagrams | 纯文本一键生成「带 SVG 配图」的 Markdown *(2026.05)*

Python 库 + [Claude Skill plugin](https://github.com/pillarliang/productivity-toolkit/tree/main/plugins/text-to-diagrams) + FastAPI 服务（Docker 一键起），自动为技术文档 / 学习笔记配图。覆盖架构图 / 流程图 / 状态机 / 数据流等 17 种图类型。

- **核心设计：LLM 只做「理解 + 摆放」，不算坐标**：10 种有公式 / 确定性布局的图（树 / 矩阵 / timeline / Sankey）走纯 Python 几何计算，7 种自由拓扑图（架构 / 流程）走 LLM 改 SVG；分工明确避免 LLM 在数学计算上幻觉
- **两层防幻觉护栏**：① 锚点 ID 对照大纲校验（编造节点 → 重生成）；② SVG 输出纯 Python 几何校验（坐标越界 / 重叠 → 反馈重试），成本 +30% 换准确率
- **可插拔多产物形态**：主产物 SVG 图 + 第二产物 Summary Infographic（结论先行文本抽成 8 种原语，按注意力金字塔拼 HTML 信息图）；API `mode` 参数共存，**删子包即下线**，架构解耦

### 企业知识库检索与问答系统 (RAG)*(2024.04 – 2024.08)*

•  扩展开源框架支持多格式文档（doc、扫描版 PDF、图片）；自研版面排序算法优化复杂版面分析与 Markdown 转换；
•  落地查询转换、混合搜索、Graph 增强等策略；用 Ragas 量化评估迭代配置。