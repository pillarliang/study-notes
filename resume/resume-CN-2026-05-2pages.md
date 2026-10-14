# 梁 柱

13485492580 | [pillarliang21@gmail.com](mailto:pillarliang21@gmail.com) | [github.com/pillarliang](https://github.com/pillarliang) | Blog: [pillarliang.github.io](https://pillarliang.github.io/)

---

## 教育背景

| 学校 | 专业 / 学位 | 时间 |
| --- | --- | --- |
| **新加坡国立大学（NUS）** | 人工智能系统 · 硕士 | 2023.08 – 2024.10 |
| **太原理工大学** | 软件工程 · 本科 | 2016.09 – 2020.07 |

---

## 工作经历

### 阿里云（北京）科技有限公司 | GenAI SA *(2026.08 – 至今)*

负责百炼国际站（Model Studio）的模型评测、客户方案验证与技术支持，参与 Qwen 商业授权合作。

- **模型评测与选型**：牵头文本模型评测方案设计，开发 API 性能评测工具，支持多模型对比、延迟与输出速度分析及可视化报告。
- **客户技术验证**：针对 Maxinsight 视频标注需求，完成 **40 组模型与推理配置评测**，分析效果、用量与性能；开发图像及具身数据标注 Demo，支持客户验证与销售演示。
- **海外授权合作**：参与 AWS、Together AI、Nebius、Modal 等伙伴的 Qwen 商业授权推进，梳理模型范围、部署支持与收费方案，形成合作决策材料。
- **模型接入与排障**：建设文本、多模态和语音调用样例；排查客户视频生成限流问题，提出配额调整与重试策略建议。

---

### Plaud AI | LLM 应用算法工程师 *(2024.11 – 2026.08)*

负责录音总结、RAG 问答、项目级摘要及 MCP / Agent 集成，覆盖底层模型调用基础设施与上层业务编排。

#### Model Hub · 统一 LLM 调用基础设施（核心贡献者）

- **统一接入与路由**：支撑全业务线 **日调用量 50 万次**；设计统一调用、同风格透传与跨厂商协议适配，接入 OpenAI、Anthropic、Gemini、DashScope、Bedrock 等模型服务。
- **调度与成本优化**：按 429 错误率渐进降权，结合权重重分配、探测流量和限速恢复，实现预付费与按量付费流量平滑切换，**单月节省成本 $1,200+**。
- **可靠性与可观测性**：构建重试、Endpoint 切换、模型降级三级容错，结合熔断与限流形成健康管理闭环；接入 Langfuse 追踪与 AppConfig 配置热加载。

#### Project Summary · Multi-Agent 项目级摘要（独立负责）

- **从 0 到 1 交付上线**：聚合多份录音与文档，支持总结、对比和进度跟踪；基于 DeepAgents + LangChain 编排文件分析、大纲生成、RAG 检索、报告合成与审阅流程。
- **长文本与并发治理**：按 token 量路由单次生成或 map-reduce；通过子 Agent 上下文隔离、文件级抽取与章节级合并控制上下文并去重，以 reducer 合并并发状态。
- **交付质量与稳定性**：接入 Temporal 异步回传及 K8s 优雅关停；实现确定性引用溯源，构建分策略、加权多采样的 G-Eval 评测体系。

<!-- pagebreak -->

#### Summary · AI 录音总结（核心业务）

- **总结链路重构**：以 Markdown 直出与 Reasoning One-Shot 替代 JSON 强约束，结合长文本 map-reduce 上线默认管道；通过六维 G-Eval 评测迭代，**评测分由 30% 提升至 70+%**。
- **专业场景优化**：为近 **30 个专业场景**构建 Framework → Sections → Polish 三步管道，支持分节并行、分步模型选择、降级与质量重试，**评测分进一步提升至 80+%**。
- **多模态与工程化**：交付 LLM 生成 HTML → Playwright 渲染 → S3 归档的总结头图链路，支持多语言字体与 CN 区域 AIGC 合规；参与 Prompt 污染治理、向量模板推荐及个性化总结。

#### Ask · RAG 问答与 MCP / Agent 集成

- **独立交付 Integration Skills**：从 0 到 1 接入 Gmail、Google Docs、Slack，打通“LLM 起草可编辑卡片 → 用户审阅 → MCP 投递”链路，覆盖 API、Handler、Checkpoint 与 MCP Client。
- **状态与上下文设计**：分离卡片展示与模型推理上下文；以配置驱动 Handler 扩展，拆分卡片态与投递态，处理部分失败及重试场景。
- **端到端集成**：实现转写、Markdown、PDF 与邮件投递，支持 PDF 失败降级及私有图片多模态输入；编写 **1,500+ 行单测**覆盖原子性、OAuth 与部分失败，并参与推荐问题、多语言和 Embedding 工程化。

---

### 腾讯（深圳）| 前端开发工程师（腾讯文档） *(2020.06 – 2022.06)*

- **文档引擎与云端集成**：重构域代码数据、排版与业务架构，兼容 OOXML 并实现域代码自动断行；设计腾讯文档 Drive 分层架构，打通移动端文件云端导入、存储与在线编辑。
- **研发效能**：封装 Word 引擎 API、升级自动化测试框架，建设首屏、大文档与协同性能测试；团队获公司研发效能奖。

---

## 专业技能

| 类别 | 技术栈 |
| --- | --- |
| **语言与后端** | Python、TypeScript / JavaScript、SQL、C#；FastAPI、Flask |
| **LLM / Agent** | LangChain、LangGraph、DeepAgents、LlamaIndex、MCP；GPT、Claude、Gemini、通义 |
| **评测与可观测性** | Langfuse、Ragas、G-Eval、OpenTelemetry |
| **数据与基础设施** | Milvus、pgvector、OpenSearch、Redis、MySQL、S3；Docker、K8s、Temporal、Kafka、AWS AppConfig |

---

## 开源 & 个人项目

### text-to-diagrams | 文本生成 SVG 配图 *(2026.05)*

- 开发 Python 库、[Claude Skill plugin](https://github.com/pillarliang/productivity-toolkit/tree/main/plugins/text-to-diagrams) 与 FastAPI 服务，覆盖 **17 种图类型**；将 LLM 语义布局与 Python 确定性坐标计算分离，结合锚点校验、几何检查与重生成控制错误，支持 SVG 与信息图输出。

### 企业知识库检索与问答系统（RAG） *(2024.04 – 2024.08)*

- 支持文档、扫描 PDF 与图片解析，优化复杂版面排序和 Markdown 转换；实践查询转换、混合检索与 Graph 增强，使用 Ragas 量化评估。
