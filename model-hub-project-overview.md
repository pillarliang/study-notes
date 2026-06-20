# Plaud Model Hub — 项目介绍（面试版）

> **配套简历条目**：工作经历 → Plaud AI →「Plaud Model Hub — 统一 LLM 调用基础设施层（核心贡献者）」  
> **用途**：面试时介绍这个项目用。先看「一句话 + 电梯陈述」，再按需展开每个核心模块。

---

## 0. 一句话定位

**Model Hub 是 Plaud AI Platform 的统一大模型调用基础设施层**：通过 Logical Model / Endpoint / Provider 三层抽象屏蔽供应商差异，内部托管**路由、容错、可观测性**，让业务方只需 `app_id + logical_model` 两个名字即可调用任何 LLM。

我作为**核心贡献者**，独立负责其中三个关键模块：① 自适应权重插件（解决 429 连锁故障）② 跨风格路由适配（OpenAI ↔ Anthropic / Gemini 双向翻译）③ Fallback 增强（多级降级链 + 环路检测）+ LangChain 集成 + AppConfig 热更新。

---

## 1. 电梯陈述（30 秒 / 60 秒两版）

### 30 秒版

> Plaud 业务线同时调用多家 LLM（Gemini / GPT / Claude / 火山 / 阿里云），每家 API 格式不同、配额限流各异。Model Hub 把这些通用问题一次性下沉：业务方只写 `client.invoke("summary-gemini", ...)` 一行代码，路由、降权、重试、降级全自动。我主导的核心创新是**自适应权重插件**——按 429 频率渐进降权取代二值切流，让 GSU 预付费和 PayGo 按量付费平滑切换，节省 30% 成本的同时可用性提升 4.3 个百分点；还有**跨风格透传**——业务方用 OpenAI SDK 写的代码可以无缝 Fallback 到 Claude，0 次转换的同时保留完整容错能力。

### 60 秒版（加技术深度）

> 在 30 秒版基础上补充：Hub 架构的核心是"单一入口 + 三种调用模式"——统一模式走 Hub 抽象字段（2 次转换），同风格透传走 0 次转换（WrappedOpenAI → OpenAI endpoint），跨风格透传走 Adapter 翻译（WrappedOpenAI → Claude endpoint）。三种模式共享同一套基础设施（路由 / 熔断 / 限流 / Fallback）。自适应权重的关键设计有三个：① 权重重分配机制（降权部分按健康 endpoint 的原权重比例重新分配，让小权重 endpoint 在应急时也能承接流量）② recovery_rate 限速恢复（multiplier 下行立即生效、上行受速率限制，避免恢复时流量瞬涨再次 429）③ 下限保护（multiplier 最低 0.1，保留 10% 探测流量，配额恢复后自动感知）。Fallback 支持多级链（gemini → gpt → claude），环路检测防循环引用，元数据回写让降级路径可追溯。

---

## 2. 项目背景：要解决什么问题

Plaud AI 的录音总结、RAG 问答、项目摘要等业务线需要同时调用多家 LLM 供应商（OpenAI / Anthropic / Google / 火山 / 阿里云等）。**直接让业务代码管理这些调用会面临三类核心问题**：


| 问题类型       | 具体表现                                                                           | 不处理的后果                    |
| ---------- | ------------------------------------------------------------------------------ | ------------------------- |
| **异构 API** | 各家 SDK 参数格式不同（Anthropic 的 `max_tokens` 必填、Gemini 的 temperature 要包在 Config 对象里） | 业务代码重复写转换逻辑，维护成本高         |
| **容错复杂**   | 429 限流、5xx 故障、区域故障频发；单一 endpoint 配额有限                                          | 一个 endpoint 429 可能打穿整条业务链 |
| **成本失控**   | GSU 预付费便宜但容量有限，PayGo 按量贵但弹性                                                    | 全量走 PayGo 成本不可控，手动切流运维压力大 |


**Model Hub 的解决方案**：通过 **Logical Model / Endpoint / Provider 三层抽象** 屏蔽供应商差异，把路由、容错、可观测性下沉成基础设施。

```
业务代码（完全不变）
    ↓
client.invoke("summary-gemini", messages=[...])
    ↓
Model Hub（自动处理）
    ├─ 路由：按权重选 Endpoint（GSU 80% / PayGo 20%）
    ├─ 容错：429 → 降权 → Failover → Fallback
    └─ 可观测：Langfuse 追踪、OTel 分布式链路
    ↓
OpenAI / Anthropic / Gemini / 火山 / 阿里云 SDK
```

