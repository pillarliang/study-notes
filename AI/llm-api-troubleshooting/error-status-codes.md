# LLM Provider 错误状态码速查

调用 OpenAI / Azure OpenAI / Vertex AI / Anthropic 时常见的错误类型与状态码,各自含义和触发场景。

## 总览

| 错误 | status_code | 含义 | 谁的责任 | 该怎么办 |
|---|---|---|---|---|
| **APITimeoutError** | None | 客户端 timeout 触发,根本没收到响应 | 我们 client | 检查 client timeout 设置 + 看 provider 健康 |
| **ReadTimeout** | None | httpx 等底层库的读超时 | 我们 client | 同上 |
| **499 ClientError** | 499 | Vertex AI 标记的"客户端关闭连接" | 我们 client(主动 abort) | 同上,跟 APITimeoutError 通常是一对 |
| **504 Gateway Timeout** | 504 | Provider 自家网关嫌 model 慢 | provider 后端 | 换 region / 换 model / 缩短 prompt |
| **503 Service Unavailable** | 503 | Provider 后端服务整体不可用 | provider | 等 / fallback |
| **502 Bad Gateway** | 502 | Provider 网关收到坏响应 | provider | 重试 / fallback |
| **429 Too Many Requests** | 429 | 限流 / quota 用完 | 我们调用太频繁 / quota 不够 | 退避重试 / 申请 quota |
| **400 BadRequestError** | 400 | 请求本身有问题 | 我们的请求 | **不要重试**,排查参数 |
| **401 Unauthorized** | 401 | 认证失败 | 我们的 key | 检查 API key |
| **413 Payload Too Large** | 413 | 请求体超大 | 我们 prompt 过长 | 分段 / 压缩 |

## 关键区分

### APITimeoutError / ReadTimeout / 499 是一回事

三者描述的是**同一个事件的不同视角**:

```
我们 client 200s 到 → abort 连接
   ↓
我们 SDK 抛 APITimeoutError(OpenAI SDK)
   或 ReadTimeout(httpx 底层)
   或 ClientError 499(Vertex SDK,因为 Vertex 在返回里写了 499)
   ↓
Provider 在自己监控里记 499 Client Closed
```

诊断信号: `status_code=None` + `original_error=APITimeoutError/ReadTimeout` = **我们 client 主动砍的**,不是 provider 拒绝。

### 504 是 Provider 自己的事,不是我们的事

详见 [[timeout-layers]]。504 出现时:
- 我们 client 还在等(没到 client timeout)
- Provider 网关已经等不及 model → 主动回 504
- 改我们的 client timeout **没有任何作用**

### 400 BadRequest 单独处理

400 跟 timeout 完全是两类问题:
- timeout 类:**可重试**(transient),fallback 也能救
- 400:**不要重试**(永久错误,重试一万次还是 400)

常见 400 触发因素:
- prompt 超过 model context window
- 参数格式错误(比如 reasoning 模型传 temperature)
- 内容触发 content filter
- 工具调用格式不合法

诊断 400 必须捞**完整请求体**,不是看 timeout 那套。

## 各 Provider 的错误命名差异

| 概念 | OpenAI SDK | httpx | Anthropic SDK | Google genai SDK |
|---|---|---|---|---|
| Client 超时 | `APITimeoutError` | `ReadTimeout` | `APITimeoutError` | `ReadTimeout` 或 `ClientError 499` |
| 限流 | `RateLimitError` (429) | - | `RateLimitError` (429) | `ClientError` (429) |
| 服务端错误 | `APIError` (5xx) | - | `APIStatusError` | `ServerError` |
| 请求错误 | `BadRequestError` (400) | - | `BadRequestError` (400) | `ClientError` (400) |

## 实战:从一行错误日志判断根因

```text
error_type=ProviderError, status_code=None, attempt=0,
original_error=APITimeoutError('Request timed out.'), error=Request timed out.
```

读这行:
- `status_code=None` → 没收到响应 → client 超时(不是 5xx)
- `original_error=APITimeoutError` → OpenAI SDK 抛的 client timeout
- `attempt=0` → 没重试就 fallback 了
- → 结论: **我们 client 等不及 abort**,需要检查 provider 那段时间健康度,以及是否该加重试

```text
status_code=499, original_error=ClientError("499 CANCELLED. ...")
```

- `status_code=499` + Vertex 的 `ClientError` → 同样是 client 砍的,只是 Vertex SDK 把 499 显式带出来了

```text
status_code=400, original_error=BadRequestError(...)
```

- 永久错误,捞完整请求体调试,**不要走 fallback 重试逻辑**

## 关联

- 三层 timeout 模型 → [[timeout-layers]]
- 出错后 fallback 怎么走 → [[fallback-design]]

#llm #status-codes #troubleshooting
