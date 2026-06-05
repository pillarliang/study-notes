# Client / Gateway / Model 三层 timeout

调用 LLM provider 时,**至少有 3 层 timeout 在同时计时**。理解这个分层是诊断 504 / 499 / ReadTimeout 的钥匙。

## 三层结构

```
我们的 client ──(请求)──> Provider Gateway ──(转发)──> Model backend
   ↑ client timeout         ↑ gateway timeout          ↑ model 处理
   我们设的,可改             provider 设的,改不了        实际算力
```

以 Vertex AI gemini-2.5-flash 为例:

```
我们的 client ──(请求)──> Vertex Gateway ──(转发)──> gemini-2.5-flash backend
   200s timeout              ~60-90s gateway timeout      model 处理
   (我们设的)                 (Google 设的,我们改不了)
```

## 三种超时场景

| 谁先到时间 | 现象 | 我们 client 看到 | Provider 看到 |
|---|---|---|---|
| **Gateway 先超时** | model 处理慢,但 Gateway 等不及 | **504 Gateway Timeout** | (它自己回的) |
| **我们 client 先超时** | Gateway 还在等,我们已经砍连接 | **APITimeoutError / ReadTimeout** | **499 Client Closed** |
| **model 正常返回** | 3 层都没到 timeout | 正常 response | 正常 |

关键洞察:

- **504 ≠ 我们 timeout 太短**。反了——504 是 provider 嫌 model 慢主动回的,跟我们的 client timeout 无关。我们改 200s → 300s 也救不了 504。
- **499 是同一件事的镜像**——provider 看到 client 主动关闭连接,所以记 499。在我们这边就表现为 ReadTimeout / APITimeoutError。
- 想减少 504,只能**换 region / 换 model / 让 prompt 短一点**(减少 model 处理时间)。
- 想减少 499,可以**加长 client timeout**——但代价是真挂的请求等更久。

## 实战案例(2026-05-25 17:30 UTC+8)

Google Cloud 控制台看到 gemini-2.5-flash:
- 504 峰值 1.2%
- 499 峰值 0.95%
- 429 0.35%

我们服务日志看到同一时段:
- 5× ReadTimeout (sc=None)
- 2× ClientError 499 CANCELLED

映射关系:

| Vertex 上看到 | 我们日志看到 | 机制 |
|---|---|---|
| 504(网关超时) | 看不到 | 我们 200s 提前砍了,Vertex 504 还没回来 |
| 499 | ReadTimeout(我们 client 砍的) | 同一件事两端 |
| 499 | ClientError 499 CANCELLED | Vertex 真的下发了 CANCELLED 状态码 |

## 关联

- 状态码细节 → [[error-status-codes]]
- 怎么定 client timeout → [[timeout-tuning]]

#llm #timeout #troubleshooting
