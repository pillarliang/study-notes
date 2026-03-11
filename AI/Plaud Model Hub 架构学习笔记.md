# Plaud Model Hub 架构学习笔记

> 基于项目源码和设计文档整理，帮助快速掌握整个项目的设计理念、架构实现和使用方式。

---

## 一、项目定位

Model Hub 是 Plaud AI Platform 的 **统一大模型调用基础设施层**。

**解决的核心问题：**

- 模型多源（OpenAI / Claude / Gemini / 国产等），各自 SDK、API、Key 管理分散
- 各业务系统直接调用 Provider：加模型/换模型都要改代码+发版
- 缺少统一的成本统计、调用观测、熔断降级能力

**核心理念：**

> 业务方只需关心 `app_id` + `logical_model`，供应商 Key、路由策略、熔断降级全部由 Model Hub 内部管理。

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                         业务层                               │
│  SDK Wrapper (OpenAI/Anthropic/GenAI)                       │
│  ModelHubClient (统一 API)                                   │
│  LangChain Integration (ChatModelHub/EmbeddingsModelHub)    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                      CoreEngine                              │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────────┐    │
│  │  Router   │  │ Plugin   │  │ Fallback / Retry       │    │
│  │ 路由策略  │  │ Chain    │  │ 降级 / 重试 / 退避     │    │
│  └──────────┘  └──────────┘  └────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    Provider 适配层                            │
│  OpenAI │ Anthropic │ GenAI │ Azure │ Bedrock │ Volcengine  │
│  DashScope │ Vertex AI                                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                      配置层                                   │
│  FileConfigStore (YAML/JSON)  │  AppConfigConfigStore (AWS) │
└─────────────────────────────────────────────────────────────┘
```

**Monorepo 结构：**

| 包 | 路径 | 职责 |
|---|---|---|
| `model-hub-core` | `packages/core/` | 核心引擎，零硬依赖 |
| `model-hub-sdk` | `packages/sdk-python/` | Python SDK，依赖 core |

---

## 三、4 种使用方式

### 3.1 SDK Wrapper（推荐，迁移成本最低）

```python
from model_hub_sdk import wrap_openai
from model_hub_sdk.config import FileConfigStore

client = wrap_openai(
    app_id="summary",
    config_store=FileConfigStore("./config.yaml"),
)

# 与原生 OpenAI SDK 完全相同的 API
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello!"}]
)
```

**原理：** 拦截原生 SDK 方法调用 → 转 ModelRequest → CoreEngine 路由 → 转回原生响应格式

支持：`wrap_openai` / `wrap_async_openai` / `wrap_anthropic` / `wrap_async_anthropic` / `wrap_genai`

### 3.2 ModelHubClient（统一 API）

```python
from model_hub_sdk import ModelHubClient

client = ModelHubClient(app_id="summary", env="prod")

response = client.chat(model="gpt-4.1", messages=[...])
embeddings = client.embed(model="text-embedding-3-small", input=["Hello"])
images = client.generate_image(model="dall-e-3", prompt="A sunset")
text = client.transcribe(model="whisper-1", audio_file=open("audio.mp3", "rb"))
```

### 3.3 LangChain 集成

```python
from model_hub_sdk.integrations.langchain import ChatModelHub, EmbeddingsModelHub

llm = ChatModelHub(app_id="ai-agent", model="gpt-4.1")
result = llm.invoke("Hello, world!")

# 支持 Tool Binding
llm_with_tools = llm.bind_tools([my_tool])

# 支持 Embedding
embeddings = EmbeddingsModelHub(app_id="ai-agent", model="text-embedding-3-small")
```

### 3.4 获取原生 Client

```python
client = ModelHubClient(app_id="summary", config_store=FileConfigStore("./config.yaml"))
openai_client = client.get_openai_client(model="gpt-4.1")
# 注意：不经过 CoreEngine，不享受路由/重试/熔断能力
```

---

## 四、配置系统

### 4.1 v2 配置格式（推荐）

```yaml
config_schema: v2
version: 1
env: dev

# ① Provider 预设（定义一次，多处引用）
providers:
  litellm:
    provider: openai
    base_url: https://litellm.example.com/v1
    api_key: ${LITELLM_API_KEY}
    defaults:
      timeout_ms: 30000

  anthropic:
    provider: anthropic
    api_key: ${ANTHROPIC_API_KEY}