**业务收益**：

- **开发效率**：业务方只关心逻辑模型名（如 `summary-gpt-4.1`），底层路由、重试、降级 100% 自动化
- **成本优化**：配置驱动的流量分配（GSU 优先消耗预付费 → PayGo 兜底），生产环境节省约 30% LLM 成本
- **可用性提升**：三层容错（自适应权重 + 熔断器 + Fallback）确保单点故障不影响业务，可用性从 95% 提升到 99.5%

---

## 3. 我的角色

**核心贡献者，独立负责三个关键模块 + 两个增值功能**：


| 模块                | 角色                                            | 代码规模                           |
| ----------------- | --------------------------------------------- | ------------------------------ |
| **自适应权重插件**       | 从 0 到 1 设计实现                                  | 520 行，25 个单元测试                 |
| **跨风格路由适配**       | 独立设计并实现 OpenAI ↔ Anthropic / GenAI 双向 Adapter | 3 个 Adapter，18 个单元测试           |
| **Fallback 增强**   | 从单级改造为多级链 + 环路检测 + 元数据回写                      | 核心逻辑 100+ 行                    |
| **LangChain 集成**  | ChatModelHub 适配 + 三个参数管理 API                  | `BaseChatModel` 子类实现           |
| **AppConfig 热更新** | 抽象 ConfigStore 接口 + 灰度发布支持                    | 3 种实现（File / AppConfig / Http） |


除此之外，**全程参与** Hub 架构设计、Provider 基类设计、参数透传机制设计、插件系统设计、测试体系建设等。

---

## 4. 整体架构：三种调用模式（核心设计）

> [!important] 这是 Model Hub 区别于普通 LLM Gateway 的核心设计
> **单一入口 + 三种调用模式**：业务方可以用 Hub 统一字段、也可以用原生 SDK 参数，甚至可以跨厂商 Fallback，**所有模式共享同一套基础设施**（路由 / 重试 / 熔断 / Fallback）。

### 4.1 三种调用模式


| 模式        | 业务代码写法                                          | 转换次数         | 适用场景                 |
| --------- | ----------------------------------------------- | ------------ | -------------------- |
| **统一模式**  | `client.chat(model="gpt-4", messages=[...])`    | 2 次          | LangChain 业务、不关心厂商差异 |
| **同风格透传** | `wrap_openai(...).chat.completions.create(...)` | **0 次**      | 已用原生 SDK、不想改代码       |
| **跨风格透传** | `wrap_openai(...) → 路由到 Anthropic`              | 2 次（adapter） | 需要跨厂商 Fallback       |


**实际案例**：业务用 `WrappedOpenAI` 调用 `gpt-4.1`，配置了 Anthropic Claude 作为 Fallback：

1. 首次调用路由到 OpenAI → 0 次转换，原生性能
2. OpenAI 429 → Engine 自动 Failover 到 Anthropic → `openai_to_anthropic` adapter 翻译
3. 业务方收到的仍是 OpenAI 格式响应 → **完全无感知**

### 4.2 整体数据流

```
业务入口（三选一）
  ├─ ChatModelHub (LangChain)
  ├─ ModelHubClient (裸客户端)
  └─ WrappedSDK (原生 SDK 透传)
       ↓
ModelRequest（统一抽象）
  passthrough_mode + messages / raw_request
       ↓
CoreEngine（单一入口 invoke()）
  ① 验证请求
  ② 插件排除不可用 Endpoint（熔断器 / 限流器）
  ③ 自适应权重调整
  ④ Router 选择 Endpoint
  ⑤ Provider 调用
       ↓
Provider 三岔分派
  ├─ 统一模式：_invoke_unified（2 次转换）
  ├─ 同风格透传：_invoke_passthrough（0 次转换）
  └─ 跨风格透传：_invoke_adapted（adapter 翻译）
       ↓
原生 SDK（openai / anthropic / genai / boto3）
```

> [!note] 能力复用矩阵（核心保证）
> 三种模式共享同一套基础设施——业务方拿不同入口、走不同模式，但任何模式都享受完整能力栈：路由决策、会话粘性、重试机制、429 退避、熔断保护、限流控制、Fallback、插件系统。

---

## 5. 我做的核心模块（按重要性讲）

### 5.1 ⭐ 自适应权重插件（核心创新，最该讲的）

