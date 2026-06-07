# 08 - Session 文件格式

> 来源：https://pi.dev/docs/latest/session-format

## 1. 总览

Session 以 JSONL 文件存储：每行一个 JSON 对象，带 `type` 字段。条目之间通过 `id` / `parentId` 形成树，使分支无需新文件，对应 [[06-Sessions-会话树|会话树]] 模型。

## 2. 文件位置

```text
~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl
```

`<path>` 是工作目录的路径，`/` 被替换成 `-`。删除 session 直接删 `.jsonl` 文件，或在 `/resume` 里 `Ctrl+D`（系统装了 `trash` CLI 时走 trash）。

## 3. 版本

| 版本 | 说明 |
|------|------|
| v1 | 线性条目序列（legacy，自动迁移） |
| v2 | 树结构，加 `id` / `parentId` |
| v3 | 当前；把 `hookMessage` role 改名为 `custom` |

旧 session 加载时自动迁移到 v3。

## 4. 消息类型（Message Types）

### 4.1 Content Blocks

```typescript
interface TextContent {
  type: "text";
  text: string;
}

interface ImageContent {
  type: "image";
  data: string;      // base64 编码
  mimeType: string;  // 例如 "image/jpeg", "image/png"
}

interface ThinkingContent {
  type: "thinking";
  thinking: string;
}

interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
}
```

### 4.2 基础消息类型（来自 pi-ai）

```typescript
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;  // Unix ms
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: string;
  provider: string;
  model: string;
  usage: Usage;
  stopReason: "stop" | "length" | "toolUse" | "error" | "aborted";
  errorMessage?: string;
  timestamp: number;
}

interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: any;      // tool 特有的元数据
  isError: boolean;
  timestamp: number;
}

interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

### 4.3 扩展消息类型（来自 pi-coding-agent）

```typescript
interface BashExecutionMessage {
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  cancelled: boolean;
  truncated: boolean;
  fullOutputPath?: string;
  excludeFromContext?: boolean;  // 用 !! 前缀的命令为 true
  timestamp: number;
}

interface CustomMessage {
  role: "custom";
  customType: string;            // extension 标识
  content: string | (TextContent | ImageContent)[];
  display: boolean;              // 是否在 TUI 显示
  details?: any;                 // extension 特有元数据
  timestamp: number;
}

interface BranchSummaryMessage {
  role: "branchSummary";
  summary: string;
  fromId: string;                // 被分支离开的条目 ID
  timestamp: number;
}

interface CompactionSummaryMessage {
  role: "compactionSummary";
  summary: string;
  tokensBefore: number;
  timestamp: number;
}
```

### 4.4 `AgentMessage` Union

```typescript
type AgentMessage =
  | UserMessage
  | AssistantMessage
  | ToolResultMessage
  | BashExecutionMessage
  | CustomMessage
  | BranchSummaryMessage
  | CompactionSummaryMessage;
```

## 5. 条目基类（Entry Base）

除了 `SessionHeader`，所有条目继承：

```typescript
interface SessionEntryBase {
  type: string;
  id: string;                // 8-char hex ID
  parentId: string | null;   // 父条目 ID（首条为 null）
  timestamp: string;         // ISO 时间戳
}
```

## 6. 条目类型（Entry Types）

### 6.1 SessionHeader

文件第一行——元数据，不在树里（无 `id`/`parentId`）。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project"}
```