# ② 模型配置（3 种格式）
models:
  # 字符串快捷语法
  ai-demo:gpt-4: litellm/gpt-4-turbo

  # 简化对象（多 endpoint）
  ai-demo:gpt-4-ha:
    endpoints:
      - provider_ref: litellm
        model: gpt-4-turbo
        weight: 70
      - provider_ref: openai
        model: gpt-4-turbo
        weight: 30

# ③ 路由策略
policies:
  default:
    strategy: weighted_random
  summary:claude-sonnet:
    strategy: priority
    fallback_model: summary:gpt-4

# ④ 插件
plugins:
  langfuse:
    enabled: true
  circuit_breaker:
    enabled: true
```

### 4.2 Key 格式

```
{app_id}:{logical_model}
```

例如：`project-summary:gemini-2.5-pro`

### 4.3 支持的 Provider

| Provider | 类型标识 | 必填 |
|---|---|---|
| OpenAI | `openai` | api_key |
| Azure OpenAI | `azure_openai` | api_key, base_url |
| Anthropic | `anthropic` | api_key |
| Google GenAI | `genai` | api_key |
| Vertex AI | `vertex_ai` | project_id, location |
| 火山引擎 | `volcengine` | api_key |
| 阿里云 DashScope | `dashscope` | api_key |
| AWS Bedrock | `bedrock` | region |

### 4.4 配置属性优先级

```
endpoint.extra > credential.extra > provider.defaults > 代码默认值
```

---

## 五、CoreEngine 核心引擎

### 5.1 请求完整生命周期

```
invoke(request)
  │
  ├── Fallback 循环检测 + 深度限制（max_fallback_depth=3）
  │
  └── _invoke_internal(request)
        │
        ├── 1. 请求验证（按 request_type 校验必填字段）
        ├── 2. before_request 插件链
        │
        ├── 3. 重试循环（attempt = 0 → max_retries）
        │     │
        │     ├── 路由选择（排除已失败 endpoint）
        │     ├── Provider 调用
        │     │
        │     ├── 成功 → 通知 Hook → 填充元数据 → after_response 插件链 → 返回
        │     │
        │     └── 失败 →
        │           ├── 429 首次 → 退避重试（同 endpoint）
        │           ├── 429 再次 → failover 到其他 endpoint
        │           ├── 5xx → 直接 failover
        │           └── 4xx → 检查其他 endpoint 或抛错
        │
        └── 4. 所有 endpoint 耗尽 → NoAvailableEndpointError
              │
              └── 有 fallback_model? → 递归调用 _invoke_with_fallback(depth+1)
```

### 5.2 重试与退避

| 参数 | 默认值 | 说明 |
|---|---|---|
| `default_max_retries` | 2 | 最大重试次数 |
| `retry_on_status_codes` | 408,409,429,500-504,529 | 可重试状态码 |
| `backoff_factor` | 1.0 | 退避基数 |
| `max_backoff_seconds` | 10.0 | 最大退避时间 |
| `jitter_factor` | 0.25 | 抖动 ±25% |

**退避公式：** `backoff = factor × 2^attempt + jitter`

- attempt=0 → ~1s, attempt=1 → ~2s, attempt=2 → ~4s (上限 10s)

### 5.3 429 处理策略

| 策略 | 行为 |
|---|---|
| `retry_after_first`（默认） | 优先用 Retry-After 头，无则指数退避 |
| `exponential_only` | 忽略 Retry-After，始终指数退避 |
| `failover_immediately` | 无 Retry-After 时立即切换 endpoint |

**处理流程：** 首次 429 → 同 endpoint 退避重试 → 再次 429 → failover → 全部 429 → fallback 模型

### 5.4 Fallback 降级

```yaml
policies:
  summary:claude-sonnet:
    fallback_model: summary:gpt-4
  summary:gpt-4:
    fallback_model: summary:gpt-3.5
```

安全机制：
- **循环检测**：A→B→A 会被检测并中止
- **深度限制**：默认最多 3 层
- **RetryStats** 记录原始模型、最终模型、所有中间模型

### 5.5 三种路由策略

| 策略 | 说明 | 适用场景 |
|---|---|---|
| **weighted_random**（默认） | 按 weight 比例随机分配 | 一般负载均衡 |
| **round_robin** | 线程安全计数器轮流分配 | 均匀分配 |
| **priority** | 选最高优先级组，组内加权随机 | 主备模式 |

**会话粘性（Session Affinity）：** 提供 session_id 时，MD5 一致性哈希确保同一 session 路由到同一 endpoint。

---

## 六、插件系统

### 6.1 执行顺序

| 插件 | Order | 作用 |
|---|---|---|
| RateLimit | 5 | 限流（最先） |
| CircuitBreaker | 10 | 熔断 |
| OTel | 10 | OpenTelemetry 追踪 |
| Langfuse | 200 | LLM 专用追踪（最后） |

### 6.2 生命周期钩子

```
before_request(request, context) → 可修改请求
    ↓
