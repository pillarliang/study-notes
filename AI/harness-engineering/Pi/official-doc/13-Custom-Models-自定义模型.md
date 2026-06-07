# 13 - Custom Models：自定义模型

> 来源：https://pi.dev/docs/latest/models

通过 `~/.pi/agent/models.json` 配置自定义 provider 和 model——支持 Ollama、vLLM、LM Studio、proxy 和其它后端。

跟 [[14-Custom-Providers-自定义-Provider]] 的区别：本笔记是**配置文件层**的方案，纯 JSON 配置；自定义 provider 是**代码层**方案，要写 TypeScript extension。

## 1. 支持的 API 类型

| API | 说明 |
|-----|------|
| `openai-completions` | OpenAI Chat Completions（兼容性最好） |
| `openai-responses` | OpenAI Responses API |
| `anthropic-messages` | Anthropic Messages API |
| `google-generative-ai` | Google Generative AI |

`api` 字段可以放在 provider 层（默认）或 model 层（覆盖）。

## 2. Provider 配置字段

| 字段 | 说明 |
|------|------|
| `baseUrl` | API 端点 URL |
| `api` | API 类型 |
| `apiKey` | API key（支持三种解析形式） |
| `headers` | 自定义 headers |
| `authHeader` | true 时自动加 `Authorization: Bearer <apiKey>` |
| `models` | model 配置数组 |
| `modelOverrides` | 对内置 model 的 per-model override |

### 值解析

`apiKey` 和 `headers` 支持三种格式（同 [[03-Providers-认证与配置#auth-json-的-key-字段三种写法|auth.json 的 key 字段]]）：

| 形式 | 写法 | 行为 |
|------|------|------|
| Shell 命令 | `"!command"` | 执行命令用 stdout；**request 时解析，无内置缓存** |
| 环境变量 | `"MY_KEY"`（纯名字） | 解析为该变量当前值 |
| 字面值 | 直接写 | 直接使用 |

## 3. Model 配置字段

| 字段 | 必需 | 默认 | 说明 |
|------|------|------|------|
| `id` | ✅ | — | 发给 API 的 model 标识 |
| `name` | ❌ | `id` | 人类可读名，**用于匹配和显示** |
| `api` | ❌ | provider 的 | 覆盖 provider 的 API |
| `reasoning` | ❌ | `false` | 是否支持 extended thinking |
| `thinkingLevelMap` | ❌ | 省略 | 把 Pi thinking level 映射到 provider 取值 |
| `input` | ❌ | `["text"]` | 输入类型（text / text+image） |
| `contextWindow` | ❌ | `128000` | context 大小（token） |
| `maxTokens` | ❌ | `16384` | 最大输出 token |
| `cost` | ❌ | 全 0 | 按 million token 的定价对象 |
| `compat` | ❌ | provider 的 | 兼容性覆盖，**和 provider 级 merge** |

## 4. Pi 如何解析 model

- `/model` 和 `--list-models` 按 model **`id`** 列出
- **`name` 字段用于 pattern 匹配**（如 `--model` patterns）和详情/状态文本显示
- 配置文件**每次 `/model` 打开都重载**——不用重启
- 可用性检查靠**已配置的 auth 是否存在**，**不会跑 shell 命令**

## 5. 示例

### 5.1 最简（Ollama）

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

### 5.2 OpenAI 兼容服务器的 compat flags

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        { "id": "gpt-oss:20b", "reasoning": true }
      ]
    }
  }
}
```

### 5.3 完整字段示例

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "llama3.1:8b",
          "name": "Llama 3.1 8B (Local)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 32000,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

### 5.4 Google AI Studio

```json
{
  "providers": {
    "my-google": {
      "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
      "api": "google-generative-ai",
      "apiKey": "GEMINI_API_KEY",
      "models": [
        {
          "id": "gemma-4-31b-it",
          "name": "Gemma 4 31B",
          "input": ["text", "image"],
          "contextWindow": 262144,
          "reasoning": true
        }
      ]
    }
  }
}
```

### 5.5 Thinking Level Map

```json
{
  "id": "deepseek-v4-pro",
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": null,
    "low": null,
    "medium": null,
    "high": "high",
    "xhigh": "max"
  }
}
```

Key 范围：`off / minimal / low / medium / high / xhigh`。值是三态：

| 写法 | 行为 |
|------|------|
| 省略 key | 默认映射 |
| 字符串 | 发给 provider |
| `null` | 不支持，UI 隐藏 |

### 5.6 自定义 Headers

```json
{
  "providers": {
    "custom-proxy": {
      "baseUrl": "https://proxy.example.com/v1",
      "apiKey": "MY_API_KEY",
      "api": "anthropic-messages",
      "headers": {
        "x-portkey-api-key": "PORTKEY_API_KEY",
        "x-secret": "!op read 'op://vault/item/secret'"
      },
      "models": []
    }
  }
}
```

### 5.7 覆盖内置 provider

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1"
    }
  }
}
```

