# AI Engineering Study Notes

## Table of Contents

### Chapter 1 · AI Engineering Introduction — AI 工程导论

从零认识 AI 工程的全景图。

| # | 笔记 | 关键词 |
|---|------|--------|
| 1.1 | [The Rise of AI Engineering](AI/ch01-ai-engineering/1.1-The_Rise_of_AI_Engineering.md) | AI 工程的兴起、行业趋势 |
| 1.2 | [Foundation Model Use Cases](AI/ch01-ai-engineering/1.2-Foundation_Model_Use_Cases.md) | 基础模型应用场景 |
| 1.3 | [Planning AI Applications](AI/ch01-ai-engineering/1.3-Planning_AI_Applications.md) | AI 应用规划、产品思维 |
| 1.4 | [AI Engineering Stack](AI/ch01-ai-engineering/1.4-AI_Engineering_Stack.md) | AI 工程技术栈 |

### Chapter 2 · Foundation Models — 基础模型

从 token 到 text，理解模型如何"思考"。

| # | 笔记 | 关键词 |
|---|------|--------|
| 2.0 | [Understanding Foundation Models and Training Data](AI/ch02-foundation-models/2.0-Understanding_Foundation_Models_and_Training_Data.md) | 基础模型概述、训练数据 |
| 2.1 | [Modeling Architecture and Size](AI/ch02-foundation-models/2.1-Modeling_Architecture_and_Size.md) | Transformer 架构、模型规模 |
| 2.2 | [Post Training and Sampling](AI/ch02-foundation-models/2.2-Post_Training_and_Sampling.md) | SFT、RLHF、后训练流程 |
| 2.3 | [Sampling](AI/ch02-foundation-models/2.3-Sampling.md) | Token 采样、Temperature、Top-k / Top-p |

### Chapter 3 · Evaluation Methodology — 评估方法论

不能度量，就不能改进。

| # | 笔记 | 关键词 |
|---|------|--------|
| 3.1 | [Challenges of Evaluating Foundation Models](AI/ch03-evaluation-methodology/3.1-Challenges_of_Evaluating_Foundation_Models.md) | 通用性、涌现性、黑箱问题 |
| 3.2 | [Understanding Language Modeling Metrics](AI/ch03-evaluation-methodology/3.2-Understanding_Language_Modeling_Metrics.md) | Cross Entropy、Perplexity、BPC、BPB |
| 3.3 | [Exact Evaluation｜精确评估](AI/ch03-evaluation-methodology/3.3-Exact_Evaluation.md) | 客观评估 vs 主观评估、可复现性 |
| 3.4 | [AI as a Judge｜AI 评审](AI/ch03-evaluation-methodology/3.4-AI_as_a_Judge.md) | 自动化评估、LLM-as-Judge、可扩展评审 |
| 3.5 | [Ranking Models with Comparative Evaluation](AI/ch03-evaluation-methodology/3.5-Ranking_Models_with_Comparative_Evaluation.md) | Pointwise vs Pairwise、Elo Rating、模型排名 |

### Chapter 4 · Evaluate AI Systems — 评估 AI 系统

构建系统化的评估流水线。

| # | 笔记 | 关键词 |
|---|------|--------|
| 4.1 | [Evaluation Criteria｜评估标准](AI/ch04-evaluate-ai-systems/4.1-Evaluation_Criteria.md) | 评估驱动开发、质量维度定义 |
| 4.2 | [Model Selection｜模型选择](AI/ch04-evaluate-ai-systems/4.2-Model_Selection.md) | 模型对比、选型策略 |
| 4.3 | [Design Your Evaluation Pipeline](AI/ch04-evaluate-ai-systems/4.3-Design_Your_Evaluation_Pipeline.md) | 评估流水线设计 |

### Chapter 5 · Prompt Engineering — 提示工程

与模型对话是一门手艺。

| # | 笔记 | 关键词 |
|---|------|--------|
| 5.1 | [Introduction to Prompting](AI/ch05-prompt-engineering/5.1-Introduction_to_Prompting.md) | Prompt 结构、In-Context Learning、System Prompt |
| 5.2 | [Prompt Engineering Best Practices](AI/ch05-prompt-engineering/5.2-Prompt_Engineering_Best_Practices.md) | 跨模型通用原则、清晰指令、生产级实践 |
| 5.3 | [Defensive Prompt Engineering](AI/ch05-prompt-engineering/5.3-Defensive_Prompt_Engineering.md) | Prompt 注入、越狱攻击、防御策略 |

### Chapter 6 · RAG, Agents & Memory — RAG、智能体与记忆

