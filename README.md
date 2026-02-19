# AI Engineering Study Notes

## Table of Contents

### Part I · Model Internals — 模型内部机制

从 token 到 text，理解模型如何"思考"。

| # | 笔记 | 关键词 |
|---|------|--------|
| - | [Self-Attention｜自注意力机制](AI/self-attention-自注意力机制.md) | Query / Key / Value、序列依赖建模 |
| - | [Sampling｜采样与推理详解](AI/Sampling-采样与推理详解.md) | Token 采样、Temperature、Top-k / Top-p |
| - | [Post-Training｜后训练详解](AI/Post-Training-后训练详解.md) | SFT、偏好对齐、RLHF、从 Base Model 到 Assistant |

### Part II · Model Evaluation — 模型评估

不能度量，就不能改进。

| # | 笔记 | 关键词 |
|---|------|--------|
| 3.1 | [Challenges of Evaluating Foundation Models](AI/3.1-Challenges_of_Evaluating_Foundation_Models.md) | 通用性、涌现性、黑箱问题 |
| 3.2 | [Understanding Language Modeling Metrics](AI/3.2-Understanding_Language_Modeling_Metrics.md) | Cross Entropy、Perplexity、BPC、BPB |
| 3.3 | [Exact Evaluation｜精确评估](AI/3.3-Exact_Evaluation.md) | 客观评估 vs 主观评估、可复现性 |
| 3.4 | [AI as a Judge｜AI 评审](AI/3.4-AI_as_a_Judge.md) | 自动化评估、LLM-as-Judge、可扩展评审 |
| 3.5 | [Ranking Models with Comparative Evaluation](AI/3.5-Ranking_Models_with_Comparative_Evaluation.md) | Pointwise vs Pairwise、Elo Rating、模型排名 |
| 4.1 | [Evaluation Criteria｜评估标准](AI/4.1-Evaluation_Criteria.md) | 评估驱动开发、质量维度定义 |

### Part III · Prompt Engineering — 提示工程

与模型对话是一门手艺。

| # | 笔记 | 关键词 |
|---|------|--------|
| 5.1 | [Introduction to Prompting](AI/5.1-Introduction_to_Prompting.md) | Prompt 结构、In-Context Learning、System Prompt |
| 5.2 | [Prompt Engineering Best Practices](AI/5.2-Prompt_Engineering_Best_Practices.md) | 跨模型通用原则、清晰指令、生产级实践 |
| 5.3 | [Defensive Prompt Engineering](AI/5.3-Defensive_Prompt_Engineering.md) | Prompt 注入、越狱攻击、防御策略 |

### Part IV · RAG — 检索增强生成

让模型不再"幻觉"。

| # | 笔记 | 关键词 |
|---|------|--------|
| 6.1 | [RAG｜检索增强生成](AI/6.1-RAG.md) | 外部知识检索、Context 构建、减少幻觉 |

---

## About

这些笔记主要基于 [AI Engineering](https://www.oreilly.com/library/view/ai-engineering/9781098166298/) (Chip Huyen) 等资料整理而成，融入了个人的理解和补充。

笔记风格：**中英混排** — 术语保留英文原文，解释和思考用中文书写，方便回顾时既能对接英文文献，又不丧失母语的思维速度。

## License

本仓库内容仅供个人学习参考。
