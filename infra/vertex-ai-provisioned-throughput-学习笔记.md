# Vertex AI Provisioned Throughput 学习笔记

> **来源**: [Google Cloud 官方文档](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/overview)
> **更新日期**: 2026-02-25

---

## 目录

1. [概述与核心概念](#1-概述与核心概念)
2. [支持的模型](#2-支持的模型)
3. [容量计算方法](#3-容量计算方法)
4. [购买流程](#4-购买流程)
5. [使用方式](#5-使用方式)
6. [Live API 专项](#6-live-api-专项)
7. [Veo 视频生成专项](#7-veo-视频生成专项)
8. [Single Zone PT](#8-single-zone-pt)
9. [错误处理与重试策略](#9-错误处理与重试策略)
10. [最佳实践与决策清单](#10-最佳实践与决策清单)

---

## 1. 概述与核心概念

### 1.1 为什么需要 Provisioned Throughput？

在 Vertex AI 上使用生成式 AI 模型时，默认的按量付费（Pay-as-you-go）模式存在两个痛点：

- **吞吐量不可预测** — 多租户共享池中，高峰期请求可能被限流（429 错误）
- **成本不可控** — 流量波动直接映射到账单波动，难以做预算

**Provisioned Throughput**（预置吞吐量，简称 PT）就是为了解决这两个问题而生的：它是一种 **固定费用、固定期限** 的订阅方式，为你在指定区域预留专属的模型推理容量。

### 1.2 核心概念：GSU

**GSU（Generative AI Scale Unit，生成式 AI 规模单位）** 是 PT 的计量单位。可以把它理解为一个"吞吐量包"：

- 每个 GSU 对应一定的 **每秒处理能力**（因模型而异）
- 例如：1 GSU 的 Gemini 2.0 Flash = 3,360 tokens/秒
- 购买 PT 时，你买的就是若干个 GSU

### 1.3 适用场景


| 场景              | 为什么选 PT         |
| --------------- | --------------- |
| 实时聊天机器人 / Agent | 需要稳定低延迟，不能容忍限流  |
| 高吞吐量生产工作负载      | 持续高并发，按量付费反而更贵  |
| 可预测的用户体验        | SLA 保障，不受其他租户影响 |
| 确定性成本控制         | 固定月费 + 可控溢出费用   |


### 1.4 PT vs 其他消费模式对比

```
┌─────────────────────┬───────────────┬────────────────┬─────────────────┐
│       维度          │  Pay-as-you-go │ Flex PayGo     │ Provisioned PT  │
├─────────────────────┼───────────────┼────────────────┼─────────────────┤
│ 计费方式            │ 按 token 计费  │ 按 token，排队 │ 固定期限订阅    │
│ 吞吐量保障          │ 无（尽力而为） │ 无             │ 有（专属容量）  │
│ 成本可预测性        │ 低            │ 低             │ 高              │
│ 最低承诺            │ 无            │ 无             │ 1 周起          │
│ 超额处理            │ N/A           │ N/A            │ 溢出到 PayGo    │
└─────────────────────┴───────────────┴────────────────┴─────────────────┘
```

---

## 2. 支持的模型

### 2.1 Google 自有模型 — 文本生成


| 模型                    | 模型 ID                       | 吞吐量/GSU     |
| --------------------- | --------------------------- | ----------- |
| Gemini 3.1 Pro        | `gemini-3.1-pro-preview`    | 500 tok/s   |
| Gemini 3 Flash        | `gemini-3-flash-preview`    | 2,015 tok/s |
| Gemini 3 Pro          | `gemini-3-pro-preview`      | 500 tok/s   |
| **Gemini 2.5 Pro**    | `gemini-2.5-pro`            | 650 tok/s   |
| **Gemini 2.5 Flash**  | `gemini-2.5-flash`          | 2,690 tok/s |
| Gemini 2.5 Flash-Lite | `gemini-2.5-flash-lite`     | 8,070 tok/s |
| Gemini 2.0 Flash      | `gemini-2.0-flash-001`      | 3,360 tok/s |
| Gemini 2.0 Flash-Lite | `gemini-2.0-flash-lite-001` | 6,720 tok/s |


> **规律**: 模型越小越快，每 GSU 提供的吞吐量越高。Flash-Lite 系列性价比最高。

### 2.2 Google 自有模型 — 多模态 & 图像


| 模型                        | 模型 ID                                | 吞吐量/GSU     | 单位     |
| ------------------------- | ------------------------------------ | ----------- | ------ |
| Gemini 3 Pro Image        | —                                    | 500 tok/s   | tokens |
| Gemini 2.5 Flash Image    | —                                    | 2,690 tok/s | tokens |
| Gemini 2.5 Flash Live API | `gemini-live-2.5-flash-native-audio` | 1,620 tok/s | tokens |
| Imagen 4 Ultra            | `imagen-4.0-ultra-generate-001`      | 0.015 img/s | 图片     |
| Imagen 4                  | `imagen-4.0-generate-001`            | 0.02 img/s  | 图片     |
| Imagen 4 Fast             | —                                    | 0.04 img/s  | 图片     |
| Imagen 3 Generate 002     | `imagen-3.0-generate-002`            | 0.02 img/s  | 图片     |
| Imagen 3 Generate 001     | —                                    | 0.025 img/s | 图片     |
| Imagen 3 Fast             | —                                    | 0.05 img/s  | 图片     |
| Virtual Try-On 001        | `virtual-try-on-001`                 | 0.02 img/s  | 图片     |


### 2.3 Google 自有模型 — 视频生成（Veo）


| 模型           | 模型 ID                       | 吞吐量/GSU     |
| ------------ | --------------------------- | ----------- |
| Veo 3.1      | `veo-3.1-generate-001`      | 0.004 视频秒/s |
| Veo 3.1 Fast | `veo-3.1-fast-generate-001` | 0.008 视频秒/s |
| Veo 3        | `veo-3.0-generate-001`      | 0.004 视频秒/s |
| Veo 3 Fast   | `veo-3.0-fast-generate-001` | 0.008 视频秒/s |


> **注意**: Veo 模型的单位是"视频秒/每秒"，即每秒能生成多少秒的视频。详见 [第7节](#7-veo-视频生成专项)。

### 2.4 合作伙伴模型 — Anthropic Claude


| 模型                | 吞吐量/GSU     | 最低购买量      | GSU 增量 |
| ----------------- | ----------- | ---------- | ------ |
| Claude Sonnet 4.6 | 350 tok/s   | **25 GSU** | 1      |
| Claude Opus 4.6   | 210 tok/s   | **35 GSU** | 1      |
| Claude Opus 4.5   | 210 tok/s   | **35 GSU** | 1      |
| Claude Sonnet 4.5 | 350 tok/s   | **25 GSU** | 1      |
| Claude Opus 4.1   | 70 tok/s    | **35 GSU** | 1      |
| Claude Haiku 4.5  | 1,050 tok/s | **8 GSU**  | 1      |
| Claude Opus 4     | 70 tok/s    | **35 GSU** | 1      |
| Claude Sonnet 4   | 350 tok/s   | **25 GSU** | 1      |


> **关键区别**: Claude 模型的吞吐量 = **输入 + 输出 token 总和**（跨所有并发请求的每秒总量）。Google 模型和开源模型最低 1 GSU 起购，而 Claude 的最低门槛显著更高（8~35 GSU）。

### 2.5 开源模型（Preview）


| 模型                      | 模型 ID                                     | 吞吐量/GSU      |
| ----------------------- | ----------------------------------------- | ------------ |
| DeepSeek-OCR            | `deepseek-ocr-maas`                       | 3,360 tok/s  |
| DeepSeek-V3.2           | `deepseek-v3.2-maas`                      | 1,680 tok/s  |
| Kimi K2 Thinking        | `kimi-k2-thinking-maas`                   | 1,680 tok/s  |
| Llama 3.3 70B           | `llama-3.3-70b-instruct-maas`             | 1,400 tok/s  |
| Llama 4 Maverick        | `llama-4-maverick-17b-128e-instruct-maas` | 2,800 tok/s  |
| Llama 4 Scout           | `llama-4-scout-17b-16e-instruct-maas`     | 4,035 tok/s  |
| MiniMax M2              | `minimax-m2-maas`                         | 3,360 tok/s  |
| OpenAI gpt-oss 120B     | `gpt-oss-120b-maas`                       | 11,205 tok/s |
| OpenAI gpt-oss 20B      | `gpt-oss-20b-maas`                        | 14,405 tok/s |
| Qwen3 235B              | `qwen3-235b-a22b-instruct-2507-maas`      | 4,035 tok/s  |
| Qwen3 Coder             | `qwen3-coder-480b-a35b-instruct-maas`     | 1,010 tok/s  |
| Qwen3-Next-80B Instruct | `qwen3-next-80b-a3b-instruct-maas`        | 6,725 tok/s  |
| Qwen3-Next-80B Thinking | `qwen3-next-80b-a3b-thinking-maas`        | 6,725 tok/s  |


### 2.6 重要约束


| 约束                              | 说明                                                           |
| ------------------------------- | ------------------------------------------------------------ |
| **必须使用精确版本 ID**                 | 不能用模型别名（如 `gemini-flash`），必须用完整 ID（如 `gemini-2.0-flash-001`） |
| **不支持批量预测**                     | PT 仅用于在线推理，Batch Prediction 不适用                              |
| **外部工具不含 PT 配额**                | Grounding、搜索等工具调用的配额独立于 PT                                   |
| **Vertex AI Search/Agents 不保证** | 由这些服务发起的模型调用不受 PT 保障                                         |
| **订阅期限**                        | 1 周 / 1 个月 / 3 个月 / 1 年                                      |
| **支持微调模型**                      | Google 模型支持，开源模型为 Preview                                    |
| **输出 token 消耗更高**               | 所有模型的输出 token burndown rate 都显著高于输入                          |


---

## 3. 容量计算方法

> **来源**: [measure-provisioned-throughput](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/measure-provisioned-throughput)

### 3.1 核心公式

计算你需要多少 GSU，分三步：

```
第一步: 计算单次请求的等效 token 数
    adjusted_tokens = Σ(各类型输入 × 对应 burndown rate) + (输出 × 输出 burndown rate)

第二步: 乘以每秒查询数（QPS）
    total_throughput = adjusted_tokens × QPS

第三步: 除以该模型每 GSU 的吞吐量
    GSUs_needed = ⌈total_throughput ÷ throughput_per_GSU⌉  （向上取整）
```

### 3.2 Burndown Rate（消耗速率）

**Burndown Rate** 是一个转换系数，将不同类型的输入（文本、音频、图片等）统一换算为"等效输入 token"。

以 **Gemini 2.0 Flash** 为例：


| 输入/输出类型        | Burndown Rate | 含义                                      |
| -------------- | ------------- | --------------------------------------- |
| 文本输入 token     | 1             | 1 个 text token = 1 个等效 token            |
| 图片输入 token     | 1             | 1 个 image token = 1 个等效 token           |
| 视频输入 token     | 1             | 1 个 video token = 1 个等效 token           |
| 音频输入 token     | 7             | 1 个 audio token = 7 个等效 token（音频处理成本更高） |
| 文本输出 token     | 4             | 1 个输出 token = 4 个等效 token（生成比理解贵）       |
| **缓存命中 token** | **0.1**       | 使用 Context Caching 时，缓存部分只算 10%（大幅节省）   |


> **关键洞察**:
>
> - 输出 token 的 burndown rate 通常是输入的 **4 倍**，这意味着输出密集型任务需要更多 GSU
> - 善用 **Context Caching** 可以将重复输入的消耗降低到 1/10（Gemini 2.5 Pro 已确认支持 0.1 burndown rate）
> - 官方文档强调：PT 不是简单地按 QPM（每分钟查询数）计量，而是综合考虑请求大小、响应大小和 QPS

### 3.3 完整计算示例

**场景**: 使用 Gemini 2.0 Flash 构建多模态客服系统

- QPS = 10（每秒 10 个请求）
- 每次请求：1,000 个文本 token + 500 个音频 token（输入）
- 每次请求：300 个文本 token（输出）

**计算过程**:

```
1. 输入等效 token:
   文本: 1,000 × 1 = 1,000
   音频: 500 × 7   = 3,500
   输入合计         = 4,500

2. 输出等效 token:
   文本: 300 × 4    = 1,200

3. 单次请求总计:
   4,500 + 1,200    = 5,700 等效 token

4. 总吞吐量:
   5,700 × 10 QPS   = 57,000 tok/s

5. 所需 GSU:
   57,000 ÷ 3,360   = 16.96 → 向上取整 = 17 GSU
```

### 3.4 重要注意事项

- PT 是按 **项目 + 区域 + 模型 + 版本** 绑定的，不能跨项目/区域共享
- 未用完的吞吐量 **不会滚动累积**
- 计量单位是 **每秒**（tokens/s），不是每分钟（TPM）
- 购买页面提供在线估算工具，建议优先使用

---

## 4. 购买流程

> **来源**: [purchase-provisioned-throughput](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/purchase-provisioned-throughput)

### 4.1 前置条件

需要 `roles/aiplatform.provisionedThroughputAdmin` 角色，包含以下关键权限：


| 权限                                         | 说明      |
| ------------------------------------------ | ------- |
| `aiplatform.provisionedThroughputs.create` | 提交订单    |
| `aiplatform.provisionedThroughputs.get`    | 查看特定订单  |
| `aiplatform.provisionedThroughputs.list`   | 查看所有订单  |
| `aiplatform.provisionedThroughputs.update` | 修改订单    |
| `aiplatform.provisionedThroughputs.cancel` | 取消待审核订单 |


> **重要**: 购买即绑定，**合同期内不可取消**（仅 Pending Review 状态可取消）

### 4.2 通过 Console 购买步骤

```
1. 进入 Google Cloud Console → Vertex AI → Provisioned Throughput 页面
2. 点击 "New order"，输入订单名称
3. 选择模型和区域
4. 使用估算工具配置参数:
   ├── QPS（每秒查询数）
   ├── 输入/输出 token 数
   ├── 媒体 token（图片/视频/音频，如适用）
   └── 缓存命中率（%）
5. 确认 GSU 数量和预估月费
6. 选择期限: 1 周 | 1 个月 | 3 个月 | 1 年
7. [可选] 设定开始日期（最多提前 2 周）
8. 设置自动续费偏好
9. 审核价格 → 输入 "CONFIRM" 确认文本 → 提交
```

### 4.3 订单生命周期

```
Pending Review → Approved → Active（开始计费）→ Expired
     │               │              │
     │ 可取消         │ 不可修改      │ 可修改:
     │               │              │ ✓ 增加 GSU（立即生效）
     │ 处理时间:      │              │ ✓ 减少 GSU（下次续费生效）
     │ 分钟 ~ 数周    │              │ ✓ 换模型（同厂商）
     │ (取决于容量)    │              │ ✓ 换区域（10 个工作日）
     │               │              │ ✓ 自动续费开关
     │               │              │ ✗ 不能取消
```

> **修改限制**: 距离到期不足 5 天且未开启自动续费时，不允许修改。有待处理的变更时也不允许新的修改。所有变更按尽力交付（best-effort）处理，通常在 10 个工作日内完成。

### 4.4 自动续费规则


| 期限   | 是否支持自动续费 |
| ---- | -------- |
| 1 周  | 不支持      |
| 1 个月 | 支持       |
| 3 个月 | 支持       |
| 1 年  | 支持       |


### 4.5 超额处理

超出预留容量的请求 **默认** 按 Pay-as-you-go 价格处理（自动溢出）。你可以通过 HTTP Header 控制是否允许溢出（详见第5节）。

### 4.6 Preview 模型迁移

如果购买了 Preview 版本模型的 PT，当 GA 版本发布后，可以将订单 **单向迁移** 到 GA 版本，也可以继续使用 Preview 版本。

---

## 5. 使用方式

> **来源**: [use-provisioned-throughput](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/use-provisioned-throughput)

### 5.1 配额执行窗口（Quota Enforcement Window）

> **来源**: 此内容来自 [use-provisioned-throughput](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/use-provisioned-throughput) 页面中的 PT quota 执行机制说明。

Vertex AI 使用**动态执行窗口**来检查配额。窗口越短，意味着配额检查越频繁、粒度越细：


| GSU 数量   | 执行窗口       | 适用场景              |
| -------- | ---------- | ----------------- |
| ≤ 3 GSU  | 40 ~ 120 秒 | 小规模部署，允许较大的单次请求突发 |
| > 3 GSU  | 5 ~ 30 秒   | 中等规模              |
| ≥ 50 GSU | 1 ~ 5 秒    | 高频请求，接近实时配额执行     |


窗口会根据模型类型和购买的 GSU 数量自动调整，以优化流量突发时的稳定性。

> **理解窗口的意义**: 官方文档原文 — "Your total usage over any [window duration]-second window can't exceed [tokens per second × window duration] tokens." 即在一个窗口周期内，你可以灵活分配 token 消耗（集中发送或均匀分布），只要不超过总量即可。
>
> 示例：1 GSU = 3,360 tok/s 的 Gemini 2.0 Flash，在 60 秒窗口内可消耗 3,360 × 60 = 201,600 tokens。

### 5.2 请求路由控制

通过 HTTP Header `X-Vertex-AI-LLM-Request-Type` 控制每个请求的路由行为：


| Header 值    | 行为                   | 场景           |
| ----------- | -------------------- | ------------ |
| 不设置（默认）     | 优先用 PT，超额自动溢出到 PayGo | 大多数场景推荐      |
| `dedicated` | 强制使用 PT，超额返回 429     | 严格成本控制，不允许溢出 |
| `shared`    | 完全绕过 PT，走 PayGo      | 测试、非关键请求     |


> **Flex PayGo 模式**: 使用 Flex PayGo 时需同时设置两个 Header：
>
> ```
> X-Vertex-AI-LLM-Request-Type: shared
> X-Vertex-AI-LLM-Shared-Request-Type: flex
> ```

### 5.3 配额调和（Quota Reconciliation）

系统在处理请求时，会先根据 **初始估算** 的输出大小预扣配额。当实际输出完成后，系统会对比实际输出与预估之差，**回溯调整** 可用配额。这意味着监控面板上可能出现短暂的配额波动。

### 5.4 代码示例

#### Python（Google Gen AI SDK）

```python
from google import genai
from google.genai.types import HttpOptions

# 方式 1: 默认 — PT 优先，自动溢出
client = genai.Client(
    vertexai=True,
    project="my-project",
    location="us-central1",
    http_options=HttpOptions(api_version="v1"),
)

# 方式 2: 强制 PT（超额返回 429）
client_dedicated = genai.Client(
    vertexai=True,
    project="my-project",
    location="us-central1",
    http_options=HttpOptions(
        api_version="v1",
        headers={"X-Vertex-AI-LLM-Request-Type": "dedicated"},
    ),
)

# 方式 3: 强制 PayGo（跳过 PT）
client_shared = genai.Client(
    vertexai=True,
    project="my-project",
    location="us-central1",
    http_options=HttpOptions(
        api_version="v1",
        headers={"X-Vertex-AI-LLM-Request-Type": "shared"},
    ),
)

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="How does AI work?",
)
```

#### Go

```go
client, err := genai.NewClient(ctx, &genai.ClientConfig{
    HTTPOptions: genai.HTTPOptions{
        APIVersion: "v1",
        Headers: http.Header{
            "X-Vertex-AI-LLM-Request-Type": []string{"shared"},
        },
    },
})
```

#### Node.js

```javascript
const client = new GoogleGenAI({
    vertexai: true,
    project: projectId,
    location: location,
    httpOptions: {
        apiVersion: 'v1',
        headers: {
            'X-Vertex-AI-LLM-Request-Type': 'shared',
        },
    },
});
```

#### Java

```java
Client client = Client.builder()
    .location("us-central1")
    .vertexAI(true)
    .httpOptions(
        HttpOptions.builder()
            .apiVersion("v1")
            .headers(Map.of(
                "X-Vertex-AI-LLM-Request-Type", "shared"))
            .build())
    .build();
```

#### REST / cURL

```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -H "X-Vertex-AI-LLM-Request-Type: dedicated" \
  "https://LOCATION-aiplatform.googleapis.com/v1/projects/PROJECT_ID/locations/LOCATION/publishers/google/models/MODEL_ID:generateContent" \
  -d '{"contents": [{"role": "user", "parts": [{"text": "Hello."}]}]}'
```

### 5.5 监控指标

通过 Cloud Monitoring 观察 PT 使用情况，资源类型为 `aiplatform.googleapis.com/PublisherModel`：


| 指标                           | 含义                |
| ---------------------------- | ----------------- |
| `/dedicated_gsu_limit`       | 你购买的 GSU 配额上限     |
| `/consumed_token_throughput` | 实际消耗的吞吐量（含配额调和计算） |
| `/tokens`                    | Token 数量分布        |
| `/token_count`               | 累计 token 使用量      |
| `/model_invocation_count`    | 预测请求数量            |
| `/first_token_latencies`     | 首 token 响应延迟      |


过滤维度：


| 维度             | 可选值                                  | 说明              |
| -------------- | ------------------------------------ | --------------- |
| `type`         | `input` / `output`                   | 区分输入和输出         |
| `request_type` | `dedicated` / `spillover` / `shared` | 区分 PT / 溢出 / 按量 |


> **注意**: 使用 Gemini 2.0 且显式启用缓存时，溢出请求可能不会标记为 `spillover`。

---

## 6. Live API 专项

> **来源**: [live-api](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/live-api)

### 6.1 什么是 Live API？

Gemini Live API 是一种支持低延迟多模态交互的会话式接口（支持音频/视频流），目前 PT 支持的模型为 **Gemini 2.5 Flash**（模型 ID: `gemini-live-2.5-flash-native-audio`，1,620 tok/s/GSU）。

### 6.2 会话流量隔离规则

**核心约束**: 一个 Live API 会话必须 **完全** 属于 PT 或 **完全** 属于 PayGo，**不支持会话内溢出（spillover）**。

```
会话 A: [PT] ──────────────────── [PT]  ✓ 合法
会话 B: [PayGo] ────────────────── [PayGo]  ✓ 合法
会话 C: [PT] ───── [PayGo] ──── [PT]  ✗ 不允许
```

系统在会话建立时检查用户 Header 和可用配额：

- 如果 PT 配额充足 → 整个会话走 PT
- 如果 PT 配额不足 → 整个会话降级为 PayGo

会话之间是顺序切换的 — 当前 PT 会话结束后，如果配额耗尽，下一个新会话可以溢出到 PayGo。

### 6.3 会话中的突发处理

如果 PT 会话 **中途** 超出配额，系统会允许 **临时突发**（burst）以维持会话连续性，而不是中断会话。但这些突发使用量仍会计入总配额，可能导致监控仪表板显示的使用量超过分配上限。

### 6.4 Session Memory 的 Token 消耗

Live API 会维护 **会话记忆**（session memory），这是一个关键的配额消耗因素：

```
请求 #1:
  音频输入:   250 tokens
  视频输入: 2,580 tokens
  合计输入: 2,830 tokens
  输出:       100 tokens
  → 消耗: 2,830 + 100 = 2,930 tokens

请求 #2:
  会话记忆: 2,830 tokens (前面所有上下文)  ← 这些也算！
  新输入:   1,000 tokens
  输出:       200 tokens
  → 消耗: 2,830 + 1,000 + 200 = 4,030 tokens

请求 #3:
  会话记忆: 3,830 tokens (持续增长)
  新输入:   ...
```

> **关键洞察**:
>
> - Session memory tokens 按 **1:1** 比率消耗配额（与标准输入 token 相同）
> - 音频输出 token 的 burndown rate 为 **24:1**（比标准文本输出更高）
> - 随着会话推进，每次请求的实际 token 消耗会 **持续增长**，因为会话记忆不断累积
> - 在估算 GSU 时务必考虑会话记忆的累积效应

### 6.5 容量规划建议

- 预留足够 GSU 以避免会话中途配额耗尽
- 利用历史 PayGo 流量模式作为参考数据来估算所需容量
- 监控仪表板可能显示 dedicated 吞吐量超出分配上限（突发导致）
- 如果 PT 配额耗尽，**新会话** 可以溢出到 PayGo，但已有 PT 会话不会切换

---

## 7. Veo 视频生成专项

> **来源**: [veo-models](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/veo-models)

### 7.1 特殊的配额执行周期

Veo 模型的配额执行窗口与文本/图像模型 **完全不同**，因为视频生成本身就是耗时操作：


| 购买 GSU 数    | 配额执行周期               |
| ----------- | -------------------- |
| 1 ~ 9 GSU   | **2,000 秒**（≈ 33 分钟） |
| 10 ~ 19 GSU | 400 秒                |
| 20 ~ 39 GSU | 200 秒                |
| 40 ~ 66 GSU | 100 秒                |
| 67+ GSU     | 60 秒                 |


> **注意**: 这些执行周期可能会变化（subject to change）。

### 7.2 执行周期 ≠ 处理延迟

这是一个容易混淆的概念：

```
示例: 1 GSU，生成 4 秒视频

实际处理: ████░░░░░░░░  (几分钟内完成)
执行窗口: ████████████████████████████████████  (2,000 秒)
                                              ↑
                                          下一个相同请求最早可提交的时间

处理很快就完成了，但你需要等到 2,000 秒窗口结束后，
才能再提交一个消耗同等配额的请求。
```

官方文档原文: "this isn't connected to the request latency. The time to process your request isn't the same as the quota enforcement period."

> **核心理解**: 执行窗口确保的是"你的请求一定会在这个时间范围内被处理"，而不是"处理需要这么久"。如果需要更高频率的视频生成，必须购买更多 GSU 来缩短执行窗口。

### 7.3 吞吐量换算

以 Veo 3（0.004 video sec/s/GSU）为例：

```
1 GSU 每秒可生成 0.004 秒视频
→ 生成 4 秒视频需要: 4 ÷ 0.004 = 1,000 秒的连续产出
→ 但实际处理并行化后会快得多

Veo 3 Fast (0.008) 效率是标准版的 2 倍
Veo 3.1 Fast (0.008) 同理
```

---

## 8. Single Zone Provisioned Throughput

> **来源**: [szpt](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/szpt)

### 8.1 什么是 Single Zone PT？

在某些 Google Cloud 区域，只有 **单个可用区** 具备 ML 处理能力。Standard PT 通常要求多可用区冗余，但 Single Zone PT 允许你在这些单可用区区域预留吞吐量。

### 8.2 与标准 PT 的对比


| 维度        | 标准 PT        | Single Zone PT                   |
| --------- | ------------ | -------------------------------- |
| 定价/GSU 结构 | ✓            | ✓ 相同                             |
| 多可用区冗余    | ✓            | ✗ 单可用区                           |
| SLA 保障    | ✓            | **✗ 无 SLA**（不属于 Covered Service） |
| 数据处理位置    | 区域内          | 严格在购买的区域内                        |
| 溢出处理      | PayGo（可能跨区域） | PayGo（**保证在区域内处理**）              |
| 批量请求      | 不支持          | 不支持                              |
| 微调        | 支持           | **不支持**                          |
| 购买方式      | Console 自助   | **需联系 Google Cloud 客户代表**        |


### 8.3 注意事项

- **无 SLA**: Single Zone PT 不是 "Covered Service"，不受 Gemini Online Inference SLA 保护
- **延迟风险**: 没有 ML 处理能力的区域可能比标准 PT 或 PayGo 延迟更高
- **溢出在区域内**: 与标准 PT 不同，SZPT 的溢出流量也保证在购买的区域内处理
- 适合对 **数据驻留** 有严格要求、且该区域只有单可用区的场景
- 可通过 REST API Header 控制溢出行为

---

## 9. 错误处理与重试策略

> **来源**: [error-code-429](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/error-code-429) & [retry-strategy](https://cloud.google.com/vertex-ai/generative-ai/docs/retry-strategy)

### 9.1 Error 429 的含义与处理

**429 = "Too Many Requests. Exceeded the Provisioned Throughput"**

#### 标准 PT 中的 429 行为

```
请求量 ≤ PT 容量:
  ├── 正常: 200 成功
  └── 偶尔故障: 变为 5XX（计入 SLA 错误率）← 注意这一点！

请求量 > PT 容量:
  ├── 默认: 溢出到 PayGo（不报错）
  ├── dedicated header: 返回 429
  └── 持续超出: 需要增加 GSU
```

#### Single Zone PT 中的特殊行为

429 错误会被转换为 5XX，但 **不计入 SLA**（因为 Single Zone PT 本身没有 SLA）。

#### 解决 429 的两种方案


| 方案     | 做法                                              | 适用场景    |
| ------ | ----------------------------------------------- | ------- |
| 允许按需溢出 | 不设置 `X-Vertex-AI-LLM-Request-Type` header（默认行为） | 优先保证可用性 |
| 增加容量   | 修改订单，增加 GSU 数量                                  | 溢出成本过高时 |


> **重要**: 对 PT 来说，频繁 429 意味着 **容量不足**，而不是暂时性过载。**重试不能解决根本问题**，需要增加 GSU 或允许溢出。

### 9.2 重试策略

#### SDK 内置重试

Google Gen AI SDK 默认包含自动重试逻辑：


| 参数                  | 默认值                          |
| ------------------- | ---------------------------- |
| `attempts`          | 5 次（Python SDK 为 4 次）        |
| `initial_delay`     | 1.0 秒                        |
| `exp_base`          | 2（指数退避基数）                    |
| `max_delay`         | 60 秒                         |
| `jitter`            | 1（随机抖动因子）                    |
| `http_status_codes` | 408, 429, 500, 502, 503, 504 |


#### 不同付费模式下的重试策略


| 付费模式                       | 推荐策略                       |
| -------------------------- | -------------------------- |
| Standard PayGo             | 指数退避重试 429                 |
| Flex PayGo                 | **不要激进重试**，改为增加超时到 30 分钟   |
| Priority PayGo             | 超额时指数退避                    |
| **Provisioned Throughput** | **频繁 429 = 需要加 GSU，重试无意义** |


#### 可重试 vs 不可重试


| 可重试                   | 不可重试                      |
| --------------------- | ------------------------- |
| 408 Request Timeout   | 400 Bad Request           |
| 429 Too Many Requests | 401 Unauthorized          |
| 5XX Server Error      | 499 Client Closed Request |
| 网络超时 / TCP 断连         | 其他永久性 4XX                 |


#### Python 自定义重试配置

**客户端级别**:

```python
from google import genai
from google.genai import types

client = genai.Client(
    vertexai=True,
    project="my-project",
    location="global",  # 全局端点，自动路由到最优区域
    http_options=types.HttpOptions(
        retry_options=types.HttpRetryOptions(
            initial_delay=1.0,
            attempts=5,
            http_status_codes=[408, 429, 500, 502, 503, 504],
        ),
        timeout=120 * 1000,  # 120 秒超时
    ),
)
```

**请求级别**（覆盖客户端配置）:

```python
response = client.models.generate_content(
    model="gemini-3-flash-preview",
    contents="Tell me a joke about a rabbit.",
    config=types.GenerateContentConfig(
        http_options=types.HttpOptions(
            retry_options=types.HttpRetryOptions(
                initial_delay=1.0,
                attempts=10,
                http_status_codes=[408, 429, 500, 502, 503, 504],
            ),
            timeout=120 * 1000,
        )
    ),
)
```

**Flex PayGo 专用配置**:

```python
client = genai.Client(
    vertexai=True,
    project="my-project",
    location="global",
    http_options=types.HttpOptions(
        api_version="v1",
        headers={
            "X-Vertex-AI-LLM-Request-Type": "shared",
            "X-Vertex-AI-LLM-Shared-Request-Type": "flex",
        },
        timeout=30 * 60 * 1000,  # 30 分钟超时
    ),
)
```

#### Java 自定义重试配置

```java
import com.google.genai.Client;
import com.google.genai.types.HttpOptions;
import com.google.genai.types.HttpRetryOptions;

HttpOptions httpOptions = HttpOptions.builder()
    .retryOptions(
        HttpRetryOptions.builder()
            .attempts(5)
            .httpStatusCodes(408, 429, 500, 502, 503, 504)
            .build())
    .build();

Client client = Client.builder()
    .project("my-project")
    .location("global")
    .vertexAI(true)
    .httpOptions(httpOptions)
    .build();
```

### 9.3 重试最佳实践 & 反模式

#### 最佳实践

- 使用 **指数退避 + 随机抖动**（避免 thundering herd，即大量客户端同时重试）
- 设置 **最大重试次数**（防止无限循环）
- 只重试 **特定错误码**（瞬态错误）
- **监控和记录** 重试行为（帮助判断是否需要扩容）
- 尽量使用 **全局端点**（global endpoint）— 自动路由到容量最充裕的区域

#### 反模式（务必避免）


| 反模式                     | 后果                       |
| ----------------------- | ------------------------ |
| 立即重试，无退避                | 加剧过载，形成级联故障              |
| 重试 4XX 客户端错误（除 408/429） | 永远不会成功，浪费资源              |
| 无限重试，不设上限               | 用户无限等待，资源浪费              |
| 盲目重试非幂等操作               | 可能导致重复创建资源               |
| 批量任务中逐条重试               | 应使用批量服务（Batch Service）代替 |


### 9.4 幂等性考量


| 操作类型                  | 幂等性                  | 可否安全重试      |
| --------------------- | -------------------- | ----------- |
| List / Get 操作         | 始终幂等                 | 安全          |
| Token 计数 / Embeddings | 始终幂等                 | 安全          |
| `generateContent`     | 非严格幂等（随机性），但不修改服务端状态 | **一般安全**    |
| 创建唯一资源（如微调模型）         | 不幂等                  | **不安全，需谨慎** |


---

## 10. 最佳实践与决策清单

### 10.1 何时选择 PT？

```
✓ 生产环境，用户体验敏感
✓ 日均吞吐量高且稳定（不是偶尔的突发）
✓ 需要明确的成本预算
✓ 需要 SLA 保障

✗ 实验/开发阶段 → 用 PayGo
✗ 流量波动极大 → 用 PayGo 或 Flex
✗ 偶尔使用 → 不划算
```

### 10.2 容量规划清单

1. **确定模型和版本** — 记住必须用精确版本 ID
2. **估算 QPS** — 峰值 QPS，而非平均值
3. **统计 token 分布** — 输入/输出的平均 token 数、媒体类型
4. **计算 burndown rate** — 不同输入类型的换算系数
5. **评估缓存命中率** — Context Caching 可节省 90% 的重复输入消耗
6. **使用在线估算工具** — 购买页面的估算器验证手算结果
7. **预留 buffer** — 建议在计算结果基础上多买 10~20% 应对峰值
8. **Live API 额外考虑** — 会话记忆会累积消耗，需预留更多余量

### 10.3 运维监控清单


| 关注指标                                              | 告警条件              |
| ------------------------------------------------- | ----------------- |
| `consumed_token_throughput / dedicated_gsu_limit` | > 80% 持续 5 分钟     |
| `spillover` 请求比例                                  | > 10%（说明 PT 容量不足） |
| 429 错误率（dedicated 模式）                             | 任何持续出现            |
| `first_token_latencies` P99                       | 超出业务 SLA          |


### 10.4 成本优化技巧

1. **善用 Context Caching** — burndown rate 降到 0.1×，对长上下文场景效果显著
2. **合理选择期限** — 长期订阅通常有折扣，但注意不可取消
3. **默认允许溢出** — 除非严格控制成本，否则让超额请求走 PayGo 比被 429 拒绝更好
4. **全局端点** — 配合 PT 使用全局端点可以获得更好的可用性
5. **监控 spillover 比例** — 如果溢出持续发生，考虑增加 GSU 比按量付溢出费更划算
6. **请求级别重试配置** — 关键请求可设置更高的重试次数，非关键请求可降低

### 10.5 多 Region 高可用架构

#### 为什么 GSU 溢出到 PayGo 还不够？

GSU 满了以后默认会自动溢出到同 region 的 PayGo，但 **PayGo 本身也有容量上限**：

- PayGo 是多租户共享池，高峰期该 region 的 PayGo 容量也可能紧张，溢出请求照样会收到 **429 错误**
- PayGo 配额（QPM/TPM）是 **per-project per-region** 的，单个 region 的配额有天花板

因此，仅依赖"同 region GSU + 自动溢出"在极端场景下仍存在不可用风险。

#### 推荐做法：双 Region 架构

在同一个 project 下设置两个 region，一个购买 GSU，另一个作为纯 PayGo 后备：

```text
请求进来
  │
  ▼
Region A (如 europe-north1) — 有 GSU
  ├── GSU 未满 → PT 处理 ✅ (低成本、有保障)
  ├── GSU 满了 → 自动溢出到同 region PayGo
  │     ├── PayGo 有容量 → 处理 ✅ (按量计费)
  │     └── PayGo 也限流 → 429 ❌
  │
  ▼ (429 时 failover)
Region B (如 us-central1) — 纯 PayGo
  └── 独立的 PayGo 容量池 → 处理 ✅
```

#### 方案对比

| 方案 | 优点 | 风险 |
| --- | --- | --- |
| 单 region（GSU + 自动溢出） | 架构简单，溢出自动处理 | PayGo 容量也有上限，高峰期可能全部 429 |
| 双 region（GSU region + 纯 PayGo region） | 双容量池，可用性更高 | 跨 region 延迟略高，需客户端做路由/failover |

#### 实现要点

- 客户端需要实现 **failover 逻辑**：当 Region A 返回 429 时，将请求重试到 Region B
- Region B 不需要购买 GSU，只使用 PayGo，因此没有固定成本
- 两个 region 的 PayGo 配额相互独立，相当于总可用配额翻倍
- 结合 [第9节](#9-错误处理与重试策略) 的重试策略，可在重试逻辑中加入跨 region failover

---

> **参考链接**:
>
> - [Provisioned Throughput Overview](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/overview)
> - [Supported Models](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/supported-models)
> - [Calculate Requirements](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/measure-provisioned-throughput)
> - [Purchase PT](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/purchase-provisioned-throughput)
> - [Use PT](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/use-provisioned-throughput)
> - [Live API](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/live-api)
> - [Veo Models](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/veo-models)
> - [Single Zone PT](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/szpt)
> - [Error Code 429](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput/error-code-429)
> - [Retry Strategy](https://cloud.google.com/vertex-ai/generative-ai/docs/retry-strategy)

