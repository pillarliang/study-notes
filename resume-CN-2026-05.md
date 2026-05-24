# 梁 柱

13485492580 ｜ [pillarliang21@gmail.com](mailto:pillarliang21@gmail.com) ｜ [github.com/pillarliang](https://github.com/pillarliang) ｜ Blog: [pillarliang.github.io](https://pillarliang.github.io/)

---

## 教育背景


| 学校                | 专业 / 学位     | 时间                |
| ----------------- | ----------- | ----------------- |
| **新加坡国立大学 (NUS)** | 人工智能系统 · 硕士 | 2023.08 – 2024.10 |
| **太原理工大学**        | 软件工程 · 本科   | 2016.09 – 2020.07 |


---

## 工作经历

### Plaud AI ｜ LLM 应用算法工程师 *(2024.10 – 至今)*

负责 Plaud AI 背后的 AI 服务,覆盖 **录音总结、RAG 问答、项目级摘要、MCP / Agent 集成** 四条核心 AI 业务线,纵向贯穿「Plaud Model Hub 底层模型路由 / 容错 / 可观测性 → 上层 Multi-Agent 业务编排」全栈。

#### Plaud Project Summary — Multi-Agent 项目级摘要服务（独立负责，端到端）

**从 0 到 1 独立交付并上线**(覆盖 Demo / 架构 / 评测全链路):聚合同一项目下的多份录音与文档,产出结构化项目报告,支持总结、对比、进度跟踪三种总结策略。

- **Agent 架构(DeepAgents + LangChain)**:基于 tiktoken 实测 token 总量动态分发 —— ≤ 60K 走 ONE-PASS 单次合成,> 60K 委派给 Map-Reduce SubAgent;借助子 Agent 上下文隔离机制,避免主 Agent context 被 Map 阶段 N 倍放大;基于 AWS Region 自动切换 Gemini (Vertex AI) / Doubao,每个 Agent 工具支持独立模型覆盖以平衡成本与质量
- **Temporal 外部服务模式 + K8s 优雅关停**:HTTP 接收 → 立即 202 → 后台异步处理 → Temporal Signal 回传;SIGTERM 后自动摘流并等待存量任务排空再退出
- **Citation 工程**:LLM 内用短 ID 节省 token,Pipeline 出口三级模糊匹配(精确 → 后缀 → 子串)还原 LLM 截断的 ID,并把内部引用 ID 替换为可读文件名,解决文件名特殊字符破坏 XML 标签解析的问题
- **自研评测体系(基于 G-Eval 方法)**:三种合成策略各设一套加权维度,LLM-as-Judge 多次采样取均值降低评分波动,使 Prompt 与 Agent 架构改动可量化对比

#### Plaud Model Hub — 统一 LLM 调用基础设施层（核心贡献者）

Plaud AI Platform 的统一大模型调用层,业务方只需声明「应用 + 逻辑模型」,通过 **Logical Model / Endpoint / Provider 三层抽象**屏蔽供应商差异,内部托管路由策略、容错降级、可观测性。主导以下模块从设计到落地:

- **自研 Adaptive Weight 插件(核心创新)**:常规 Circuit Breaker 一限流就把 endpoint 二值排除,但 Vertex AI GSU + PayGo 双计费场景下这会**瞬间把 100% 流量打到 PayGo、引发成本飙升 + 连锁 429**。设计基于滑动窗口的**渐进降权**机制,按实时 429 率连续调节 Router 权重平滑转移流量,无显式状态机即可自动恢复;与 Rate Limiter / Circuit Breaker 协同形成「渐进降权 / 二值排除 / 完全熔断」三层 endpoint 健康管理体系
- **跨风格无感路由 + Provider 基类抽象**:扩展 OpenAI ↔ Anthropic / GenAI 双向适配器统一图像 / 音频 content block 格式转换,**业务侧只写一份 OpenAI 风格代码即可路由到任意异构 Provider**;同时扩展 OpenAI 兼容 Provider 基类同时支持 Chat 与 Embedding,Azure / Volcengine / DashScope 等供应商只需继承基类即可接入;新增 Provider 接入 Claude on Vertex AI
- **Fallback 链路增强**:支持 fallback 模型**列表**(List 模式)与自动环路检测;响应元数据写入实际命中的 fallback 模型,业务方可审计完整降级路径
- **LangChain Runnable 适配**:把内部调用 Hub 包装成 LangChain Runnable,**上层业务直接用链式 API 写代码,不用换两套调用风格**;补齐三个 Runnable 参数管理 API(配置注入 / 工具绑定 / 实例克隆)对齐协议;响应元数据暴露重试统计与 token usage 让 chain 可追踪成本
- **AppConfig 热更新 + 多格式配置存储**:接入 AWS AppConfig 轮询刷新,**凭证 / 路由策略 / 插件配置全部零停机变更**;扩展存储后端支持 YAML / JSON 双格式与自定义 profile

#### Plaud Summary — AI 录音总结服务（核心业务，除 Mark Note 外全链路参与）

Plaud AI 最核心产品形态。覆盖海外 / CN 多 Region、多语种、十余种业务场景的录音总结链路；底层依赖 Kafka (MSK) 异步消费 + Redshift 数仓 + FastAPI + Langfuse 全链路追踪。

- **主导「结构化 JSON 输出 → Reasoning One-Shot」总结链路重构（最大收益项）**：旧链路用强 schema 收敛输出，但 reasoning 类模型的 output budget 会被 JSON 结构吃掉，长上下文场景推理深度被压缩；改为 Markdown 直出 + 轻量后处理还原结构，配合推理强度精控 + 按用户画像注入的 Persona Prompt，并为十余个业务场景各自定制 One-Shot 模板。新链路在人评维度显著优于旧版，**已替换为线上默认管道**
- **从 0 到 1 搭建「多维总结」(Multi-Dimensional Summary)**：把单一总结拆成 **12 个并行维度**（精华 / 关键数据 / 待办 / 发言人 / 权力动态 / 意图分析 / 概念猎手 / 沟通效率 / 会议效果评价 / 对话金句 等），设计「基类 + 配置注册中心」的可插拔架构，每维度独立配 chunk size / persona / 两阶段 prompt，通过 LangChain 并行分发；并加内容相关性预判链路，在维度无关时主动跳过，避免空白卡片污染体验
- **User-Custom Prompt 污染治理（端到端 case study）**：线上发现用户自定义 prompt 会泄露模型出处声明、注入示例污染输出。设计「正则门控 + 小模型语义复核」**双层方案** —— 正则只判定"是否出现具体模型标识"做廉价兜底，由小模型分辨"归属声明 vs 正文讨论"避免误伤正文中的模型对比内容，仅在归属语句中替换；同时强化 system prompt 的语言覆盖与示例隔离，CN region 单独适配

#### Plaud Ask — RAG 问答

**独立从 0 到 1 交付 Integration Skills**(Gmail + Google Docs + Slack):用户一键发邮件 / 存为 Google Doc,LLM 起草可编辑卡片、审阅后经 MCP 投递。覆盖 API → Handler → Checkpoint → MCP Client 全栈;同时长期负责推荐问题、多语言治理、Embedding 工程化。

- **绕开 LangGraph 又必须与之共存的 Checkpoint 旁路写入**(最棘手设计):卡片走 REST 独立调用、不是 agent 跑的产物,**但又必须进入同一份对话历史让后续 agent 看到**。直接读写 LangGraph 的 PostgresSaver、对齐其内部版本号格式;regenerate 把「作废旧卡 + 追加新 QA 对」**原子合并成单次写入**杜绝中间态;读写前后清理 Redis 缓存防脏读
- **卡片不污染 LLM 上下文**:卡片是 UI artifact,若混入对话历史,**模型会把自己起草的草稿当用户历史发言反向引用**。分离「前端展示历史」与「LLM 推理历史」两通道,卡片只进前者
- **配置驱动 Handler + 双状态机**:配置注册表 + builder 装饰器替代 if/elif,**新增技能只追加配置**;按所需 MCP 工具数自动路由三种执行模式(DB 直读 / 单工具 / 并发 gather);卡片态与投递态分离,部分收件人失败时卡片仍标成功、失败邮箱单独暴露给前端
- **Email 链 + MCP Client**:`转写 → Markdown → PDF → 投递` 四段链,PDF 失败非阻断;笔记内 S3 key 在传给 MCP 前替换为 presigned URL,解决**远端 LLM 无法读图**;MCP Client 把 JSON-RPC 错误码分类映射,OAuth 未授权在 API 层转 200 + 授权 URL 而非 500;1500+ 行单测覆盖 regenerate 原子性、OAuth fallback、部分投递失败等边界

---

### AI Startup｜ LLM 应用 *(2024.04 – 2024.08)*

#### 企业知识库检索与问答系统 (RAG)

- 主导从零到一搭建完整 RAG 应用，为 B 端企业提供内部知识库解决方案
- 扩展开源框架支持多格式文档（doc、扫描版 PDF、图片）；自研版面排序算法优化复杂版面分析与 Markdown 转换
- 落地查询转换、混合搜索、Graph 增强等策略；用 Ragas 量化评估迭代配置；

#### 智能访谈报告分析系统

- 设计多阶段、多 Agent 系统，实现 LLM 稳定生成 JSON Schema 输出访谈报告
- Prompt Tuning 微调 BERT 替代 LLM 做情感分析，显著降低延时；并行实现传统 RAG 与 GraphRAG 双方案

---

### 微软（中国）｜ 算法研究实习生（Excel Copilot） *(2024.01 – 2024.02)*

- 主导 **LLM4Causality** 研究：梳理学术文献构建基准因果图，设计元数据解析系统提升预处理效率
- 设计专业推理 Prompts，量化评估证明 LLM 在识别复杂因果连接上的优越性
- 用 Excel Lambda 实现 ML 概念验证（One-Hot、线性 / 岭回归），验证 Excel 内做高级数据分析的可行性

---

### 腾讯（深圳）｜ 前端开发工程师（腾讯文档） *(2020.06 – 2022.06)*

- **域代码格式标准化**：制定数据格式规范，重构数据 / 排版 / 业务三层架构，100% 兼容 OOXML；首次在排版引擎实现域代码自动断行（行业领先）
- **腾讯文档 Drive 研发**：设计四层架构（视图 / 控制 / 状态 / 网络），打通移动端本地文件云端导入、存储与在线编辑
- **研发效能建设**：封装 Word 编辑引擎 API、升级团队自动化测试框架，从零编写性能测试用例（首屏加载、大文档、协同编辑）；团队荣获**公司研发效能奖**

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

### text-to-diagrams ｜ 把纯文本一键变成「带 SVG 配图」的 Markdown *(2026.05)*

Python 库 + [Claude Skill plugin](https://github.com/pillarliang/productivity-toolkit/tree/main/plugins/text-to-diagrams) + 自带 FastAPI Web 服务(Docker 一键起),覆盖 17 种图类型。

- **核心判断 —— 别让 LLM 算坐标**。17 种图里 10 种有公式或确定性布局算法 → 纯 Python 渲染,毫秒级、零 LLM、布局保证不重叠;只剩 7 种自由拓扑的(架构 / 时序 / ER 等)才走 LLM 改 SVG 模板,**LLM 的价值在"理解 + 摆放"而非数学**
- **两条防幻觉护栏**: 规划阶段输出的锚点 ID 立即对照大纲校验、编造即反馈重生成,插入位置不会写到不存在的章节; LLM 输出 SVG 后纯 Python 几何校验(< 1ms)检查越界 / 重叠 / 网格 / 焦点,**仅有问题时给具体修复建议重试 1 次,干净 case 零额外 token**(整体成本仅 +30%、非双倍)
- **可插拔的第二产物形态(Summary Infographic)**:把"结论先行型"文本(会议纪要 / 面试评估 / 病例)抽成 8 种结构化信息原语,按"注意力金字塔"纯 Python 拼成单文件 HTML 信息图、零渲染期 LLM;两种产物通过 API `mode` 字段共存,**删整个子包即下线、主流程一行不变**

