# 08 - Extensions：扩展编写

> 来源：https://pi.dev/docs/latest/extensions

## 1. Extensions 是什么

Extension 是 TypeScript 模块，扩展 Pi coding agent 的行为。能力清单：

- 订阅生命周期事件（拦截/修改 tool 调用）
- 注册自定义 tool 给 LLM 调用
- 注册 slash 命令
- 拉起用户交互对话框
- 自定义 TUI 组件
- 持久化 session 状态
- 自定义渲染逻辑

**运行权限**：full system permissions。**只装受信任来源**。

**加载方式**：通过 [jiti](https://github.com/unjs/jiti)——TypeScript 文件无需编译就能加载。

## 2. 工厂模式编写一个 extension

extension 默认导出一个工厂函数，接收 `ExtensionAPI`。可同步可异步——异步会被 Pi `await` 完成后再触发 `session_start`、`resources_discover` 或 provider flush。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 订阅事件
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });

  // 拦截危险 tool 调用
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  // 注册自定义 tool
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone by name",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // 注册 slash 命令
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello ${args || "world"}!`, "info");
    },
  });
}
```

异步示例——动态拉取 model 列表：

```typescript
export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const payload = await response.json();
  pi.registerProvider("local-openai", {
    baseUrl: "http://localhost:1234/v1",
    apiKey: "LOCAL_OPENAI_API_KEY",
    api: "openai-completions",
    models: payload.data.map((model) => ({
      id: model.id,
      name: model.name ?? model.id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: model.context_window ?? 128000,
      maxTokens: model.max_tokens ?? 4096,
    })),
  });
}
```

## 3. 加载方式

### CLI flag

```bash
pi -e ./my-extension.ts
```

文档建议 `-e` 仅用于快速测试。

