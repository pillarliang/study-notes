# Plaud Model Hub 架构图

基于 `plaud-model-hub` 代码仓库与《PlaudModelHub完全指南》整理。

---

## 一、Monorepo 包依赖关系

```
┌─────────────────────────────────────────────────────────────────┐
│                         plaud-model-hub/                         │
│                                                                  │
│   packages/                                                      │
│   ├── core/         → model-hub-core   （零硬依赖核心引擎）         │
│   └── sdk-python/   → model-hub-sdk    （依赖 core）              │
│                                                                  │
│   services/                                                      │
│   └── model-hub-api/    FastAPI 服务（依赖 core，不依赖 sdk）       │
│                                                                  │
│   schemas/         配置 JSON Schema                               │
│   examples/        集成示例                                        │
│   tools/           开发工具                                        │
└─────────────────────────────────────────────────────────────────┘

依赖规则：
    Core    ──▶ 不依赖 SDK、不依赖 Transport
    SDK     ──▶ 依赖 Core
    Service ──▶ 依赖 Core，不依赖 SDK
```

---

## 二、分层总览（自上而下）

```
╔═══════════════════════════════════════════════════════════════════════╗
║                        ① 业务接入层 (SDK)                              ║
║   packages/sdk-python/src/model_hub_sdk/                              ║
║                                                                       ║
║   ┌─ Wrappers (原生 SDK 透传) ──────────────────────────────────────┐  ║
║   │  wrap_openai / wrap_async_openai                              │  ║
║   │  wrap_anthropic / wrap_async_anthropic                        │  ║
║   │  wrap_genai / wrap_async_genai                                │  ║
║   └───────────────────────────────────────────────────────────────┘  ║
║   ┌─ ModelHubClient (统一 API) ───────────────────────────────────┐   ║
║   │  chat / embed / generate_image / transcribe / speak /        │   ║
║   │  moderate / rerank / get_openai_client …                     │   ║
║   └──────────────────────────────────────────────────────────────┘   ║
║   ┌─ Integrations ─────────────────────────────────────────────┐     ║
║   │  LangChain: ChatModelHub / EmbeddingsModelHub              │     ║
║   └────────────────────────────────────────────────────────────┘     ║
╚══════════════════════════════╤════════════════════════════════════════╝
                               │  ModelRequest / RoutingContext
                               ▼
╔═══════════════════════════════════════════════════════════════════════╗
║                     ② Transport 传输层                                 ║
║   sdk/transport/                                                      ║
║   ┌──────────────────────────┐  ┌────────────────────────────────┐   ║
║   │ DirectProviderTransport  │  │ HttpApiTransport (规划中, P2)   │   ║
║   │ 进程内调用 CoreEngine      │  │ 通过 HTTP 调用 model-hub-api    │   ║
║   └──────────────────────────┘  └────────────────────────────────┘   ║
╚══════════════════════════════╤════════════════════════════════════════╝
                               │
                               ▼
╔═══════════════════════════════════════════════════════════════════════╗
║                     ③ CoreEngine 执行引擎                              ║
║   packages/core/src/model_hub_core/engine.py                          ║
║                                                                       ║
║   ┌─────────────────────────────────────────────────────────────┐   ║
║   │                   请求生命周期                                │   ║
║   │                                                             │   ║
║   │  ① Router 选 Endpoint ──▶ ② Plugin before_request          │   ║
║   │        ▲                        │                           │   ║
║   │        │                        ▼                           │   ║
║   │        │                  ③ Provider.invoke                 │   ║
║   │        │                        │                           │   ║
║   │        │             ┌──────────┴──────────┐                │   ║
║   │        │             │                     │                │   ║
║   │        │         成功                    失败               │   ║
║   │        │             │                     │                │   ║
║   │        │             ▼                     ▼                │   ║
║   │        │   ④ Plugin after_response   ⑤ Retry/Failover      │   ║
║   │        │             │                     │                │   ║
║   │        │             ▼                     └── loop ─▶      │   ║
║   │        │         return                                    │   ║
║   │        │                                                   │   ║
║   │        └─── ⑥ 所有 endpoint 耗尽 → Fallback 换 logical model│   ║
║   └─────────────────────────────────────────────────────────────┘   ║
║                                                                      ║
║   ┌─ Router (router.py) ──────────┐  ┌─ PluginChain ───────────┐   ║
║   │ weighted / round-robin /      │  │ EndpointFilterProvider  │   ║
║   │ priority / session-affinity   │  │ WeightAdjustmentProvider│   ║
║   │ tried_endpoints 排除集合        │  │ InvocationLifecycleHook │   ║
║   └──────────────────────────────┘  │ RetryInfoProvider       │   ║
║                                     │ RetryLifecycleHook      │   ║
║   ┌─ Retry / Fallback ───────────┐  └─────────────────────────┘   ║
║   │ 指数退避 + 抖动                │                                 ║
║   │ Retry-After 头优先             │  内置状态码: 408/409/429/5xx   ║
║   │ Fallback Chain / List 两种模式 │  max_fallback_depth 防递归    ║
║   └──────────────────────────────┘                                 ║
╚══════════════════════════════╤════════════════════════════════════════╝
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
╔════════════════════╗ ╔═══════════════════╗ ╔══════════════════════╗
║  ④ Plugin 插件层    ║ ║  ⑤ Provider 适配层 ║ ║  ⑥ Config 配置层      ║
║  core/plugins/     ║ ║  core/providers/   ║ ║  core/config/        ║
║                    ║ ║                    ║ ║                      ║
║ • rate_limit       ║ ║ Registry           ║ ║ ConfigStore (抽象)    ║
║   令牌桶+429解析    ║ ║ (registry.py)      ║ ║   │                  ║
║ • circuit_breaker  ║ ║   │                ║ ║   ├── FileConfigStore ║
║   CLOSED/OPEN/     ║ ║   ├── openai       ║ ║   │     YAML / JSON  ║
║   HALF_OPEN        ║ ║   ├── azure_openai ║ ║   │                  ║
║ • adaptive_weight  ║ ║   ├── anthropic    ║ ║   ├── AppConfigStore  ║
║   滑窗 429 渐进降权  ║ ║   ├── genai        ║ ║   │     AWS AppConf  ║
║                    ║ ║   ├── google       ║ ║   │                  ║
║ SDK 侧可选插件:      ║ ║   ├── vertex_ai    ║ ║   └── HttpConfigStore ║
║ • langfuse (成本)   ║ ║   ├── bedrock      ║ ║         model-hub-api║
║ • otel    (指标)    ║ ║   ├── volcengine   ║ ║                      ║
║ Service 侧:         ║ ║   ├── dashscope    ║ ║ parser / validator / ║
║ • prometheus        ║ ║   └── openai_compat║ ║ resolver / env_utils ║
║                    ║ ║                    ║ ║                      ║
║ 钩子点:             ║ ║ ModelProvider.invoke║ ║ Policy / Endpoint /  ║
║ before_request      ║ ║  ├─ _invoke_unified║ ║ LogicalModel 数据模型  ║
║ after_response      ║ ║  ├─ _invoke_passthru║ ║                      ║
║ on_error            ║ ║  └─ _invoke_adapted║ ║                      ║
╚════════════════════╝ ╚═════════╤══════════╝ ╚══════════════════════╝
                                 │
                                 ▼
                       ╔════════════════════╗
                       ║  ⑦ Adapter 跨风格   ║
                       ║  core/adapters/    ║
                       ║                    ║
                       ║ registry.py        ║
                       ║ openai_to_anthropic║
                       ║ openai_to_genai    ║
                       ║ openai_to_bedrock  ║
                       ║ anthropic_to_openai║
                       ║ genai_to_openai    ║
                       ║                    ║
                       ║ 双向参数/响应转换    ║
                       ╚════════════════════╝
```

