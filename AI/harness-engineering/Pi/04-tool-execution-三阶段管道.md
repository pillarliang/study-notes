---
title: tool execution：三阶段管道与结果顺序
aliases:
  - Pi tool execution
  - 工具执行三阶段管道
  - 05-tool-execution-三阶段管道
tags: [Pi, tools, agent-loop, source-code]
source_repository: earendil-works/pi
source_commit: 534bcbff
related:
  - "[[03-agentLoop-无状态循环引擎]]"
  - "[[05-Agent-有状态运行时外壳]]"
---

# tool execution：三阶段管道与结果顺序

> [!summary] 核心结论
> 正常 tool call 经过 `prepare → execute → finalize`。prepare 决定能否进入真实执行，execute 产生副作用，finalize 确定最终结果。多个调用可以并发，但结果消息仍按模型请求顺序写回。

本文以 `earendil-works/pi` commit `534bcbff` 为基线。

---

## 0. 先区分三种数据

### `ToolCall`

模型提出的结构化请求：

```json
{
  "type": "toolCall",
  "id": "call-read-1",
  "name": "read",
  "arguments": {"path": "config.json"}
}
```

`id` 是关联本次请求与结果的标识；`name` 用于查找已注册 tool；`arguments` 是待处理参数。

### 内部 tool result

`execute` 或错误处理产生的中间结果，尚未转换成 transcript message。它可以携带 content、details、isError 与 terminate。

### `ToolResultMessage`

写回工作消息、供后续 LLM 请求读取的 finalized message：

```json
{
  "role": "toolResult",
  "toolCallId": "call-read-1",
  "toolName": "read",
  "content": [{"type": "text", "text": "mode=dev"}],
  "details": {"path": "config.json"},
  "isError": false,
  "timestamp": 1730000000000
}
```

正常路径为：

```text
ToolCall
  → prepare
  → execute
  → finalize
  → ToolResultMessage
  → context.messages
  → 下一次 LLM 请求
```

---

## 1. `prepare`：决定能否进入真实执行

固定顺序是：

```text
按 name 查找 tool
  → 可选 prepareArguments
  → schema 验证
  → 可选 beforeToolCall
  → prepared 或 immediate
```

### 1.1 查找 tool

只在本次 context 的 tools 中按 name 查找。不存在时生成错误结果，不调用任何 tool 实现。

### 1.2 `prepareArguments`

tool 可选提供参数兼容转换，并且它发生在 schema 验证之前。例如把旧 edit 参数：

```json
{"path":"file.txt","oldText":"before","newText":"after"}
```

转换为当前结构：

```json
{
  "path":"file.txt",
  "edits":[{"oldText":"before","newText":"after"}]
}
```

转换抛错时，prepare 收敛为错误结果。

### 1.3 schema 验证

验证必填字段、类型与嵌套结构。验证通过只说明参数形状合法，不代表业务策略允许执行，也不保证副作用一定成功。

### 1.4 `beforeToolCall`

应用可在参数验证后决定继续或阻止。它适合实现路径策略、审批或审计。

如果 hook 改写已验证参数，当前基线不会自动再次做 schema 验证；hook 作者必须保证改写后的值仍满足 tool 契约。

### 1.5 两种 prepare 结果

- **`prepared`**：保存已选 tool、处理后的参数和 ToolCall；之后进入 execute 与 finalize。
- **`immediate`**：已经得到错误或阻止结果；跳过 execute 与 finalize，直接产生结束事件和结果消息。

prepare 的边界是“是否允许调用具体 tool 的 execute”，不是物理 sandbox。

---

## 2. `execute`：真正产生副作用

只有 `prepared` 调用进入 execute。循环传入：

- tool call ID；
- prepare 后的参数；
- `AbortSignal`；
- 可选进度上报函数。

read、edit、bash 或网络工具的主要动作都发生在这里。

### 正常返回

得到内部成功结果，`isError` 默认为 false，然后继续 finalize。

### 抛出异常

异常被转换成内部错误结果，`isError` 设为 true，然后**仍进入 finalize**。tool 错误因此通常成为模型可读的结果，而不是直接杀死整个 agent run。

execute 已经发生的文件、进程或网络副作用不会由管道自动回滚。

---

## 3. `finalize`：确定最终结果

finalize 只处理已经进入 execute 的调用。

若应用提供 `afterToolCall`，它可以按字段改写：

- content；
- details；
- isError；
- terminate。

未提供 hook 或返回不需要覆盖的字段时，保留 execute 结果。

如果 `afterToolCall` 抛异常，原结果被新的错误结果替换；已经发生的外部副作用仍不回滚。

finalize 完成后，管道发出 tool 结束事件，并准备 `ToolResultMessage`。

---

## 4. 一批调用怎样调度