> 代码：`core/plugins/adaptive_weight.py`（520 行，25 个单元测试）

#### 5.1.1 问题 / 为什么要做

**现有限流器的局限**：对 429 是二值处理（可用 / 不可用），多 endpoint 场景下容易连锁故障。

```yaml
# 典型配置：GSU 预付费 80% / PayGo 按量 20%
project-summary:gemini-2.5-pro:
  endpoints:
    - provider: vertex
      model: gemini-2.5-pro
      weight: 80
      extra: { billing: gsu }     # 预付费配额，便宜
    - provider: vertex
      model: gemini-2.5-pro
      weight: 20
      extra: { billing: paygo }   # 按量计费，贵
```

**没有自适应权重时的问题**：

```
t=0     正常: GSU 80%, PayGo 20%

t=10    突发流量，GSU 容量打满，返回 429
        → Engine 退避重试 → 又 429 → failover 到 PayGo
        → 限流器标记 GSU is_limited=True，不可用 60 秒

t=10~70 GSU: 完全不可用（被排除）
        PayGo: 承担 100% 流量
        → PayGo 突然从 20% 跳到 100%，成本飙升
        → PayGo 也可能扛不住 → 也 429
        → 两个都不可用 → NoAvailableEndpointError !!!

t=70    GSU 限流过期，突然恢复到 weight=80
        → 80% 流量瞬间涌回 GSU → 又 429 → 又被标记不可用
        → 反复震荡
```

#### 5.1.2 方案 / 自适应权重的核心设计

**理念**：429 是"太忙"不是"坏了"，应该少给流量（降权）而不是暂时别用（熔断）。

**公式**：

```
error_rate = 窗口内 429 次数 / 窗口内总调用次数
multiplier = max(min_weight_ratio, 1.0 - error_rate × penalty_factor)
effective_weight = base_weight × multiplier
```

**默认参数下（penalty_factor=1.5, min_weight_ratio=0.1）的降权曲线**：


| 429 错误率 | multiplier | base=80 → effective | base=20 → effective |
| ------- | ---------- | ------------------- | ------------------- |
| 0%      | 1.0        | 80                  | 20                  |
| 10%     | 0.85       | 68                  | 17                  |
| 33%     | 0.50       | 40                  | 10                  |
| 50%     | 0.25       | 20                  | 5                   |
| 60%+    | 0.10（下限）   | 8                   | 2                   |


#### 5.1.3 三个关键设计（面试重点讲）

**设计① 权重重分配机制**

> [!important] 这是方案的核心创新
> 降权 endpoint 减少的权重，按健康 endpoint 的**原权重比例**重新分配，让小权重 endpoint 在应急时也能承接流量。

**公式**：

```
D = {i : multiplier_i < 1.0}    降权集合（贡献者）
H = {i : multiplier_i = 1.0}    健康集合（接收者）

减少总量：
  Δ = Σ_{i ∈ D} base_weight_i × (1 - multiplier_i)

接收方加成（按原权重比例分）：
  bonus_j = Δ × base_weight_j / Σ_{k ∈ H} base_weight_k    （j ∈ H）

最终 effective weight：
  降权方：effective_i = base_weight_i × multiplier_i
  健康方：effective_j = base_weight_j + bonus_j
```

**数值演练**：

```
配置：gemini=66, gpt-5=34, gpt-4-1=1, o3=1
gemini 和 gpt-5 同时 100% 429（multiplier 都=0.1）

无重分配:
  gemini=6.6, gpt-5=3.4, gpt-4-1=1, o3=1
  gpt-4-1 占比 = 1/12 = 8.3%   ← base=1 的天花板

有重分配:
  减少总量 Δ = 66×0.9 + 34×0.9 = 90
  健康集合 H = {gpt-4-1, o3}，base 比例 1:1
  bonus_gpt-4-1 = 90 × 0.5 = 45
  bonus_o3 = 90 × 0.5 = 45
  
  effective: gemini=6.6, gpt-5=3.4, gpt-4-1=46, o3=46
  gpt-4-1 占比 = 46/102 = 45.1%  ← 真正承接应急流量
```

**为什么按原权重比例分**：配置者写下 `gpt-5: 34, gpt-4-1: 1` 时已经表达了"gpt-5 比 gpt-4-1 重要 34 倍"的偏好。应急时仍应尊重这个比例。

**设计② recovery_rate 限速恢复**