---

## 三、Passthrough 三种调用模式

`core/models.py` 中 `PassthroughMode` 决定 `ModelProvider.invoke` 走哪条分派分支：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ModelProvider.invoke(request)                     │
│                                                                      │
│   request.is_passthrough?                                            │
│           │                                                          │
│     ┌─────┴─────┐                                                    │
│     │ False     │ True                                               │
│     ▼           ▼                                                    │
│ ┌────────┐  request.source_style == self.api_style?                  │
│ │DISABLED│    │                                                      │
│ │(统一)   │  ┌─┴─┐                                                    │
│ └───┬────┘ Yes  No                                                  │
│     │       │    │                                                   │
│     │       ▼    ▼                                                   │
│     │  ┌────────┐  ┌────────────┐                                    │
│     │  │SAME_   │  │CROSS_STYLE │                                    │
│     │  │STYLE   │  │            │                                    │
│     │  └───┬────┘  └─────┬──────┘                                    │
│     │      │             │                                           │
│     ▼      ▼             ▼                                           │
│ ┌──────────┐ ┌─────────┐ ┌───────────────────────────┐               │
│ │_invoke_  │ │_invoke_ │ │_invoke_adapted            │               │
│ │unified   │ │passthru │ │                           │               │
│ │          │ │         │ │ Adapter.to_provider() ──▶ │               │
│ │ Model    │ │ raw_    │ │   SDK 调用 ──▶             │               │
│ │Request ➜ │ │request  │ │ Adapter.from_provider()   │               │
│ │ SDK 参数 │ │ 直接透传  │ │                           │               │
│ │ 2 次转换 │ │ 0 次转换 │ │ 2 次转换 (跨风格)          │               │
│ └──────────┘ └─────────┘ └───────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘

ModelRequest 透传字段:
    passthrough_mode   enum DISABLED/SAME_STYLE/CROSS_STYLE
    source_style       "openai" / "anthropic" / "genai"
    raw_request        dict，用户原始参数
    raw_method         "chat.completions.create" 等

ModelResponse 透传字段:
    raw_response       原生响应对象
    raw_error          原生异常对象
```

---

## 四、容错机制：两层协同

```
                   ┌─────────────────────────────┐
                   │    请求进入 CoreEngine        │
                   └──────────────┬──────────────┘
                                  │
                                  ▼
 ┌────────────────────────── 层 2：健康管理（事前） ──────────────────┐
 │                                                                    │
 │  ① Endpoint 过滤（Plugin.EndpointFilterProvider）                   │
 │     ├─ Rate Limiter  is_limited? / 令牌桶有令牌?                    │
 │     └─ Circuit Breaker state == OPEN? → 排除                       │
 │                                                                    │
 │  ② 权重调整（Plugin.WeightAdjustmentProvider）                      │
 │     └─ Adaptive Weight 按滑窗 429 错误率渐进降权                     │
 │                                                                    │
 │  ③ Router 按策略选 endpoint                                         │
 └──────────────────────────────┬────────────────────────────────────┘
                                ▼
 ┌──────────────────────────── 发送请求 ─────────────────────────────┐
 │  Provider.invoke(request, decision)                               │
 └──────────────────────────────┬────────────────────────────────────┘
                                │
                   ┌────────────┴────────────┐
                   │                         │
                   ▼                         ▼
                 成功                       失败
                   │                         │
                   │       ┌─── 层 1：单次救援（事后）────────────┐
                   │       │                                   │
                   │       │  ① Retry 同 endpoint 退避重试       │
                   │       │     （429：Retry-After / 指数退避） │
                   │       │                                   │
                   │       │  ② Failover 换 endpoint            │
                   │       │     tried_endpoints 排除已用       │
                   │       │                                   │
                   │       │  ③ Fallback 换 logical_model       │
                   │       │     NoAvailableEndpointError 触发  │
                   │       │     Chain / List 两种模式          │
                   │       └──────────────┬────────────────────┘
                   │                      │
                   └──────┬───────────────┘
                          ▼
          Plugin.InvocationLifecycleHook.after_response
                          │
           ┌──────────────┴───────────────┐
           │  回写层 2 统计（滑动窗口）      │
           │  - CircuitBreaker 成败计数    │
           │  - AdaptiveWeight 429 频率   │
           │  - RateLimiter 消耗/补充令牌  │
           └──────────────────────────────┘
