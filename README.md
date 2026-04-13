# Study Notes

个人学习笔记仓库，涵盖 AI 工程、Harness 工程、基础设施、开发工具与工作项目。

## 目录结构

```text
study-notes/
├── AI/
│   ├── agents/              AI Agent 相关（MCP、ADK、框架笔记）
│   ├── ai-engineering/      AI Engineering 书籍笔记（ch01–ch10 + 补充）
│   └── harness-engineering/ Harness Engineering 笔记（Claude Code 工程深度解读）
├── infra/                   基础设施（Docker、K8s、部署、可观测性）
├── tools/                   开发工具（Claude Code 等）
├── work/                    工作项目（OpenClaw、Model Hub、Ask）
└── misc/                    杂项（语言标识等）
```

---

## AI / Agents — AI Agent 相关

| 笔记 | 关键词 | 摘要 |
| ------ | -------- | ------ |
| [MCP 学习笔记](AI/agents/MCP_学习笔记.md) | Model Context Protocol、工具调用、MCP Server | MCP 协议的架构设计与生态定位，定义了 LLM 应用与外部系统（工具、数据源、服务）之间的统一接口标准。笔记梳理了 Server/Client 交互流程、工具声明规范以及在 Agent 框架中的集成方式。 |
| [Agent Skill 设计模式](AI/agents/agent_skills_notes.md) | Agent Skills 规范、五大设计模式、Skill 安装与分发 | Agent Skills（agentskills.io 开放规范）的五大设计模式总结，厘清 Tool（原子操作）与 Skill（可组合能力单元）的本质区别。涵盖 SKILL.md 规范、渐进式加载机制、`npx skills` CLI 与 Claude Code Plugin 安装方式，以及 Skill 的跨平台分发策略。 |
| [create_agent 开发指南](AI/agents/create_agent%20开发指南.md) | Agent 创建、框架配置 | Agent 开发的端到端指南，从框架选型、工具定义、状态管理到部署上线的完整流程。包含配置模板、常见陷阱与调试技巧。 |
| [DeepAgents 学习笔记](AI/agents/deepagents%20学习笔记.md) | Multi-agent、任务分解 | DeepAgents 框架的多层次 Agent 协作模式，探讨如何将复杂任务分解为子任务并分配给专精 Agent。涵盖深度推理策略、Agent 间通信机制与任务编排设计。 |
| [AI Agent 运作原理 — 以 OpenClaw 为例](AI/agents/AI%20Agent%20运作原理%20—%20以%20OpenClaw%20为例.md) | Agent 工作流、工具调用链路 | 以 OpenClaw 为实例拆解 Agent 的完整运行机制，包括查询循环（Query Loop）、工具调用链路、记忆管理与多轮推理的协调方式。帮助理解 Agent 从接收指令到产出结果的全路径。 |
| [Agent 工具设计的第三代演进](AI/agents/agent_advanced_tools.md) | 动态工具发现、代码编排、结构化输出 | Agent 工具设计从第一代 API 封装、第二代 ACI 接口到第三代 Advanced Tool Use 的演进脉络。重点讲解 Tool Search（动态发现）、Programmatic Tool Calling（代码编排）和 Structured Output 三大方向，附带准确率与 Token 成本的量化收益。 |

---

## AI / Harness Engineering — Claude Code 工程深度解读

基于《Harness Engineering — 以 Claude Code 为样本》整理，探讨如何将不稳定的 AI 模型收束进可持续运行的工程秩序。

