# Alibaba ROLL 与 ROCK 学习指南

![[assets/fig-global-map.png|860]]

> [!abstract] 这份笔记是什么
> 一份面向上手的概念 + 实操指南。定位为**综述 + 实操**：先讲清 ROLL 和 ROCK 到底解决什么问题、为什么成对出现，再给最小可跑的安装与用法。
>
> **先纠正一个常见误解**：ROLL 和 ROCK 不是 PyTorch、TensorFlow 那种通用深度学习框架，不能用来"从零搭一个神经网络"。它们是**用强化学习（RL）训练大语言模型（LLM）**这条专门赛道上的上下游工具——一个管训练，一个管环境。

## 一、一句话本质

ROLL 和 ROCK 是阿里巴巴开源的一对工具，用**强化学习**来训练大语言模型——让模型靠"自己答题、被打分、再改进"来变强，而不是像监督微调（SFT）那样照抄标准答案。其中 **ROLL 是训练引擎**：它把"模型生成回答 → 给回答打分 → 据此更新模型"这条主循环**拆成生成、打分、更新几个独立的工作单元（称为"角色"role）**，每个角色都能单独分配 GPU、单独扩缩、单独选后端，再由 ROLL 统一编排到大量 GPU 上协同运转。正因如此，**同一份训练代码几乎不用改，就能从单机一路扩到上千张卡**（官方称它为 RL"扩展库"，不只是个跑流程的引擎）；**ROCK 是环境工厂**，当模型需要在一个环境里多步交互来学习（比如调用工具、玩游戏、操作终端这类 agent 任务）时，它提供大量互相隔离、可弹性扩展的沙箱环境，让模型在里面反复试错。

拆开看：

- **ROLL** = RL 训练引擎 / 扩展库。把"生成 → 打分 → 更新参数"这条主循环**拆成可独立调度的角色**，灵活铺到 GPU 集群上（单机 → 上千卡），算法与后端都能自由替换（详见 §三）。
- **ROCK** = RL 环境工厂。负责给 agentic RL 提供大量可隔离、可扩展的沙箱环境，让模型在里面"试错"。

两者由阿里同一团队（淘天未来生活实验室 + 阿里 AI Engine 团队）联合推出，ROCK 是 ROLL 生态里负责"环境基础设施"的那一块。**ROLL 是训练大脑，ROCK 是练习场。**

## 二、背景原理：从强化学习基础到这两个工具的由来

这一章按四条递进的主线展开：RL 的回路与术语（§2.1）→ RL 到底怎么学会的（§2.2）→ 为什么训练 LLM 会走到 RL（§2.3）→ 用 RL 训 LLM 在工程上难在哪、于是有了 ROLL 和 ROCK（§2.4）。

### 2.1 强化学习的回路与基本术语

强化学习的基本回路：**智能体（agent）看观测（observation）→ 选动作（action）→ 环境给奖励（reward）；如此往复，目标是最大化拿到的奖励总和。** 后文的术语都挂在这个回路上：
智能体会跟环境互动。环境会给智能体一个观测（observation），智能体看到这个观测后，会采取一个动作，从而影响环境。环境会给出新的观测，智能体会给出新的动作。

![[assets/fig-rl-loop.png|638]]

- **policy（策略）**：agent "看到 observation 该选哪个 action" 的决策规则。在 LLM 里，**policy 就是模型本身**；"训练"就是调 policy 的参数，让它更倾向选出能拿高奖励的 action。
- **trajectory（轨迹）**：一次互动从头到尾，把 (observation, action, reward) 串成的完整序列。一条 trajectory = 一段完整经历。
- **rollout**：让当前 policy 真去环境里跑一遍、产出一条 trajectory 的过程。LLM 语境里，rollout 就是"模型生成一段回答或一串交互"。
- **return（回报）**：一条 trajectory 上奖励的总和，也就是 RL 要最大化的那个"奖励总和"。

回路里还剩两个关键问题没答：**reward 具体怎么变成模型的进步？** 这就是 §2.2 的内容。

### 2.2 RL 是怎么学会的：从 reward 到一次参数更新

RL 改进模型，本质是一个三步循环：**让 policy 生成回答 → 给回答打分 → 用分数更新 policy**。看懂这个循环，先看"更新"这一步在神经网络里到底是什么。

**先回顾监督训练怎么更新参数。** 普通监督训练（比如 SFT）每步只反复做两件事：