> [!important] 这是重分配机制的必备配套
> 降权立即生效，但恢复受速率限制，避免恢复时流量瞬涨再次 429。

**公式**：

```
target = max(min_weight_ratio, 1.0 - error_rate × penalty_factor)

if target < prev_multiplier:
    multiplier = target                                      # 下行：立即生效
else:
    cap = downgrade_start_multiplier + recovery_rate × dt
    multiplier = min(target, cap)                            # 上行：限速爬升
```

**效果**（recovery_rate=0.05 时）：

```
t=180s  m=0.10  gemini  6.5%, gpt-4-1 45.1%
t=183s  m=0.25  gemini 16.2%, gpt-4-1 37.7%
t=189s  m=0.55  gemini 35.6%, gpt-4-1 23.0%
t=198s  m=1.00  gemini 64.7%, gpt-4-1  1.0%

18 秒线性过渡，给上游配额真正恢复的时间，给备份平滑卸载的时间。
```

**为什么需要限速**：重分配让恢复时刻同时双向变化——主力涨、备份跌，冲击比纯自降权更强。主力 ×11 瞬涌可能再次 429；备份 ÷45 瞬卸资源浪费。

**设计③ 下限保护（min_weight_ratio=0.1）**

> [!note] 这个设计让系统无需专门的状态机就能感知恢复
> 即使 endpoint 100% 429，multiplier 也只压到 0.1，保留 10% 探测流量。

**对比"降到 0"的方案**：

```
方案 1：multiplier 可降到 0
  GSU 100% 429 → multiplier=0 → effective_weight=0 → Router 永不选 GSU
  → GSU 配额恢复了，没人知道，因为没有请求去试
  → 必须靠外部信号（定时探测 / 人工切回）才能发现

方案 2：保留下限（当前方案）
  GSU 100% 429 → multiplier=0.1 → effective_weight=8 → Router 仍小概率选 GSU
  → 少量请求充当探测包
  → GSU 配额一旦恢复，下次探测立刻成功 → 滑窗 429 比例下降 → multiplier 自动回升
```

**代价**：配额没恢复时，那 10% 流量会被 429 拒掉，触发救援链（Retry → Failover）。这是值得的——业务请求被救援链兜住不丢失，同时为系统提供恢复信号。

#### 5.1.4 效果 / 生产案例

**案例：GSU 配额耗尽自动降级**

```yaml
# 配置
summary-gemini:
  endpoints:
    - vertex-gsu (weight=80, GSU 预付费)
    - vertex-paygo (weight=20, 按量付费)
```

- **事件**：2024-11-15 晚高峰，GSU 配额提前耗尽，429 率飙升到 60%
- **Hub 自动处理**：
  1. t=0s → 检测 GSU 429 率 10% → 降权到 w=68（multiplier=0.85）
  2. t=30s → 429 率 30% → 降权到 w=40（multiplier=0.50）
  3. t=60s → 429 率 60% → 降权到 w=8（multiplier=0.10），PayGo 承接 71% 流量
  4. t=90s → 流量高峰过去，GSU 429 率下降 → 18 秒线性恢复到 80%
- **结果**：业务方无感知，成本增加仅 10%（vs 全量 PayGo 增加 50%）

**核心指标**：


| 指标         | 无自适应权重     | 有自适应权重    | 提升      |
| ---------- | ---------- | --------- | ------- |
| **可用性**    | 95.2%      | 99.5%     | +4.3 pp |
| **LLM 成本** | $12,000/月  | $8,400/月  | -30%    |
| **故障恢复时间** | 人工切流 15 分钟 | 自动恢复 2 分钟 | -87%    |


---

### 5.2 ⭐ 跨风格路由适配（技术亮点）

> 代码：`core/adapters/` + `core/providers/base.py`

#### 5.2.1 问题 / 为什么要做

**业务痛点**：业务方用 OpenAI SDK 写的代码，想 Fallback 到 Anthropic / Gemini 怎么办？

**传统方案的问题**：

- 重写代码 → 维护成本高
- 只能 Fallback 到同厂商 → 容错能力受限

#### 5.2.2 方案 / Provider 基类三岔分派

> [!important] 这是 Model Hub 区别于普通 LLM gateway 的核心设计
> 每个 Provider 的 `invoke` 方法根据 `passthrough_mode` 和 `source_style` 三岔分派。

**核心代码**（Provider 基类）：

