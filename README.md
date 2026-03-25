# Study Notes

个人学习笔记仓库，涵盖 AI 工程、基础设施、开发工具与工作项目。

## 目录结构

```text
study-notes/
├── AI/
│   ├── agents/          AI Agent 相关（MCP、ADK、框架笔记）
│   └── ai-engineering/  AI Engineering 书籍笔记（ch01–ch10）
├── infra/               基础设施（Docker、K8s、部署、可观测性）
├── tools/               开发工具（Claude Code 等）
├── work/                工作项目（OpenClaw、Model Hub、Ask）
└── misc/                杂项
```

---

## AI / Agents — AI Agent 相关

| 笔记 | 关键词 |
| ------ | -------- |
| [MCP 学习笔记](AI/agents/MCP_学习笔记.md) | Model Context Protocol、工具调用、MCP Server |
| [ADK Skill 设计模式](AI/agents/adk-skill-design-patterns-学习笔记.md) | Agent SDK、Skill 设计、工具封装 |
| [create_agent 开发指南](AI/agents/create_agent%20开发指南.md) | Agent 创建、框架配置 |
| [DeepAgents 学习笔记](AI/agents/deepagents%20学习笔记.md) | Multi-agent、任务分解 |
| [AI Agent 运作原理 — 以 OpenClaw 为例](AI/agents/AI%20Agent%20运作原理%20—%20以%20OpenClaw%20为例.md) | Agent 工作流、工具调用链路 |
| [解剖小龍蝦 — OpenClaw 运作原理（讲座整理）](AI/agents/解剖小龍蝦%20—%20以%20OpenClaw%20為例介紹%20AI%20Agent%20的運作原理-transcript-clean.md) | 讲座文字稿、OpenClaw 原理 |

---

## AI / AI Engineering — 书籍笔记