Provider 调用
    ↓
after_response(request, response, context) → 可修改响应
    ↓
on_error(request, error, context) → 错误处理
```

高级钩子（Engine 级别）：

| 接口 | 方法 | 实现者 |
|---|---|---|
| `EndpointFilterProvider` | `get_unavailable_endpoints()` | 熔断器、限流器 |
| `InvocationLifecycleHook` | `on_invocation_success/failure()` | 熔断器 |
| `RetryInfoProvider` | `get_suggested_backoff()` | 限流器 |
| `RetryLifecycleHook` | `on_retry_attempt/exhausted()` | 可观测性插件 |

### 6.3 熔断器（Circuit Breaker）

**状态机：**

```
CLOSED（正常）
  │ 失败率 > threshold 且 calls >= min_calls
  ▼
OPEN（熔断）
  │ 等待 open_duration_seconds
  ▼
HALF_OPEN（半开探测）
  │ 探测全部成功 → CLOSED
  │ 任一失败 → OPEN
```

| 参数 | 默认值 | 说明 |
|---|---|---|
| `failure_threshold` | 0.5 | 失败率阈值 50% |
| `min_calls` | 5 | 触发评估的最小调用数 |
| `window_size_seconds` | 60 | 滑动窗口 60s |
| `open_duration_seconds` | 30 | 熔断持续 30s |
| `half_open_max_calls` | 3 | 半开探测次数 |

实现：滑动窗口用 deque 存 `(timestamp, is_success)` 元组，线程安全（RLock）。

### 6.4 限流器（Rate Limiter）

**双重机制：**

| 类型 | 机制 | 说明 |
|---|---|---|
| 主动限流 | Token Bucket | 客户端侧令牌桶，控制请求速率 |
| 被动限流 | 429 响应解析 | 解析 Retry-After / X-RateLimit-* 头 |

> ⚠️ **关键配置**：`exclude_429_from_circuit_breaker: true`，防止 429 计入熔断统计（双重惩罚问题）。

### 6.5 Langfuse 插件

- before_request → `start_generation(name, input, metadata)`
- after_response → `generation.update(output, usage, model, latency)` + `end()`
- on_error → `generation.update(level="ERROR")` + `end()`

记录：app_id, env, logical_model, endpoint_id, provider, latency_ms, token_usage, trace_id

### 6.6 OTEL 插件

标准 OpenTelemetry Span，属性命名空间：
- `modelhub.*`：Model Hub 自定义属性
- `gen_ai.*`：OpenAI 兼容标准属性

---

## 七、Provider 适配层

### 7.1 统一接口

```python
class ModelProvider(ABC):
    def name() → str
    def supported_request_types() → list[RequestType]
    def invoke(request, decision) → ModelResponse
    def invoke_stream(request, decision) → Iterator[StreamChunk]
```

### 7.2 支持的请求类型矩阵

| 类型 | OpenAI | Anthropic | GenAI | Azure | Bedrock | Volcengine | DashScope |
|---|---|---|---|---|---|---|---|
| Chat | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Embedding | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Image | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Transcription | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Speech | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |

### 7.3 错误码映射

所有 Provider 错误映射为统一 `ProviderError`：

| 状态码 | 可重试 | 触发熔断 | 典型场景 |
|---|---|---|---|
| 429 | ✅ | ⚙️ 可配置 | Rate Limit |
| 500 | ✅ | ✅ | Internal Server Error |
| 503 | ✅ | ✅ | Service Unavailable |
| 529 | ✅ | ✅ | Anthropic Overloaded |
| 400 | ❌ | ❌ | Bad Request |
| 401 | ❌ | ❌ | Auth Error |

### 7.4 Client 连接池

`ClientPool`（线程安全 LRU，默认 100）：

```
缓存 Key = {provider}:{base_url}:{sha256(api_key)[:16]}
```

相同配置的 endpoint 共享 client 实例。

---

## 八、Wrapper 实现原理

### 执行链路

```
用户: client.chat.completions.create(model="gpt-4", messages=[...])
  │
  ├── WrappedOpenAI.__getattr__("chat") → 拦截
  ├── WrappedCompletions.create()
  │     ├── convert_openai_messages_to_hub()   # 消息格式转换
  │     ├── extract_openai_provider_params()   # 提取 tools, temperature 等
  │     ├── build ModelRequest                 # 构建统一请求
  │     ├── CoreEngine.invoke()               # 引擎执行
  │     └── to_openai_completion()            # 转回 OpenAI 格式
  │
  └── 返回 openai.ChatCompletion（与原生完全兼容）
