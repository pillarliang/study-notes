# Post-Training（后训练）详解

> 预训练产出的 base model 只会"续写文本"，Post-Training 是将其转化为**有用、安全的 AI 助手**的关键阶段。

---

## 1. 为什么需要 Post-Training

预训练（Pre-training）通过海量文本学到了语言知识和世界知识，但 base model 存在根本性不足：

- **只会补全（completion）**：给定前缀后续写最可能的 token，无法直接理解"指令"并给出"回答"。
- **缺少对齐（alignment）**：不知道什么回答是有帮助的、什么回答是有害的。
- **格式不可控**：无法按照用户期望的格式（对话、结构化输出等）进行响应。

Post-Training 正是弥合 base model → 可用 assistant 之间鸿沟的过程，核心包含两大阶段：**Supervised Finetuning (SFT)** 与 **Preference Finetuning**。

---

## 2. Supervised Finetuning（SFT，监督微调）

### 2.1 基本思路

在**高质量的 (input, output) 示例对**上继续训练模型，使其学会按照指令生成期望输出。训练目标与预训练相同——最大化下一个 token 的概率——但训练数据从海量网页文本变为**精心构造的示范数据（demonstration data）**。

### 2.2 数据来源与构造

SFT 数据的来源多样，按质量与成本递增排列：

| 来源 | 说明 |
|------|------|
| **用户交互数据** | 真实用户与模型的对话，需脱敏/过滤 |
| **人工标注者** | 专业标注团队根据 guideline 手写高质量回复 |
| **模型生成 + 人工筛选** | 模型批量生成候选回复，人工挑选最优 |
| **强模型蒸馏** | 用更强的模型（如 GPT-4）生成训练数据给较弱模型使用 |

#### 关键数据设计要素

- **多样性**：覆盖开放问答、数学推理、代码生成、多轮对话、创意写作、安全拒绝等多任务场景。
- **格式一致性**：统一使用 chat template（如 `<|user|>...<|assistant|>...`），使模型学会区分角色和轮次。
- **质量 > 数量**：研究（如 LIMA, 2023）表明，仅约 **1,000 条精选高质量示例**即可显著改变模型行为——"数据质量远比数量重要"。

### 2.3 SFT 的训练细节

- 使用与预训练相同的 **交叉熵损失（cross-entropy loss）**。
- 通常只对 **output 部分** 计算 loss（不对 input/prompt 部分计算 loss），以避免模型学习"复述"用户输入。
- 学习率通常比预训练低一到两个数量级，训练 epoch 数也较少，以防过拟合。

### 2.4 SFT 的局限

- **上限受限于标注者水平**：模型最多学到标注者能写出的最佳回复的水平。
- **只学"做什么"，不学"不做什么"**：SFT 示范只给了正例，模型不知道哪些回复是差的或有害的。
- **偏好无法精准表达**：有些微妙的质量偏好（如"更简洁"vs"更详尽"）难以通过单一示范传递。

> 这些局限正是 Preference Finetuning 要解决的问题。

---

## 3. Preference Finetuning（偏好微调）

### 3.1 核心动机

SFT 教会模型"怎么回复"，Preference Finetuning 教会模型"哪种回复更好"。通过**比较信号（comparison signal）**而非绝对示范来调优，可以超越标注者水平——标注者更容易**判断**哪个好，而不是**写出**最好的回复。

### 3.2 偏好数据的收集

典型流程：

1. 给定 prompt，让模型生成 **两个（或多个）候选回复**。
2. 人类标注者（或更强模型）对这些回复进行 **排序/打分/选择**。
3. 形成偏好对：**(prompt, chosen response, rejected response)**。

标注维度通常包括：有帮助性（helpfulness）、真实性（truthfulness）、无害性（harmlessness）、格式质量等。

### 3.3 Reward Model（奖励模型）

#### 作用

将人类偏好"压缩"为一个可自动打分的模型，为后续 RL 训练提供标量反馈信号。

#### 训练方法

- 基于 **Bradley-Terry 模型**：给定偏好对 (chosen, rejected)，训练 RM 使得 chosen 的得分高于 rejected 的得分。
- 损失函数：

$$ \mathcal{L}_{RM} = -\log\sigma\bigl(r_\theta(x, y_w) - r_\theta(x, y_l)\bigr) $$

其中 $y_w$ 为 chosen response，$y_l$ 为 rejected response，$r_\theta$ 为 RM 输出的标量分数。

#### 架构选择

- 常见做法：在 base LLM 上加一个 **scalar head**（线性层输出单个标量分数）。
- RM 本身也是一个 LLM，因此自身具备语言理解能力。

#### 挑战

- **标注一致性**：不同标注者对同一组回复的偏好可能不同，需要清晰的标注规范和质控流程。
- **过优化风险（reward hacking）**：模型可能学到骗过 RM 但实际质量并未提升的"捷径"。
- **泛化性**：RM 在未见过的 prompt 分布上的打分可靠性是关键难点。

