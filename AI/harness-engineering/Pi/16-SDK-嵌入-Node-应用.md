# 09 - SDK 嵌入 Node 应用

> 来源：https://pi.dev/docs/latest/sdk

## 1. 用途与定位

Pi SDK（`@earendil-works/pi-coding-agent`）让 Pi 的 agent 能力可以被嵌进 Node.js 应用——做自定义 UI、自动化 pipeline、sub-agent、集成测试等。

跟其它集成方式的取舍：

| 场景 | 选 |
|------|----|
| 同进程集成、类型安全、直接读 agent 状态、能注册 tool/extension | **SDK** |
| 跨语言客户端、进程隔离、语言无关 | **RPC**（见 [[17-RPC-模式]]） |
| 只想拿事件流做监控/对接 | **JSON 模式**（见 [[18-JSON-事件流]]） |

## 2. 安装

```bash
npm install @earendil-works/pi-coding-agent
```

SDK 跟 CLI 同一个 npm 包，不用单独装。

## 3. 最小例子

```ts
import {
  AuthStorage,
  createAgentSession,
  ModelRegistry,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const authStorage = AuthStorage.create();
const modelRegistry = ModelRegistry.create(authStorage);

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage,
  modelRegistry,
});

session.subscribe((event) => {
  if (
    event.type === "message_update" &&
    event.assistantMessageEvent.type === "text_delta"
  ) {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("What files are in the current directory?");
```

## 4. 核心概念

### 4.1 `createAgentSession()`

工厂函数，构造一个 `AgentSession`。默认用 `DefaultResourceLoader` 自动发现 extensions、skills、prompt templates、themes、context files。

签名：

```ts
function createAgentSession(
  options: CreateAgentSessionOptions
): Promise<CreateAgentSessionResult>;

interface CreateAgentSessionResult {
  session: AgentSession;
  extensionsResult: LoadExtensionsResult;
  modelFallbackMessage?: string;
}
```

### 4.2 `AgentSession`

agent 生命周期、消息历史、model 状态、压缩、事件流的统一管理对象。

**关键方法**：

| 方法 | 用途 |
|------|------|
| `prompt(text, options?)` | 发送 prompt |
| `steer(text)` / `followUp(text)` | streaming 期间排队消息 |
| `subscribe(listener)` | 订阅事件；返回 unsubscribe 函数 |
| `setModel(model)` | 切 model |
| `setThinkingLevel(level)` | 切 thinking |
| `cycleModel()` / `cycleThinkingLevel()` | 循环切 |
| `compact(customInstructions?)` | 触发压缩 |
| `abortCompaction()` | 取消压缩 |
| `abort()` | 中断当前 agent 操作 |
| `dispose()` | 释放资源 |
| `navigateTree(targetId, options?)` | 在树里跳转 |

**属性**：`sessionFile`、`sessionId`、`agent`、`model`、`thinkingLevel`、`messages`、`isStreaming`。

### 4.3 `AgentSessionRuntime`（进阶）

需要"替换活动 session 并重建 cwd 绑定状态"时用。`AgentSessionRuntime` 负责处理：

- `newSession()`
- `switchSession()`
- `fork()` / `fork(id, { position: "at" })`（即 clone）
- `importFromJsonl()`

> 注意：**事件订阅是绑定在具体 `AgentSession` 上的**，session 被替换后要重新订阅。

```ts
const createRuntime: CreateAgentSessionRuntimeFactory = async ({
  cwd, sessionManager, sessionStartEvent,
}) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({
      services, sessionManager, sessionStartEvent,
    })),
    services,
    diagnostics: services.diagnostics,
  };
};

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});
```

## 5. Prompt 选项

```ts
interface PromptOptions {
  expandPromptTemplates?: boolean;
  images?: ImageContent[];
  streamingBehavior?: "steer" | "followUp";  // streaming 时必填
  source?: InputSource;
  preflightResult?: (success: boolean) => void;  // 每次 prompt() 调一次
}
```

- Extension 命令（如 `/mycommand`）**立刻执行**——即使在 streaming 中
- 文件型 `.md` template 在发送前展开