```

### 响应格式转换

| 源 | → OpenAI | → Anthropic | → GenAI |
|---|---|---|---|
| ModelResponse (Chat) | ChatCompletion | Message | GenerateContentResponse |
| StreamChunk | ChatCompletionChunk | RawMessageStreamEvent | GenerateContentResponse |

---

## 九、数据模型速查

### ModelRequest 关键字段

| 字段 | 说明 |
|---|---|
| `logical_model` | 逻辑模型名 |
| `app_id` | 应用标识 |
| `request_type` | CHAT / EMBEDDING / IMAGE_GENERATION 等 |
| `messages` | Chat 消息列表 |
| `session_id` | 会话 ID（粘性路由） |
| `timeout_ms` | 请求级超时覆盖 |
| `max_retries` | 请求级重试覆盖 |
| `provider_params` | 透传给 Provider |

### ModelResponse 关键字段

| 字段 | 说明 |
|---|---|
| `choices` | Chat 响应 |
| `usage` | token 用量 |
| `model` | 物理模型名 |
| `provider` | Provider 名 |
| `endpoint_id` | 命中的 endpoint |
| `latency_ms` | 总延迟 |
| `retry_stats` | 重试统计 |

### RetryStats

| 字段 | 说明 |
|---|---|
| `retry_count` | 总重试次数 |
| `failover_count` | 切换 endpoint 次数 |
| `total_backoff_ms` | 总等待时间 |
| `endpoints_tried` | 尝试过的 endpoint |
| `fallback_models_tried` | 尝试过的 fallback 模型 |
| `original_model` / `final_model` | 原始/最终模型 |

---

## 十、最佳实践

### ✅ 推荐

- 始终启用熔断器和限流器
- `exclude_429_from_circuit_breaker: true`（防止双重惩罚）
- 配置 fallback 链防止级联故障
- 多 endpoint 使用 weighted_random + session affinity
- 开发用 Langfuse，生产用 OTEL（不要同时开）

### ❌ 避免

- 同时启用 Langfuse 和 OTEL（重复记录）
- 429 计入熔断统计
- 循环 fallback（A→B→A）
- 不配置 fallback + 单 endpoint（单点故障）
- 关闭熔断器和限流器

### 环境配置参考

**开发环境：**
```yaml
plugins:
  langfuse: { enabled: true }
  circuit_breaker: { enabled: true, failure_threshold: 0.5, min_calls: 3 }
  rate_limit: { enabled: true }
```

**生产环境：**
```yaml
plugins:
  otel: { enabled: true }
  circuit_breaker: { enabled: true, failure_threshold: 0.3, min_calls: 10 }
  rate_limit: { enabled: true, exclude_429_from_circuit_breaker: true }
```

---

## 十一、关键文件速查

| 功能 | 文件 |
|---|---|
| 核心引擎 | `packages/core/src/model_hub_core/engine.py` |
| 路由策略 | `packages/core/src/model_hub_core/router.py` |
| 数据模型 | `packages/core/src/model_hub_core/models.py` |
| 错误模型 | `packages/core/src/model_hub_core/errors.py` |
| 熔断器 | `packages/core/src/model_hub_core/plugins/circuit_breaker.py` |
| 限流器 | `packages/core/src/model_hub_core/plugins/rate_limit.py` |
| SDK Client | `packages/sdk-python/src/model_hub_sdk/client/hub_client.py` |
| Client Factory | `packages/sdk-python/src/model_hub_sdk/client/factory.py` |
| OpenAI Wrapper | `packages/sdk-python/src/model_hub_sdk/wrappers/openai_wrapper.py` |
| LangChain 集成 | `packages/sdk-python/src/model_hub_sdk/integrations/langchain/chat_model.py` |
| 配置解析 | `packages/core/src/model_hub_core/config/` |
| 完整配置示例 | `examples/config-complete.yaml` |
| 设计文档 | `docs/design.md` |