---

## 4. 利用 Reward Model 微调策略

### 4.1 RLHF（Reinforcement Learning from Human Feedback）

最经典的偏好微调范式，核心流程为：

```
SFT Model → 生成回复 → RM 打分 → PPO 更新策略 → 循环迭代
```

#### PPO（Proximal Policy Optimization）

- 将 LLM 视为"策略（policy）"，RM 的打分作为"奖励（reward）"。
- 使用 PPO 算法迭代优化，使模型倾向于生成高 RM 分数的回复。
- **KL 惩罚项**：在奖励函数中加入当前策略与 SFT 模型之间的 KL 散度惩罚，防止模型偏离太远导致语言质量退化或 reward hacking。

$$
\text{reward} = r_\theta(x, y) - \beta \cdot \text{KL}\bigl[\pi_\phi(y|x) \| \pi_{\text{SFT}}(y|x)\bigr]
$$



#### RLHF 的优势

- 能超越 SFT 标注者的水平（因为 RM 捕捉了"好/坏"的判断，而判断比生成更容易做好）。
- 可以同时优化多维度目标（有用性 + 安全性）。

#### RLHF 的挑战

- **训练不稳定**：PPO 涉及同时训练 4 个模型（policy, reference, RM, value model），调参复杂。
- **计算成本高**：每次更新都需要模型生成 + RM 推理 + PPO 梯度计算。
- **Reward hacking**：模型可能找到 RM 的漏洞而非真正提升质量。

### 4.2 DPO（Direct Preference Optimization）

#### 核心思想

绕过 Reward Model，**直接用偏好对数据优化策略模型**。通过数学推导，将 RL 目标转化为一个简洁的分类损失：

$$\mathcal{L}_{\text{DPO}} = -\log\sigma\Bigl(\beta\bigl[\log\frac{\pi_\theta(y_w|x)}{\pi_{\text{ref}}(y_w|x)} - \log\frac{\pi_\theta(y_l|x)}{\pi_{\text{ref}}(y_l|x)}\bigr]\Bigr)$$

#### DPO 的优势

- **无需训练单独的 RM**，简化了训练流水线。
- **无需 RL 采样循环**，训练效率显著提高，更加稳定。
- 实现简单，类似标准的 supervised training。

#### DPO 的局限

- **离线方法**：使用固定的偏好数据集，无法在训练过程中探索新策略生成的回复。
- **对数据质量敏感**：偏好对质量直接影响最终效果。
- 在某些场景下效果不如 RLHF（尤其在需要大量探索的复杂任务上）。

### 4.3 RLHF vs DPO 对比

| 维度 | RLHF (PPO) | DPO |
|------|-----------|-----|
| **是否需要 RM** | 需要单独训练 | 不需要 |
| **训练方式** | 在线 RL 迭代 | 离线监督学习 |
| **训练稳定性** | 较低，调参复杂 | 较高，接近标准微调 |
| **计算成本** | 高（4 模型协同） | 低（仅 policy + reference） |
| **探索能力** | 强（在线生成新回复） | 弱（固定数据集） |
| **工程复杂度** | 高 | 低 |
| **适用场景** | 复杂多维度对齐 | 快速迭代、资源有限场景 |

---

## 5. Post-Training 的整体 Pipeline 总结

```
Base Model（预训练完成）
    │
    ▼
┌─────────────────────┐
│  Stage 1: SFT       │  ← 高质量 (instruction, response) 示范数据
│  学会"如何回复"      │
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  Stage 2: 偏好微调   │  ← 偏好对 (chosen, rejected)
│  学会"哪种回复更好"  │
│                     │
│  路线 A: RLHF       │  → RM + PPO 在线迭代
│  路线 B: DPO        │  → 直接偏好优化
└─────────────────────┘
    │
    ▼
  Aligned Assistant（对齐后的助手模型）
```

---

## 6. 关键 Takeaways

1. **Post-Training 是 base model 到可用产品的桥梁**：没有后训练，LLM 只是一个强大的文本补全引擎。
2. **SFT 奠基，偏好微调提升**：SFT 建立基本的指令遵循能力；偏好微调在此基础上进一步对齐人类价值观。
3. **判断比生成更容易**：偏好微调的根本洞察——人类更擅长"评价"而非"创作"，这使得模型可以超越标注者的生成能力。
4. **质量优于数量**：无论 SFT 还是偏好数据，少量高质量数据往往优于大量低质量数据。
5. **RLHF 与 DPO 是权衡选择**：RLHF 更强大但更复杂；DPO 更简洁但牺牲了在线探索能力。实际工程中两者常组合使用。
6. **Reward Hacking 是持续挑战**：模型可能学会"欺骗" RM 而非真正提升回复质量，需要持续监控和缓解策略。
