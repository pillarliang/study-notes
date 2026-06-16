---
title: LLM 究竟如何运作
source: https://www.0xkato.xyz/how-llms-actually-work/
author:
published: 2026-06-01
created: 2026-06-08
description: 从 token 到 Transformer 块再到逐 token 生成循环，完整梳理现代 LLM 的工作原理
tags:
  - AI
---

现代 LLM 基本上就是 Transformer 块的反复堆叠，因此理解 Transformer 的运作原理，就能掌握 LLM 的绝大部分核心。

本文介绍现代 Transformer 架构 LLM 内部的主要机制，不涉及复杂的数学推导。当然，数学原理值得深入学习，这里更侧重于帮助建立直觉。

当前主流 LLM 共享同一个 Transformer 家族的"骨架"。主要区别来自各家训练数据的差异、参数规模与配置的选择，以及后训练阶段的方法。读完本文，应该能够读懂现代 LLM 论文或模型卡，了解每个章节对应的是哪个架构组件。

梳理路径如下：

1. Token：文本如何变成一串整数
2. Embedding（嵌入）：这些整数如何获得"意义"
3. 位置编码：模型如何感知 token 的顺序
4. 注意力机制：token 之间如何交换信息
5. 多头注意力：模型如何同时追踪多种关系
6. 前馈网络：模型大量知识结构的"存储库"
7. 残差流与层归一化：支撑超深网络可训练的关键
8. 预测下一个 token：模型的输出形式及文本生成循环
9. 架构与训练权重的区别：现代大模型的共性与差异

