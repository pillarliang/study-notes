---
title: agentLoop：无状态循环引擎
aliases:
  - Pi agentLoop
  - agentLoop 无状态循环引擎
  - 04-agentLoop-无状态循环引擎
tags: [Pi, agent-loop, runtime, source-code]
source_repository: earendil-works/pi
source_commit: 534bcbff
related:
  - "[[02-Agent-Loop-模块交互]]"
  - "[[04-tool-execution-三阶段管道]]"
  - "[[05-Agent-有状态运行时外壳]]"
---

# `agentLoop`：无状态循环引擎

> [!summary] 核心职责
> `agentLoop` 接收已经准备好的 context 和策略回调，推进一次 agent run：调用 LLM、处理 assistant、执行工具批次、写回结果并决定是否继续。它不拥有 session 持久化，也不负责产品层的 retry 与 compaction。

本文以 `earendil-works/pi` commit `534bcbff` 为基线。具体行为仍应按目标版本源码复核。

---

## 0. 本文的对象边界

message、context、loop turn、agent run 与 session 的统一定义见 [[00-Agent-Harness-知识全景图#2. 六个术语必须分开|00-Agent-Harness-知识全景图 §2「六个术语必须分开」]]。

本文只需再固定一个职责结论：`agentLoop` 只拥有**单个 agent run** 的局部控制状态；一条用户 prompt 可由产品层启动多个 run，而每个 run 又可包含多个 turn。

---

## 1. “无状态”到底是什么意思

“无状态”不是说函数运行时没有变量，也不是保证绝不修改传入对象。它表示：

- 不从 session 文件读取历史；
- 不把结果写入长期存储；
- 不在两次 run 之间保留必须复用的 loop 实例；
- 不自行决定产品层 retry 或 compaction；
- 本次 run 所需的 model、messages、tools 与回调都由调用方提供。

循环内部仍要保存当前 context、pending messages、最新 assistant、tool batch 与停止判断。这些状态只服务当前 run，run 结束后即失去生命周期。

跨 run transcript、队列和 active-run 锁属于 [[05-Agent-有状态运行时外壳|Agent]]；JSONL 会话树属于产品层 `SessionManager`。

---

## 2. 一个 turn 的完整边界

正常完成的 turn 按以下顺序推进：

```text
turn_start
  → 合入本轮 pending messages
  → transformContext
  → convertToLlm
  → 调用一次 StreamFunction
  → assistant 流式生成并最终定型
  → 若有可处理的 tool call，执行 tool batch
  → 将实际产生的 toolResult 加入工作消息
  → turn_end
```

一个 turn 的稳定判断是：

```text
1 次 LLM 请求
+ 1 条最终 assistant message
+ 0 到 N 条本批实际产生的 toolResult
```

这里不能写成“模型请求的每个 tool call 必有结果”。在取消路径中，尚未启动的调用可能既没有执行事件，也没有结果消息。正常、未中途取消的批次才满足请求与结果完整对应。

如果 assistant 因输出长度限制而截断，tool 参数可能看似合法但语义不完整。Pi 不执行这批调用，而是对已识别的调用生成错误结果，让模型在后续 turn 重新请求。

---

## 3. 双层循环为什么存在

`agentLoop` 同时处理两种排队语义：

- `steering`：当前任务未结束，下一步尽快改变方向；
- `follow-up`：当前任务完成后，再追加一项工作。

简化后的结构如下：

```javascript
while (true) {                          // 外层：follow-up
  let pending = initialMessages
  let continueFromTools = true

  while (continueFromTools || pending.length > 0) { // 内层：当前任务
    append(pending)
    const assistant = await streamAssistant()

    if (assistant.stopReason === "error" ||
        assistant.stopReason === "aborted") break

    const batch = await executeRequestedTools(assistant)
    continueFromTools = !batch.terminate

    await prepareNextTurn()
    if (await shouldStopAfterTurn()) break

    pending = await getSteeringMessages()
  }

  const followUp = await getFollowUpMessages()
  if (followUp.length === 0) break
  initialMessages = followUp
}
```

这是心智模型，不是逐行源码抄写。关键时机是：

1. 当前 assistant 与本批工具先闭合；
2. turn 边界允许准备下一轮；
3. 上层强制停止检查早于队列消费；
4. `steering` 在内层循环的安全边界进入；
5. 只有当前任务本可结束时才读取 `follow-up`。

因此 `steering` 不是“立即打断”，`follow-up` 也不是“下一次用户 prompt”。产品层的 `nextTurn` 更晚，见 [[02-Agent-Loop-模块交互#5. 三种排队消息的时机|02-Agent-Loop-模块交互 §5「三种排队消息的时机」]]。

---

## 4. 续轮与停止条件

### 4.1 模型不再请求工具

assistant 只给最终文本、没有 steering 或 follow-up 时，循环自然结束。

### 4.2 工具结果要求继续

普通 toolResult 需要再次交给 LLM 解释或继续决策，因此会驱动下一个 turn。

### 4.3 整批结果都设置 `terminate`

只有本批**实际产生了结果**，且每个最终结果都明确设置 `terminate: true`，该批才不再仅因 toolResult 自动调用 LLM。

`terminate` 不是“强制关闭整个 agent run”。若仍有 steering 或 follow-up，运行仍可能继续。聚合细节见 [[04-tool-execution-三阶段管道#7. `terminate` 的精确含义|04-tool-execution-三阶段管道 §7「terminate 的精确含义」]]。

### 4.4 `shouldStopAfterTurn`

上层可在当前 assistant 与工具批次完成后要求停止。该判断不会留下执行到一半的 tool；一旦停止，本次 run 不再消费其后的 steering 或 follow-up。

### 4.5 模型错误或取消

`error` 与 `aborted` 会结束当前 run。`agentLoop` 不自行 backoff、换模型或 retry；产品层决定是否再次启动 run。

### 4.6 没有内建最大 turn 数

需要 turn budget 时，调用方必须计数，并通过停止回调实现上限。

---

## 5. 内部消息如何成为 LLM 输入

Pi 在 agent 层使用可扩展的 `AgentMessage`，在模型协议层使用标准 `Message`。每次 LLM 调用前经过两步：

```text
AgentMessage[]
  → transformContext(AgentMessage[])
  → convertToLlm(AgentMessage[])
  → Message[]
  → StreamFunction
```

### 5.1 `transformContext`

回答“本次请求应该看到哪些内部消息”。它可以裁剪、重排、补充资料或用摘要替换历史投影。返回的新数组只影响本次模型视图，不会自动改写 session 事实。

### 5.2 `convertToLlm`

回答“每种内部消息如何转换成模型协议接受的消息”。UI-only 消息可被过滤，自定义摘要消息可被展开为标准 user/assistant 内容。

顺序不能颠倒：一旦先转成标准 `Message`，自定义消息携带的产品语义可能已经丢失，后续就无法据此治理 context。

---

## 6. 模型访问由 `StreamFunction` 注入

`agentLoop` 不直接 import Anthropic、OpenAI 或 Google SDK。它只调用一个满足 `StreamFunction` 契约的具体函数：

```text
agentLoop
  → StreamFunction(model, context, options)
  → Models 根据 model.provider 路由
  → Provider 处理认证与 wire protocol
  → 统一 AssistantMessageEventStream
```

需要区分类型与实现：

- `StreamFunction` 是 TypeScript 函数类型；
- 具体实现才真正访问 provider；
- 测试可以传入不访问网络的 fake stream；
- 请求/model/runtime 失败应编码为流中的 error/aborted 终止，而不是在函数入口直接抛出。

循环每次模型调用前可重新获取认证材料，因此长时间工具执行后无需继续持有已过期凭证。

---

## 7. turn 边界提供哪些控制点

### 7.1 准备下一轮

上层可在 turn 完成后、下一次 LLM 请求前更新 context、model 或 thinking level。已经完成的 assistant 与工具副作用不会被反向修改。

信息较少的回调只依据外部状态决定更新；信息较完整的版本还能读取刚结束的 assistant 与 toolResults。若两者同时存在，`Agent` 采用信息更完整的路径。

### 7.2 停止判断

`shouldStopAfterTurn` 让调用方在 turn 安全闭合后停止 run。

### 7.3 消息队列

`getSteeringMessages` 与 `getFollowUpMessages` 在固定时点取消息，避免任意时刻直接改写 context 造成顺序破坏。

### 7.4 工具 hook

`beforeToolCall` 与 `afterToolCall` 属于 tool pipeline 的控制点。loop 只调用它们；错误、参数和结果语义见 [[04-tool-execution-三阶段管道]]。

消息转换、凭证获取、停止判断和队列读取等系统回调应返回安全结果，不应抛异常。tool hook 则由工具管道捕获并转换成错误结果。

---

## 8. 事件与消息是两种输出

### 8.1 事件描述过程

典型顺序为：

```text
agent_start
  → turn_start
  → message_start / message_update* / message_end
  → tool_execution_start / update* / end
  → toolResult message events
  → turn_end
  → ...
  → agent_end
```

并行工具的 execution 事件可按完成时间交错。

### 8.2 消息描述结果

user、assistant 与实际产生的 toolResult 组成本次 run 的新增消息。它们供 `Agent` 更新 transcript，也可由产品层持久化；流式 delta 不进入 session 事实。

### 8.3 闭合保证取决于调用层

正常路径和模型返回的 error/aborted 会由 loop 发出结束事件。若系统策略回调意外抛异常，直接调用底层 loop 时不保证自行补齐所有事件。

通过 `Agent` 调用时，外壳有 rejection 安全网：把未捕获异常转换成失败 assistant，并尽量发出闭合事件。这个安全网属于 `Agent`，不能无条件归因于 `agentLoop`。

---

## 9. 不属于本篇的机制

- tool 的三阶段管道、批并发和取消：[[04-tool-execution-三阶段管道]]
- transcript、listener、idle、互斥和 abort API：[[05-Agent-有状态运行时外壳]]
- session tree 与 compaction：[[official-doc/06-Sessions-会话树]]、[[official-doc/07-Compaction-上下文压缩]]
- 产品层 retry、`nextTurn` 与 prompt settled：[[02-Agent-Loop-模块交互]]

---

## 10. 源码入口

- `packages/agent/src/agent-loop.ts`：run 主体、turn、队列与停止判断。
- `packages/agent/src/types.ts`：context、message、tool、event 与 callback 契约。
- `packages/agent/src/stream-fn.ts`：默认 stream function 的注册入口。
- `packages/agent/src/agent.ts`：有状态外壳如何装配并调用 loop。