| 笔记 | 关键词 | 摘要 |
| ------ | -------- | ------ |
| [序言 & 第 1-2 章](AI/harness-engineering/book1-claude-code-c1-c2-学习笔记.md) | Harness 定义、控制面、可靠性原则 | 定义 Harness Engineering 的核心命题：不是模型能不能写代码，而是模型进入终端和团队流程后如何不把系统带偏。梳理 AI Coding Agent 的六大可靠性原则与控制面架构，阐明约束优先于能力的工程哲学。 |
| [第 3-4 章](AI/harness-engineering/book1-claude-code-c3-c4-学习笔记.md) | Query Loop、状态管理、生命周期 | Agent 主循环（Query Loop）的运行时结构与状态管理机制。分析 Agent 系统的生命周期阶段、可变状态的隔离策略，以及跨轮迭代中如何保持上下文一致性与行为可预测性。 |
| [第 5-6 章](AI/harness-engineering/book1-claude-code-c5-c6-学习笔记.md) | CLAUDE.md、上下文治理、内存预算 | 上下文治理作为 Agent 系统主路径的设计理念。详解 CLAUDE.md 的三层分级机制（全局/项目/目录）、内存预算制度的容量规划，以及信息从注入到淘汰的完整生命周期管理。 |
| [第 7-9 章](AI/harness-engineering/book1-claude-code-c7-c9-学习笔记.md) | 多 Agent、Forked Agent、Cache、团队制度 | 多 Agent 架构中的职责分区与验证机制。涵盖 Forked Agent 的隔离执行模型、KV Cache 的命中率优化策略、角色分离（执行者/审查者）的设计，以及如何将工程结构固化为团队可复用的制度。 |

---

## AI / AI Engineering — 书籍笔记

