# 14 - Custom Providers：自定义 Provider

> 来源：https://pi.dev/docs/latest/custom-provider

通过 extension 的 `pi.registerProvider()` 注册自定义 model provider——能跑 proxy、self-hosted 端点、OAuth/SSO 流程、非标准 streaming API。

跟 [[13-Custom-Models-自定义模型]] 的区别：

| 维度 | 13 - Custom Models | 14 - Custom Providers |
|------|--------------------|----------------------|
| 形式 | JSON 配置（`models.json`） | TypeScript extension |
| 复杂度 | 低 | 高 |
| 能力 | 改 baseUrl、加 model、调 compat flag | 自定义 OAuth、自定义 stream 实现、动态发现 |
| 适合 | 套用现成 API 类型 | 协议非标 / 要 OAuth / 要动态 |

前置知识：[[08-Extensions-扩展编写]]。

## 1. 基础注册

### 1.1 覆盖现有 provider（仅改 baseUrl 走 proxy）

```ts
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com"
});
```

只给 `baseUrl` 和/或 `headers` 时，**已有的 model 列表保留不变**。

### 1.2 注册全新 provider

带上 `models` 字段——它会**替换该 provider 名下之前的 model 列表**。

### 1.3 注销

```ts
pi.unregisterProvider(name);
```

清掉动态 model、OAuth 注册、stream handler。被覆盖的内置行为**恢复**。初次加载后无需 `/reload`。

### 1.4 Async factory

Extension factory 可以是 `async`——典型用途是**动态 model 发现**（启动前 fetch `/v1/models`）。

## 2. API 类型

`api` 字段选 streaming 实现，候选值：

| 值 | 用途 |
|----|------|
| `anthropic-messages` | Anthropic Messages |
| `openai-completions` | OpenAI Chat Completions（**最兼容**） |
| `openai-responses` | OpenAI Responses |
| `azure-openai-responses` | Azure OpenAI Responses |
| `openai-codex-responses` | OpenAI Codex Responses |
| `mistral-conversations` | Mistral |
| `google-generative-ai` | Google Generative AI |
| `google-vertex` | Google Vertex AI |
| `bedrock-converse-stream` | Amazon Bedrock Converse |

**多数 OpenAI 兼容服务用 `openai-completions`**，靠 model 级的 `compat` flag 和 `thinkingLevelMap` 调差异。

### Auth header 快捷方式

端点只需要 `Authorization: Bearer <key>`：

```ts
pi.registerProvider("custom-api", {
  baseUrl: "https://api.example.com",
  apiKey: "MY_API_KEY",
  authHeader: true,
  api: "openai-completions",
  models: []
});
```

## 3. OAuth / SSO

provider 可以挂一个 `oauth` 对象——跟 `/login` 命令集成。

### 3.1 `login` 回调收的 `OAuthLoginCallbacks`

| 方法 | 用途 |
|------|------|
| `onAuth({ url })` | 打开浏览器 URL |
| `onDeviceCode({ userCode, verificationUri, intervalSeconds?, expiresInSeconds? })` | 显示 device code |
| `onPrompt({ message })` | 收一行文本响应 |
| `onSelect({ message, options })` | 显示选择器 |

### 3.2 oauth 对象必须实现

- `login`
- `refreshToken`
- `getApiKey`
- 可选 `modifyModels(models, credentials)`——基于用户 session 改 model 列表（如按区域换端点）

### 3.3 凭据持久化

存到 `~/.pi/agent/auth.json`：

```ts
interface OAuthCredentials {
  refresh: string;
  access: string;
  expires: number;  // ms 时间戳
}
```

注册后用户用 `/login <provider-name>` 登录。

## 4. 自定义 Streaming API

协议非标的话，提供 `streamSimple`。骨架：

```ts
function streamMyProvider(
  model: Model<any>,
  context: Context,
  options?: SimpleStreamOptions
): AssistantMessageEventStream {
  const stream = createAssistantMessageEventStream();

  (async () => {
    const output: AssistantMessage = {
      role: "assistant",
      content: [],
      api: model.api,
      provider: model.provider,
      model: model.id,
      usage: { /* ...zeros... */ },
      stopReason: "stop",
      timestamp: Date.now(),
    };

    try {
      stream.push({ type: "start", partial: output });
      // ...发请求、push content event...
      stream.push({ type: "done", reason: output.stopReason, message: output });
      stream.end();
    } catch (error) {
      output.stopReason = options?.signal?.aborted ? "aborted" : "error";
      output.errorMessage = error instanceof Error ? error.message : String(error);
      stream.push({ type: "error", reason: output.stopReason, error: output });
      stream.end();
    }
  })();

  return stream;
}
```