```python
def invoke(self, request, decision) -> ModelResponse:
    if not request.is_passthrough:
        return self._invoke_unified(request, decision)     # 统一模式
    if request.source_style == self.api_style:
        return self._invoke_passthrough(request, decision) # 同风格透传
    else:
        return self._invoke_adapted(request, decision)     # 跨风格透传
```


| 方法                    | 走它的场景                             | 转换次数            | 厂商独有参数       | 谁实现        |
| --------------------- | --------------------------------- | --------------- | ------------ | ---------- |
| `_invoke_unified`     | 入口是 ChatModelHub / ModelHubClient | 2 次             | 受统一字段限制      | 子类**必须**实现 |
| `_invoke_passthrough` | Wrapped 路由到本家 endpoint            | **0 次**         | 无损           | 基类默认实现     |
| `_invoke_adapted`     | Wrapped 路由到异家 endpoint            | 2 次（adapter 内部） | 经 adapter 翻译 | 基类默认实现     |


#### 5.2.3 Adapter 矩阵

**只在真正跨风格时才走**。adapter 是 N×N 矩阵里的稀疏几条边：


| Source ↓ / Target → | openai | anthropic | genai | bedrock |
| ------------------- | ------ | --------- | ----- | ------- |
| **openai**          | —      | ✓         | ✓     | ✓       |
| **anthropic**       | ✓      | —         | ✗     | ✗       |
| **genai**           | ✓      | ✗         | —     | ✗       |


**每个 Adapter 实现 4 个方法**：

```python
class APIStyleAdapter(ABC):
    def adapt_request(self, kwargs) -> dict          # 入参翻译
    def adapt_response(self, response) -> Any        # 响应翻译
    def adapt_stream(self, stream) -> Iterator       # 流式响应翻译
    def get_target_method(self, source_method) -> str  # 方法路径映射
```

#### 5.2.4 技术亮点

**亮点① OPENAI_COMPATIBLE 家族**

> [!note] 容易被忽略的细节
> Wrapped OpenAI 路由到 Azure / 火山 / DashScope / LiteLLM 是 SAME_STYLE，不是 CROSS_STYLE。

```python
OPENAI_COMPATIBLE = {"openai", "azure_openai", "volcengine", "dashscope", "litellm"}

def get_adapter(source_style, target_provider):
    if source_style == "openai" and target_provider in OPENAI_COMPATIBLE:
        return None   # ← 不需要 adapter，走零转换
    ...
```

**亮点② 新增 Claude on Vertex AI 支持**

- 在 Vertex AI Provider 基类补充 Anthropic API 风格适配
- 业务代码 0 改动，配置添加一个 endpoint 即可

**亮点③ 多模态内容翻译**（ADR-024）

OpenAI 的图片是 `image_url` 对象，Anthropic 的图片要转成 `image` block：

```python
# OpenAI 格式
{
  "type": "image_url",
  "image_url": {"url": "https://..."}
}

# Anthropic 格式（adapter 翻译）
{
  "type": "image",
  "source": {
    "type": "url",
    "url": "https://..."
  }
}
```

---

### 5.3 Fallback 增强

> 代码：`core/engine.py`（Fallback 逻辑 190-246 行）

#### 5.3.1 问题 / 原有设计的局限

**原有设计**：单级 Fallback，配置 `fallback_model` 字段。

**局限**：

- 只能降级一次
- 没有环路检测
- 降级路径不可追溯

#### 5.3.2 我的改进

**改进① 多级 Fallback 链**

```yaml
policies:
  summary-gemini:
    fallback_model:
      - summary-gpt      # 一级降级：跨厂商
      - summary-claude   # 二级降级
      - summary-free     # 三级降级：免费模型聚合
```

**改进② 环路检测**

```python
# 自动检测 A → B → A 的循环引用
# 启动时报错而非运行时死循环
```

**改进③ 元数据回写**

```python
# 响应中记录实际命中的模型
response.metadata["effective_model"] = "summary-gpt"  # 实际走了 Fallback
response.metadata["fallback_chain"] = ["summary-gemini", "summary-gpt"]
```

**业务价值**：成本审计、降级路径可追溯。

#### 5.3.3 透传模式的 Fallback 保留（ADR-023）

> [!important] 关键约束
> Fallback 阶段必须保留 `passthrough_mode / source_style / raw_request / raw_method` 4 个字段，否则透传请求在 Fallback 阶段会退化成空的统一请求。

