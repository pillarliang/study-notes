# LLM client timeout 怎么定

设置 LLM API 调用的 client timeout 时,常见误区是"出过 timeout 就加大"。本文讲怎么基于数据决策。

## 核心认知

**client timeout 决定的不是"model 能不能完成",而是"用户多快进入 fallback"。**

```
provider 真挂时:
  timeout=600s → 等 10min 失败 → fallback 4min → 用户总等 14min
  timeout=180s → 等 3min 失败 → fallback 4min → 用户总等 7min
最终都用 fallback 完成,但用户体验天差地别。
```

> 等待时间是**无收益**的——provider 不会突然在第 9 分钟好起来。

## 设置原则

### 原则 1:基于 p99 latency,不要拍脑袋

| 正常 p99 | timeout 设到 | buffer 倍数 |
|---|---|---|
| 30s | 90s | 3× |
| 60s | 180s | 3× |
| 120s | 300s | 2.5× |
| 300s | 600s | 2× |

经验:**2.5~3 倍 p99** 是合理的。再大就是浪费用户时间。

获取 p99 的方法:
- 拉**正常时段**(过滤掉故障期)成功请求的 latency 分布
- 监控系统直接看 model_request_duration_p99 metric

### 原则 2:区分 timeout 错误的分布

| 现象 | 含义 | 该做的 |
|---|---|---|
| timeout 集中在**某时段** | provider 抖动,正常时段没事 | **不要加 timeout**,加 fallback / 熔断 |
| timeout **均匀分布**全天 | timeout 设短了,常态 p99 接近 timeout | **加大 timeout** 或换更快的 model |
| timeout 随 input 长度增长 | 长 prompt 触顶 timeout | 走**动态 timeout** 或分段处理 |

### 原则 3:reasoning model 单独处理

reasoning 模型(gpt-5 系、o1 系)的处理时间 = reasoning tokens + output tokens,**比普通 model 慢 5-10 倍**。

```yaml
gpt-4o:            timeout 60-90s    # 普通
claude-sonnet-4.5: timeout 120-180s  # 中等
gpt-5.5 (effort=low):  timeout 180-300s  # reasoning
gpt-5.5 (effort=high): timeout 600s+      # 重 reasoning
```

reasoning effort 是关键变量,effort=high 时 timeout 可能要翻 3-5 倍。

## 进阶:按 token 数动态调整

固定 timeout 对所有请求一刀切。更聪明的做法是按 input token 数动态算:

```python
def calc_timeout_ms(input_tokens: int, model: str) -> int:
    if model.startswith("gpt-5"):
        # gpt-5 系 reasoning 模型
        base_ms = 60_000               # 60s 基础
        per_1k_tokens = 4_000          # 每 1K input 加 4s(经验值,需校准)
        cap_ms = 300_000               # 5min 封顶
        return min(base_ms + input_tokens * per_1k_tokens // 1000, cap_ms)
    elif model.startswith("claude"):
        base_ms = 30_000
        per_1k_tokens = 2_000
        cap_ms = 180_000
        return min(base_ms + input_tokens * per_1k_tokens // 1000, cap_ms)
```

收益:

| 输入大小 | 动态 timeout | 对比固定 600s |
|---|---|---|
| 5K tokens(短) | 80s | 故障时**快速止血** |
| 25K tokens(中) | 160s | 仍有合理 buffer |
| 100K tokens(长) | 5min(封顶) | 触发分段处理 |

## 配套机制

光改 timeout 不够,以下三件套要一起做:

### 1. 有限重试

```yaml
retry:
  max_attempts: 2          # 不要无限重试,会雪崩
  backoff_ms: 500          # 指数退避基数
  retry_on:                # 只对 transient 重试
    - timeout
    - 502
    - 503
    - 504
  # 注意 ❌ 不要 retry 400 / 401 / 内容过滤
```

短 timeout + 1 次重试 通常优于 长 timeout 0 重试——单次随机慢可以救回来,且用户感知延迟更短。

### 2. Endpoint 熔断

短时间高频失败 → 主动熔断该 endpoint K 秒,直接走 fallback。详见 [[fallback-design]]。

### 3. Metrics 完备

最低要记:
- `actual_duration_ms`:请求实际花了多久(反算合理 timeout)
- `timeout_ms`:实际生效的 timeout(诊断是否设错)
- `attempt`:第几次尝试(看重试是否生效)
- `endpoint_consecutive_failures`:连续失败次数(熔断决策依据)

## 实战参考(2026-05-25 plaud-summary)

观察到的 timeout 集中在 6 分钟窗口,前后正常时段成功率 100%:

| Model | 失败 trace input tokens | 当时 timeout | 判断 |
|---|---|---|---|
| gpt-5.5 | 12K / 25K / 20K | 600s | 不是长度问题(12K 都挂),是 Sweden 抖动 |

→ **加 timeout 救不了**,该缩短(600s→180s)+ 改 fallback 跨 provider。

## 关联

- 三层 timeout 原理 → [[timeout-layers]]
- 错误识别 → [[error-status-codes]]
- Fallback 设计 → [[fallback-design]]

#llm #timeout #tuning