基于 [AI Engineering](https://www.oreilly.com/library/view/ai-engineering/9781098166298/) (Chip Huyen) 整理，融入个人理解与补充。

### Chapter 1 · AI Engineering Introduction — AI 工程导论

从零认识 AI 工程的全景图。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 1.1 | [The Rise of AI Engineering](AI/ai-engineering/ch01-ai-engineering/1.1-The_Rise_of_AI_Engineering.md) | AI 工程的兴起、行业趋势 |
| 1.2 | [Foundation Model Use Cases](AI/ai-engineering/ch01-ai-engineering/1.2-Foundation_Model_Use_Cases.md) | 基础模型应用场景 |
| 1.3 | [Planning AI Applications](AI/ai-engineering/ch01-ai-engineering/1.3-Planning_AI_Applications.md) | AI 应用规划、产品思维 |
| 1.4 | [AI Engineering Stack](AI/ai-engineering/ch01-ai-engineering/1.4-AI_Engineering_Stack.md) | AI 工程技术栈 |

### Chapter 2 · Foundation Models — 基础模型

从 token 到 text，理解模型如何"思考"。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 2.0 | [Understanding Foundation Models and Training Data](AI/ai-engineering/ch02-foundation-models/2.0-Understanding_Foundation_Models_and_Training_Data.md) | 基础模型概述、训练数据 |
| 2.1 | [Modeling Architecture and Size](AI/ai-engineering/ch02-foundation-models/2.1-Modeling_Architecture_and_Size.md) | Transformer 架构、模型规模 |
| 2.2 | [Post Training and Sampling](AI/ai-engineering/ch02-foundation-models/2.2-Post_Training_and_Sampling.md) | SFT、RLHF、后训练流程 |
| 2.3 | [Sampling](AI/ai-engineering/ch02-foundation-models/2.3-Sampling.md) | Token 采样、Temperature、Top-k / Top-p |

### Chapter 3 · Evaluation Methodology — 评估方法论

不能度量，就不能改进。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 3.1 | [Challenges of Evaluating Foundation Models](AI/ai-engineering/ch03-evaluation-methodology/3.1-Challenges_of_Evaluating_Foundation_Models.md) | 通用性、涌现性、黑箱问题 |
| 3.2 | [Understanding Language Modeling Metrics](AI/ai-engineering/ch03-evaluation-methodology/3.2-Understanding_Language_Modeling_Metrics.md) | Cross Entropy、Perplexity、BPC、BPB |
| 3.3 | [Exact Evaluation](AI/ai-engineering/ch03-evaluation-methodology/3.3-Exact_Evaluation.md) | 客观评估 vs 主观评估、可复现性 |
| 3.4 | [AI as a Judge](AI/ai-engineering/ch03-evaluation-methodology/3.4-AI_as_a_Judge.md) | 自动化评估、LLM-as-Judge |
| 3.5 | [Ranking Models with Comparative Evaluation](AI/ai-engineering/ch03-evaluation-methodology/3.5-Ranking_Models_with_Comparative_Evaluation.md) | Pointwise vs Pairwise、Elo Rating |

### Chapter 4 · Evaluate AI Systems — 评估 AI 系统

构建系统化的评估流水线。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 4.1 | [Evaluation Criteria](AI/ai-engineering/ch04-evaluate-ai-systems/4.1-Evaluation_Criteria.md) | 评估驱动开发、质量维度定义 |
| 4.2 | [Model Selection](AI/ai-engineering/ch04-evaluate-ai-systems/4.2-Model_Selection.md) | 模型对比、选型策略 |
| 4.3 | [Design Your Evaluation Pipeline](AI/ai-engineering/ch04-evaluate-ai-systems/4.3-Design_Your_Evaluation_Pipeline.md) | 评估流水线设计 |

### Chapter 5 · Prompt Engineering — 提示工程

与模型对话是一门手艺。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 5.1 | [Introduction to Prompting](AI/ai-engineering/ch05-prompt-engineering/5.1-Introduction_to_Prompting.md) | Prompt 结构、In-Context Learning、System Prompt |
| 5.2 | [Prompt Engineering Best Practices](AI/ai-engineering/ch05-prompt-engineering/5.2-Prompt_Engineering_Best_Practices.md) | 跨模型通用原则、生产级实践 |
| 5.3 | [Defensive Prompt Engineering](AI/ai-engineering/ch05-prompt-engineering/5.3-Defensive_Prompt_Engineering.md) | Prompt 注入、越狱攻击、防御策略 |

### Chapter 6 · RAG, Agents & Memory — RAG、智能体与记忆

让模型连接外部世界。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 6.1 | [RAG](AI/ai-engineering/ch06-rag-agents-memory/6.1-RAG.md) | 外部知识检索、Context 构建、减少幻觉 |
| 6.2 | [Agents](AI/ai-engineering/ch06-rag-agents-memory/6.2-Agents.md) | 工具调用、规划、多步推理 |
| 6.3 | [Memory](AI/ai-engineering/ch06-rag-agents-memory/6.3-Memory.md) | 短期 / 长期记忆、上下文管理 |

### Chapter 7 · Finetuning — 微调

让模型更懂你的领域。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 7.0 | [Finetuning Overview](AI/ai-engineering/ch07-finetuning/7.0-Finetuning_Overview.md) | 微调概述、适用场景 |
| 7.1 | [When to Finetune](AI/ai-engineering/ch07-finetuning/7.1-When_to_Finetune.md) | 微调时机、成本收益分析 |
| 7.2 | [Memory Bottlenecks](AI/ai-engineering/ch07-finetuning/7.2-Memory_Bottlenecks.md) | 显存瓶颈、梯度检查点 |
| 7.3 | [PEFT and LoRA](AI/ai-engineering/ch07-finetuning/7.3-PEFT_and_LoRA.md) | 参数高效微调、低秩适应 |
| 7.4 | [Model Merging](AI/ai-engineering/ch07-finetuning/7.4-Model_Merging.md) | 模型合并、权重融合 |
| 7.5 | [Finetuning Tactics](AI/ai-engineering/ch07-finetuning/7.5-Finetuning_Tactics.md) | 微调实战技巧 |
| 7.6 | [Summary](AI/ai-engineering/ch07-finetuning/7.6-Summary.md) | 微调总结 |

### Chapter 8 · Dataset Engineering — 数据集工程

数据决定模型上限。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 8.0 | [Dataset Engineering Overview and Data Curation](AI/ai-engineering/ch08-dataset-engineering/8.0-Dataset_Engineering_Overview_and_Data_Curation.md) | 数据工程概述、数据整理 |
| 8.1 | [Data Augmentation and Synthesis](AI/ai-engineering/ch08-dataset-engineering/8.1-Data_Augmentation_and_Synthesis.md) | 数据增强、合成数据 |
| 8.2 | [Data Processing and Summary](AI/ai-engineering/ch08-dataset-engineering/8.2-Data_Processing_and_Summary.md) | 数据处理流程 |

### Chapter 9 · Inference Optimization — 推理优化

让模型跑得更快更省。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 9.0 | [Inference Optimization Overview](AI/ai-engineering/ch09-inference-optimization/9.0-Inference_Optimization_Overview.md) | 推理优化概述、延迟与吞吐 |
| 9.1 | [Inference Optimization Techniques](AI/ai-engineering/ch09-inference-optimization/9.1-Inference_Optimization_Techniques.md) | 量化、KV Cache、投机解码 |

### Chapter 10 · AI Engineering Architecture — AI 工程架构

从原型到生产的最后一公里。

| # | 笔记 | 关键词 |
| --- | ------ | -------- |
| 10.0 | [AI Engineering Architecture](AI/ai-engineering/ch10-ai-engineering-architecture/10.0-AI_Engineering_Architecture.md) | 系统架构、部署模式 |
| 10.1 | [User Feedback](AI/ai-engineering/ch10-ai-engineering-architecture/10.1-User_Feedback.md) | 用户反馈、持续改进 |

### Supplementary · 补充笔记

| 笔记 | 关键词 |
| ------ | -------- |
| [Self-Attention 自注意力机制](AI/ai-engineering/supplementary/self-attention-自注意力机制.md) | Query / Key / Value、序列依赖建模 |
| [Sampling 采样与推理详解](AI/ai-engineering/supplementary/Sampling-采样与推理详解.md) | Token 采样、Temperature、Top-k / Top-p |
| [Post-Training 后训练详解](AI/ai-engineering/supplementary/Post-Training-后训练详解.md) | SFT、偏好对齐、RLHF |
| [Transformer 架构全景解析](AI/ai-engineering/supplementary/Transformer-架构全景解析.md) | Transformer 结构、注意力机制 |

---

## Infra — 基础设施

| 笔记 | 关键词 |
| ------ | -------- |
| [Docker 学习笔记](infra/docker-learning.md) | 容器化、镜像、Dockerfile |
| [Vertex AI Provisioned Throughput](infra/vertex-ai-provisioned-throughput-学习笔记.md) | GCP、吞吐量预留、成本控制 |
| [Static Code Analysis Guide](infra/static-code-analysis-guide.md) | 静态分析、代码质量 |
| [Temporal 学习笔记](infra/temporal_notes.md) | 工作流引擎、任务编排 |
| [Helm & EKS 学习笔记](infra/deploy/helm-eks-学习笔记.md) | Helm Chart、AWS EKS、K8s 部署 |
| [Kubernetes Pod 优雅关闭](infra/deploy/kubernetes-pod-graceful-shutdown.md) | Graceful Shutdown、信号处理 |
| [OpenTelemetry Guide](infra/observability/opentelemetry-guide.md) | 可观测性、Trace、Metrics、Logs |

---

## Tools — 开发工具

| 笔记 | 关键词 |
| ------ | -------- |
| [Claude Code 使用文档](tools/Claude_Code_使用文档.md) | Claude Code CLI、配置、使用技巧 |

---

## Work — 工作项目

| 笔记 | 关键词 |
| ------ | -------- |
| [OpenClaw 笔记](work/openclaw/openclaw-notes.md) | OpenClaw 系统、架构设计 |
| [Single File Ask Flow](work/ask/single_file_ask_flow.md) | Ask 功能流程设计 |
| [Plaud Model Hub 架构学习笔记](work/model-hub/Plaud%20Model%20Hub%20架构学习笔记.md) | Model Hub 架构、模型管理 |
| [Model Hub Config Guide](work/model-hub/model-hub-config-guide.md) | 配置指南 |

---

## License

本仓库内容仅供个人学习参考。