### 事件序列

先 push 一个 `start`，然后任意多个 content 事件（text / thinking / toolcall——每种都有 `_start` / `_delta` / `_end` 三态变体，按 `contentIndex` 索引），最后 `done` 或 `error`。

**每个事件都带演化中的 `partial` 消息状态**。

### Tool call 处理

streaming 来的 JSON 片段累积——**每次 delta 后试 `JSON.parse`，失败时忽略**，等 call end 时再保证拿到完整 JSON。

### Usage 与 cost

从上游响应填充，最后用 `calculateCost(model, output.usage)` finalize。

## 5. Context Overflow 处理

Pi 在 stream 以 `stopReason === "error"` 且 `errorMessage` 匹配已知 overflow pattern 时，**自动 compact 并 retry**。

provider 用别的措辞报 overflow 错误时，在同一个 extension 里用 `message_end` handler **规整化**：错误消息前缀加 `context_length_exceeded:`，Pi 的探测器就识别了。

**重要 guard**：

- 按 `message.provider` 和 `ctx.model?.provider` 限定作用范围
- 用 **provider 特定的正则** 匹配——不要把 rate-limit 错误改写了（rate-limit 走 Pi 的 retry path）
- 如果消息已经包含 `context_length_exceeded`，跳过（保证幂等）

## 6. ProviderConfig 接口

所有字段（除特别说明外）可选：

| 字段 | 说明 |
|------|------|
| `name` | 显示名 |
| `baseUrl` | 端点（定义 model 时必需） |
| `apiKey` | 字面值或环境变量名（定义 model 且不用 oauth 时必需） |
| `api` | 支持的 API 类型之一 |
| `streamSimple` | 自定义 stream 函数 |
| `headers` | 额外 header（值也可以是环境变量名） |
| `authHeader` | true 时自动加 `Authorization: Bearer ...` |
| `models` | 给了就**替换**已有 model 列表 |
| `oauth` | `{ name, login, refreshToken, getApiKey, modifyModels? }` |

## 7. ProviderModelConfig 接口

| 字段 | 说明 |
|------|------|
| `id`, `name` | 标识和显示名 |
| `api?`, `baseUrl?` | per-model 覆盖 |
| `reasoning: boolean` | 是否支持 extended thinking |
| `thinkingLevelMap?` | Pi level → provider 字符串；`null` 表示不支持 |
| `input: ("text" \| "image")[]` | 输入类型 |
| `cost: { input, output, cacheRead, cacheWrite }` | 每 million token |
| `contextWindow`, `maxTokens` | 上下文和最大输出 |
| `headers?` | per-model header |
| `compat?` | 见下 |

## 8. `compat` Flags

覆盖 OpenAI 兼容 / Anthropic 兼容 provider 的各种 quirk：

### OpenAI 侧

`supportsStore` / `supportsDeveloperRole` / `supportsReasoningEffort` / `supportsUsageInStreaming` / `maxTokensField`（`"max_completion_tokens"` 或 `"max_tokens"`）/ `requiresToolResultName` / `requiresAssistantAfterToolResult` / `requiresThinkingAsText` / `requiresReasoningContentOnAssistantMessages` / `thinkingFormat`（`openai` / `openrouter` / `deepseek` / `together` / `zai` / `qwen` / `qwen-chat-template`）/ `cacheControlFormat: "anthropic"`

### Anthropic 侧

`supportsEagerToolInputStreaming` / `supportsLongCacheRetention` / `sendSessionAffinityHeaders` / `supportsCacheControlOnTools` / `forceAdaptiveThinking`

### thinkingFormat 取值差异

不同值对应不同的 wire 格式。举例：

- `qwen` 是 DashScope 的顶层 `enable_thinking`
- `qwen-chat-template` 目标是本地 Qwen server 读取的 `chat_template_kwargs.enable_thinking`

## 9. 测试

文档建议**复制 `packages/ai/test/` 下的测试文件**跑在自己的 provider/model 上：

- `stream.test.ts`
- `tokens.test.ts`
- `abort.test.ts`
- `context-overflow.test.ts`
- `cross-provider-handoff.test.ts`

跑通确保跟内置 provider 行为对齐。

## 10. 参考 extension

文档点名两个完整示例：

- `custom-provider-anthropic`
- `custom-provider-gitlab-duo`