```

**三种健康信号的分工：**

| 机制              | 应对错误  | 响应方式    | 粒度     |
| --------------- | ----- | ------- | ------ |
| Rate Limiter    | 429   | 标记不可用/控速 | 二值 / 控频 |
| Adaptive Weight | 429 频发 | 渐进降权    | 连续     |
| Circuit Breaker | 非 429 (5xx) | 完全熔断    | 二值     |

---

## 五、配置系统与数据模型

```
┌──────────────────────────── ConfigStore ────────────────────────────┐
│                                                                     │
│  FileConfigStore        AppConfigConfigStore     HttpConfigStore    │
│  (YAML / JSON)          (AWS AppConfig)          (model-hub-api)    │
│         │                      │                        │           │
│         └──────────────────────┼────────────────────────┘           │
│                                ▼                                    │
│                  parser → validator → resolver                      │
│                                │                                    │
│                                ▼                                    │
│         ┌────────────────────────────────────────────┐              │
│         │             RuntimeConfig                  │              │
│         │                                            │              │
│         │  models:  { "summary:gpt-4.1": LogicalModel│              │
│         │                ├── endpoints: [Endpoint]   │              │
│         │                │    ├─ provider            │              │
│         │                │    ├─ model               │              │
│         │                │    ├─ weight / priority   │              │
│         │                │    └─ credentials         │              │
│         │                └── policy: Policy          │              │
│         │                     ├─ routing_strategy   │              │
│         │                     ├─ fallback_model     │              │
│         │                     └─ timeout/retry       │              │
│         │  credentials: { ... }                      │              │
│         └────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、关键文件速查

| 模块             | 路径                                                              |
| -------------- | --------------------------------------------------------------- |
| CoreEngine     | `packages/core/src/model_hub_core/engine.py`                    |
| Router         | `packages/core/src/model_hub_core/router.py`                    |
| 数据模型           | `packages/core/src/model_hub_core/models.py`                    |
| Provider 基类    | `packages/core/src/model_hub_core/providers/base.py`            |
| Provider 注册表   | `packages/core/src/model_hub_core/providers/registry.py`        |
| Adapter 注册表    | `packages/core/src/model_hub_core/adapters/registry.py`         |
| Plugin 基类      | `packages/core/src/model_hub_core/plugins/base.py`              |
| RateLimiter    | `packages/core/src/model_hub_core/plugins/rate_limit.py`        |
| CircuitBreaker | `packages/core/src/model_hub_core/plugins/circuit_breaker.py`   |
| AdaptiveWeight | `packages/core/src/model_hub_core/plugins/adaptive_weight.py`   |
| ConfigStore    | `packages/core/src/model_hub_core/config/store.py`              |
| ModelHubClient | `packages/sdk-python/src/model_hub_sdk/client/hub_client.py`    |
| OpenAI Wrapper | `packages/sdk-python/src/model_hub_sdk/wrappers/openai/`        |
| Transport      | `packages/sdk-python/src/model_hub_sdk/transport/`              |
| LangChain 集成   | `packages/sdk-python/src/model_hub_sdk/integrations/langchain/` |
| Langfuse 插件   | `packages/sdk-python/src/model_hub_sdk/plugins/langfuse.py`     |
| OTel 插件       | `packages/sdk-python/src/model_hub_sdk/plugins/otel.py`         |
| FastAPI 服务    | `services/model-hub-api/src/model_hub_api/main.py`              |
| Prometheus 插件 | `services/model-hub-api/src/model_hub_api/plugins/prometheus.py` |

---

## 七、一张图看完整调用路径

```
业务代码
  │  client.chat.completions.create(model="gpt-4.1", messages=[...])
  ▼
┌─────────────────────────── SDK Wrapper / Client ───────────────────────────┐
│ wrap_openai()  →  拦截原生方法                                               │
│               →  打包 ModelRequest(raw_request=…, source_style="openai")   │
└─────────────────────────────────┬──────────────────────────────────────────┘
                                  ▼
┌──────────────────────────────── Transport ─────────────────────────────────┐
│ DirectProviderTransport.invoke(request, context)                           │
└─────────────────────────────────┬──────────────────────────────────────────┘
                                  ▼
┌──────────────────────────────── CoreEngine ────────────────────────────────┐
│  1. 解析 RuntimeConfig → 找到 LogicalModel & Policy                         │
│  2. Plugin 链: EndpointFilter → 排除 OPEN/Limited endpoint                  │
│  3. Plugin 链: WeightAdjustment → Adaptive 调整权重                         │
│  4. Router.pick_endpoint(strategy, tried_endpoints)                         │
│  5. Plugin.before_request                                                   │
│  6. ProviderRegistry.get(endpoint.provider).invoke(request, decision)       │
│        ├─ DISABLED     → _invoke_unified   (Model→SDK)                      │
│        ├─ SAME_STYLE   → _invoke_passthrough (零转换)                        │
│        └─ CROSS_STYLE  → _invoke_adapted (经 Adapter)                       │
│  7. 成功 → Plugin.after_response → 统计回写 → 返回                            │
│     失败 → Retry → Failover → Plugin.on_error                              │
│           └─ 所有 endpoint 耗尽 → Fallback 换 logical_model → 回到 1         │
└─────────────────────────────────┬──────────────────────────────────────────┘
                                  ▼
            原生 SDK 响应 (OpenAI / Anthropic / GenAI / …)
                                  │
                                  ▼
                           Wrapper 拆包
                                  │
                                  ▼
                      业务代码拿到 ChatCompletion 对象
```

---

## 相关文档

- [[PlaudModelHub完全指南]]：逐章讲解每个模块的设计与代码位置
- [[PlaudModelHub思维导图]]：知识点大纲
- [[自适应权重插件设计方案]]：Adaptive Weight 插件的设计详情