## 6. 事件类型

通过 `session.subscribe(listener)` 订阅。事件分组：

| 分组 | 事件 |
|------|------|
| **Streaming** | `message_update`（含 `text_delta`、`thinking_delta`） |
| **Tools** | `tool_execution_start`、`tool_execution_update`、`tool_execution_end` |
| **Messages** | `message_start`、`message_end` |
| **Agent** | `agent_start`、`agent_end` |
| **Turns** | `turn_start`、`turn_end` |
| **Session** | `queue_update`、`compaction_start/end`、`auto_retry_start/end` |

事件字段细节参考 [[18-JSON-事件流]]——SDK 事件与 JSON 模式同源。

## 7. 配置项详解

### 7.1 目录

| 选项 | 默认 | 说明 |
|------|------|------|
| `cwd` | `process.cwd()` | 项目资源发现根（`.pi/extensions`、`.pi/skills`、`.pi/prompts`、`AGENTS.md`） |
| `agentDir` | `~/.pi/agent` | 全局配置目录（含 `settings.json`、`models.json`、`auth.json`、`sessions/`） |

### 7.2 Model

```ts
import { getModel } from "@earendil-works/pi-ai";

const opus = getModel("anthropic", "claude-opus-4-5");
const customModel = modelRegistry.find("my-provider", "my-model");
const available = await modelRegistry.getAvailable();
```

Thinking levels：`off, minimal, low, medium, high, xhigh`。`scopedModels` 定义循环顺序。

不指定 model 时按以下顺序回落：**session 恢复 → settings 默认 → 第一个可用**。

### 7.3 API Keys / OAuth 解析优先级

1. Runtime 覆盖（`setRuntimeApiKey`）
2. `auth.json` 存储的凭据
3. 环境变量（`ANTHROPIC_API_KEY` 等）
4. `models.json` custom provider 的 fallback resolver

### 7.4 System Prompt 覆盖

通过 `ResourceLoader` 替换：

```ts
const loader = new DefaultResourceLoader({
  systemPromptOverride: () => "You are a helpful assistant.",
});
await loader.reload();
```

### 7.5 Tools

内置：`read`、`bash`、`edit`、`write`、`grep`、`find`、`ls`。
默认开：`read`、`bash`、`edit`、`write`。

- `noTools: "all"` 全关
- `noTools: "builtin"` 关掉默认，保留 extension/自定义 tool

### 7.6 自定义 Tool

```ts
import { Type } from "typebox";
import { defineTool } from "@earendil-works/pi-coding-agent";

const myTool = defineTool({
  name: "my_tool",
  label: "My Tool",
  description: "Does something useful",
  parameters: Type.Object({
    input: Type.String({ description: "Input value" }),
  }),
  execute: async (_toolCallId, params) => ({
    content: [{ type: "text", text: `Result: ${params.input}` }],
    details: {},
  }),
});

const { session } = await createAgentSession({ customTools: [myTool] });
```

### 7.7 Extensions

`ResourceLoader` 加载。可传 `additionalExtensionPaths` 或内联 `extensionFactories`。Extensions 能注册 tool、订阅事件、加命令。多 extension 间用 `createEventBus()` 通信。

### 7.8 Skills

通过 `DefaultResourceLoader` 的 `skillsOverride` 提供。Skill 形状：`{ name, description, filePath, baseDir, source }`。

### 7.9 Context Files

通过 `agentsFilesOverride` 注入虚拟的 `AGENTS.md` 内容。

### 7.10 Slash 命令（prompt template）

通过 `promptsOverride` 提供：`{ name, description, source, content }`。

### 7.11 Session 管理

| 工厂 | 用途 |
|------|------|
| `SessionManager.inMemory()` | 不存盘 |
| `SessionManager.create(process.cwd())` | 新建持久 session |
| `SessionManager.continueRecent(process.cwd())` | 续接最新 |
| `SessionManager.open("/path/to/session.jsonl")` | 打开指定文件 |
| `SessionManager.list(process.cwd())` | 列项目 session |
| `SessionManager.listAll(process.cwd())` | 列所有 session |

