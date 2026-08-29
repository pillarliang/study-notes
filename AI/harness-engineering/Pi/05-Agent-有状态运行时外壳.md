---
title: Agent：有状态运行时外壳
aliases:
  - Pi Agent 有状态运行时外壳
  - Agent stateful wrapper
  - 06-Agent-有状态运行时外壳
tags: [Pi, agent, runtime, source-code]
source_repository: earendil-works/pi
source_commit: 534bcbff
related:
  - "[[03-agentLoop-无状态循环引擎]]"
  - "[[04-tool-execution-三阶段管道]]"
---

# `Agent`：有状态运行时外壳

> [!summary] 核心职责
> `Agent` 包在无状态 `agentLoop` 外面，保存跨 run 继续使用的内存状态，保证同一实例只有一个 active run，并通过事件把 loop 结果归约回 state。它不是 JSONL session 系统，也不负责产品层资源发现、retry 或 compaction。

本文以 `earendil-works/pi` commit `534bcbff` 为基线。

---

## 0. `Agent` 在哪一层

```text
AgentSession〔产品层〕
└── Agent〔有状态运行外壳〕
    └── agentLoop〔单次 run 控制流〕
        └── tool execution〔单批副作用管道〕
```

`Agent` 连接多次 run，但不等于长期 session。它保存当前工作状态；`SessionManager` 才保存可恢复的会话事实。

---

## 1. `Agent` 拥有什么状态

### 跨 run transcript

`state.messages` 保存当前 user、assistant、toolResult 与可扩展 AgentMessage。它是内存 transcript，不是第三个独立于 messages 的容器。

### 运行配置

保存当前 system prompt、model、thinking level、tools、stream function 与策略回调。配置来源和选择逻辑由调用方决定。

### 当前进行中的状态

包括 streaming message、pending tool call IDs 与 error message。它们由事件归约更新，供 UI 或 listener 读取。

### 控制状态

包括 active run、AbortController、steering queue、follow-up queue 和 listeners。

---

## 2. `Agent` 明确不负责什么

以下责任属于 `AgentSession`、`SessionManager` 或宿主：

- 打开、保存、分支和恢复 JSONL session；
- 发现 settings、extensions、skills、prompts 与 themes；
- 选择模型和构造 system prompt；
- 决定 retry 与 compaction；
- 提供具体 coding tools；
- 渲染 UI 或传输 RPC/JSON 事件。

因此，`Agent` 是 stateful runtime wrapper，不是完整产品 runtime。

---

## 3. 一次 `prompt()` 怎样连接状态与 loop

假设 `Agent` 已有 transcript，收到一条新 user message：

```text
检查当前没有 active run
  → 建立 active run 与 AbortController
  → 用 Agent state 准备 context 和 AgentLoopConfig
  → agentLoop 推进一个或多个 loop turn
  → loop 逐个发出 AgentEvent
  → Agent 先归约 state，再通知 listeners
  → agentLoop 发出 agent_end
  → listeners 处理完 agent_end
  → 清理 active run
  → Agent 进入 idle
```

这里有两个消息视图：

- `state.messages`：跨 run 保存的 transcript；
- `context.messages`：本次 agentLoop 使用的工作数组。

运行开始后，外部直接修改 `Agent` state 不会自动改变已经进行到一半的 context。当前 run 只能通过已定义的 turn 边界、队列或取消信号受到控制。

---

## 4. loop 事件统一归约事件派生状态

active run、busy/idle 等生命周期状态还会在启动与清理方法中直接更新。这里讨论的是 transcript、streaming message、pending tool calls 与 error 等**由 loop 事件派生的状态**；它们统一按下面顺序归约：

```text
agentLoop emit(event)
  → Agent 根据 event 更新 state
  → 按订阅顺序 await listeners
```

典型归约包括：

- `message_start` / `message_update`：更新 streaming message；
- `message_end`：清除 streaming 状态，并把 finalized message 加入 transcript；
- `tool_execution_start`：把 tool call 标记为 pending；
- `tool_execution_end`：移除 pending 标记；
- `turn_end`：记录 assistant error；
- `agent_end`：结束本次 loop 的 streaming 状态。

因此 listener 收到 `message_end` 时，对应 message 已经存在于 `state.messages`；收到 tool start/end 时，pending 集合也已反映该事件。

---

## 5. listener 属于运行路径

listeners 不是 fire-and-forget 通知。`Agent` 在状态归约后，按订阅顺序逐个 await：

```text
更新 state
  → listener A 完成
  → listener B 完成
  → listener C 完成
  → 当前事件处理完成
```

这提供确定的单事件观察顺序，也产生背压：慢 listener 会延长运行。并行工具可能让不同工具的事件按完成时间交错，但每个事件内部仍遵守“先归约、后通知”。

持久化、UI 或日志可以由 listener 驱动；是否真的写盘取决于使用 `Agent` 的产品层，不能因此说 `Agent` 自己拥有 session 存储。

---

## 6. 四个生命周期时点

### 6.1 active run