1. **前向（forward）**：把输入喂进模型，逐层算出输出（预测）。
2. **反向（backward，反向传播）**：用 loss 函数算出输出与标准答案的误差，反向传播求出 loss 对每个参数的梯度，再用梯度下降更新参数。

它能成立的前提是**有"标准答案"可比**：loss 衡量"预测离标准答案多远"，参数朝缩小这个差距的方向调。

**RL 没有标准答案。** 它手里只有"模型自己生成的回答"和"环境给这份回答的 reward"。于是分两步，把 reward 变成一次参数更新。

**第一步，reward → advantage：先减掉一个基准。** reward 是环境对单份回答打的分（可以是 0/1，也可以是连续值，取决于哪种范式——见 §2.3；对学习机制来说它只是"一个数"）。但直接拿 reward 更新有个毛病：分数高不代表"比平时好"。所以 RL 真正用的是 **advantage（优势）= 这份回答的 reward − 一个基准**，基准常取"同一道题、同一模型一次生成的一组回答的平均分"。

> [!warning] 目标是最大化 reward，但更新用的是 advantage——这俩不矛盾
> **目标**（训练最终要达成的）始终是"把 reward / return 做到最高"，这没变。**advantage 不是另一个目标，而是为了又稳又快地逼近这个目标，每一步更新时用的信号。** 为什么不直接用 reward 当更新权重？因为若 reward 全是正数，连差回答的梯度权重也是正的——**差回答也会被推高，只是推得少**，又慢又晃。减掉基准后，比平均好的才为正（推高）、比平均差的为负（压低），方向更准、噪声更小。而且这个基准不依赖"具体选了哪个 action"，数学上**不改变优化的最终方向**（无偏），只降低每步抖动。一句话：要登的那座"reward 最高"的山顶没变，advantage 只是让爬山每一步迈得更稳、不走歪。

> [!note] 用具体数字看 advantage（reward 是 0/1 的情形）
> 一道题生成 8 份回答，3 份对（reward=1）、5 份错（reward=0），组平均 = 3/8 = 0.375。对的每份 advantage = 1 − 0.375 = **+0.625**，错的 = 0 − 0.375 = **−0.375**。**单份 reward 只有 0/1，但减掉小数的组平均后，advantage 就有了正负和大小。**
> 这个基准还自动随难度伸缩：难题（8 份只对 1 份）那份对的 advantage 高达 +0.875、奖励更猛；而 8 份全对或全错时 advantage 全为 0——这组没有可学的信号。这也是"为什么要一次生成一组"：单独一份没有参照，算不出 advantage。

> [!note] reward 也可以是连续分：RLHF 的 reward model 怎么来的
> 上面的例子用的是 0/1 reward（RLVR 那种程序判定）；reward 也能是连续分，最典型的是 RLHF。看似矛盾——人给的明明是"A 比 B 好"（二选一），为什么 reward model 打出来是连续分？因为**人工标注的形式 ≠ 模型输出的形式**。人标的成对偏好（A>B、B>C、A>D…）是用来**训练 reward model** 的：训练目标要求"被选中的答案得分高于落选的"，海量互相交叠的比较汇到一起，逼着模型把所有回答排到**一根连续的质量轴**上，才能自洽。
> 最贴切的类比是 **Elo 积分**：每盘棋只有输 / 赢（二元），但大量对局能给每个棋手算出连续的 Elo 分。训好之后，reward model 拿到任意单份回答就直接打出一个绝对分；之后就拿这个分当 reward，去算 advantage、更新参数。
> 对比 RLVR：它的 0/1 由**程序**直接判定、直接当 reward，没有"训一个模型"这一步——所以数学 / 代码用 RLVR，"回答好不好"这种没法程序验证的才用 reward model。（RLVR / RLHF 等范式的全貌见 §2.3。）

**第二步，advantage → 参数更新：把回答当带权重的示范。** 拿到 advantage 后，更新仍然走 forward + backward，只是把**模型自己生成的这份回答**当成"示范答案"，并给它的梯度**乘上 advantage 当带符号的权重**：

- advantage 为正 → 朝"更可能生成这份回答"的方向调参数，即**推高**它的生成概率；
- advantage 为负 → 反过来，朝"更不可能生成"调，即**压低**；
- |advantage| 越大，这一步调得越狠；advantage = 0 时梯度为 0，参数不动。

