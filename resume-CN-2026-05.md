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

负责 Plaud AI 背后的 AI 服务,覆盖 **录音总结、RAG 问答、项目级摘要、MCP / Agent 集成** 四条业务线,纵向贯穿「Model Hub 底层路由 / 容错 / 可观测性 → 上层 Multi-Agent 业务编排」全栈。

#### Plaud Project Summary — Multi-Agent 项目级摘要服务（独立负责,端到端）

**从 0 到 1 独立交付并上线**:聚合多份录音与文档产出结构化项目报告,支持总结 / 对比 / 进度跟踪三种策略。

- **Agent 架构 (DeepAgents + LangChain)**:按 token 量分发 ONE-PASS / Map-Reduce,子 Agent 隔离主 context
- **Temporal 外部服务 + K8s 优雅关停**:HTTP 即返 202、Signal 异步回传;SIGTERM 摘流后排空再退出
- **Citation 工程**:LLM 内短 ID 省 token,出口三级模糊匹配还原并映射回文件名
- **自研评测体系 (G-Eval)**:三种策略各配加权维度,LLM-as-Judge 多采样让改动可量化对比

#### Plaud Model Hub — 统一 LLM 调用基础设施层（核心贡献者）

Plaud AI Platform 的统一大模型调用层,通过 **Logical Model / Endpoint / Provider 三层抽象** 屏蔽供应商差异,内部托管路由 / 容错 / 可观测性。

- **自研 Adaptive Weight 插件 (核心创新)**:滑动窗口**渐进降权**替代二值切流,按 429 率连续调权,解决 GSU+PayGo 下成本飙升 + 连锁 429
- **跨风格路由 + Provider 基类**:OpenAI ↔ Anthropic / GenAI 双向适配,一份代码路由异构 Provider;基类支持 Chat / Embedding,新增 Claude on Vertex AI
- **Fallback 增强**:模型列表 + 环路检测,元数据回写命中模型,降级路径可审计
- **LangChain Runnable 适配**:Hub 包成 Runnable,业务直用链式 API;补齐三个参数管理 API,元数据可追成本
- **AppConfig 热更新 + 多格式存储**:凭证 / 路由 / 插件零停机变更;后端支持 YAML / JSON

#### Plaud Summary — AI 录音总结服务（核心业务,除 Mark Note 外全链路参与）

Plaud AI 最核心产品形态。覆盖海外 / CN 多 Region、多语种、十余种业务场景;底层依赖 Kafka (MSK) 异步消费 + Redshift + FastAPI + Langfuse 全链路追踪。

- **主导「JSON → Reasoning One-Shot」总结链路重构（最大收益项）**:Markdown 直出释放被强 schema 吃掉的 reasoning output budget;配合推理强度精控、用户画像 Persona、十余场景 One-Shot 模板,**已上线为默认管道**
- **从 0 到 1 搭建「多维总结」**:拆成 12 个并行维度,「基类 + 注册中心」可插拔架构,每维度独立 chunk size / persona / prompt,相关性预判跳空白维度
- **User-Custom Prompt 污染治理**:针对 prompt 泄露模型出处、注入示例,「正则门控 + 小模型复核」双层方案,分辨「归属 vs 正文」避免误伤

#### Plaud Ask — RAG 问答

**独立从 0 到 1 交付 Integration Skills**(Gmail + Google Docs + Slack):LLM 起草可编辑卡片、用户审阅后经 MCP 投递;覆盖 API → Handler → Checkpoint → MCP Client 全栈;同时长期负责推荐问题、多语言治理、Embedding 工程化。

- **Checkpoint 旁路写入 (最棘手设计)**:卡片非 agent 产物却须进对话历史让 agent 可见,直接读写 LangGraph PostgresSaver 对齐版本号;regenerate「作废旧卡 + 追加新 QA」原子合并
- **卡片不污染 LLM 上下文**:分「展示」与「LLM 推理」两通道,避免模型把自己草稿当用户发言反向引用
- **配置驱动 Handler + 双状态机**:注册表 + builder 替代 if/elif,新技能只追加配置;按 MCP 工具数路由三模式;卡片态 / 投递态分离,部分失败仍标成功
- **Email 链 + MCP Client**:`转写 → Markdown → PDF → 投递` PDF 失败非阻断;S3 key 换 presigned URL 让远端 LLM 读图;1500+ 行单测覆盖原子性 / OAuth / 部分失败

---

### AI Startup｜ LLM 应用 *(2024.04 – 2024.08)*

#### 企业知识库检索与问答系统 (RAG)

- 主导从 0 到 1 搭建 RAG 应用,为 B 端企业提供知识库解决方案
- 扩展开源框架支持多格式文档(doc / 扫描 PDF / 图片);自研版面排序算法优化版面分析与 Markdown 转换
- 落地查询转换、混合搜索、Graph 增强等策略,Ragas 量化评估迭代

#### 智能访谈报告分析系统

- 设计多阶段、多 Agent 系统让 LLM 稳定输出 JSON Schema 访谈报告
- Prompt Tuning 微调 BERT 替代 LLM 做情感分析降延时;并行实现传统 RAG / GraphRAG 双方案

---

### 微软（中国）｜ 算法研究实习生（Excel Copilot） *(2024.01 – 2024.02)*

- 主导 **LLM4Causality** 研究:梳理文献构建基准因果图,设计元数据解析提升预处理效率
- 设计专业推理 Prompts,量化评估 LLM 在复杂因果识别上的优越性
- 用 Excel Lambda 实现 ML 概念验证(One-Hot / 线性 / 岭回归),证明 Excel 做高级分析可行

---

### 腾讯（深圳）｜ 前端开发工程师（腾讯文档） *(2020.06 – 2022.06)*

- **域代码格式标准化**:制定数据规范,重构数据 / 排版 / 业务三层架构 100% 兼容 OOXML;首次实现排版引擎域代码自动断行(行业领先)
- **腾讯文档 Drive 研发**:设计视图 / 控制 / 状态 / 网络四层架构,打通移动端文件云端导入 / 存储 / 在线编辑
- **研发效能建设**:封装 Word 引擎 API、升级团队自动化测试框架,从 0 编写性能测试用例(首屏 / 大文档 / 协同);团队荣获**公司研发效能奖**

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

Python 库 + [Claude Skill plugin](https://github.com/pillarliang/productivity-toolkit/tree/main/plugins/text-to-diagrams) + FastAPI 服务(Docker 一键起),覆盖 17 种图类型。

- **核心判断:别让 LLM 算坐标**。10 种公式 / 确定性布局走纯 Python,7 种自由拓扑走 LLM 改 SVG,**LLM 价值在「理解 + 摆放」而非数学**
- **两条防幻觉护栏**:锚点 ID 对照大纲校验、编造则重生成;SVG 输出纯 Python 几何校验,有问题才反馈重试,整体成本 +30%
- **可插拔第二产物形态 (Summary Infographic)**:结论先行文本抽成 8 种原语,按「注意力金字塔」纯 Python 拼成 HTML 信息图;API `mode` 共存,**删子包即下线**