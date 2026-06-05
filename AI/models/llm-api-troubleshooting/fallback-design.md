# Fallback 链设计原则

LLM provider 抖动是常态,fallback 是必备。但**配错的 fallback 在故障时等于没有**。

## 一句话

> Fallback 的价值不在"配上了",而在**故障传染**到 fallback 目标的**概率有多低**。

## 三个反模式

### 反模式 1:同 region fallback

```yaml
# ❌ 反例
summary:gpt-5.5:
  provider: azure-sweden-central
summary:gpt-5:
  provider: azure-sweden-central   # 同区域
fallback: gpt-5.5 → gpt-5
```

Azure Sweden Central 整体抖动时,**两个 endpoint 一起挂**。fallback 触发 → fallback 也 timeout → 用户等了两倍时间最后还是失败。

实战 2026-05-25:Sweden Central 那段时间 gpt-5.5(3 次)+ gpt-5(2 次)同时 APITimeout。

### 反模式 2:同 provider fallback

```yaml
# ❌ 反例
claude-sonnet-4.6 → claude-sonnet-4.5
# 都走 Anthropic Vertex
```

Anthropic 后端故障 / Vertex 网关故障 → 4.6 和 4.5 都不通。实战见过 trace `20260525084601` 两步都 timeout。

### 反模式 3:循环 fallback

```yaml
# ❌ 反例
summary:gemini-2.5-flash:
  fallback_model: [summary:gemini-3-flash]
summary:gemini-3-flash:
  fallback_model: [summary:gemini-2.5-flash]   # 兜回去了
```

Vertex 整体不可用时,policy 在两者之间打转,等于没有 fallback。

## 正确做法

### 原则 1:fallback 至少跨 1 个 provider

```yaml
# ✅ 推荐
gpt-5.5 → gpt-5 → claude-sonnet-4.5
# Azure → Azure → Anthropic Vertex(换 provider)
```

```yaml
# ✅ 推荐
gemini-2.5-flash → gpt-5
# Vertex → Azure(换 provider)
```

故障基本不会同时打到两个 provider。

### 原则 2:同一 model 配多 region endpoint

```yaml
# ✅
summary:gpt-5.5:
  endpoints:
    - id: ep-gpt-5-5-sweden
      provider: azure-sweden-central
      weight: 70
    - id: ep-gpt-5-5-eastus
      provider: azure-east-us
      weight: 30
```

Sweden 出问题,流量自动切到 East US。比 fallback 到另一个 model 更优,因为**质量不变**。

### 原则 3:fallback 链显式终结

```yaml
# ✅ 末端兜底到稳定的通用 model
gpt-5.5 → gpt-5 → claude-sonnet-4.5 → gpt-4o-mini  # 最后一步用最稳定的
```

避免末端进入循环,也保证最差有个能跑的兜底。

## Fallback 触发的成本

每一次 fallback 都是有代价的:

| 代价类型 | 说明 |
|---|---|
| 用户感知延迟 | 等原 endpoint timeout → 才进入 fallback,延迟叠加 |
| 质量下降 | fallback model 通常能力不同,输出质量可能不一致 |
| Fallback 雪崩 | fallback model 流量突增,可能被打挂 |
| 成本变化 | fallback model 单价可能更贵或更便宜 |

所以**减少 fallback 触发** > **依赖 fallback 兜底**。优化顺序:

1. 选稳定的主 endpoint(高可用 region / 配多 region 备份)
2. 加合理重试(短 timeout + 1 次重试 > 长 timeout 0 重试)
3. 跨 provider 的 fallback 作为最后防线

## 熔断:让 fallback 提前生效

如果一个 endpoint 在短时间内连续失败 N 次,**主动熔断**该 endpoint K 秒,后续请求直接打到 fallback。

收益: 避免"每次请求都先去试这个明显挂掉的 endpoint,等 200s 才降级"。用户感知延迟从分钟级降到秒级。

## 关联

- 状态码识别 → [[error-status-codes]]
- timeout 设置 → [[timeout-tuning]]

#llm #fallback #reliability