"推高 / 压低"改的就是"这串 token 以后还会不会被模型吐出来"。两步合起来：**好于平均的回答被强化、差于平均的被抑制，模型就一点点学会多产出能拿高 reward 的回答。**

**这一整套配方，就叫一个 RL 算法。** 一句话：**RL 算法 = 把"生成的回答 + 它们的 reward"换算成"对 policy 的一次参数更新"的具体做法。** §3.3 里 PPO、GRPO 等算法的差别，主要在三处：

- **怎么定基准、算 advantage**：例如 GRPO（DeepSeek 用的）不额外训一个 value 网络（一个专估"当前局面值多少分"的模型来当基准），而是直接用"一组回答的平均"当基准，省资源——所以大规模、agentic 场景偏爱它。
- **怎么防止更新过猛**：clipping（裁剪）限制每步幅度，防止 policy 一步跳太远把模型训崩；reward normalization（归一化）把奖励缩放到稳定范围。这些都是可调的训练配置（ROLL 对它们的支持见 §3.3）。
- **按什么粒度分配信用**：把 advantage 算给整条 trajectory（TrajectoryWise，如 StarPO），还是拆到每一步 action（StepWise，如 GiGPO）。多步任务里，逐步分配能更准地定位是哪一步做对了。

### 2.3 LLM 后训练为什么走到 RL

一个 LLM 的能力分两段练成：

1. **预训练（pre-training）**：在海量文本上学"接下一个 token"，得到基础能力。
2. **后训练（post-training）**：把基础模型对齐到"有用、能推理、会用工具"。

后训练这几种方法顺着两条线往前递进。拿"教学生解数学题"打比方，这两条线很具体：

- **第一条线：给模型的反馈越来越少（"稀疏"指这个）。** SFT 给整份标准解答让模型照抄，每一步都手把手教；RLHF 退一步，只说"这份解法比那份好"；RLVR 再退，只判"最终答案对还是错"；agentic 最狠，要走完一长串操作，到最后才给一句"成功 / 失败"。喂给模型的信息一档比一档少。
- **第二条线：奖励越来越对准"真正想要的结果"（"接近真实目标"指这个）。** 还拿解数学题说。SFT 奖励的是"答得像不像那份范例解答"——但"写得像"不等于"算对了"：模型可能套着范例的格式，写出一段看着工整、其实算错的过程；换一道没背过的题还会直接卡住。RLVR、agentic 不看过程像不像，只认"最终答案对不对""任务完成没有"，模型于是被逼着真把题解对，而不是把解答写得像。一句话：SFT 优化的是"模仿示范"，RLVR / agentic 优化的是"达成目标"——而"模仿得像"和"真做成了"常常是两码事。

四种范式沿这两条线一档档排开，差别正是每种范式的 reward **从哪来、有多密**：

- **SFT**（监督微调）：给标准答案，模型照抄。信号最密，但只能学到示范覆盖的范围。
- **RLHF**（基于人类反馈的 RL）：reward 来自一个 **reward model**（人标偏好训出来的打分模型，二元偏好怎么变连续分见 §2.2），信号从"标准答案"降为"好坏分"。
- **RLVR**（Reinforcement Learning with Verifiable Rewards，可验证奖励 RL）：reward 来自**程序自动判定**（数学题验算、代码跑测试），结果是 0/1 这种离散分，不需要人标。
- **Agentic RL**：模型要在一个环境里**多轮交互**（用工具、玩游戏、操作终端），reward 来自环境最终的成败。信号最稀疏，也最接近真实任务。

> [!note] 四种里只有 SFT 不是强化学习
> 分界线在"模型从什么里学"：**SFT** 照抄固定的标准答案、不看自己生成了什么，是监督学习；**RLHF / RLVR / Agentic RL** 都走 §2.2 那个"生成 → 打分 → 更新"回路，区别只在"谁来打分"（reward model / 程序验证 / 环境成败）。这也是为什么后三种都需要"生成 + 打分 + 更新"三种计算（见 §2.4），而 SFT 只需要前向 + 反向。

> [!tip] DPO 是条捷径
> DPO（Direct Preference Optimization）跳过"训 reward model + 跑 RL"这两步，直接拿偏好对（A 比 B 好）来微调模型。它简单，但只适合偏好对齐，做不了需要环境多轮交互的 agentic 任务。

