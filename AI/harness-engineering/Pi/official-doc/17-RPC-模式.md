# 10 - RPC 模式

> 来源：https://pi.dev/docs/latest/rpc

## 1. 用途与定位

RPC 模式让 Pi 以 headless 方式跑起来，通过 stdin/stdout 上的 JSON 通信——适合嵌进 IDE、自定义 UI 或其他应用。

Node/TypeScript 用户**通常应该直接用 `AgentSession`**（见 [[16-SDK-嵌入-Node-应用]]）而不是 spawn 子进程；RPC 主要给跨语言客户端、需要进程隔离、或者要做语言无关 build 的场景。

## 2. 启动

```bash
pi --mode rpc [options]
```

常用选项：

| Flag | 说明 |
|------|------|
| `--provider <name>` | anthropic / openai / google 等 |
| `--model <pattern>` | 支持 `provider/id` 和 `:<thinking>` 后缀 |
| `--no-session` | 关掉 session 持久化 |
| `--session-dir <path>` | 自定义 session 存储目录 |

## 3. 协议总览

| 方向 | 内容 |
|------|------|
| **stdin → pi**：Commands | 每行一个 JSON 对象 |
| **pi → stdout**：Responses | `type: "response"`，含 `success`；可选 `id` 关联请求 |
| **pi → stdout**：Events | 流式 JSON 行，**无 `id` 字段** |

### Framing 注意

严格 JSONL，**只用 LF（`\n`）作分隔符**，有 trailing `\r` 要剥掉。

**Node `readline` 不合规**——它会额外按 U+2028 / U+2029 分行，可能误切 JSON。下面 Node 示例展示了合规的 JSONL reader。

## 4. Commands

### 4.1 Prompting

#### `prompt`

```json
{"id": "req-1", "type": "prompt", "message": "Hello, world!"}
```

带图片：

```json
{"id": "req-1", "type": "prompt", "message": "Analyze",
 "images": [{"type":"image","data":"base64...","mimeType":"image/png"}]}
```

**streaming 期间发 `prompt` 必须指定 `streamingBehavior`**：

| 值 | 行为 |
|----|------|
| `"steer"` | 排队，**当前 assistant turn 的 tool 调用全完后、下次 LLM 调用前**送达 |
| `"followUp"` | 排队，**agent 完全停下来**才送达 |

Extension 命令（如 `/mycommand`）**立刻执行**——即使在 streaming 中。Skill（`/skill:name`）和 template（`/template`）命令在发送前展开。

响应：

```json
{"id":"req-1","type":"response","command":"prompt","success":true}
```

**Acceptance 后的失败**通过 event 流报，不是第二条 response。

#### `steer`

队列里塞一条 steering 消息（**不能用于 extension 命令**）。

```json
{"type": "steer", "message": "Stop and do this instead"}
```

#### `follow_up`

agent 没有更多 tool 调用 / steering 时才送达。

```json
{"type": "follow_up", "message": "After you're done, also do this"}
```

#### `abort`

中断当前 agent 操作。

#### `new_session`

开新 session。可被 `session_before_switch` extension handler 取消。可选 `parentSession`。返回 `data.cancelled`。

### 4.2 State

#### `get_state`

```json
{
  "model": {...},
  "thinkingLevel": "medium",
  "isStreaming": false,
  "isCompacting": false,
  "steeringMode": "all",
  "followUpMode": "one-at-a-time",
  "sessionFile": "...",
  "sessionId": "abc123",
  "sessionName": "my-feature-work",
  "autoCompactionEnabled": true,
  "messageCount": 5,
  "pendingMessageCount": 0
}
```

#### `get_messages`

返回 `{messages: [...]}`，元素是 `AgentMessage` 对象。

### 4.3 Model

| Command | 入参 | 返回 |
|---------|------|------|
| `set_model` | `{provider, modelId}` | 完整 Model |
| `cycle_model` | — | `{model, thinkingLevel, isScoped}`；只有一个 model 时返回 null data |
| `get_available_models` | — | `{models: [...]}` |

### 4.4 Thinking

| Command | 说明 |
|---------|------|
| `set_thinking_level` | levels：`off / minimal / low / medium / high / xhigh`（`xhigh` 仅 OpenAI codex-max） |
| `cycle_thinking_level` | 返回 `{level}` 或 null data |

### 4.5 队列模式

| Command | 取值 |
|---------|------|
| `set_steering_mode` | `"all"` 或 `"one-at-a-time"`（默认） |
| `set_follow_up_mode` | 同上 |

### 4.6 Compaction

| Command | 说明 |
|---------|------|
| `compact` | 可选 `customInstructions`；返回 `{summary, firstKeptEntryId, tokensBefore, details}` |
| `set_auto_compaction` | `{enabled: bool}` |

### 4.7 Retry

