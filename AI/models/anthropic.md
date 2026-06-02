# Anthropic

## Claude on AWS Bedrock:endpoint、SDK、认证三件事

> 参考:[AWS Bedrock endpoints 官方文档](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints.html) · [Bedrock API Keys](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys.html)

### 1. Bedrock 有两个 inference endpoint


| Endpoint                   | 域名                                       | 支持 API                                                       | 跨区推理 profile | 用途定位                                                    |
| -------------------------- | ---------------------------------------- | ------------------------------------------------------------ | ------------ | ------------------------------------------------------- |
| `bedrock-runtime`(经典)      | `bedrock-runtime.{region}.amazonaws.com` | InvokeModel / Converse / Chat Completions / **Messages API** | ✅            | 原生 AWS API、需要 `us.` / `global.` 跨区 profile              |
| `bedrock-mantle`(新,AWS 主推) | `bedrock-mantle.{region}.api.aws`        | Responses / Chat Completions / **Messages API**              | ❌            | OpenAI SDK 零迁移、server-side tool use、Projects/Workspaces |


**两个 endpoint 都同时支持 SigV4 和 Bearer Token 认证。**

### 2. 4 种 SDK 调用方式


| SDK                              | 命中 endpoint       | base_url | key                                  | 协议                      | 何时选                                                          |
| -------------------------------- | ----------------- | -------- | ------------------------------------ | ----------------------- | ------------------------------------------------------------ |
| `anthropic[bedrock]`             | `bedrock-runtime` | 自动       | 可选(传则 Bearer,不传 SigV4)               | Anthropic Messages 原生   | 要 Claude 全特性(cache_control / extended thinking / 跨区 profile) |
| `boto3` `bedrock-runtime` client | `bedrock-runtime` | 自动       | env `AWS_BEARER_TOKEN_BEDROCK` 或 IAM | AWS Converse(字段不同)      | 已有 boto3 代码,不需要 Anthropic 原生字段                               |
| `openai` SDK                     | `bedrock-mantle`  | **必传**   | **必传**(Bearer)                       | OpenAI Chat Completions | OpenAI 代码零迁移,接受失去跨区 profile                                  |
| `requests` / `curl`              | 任一                | 自拼       | 自塞 `Authorization` header            | 看打哪个                    | 极简、跨语言                                                       |


### 3. 5 种认证方式(都适用于上面任意 SDK)


| #   | 方式                                                                 | 怎么传                                                           | 寿命           | 定位                 |
| --- | ------------------------------------------------------------------ | ------------------------------------------------------------- | ------------ | ------------------ |
| 1   | **IAM Role**(IRSA / EC2 Instance Profile / ECS Task Role / Lambda) | 什么都不传,boto3 默认凭证链兜底                                           | 15min 自动刷新   | **生产首选**           |
| 2   | **AK / SK**                                                        | `aws_access_key` + `aws_secret_key`(+ 可选 `aws_session_token`) | 长期或 STS 临时   | 本地兜底 / 临时凭证        |
| 3   | **AWS Profile**                                                    | `aws_profile="xxx"` 走 `~/.aws/credentials`                    | 看 profile 类型 | 本地多账号              |
| 4   | **Short-term Bedrock API Key**(Bearer)                             | `api_key=` 或 env `AWS_BEARER_TOKEN_BEDROCK`                   | ≤ 12h,自动刷新   | **AWS 官方生产推荐**     |
| 5   | **Long-term Bedrock API Key**(Bearer)                              | 同上                                                            | 自定义到期日       | 仅 exploration / 本地 |


**关键认知**:

- Bearer Token 模式**完全绕过 SigV4**,直接发 `Authorization: Bearer ...` 头。
- 但 IAM **不被绕过** —— 调用方仍需有 `bedrock:CallWithBearerToken` 权限,可用 condition key `bedrock:bearerTokenType` 区分 `SHORT_TERM` / `LONG_TERM`。
- Short-term key **继承生成时的 IAM principal 权限**,所以是"用 IAM 但用 Bearer 传输"的混合体。
- Long-term key **会创建独立 IAM User**,所以才"不推荐生产用"(长期凭证轮换难)。
- `AWS_BEARER_TOKEN_BEDROCK` 是 AWS 全局约定,boto3 / anthropic SDK / AWS CLI 都认。
- anthropic SDK 里 `api_key` 与 AWS 凭证 kwargs **互斥**,同时传会 `ValueError`。