> [!note] 这条演进决定了工具长什么样
> 越往后，"环境"越重要。SFT 不需要环境，RLVR 需要一个验证器，agentic RL 则需要一整套能多轮交互的环境。ROLL 覆盖从 SFT 到 agentic 的全部范式，ROCK 专门补齐 agentic RL 那一块最重的环境需求。

### 2.4 用 RL 训 LLM，难在"一个训练步里有三种计算要协同"

§2.2 说过，RL 改进模型是"生成 → 打分 → 更新"转圈。把它落到训练 LLM 的**一个训练步**里，就是三段**性质完全不同**的计算，按顺序跑完算一步，再回到第一步：

1. **生成 rollout**：让当前模型对一批题目各生成回答（即 §2.1 的 rollout）。这是**纯推理**——只有前向、没有反向，和平时拿模型聊天同一种计算，追求"吐 token 快、吞吐高"。用 vLLM、SGLang 这类推理引擎。
2. **算 reward**：给每份回答打分（程序验证或 reward model，见 §2.3），追求"判得准"。
3. **更新参数**：把分数换算成 advantage、做一次参数更新——也就是 §2.2 那套 forward + backward（只是 loss 用 advantage 加权）。它要存梯度和优化器状态，**极吃显存、得多卡并行**。用 FSDP2、Megatron 这类训练框架。

> [!note] 这三步对照 §2.1 的基本回路
> 这三步其实就是 §2.1 那个 RL 回路的落地：被训练的**模型** = agent / policy；
> 第 1 步生成的**回答** = agent 的 action（rollout 是"生成"这个过程，回答是它的产物）；
> 第 2 步的**打分者** = environment（RLVR 是验证程序、RLHF 是 reward model、agentic 是真实环境），它给出的分就是 **reward**；
> 第 3 步**更新参数** = 拿 reward 回头调 policy。所以"算 reward"逻辑上就是回路里"环境给奖励"那一环，只是这里的"环境"不是物理世界，而是验证程序 / reward model / 沙箱。

![[assets/fig-rl-train-step.png|760]]

**难点在这里。** 监督训练自始至终只有第 3 种计算在循环，一套训练框架就够；RL 却把三种**性质相反**的计算凑进一个训练步：第 1 步要高吞吐推理引擎，第 3 步要省显存、能多卡并行的训练框架，两者优化方向几乎打架。可它们偏偏要**挤在同一批 GPU 上轮流跑**，还要不停互相传数据——生成的回答送去打分，分数送去更新，更新出的新模型再送回去生成。

于是冒出一堆新问题：谁在什么时刻占用哪些 GPU？中间数据怎么高效搬运？**这就是 ROLL 要解决的核心问题**——它把"生成 / 打分 / 更新"封装成能各自独立调度的角色，统一编排到集群上（详见 §3.1）。

而当第 1 步不再是"生成一段回答"，而是"在一个真实环境里**多轮交互**"（agentic）时，又多一个新问题：成千上万个环境实例怎么并发、隔离、互不污染——**这就轮到 ROCK 了**。

## 三、ROLL：RL 训练引擎

### 3.1 本质：Ray 多角色分布式 + 策略抽象

ROLL 的设计落在两个支点上：

- **多角色分布式架构（multi-role distributed architecture）**：基于 Ray，把 §2.4 那三种计算（生成 / 打分 / 更新）封装成可独立调度的角色（role）。Ray 负责把这些角色灵活地铺到集群的 GPU 上，支持异构任务调度。

> [!info] Ray 是什么
> 一个开源的**分布式计算框架**（出自 UC Berkeley）：让你几乎不改 Python 代码，就把计算铺到一整个集群的多机多卡上跑，不用手写"哪台机器跑哪段、数据怎么跨机传"这些底层调度。核心抽象是 **Actor**——把一个普通类标记一下，就变成常驻某节点、有状态的服务进程（很适合一个一直占着 GPU 的模型角色）。ROLL 正是把"生成 / 打分 / 更新"各包成一个 Ray Actor，剩下的铺卡和数据传输交给 Ray，于是不必从零造分布式底座。

- **策略抽象（strategy abstraction）**：把"用哪个后端跑这个角色"抽象成可替换的策略。同一份训练逻辑，底层换 vLLM 还是 SGLang、换 FSDP2 还是 Megatron，上层代码不用动。这是它能"从单机无缝扩到上千卡"的关键。