让模型连接外部世界。

| # | 笔记 | 关键词 |
|---|------|--------|
| 6.1 | [RAG｜检索增强生成](AI/ch06-rag-agents-memory/6.1-RAG.md) | 外部知识检索、Context 构建、减少幻觉 |
| 6.2 | [Agents｜AI 智能体](AI/ch06-rag-agents-memory/6.2-Agents.md) | 工具调用、规划、多步推理 |
| 6.3 | [Memory｜记忆系统](AI/ch06-rag-agents-memory/6.3-Memory.md) | 短期 / 长期记忆、上下文管理 |

### Chapter 7 · Finetuning — 微调

让模型更懂你的领域。

| # | 笔记 | 关键词 |
|---|------|--------|
| 7.0 | [Finetuning Overview](AI/ch07-finetuning/7.0-Finetuning_Overview.md) | 微调概述、适用场景 |
| 7.1 | [When to Finetune](AI/ch07-finetuning/7.1-When_to_Finetune.md) | 微调时机、成本收益分析 |
| 7.2 | [Memory Bottlenecks](AI/ch07-finetuning/7.2-Memory_Bottlenecks.md) | 显存瓶颈、梯度检查点 |
| 7.3 | [PEFT and LoRA](AI/ch07-finetuning/7.3-PEFT_and_LoRA.md) | 参数高效微调、低秩适应 |
| 7.4 | [Model Merging](AI/ch07-finetuning/7.4-Model_Merging.md) | 模型合并、权重融合 |
| 7.5 | [Finetuning Tactics](AI/ch07-finetuning/7.5-Finetuning_Tactics.md) | 微调实战技巧 |
| 7.6 | [Summary](AI/ch07-finetuning/7.6-Summary.md) | 微调总结 |

### Chapter 8 · Dataset Engineering — 数据集工程

数据决定模型上限。

| # | 笔记 | 关键词 |
|---|------|--------|
| 8.0 | [Dataset Engineering Overview and Data Curation](AI/ch08-dataset-engineering/8.0-Dataset_Engineering_Overview_and_Data_Curation.md) | 数据工程概述、数据整理 |
| 8.1 | [Data Augmentation and Synthesis](AI/ch08-dataset-engineering/8.1-Data_Augmentation_and_Synthesis.md) | 数据增强、合成数据 |
| 8.2 | [Data Processing and Summary](AI/ch08-dataset-engineering/8.2-Data_Processing_and_Summary.md) | 数据处理流程、总结 |

### Chapter 9 · Inference Optimization — 推理优化

让模型跑得更快更省。

| # | 笔记 | 关键词 |
|---|------|--------|
| 9.0 | [Inference Optimization Overview](AI/ch09-inference-optimization/9.0-Inference_Optimization_Overview.md) | 推理优化概述、延迟与吞吐 |
| 9.1 | [Inference Optimization Techniques](AI/ch09-inference-optimization/9.1-Inference_Optimization_Techniques.md) | 量化、KV Cache、投机解码 |

### Chapter 10 · AI Engineering Architecture — AI 工程架构

从原型到生产的最后一公里。

| # | 笔记 | 关键词 |
|---|------|--------|
| 10.0 | [AI Engineering Architecture](AI/ch10-ai-engineering-architecture/10.0-AI_Engineering_Architecture.md) | 系统架构、部署模式 |
| 10.1 | [User Feedback](AI/ch10-ai-engineering-architecture/10.1-User_Feedback.md) | 用户反馈、持续改进 |

### Supplementary · 补充笔记

深入特定主题的专题笔记。

| # | 笔记 | 关键词 |
|---|------|--------|
| - | [Self-Attention｜自注意力机制](AI/supplementary/self-attention-自注意力机制.md) | Query / Key / Value、序列依赖建模 |
| - | [Sampling｜采样与推理详解](AI/supplementary/Sampling-采样与推理详解.md) | Token 采样、Temperature、Top-k / Top-p |
| - | [Post-Training｜后训练详解](AI/supplementary/Post-Training-后训练详解.md) | SFT、偏好对齐、RLHF、从 Base Model 到 Assistant |

---

## About

这些笔记主要基于 [AI Engineering](https://www.oreilly.com/library/view/ai-engineering/9781098166298/) (Chip Huyen) 等资料整理而成，融入了个人的理解和补充。

笔记风格：**中英混排** — 术语保留英文原文，解释和思考用中文书写，方便回顾时既能对接英文文献，又不丧失母语的思维速度。

## License

本仓库内容仅供个人学习参考。
