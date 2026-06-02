# 11 - JSON 事件流（`--mode json`）

> 来源：https://pi.dev/docs/latest/json

## 1. 用途与定位

`pi --mode json "Your prompt"` 把所有 session event 以 JSON 行（JSONL）输出到 stdout——适合把 Pi 集成进其他工具或自定义 UI 做监控/对接。

跟其它集成方式的取舍：

| 场景 | 选 |
|------|----|
| 同进程嵌入 Node | **SDK**（[[16-SDK-嵌入-Node-应用]]） |
| 跨语言客户端、双向控制 | **RPC**（[[17-RPC-模式]]） |
| 单向读事件流、跑完即止 | **JSON 模式**（本笔记） |

## 2. 输出结构

### 2.1 Session Header（第一行）

```json
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path"}
```

字段：`type`、`version`、`id`、`timestamp`、`cwd`。

### 2.2 后续行

agent 运行过程中产生的 event，每行一个独立 JSON 对象。

## 3. Event 类型

事件来自两个 type union：`AgentSessionEvent`（session 层包装）和 `AgentEvent`（基础 agent event）。

### 3.1 Session 层事件（`AgentSessionEvent`）

| 事件 | 字段 | 触发时机 |
|------|------|----------|
| `queue_update` | `steering: readonly string[]`, `followUp: readonly string[]` | steering/follow-up 队列变化时；发送当前**完整**队列（非 delta） |
| `compaction_start` | `reason: "manual" \| "threshold" \| "overflow"` | 压缩开始（手动或自动） |
| `compaction_end` | `reason`, `result: CompactionResult \| undefined`, `aborted: boolean`, `willRetry: boolean`, `errorMessage?: string` | 压缩结束 |
| `auto_retry_start` | `attempt: number`, `maxAttempts: number`, `delayMs: number`, `errorMessage: string` | 自动 retry 开始 |
| `auto_retry_end` | `success: boolean`, `attempt: number`, `finalError?: string` | 自动 retry 结束 |

### 3.2 基础 Agent 事件（`AgentEvent`）

**Agent 生命周期**：

| 事件 | 字段 | 时机 |
|------|------|------|
| `agent_start` | — | agent run 开始 |
| `agent_end` | `messages: AgentMessage[]` | 结束时含最终消息列表 |

**Turn 生命周期**：

| 事件 | 字段 | 时机 |
|------|------|------|
| `turn_start` | — | 每个 turn 开始 |
| `turn_end` | `message: AgentMessage`, `toolResults: ToolResultMessage[]` | turn 完成后，含所有 tool 结果 |

**Message 生命周期**：

| 事件 | 字段 | 时机 |
|------|------|------|
| `message_start` | `message: AgentMessage` | 新消息开始（通常是 assistant） |
| `message_update` | `message: AgentMessage`, `assistantMessageEvent: AssistantMessageEvent` | streaming delta |
| `message_end` | `message: AgentMessage` | 消息最终态 |

**Tool 执行**：

| 事件 | 字段 | 时机 |
|------|------|------|
| `tool_execution_start` | `toolCallId`, `toolName`, `args` | 调用开始 |
| `tool_execution_update` | `toolCallId`, `toolName`, `args`, `partialResult` | 流式/部分结果（`partialResult` 是**累计**值） |
| `tool_execution_end` | `toolCallId`, `toolName`, `result`, `isError: boolean` | 调用完成 |

## 4. 消息类型

**Base**（`packages/ai/src/types.ts`）：

- `UserMessage`
- `AssistantMessage`
- `ToolResultMessage`

**Extended**（`packages/coding-agent/src/core/messages.ts`）：

- `BashExecutionMessage`
- `CustomMessage`
- `BranchSummaryMessage`
- `CompactionSummaryMessage`

完整字段定义见 [[15-Session-文件格式#消息类型message-types]]。

## 5. 输出序列示例

```json
{"type":"agent_start"}
{"type":"turn_start"}
{"type":"message_start","message":{"role":"assistant","content":[],...}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_delta","delta":"Hello",...}}
{"type":"message_end","message":{...}}
{"type":"turn_end","message":{...},"toolResults":[]}
{"type":"agent_end","messages":[...]}
```

## 6. 结合 `jq` 过滤

```bash
pi --mode json "List files" 2>/dev/null | jq -c 'select(.type == "message_end")'
```

只输出完整的 message_end 事件。`2>/dev/null` 把 stderr 丢掉，只让 JSON 进 pipe。

## 7. 关键提醒

- **手动和自动 compaction 共享** `compaction_start` / `compaction_end` 一对事件，靠 `reason` 字段区分
- `queue_update` **总发送完整队列**，不是 delta
- 跟 [[17-RPC-模式]] 不同的是：JSON 模式是**单向**输出流，不接受 stdin command