```python
# Fallback 阶段必须保留透传字段
fallback_request = ModelRequest(
    passthrough_mode=original_request.passthrough_mode,  # 保留
    source_style=original_request.source_style,          # 保留
    raw_request=original_request.raw_request,            # 保留
    raw_method=original_request.raw_method,              # 保留
    # ... 其他字段
)
```

#### 5.3.4 生产案例

**案例：跨厂商 Fallback 救命**

- **配置**：`gemini → gpt → claude` 三级 Fallback
- **事件**：2024-12-08 Gemini 全球故障 2 小时
- **Hub 处理**：自动切换到 GPT，业务方日志显示 `effective_model=gpt-4o`
- **结果**：可用性 100%，成本增加 15%（GPT 比 Gemini 略贵），无客诉

---

### 5.4 LangChain Runnable 适配

> 代码：`sdk/chains/chat_model_hub.py`

#### 5.4.1 目标

让 Model Hub 无缝接入 LangChain / LangGraph 业务。

#### 5.4.2 实现

**① ChatModelHub 继承 BaseChatModel**

```python
class ChatModelHub(BaseChatModel):
    def _generate(self, messages, stop=None, run_manager=None, **kwargs):
        # 调用 Hub 的 CoreEngine
    
    def _stream(self, messages, stop=None, run_manager=None, **kwargs):
        # 流式调用
```

**② 补齐三个参数管理 API**

```python
client.bind(temperature=0.7)               # 绑定默认参数
client.with_fallbacks([fallback_client])   # LangChain 原生 Fallback
client.configurable_fields(...)            # 动态切换模型
```

**③ 元数据追溯成本**

```python
# LangChain 的 AIMessage.response_metadata 携带 Hub 元数据
response_metadata = {
    "model_name": "gemini-2.5-pro",
    "effective_endpoint": "vertex-gsu-us",
    "token_usage": {"prompt_tokens": 120, "completion_tokens": 80},
    "cost_usd": 0.0024  # Hub 自动计算
}
```

**业务价值**：Plaud Summary / Ask / Project Summary 等 LangChain 业务无改动即享受 Hub 能力。

---

### 5.5 AppConfig 热更新 + 多格式存储

> 代码：`core/config/store.py`

#### 5.5.1 技术方案

**① 抽象 ConfigStore 接口**

```python
class ConfigStore(ABC):
    @abstractmethod
    def load_config(self, env: str) -> dict

class FileConfigStore(ConfigStore)       # YAML / JSON
class AppConfigConfigStore(ConfigStore)  # AWS AppConfig
class HttpConfigStore(ConfigStore)       # HTTP API
```

**② 热更新机制**

```python
# CoreEngine 内部定时轮询（默认 60s）
if config_store.version > current_version:
    reload_config()
    logger.info(f"Config reloaded: v{current_version} → v{new_version}")
```

**③ 灰度发布支持**

AppConfig 的百分比部署：

```yaml
# 10% 流量试跑新配置
deployment_strategy: LINEAR_10_PERCENT_EVERY_10_MINUTES
```

#### 5.5.2 生产案例

- 生产环境模型切换（Gemini 2.5 → 2.0）：修改 YAML → AppConfig 推送 → 10 分钟全量覆盖，**0 停机**
- 紧急降权某 endpoint（429 过多）：改 `weight: 80 → 20` → 1 分钟生效

---

## 6. 我解决的技术难点（STAR 式）

### 6.1 参数透传的三层优先级

> 详细文档：[[参数透传机制详解]] 20 页技术文档

**S（问题）**：Hub 的参数来源有三层（YAML 配置、构造器、调用时 kwargs），各 Provider 处理方式不同。

**T/A（方案）**：

**① 统一优先级规则**

```
调用时 kwargs > endpoint 配置 > provider defaults > engine_config
```

**② provider_params 透传袋**

```python
# Hub 认识的参数：提升到 ModelRequest 顶层
temperature, max_tokens, ...

# Hub 不认识的参数：打包进 provider_params
extra_body, metadata, ...
```

**③ 各 Provider 差异处理**


| Provider  | 处理方式                     | 不认识的参数    |
| --------- | ------------------------ | --------- |
| OpenAI    | 整袋透传 `**provider_params` | 服务端返回 400 |
| Anthropic | 整袋透传 + 显式 pop            | 服务端返回 400 |
| Gemini    | 白名单逐项 `.get`             | **静默丢弃**  |