> [!tip] AutoDeviceMapping
> ROLL 提供 AutoDeviceMapping，可以自定义把哪个角色放到哪些设备上。比如生成和训练共享同一批卡（节省资源），或各自独占（追求吞吐）——靠配置切换，不改代码。

### 3.2 后端：训练侧和生成侧分开选


| 角色               | 可选后端                      | 说明                                               |
| ---------------- | ------------------------- | ------------------------------------------------ |
| 生成 / 推理（rollout） | **vLLM**、**SGLang**       | 高吞吐推理引擎，负责让模型产出回答；支持 FP8 rollout                 |
| 训练（training）     | **FSDP2**、**Megatron-LM** | Megatron 支持 5D 并行（dp/tp/pp/cp/ep）；两者都支持 **LoRA** |


> [!info] 5D 并行是什么
> 大模型训练把计算切到多卡的五个维度：data（数据）、tensor（张量切分）、pipeline（流水线分层）、context（长序列切分）、expert（MoE 专家切分）。卡越多、模型越大，越需要多维度组合切分。单机小模型用 FSDP2 就够，上千卡训大模型才需要 Megatron 的 5D。

### 3.3 算法：20+ RL 算法开箱即用

这里说的"算法"，指的是**把"生成的回答 + reward"换算成"对 policy 的一次参数更新"的具体做法**（即 §2.2 讲的 RL 算法）。**它不是奖励函数**（奖励函数管"每份回答得几分"，见 §2.3）；两层正交，可自由搭配（如 RLVR 的 0/1 奖励 + GRPO 算法 = DeepSeek-R1 的经典组合）。各算法的差别都落在三处：**怎么定基准算 advantage、怎么防更新过猛、按什么粒度分配信用**（§2.2 已详述）。

- **基础 RL**：
  - **PPO**：最经典；带 clipping 防更新过猛，但需**额外训一个 value 网络**当基准。RLHF 最早用它。
  - **GRPO**：DeepSeek 提出，**去掉 value 网络**，直接用"一组回答的平均"当基准（§2.2 用具体数字演示过），省资源、最流行。
  - **GSPO**：阿里 Qwen 提出的 GRPO 变体，把优化粒度放到**整句序列**级别，大模型 / MoE 上更稳。
  - **Reinforce++/ TOPR / RAFT**++：REINFORCE 及其他思路的变体（RAFT++ 偏"挑高分回答再微调"，TOPR 是 off-policy 改进），都是"算 advantage / 控更新"的不同配方。
- **Agentic 专用**：StarPO（按整条轨迹优化，TrajectoryWise）、GiGPO（按单步优化，StepWise）。
- **配套丰富配置**：reward normalization、clipping、advantage estimation 等。

### 3.4 五种 Pipeline：对号入座选范式

ROLL 把不同后训练范式封装成 pipeline，选哪个取决于"监督信号长什么样"（呼应 §2.3）：


| Pipeline    | 用在什么场景               | 对应范式       |
| ----------- | -------------------- | ---------- |
| **SFT**     | 有标准答案，直接监督微调         | SFT        |
| **DPO**     | 有偏好对（A 比 B 好），直接偏好优化 | 偏好对齐       |
| **RLVR**    | 数学 / 代码 / 问答，答案可程序验证 | 可验证奖励 RL   |
| **Agentic** | 游戏、工具调用、多轮对话等环境交互    | agentic RL |
| **Distill** | 大模型蒸馏到小模型（含 VLM）     | 知识蒸馏       |


> [!tip] 上手优先级
> 想验证流程是否跑通 → 先跑 **SFT** 或 **RLVR**（不依赖外部环境，最简单）。要做 agent 训练、需要环境 → 上 **Agentic** pipeline，这时才需要搭配 ROCK。

### 3.5 安装与最小用法

安装见官方指南：`https://alibaba.github.io/ROLL/docs/Getting Started/Installation/`

最小 RLVR 配置（以 Qwen2.5-7B 为例）：

```yaml
# rlvr_config.yaml
model:
  model_name: qwen2.5-7b

training:
  total_train_steps: 1000
  learning_rate: 1e-5

rollout:
  num_rollout_workers: 4

inference:
  backend: vllm          # 生成后端，可换 sglang
```

单机启动：

```bash
python -m roll.main \
  --config examples/qwen2.5-7B-rlvr_megatron/rlvr_config.yaml
```

多机分布式：

```bash
bash examples/qwen2.5-7B-rlvr_megatron/run_distributed.sh
```