| Command | 说明 |
|---------|------|
| `set_auto_retry` | `{enabled: bool}`；overloaded / rate-limit / 5xx 时 retry |
| `abort_retry` | 取消进行中的 retry |

### 4.8 Bash

#### `bash`

```json
{"command": "..."}
```

返回 `{output, exitCode, cancelled, truncated}`，截断时多一个 `fullOutputPath`。

**重要语义**：`bash` command 立刻执行并存为 `BashExecutionMessage`（**不发 event**）。下次 `prompt` 时，被转成 user message 给 LLM 看，形如：

```text
Ran `<cmd>`
```

```text
{output}
```

多次 bash 累积。

#### `abort_bash`

中断进行中的 bash command。

### 4.9 Session

| Command | 说明 |
|---------|------|
| `get_session_stats` | 计数 + `tokens`（input/output/cacheRead/cacheWrite/total）+ `cost` + `contextUsage`（tokens / contextWindow / percent）。压缩后 `contextUsage.tokens`/`percent` 为 null，直到下次 assistant 响应 |
| `export_html` | 可选 `outputPath`；返回 `{path}` |
| `switch_session` | `{sessionPath}`；可取消；返回 `data.cancelled` |
| `fork` | `{entryId}`；可被 `session_before_fork` 取消；返回 `{text, cancelled}` |
| `clone` | 复制当前活动分支到当前位置；可取消 |
| `get_fork_messages` | 返回 user message：`[{entryId, text}, ...]` |
| `get_last_assistant_text` | 返回 `{text}` 或 `{text: null}` |
| `set_session_name` | `{name}`；通过 `get_state.sessionName` 可读 |

### 4.10 命令发现

#### `get_commands`

返回命令列表，每项含：

| 字段 | 含义 |
|------|------|
| `name` | 命令名 |
| `description` | 描述 |
| `source` | `"extension"` / `"prompt"` / `"skill"` |
| `location?` | `"user"` / `"project"` / `"path"` |
| `path?` | 来源路径 |

**内置 TUI 命令不在返回里**。

## 5. Events（stdout JSONL，无 `id`）

| 事件 | 含义 |
|------|------|
| `agent_start` / `agent_end` | agent run 边界；`agent_end` 含全部生成的消息 |
| `turn_start` / `turn_end` | 一个 turn = 一次 assistant 响应 + tool 调用与结果 |
| `message_start` / `message_end` | 每条消息的生命周期 |
| `message_update` | streaming delta，见下 |
| `tool_execution_start` / `update` / `end` | tool 生命周期，靠 `toolCallId` 关联 |
| `queue_update` | 含当前 `steering` 和 `followUp` 数组 |
| `compaction_start` / `end` | `reason: "manual" \| "threshold" \| "overflow"` |
| `auto_retry_start` / `end` | 含 `attempt`、`maxAttempts`、`delayMs`、`errorMessage` |
| `extension_error` | extension 抛错 |

### 5.1 `message_update` streaming delta

`assistantMessageEvent.type` 取值：`start`、`text_start`、`text_delta`、`text_end`、`thinking_start`、`thinking_delta`、`thinking_end`、`toolcall_start`、`toolcall_delta`、`toolcall_end`（含完整 `toolCall`）、`done`（reason `stop`/`length`/`toolUse`）、`error`（reason `aborted`/`error`）。

### 5.2 `tool_execution_update`

含 `partialResult`——**是累计内容**（不是 delta）。客户端每次更新直接替换显示即可。

### 5.3 Compaction 注意

- 若 `reason: "overflow"` 且成功，event 含 `willRetry: true`——agent 自动重试 prompt
- 若被取消：`result: null, aborted: true`
- 失败：`errorMessage` 描述问题

## 6. Extension UI Sub-Protocol

Extensions 调 `ctx.ui.*` 方法时，会在 stdout 上产生 `extension_ui_request` 事件。

### 6.1 等待响应的对话框

| 方法 | 入参 | 响应 |
|------|------|------|
| `select` | `{title, options[], timeout?}` | `{value}` 或 `{cancelled}` |
| `confirm` | `{title, message, timeout?}` | `{confirmed: bool}` 或 `{cancelled}` |
| `input` | `{title, placeholder?}` | `{value}` 或 `{cancelled}` |
| `editor` | `{title, prefill?}` | `{value}` 或 `{cancelled}` |

带 `timeout` 的请求过期后**自动 resolve**（客户端不用自己 track 计时）。

### 6.2 Fire-and-forget（无响应）

| 方法 | 入参 |
|------|------|
| `notify` | `{message, notifyType: "info"|"warning"|"error"}` |
| `setStatus` | `{statusKey, statusText?}`（省略 text 即清除） |
| `setWidget` | `{widgetKey, widgetLines?, widgetPlacement: "aboveEditor"|"belowEditor"}`（RPC 模式下只支持字符串数组；component factory 被忽略） |
| `setTitle` | `{title}` |
| `set_editor_text` | `{text}` |