**R（结果）**：最坑的是 Gemini 的静默丢弃，业务方以为参数生效了（本地不报错），实际没传给服务端。输出 20 页技术文档 + 单元测试覆盖各 Provider 的参数校验。

### 6.2 多 endpoint 的熔断与限流联动

**S（问题）**：熔断器统计 429 会和自适应权重冲突（双重惩罚）。

**T/A（方案）**：

**① 错误分类隔离**

```yaml
rate_limit:
  exclude_429_from_circuit_breaker: true  # 429 不计入熔断统计

adaptive_weight:
  error_codes: [429]  # 只对 429 降权
```

**② soft_limit_on_429 联动**

```python
# 自适应权重启用时，限流器不排除 endpoint
if adaptive_weight.enabled:
    rate_limiter.soft_limit_on_429 = True
```

**③ 三种健康信号互不重叠**


| 信号        | 处理者         | 行为                     |
| --------- | ----------- | ---------------------- |
| 429 限流    | 自适应权重 + 限流器 | 渐进降权 + 解析 Retry-After  |
| 5xx 服务端错误 | 熔断器         | 完全熔断（OPEN → HALF_OPEN） |
| 发送过快      | 限流器令牌桶      | 主动钳制                   |


**R（结果）**：三种信号各司其职，不会重复惩罚同一个错误。

---

## 7. 工程实践与质量保障

### 7.1 测试覆盖


| 模块             | 单元测试数 | 覆盖率 | 关键测试点                           |
| -------------- | ----- | --- | ------------------------------- |
| CoreEngine     | 45 个  | 92% | Retry、Failover、Fallback 链       |
| AdaptiveWeight | 25 个  | 95% | 权重重分配、recovery_rate、下限保护        |
| Adapter        | 18 个  | 88% | openai ↔ anthropic / genai 双向翻译 |
| Provider       | 30 个  | 90% | 各厂商 SDK 调用、错误处理                 |


**特殊测试场景**：

- 模拟 429 连锁故障（3 个 endpoint 先后 429）
- Fallback 链环路检测（A → B → A）
- 透传模式下的多模态内容（图片 + 文本）翻译

### 7.2 可观测性

**① Langfuse 集成**

- 每次调用自动记录：模型、tokens、延迟、成本
- Fallback 路径可视化：`gemini (429) → gpt (成功)`

**② OTel 分布式追踪**

- Span 层次：`invoke → route → provider_call → sdk_http`
- 自定义 Attributes：`effective_endpoint`, `fallback_chain`

**③ 结构化日志**

```python
logger.info(
    "Adaptive weight adjusted",
    extra={
        "endpoint_id": "vertex-gsu-us",
        "multiplier": 0.25,
        "error_rate": 0.50,
        "effective_weight": 20
    }
)
```

### 7.3 文档输出

- [[PlaudModelHub完全指南]] 200 页技术文档
- [[PlaudModelHub架构介绍]] 30 页架构讲解（配 draw.io 架构图）
- [[PlaudModelHub调度与容错]] 30 页容错设计
- [[自适应权重插件设计方案]] 17 页 ADR
- [[参数透传机制详解]] 20 页

---

## 8. 面试高频追问 & 应答要点

> [!question] 自适应权重和熔断器的区别？

两者处理不同类型的故障：

- **自适应权重**：429（太忙） → **渐进降权**，multiplier 连续变化（1.0 → 0.7 → 0.4 → 0.1）
- **熔断器**：5xx（坏了） → **二值切换**，CLOSED / OPEN / HALF_OPEN 三状态

为什么不能合并？429 是"我没坏、只是你请求太多"，减少请求后很快恢复；5xx 是"我出问题了"，需要等修复。前者适合连续调整，后者适合完全隔离。

通过 `exclude_429_from_circuit_breaker=true` 隔离，避免双重惩罚。

---

> [!question] 权重重分配的"维度"是什么意思？

不是指"维度"，是指**降权部分的重新分配方式**。

关键点：降权 endpoint 减少的权重（Δ），按健康 endpoint 的**原权重比例**分配回去。

为什么不是平均分？因为配置者写下 `gpt-5: 34, gpt-4-1: 1` 时已经表达了偏好——应急时仍应尊重这个比例。

效果：双 endpoint 同时故障时，小权重 endpoint 从 8% 占比放大到 45%，真正承接应急流量。

---

> [!question] recovery_rate 怎么选值？

默认 0.05（每秒），从 0.1 爬到 1.0 需要 18 秒。取值依据：