从 `prompt()` 或 `continue()` 成功启动，到所有 loop 事件与 listener 处理完成。此时 Agent 为 busy。

### 6.2 `agent_end`

当前 `agentLoop` invocation 的终止事件。它表示 loop 不再为本次 run 产生新的正常运行事件，但 listener 仍在处理，产品层也可能随后启动另一个 run。

### 6.3 idle

所有 listeners 已处理完 `agent_end`，active run 已清理。此时才可以安全启动下一次 run 或 reset。

### 6.4 prompt settled

这是更高层 `AgentSession` 的概念：retry、compaction 与 continuation 都不再自动启动。它不属于 `Agent` 自身的 idle 定义。

---

## 7. `waitForIdle()` 的完成语义

`waitForIdle()` 是 `Agent` 的异步等待方法：

```ts
await agent.waitForIdle()
// 当前调用时已经存在的 active run 及其 listeners 已完成
```

- 调用时正在运行：等待该 active run 进入 idle；
- 调用时已经 idle：立即完成；
- 不等待未来才启动的新 run。

`await agent.prompt(...)` 本身也会等待它启动的 run 及 listeners 完成。`waitForIdle()` 主要供没有发起当前 prompt、但需要等待它结束的其他调用者使用。

---

## 8. 同一 `Agent` 只允许一个 active run

`Agent` 从启动 run 到回到 idle 之间拒绝再次调用 `prompt()` 或 `continue()`。这项互斥避免两个 loop 同时改写 transcript、streaming state 和队列。

运行中追加意图应使用：

- `steer()`：加入 steering queue；
- `followUp()`：加入 follow-up queue。

这两个方法只入队，不会自行启动一个已经 idle 的 Agent。具体消费时机见 [[03-agentLoop-无状态循环引擎#3. 双层循环为什么存在|03-agentLoop-无状态循环引擎 §3「双层循环为什么存在」]]。

这项互斥只约束当前 Agent 实例，不是进程级锁，也不等于 session 文件的跨进程 writer lock。

---

## 9. `prompt()` 与 `continue()` 的入口差异

### `prompt(newMessage)`

带着新 user/custom message 启动 run，请求下一条 assistant。

### `continue()`

从已有 transcript 继续。普通情况下，末尾应是可作为模型输入起点的 user 或 toolResult；如果末尾是 assistant，则必须存在待消费队列，否则没有新的输入可供模型继续。

产品层可在 retry、compaction 或 extension 注入后调用 `continue()`；因此一条用户 prompt 可能对应多个 Agent run。

---

## 10. abort 是协作式取消

`abort()` 触发当前 run 的取消信号，并把 `AbortSignal` 传给模型请求、tool、hook 与 listener。它不能强杀任意 JavaScript 异步函数；各组件必须主动响应信号。

可观察结果取决于取消时机：

- provider 响应取消，assistant 以 `aborted` 结束；
- tool 响应取消，可抛错或返回取消结果；
- tool 忽略取消，仍可能完成副作用并返回成功；
- listener 忽略信号，也会延迟 idle。

工具阶段的细分见 [[04-tool-execution-三阶段管道#6. 取消是协作式的|04-tool-execution-三阶段管道 §6「取消是协作式的」]]。

---

## 11. 未捕获异常的安全网

tool 与 tool hook 的预期错误通常已经在工具管道中转换成 ToolResultMessage，不会来到这里。

若其他未捕获异常从 loop 逸出，`Agent` 外壳会把 rejection 转成失败 assistant，并发出相应的 message/turn/agent 结束事件，使 state 有机会记录失败并最终回到 idle。

```text
loop rejection
  → failure assistant
  → message_end
  → turn_end
  → agent_end
  → listeners drain
  → idle
```

这是一层外壳安全网，不意味着任意用户回调都可以随意抛异常；系统策略回调仍应遵守“不抛出”的契约。

---

## 12. 与 session 系统的边界

- `Agent.state.messages`：当前内存 transcript；
- `SessionManager` entries：长期 append-only 事实；
- `AgentSession`：把 finalized message 持久化，并在恢复、分支或 compaction 后替换 Agent 的工作投影。

所以：

> `Agent` 让多次 run 共享内存状态；`SessionManager` 让状态跨进程和分支继续存在；`AgentSession` 把两者连接成产品行为。

---

## 13. 源码入口

- `packages/agent/src/agent.ts`：state、active run、事件归约、队列、listener 与 idle。
- `packages/agent/src/types.ts`：`AgentState`、`AgentMessage`、`AgentContext` 与 `AgentEvent`。
- `packages/agent/src/agent-loop.ts`：Agent 启动的无状态循环。
- `packages/agent/test/agent.test.ts`：listener、队列、重入、reset 与异常闭合测试。

## 14. 相关笔记

- 单次 run 控制流：[[03-agentLoop-无状态循环引擎]]
- tool 错误与取消：[[04-tool-execution-三阶段管道]]
- 产品级 session 与 compaction：[[official-doc/06-Sessions-会话树]]、[[official-doc/07-Compaction-上下文压缩]]