**Merge 规则**：

- 内置 model **保留**
- 自定义 model 按 `id` **upsert**
- 同 `id` 替换内置
- 新 `id` 追加

### 5.8 Per-model 覆盖

```json
{
  "providers": {
    "openrouter": {
      "modelOverrides": {
        "anthropic/claude-sonnet-4": {
          "name": "Claude Sonnet 4 (Bedrock Route)",
          "compat": {
            "openRouterRouting": {
              "only": ["amazon-bedrock"]
            }
          }
        }
      }
    }
  }
}
```

支持的 override 字段：`name` / `reasoning` / `input` / `cost` / `contextWindow` / `maxTokens` / `headers` / `compat`。

## 6. Anthropic Messages 兼容性字段

| 字段 | 用途 |
|------|------|
| `supportsEagerToolInputStreaming` | 接受 per-tool `eager_input_streaming`（默认 true） |
| `supportsLongCacheRetention` | 接受 `cache_control.ttl: "1h"` 长 cache 保留 |
| `sendSessionAffinityHeaders` | 发送 `x-session-affinity` header |
| `supportsCacheControlOnTools` | tool 上接受 cache_control 标记 |
| `forceAdaptiveThinking` | 用 adaptive thinking 代替老的 budget-based |

## 7. OpenAI 兼容性字段

| 字段 | 用途 |
|------|------|
| `supportsStore` | 支持 `store` 字段 |
| `supportsDeveloperRole` | 用 `developer` 而不是 `system` role |
| `supportsReasoningEffort` | 支持 `reasoning_effort` 参数 |
| `supportsUsageInStreaming` | 支持 streaming 的 usage 包含 |
| `maxTokensField` | `max_completion_tokens` 或 `max_tokens` |
| `requiresToolResultName` | tool result 上要带 `name` |
| `requiresAssistantAfterToolResult` | tool result 后插一条 assistant 消息 |
| `requiresThinkingAsText` | 把 thinking block 转纯文本 |
| `requiresReasoningContentOnAssistantMessages` | 加空的 `reasoning_content` |
| `thinkingFormat` | `reasoning_effort` / `openrouter` / `deepseek` / `together` / `zai` / `qwen` / `qwen-chat-template` |
| `cacheControlFormat` | 当前只支持 `anthropic` |
| `supportsStrictMode` | tool 定义里加 `strict` |
| `supportsLongCacheRetention` | 长 cache 保留 |
| `openRouterRouting` | OpenRouter 路由偏好 |
| `vercelGatewayRouting` | Vercel AI Gateway 路由 |

### OpenRouter Routing 示例

```json
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "apiKey": "OPENROUTER_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "openrouter/anthropic/claude-3.5-sonnet",
          "name": "OpenRouter Claude 3.5 Sonnet",
          "compat": {
            "openRouterRouting": {
              "allow_fallbacks": true,
              "require_parameters": false,
              "data_collection": "deny",
              "zdr": true,
              "enforce_distillable_text": false,
              "order": ["anthropic", "amazon-bedrock", "google-vertex"],
              "only": ["anthropic", "amazon-bedrock"],
              "ignore": ["gmicloud", "friendli"],
              "quantizations": ["fp16", "bf16"],
              "sort": { "by": "price", "partition": "model" },
              "max_price": { "prompt": 10, "completion": 20 },
              "preferred_min_throughput": { "p50": 100, "p90": 50 },
              "preferred_max_latency": { "p50": 1, "p90": 3, "p99": 5 }
            }
          }
        }
      ]
    }
  }
}
```

### Vercel AI Gateway 示例

```json
{
  "providers": {
    "vercel-ai-gateway": {
      "baseUrl": "https://ai-gateway.vercel.sh/v1",
      "apiKey": "AI_GATEWAY_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "moonshotai/kimi-k2.5",
          "name": "Kimi K2.5 (Fireworks via Vercel)",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 0.6, "output": 3, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 262144,
          "maxTokens": 262144,
          "compat": {
            "vercelGatewayRouting": {
              "only": ["fireworks", "novita"],
              "order": ["fireworks", "novita"]
            }
          }
        }
      ]
    }
  }
}
```