- **太快（0.1）**：9 秒爬满，上游配额可能还没恢复 → 再次 429
- **太慢（0.02）**：45 秒爬满，备份 endpoint 承接时间过长 → 成本增加

生产调优建议：

- GSU 配额恢复快（几秒）→ 0.1
- 默认场景 → 0.05
- 上游配额恢复慢（分钟级）→ 0.02

也可以设为 0 关闭限速，但只适合单 endpoint 故障场景，多 endpoint 故障时会振荡。

---

> [!question] 跨风格透传的技术难点？

难点在于各家 SDK 的**参数格式差异** + **流式响应格式差异**。

**① 参数翻译**：

- OpenAI 的 `messages` 是扁平数组，Anthropic 的 `system` 要单独提取
- OpenAI 的图片是 `image_url` 对象，Gemini 的图片要转成 `inline_data` 的 base64

我实现了 `OpenAIToAnthropicAdapter` / `OpenAIToGenAIAdapter`，每个 Adapter 有 4 个方法：`adapt_request` / `adapt_response` / `adapt_stream` / `get_target_method`。

**② 流式翻译**：

- OpenAI 的 stream 是 `delta.content`
- Anthropic 的是 `content_block_delta.delta.text`

Adapter 必须把 Anthropic 的 chunk 重新包装成 OpenAI 格式，业务方的 `for chunk in stream` 完全不用改。

**③ Fallback 保留（ADR-023）**：
跨风格 Fallback 时，`raw_request` 可能不适用新 Provider。解决方案是 Fallback 阶段必须保留 `passthrough_mode / source_style / raw_request / raw_method` 4 个字段，否则透传请求会退化成空的统一请求。

---

> [!question] 为什么不直接用 LiteLLM？

LiteLLM 是通用代理，但有三个限制：

1. **容错不足**：只有基础重试，没有我们需要的自适应降权 + 权重重分配
2. **可观测性弱**：Langfuse 集成浅，无法追溯 Fallback 链、成本归因
3. **定制困难**：Plaud 的 GSU + PayGo 成本结构、多级 Fallback 需求，LiteLLM 无法覆盖

Model Hub 定位是 Plaud 内部基础设施，深度集成业务需求，不是替代 LiteLLM，而是在它之上做**容错 + 成本优化**。

---

> [!question] 你具体负责哪些、和团队怎么分工？

**我独立负责**：

- 自适应权重插件（从 0 到 1 设计实现）
- 跨风格路由适配（OpenAI ↔ Anthropic / GenAI 双向 Adapter）
- Fallback 增强（多级链 + 环路检测 + 元数据回写）
- LangChain 集成（ChatModelHub 适配）
- AppConfig 热更新（抽象 ConfigStore 接口）

**全程参与**：

- Hub 架构设计（单一入口 + 三种调用模式）
- Provider 基类设计（三岔分派）
- 参数透传机制设计（三层优先级）
- 插件系统设计（Protocol 接口）
- 测试体系建设（95% 覆盖率）

**团队协作**：

- 用户：Plaud Summary / Ask / Project Summary 三条业务线，10+ 工程师
- 技术分享：2024-11 团队技术分享《Model Hub 调度与容错》（30 分钟）
- 新人培训：2025-01 Onboarding 培训讲师

---

## 9. 关键代码位置速查（被要求看代码时）


| 模块               | 路径                                |
| ---------------- | --------------------------------- |
| **自适应权重插件**      | `core/plugins/adaptive_weight.py` |
| **跨风格 Adapter**  | `core/adapters/`                  |
| **Provider 基类**  | `core/providers/base.py`          |
| **CoreEngine**   | `core/engine.py`                  |
| **Fallback 逻辑**  | `core/engine.py:190-246`          |
| **LangChain 集成** | `sdk/chains/chat_model_hub.py`    |
| **ConfigStore**  | `core/config/store.py`            |
| **插件 Protocol**  | `core/plugins/base.py`            |


---

## 10. 技术栈


| 类别          | 用到的东西                                         |
| ----------- | --------------------------------------------- |
| **语言**      | Python 3.10+                                  |
| **框架**      | FastAPI                                       |
| **LLM SDK** | openai, anthropic, google-generativeai, boto3 |
| **可观测性**    | Langfuse, OpenTelemetry                       |
| **配置管理**    | AWS AppConfig, YAML                           |
| **测试**      | pytest, unittest.mock                         |
| **工具**      | uv, Ruff, MyPy                                |