> [!note] 可观测性
> 训练过程已接入 SwanLab、WandB、TensorBoard，配置里指定即可看 loss、reward 曲线。

## 四、ROCK：agentic RL 的环境工厂

### 4.1 本质：可隔离、可扩展的沙箱环境管理

全名 **ROCK = Reinforcement Open Construction Kit**。当训练进入 agentic 阶段，模型需要在成千上万个环境实例里反复试错。ROCK 做的就是**构建、部署、编排这些环境**，并保证它们互不干扰、能横向扩展。它基于 Docker 做容器编排，提供多级隔离机制保证环境稳定运行。

### 4.2 架构：三类节点 + SDK/CLI

ROCK 是 client-server 分布式设计，三类节点各司其职：

![[assets/fig-rock-arch.png|820]]

- **Admin**：中央调度，管环境部署和沙箱资源分配。
- **Worker**：执行节点，提供物理机算力给沙箱。
- **Rocklet**：轻量代理，负责 SDK 调用到沙箱之间的通信。

### 4.3 接口：兼容 GEM 协议，make/reset/step

ROCK 对外接口对齐 **GEM 协议标准**——也就是 OpenAI Gym 那一套经典的 `make() / reset() / step()` 强化学习环境接口。会用 Gym，就会用 ROCK。

它支持多种 action 协议，覆盖不同类型的环境交互：

- **GEM**：标准 RL 环境动作（如游戏里的 up/down）。
- **Bash**：在沙箱里执行 shell 命令（适合代码 / 终端类 agent）。
- **Chat**：对话式交互。

环境运行时是**有状态的（stateful）**，资源额度可配置，生命周期自动管理。

### 4.4 安装与最小用法

```bash
git clone https://github.com/alibaba/ROCK.git
cd ROCK
uv venv --python 3.11 --python-preference only-managed
uv sync --all-extras
source .venv/bin/activate
rock admin start          # 启动 Admin 调度节点
```

最小交互代码（和 Gym 写法一模一样）：

```python
import rock

env = rock.make("game:Sokoban-v0-easy")        # 创建一个推箱子环境
observation, info = env.reset(seed=42)          # 重置，拿到初始观测
observation, reward, terminated, truncated, info = env.step("up")  # 执行一个动作
env.close()
```

## 五、两者怎么合：一个 agentic RL 训练步的闭环

ROLL 和 ROCK 的协作发生在 ROLL 的 **Agentic Pipeline** 里。一个训练步的数据闭环：

![[assets/fig-agentic-loop.png|820]]

- ROLL 的生成角色产出动作，通过 ROCK 的 SDK 发给沙箱环境。
- ROCK 在隔离的 Worker 上执行动作，返回新观测和奖励。
- ROLL 收集整条轨迹，用 agentic 算法（StarPO / GiGPO）算优势、更新参数。
- 循环往复。**ROCK 负责"试错的场地"，ROLL 负责"从试错里学习"。**

## 六、学习路线建议

> [!tip] 三步上手
>
> 1. **跑通最简单的**：先用 ROLL 的 SFT 或 RLVR pipeline，单机 + 一个小模型（如 Qwen2.5-7B），不碰环境，确认训练循环能转。
> 2. **理解 agentic 闭环**：单独装 ROCK，用 `rock.make()` 跑一个 Sokoban 环境，手动 step 几步，感受 Gym 式接口。
> 3. **打通全链路**：用 ROLL 的 Agentic pipeline 接上 ROCK 环境，跑一次完整的 agentic RL 训练。

> [!warning] 何时不需要它们
>
> - 只想做推理 / 部署 → 直接用 vLLM、SGLang 即可，不需要 ROLL。
> - 只做 SFT 微调小模型 → LLaMA-Factory 等更轻。ROLL 的价值在**大规模、多角色、RL/agentic** 场景。
> - 不做 agentic、奖励能用纯规则算 → 只用 ROLL，不需要 ROCK。

---

## 参考来源

- [alibaba/ROLL · GitHub](https://github.com/alibaba/ROLL)
- [ROLL 官方文档](https://alibaba.github.io/ROLL/)
- [ROLL 论文：Reinforcement Learning Optimization for Large-Scale Learning (arXiv 2506.06122)](https://arxiv.org/pdf/2506.06122)
- [alibaba/ROCK · GitHub](https://github.com/alibaba/ROCK)