**树相关 API**：`getEntries()`、`getTree()`、`getPath()`、`getLeafEntry()`、`getEntry(id)`、`getChildren(id)`、`getLabel(id)`、`appendLabelChange(id, label)`、`branch(id)`、`branchWithSummary(id, summary)`、`createBranchedSession(leafId)`。

完整 API 见 [[15-Session-文件格式#SessionManager-API]]。

### 7.12 Settings

| 工厂 | 用途 |
|------|------|
| `SettingsManager.create(cwd?, agentDir?)` | 文件 backed（merge `~/.pi/agent/settings.json` + `<cwd>/.pi/settings.json`） |
| `SettingsManager.inMemory(settings?)` | 不读写文件 |

实例方法：

| 方法 | 用途 |
|------|------|
| `applyOverrides({...})` | 临时覆盖 |
| `flush()` | 持久化边界（await 之后才保证写盘） |
| `drainErrors()` | 取出 I/O 错误（不自动打印） |

## 8. 运行 Modes

Runtime 之上的三种内置 mode：

| Mode | 用途 |
|------|------|
| `InteractiveMode` | 完整 TUI（editor、chat history、内置命令） |
| `runPrintMode(runtime, { mode: "text", initialMessage, initialImages, messages })` | one-shot，跑完输出退出 |
| `runRpcMode(runtime)` | JSON-RPC 子进程集成 |

## 9. 完整导出列表

```text
// Factory
createAgentSession, createAgentSessionRuntime, AgentSessionRuntime

// Auth and Models
AuthStorage, ModelRegistry

// Resource loading
DefaultResourceLoader, type ResourceLoader, createEventBus

// Helpers
defineTool

// Session management
SessionManager, SettingsManager

// Tool factories
createCodingTools, createReadOnlyTools
createReadTool, createBashTool, createEditTool, createWriteTool
createGrepTool, createFindTool, createLsTool

// Types
CreateAgentSessionOptions, CreateAgentSessionResult,
ExtensionFactory, ExtensionAPI, ToolDefinition,
Skill, PromptTemplate, Tool
```

## 10. 完整示例

```ts
import { getModel } from "@earendil-works/pi-ai";
import { Type } from "typebox";
import {
  AuthStorage, createAgentSession, DefaultResourceLoader,
  defineTool, ModelRegistry, SessionManager, SettingsManager,
} from "@earendil-works/pi-coding-agent";

const authStorage = AuthStorage.create("/custom/agent/auth.json");
if (process.env.MY_KEY)
  authStorage.setRuntimeApiKey("anthropic", process.env.MY_KEY);

const modelRegistry = ModelRegistry.create(authStorage);
const model = getModel("anthropic", "claude-opus-4-5");
if (!model) throw new Error("Model not found");

const statusTool = defineTool({
  name: "status",
  label: "Status",
  description: "Get system status",
  parameters: Type.Object({}),
  execute: async () => ({
    content: [{ type: "text", text: `Uptime: ${process.uptime()}s` }],
    details: {},
  }),
});

const settingsManager = SettingsManager.inMemory({
  compaction: { enabled: false },
  retry: { enabled: true, maxRetries: 2 },
});

const loader = new DefaultResourceLoader({
  cwd: process.cwd(),
  agentDir: "/custom/agent",
  settingsManager,
  systemPromptOverride: () => "You are a minimal assistant. Be concise.",
});
await loader.reload();

const { session } = await createAgentSession({
  cwd: process.cwd(),
  agentDir: "/custom/agent",
  model,
  thinkingLevel: "off",
  authStorage,
  modelRegistry,
  tools: ["read", "bash", "status"],
  customTools: [statusTool],
  resourceLoader: loader,
  sessionManager: SessionManager.inMemory(),
  settingsManager,
});

session.subscribe((event) => {
  if (
    event.type === "message_update" &&
    event.assistantMessageEvent.type === "text_delta"
  ) {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("Get status and list files.");
```
