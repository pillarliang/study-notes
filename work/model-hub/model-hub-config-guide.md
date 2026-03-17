# Model Hub SDK 配置完全指南

> 这是一份关于 Model Hub SDK 配置的完整参考文档，包括插件配置、限流器、熔断器和多 endpoint 路由机制。

**最后更新**：2026-03-05

---

## 目录

1. [插件配置详解](#插件配置详解)
2. [限流器（Rate Limiter）](#限流器rate-limiter)
3. [熔断器（Circuit Breaker）](#熔断器circuit-breaker)
4. [多 Endpoint 配置](#多-endpoint-配置)
5. [降级机制（Fallback）](#降级机制fallback)
6. [配置最佳实践](#配置最佳实践)

---

## 插件配置详解

### 四个核心插件

Model Hub SDK 支持四个可选插件，用于可观测性、监控和容错：

```yaml
plugins:
  langfuse:       # LLM 调用追踪
  otel:           # 分布式追踪（云原生）
  circuit_breaker: # 熔断器（故障隔离）
  rate_limit:      # 限流器（主动/被动限流）
```

---

## 插件 1：Langfuse（可观测性）

**用途**：LLM 调用的追踪、分析和调试

```yaml
plugins:
  langfuse:
    enabled: true                                # 是否启用
    public_key: ${LANGFUSE_PUBLIC_KEY:-}        # 公开密钥
    secret_key: ${LANGFUSE_SECRET_KEY:-}        # 秘密密钥
    host: ${LANGFUSE_HOST:-https://cloud.langfuse.com}  # 服务地址
```

**字段说明**：

| 字段 | 说明 | 例子 |
|------|------|------|
| **enabled** | 是否启用 | true / false |
| **public_key** | 认证凭证（公钥） | pk_xxx |
| **secret_key** | 认证凭证（秘钥） | sk_xxx |
| **host** | Langfuse 服务地址 | https://cloud.langfuse.com（官方）<br/>https://自建.com（自建实例） |

**环境变量支持**：
- `${LANGFUSE_PUBLIC_KEY:-}` 从环境变量读取，为空时用默认值
- `${LANGFUSE_HOST:-https://cloud.langfuse.com}` 未设置时用默认官方地址

**推荐配置**：
- **开发环境**：启用 Langfuse + 禁用 OTEL
- **生产环境**：仅用 OTEL（避免与 Langfuse 重复记录）

---

## 插件 2：OpenTelemetry（OTEL）

**用途**：分布式追踪和遥测（云原生标准）

```yaml
plugins:
  otel:
    enabled: false                    # 默认禁用，避免与 Langfuse 重复
    service_name: model-hub-sdk       # 服务名称，用于识别追踪来源
```

**字段说明**：

| 字段 | 说明 | 例子 |
|------|------|------|
| **enabled** | 是否启用 | true / false |
| **service_name** | 服务名，用于区分追踪来源 | my-app-name |

**注意**：
- 如果同时启用 Langfuse 和 OTEL，会导致追踪记录重复
- Langfuse v3+ 基于 OpenTelemetry 实现，建议二选一
- 生产推荐用 OTEL，开发推荐用 Langfuse

---

## 插件 3：Langfuse vs OTEL 对比

| 对比项 | Langfuse | OTEL |
|--------|----------|------|
| **用途** | LLM 专用追踪平台 | 云原生分布式追踪 |
| **学习曲线** | 低（针对 LLM） | 中等（通用）|
| **与其他服务集成** | 有限 | 优秀（Jaeger、Datadog 等）|
| **开发体验** | 优秀（LLM 友好UI） | 需额外工具可视化 |
| **生产推荐** | 可选 | 推荐 |

---

## 限流器（Rate Limiter）

限流器是 **两层防护** 机制，保护客户端和下游服务。

### 工作原理

```
第 1 层：被动限流（响应 429）
  └─ 服务器返回 429 → 解析 Retry-After → 标记 endpoint 不可用

第 2 层：主动限流（令牌桶）
  └─ 发送请求前自检 → 无令牌则拒绝 → 避免触发 429
```

### 配置详解

```yaml
plugins:
  rate_limit:
    enabled: true                          # 是否启用限流器

    # === 主动限流（令牌桶算法）===
    enable_token_bucket: true              # 是否启用令牌桶
    bucket_capacity: 10                    # 桶容量（最大令牌数）
    refill_rate: 1.0                       # 令牌补充速率（每秒）

    # === 被动限流（429 响应处理）===
    respect_retry_after: true              # 是否遵守服务端 Retry-After 头
    max_retry_after_seconds: 300.0         # 最大接受的等待时间（秒）
    default_retry_after_seconds: 60.0      # 无 Retry-After 时的默认等待

    # === 与熔断器的交互 ===
    exclude_429_from_circuit_breaker: true # 429 是否排除在熔断统计之外
```

### 字段详解

#### 令牌桶配置

```yaml
enable_token_bucket: true       # 启用/禁用令牌桶主动限流
bucket_capacity: 10             # 桶的最大容量
refill_rate: 1.0                # 每秒补充的令牌数
```

**工作流程**：

```
初始状态（bucket_capacity=10, refill_rate=1.0）:
  第 0 秒：[●●●●●●●●●●] 10 个令牌

请求消耗:
  第 0 秒：请求来 → -1 令牌 → [●●●●●●●●●] 9 个令牌
  第 1 秒：补充 1 个 → [●●●●●●●●●●] 10 个令牌

突发流量:
  第 0 秒：10 个请求 → 令牌用尽 [　　　　　] 0 个令牌
  第 1 秒：补充 1 个 → [●] 1 个令牌
  第 2 秒：补充 1 个 → [●●] 2 个令牌
  ...
  第 10 秒：恢复满负荷 [●●●●●●●●●●]
```

**调优建议**：
- 低并发：`capacity=10, refill_rate=1.0`（每秒 1 个请求）
- 中等并发：`capacity=50, refill_rate=5.0`（每秒 5 个请求）
- 高并发：`capacity=100, refill_rate=10.0`（每秒 10 个请求）

#### 被动限流配置

```yaml
respect_retry_after: true              # 遵守服务端的 Retry-After 头
max_retry_after_seconds: 300.0         # 即使服务器要求等 10 分钟，最多等 300 秒
default_retry_after_seconds: 60.0      # 服务器没有 Retry-After 时，默认等待 60 秒
```

**流程**：
```
收到 429 响应
  ↓
解析响应头的 Retry-After 字段
  ├─ 如果有 Retry-After
  │  └─ 取 min(Retry-After, max_retry_after_seconds)
  │
  └─ 如果无 Retry-After
     └─ 用 default_retry_after_seconds
  ↓
计算限流截止时间 = now + retry_after
  ↓
标记 endpoint 为不可用，直到限流时间过期
```

#### 与熔断器的交互

```yaml
exclude_429_from_circuit_breaker: true  # 429 不计入熔断统计
```

**为什么要排除 429**？

```
假设 endpoint 在 60 秒内连续 429

❌ 如果不排除：
   100% 失败率 > 50% 阈值
   ↓
   触发熔断 OPEN 状态（额外 30 秒）
   ↓
   总不可用时间 = 60 + 30 = 90 秒（太严重）

✅ 如果排除（推荐）：
   429 不计入熔断统计
   ↓
   只限流 60 秒
   ↓
   时间过期后自动恢复（无双重惩罚）
```

### 429 过多时的行为

```
时间    事件                              状态
──────────────────────────────────────────────────────────
0s     发送请求 → 成功                  endpoint: ✓ 正常

10s    频繁请求 → 服务器返回 429       限流器记录：
       Retry-After: 120                is_limited=True
                                       limited_until=130s

10-130s 任何新请求都被本地拒绝          endpoint: ✗ 不可用
       （不会真的发送到服务器）        reason: 被动限流

130s   限流时间过期                      endpoint: ✓ 恢复

⚠️  关键点：限流期间的请求被本地拒绝，减少了实际发送的 429 数量
```

### 多 Endpoint 时的限流行为

```yaml
models:
  ai-demo:gpt-4o:
    endpoints:
      - provider: openai      # Endpoint A
        model: gpt-4o
        weight: 60
      - provider: azure       # Endpoint B
        model: gpt-4o
        weight: 40
```

**场景**：OpenAI 配额耗尽

```
时间线：
0-60s   OpenAI 正常 → 60% 流量
        Azure 正常  → 40% 流量

60s     OpenAI 返回 429
        Endpoint A 被标记不可用

60-120s 流量自动转移：
        Endpoint B 承载 100% 流量
        用户无感知（自动容灾）

120s    限流时间过期
        恢复 60:40 分配
```

---

## 熔断器（Circuit Breaker）

熔断器是故障隔离机制，防止故障传播和雪崩效应。

### 状态机

```
CLOSED（正常）
  ↓ [失败率 > threshold 且 calls ≥ min_calls]
OPEN（熔断中）⏱ [等待 open_duration_seconds]
  ↓
HALF_OPEN（探测中）
  ├─ 探测请求成功 → CLOSED（恢复正常）
  └─ 探测请求失败 → OPEN（继续熔断）
```

### 配置

```yaml
plugins:
  circuit_breaker:
    enabled: true                       # 是否启用熔断器
    failure_threshold: 0.5              # 失败率阈值（50%）
    min_calls: 5                        # 最小调用次数
    window_size_seconds: 60             # 滑动时间窗口（秒）
    open_duration_seconds: 30           # 熔断持续时间（秒）
    half_open_max_calls: 3              # 半开状态最多试探请求数
```

### 字段详解

| 字段 | 默认值 | 说明 | 例子 |
|------|--------|------|------|
| **failure_threshold** | 0.5 | 失败率阈值 | 0.5 = 50% 失败率触发 |
| **min_calls** | 5 | 最小调用次数 | 至少 5 次调用才判断 |
| **window_size_seconds** | 60 | 时间窗口大小 | 统计过去 60 秒的请求 |
| **open_duration_seconds** | 30 | 熔断持续时间 | 熔断后冷静 30 秒 |
| **half_open_max_calls** | 3 | 半开状态探测次数 | 最多试 3 次请求 |

### 工作场景

```
假设：min_calls=5, failure_threshold=0.5, window_size_seconds=60

时间轴：
0-10s   请求 1-5 全部失败（5/5 = 100% 失败率 > 50%）
        ✓ 达到 min_calls=5
        ✓ 失败率 > 阈值
        → 触发熔断，进入 OPEN 状态

10-40s  OPEN 熔断状态（30 秒）
        所有请求立即拒绝，不调用服务
        ✓ 保护了 API 配额，减少不必要的 API 调用

40s     熔断时间过期
        进入 HALF_OPEN 状态
        允许最多 3 个探测请求通过

42s     3 个探测请求都成功
        ✓ 服务已恢复
        → 返回 CLOSED 状态（正常）

42+     接收正常流量
```

### 错误分类

```
可重试错误（参与熔断）：
  - 408: Request Timeout
  - 409: Conflict
  - 429: Rate Limit ← 但通常被 exclude_429_from_circuit_breaker 排除
  - 5xx: Server Error

不可重试错误（不参与熔断）：
  - 400: Bad Request
  - 401: Unauthorized
  - 403: Forbidden
  - 404: Not Found
```

---

## 多 Endpoint 配置

一个 Model 可以有多个物理 endpoint，用于负载均衡、灾备、成本优化等。

### 三种配置格式

#### 1️⃣ 字符串格式（最简洁）

```yaml
models:
  # 格式: "provider_preset/physical_model[@version]"
  project-summary:gemini-2.5-pro: vertex/gemini-2.5-pro
  project-summary:qwen3-max: qwen/qwen3-max
```

**特点**：
- 最简洁
- 单 endpoint
- 使用 provider 默认配置

---

#### 2️⃣ 简化对象格式（单 endpoint，自定义参数）

```yaml
models:
  project-summary:gpt-4o-custom:
    provider: litellm
    model: gpt-4o-2024-11-20
    weight: 80                # 权重（默认 100）
    enabled: true             # 是否启用
    timeout_ms: 60000         # 自定义超时 60 秒
    max_retries: 3            # 最大重试 3 次
    priority: 10              # 优先级（用于 PRIORITY 路由）
```

**特点**：
- 灵活控制参数
- 单 endpoint
- 可覆盖默认配置

---

#### 3️⃣ 完整格式（多 endpoint）⭐

```yaml
models:
  project-summary:gemini:
    endpoints:
      - provider: vertex
        model: gemini-2.5-pro
        weight: 30              # 权重：30%
        enabled: true
        timeout_ms: 30000
        priority: 10

      - provider: vertex
        model: gemini-3.1-pro-preview
        weight: 50              # 权重：50%
        enabled: true
        timeout_ms: 30000
        priority: 20

      - provider: vertex
        model: gemini-3-flash-preview
        weight: 20              # 权重：20%
        enabled: true
        timeout_ms: 20000
        priority: 5
```

**特点**：
- 多 endpoint
- 按权重分配流量
- 自动故障转移

### Endpoint 完整字段

```yaml
endpoints:
  - id: ep-primary                # [可选] Endpoint ID，不指定则自动生成
    provider: azure-east          # [必填] Provider 预设名或类型
    model: gpt-4o                 # [必填] 物理模型名
    base_url: https://...         # [可选] 基础 URL（覆盖预设）
    credential_ref: cred-azure    # [可选] 凭证引用
    weight: 60                     # [可选] 路由权重，默认 100
    enabled: true                 # [可选] 是否启用，默认 true
    priority: 10                   # [可选] 优先级（PRIORITY 策略用）
    region: us-east               # [可选] 区域标识
    timeout_ms: 30000             # [可选] 超时（毫秒）
    max_retries: 2                # [可选] 最大重试次数
    request_types:                # [可选] 支持的请求类型
      - chat                       #   chat: 聊天补全
      - embedding                  #   embedding: 向量嵌入
    extra: {}                     # [可选] 厂商特定配置
```

### 多 Endpoint 应用场景

#### 场景 1：负载均衡（不同版本）

```yaml
models:
  ai-demo:gpt-4o-balanced:
    endpoints:
      - provider: openai
        model: gpt-4o-2024-11-20  # 新版本
        weight: 60                # 60% 流量（验证）

      - provider: openai
        model: gpt-4o-mini        # 旧版本
        weight: 40                # 40% 流量（成本优化）
```

**效果**：灰度发布，既验证新版本，又降低成本

---

#### 场景 2：多区域容灾

```yaml
models:
  project-summary:gemini-global:
    endpoints:
      - provider: vertex-us       # 美国区（低延迟）
        model: gemini-2.5-pro
        weight: 50
        region: us
        priority: 10

      - provider: vertex-eu       # 欧盟区（备选）
        model: gemini-2.5-pro
        weight: 30
        region: eu
        priority: 5

      - provider: openrouter      # 公网 fallback
        model: "google/gemini-2.0-flash"
        weight: 20
        region: global
        priority: 1
```

**效果**：
- 优先用美国区（低延迟）
- 美国区故障 → 自动转移到欧盟区
- 都故障 → 使用 OpenRouter

---

#### 场景 3：成本优化（混合厂商）

```yaml
models:
  project-summary:summary-cheap:
    endpoints:
      - provider: openrouter      # 免费或便宜
        model: "tngtech/deepseek-r1t2-chimera:free"
        weight: 50

      - provider: openrouter
        model: "kwaipilot/kat-coder-pro:free"
        weight: 30

      - provider: litellm         # 付费 fallback
        model: gpt-4o-mini
        weight: 20
```

**效果**：优先用免费模型，配额用完再用付费的

---

#### 场景 4：多版本灰度

```yaml
models:
  ai-demo:gpt-4o-canary:
    endpoints:
      - provider: openai
        model: gpt-4o-2024-12-20  # 最新版本（金丝雀）
        weight: 10                # 10% 流量

      - provider: openai
        model: gpt-4o-2024-11-20  # 稳定版本
        weight: 90                # 90% 流量
```

**效果**：小比例流量用新版本，大比例用稳定版本

---

### 路由策略

#### WEIGHTED_RANDOM（默认）

```yaml
policies:
  default:
    strategy: WEIGHTED_RANDOM
    enable_session_affinity: true  # ← 会话粘性
```

**行为**：

```
第一个请求（session_id=abc）
  → 按权重随机选择
  → 选中 endpoint-1

第二个请求（session_id=abc）
  → 粘性路由（会话亲和性）
  → 仍用 endpoint-1（避免模型差异）

第三个请求（session_id=xyz）
  → 新 session
  → 重新按权重选择
```

**优点**：
- 会话一致性（同用户始终用同一模型）
- 降低模型间的行为差异
- 适合多轮对话场景

---

#### ROUND_ROBIN（轮询）

```yaml
policies:
  ai-demo:load-balanced:
    strategy: ROUND_ROBIN
    enable_session_affinity: false
```

**行为**：

```
第 1 个请求 → endpoint-1
第 2 个请求 → endpoint-2
第 3 个请求 → endpoint-3
第 4 个请求 → endpoint-1（循环）
```

**优点**：
- 均匀分布请求
- 不考虑权重
- 适合无状态场景

---

#### PRIORITY（优先级）

```yaml
models:
  ai-demo:priority-example:
    endpoints:
      - provider: azure-east
        model: gpt-4o
        priority: 10  # 最高优先级

      - provider: openai
        model: gpt-4o
        priority: 5   # 中等优先级

      - provider: litellm
        model: gpt-4o-mini
        priority: 1   # 最低优先级

policies:
  ai-demo:priority-example:
    strategy: PRIORITY
```

**行为**：

```
请求 → priority=10 的 endpoint（Azure）
       ↓ 故障（熔断/限流）
       → priority=5 的 endpoint（OpenAI）
       ↓ 故障
       → priority=1 的 endpoint（LiteLLM）
```

**优点**：
- 清晰的故障转移优先级
- 适合成本敏感的场景（优先用贵的，便宜的当备选）

---

## 降级机制（Fallback）

### Fallback 的触发条件

Fallback **不是** 因为 429 触发的，而是因为所有 endpoint 都不可用时触发。

```
发送请求到 Model A
       ↓
检查 endpoint 可用性（限流器、熔断器）
       ↓
❌ 如果 Model A 还有其他可用 endpoint
   └─ 自动转移到其他 endpoint（failover）
   └─ ❌ 不触发 fallback

❌ 如果 Model A 的所有 endpoint 都不可用
   └─ 抛出 NoAvailableEndpointError
   └─ ✅ 触发 fallback
   └─ 自动降级到 fallback_model
```

### 配置 Fallback

```yaml
policies:
  # 降级链：Pro → Flash
  project-summary:gemini-3.1-pro-preview:
    fallback_model: "project-summary:gemini-2.5-pro"

  project-summary:gemini-2.5-pro:
    fallback_model: "project-summary:gemini-3-flash-preview"

  project-summary:gemini-3-flash-preview:
    fallback_model: "project-summary:gemini-3.1-flash-lite-preview"

  # CN 模型 fallback 链
  project-summary:qwen3-max:
    fallback_model: "project-summary:qwen-plus"
```

**格式**：
```
fallback_model: "app_id:logical_model"
```

- 完整格式：`"other-app:gpt-3.5"` → 用指定 app_id
- 简化格式：`"gpt-3.5"` → 用原始请求的 app_id

### Fallback 工作流程

```
时间    事件                                状态
────────────────────────────────────────────────────────
0s     请求 Model A
       ├─ endpoint-1: 被限流（429）
       ├─ endpoint-2: 被限流（429）
       └─ endpoint-3: 被限流（429）

       所有 endpoint 都不可用
       → 抛出 NoAvailableEndpointError

1s     捕获异常，检查 fallback_model
       fallback_model: "project:Model B"

       创建新请求，切换到 Model B

2s     请求 Model B
       ├─ endpoint-1: ✓ 可用
       └─ 发送请求成功

3s     返回响应给用户
       （虽然用的是 Model B，但请求成功）
```

### Fallback 保护机制

```yaml
# 防止无限循环
max_fallback_depth: 3  # 最多 fallback 3 次

# 防止循环引用
# 如果检测到循环（A → B → A），会直接报错
```

**例子**：

```
❌ 循环配置（会报错）：
  Model A → fallback to Model B
  Model B → fallback to Model A

✅ 正确的链式配置：
  Model A → Model B → Model C → Model D
```

### 多 Endpoint + Fallback 的交互

**场景：单个模型有多个 endpoint + fallback**

```
Model A 有 3 个 endpoint：
  - endpoint-1: Azure（被限流）
  - endpoint-2: OpenAI（被限流）
  - endpoint-3: OpenRouter（被限流）

请求流程：
1. 尝试 endpoint-1 → 429（不可用）
2. 尝试 endpoint-2 → 429（不可用）
3. 尝试 endpoint-3 → 429（不可用）
4. 所有 endpoint 都不可用
5. ✅ 触发 fallback → Model B

关键：只有当 Model A 的所有 endpoint 都失败，才会 fallback
```

---

## 配置最佳实践

### 1. 环境隔离

```yaml
# 开发环境
env: dev
plugins:
  langfuse:
    enabled: true    # 便于本地调试
  otel:
    enabled: false   # 避免干扰
  circuit_breaker:
    enabled: true    # 保护下游
  rate_limit:
    enabled: true    # 保护下游

# 生产环境
env: prod
plugins:
  langfuse:
    enabled: false   # 用 OTEL 避免重复
  otel:
    enabled: true    # 云原生监控
  circuit_breaker:
    enabled: true    # 必须启用
  rate_limit:
    enabled: true    # 必须启用
```

---

### 2. 限流和熔断配置

#### 低并发场景

```yaml
rate_limit:
  bucket_capacity: 5
  refill_rate: 0.5       # 每 2 秒 1 个请求

circuit_breaker:
  failure_threshold: 0.5
  min_calls: 3           # 只需 3 次失败
  window_size_seconds: 30
  open_duration_seconds: 15
```

#### 高并发场景

```yaml
rate_limit:
  bucket_capacity: 100
  refill_rate: 20.0      # 每秒 20 个请求

circuit_breaker:
  failure_threshold: 0.3  # 更敏感（30% 就触发）
  min_calls: 20           # 需要 20 次失败确认
  window_size_seconds: 60
  open_duration_seconds: 60
```

---

### 3. 多 Endpoint 权重分配

#### 成本优先

```yaml
endpoints:
  - provider: cheap       # 免费或便宜
    weight: 70
  - provider: expensive   # 付费
    weight: 30           # fallback 用
```

#### 质量优先

```yaml
endpoints:
  - provider: best        # 最好的模型
    weight: 50
  - provider: good        # 次好的
    weight: 40
  - provider: decent      # 可接受
    weight: 10
```

#### 均衡策略

```yaml
endpoints:
  - provider: provider-1  # 平衡成本和质量
    weight: 40
  - provider: provider-2
    weight: 35
  - provider: provider-3
    weight: 25
```

---

### 4. 错误恢复时间配置

```yaml
# 快速恢复（适合暂时故障）
circuit_breaker:
  open_duration_seconds: 15  # 15 秒后重试

rate_limit:
  default_retry_after_seconds: 30  # 30 秒后重试

# 保守恢复（适合持久故障）
circuit_breaker:
  open_duration_seconds: 60  # 60 秒后重试

rate_limit:
  default_retry_after_seconds: 120  # 120 秒后重试
```

---

### 5. 完整配置模板

```yaml
config_schema: v2
version: 1
env: prod

providers:
  primary:
    provider: openai
    base_url: https://api.openai.com/v1
    api_key: ${OPENAI_API_KEY:-}

  backup:
    provider: anthropic
    api_key: ${ANTHROPIC_API_KEY:-}

models:
  # 多 endpoint，故障自动转移
  app-name:gpt-4o:
    endpoints:
      - provider: primary
        model: gpt-4o
        weight: 70
        timeout_ms: 30000

      - provider: backup
        model: claude-opus-4-5
        weight: 30
        timeout_ms: 40000

policies:
  default:
    strategy: WEIGHTED_RANDOM
    enable_session_affinity: true

  # Fallback 链
  app-name:gpt-4o:
    fallback_model: "app-name:gpt-4o-mini"

  app-name:gpt-4o-mini:
    fallback_model: "app-name:backup-model"

plugins:
  langfuse:
    enabled: true
    public_key: ${LANGFUSE_PUBLIC_KEY:-}
    secret_key: ${LANGFUSE_SECRET_KEY:-}

  circuit_breaker:
    enabled: true
    failure_threshold: 0.5
    min_calls: 5
    window_size_seconds: 60
    open_duration_seconds: 30

  rate_limit:
    enabled: true
    enable_token_bucket: true
    bucket_capacity: 20
    refill_rate: 2.0
    respect_retry_after: true
    exclude_429_from_circuit_breaker: true
```

---

## 常见问题

### Q1: 429 会导致 Fallback 吗？

**A**: 不完全是。429 会标记 endpoint 为不可用，但如果模型还有其他可用 endpoint，会自动转移（failover），不会 fallback。只有当所有 endpoint 都不可用时，才会触发 fallback 到另一个模型。

---

### Q2: 限流器和熔断器应该都启用吗？

**A**: 是的。它们各司其职：
- **限流器**：保护客户端和下游服务
- **熔断器**：防止故障传播

一起启用效果最好。

---

### Q3: 同时启用 Langfuse 和 OTEL 会怎样？

**A**: 会导致追踪记录重复。建议：
- 开发：Langfuse + 禁用 OTEL
- 生产：OTEL + 禁用 Langfuse

---

### Q4: 如何快速恢复故障 Endpoint？

**A**: 减小 `open_duration_seconds` 和 `default_retry_after_seconds`。但要谨慎，太快恢复可能导致重复故障。

---

### Q5: 权重 weight 和优先级 priority 的区别？

**A**:
- **weight**：用于 WEIGHTED_RANDOM 和 ROUND_ROBIN 策略，控制流量分配比例
- **priority**：用于 PRIORITY 策略，控制故障转移的优先级顺序

可以同时配置，priority 用于故障转移，weight 用于正常流量分配。

---

## 快速查询表

### 插件启用/禁用矩阵

| 环境 | Langfuse | OTEL | 熔断器 | 限流器 |
|------|----------|------|--------|--------|
| 本地开发 | ✅ | ❌ | ✅ | ✅ |
| 测试环境 | ✅ | ❌ | ✅ | ✅ |
| 生产环境 | ❌* | ✅ | ✅ | ✅ |

*如果需要 Langfuse，禁用 OTEL

---

### 429 处理流程

```
收到 429
  ↓
解析 Retry-After
  ↓
标记 endpoint 不可用（限流中）
  ↓
检查是否还有其他可用 endpoint
  ├─ 有 → failover 到其他 endpoint（不 fallback）
  └─ 无 → fallback 到 fallback_model
```

---

### 多 Endpoint 自动转移流程

```
Model A 有多个 endpoint，其中一个 429
  ↓
endpoint 被标记不可用
  ↓
路由器排除该 endpoint
  ↓
选择其他可用 endpoint
  ↓
请求成功转移（用户无感知）
```

---

## 参考资源

- [Model Hub SDK 配置完整示例](./examples/config-complete.yaml)
- [限流器源代码](../packages/core/src/model_hub_core/plugins/rate_limit.py)
- [熔断器源代码](../packages/core/src/model_hub_core/plugins/circuit_breaker.py)
- [路由和 Fallback 源代码](../packages/core/src/model_hub_core/engine.py)

---

**文档版本**：1.0
**最后更新**：2026-03-05
**贡献者**：Claude Code