基于 [AI Engineering](https://www.oreilly.com/library/view/ai-engineering/9781098166298/) (Chip Huyen) 整理，融入个人理解与补充。

### Chapter 1 · AI Engineering Introduction — AI 工程导论

从零认识 AI 工程的全景图。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 1.1 | [The Rise of AI Engineering](AI/ai-engineering/ch01-ai-engineering/1.1-The_Rise_of_AI_Engineering.md) | AI 工程的兴起、行业趋势 | 回顾从传统语言模型到大语言模型的演化路径，阐述规模化带来的能力跃迁（涌现能力）与"模型即服务"的范式转变。讨论 AI 工程作为新兴学科的定位及其与传统 ML 工程的分野。 |
| 1.2 | [Foundation Model Use Cases](AI/ai-engineering/ch01-ai-engineering/1.2-Foundation_Model_Use_Cases.md) | 基础模型应用场景 | 系统梳理基础模型的八大应用场景分类，从文本生成、代码辅助到多模态理解。分析企业端与消费者端的差异化需求，以及不同行业在 AI 采用中面临的机遇与阻力。 |
| 1.3 | [Planning AI Applications](AI/ai-engineering/ch01-ai-engineering/1.3-Planning_AI_Applications.md) | AI 应用规划、产品思维 | 构建 AI 应用的规划框架，包括如何评估用例可行性、定义 AI 与人类的角色分工、设定可量化的成功指标。强调分阶段交付策略，从 PoC 到生产逐步验证假设。 |
| 1.4 | [AI Engineering Stack](AI/ai-engineering/ch01-ai-engineering/1.4-AI_Engineering_Stack.md) | AI 工程技术栈 | 拆解 AI 工程技术栈的分层结构，阐明 AI 工程相比传统 ML 工程的三大核心差异：模型作为外部服务、Prompt 作为编程接口、评估驱动的迭代方式。介绍模型适配方法论的选择路径。 |

### Chapter 2 · Foundation Models — 基础模型

从 token 到 text，理解模型如何"思考"。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 2.0 | [Understanding Foundation Models and Training Data](AI/ai-engineering/ch02-foundation-models/2.0-Understanding_Foundation_Models_and_Training_Data.md) | 基础模型概述、训练数据 | 介绍基础模型的三大设计维度——训练数据（规模与质量）、模型架构（大小与结构）、后训练（对齐与微调）。分析每个维度如何影响下游应用的能力边界与行为特征。 |
| 2.1 | [Modeling Architecture and Size](AI/ai-engineering/ch02-foundation-models/2.1-Modeling_Architecture_and_Size.md) | Transformer 架构、模型规模 | 梳理模型架构从 Seq2Seq 到 Transformer 的演化历程，重点分析 Encoder-only、Decoder-only、Encoder-Decoder 三种变体的适用场景。讨论参数规模对模型性能、推理延迟和部署成本的影响。 |
| 2.2 | [Post Training and Sampling](AI/ai-engineering/ch02-foundation-models/2.2-Post_Training_and_Sampling.md) | SFT、RLHF、后训练流程 | 后训练阶段的两大核心环节：监督微调（SFT）使模型学会遵循指令，偏好微调（RLHF/DPO）使模型输出对齐人类偏好。同时介绍采样策略如何在推理时控制输出的质量与多样性。 |
| 2.3 | [Sampling](AI/ai-engineering/ch02-foundation-models/2.3-Sampling.md) | Token 采样、Temperature、Top-k / Top-p | 深入讲解 Token 级别的采样机制，从 logits 到概率分布的转换过程。详细分析 Temperature、Top-K、Top-P（核采样）等参数如何影响输出的创造性与一致性，以及不同场景下的参数调优建议。 |

### Chapter 3 · Evaluation Methodology — 评估方法论

不能度量，就不能改进。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 3.1 | [Challenges of Evaluating Foundation Models](AI/ai-engineering/ch03-evaluation-methodology/3.1-Challenges_of_Evaluating_Foundation_Models.md) | 通用性、涌现性、黑箱问题 | 剖析基础模型评估面临的结构性困难：智能与评估能力的不对称、开放式输出难以标准化、涌现能力无法提前预测。讨论为何传统 ML 评估范式在 LLM 时代需要根本性重构。 |
| 3.2 | [Understanding Language Modeling Metrics](AI/ai-engineering/ch03-evaluation-methodology/3.2-Understanding_Language_Modeling_Metrics.md) | Cross Entropy、Perplexity、BPC、BPB | 从信息论角度推导语言建模的四大核心指标——熵、交叉熵、困惑度（Perplexity）、每字符比特数（BPC）。阐明它们之间的数学关系及在模型质量评估中的实际意义。 |
| 3.3 | [Exact Evaluation](AI/ai-engineering/ch03-evaluation-methodology/3.3-Exact_Evaluation.md) | 客观评估 vs 主观评估、可复现性 | 精确评估方法的体系化梳理，包括功能正确性（pass@k）、字符串相似度（BLEU/ROUGE）、语义相似度等指标。重点讨论其在代码生成、文本摘要和任务完成评估中的适用场景与局限。 |
| 3.4 | [AI as a Judge](AI/ai-engineering/ch03-evaluation-methodology/3.4-AI_as_a_Judge.md) | 自动化评估、LLM-as-Judge | 以 LLM 作为评估者的方法论，解决主观质量（流畅度、有用性、安全性）难以自动化度量的问题。分析 LLM-as-Judge 与人类评估的相关性系数、常见偏差（位置偏差、冗长偏差）及生产环境的应用模式。 |
| 3.5 | [Ranking Models with Comparative Evaluation](AI/ai-engineering/ch03-evaluation-methodology/3.5-Ranking_Models_with_Comparative_Evaluation.md) | Pointwise vs Pairwise、Elo Rating | 比较评估方法论，从 Pointwise 打分到 Pairwise 对决的优劣对比。详解胜率计算、Bradley-Terry 模型与 Elo Rating 排名算法在大规模模型选择中的实践应用。 |

### Chapter 4 · Evaluate AI Systems — 评估 AI 系统

构建系统化的评估流水线。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 4.1 | [Evaluation Criteria](AI/ai-engineering/ch04-evaluate-ai-systems/4.1-Evaluation_Criteria.md) | 评估驱动开发、质量维度定义 | 提出评估标准的四维框架：领域特定能力、生成能力、指令遵从度、成本与延迟。倡导评估驱动开发（EDD）理念——先定义评估标准，再开始系统构建。 |
| 4.2 | [Model Selection](AI/ai-engineering/ch04-evaluate-ai-systems/4.2-Model_Selection.md) | 模型对比、选型策略 | 模型选型的系统化方法，区分硬属性（上下文长度、多模态支持、许可证）和软属性（推理能力、指令遵循）两类筛选维度。介绍迭代式选型流程与性价比评估的量化方法。 |
| 4.3 | [Design Your Evaluation Pipeline](AI/ai-engineering/ch04-evaluate-ai-systems/4.3-Design_Your_Evaluation_Pipeline.md) | 评估流水线设计 | 设计端到端评估流水线的方法论，从测试集构建、分层评估（单元/集成/端到端）到持续评估的自动化。讨论开放式任务的基准设计难题及迭代优化策略。 |

### Chapter 5 · Prompt Engineering — 提示工程

与模型对话是一门手艺。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 5.1 | [Introduction to Prompting](AI/ai-engineering/ch05-prompt-engineering/5.1-Introduction_to_Prompting.md) | Prompt 结构、In-Context Learning、System Prompt | Prompt 的基础结构与第一性原理，剖析 System Prompt、User Prompt、Few-shot 示例的协作机制。讨论模型的指令遵循能力、Prompt 鲁棒性及上下文长度的有效利用率。 |
| 5.2 | [Prompt Engineering Best Practices](AI/ai-engineering/ch05-prompt-engineering/5.2-Prompt_Engineering_Best_Practices.md) | 跨模型通用原则、生产级实践 | 总结跨模型通用的 Prompt 工程原则：清晰指令、角色设定、Few-shot 示例、输出格式约束、思维链引导。每条原则配有正反对比示例，强调从实验到生产的可维护性。 |
| 5.3 | [Defensive Prompt Engineering](AI/ai-engineering/ch05-prompt-engineering/5.3-Defensive_Prompt_Engineering.md) | Prompt 注入、越狱攻击、防御策略 | Prompt 安全防御的系统化梳理，覆盖三类攻击向量（Prompt 提取、越狱注入、信息泄露）和六大风险场景。介绍输入过滤、输出检测、权限隔离等多层防御机制的设计与实现。 |

### Chapter 6 · RAG, Agents & Memory — RAG、智能体与记忆

让模型连接外部世界。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 6.1 | [RAG](AI/ai-engineering/ch06-rag-agents-memory/6.1-RAG.md) | 外部知识检索、Context 构建、减少幻觉 | 检索增强生成（RAG）的核心价值：让模型在推理时接入外部知识，减少幻觉、保持信息时效性。详解 RAG 与微调的对比选择，以及向量数据库、召回策略、重排序的工程实现。 |
| 6.2 | [Agents](AI/ai-engineering/ch06-rag-agents-memory/6.2-Agents.md) | 工具调用、规划、多步推理 | Agent 的定义、能力边界与工作方式，核心是感知-推理-行动的循环。讨论工具调用机制、多步推理规划、信息流管理（上下文传递与结果聚合），以及 Agent 可靠性的工程挑战。 |
| 6.3 | [Memory](AI/ai-engineering/ch06-rag-agents-memory/6.3-Memory.md) | 短期 / 长期记忆、上下文管理 | AI 系统的三层记忆架构：内部知识（参数化记忆）、短期记忆（会话上下文）、长期记忆（持久化存储）。重点讨论如何管理会话内的信息溢出问题与跨会话的记忆持久化策略。 |

### Chapter 7 · Finetuning — 微调

让模型更懂你的领域。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 7.0 | [Finetuning Overview](AI/ai-engineering/ch07-finetuning/7.0-Finetuning_Overview.md) | 微调概述、适用场景 | 微调的全景概述与迁移学习基础，阐明微调相比纯 Prompt 方法在领域适配、输出风格控制和 Token 效率上的优势。梳理本章从内存瓶颈到实战技巧的完整结构。 |
| 7.1 | [When to Finetune](AI/ai-engineering/ch07-finetuning/7.1-When_to_Finetune.md) | 微调时机、成本收益分析 | 微调的四大动机（质量提升、小模型替代大模型、Token 优化、任务特化）与常见误区。提供成本收益分析框架，帮助判断何时该微调、何时该坚持 Prompt 方案。 |
| 7.2 | [Memory Bottlenecks](AI/ai-engineering/ch07-finetuning/7.2-Memory_Bottlenecks.md) | 显存瓶颈、梯度检查点 | 深入分析微调的内存瓶颈来源：模型参数、梯度、优化器状态、激活值四大组成部分的显存消耗计算。介绍数值精度（FP32/FP16/BF16）选择与梯度检查点、量化等缓解策略。 |
| 7.3 | [PEFT and LoRA](AI/ai-engineering/ch07-finetuning/7.3-PEFT_and_LoRA.md) | 参数高效微调、低秩适应 | 参数高效微调（PEFT）的核心思想——冻结大部分参数，只训练少量可学习参数。重点讲解 LoRA 的低秩分解原理、秩（r）与 alpha 的配置策略，以及多 LoRA 适配器的部署方案。 |
| 7.4 | [Model Merging](AI/ai-engineering/ch07-finetuning/7.4-Model_Merging.md) | 模型合并、权重融合 | 模型合并的三种主要方法：权重求和、层堆叠与模型拼接，各自的适用场景与限制。讨论如何通过合并解决多任务微调中的灾难性遗忘问题，以及端侧部署的优化思路。 |
| 7.5 | [Finetuning Tactics](AI/ai-engineering/ch07-finetuning/7.5-Finetuning_Tactics.md) | 微调实战技巧 | 微调实战中的关键决策：基座模型选择（开源 vs 闭源）、微调方法选择（全参 vs PEFT）、框架选择（Hugging Face/Axolotl 等）。附带学习率、批大小、Epoch 数等超参数的调优经验。 |
| 7.6 | [Summary](AI/ai-engineering/ch07-finetuning/7.6-Summary.md) | 微调总结 | 微调章节的脉络回顾，从迁移学习的理论基础出发，串联内存优化、PEFT/LoRA、模型合并到超参数调优的完整知识链。总结微调在 AI 工程中的定位与实践决策框架。 |

### Chapter 8 · Dataset Engineering — 数据集工程

数据决定模型上限。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 8.0 | [Dataset Engineering Overview and Data Curation](AI/ai-engineering/ch08-dataset-engineering/8.0-Dataset_Engineering_Overview_and_Data_Curation.md) | 数据工程概述、数据整理 | 数据集工程的全景概述，倡导"以数据为中心的 AI"范式——模型能力的天花板由数据决定。梳理数据工作的战略地位与三层处理流程（采集→整理→验证），强调数据质量优先于数量。 |
| 8.1 | [Data Augmentation and Synthesis](AI/ai-engineering/ch08-dataset-engineering/8.1-Data_Augmentation_and_Synthesis.md) | 数据增强、合成数据 | 数据增强与合成数据的五个价值维度：扩充数量、提升覆盖率、增强质量、保护隐私、知识蒸馏。讨论合成数据与真实数据的混合比例策略，以及避免"模型塌缩"的注意事项。 |
| 8.2 | [Data Processing and Summary](AI/ai-engineering/ch08-dataset-engineering/8.2-Data_Processing_and_Summary.md) | 数据处理流程 | 数据处理的系统化步骤：数据检查（分布分析）、去重（精确/近似）、清洗（噪声过滤）、格式化（统一模板）与统计验证。提供效率建议与常见数据质量问题的排查清单。 |

### Chapter 9 · Inference Optimization — 推理优化

让模型跑得更快更省。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 9.0 | [Inference Optimization Overview](AI/ai-engineering/ch09-inference-optimization/9.0-Inference_Optimization_Overview.md) | 推理优化概述、延迟与吞吐 | 推理优化的三个层面概览：模型级（量化、蒸馏、剪枝）、硬件级（GPU 利用率、批处理）、服务级（缓存、调度）。讨论延迟、吞吐量与成本之间的权衡策略。 |
| 9.1 | [Inference Optimization Techniques](AI/ai-engineering/ch09-inference-optimization/9.1-Inference_Optimization_Techniques.md) | 量化、KV Cache、投机解码 | 推理优化的具体技术方案：模型级包括量化（INT8/INT4）、知识蒸馏、结构剪枝；服务级包括 KV Cache 优化（PagedAttention）、投机解码（Speculative Decoding）、连续批处理等。每种技术附带适用场景与性能收益分析。 |

### Chapter 10 · AI Engineering Architecture — AI 工程架构

从原型到生产的最后一公里。

| # | 笔记 | 关键词 | 摘要 |
| --- | ------ | -------- | ------ |
| 10.0 | [AI Engineering Architecture](AI/ai-engineering/ch10-ai-engineering-architecture/10.0-AI_Engineering_Architecture.md) | 系统架构、部署模式 | AI 应用架构的渐进式演化路径，从最简单的"Prompt + API"到包含网关、缓存、评估、反馈的完整平台。讨论各阶段的组件选型与系统复杂度的管理策略。 |
| 10.1 | [User Feedback](AI/ai-engineering/ch10-ai-engineering-architecture/10.1-User_Feedback.md) | 用户反馈、持续改进 | 用户反馈系统的设计与实现，区分显式反馈（评分、纠正）和隐式反馈（点击、停留、改写）两种信号源。讨论如何从反馈数据中提取改进信号，构建"模型→用户→数据→模型"的持续改进飞轮。 |

### Supplementary · 补充笔记

| 笔记 | 关键词 | 摘要 |
| ------ | -------- | ------ |
| [Self-Attention 自注意力机制](AI/ai-engineering/supplementary/self-attention-自注意力机制.md) | Query / Key / Value、序列依赖建模 | 自注意力机制的第一性原理推导，从 QKV 矩阵的含义出发，逐步讲解注意力分数计算、Softmax 归一化与加权求和的完整流程。直观解释模型如何学习序列内部的依赖关系。 |
| [Sampling 采样与推理详解](AI/ai-engineering/supplementary/Sampling-采样与推理详解.md) | Token 采样、Temperature、Top-k / Top-p | 采样过程的完整拆解：从 logits 输出到 softmax 概率化，再到 Temperature 缩放、Top-K 截断、Top-P 核采样等策略的数学原理与实际效果。帮助理解不同参数组合对生成文本多样性的精确控制。 |
| [Post-Training 后训练详解](AI/ai-engineering/supplementary/Post-Training-后训练详解.md) | SFT、偏好对齐、RLHF | 后训练两大阶段的深入解析：SFT 阶段的数据构造方法（指令-回复对）与训练策略；偏好对齐阶段的 RLHF、DPO 等算法原理及其在实际模型对齐中的工程实现。 |
| [Transformer 架构全景解析](AI/ai-engineering/supplementary/Transformer-架构全景解析.md) | Transformer 结构、注意力机制 | 从第一性原理完整拆解 Transformer 架构：输入嵌入、位置编码、多头自注意力、前馈网络、残差连接与层归一化。每个模块配有维度变化的可视化说明，帮助建立从输入 Token 到输出概率的完整计算图景。 |

---

## Infra — 基础设施

| 笔记 | 关键词 | 摘要 |
| ------ | -------- | ------ |
| [Docker 学习笔记](infra/docker-learning.md) | 容器化、镜像、Dockerfile | 面向 Python 工程师的 Docker 入门与实践，涵盖容器化核心概念、Dockerfile 编写规范、镜像分层机制与 Docker Compose 多服务编排。 |
| [Vertex AI Provisioned Throughput](infra/vertex-ai-provisioned-throughput-学习笔记.md) | GCP、吞吐量预留、成本控制 | Google Cloud Vertex AI 预置吞吐量（PT）的完整学习笔记，包括 PT 容量单位的计算方式、购买策略、按需与预留的成本对比，以及在高并发 AI 推理场景下的优化实践。 |
| [Static Code Analysis Guide](infra/static-code-analysis-guide.md) | 静态分析、代码质量 | Python 项目静态代码检查的配置指南，以 Ruff（lint + format）和 MyPy（类型检查）为核心工具链。包含 pre-commit 钩子集成、CI 配置与团队规范统一的实践建议。 |
| [Temporal 学习笔记](infra/temporal_notes.md) | 工作流引擎、任务编排 | Temporal 工作流引擎的核心概念与应用场景，重点解决 AI 应用中的故障自动恢复、状态持久化追踪与长耗时任务编排。包含 Workflow/Activity 模型、重试策略与 Python SDK 集成。 |
| [Helm & EKS 学习笔记](infra/deploy/helm-eks-学习笔记.md) | Helm Chart、AWS EKS、K8s 部署 | Helm 包管理器与 AWS EKS 的联合使用指南，讲解 Kubernetes 资源模板化（Chart 结构、values 覆盖）与多环境（dev/staging/prod）部署策略。 |
| [Kubernetes Pod 优雅关闭](infra/deploy/kubernetes-pod-graceful-shutdown.md) | Graceful Shutdown、信号处理 | 从第一性原理拆解 Kubernetes Pod 优雅终止的完整流程：PreStop Hook → SIGTERM 信号 → 宽限期 → SIGKILL。重点讨论滚动更新中的连接排空、长耗时任务保护与健康检查配合。 |
| [OpenTelemetry Guide](infra/observability/opentelemetry-guide.md) | 可观测性、Trace、Metrics、Logs | OpenTelemetry 可观测性框架的完整指南，覆盖链路追踪（Trace）、指标（Metrics）、日志（Logs）三大支柱。包含 Python SDK 的集成实践、Span 上下文传播与导出器配置。 |

---

## Tools — 开发工具

| 笔记 | 关键词 | 摘要 |
| ------ | -------- | ------ |
| [Claude Code 使用文档](tools/Claude_Code_使用文档.md) | Claude Code CLI、配置、使用技巧 | Claude Code CLI 工具的使用指南，涵盖安装配置、身份认证、核心命令与常见工作流程。帮助快速上手 Claude Code 进行日常开发。 |
| [解剖小龍蝦 — OpenClaw 运作原理（讲座整理）](tools/解剖小龍蝦%20—%20以%20OpenClaw%20為例介紹%20AI%20Agent%20的運作原理-transcript-clean.md) | 讲座文字稿、OpenClaw 原理 | 以 OpenClaw 为案例讲解 AI Agent 核心运作原理的讲座清洁版文字稿。从实际系统出发，串讲 Agent 的感知、推理、工具调用与记忆管理的完整链路。 |

---

## Work — 工作项目

| 笔记 | 关键词 | 摘要 |
| ------ | -------- | ------ |
| [OpenClaw 笔记](work/openclaw/openclaw-notes.md) | OpenClaw 系统、架构设计 | OpenClaw 的安装部署、使用配置与系统架构笔记。OpenClaw 是一个本地可控的 Agent 网关系统，笔记记录了其核心模块划分与运行机制。 |
| [Single File Ask Flow](work/ask/single_file_ask_flow.md) | Ask 功能流程设计 | 单文件 Ask 功能的完整技术流程文档，基于 LangGraph 构建的 RAG 管道实现。详解从用户提问到 SSE 流式响应的全链路：文档解析、向量检索、上下文组装与流式协议。 |
| [Plaud Model Hub 架构学习笔记](work/model-hub/Plaud%20Model%20Hub%20架构学习笔记.md) | Model Hub 架构、模型管理 | Plaud Model Hub 的整体架构学习笔记，分析其 SDK 设计理念、模型生命周期管理与生态集成方式。 |
| [Model Hub Config Guide](work/model-hub/model-hub-config-guide.md) | 配置指南 | Model Hub SDK 的配置完全指南，涵盖插件配置、请求限流、熔断保护、多 Endpoint 路由与降级机制。提供各配置项的详细说明与生产环境推荐值。 |

---

## Misc — 杂项

| 笔记 | 关键词 | 摘要 |
| ------ | -------- | ------ |
| [LLM 时代的语言标识指南](misc/language-codes-in-llm-era.md) | ISO 639、BCP 47、语言代码 | 从 ISO 639 到 BCP 47 的语言标识标准体系全景解析。结合 LLM 时代的实际场景——语种检测、ASR 语言提示、多语言 RAG、Prompt 切换——讲解如何选择和使用正确的语言代码。 |

---

## License

本仓库内容仅供个人学习参考。