### 4. `AnthropicBedrock` SDK 底层做了什么

```
你的代码:  client.messages.create(model=..., messages=..., cache_control=...,
                                  output_config=..., thinking=...)
              ↓
HTTP 层:   POST https://bedrock-runtime.{region}.amazonaws.com
                /model/{model_id}/invoke           ← 走 InvokeModel,不是 Converse
           Authorization: AWS4-HMAC-SHA256 ...    (SigV4 模式)
                       或: Bearer <api-key>         (Bearer 模式)
           Body: 与 SaaS 完全同一份 Anthropic Messages JSON
                 + 多塞 "anthropic_version": "bedrock-2023-05-31"
              ↓
AWS:       Bedrock Runtime → 路由 → Anthropic 原生响应 JSON
流式:      AWS eventstream 帧 → SDK 还原成 Anthropic 原生 SSE chunk 透传
```

**核心要点:wire format 是 Anthropic Messages 原生 JSON,不是 AWS 自定义的 Converse 格式。** 这就是为什么 SaaS / Vertex / Bedrock 三条路径能共享同一份业务逻辑——它们只在 endpoint + 认证签名上有差异。

### 5. 跨区推理 profile:`us.` vs `global.`


| 维度          | `us.anthropic.claude-...`                    | `global.anthropic.claude-...` |
| ----------- | -------------------------------------------- | ----------------------------- |
| 路由范围        | 仅 US 区域(us-east-1 / us-east-2 / us-west-2 等) | 全球所有可用区域                      |
| 数据驻留        | 保证留美国                                        | **不保证**,可能落 EU/APAC           |
| 容量池         | US 容量                                        | 全球容量,429 概率更低                 |
| 延迟 p99      | 稳定                                           | 命中远端区域时显著上升                   |
| 合规          | 满足美国数据驻留                                     | **不满足**                       |
| 适用 endpoint | 仅 `bedrock-runtime`                          | 仅 `bedrock-runtime`           |


**默认用 `us.`,只在 US 容量持续紧张或需要全球容灾时再考虑 `global.`。**

IAM policy 必须覆盖 profile 实际路由到的所有 region 的 `foundation-model/anthropic.claude-`*:

```json
"Resource": [
  "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",
  "arn:aws:bedrock:*::inference-profile/us.anthropic.claude-*"
]
```

### 6. 决策树:怎么选?

```
要调 Bedrock 上的 Claude
│
├─ 用 Claude 全特性(cache_control / extended thinking / 跨区 profile)?
│   ├─ 是 → anthropic SDK + bedrock-runtime
│   └─ 否 ↓
│
├─ 已有 OpenAI 代码想零改动迁过来?
│   ├─ 是 → openai SDK + bedrock-mantle(注意失去跨区 profile)
│   └─ 否 ↓
│
├─ 已有 boto3 / Converse 代码?
│   └─ 是 → boto3,Bearer Token 通过 env 注入
│
└─ 极简 / 跨语言 → raw HTTPS + Bearer
```

**生产部署默认组合**:`anthropic SDK + bedrock-runtime + IRSA(IAM Role)` 或 `+ short-term API key`。

### 7. 三条 Anthropic 路径横向对比(SaaS / Vertex / Bedrock)


| 维度                          | SaaS                            | Vertex                                              | Bedrock                                             |
| --------------------------- | ------------------------------- | --------------------------------------------------- | --------------------------------------------------- |
| Endpoint                    | `api.anthropic.com/v1/messages` | `{region}-aiplatform.googleapis.com/.../rawPredict` | `bedrock-runtime.{region}.amazonaws.com/.../invoke` |
| 认证                          | `x-api-key` header              | OAuth2 Bearer(GCP SA)                               | SigV4 **或** Bearer Token                            |
| `anthropic_version` body 字段 | 默认                              | `vertex-...`                                        | `bedrock-2023-05-31`                                |
| Body schema                 | Anthropic Messages 原生           | **同左**                                              | **同左**                                              |
| SDK 类                       | `Anthropic`                     | `AnthropicVertex`                                   | `AnthropicBedrock`                                  |


**唯一真正不同的**:endpoint 拼接、认证签名、`anthropic_version` 字段值。其余 100% 对称,这是 Hub 三条路径能共享同一份 `AnthropicProvider` 业务逻辑的物理基础。