### 6.3 响应结构

```json
{"type":"extension_ui_response","id":"uuid-1","value":"Allow"}
{"type":"extension_ui_response","id":"uuid-2","confirmed":true}
{"type":"extension_ui_response","id":"uuid-3","cancelled":true}
```

### 6.4 RPC 模式下的功能降级

| API | 行为 |
|-----|------|
| `custom()` | 返回 `undefined` |
| `setWorkingMessage`、`setWorkingIndicator`、`setFooter`、`setHeader`、`setEditorComponent`、`setToolsExpanded` | no-op |
| `getEditorText()` | 返回 `""` |
| `getToolsExpanded()` | 返回 `false` |
| `pasteToEditor()` | 转交给 `setEditorText()` |
| `getAllThemes()` | 返回 `[]` |
| `getTheme()` | 返回 `undefined` |
| `setTheme()` | 返回 `{success: false, error: "..."}` |
| `ctx.hasUI` | 仍为 `true` |

完整 UI API 见 [[19-TUI-Components-构建终端-UI]]。

## 7. 错误处理

失败的 command：

```json
{"type":"response","command":"set_model","success":false,
 "error":"Model not found: invalid/model"}
```

JSON 解析错误：`"command":"parse"`。

## 8. 类型参考

### `Model`

`id`, `name`, `api`, `provider`, `baseUrl`, `reasoning`, `input[]`, `contextWindow`, `maxTokens`, `cost: {input, output, cacheRead, cacheWrite}`。

### `UserMessage`

`{role:"user", content, timestamp, attachments[]}`——content 可以是 string 或 `TextContent`/`ImageContent` 数组。

### `AssistantMessage`

`{role:"assistant", content[], api, provider, model, usage:{input,output,cacheRead,cacheWrite,cost:{...}}, stopReason, timestamp}`。
stopReason：`stop` / `length` / `toolUse` / `error` / `aborted`。
content block：`text` / `thinking` / `toolCall`。

### `ToolResultMessage`

`{role:"toolResult", toolCallId, toolName, content[], isError, timestamp}`。

### `BashExecutionMessage`

`{role:"bashExecution", command, output, exitCode, cancelled, truncated, fullOutputPath, timestamp}`。

### `Attachment`

`{id, type, fileName, mimeType, size, content, extractedText, preview}`。

完整消息类型见 [[15-Session-文件格式#消息类型message-types]]。

## 9. Python 客户端示例

```python
import subprocess, json

proc = subprocess.Popen(
    ["pi", "--mode", "rpc", "--no-session"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True
)

def send(cmd):
    proc.stdin.write(json.dumps(cmd) + "\n")
    proc.stdin.flush()

send({"type": "prompt", "message": "Hello!"})

for line in proc.stdout:
    event = json.loads(line)
    if event.get("type") == "message_update":
        delta = event.get("assistantMessageEvent", {})
        if delta.get("type") == "text_delta":
            print(delta["delta"], end="", flush=True)
    if event.get("type") == "agent_end":
        print()
        break
```

## 10. Node.js 客户端示例（JSONL-safe reader）

```javascript
const { spawn } = require("child_process");
const { StringDecoder } = require("string_decoder");

const agent = spawn("pi", ["--mode", "rpc", "--no-session"]);

function attachJsonlReader(stream, onLine) {
  const decoder = new StringDecoder("utf8");
  let buffer = "";
  stream.on("data", (chunk) => {
    buffer += typeof chunk === "string" ? chunk : decoder.write(chunk);
    while (true) {
      const i = buffer.indexOf("\n");
      if (i === -1) break;
      let line = buffer.slice(0, i);
      buffer = buffer.slice(i + 1);
      if (line.endsWith("\r")) line = line.slice(0, -1);
      onLine(line);
    }
  });
  stream.on("end", () => {
    buffer += decoder.end();
    if (buffer.length > 0) {
      onLine(buffer.endsWith("\r") ? buffer.slice(0, -1) : buffer);
    }
  });
}

attachJsonlReader(agent.stdout, (line) => {
  const event = JSON.parse(line);
  if (event.type === "message_update") {
    const e = event.assistantMessageEvent;
    if (e.type === "text_delta") process.stdout.write(e.delta);
  }
});

agent.stdin.write(JSON.stringify({ type: "prompt", message: "Hello" }) + "\n");

process.on("SIGINT", () => {
  agent.stdin.write(JSON.stringify({ type: "abort" }) + "\n");
});
```

文档点名的参考实现：

- `src/modes/rpc/rpc-client.ts`（TypeScript 强类型客户端）
- `test/rpc-example.ts`（交互式示例）
- `examples/rpc-extension-ui.ts` 配合 `examples/extensions/rpc-demo.ts`