![Transformer pipeline from tokenization to next-token prediction|324](https://www.0xkato.xyz/assets/transformer-pipeline.png)

文中穿插有"小知识卡"，便于各种背景的读者理解。

---

## 分词（Tokenization）

模型并不直接处理文本，而是处理整数 ID。将文本转换为整数序列的过程叫做分词（tokenization）。

分词器接受字符串，输出一串整数，每个整数对应词表中的一个条目。现代 LLM 的词表通常包含数万到数十万个条目。

> **小知识：token ID**
> token ID 是模型用来代表一个词表条目的整数。模型操作的是数字，不是文字本身。

token 通常不是完整单词，而是"子词片段"。比如"tokenization"可能被切分成 ["token", "ization"]，"running"变成 ["run", "ning"]。这样设计的原因是效率：整词词表规模过大且无法泛化到新词；纯字符级词表粒度过细，迫使模型从头学习最基础的模式。子词分词法折中两者——高频片段拥有独立 token，生僻词或新词则由更小的单元组合而成。

> **小知识：词表**
> 词表是分词器的固定片段集合。每个片段有一个 ID，模型只接收词表里的 ID。

这种设计有时会产生出乎意料的问题。经典例子：问 LLM "strawberry" 这个词里有几个 R。早期 LLM 经常答错——不是模型不会数数，而是它操作的不是字母，只是 token ID。分词方式可能把多个字母合并进同一个 token，根本不是按字母粒度处理的。

![Tokenization turns text into token IDs|177](https://www.0xkato.xyz/assets/transformer-tokenization.png)

不同模型家族使用不同的分词器。GPT 系列使用 BPE（字节对编码）变体；LLaMA 系列用 SentencePiece。选择会影响推理效率（token 数越少越快）以及多语言覆盖等，但基本流程相同：文本进，整数出。

拿到整数序列后，下一步是让这些整数拥有实际"意义"。

---

## 嵌入（Embedding）

token ID（比如 1024）本身只是一个索引，没有任何含义。赋予它含义的是一张巨大的查找表：嵌入矩阵（Embedding Matrix）。

每个模型都有一个嵌入矩阵，行数等于词表大小，每行是一个长向量。行的长度就是模型的隐层维度（hidden size）。以 7B 参数量级的模型为例，每个 token 对应 4096 个数值；更大的模型通常向量维度更宽。

> **小知识：向量**
> 向量是一串数字。在 Transformer 中，每个 token 都会被表示为一个向量，以便进行数学运算。

分词器输出一个整数，模型就查表取出对应行的向量——这就是该 token 的 embedding，是模型训练过程中学到的对该 token"含义"的表征。

> **小知识：嵌入矩阵**
> 嵌入矩阵是一张查找表：token ID 进，学到的向量出。

这些嵌入有一个有趣的特性：语义相近的 token，其向量在空间中也相互靠近。"king"与"queen"是近邻，"Paris"与"France"也是近邻。这不是人为硬编码的，而是模型为了提高文本预测准确率，在训练过程中自发形成的。

甚至可以在嵌入空间里做"算术"，比如著名的 `king − man + woman ≈ queen`。嵌入空间的几何结构确实承载了语义关系，尽管没有人明确要求模型这样组织。

![Embedding space analogy with semantic relationships|331](https://www.0xkato.xyz/assets/transformer-embedding-analogy.png)

需要注意：此时每个 token 已被替换为其 embedding 向量，但该向量并不包含 token 在序列中的位置信息。无论"dog"出现在序列的第一位还是第五位，对应的向量完全相同——序列顺序的信息仍然缺失。

这正是位置编码要解决的问题。

---

## 位置编码（Positional Encoding）

自注意力本身没有内置的词序表示。没有位置信号，模型无法直接判断"dog"是在"bites"之前还是之后。

词序会改变含义，因此需要一种机制将每个 token 的位置信息注入到计算过程中。

> **小知识：位置编码**
> 位置编码是将序列顺序信息告知模型的机制，让模型知道每个 token 在序列中的位置。

原始 Transformer（Vaswani 等，2017）的做法是：为每个位置生成一组特定的数值模式，并在所有处理开始前直接加到对应 token 的 embedding 上。这些模式来自不同频率的正弦余弦函数。这样，"dog"在位置 1 的 embedding 和在位置 5 的 embedding，就因为叠加了不同的位置模式而有所不同。

这个方案有效，正弦编码还有一定的外推能力（能泛化到训练中未见过的更长序列）。但随着模型规模扩大，加法式位置方案暴露出两个问题：

第一，同一个向量需要同时承载"含义"和"位置"，表达容量有限。

第二，学习得到的绝对位置 embedding 泛化性有限。若训练时见过的最长序列是 2048 个 token，模型就从未见过位置 5000，该位置的 embedding 根本没有被充分学习。

现代模型大多采用一种名为旋转位置编码（RoPE, Rotary Position Embeddings）的方案，由 Su 等于 2021 年提出，目前已被 LLaMA、Mistral、Gemma、Qwen 等主流开源模型广泛采用。其核心思想是：不再把位置信息加到 token 向量上，而是在 attention 阶段对 Query 和 Key 向量施加一个与位置相关的旋转角度——位置 1 对应小幅旋转，位置 100 对应更大的旋转角。当两个 token 在 attention 中做比较时，起作用的是两者旋转角度之差，这天然编码了它们之间的相对距离。

> **小知识：RoPE**
> RoPE 即旋转位置编码。不叠加位置向量，而是对 Query 和 Key 做带有位置信息的旋转，使相对距离在 attention 计算中自然呈现。

![Rotary position embeddings rotate vectors by position|330](https://www.0xkato.xyz/assets/transformer-rope.png)

RoPE 有几个实际优势：自然编码相对位置（这正是 attention 所需要的）、更好地泛化到更长文本，且不需要额外的模型参数。

即使有了良好的位置编码，现代 LLM 仍存在"迷失在中间"（lost in the middle，Liu 等，2023）的问题——对长文本首部和尾部的信息利用更为可靠，埋在中间的信息则相对容易被忽视。这也是工程上建议"重要信息放前面"或"结尾重复关键内容"的底层原因，并非玄学，而是由模型机制决定的。

位置与含义都编码完毕后，接下来的问题是：token 之间如何实际交换信息？

---

## 注意力机制（Attention）

这正是 Transformer 得名的来源——注意力（attention）。

每个 Transformer 层内，注意力机制只做一件事：让每个 token 观察它被允许看到的其他 token，并判断哪些 token 对当前的预测最重要。

具体做法是赋予每个 token 三重角色，分别通过三个线性变换得到三个新向量：Query（Q）、Key（K）、Value（V）。

> **小知识：Q/K/V**
> Query 是"我在寻找什么"，Key 是"我能与什么匹配"，Value 是"匹配成功时传递的信息"。

- Query：希望从其他 token 获取什么信息
- Key：能为其他 token 提供什么特征
- Value：当有人来匹配时，会传递出去的信息内容

同一个 token 同时扮演三种角色。Q、K、V 的变换矩阵全部在训练过程中学习，不是手工指定的。

匹配通过相似度分数来完成。每个 token 的 Query 与其被允许看到的每个 token 的 Key 做缩放点积运算，直觉上衡量的是两个向量的对齐程度。缩放是为了保持数值稳定，防止后续 softmax 出现异常。

> **小知识：点积**
> 点积是衡量两个向量对齐程度的基本运算，数值越大代表越相关。

匹配分数经 softmax 转换为权重，所有权重之和为 1。分数高的 token 对应更高的权重，最终对 Value 向量做加权平均。

> **小知识：softmax**
> softmax 将一组原始分数转换为总和为 1 的权重，大分数占主导，小分数被压缩。

举例说明：句子 "The cat that I saw yesterday was sleeping."，当模型处理"was"时，需要判断是谁在"睡觉"。"was"的 Query 向量与序列中各 token 的 Key 向量做点积，"cat"的匹配分数最高（模型已学会：动词"was"需要主语，而"cat"这类名词产生的 Key 向量恰好与此对齐），"yesterday"的分数最低。softmax 后，"cat"对应 Value 向量的权重占主导，"was"的新表征因此主要由"cat"的信息塑造。这就是模型能在隔了若干位置之后仍找到正确指代的机制。

GPT 类语言模型从左到右逐 token 生成文本，位置 5 的 token 只能看到位置 1 至 5，不能访问尚未生成的后续 token。这通过因果遮罩（causal masking）实现：未来 token 的匹配分数被设为极小值，经 softmax 后权重近乎为零，无法参与计算。

> **小知识：因果遮罩**
> 因果遮罩屏蔽未来位置的 token，使 decoder-only 语言模型在预测下一个 token 时无法"提前看"。

![Attention heatmap showing causal masking and high attention to cat|356](https://www.0xkato.xyz/assets/transformer-attention-heatmap.png)

可解释性研究中最有趣的发现之一是"归纳头"（induction head，Anthropic，2022）：某些 attention head 专门学会识别序列中形如 "A B … A" 的模式，在第二次遇到 A 时，会回头查找第一次 A 后面跟的是什么，并将其延续出来。这是 LLM "上下文学习"能力目前最清晰的已知机制之一。

> **小知识：归纳头**
> 归纳头是一类 attention head，专门检测序列中的重复模式，帮助模型将其延续下去。

注意力机制的主要计算代价在于：每个 token 需要与它能看到的所有 token 逐一比较，序列长度翻倍，计算量大约增至四倍。这也是长 prompt 推理成本高昂的原因，也催生了大量提升注意力效率的研究（FlashAttention、稀疏注意力、线性注意力等）。

但单次 attention 只能为模型提供一种观察 token 关系的视角，远不够用。

---

## 多头注意力（Multi-head Attention）

单次 attention 只能捕捉一类 token 关系，而自然语言中同时存在主谓、指代、长程依赖、局部短语等多种关系。多头注意力的解决方案是：并行运行多组 attention，每组在各自专属的较小空间内独立运作，每组称为一个"头"（head）。

> **小知识：attention head**
> 一个 attention head 就是一次独立的注意力运算，拥有自己的学习投影矩阵。

常见的误解（很多教程也会犯）：每个 head 并不是简单地"切分"原始 token 向量。每个 head 有自己独立的变换矩阵，将完整 token 向量映射到自己专属的较小维度空间。比如模型每个 token 有 4096 维，共 32 个 head，表面上每个 head 负责 128 维，但实际上这 128 维是对完整 4096 维向量的一次学习投影，而非固定分段。每个 head 观察的是"同一 token 的不同视角"，而不是"token 的不同分块"。

每个 head 独立完成注意力运算，所有 head 的输出拼接在一起，再经过一个线性变换混合回完整尺寸的向量。这个混合层也是训练得到的。

![Multi-head attention combines specialized attention heads|298](https://www.0xkato.xyz/assets/transformer-multi-head-attention.png)

有趣的是，不同 head 在训练后往往呈现出部分功能分化，尽管没有人为规定各 head 的职责。研究者发现了负责语法配对（动词-宾语、冠词-名词）的 head、处理代词指代的 head、追踪位置模式的 head，以及归纳头等更多类型。一个 Transformer 层可能有 32 个 head，几十层叠加后一个典型 LLM 共有数千个 attention head，每个都贡献一个独立学习到的视角。

与多头注意力密切相关的工程问题是 KV 缓存（KV Cache）：文本生成过程中，每个 head 的 Key 和 Value 向量需要为所有已生成 token 保留在内存中，这样在生成新 token 时就不必从头重新计算。KV 缓存是长文本推理内存开销的主要来源。

> **小知识：KV 缓存**
> KV 缓存在生成过程中保存历史 token 的 Key 和 Value 向量，避免每次生成新 token 时重新计算整个前缀。

现代 decoder-only LLM 大多采用"分组查询注意力"（Grouped-Query Attention, GQA）：多个 query head 共用同一组 key 和 value head，而非每个 head 各自独立。LLaMA-2 70B 有 64 个 query head，但只有 8 个 KV head；Mistral 7B 有 32 个 query head，8 个 KV head。效果与完整多头注意力相近，但 KV 缓存占用和推理成本大幅降低。

> **小知识：GQA**
> GQA 让多个 query head 共享更少的 key/value head，在保留多角度 query 的同时显著减少 KV 缓存开销。

---

## 前馈网络（Feed-forward Network）

注意力机制完成 token 间信息混合之后，每一层还有一个常被忽视的步骤：前馈网络（FFN）。

如果说注意力机制处理的是 token 之间的信息流动，那么前馈网络处理的是每个 token 自身的进一步变换——对每个 token 的向量单独运算，互不干扰。

FFN 的处理流程分三步：

1. 将 token 向量扩展到更高维度（原始 Transformer 扩展 4 倍，现代采用 SwiGLU 的模型扩展比例有所不同）
2. 施加一个非线性变换
3. 压缩回原始维度

![Feed-forward network expands, transforms, and compresses each token vector|371](https://www.0xkato.xyz/assets/transformer-ffn.png)

中间的非线性步骤值得专门理解。非线性函数会对输入进行"弯曲"处理。最简单的 ReLU 对负数输出零，对正数原样通过。

> **小知识：非线性**
> 非线性防止网络退化为等效的单层线性变换——没有非线性，无论堆叠多少层，整体计算都等价于一次矩阵乘法。

若全部使用线性层，无论堆叠多少层，数学上都等价于单个线性变换。非线性使 FFN 能够学习远超单个矩阵所能表达的复杂结构。

非线性函数的选择经历了演进：原始 Transformer 用 ReLU，GPT 和 BERT 改用 GELU，现代的 LLaMA、Mistral、PaLM 等采用 SwiGLU。扩展-压缩的结构保持不变，非线性本身在持续迭代。

值得注意：在稠密 Transformer 模型中，大部分参数实际上在 FFN 里，而非注意力子模块中。

这些参数并非通用的随机权重——模型中大量存储的事实和语义结构就分布在 FFN 里。研究者发现，FFN 中的某些神经元与特定概念或事实高度关联：有的神经元遇到埃菲尔铁塔相关文本就会强烈激活，有的对编程语言敏感，有的专门响应过去式动词。当模型"知道"巴黎是法国首都时，这个知识就分布在特定层的 FFN 权重与激活模式之中。

FFN 的这一"记忆存储"特性带来了一个有趣的现象：研究者甚至可以在不重新训练模型的情况下，直接修改模型中存储的部分事实。例如，ROME（Rank-One Model Editing）方法，通过对某一层 FFN 权重矩阵进行针对性的低秩编辑，可以将"埃菲尔铁塔在巴黎"改为"埃菲尔铁塔在罗马"，此后模型生成的内容也会反映出修改后的关联关系。

一些前沿大模型开始用"专家混合"（Mixture of Experts, MoE）结构替换稠密 FFN。每层不再只有一个前馈网络，而是有多组并行的 FFN（称为"专家"），再由一个小型路由器动态决定每个 token 交给哪些专家处理。以 Mixtral 8x7B 为例，每层有 8 个专家，每个 token 只激活其中 2 个。总参数量大幅增加，但单 token 的实际推理计算量增长幅度远小于参数量的增长——实现了参数规模与推理成本的部分解耦。

> **小知识：MoE**
> Mixture of Experts（专家混合）即每层有多组 FFN，每个 token 只经过其中少数几个专家处理。

Mixtral 8x7B 总参数约 467 亿，但每 token 实际使用约 129 亿参数。MoE 已成为超大模型常见的架构选择，使参数总量能独立于推理成本增长。

---

## 残差流与层归一化（Residual Stream & Layer Normalization）

残差流使模型的计算变成"累加"而非"覆盖"。每次 attention 或 FFN 处理后，输出通常不会直接替换原有的 token 向量，而是与原向量相加。新向量 = 旧向量 + 子模块输出。

> **小知识：残差连接**
> 残差连接将子模块的输出加回其输入向量，为信息和梯度在网络中提供了一条直通捷径。

在三十层、五十层乃至百层的堆叠中，每一层的贡献都会累积进去，而不是简单覆盖上一层的结果。这种持续累加的过程称为"残差流"。其中值得关注的特性是：最初的输入 embedding 通过加法通路，仍然可以直接影响到很深的后续层，与每个子模块的贡献混合在一起。

![Residual stream accumulates attention and feed-forward outputs|234](https://www.0xkato.xyz/assets/transformer-residual-stream.png)

残差连接并非 Transformer 的发明，最早出现在 ResNet（He 等，2015）图像识别网络中，最初是为了解决深层网络训练失败的问题。随着网络层数加深，训练信号回传时会逐渐消失（或有时爆炸），导致模型无从有效学习。引入直通捷径后，信号可以不经过所有层直接回传，使训练上百层的深网成为可能。Transformer 直接继承了这一设计。

在现代可解释性研究中，残差流已成为理解网络内部工作机制的核心对象。每一个组件——每个 attention head、每个 FFN，乃至最后的 unembedding 步骤——都从残差流中读取，再将结果写回。

层归一化（Layer Normalization）的存在出于更直接的工程需求。若没有归一化，残差流中持续累加的数值往往会急剧增大或骤然归零，导致训练失败。层归一化的作用是将每个 token 向量的数值规范到稳定的范围内。

> **小知识：层归一化**
> 层归一化对 token 向量的数值进行重新缩放，使训练过程中的数值始终保持在稳定范围内。

最初的 Transformer（2017）在每个子模块之后做归一化（post-norm）。这种方式在浅层网络中有效，但随着深度增加，训练稳定性下降。现代 Transformer（GPT-2 起、LLaMA、Mistral 等）普遍采用在每个子模块之前做归一化（pre-norm）的方式，使超深网络的训练更加可靠。

归一化的具体实现也在演进。许多现代开源模型（LLaMA、Mistral、Gemma、Phi 等）采用更简洁的 RMSNorm。原始 LayerNorm 做两件事：先减均值，再做缩放；RMSNorm 去掉减均值步骤，只保留缩放。实验结果表明主要收益来自缩放，省去减均值的步骤使计算更高效。

> **小知识：RMSNorm**
> RMSNorm 是一种更高效的归一化方式，只做尺度缩放，不做均值减除。

这些并不华丽但至关重要的基础设施，决定了模型能否顺利扩展到极深规模。没有残差连接，深层模型训练极为困难；没有归一化，数值会迅速爆炸或消失。两者兼备，才能支撑起百层乃至更深的现代大模型。

---

## 预测下一个 token

所有注意力与 FFN 层处理完毕后，序列中的每个 token 都有一个最终向量。生成阶段，模型只取序列最后一个 token 的向量来预测下一个词。

这个最终向量被转换为词表中每个可能的下一个 token 对应的一个分数。如果词表有 10 万个 token，就得到 10 万个数值。这些原始分数称为 logit，它们还不是概率，可以是任意正负值。

> **小知识：logit**
> logit 是每个候选下一个 token 的原始分数，经 softmax 后才转化为概率。

softmax 将 logit 转换为下一个 token 的概率分布，运算方式与 attention 中相同。

生成 token 时，模型并不总是选择概率最大的那一个。解码参数控制输出的确定性与多样性：temperature 调整概率分布的锐度，top-k 和 top-p 将候选范围限定在最可能的 token 之中。因此同一个模型可以在不同设置下表现出精确或富有创造力的两种风格。

> **小知识：temperature**
> temperature 控制采样的随机程度。低 temperature 使模型更保守，高 temperature 使输出更多样。

新 token 选定后，加入输入序列，通常复用 KV 缓存避免重新计算前缀。随后运行新 token 的 attention 和 FFN，得到新的最终向量，预测再下一个 token。如此循环，直到模型输出序列结束符或达到长度上限。整段文字就是逐步在这个循环中生成的。

"预测下一个 token"是 base LLM 的全部训练目标。base 模型不直接针对事实准确性、对话能力、推理能力或编程能力训练，只是纯粹最大化对下一个 token 的预测准确率。指令跟随、偏好对齐、安全性和对话能力，都是后续后训练阶段的产物。

近年出现了一项值得关注的效率创新——投机解码（speculative decoding）：一个小型快速模型预先提出若干候选 token，大模型并行验证这些候选。若候选被大模型接受，则批量采纳；否则回退到大模型自行生成。正确实施时，输出分布与单独运行大模型完全一致，但整个循环速度大幅提升。

> **小知识：投机解码**
> 投机解码让小型草稿模型超前猜测若干 token，再让大模型并行验证，在保持输出质量的同时显著提速。

逐 token 预测循环是整个架构中最简洁的部分，也是驱动一切运转的核心。

---

## 架构与训练权重的区别

我们已经梳理了核心机制：token、嵌入、位置编码、注意力与多头注意力、前馈网络、残差流与归一化，以及输出侧的逐 token 预测循环。这是对完整架构的一次通览。

那么 GPT、Claude、Gemini、LLaMA 之间到底有何本质不同？各家公开的细节不尽相同，专有模型也不会公布所有架构选择。但在本文所涉及的层级上，它们大体都处于同一个 Transformer 家族的设计空间之中。

当前主流的基于 Transformer 的 LLM 共享相同的宏观结构：分词、嵌入、位置编码、堆叠的 Transformer 层（每层包含多头注意力与前馈网络）、残差流、层归一化、下一个 token 预测。

模型之间的差异集中在：

1. 训练得到的权重本身——来自不同规模的不同训练数据
2. 配置参数——层数、词表大小、head 数、参数总量、MoE 还是稠密结构
3. 后训练内容——指令微调、人类反馈偏好训练、安全控制等施加在 base 模型之上

> **小知识：权重**
> 权重是模型内部所有学习得到的数值。训练过程不断调整这些数值，直到模型能够准确预测文本。

2023—2025 年间，"现代 Transformer"逐渐收敛到一套共识选择，尽管不同团队各自独立摸索到了相似结论：前归一化（pre-norm）、RMSNorm、RoPE、SwiGLU、分组查询注意力，以及部分最大模型使用的 MoE。这些并非一蹴而就，而是在原始 2017 年设计基础上历经约五年持续改进的积累。

---

## 展望

Transformer 家族架构在机器学习历史上实属罕见地完成了跨领域统一。此前，不同问题各有专属的网络架构：图像识别一套，语言处理一套，音频处理又是另一套，各领域几乎不共享方法。

如今 Transformer 已渗透语言、视觉、音频和多模态系统，吸纳了该领域的大部分。

这种格局未必会永远持续。Mamba 和其他状态空间模型在超长序列处理上已是可信的替代架构；混合架构正在探索之中；MoE 本身已经从根本上改写了"前沿标准架构"的定义——五年前这还被视为异想天开。

但无论架构如何演变，本文所介绍的这些核心机制（token、嵌入、位置编码、注意力、前馈网络、残差流与归一化、逐 token 预测）都是持久的底层基础。即便算法迭代，任何序列模型都必须以某种形式解决这些相同的问题。

读完本文，应该能够读懂现代 Transformer 论文或模型卡，理解其中每个架构术语背后的真实含义。这正是这篇文章的目标。