带 parent 的（由 `/fork`、`/clone` 或 `newSession({ parentSession })` 创建）：

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project","parentSession":"/path/to/original/session.jsonl"}
```

### 6.2 SessionMessageEntry

对话消息，`message` 字段是一个 `AgentMessage`。

```json
{"type":"message","id":"a1b2c3d4","parentId":"prev1234","timestamp":"2024-12-03T14:00:01.000Z","message":{"role":"user","content":"Hello"}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"...","message":{"role":"assistant","content":[{"type":"text","text":"Hi!"}],"provider":"anthropic","model":"claude-sonnet-4-5","usage":{...},"stopReason":"stop"}}
{"type":"message","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"...","message":{"role":"toolResult","toolCallId":"call_123","toolName":"bash","content":[{"type":"text","text":"output"}],"isError":false}}
```

### 6.3 ModelChangeEntry

session 中途切 model 留痕。

```json
{"type":"model_change","id":"d4e5f6g7","parentId":"c3d4e5f6","timestamp":"...","provider":"openai","modelId":"gpt-4o"}
```

### 6.4 ThinkingLevelChangeEntry

切 reasoning level 留痕。

```json
{"type":"thinking_level_change","id":"e5f6g7h8","parentId":"d4e5f6g7","timestamp":"...","thinkingLevel":"high"}
```

### 6.5 CompactionEntry

context 被压缩时创建，存历史消息的摘要。

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","timestamp":"...","summary":"User discussed X, Y, Z...","firstKeptEntryId":"c3d4e5f6","tokensBefore":50000}
```

可选字段：

- `details`：默认 `{ readFiles: string[], modifiedFiles: string[] }`，也可被 extension 自定义
- `fromHook`：`true` 表示由 extension 生成

机制见 [[07-Compaction-上下文压缩]]。

### 6.6 BranchSummaryEntry

`/tree` 跨分支跳转时创建——被抛弃分支到公共祖先这段的 LLM 摘要。

```json
{"type":"branch_summary","id":"g7h8i9j0","parentId":"a1b2c3d4","timestamp":"...","fromId":"f6g7h8i9","summary":"Branch explored approach A..."}
```

`details` 和 `fromHook` 同 CompactionEntry。

### 6.7 CustomEntry

extension 状态持久化——**不进入 LLM context**。

```json
{"type":"custom","id":"h8i9j0k1","parentId":"g7h8i9j0","timestamp":"...","customType":"my-extension","data":{"count":42}}
```

用 `customType` 识别自己 extension 的条目。

### 6.8 CustomMessageEntry

extension 注入的消息——**会进入 LLM context**。

```json
{"type":"custom_message","id":"i9j0k1l2","parentId":"h8i9j0k1","timestamp":"...","customType":"my-extension","content":"Injected context...","display":true}
```

字段：

- `content`：string 或 `(TextContent | ImageContent)[]`
- `display`：`true` 在 TUI 显示（带特殊样式），`false` 隐藏
- `details`：可选 extension 元数据（不发给 LLM）

### 6.9 LabelEntry

用户在条目上加书签/标记。

```json
{"type":"label","id":"j0k1l2m3","parentId":"i9j0k1l2","timestamp":"...","targetId":"a1b2c3d4","label":"checkpoint-1"}
```

`label` 设为 `undefined` 清掉。

### 6.10 SessionInfoEntry

session 元数据——比如用户定义的显示名（由 `/name` 或 `pi.setSessionName()` 设置）。

```json
{"type":"session_info","id":"k1l2m3n4","parentId":"j0k1l2m3","timestamp":"...","name":"Refactor auth module"}
```

设了名字之后，`/resume` 显示这个名字而不是第一条消息。

## 7. 树结构

- 首条 `parentId: null`
- 后续每条通过 `parentId` 指向 parent
- 分支 = 从早期条目派生新的 child
- "leaf" = 当前位置

```text
[user msg] ─── [assistant] ─── [user msg] ─── [assistant] ─┬─ [user msg] ← current leaf
                                                            │
                                                            └─ [branch_summary] ─── [user msg] ← 备选分支
```

## 8. Context Building

`buildSessionContext()` 从当前 leaf 走到 root，产出 LLM 消息列表：

1. 收集路径上所有条目
2. 提取活动 model 和 thinking level
3. 路径上若有 `CompactionEntry`：
   - 先输出 summary
   - 再输出从 `firstKeptEntryId` 到 compaction 点的消息
   - 最后是 compaction 之后的消息
4. 把 `BranchSummaryEntry` 和 `CustomMessageEntry` 翻译成合适的 message 格式

## 9. 解析示例

```javascript
import { readFileSync } from "fs";

const lines = readFileSync("session.jsonl", "utf8").trim().split("\n");

for (const line of lines) {
  const entry = JSON.parse(line);

  switch (entry.type) {
    case "session":
      console.log(`Session v${entry.version ?? 1}: ${entry.id}`);
      break;
    case "message":
      console.log(`[${entry.id}] ${entry.message.role}: ${JSON.stringify(entry.message.content)}`);
      break;
    case "compaction":
      console.log(`[${entry.id}] Compaction: ${entry.tokensBefore} tokens summarized`);
      break;
    case "branch_summary":
      console.log(`[${entry.id}] Branch from ${entry.fromId}`);
      break;
    case "custom":
      console.log(`[${entry.id}] Custom (${entry.customType}): ${JSON.stringify(entry.data)}`);
      break;
    case "custom_message":
      console.log(`[${entry.id}] Extension message (${entry.customType}): ${entry.content}`);
      break;
    case "label":
      console.log(`[${entry.id}] Label "${entry.label}" on ${entry.targetId}`);
      break;
    case "model_change":
      console.log(`[${entry.id}] Model: ${entry.provider}/${entry.modelId}`);
      break;
    case "thinking_level_change":
      console.log(`[${entry.id}] Thinking: ${entry.thinkingLevel}`);
      break;
  }
}
```

## 10. SessionManager API

### 10.1 静态创建方法

| 方法 | 用途 |
|------|------|
| `SessionManager.create(cwd, sessionDir?)` | 新 session |
| `SessionManager.open(path, sessionDir?)` | 打开已有文件 |
| `SessionManager.continueRecent(cwd, sessionDir?)` | 续接最近一次或新建 |
| `SessionManager.inMemory(cwd?)` | 不存盘（测试用） |
| `SessionManager.forkFrom(sourcePath, targetCwd, sessionDir?)` | 从其它项目 fork |

### 10.2 静态列表方法

| 方法 | 用途 |
|------|------|
| `SessionManager.list(cwd, sessionDir?, onProgress?)` | 列指定目录下的 session |
| `SessionManager.listAll(onProgress?)` | 跨所有项目列 |

### 10.3 实例方法 — Session 管理

| 方法 | 用途 |
|------|------|
| `newSession(options?)` | 开新 session，options 含 `{ parentSession?: string }` |
| `setSessionFile(path)` | 切到另一个文件 |
| `createBranchedSession(leafId)` | 把分支抽到新文件 |

### 10.4 实例方法 — 追加（都返回 entry ID）

| 方法 | 写入条目类型 |
|------|-------------|
| `appendMessage(message)` | `message` |
| `appendThinkingLevelChange(level)` | `thinking_level_change` |
| `appendModelChange(provider, modelId)` | `model_change` |
| `appendCompaction(summary, firstKeptEntryId, tokensBefore, details?, fromHook?)` | `compaction` |
| `appendCustomEntry(customType, data?)` | `custom`（不进 context） |
| `appendSessionInfo(name)` | `session_info` |
| `appendCustomMessageEntry(customType, content, display, details?)` | `custom_message`（进 context） |
| `appendLabelChange(targetId, label)` | `label` |

### 10.5 实例方法 — 树导航

| 方法 | 用途 |
|------|------|
| `getLeafId()` | 当前 leaf 的 ID |
| `getLeafEntry()` | 当前 leaf 条目 |
| `getEntry(id)` | 按 ID 取 |
| `getBranch(fromId?)` | 从某条目走到 root |
| `getTree()` | 完整树 |
| `getChildren(parentId)` | 直接 child |
| `getLabel(id)` | 条目 label |
| `branch(entryId)` | 把 leaf 移到早期条目 |
| `resetLeaf()` | 把 leaf 重置为 null |
| `branchWithSummary(entryId, summary, details?, fromHook?)` | 带摘要的分支 |

### 10.6 实例方法 — Context 与信息

| 方法 | 用途 |
|------|------|
| `buildSessionContext()` | 返回喂给 LLM 的 messages、thinkingLevel、model |
| `getEntries()` | 所有条目（不含 header） |
| `getHeader()` | header 元数据 |
| `getSessionName()` | 最新 `session_info` 条目里的显示名 |
| `getCwd()` | 工作目录 |
| `getSessionDir()` | 存储目录 |
| `getSessionId()` | UUID |
| `getSessionFile()` | 文件路径（内存模式为 `undefined`） |
| `isPersisted()` | 是否存盘 |