### 4.1 整批串行的条件

满足任一条件时，本条 assistant message 中的整批 tool call 按请求顺序执行：

1. 本次运行把 tool execution 配置为 sequential；
2. 本批任一已找到的 tool 声明 `executionMode: "sequential"`。

这是“一票串行”：一个 sequential tool 会让同批其他调用也等待。未被请求的 sequential tool 不影响当前批次。

### 4.2 默认并行不是所有阶段都并行

当前基线的并行路径是：

```text
按请求顺序完成 prepare
  → 并发运行 prepared 调用的 execute + finalize
  → 等待批次收齐
  → 按请求顺序构造并写回 ToolResultMessage
```

假设模型依次请求：

```text
slowRead：80 ms
fastGrep：10 ms
```

可观察到：

```text
请求顺序：slowRead → fastGrep
结束事件：fastGrep → slowRead
结果消息：slowRead → fastGrep
```

事件按完成时间发出，让 UI 尽早显示进度；消息按请求顺序写回，使 transcript 与后续 LLM 输入保持确定性。

因此收到 `tool_execution_end` 只表示该调用已经得到 finalized internal result，不表示它的 message 已经进入 context。

---

## 5. 旁路与错误分类

### 5.1 assistant 输出被截断

assistant 以长度上限停止时，Pi 不执行其中识别到的 tool call。因为截断参数即使能解析，也不能证明语义完整。

这些已识别调用会得到错误结果；prepare、execute、finalize 都跳过。

### 5.2 prepare 失败或阻止

以下情况产生 `immediate` 结果：

- tool 不存在；
- `prepareArguments` 抛错；
- schema 验证失败；
- `beforeToolCall` 阻止或抛错；
- prepare 阶段检测到取消。

它们不进入真实 execute，也不进入 finalize。

### 5.3 execute 抛错

```text
execute throw
  → 内部 error result
  → finalize / afterToolCall
  → ToolResultMessage
```

`afterToolCall` 可以改写该错误结果。

### 5.4 finalize 抛错

```text
afterToolCall throw
  → 用新的 error result 替换原结果
  → 外部副作用保持原状
  → ToolResultMessage
```

---

## 6. 取消是协作式的

`AbortSignal` 只通知“已经请求取消”，不能强行停止任意 JavaScript Promise。tool 与 hook 必须主动检查信号或把它继续传给支持取消的 API。

取消发生在不同阶段，结果不同：

### 尚未启动

调度器可停止启动后续调用。这些调用可能没有 start 事件，也没有 ToolResultMessage。

### `beforeToolCall` 正在运行

循环不能在函数内部强杀它。hook 若响应取消并返回，prepare 可生成 `Operation aborted` 一类 immediate 错误；若忽略信号，循环只能等待。

### execute 已经开始

tool 若配合取消，可停止或抛错；若忽略信号并成功完成，最终结果仍可能是成功。execute 结束后仍进入 finalize。

### `afterToolCall` 正在运行

同样必须等待 hook 返回或抛错，才能确定最终结果。

所以“已调用 abort”不等于“所有工具已经停止”，也不等于“每个结果自动变成错误”。

---

## 7. `terminate` 的精确含义

一批调用只有在以下条件同时满足时，才不再仅因本批 toolResult 自动发起下一次 LLM 请求：

1. 本批实际产生了至少一个最终结果；
2. 每个最终结果都明确设置 `terminate: true`。

```text
true + 未设置  → 仍由 toolResult 驱动续轮
true + true    → 本批不再要求自动续轮
```

`terminate` 可来自 execute 结果、`afterToolCall` 改写，或 prepare 阻止时产生的 immediate 结果。聚合检查的是实际结果，不是最初请求但可能未启动的全部调用。

它不强制终止整个 agent run；steering 或 follow-up 仍可继续推动循环。续轮优先级见 [[03-agentLoop-无状态循环引擎#4. 续轮与停止条件|03-agentLoop-无状态循环引擎 §4「续轮与停止条件」]]。

---

## 8. 边界外事项

- 何时开始下一次 LLM 调用：[[03-agentLoop-无状态循环引擎]]
- abort API、active run 和 idle：[[05-Agent-有状态运行时外壳]]
- 具体 read/edit/bash 的产品工具行为：从 [[official-doc/22-从零到一搭建-Agent-完整技术文档]] 下钻
- sandbox 与 OS 权限：[[study-notes/AI/harness-engineering/harness-survey-etclovg/01-E-执行环境与沙箱]]

---

## 9. 源码入口

- `packages/agent/src/agent-loop.ts`：三阶段处理、批调度、事件与消息顺序。
- `packages/agent/src/types.ts`：tool、hook、result 与运行配置契约。
- `packages/ai/src/utils/validation.ts`：tool 参数验证。