### 自动发现位置

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/extensions/*.ts` | 全局 |
| `~/.pi/agent/extensions/*/index.ts` | 全局（子目录） |
| `.pi/extensions/*.ts` | 项目级 |
| `.pi/extensions/*/index.ts` | 项目级（子目录） |

### settings.json

```json
{
  "packages": [
    "npm:@foo/bar@1.0.0",
    "git:github.com/user/repo@v1"
  ],
  "extensions": [
    "/path/to/local/extension.ts",
    "/path/to/local/extension/dir"
  ]
}
```

自动发现位置下的 extension 支持 `/reload` 热重载。

## 4. 文件布局

### 单文件

```text
~/.pi/agent/extensions/my-extension.ts
```

### 带 index.ts 的目录

```text
my-extension/
├── index.ts
├── tools.ts
└── utils.ts
```

### 带 npm 依赖的 package

```text
my-extension/
├── package.json
├── package-lock.json
├── node_modules/
└── src/index.ts
```

`package.json` 可以在 `pi.extensions` 数组里声明 entry point。可用 import：

- `@earendil-works/pi-coding-agent`
- `typebox`
- `@earendil-works/pi-ai`（含 `StringEnum`）
- `@earendil-works/pi-tui`
- Node.js 内置模块

## 5. 全部生命周期事件

### 5.1 Resource

| Event | 用途 |
|-------|------|
| `resources_discover` | 贡献额外的 skill / prompt / theme 路径；reason 是 `"startup"` 或 `"reload"` |

### 5.2 Session

| Event | 时机 |
|-------|------|
| `session_start` | session 开启、载入或重载时触发；reason：`"startup" / "reload" / "new" / "resume" / "fork"` |
| `session_before_switch` | `/new` 或 `/resume` 之前；可返回 `{ cancel: true }` 取消 |
| `session_before_fork` | fork 前 |
| `session_before_compact` | 压缩前 |
| `session_before_tree` | `/tree` 前 |
| `session_compact` | 压缩完成通知 |
| `session_tree` | tree 完成通知 |
| `session_shutdown` | runtime 拆除前；reason：`"quit" / "reload" / "new" / "resume" / "fork"` |

### 5.3 Agent

| Event | 用途 |
|-------|------|
| `before_agent_start` | 可注入持久消息和/或修改 system prompt；提供 `systemPromptOptions`（custom prompts、guidelines、snippets、cwd、context files、skills） |
| `agent_start` / `agent_end` | run 边界 |
| `turn_start` / `turn_end` | 每个 turn 的边界 |
| `message_start` / `message_update` / `message_end` | message 生命周期；`message_end` handler 可返回 `{ message }` 替换最终消息（role 必须匹配） |
| `tool_execution_start` / `update` / `end` | tool 执行 |
| `context` | 每次 LLM 调用前**非破坏性**修改 messages |
| `before_provider_request` | 发送前检查或替换 provider payload |
| `after_provider_response` | 在 stream 消费前收 HTTP 状态码和 headers |

### 5.4 Model

| Event | 用途 |
|-------|------|
| `model_select` | source：`"set" / "cycle" / "restore"` |
| `thinking_level_select` | 只通知，返回值被忽略 |

### 5.5 Tool

| Event | 用途 |
|-------|------|
| `tool_call` | 可通过 `{ block: true, reason?: string }` 阻断；`event.input` **可变**，改了立即生效，**不再校验** |
| `tool_result` | 可修改结果；多个 handler 按加载顺序链式执行 |

### 5.6 User Bash 与 Input

| Event | 用途 |
|-------|------|
| `user_bash` | `!` 或 `!!` 触发；可提供自定义 operation、包装内置 local bash，或完全替换结果 |
| `input` | 返回 `{ action: "continue" \| "transform" \| "handled" }` |

**处理顺序**：extension 命令 → `input` 事件 → skill 展开 → template 展开 → agent 处理。

## 6. ExtensionAPI 方法清单

| 方法 | 用途 |
|------|------|
| `pi.on(event, handler)` | 订阅生命周期事件 |
| `pi.registerTool(definition)` | 注册 LLM 可调用 tool（加载时和运行时都能用） |
| `pi.sendMessage(message, options?)` | 注入自定义消息；`deliverAs`：`"steer"`（默认）/ `"followUp"` / `"nextTurn"` |
| `pi.sendUserMessage(content, options?)` | 发 user 消息；streaming 时必须指定 `deliverAs` |
| `pi.appendEntry(customType, data?)` | 持久化状态（不进 LLM context） |
| `pi.setSessionName(name)` / `pi.getSessionName()` | session 显示名 |
| `pi.setLabel(entryId, label)` | 给条目加书签；传 `undefined` 清掉 |
| `pi.registerCommand(name, options)` | 加 `/command`；可带 `getArgumentCompletions` |
| `pi.getCommands()` | 列出可用 slash 命令 |
| `pi.registerMessageRenderer(customType, renderer)` | 自定义 TUI 消息渲染 |
| `pi.registerShortcut(shortcut, options)` | 注册键位 |
| `pi.registerFlag(name, options)` / `pi.getFlag(name)` | 加 CLI flag |
| `pi.exec(command, args, options?)` | 执行 shell 命令 |
| `pi.getActiveTools()` / `pi.getAllTools()` / `pi.setActiveTools(names)` | 管理 active tool |
| `pi.setModel(model)` | 没 API key 时返回 false |
| `pi.getThinkingLevel()` / `pi.setThinkingLevel(level)` | levels：`off / minimal / low / medium / high / xhigh` |
| `pi.events.on/emit` | 跨 extension 事件总线 |
| `pi.registerProvider(name, config)` / `pi.unregisterProvider(name)` | 动态 provider，详见 [[14-Custom-Providers-自定义-Provider]] |

## 7. ExtensionContext（`ctx`）

所有 handler 都能拿到：

- `ctx.ui` — 对话框（`select` / `confirm` / `input` / `editor`）、`notify` / `setStatus` / `setWidget` / `setFooter` / `setTitle` / `setEditorText` / `pasteToEditor` / `addAutocompleteProvider`
- `ctx.hasUI` — print / JSON 模式下为 `false`
- `ctx.cwd` — 工作目录
- `ctx.sessionManager` — 只读 session 访问（`getEntries` / `getBranch` / `getLeafId`）
- `ctx.modelRegistry` / `ctx.model`
- `ctx.signal` — 当前 agent 的 AbortSignal（typically 在 turn event 内有效）
- `ctx.isIdle()` / `ctx.abort()` / `ctx.hasPendingMessages()`
- `ctx.shutdown()` — 优雅退出，会等到 idle
- `ctx.getContextUsage()` — 当前 token 用量
- `ctx.compact()` — 触发压缩，可带 `onComplete` / `onError`
- `ctx.getSystemPrompt()` — 当前 system prompt 字符串

## 8. ExtensionCommandContext（仅 command handler）

在 `ExtensionContext` 之上加了 session 控制方法——文档强调 **"在 event handler 里调会死锁"**：

- `ctx.waitForIdle()`
- `ctx.newSession(options?)`——含 `parentSession` / `setup` / `withSession`
- `ctx.fork(entryId, options?)`——`position: "before" | "at"`
- `ctx.navigateTree(targetId, options?)`
- `ctx.switchSession(sessionPath, options?)`
- `ctx.reload()`——跑 `/reload` 流程

## 9. 自定义 Tool

Tool 必须定义：`name`、`label`、`description`、`parameters`（typebox）、`execute`。
可选字段：`promptSnippet`、`promptGuidelines`、`prepareArguments`、`renderCall`、`renderResult`、`renderShell`。

**重要规则**：

- 字符串枚举用 `@earendil-works/pi-ai` 的 `StringEnum`（Google API 兼容性）
- `execute` 抛错才会被识别为错误；返回错误内容不会自动设置 error flag
- 返回 `terminate: true` 提示跳过后续 LLM 调用（仅当批次内**所有** result 都 terminate 时生效）
- 内置 50KB / 2000 行截断：用 `truncateHead` / `truncateTail` / `formatSize` / `DEFAULT_MAX_BYTES` / `DEFAULT_MAX_LINES`
- 改文件的 tool 用 `withFileMutationQueue()` 避免并行竞态

**覆盖内置 tool**：extension 可以注册同名 tool 来覆盖 `read` / `bash` / `edit` / `write` / `grep` / `find` / `ls`。渲染逻辑按 slot 继承。

**远端执行接口**：`ReadOperations` / `WriteOperations` / `EditOperations` / `BashOperations` / `LsOperations` / `GrepOperations` / `FindOperations`。用 `createLocalBashOperations()` 复用 Pi 的本地后端。

## 10. State 管理

**state 存在 tool result 的 `details` 字段里**——这样分支切换时 state 跟随 tree 路径自动正确。

在 `session_start` 里通过遍历 `ctx.sessionManager.getBranch()` 找 `entry.message.role === "toolResult"` 来重建状态。

## 11. 自定义 UI

- 对话框支持 `timeout`（带实时倒计时）和 `signal`（AbortSignal）；timeout 返回 `undefined` / `false`
- Widget 可放 `aboveEditor`（默认）或 `belowEditor`
- `setFooter` **完全替换**内置 footer
- 用 `keyHint(keybindingId, description)` 和 `keyText(keybindingId)` 显示带按键提示的字串
- Keybinding namespace：`app.*` 给 coding-agent（如 `app.tools.expand`），`tui.*` 给 TUI（如 `tui.select.confirm`）

完整组件 pattern（SelectList、BorderedLoader、SettingsList、autocomplete）见 [[19-TUI-Components-构建终端-UI]]。

## 12. 官方示例

工作示例在 `packages/coding-agent/examples/extensions/` 下（pi GitHub repo）。
