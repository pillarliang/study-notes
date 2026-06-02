# 工作报告 - 2026-05-28

## 📅 2026-05-21 ~ 2026-05-28 工作摘要

本周工作聚焦三条主线：

1. **结构化输出兼容性收敛**：在 plaud-model-hub 完成 ADR-017/018/021 三项相关改造，统一不同 Provider（OpenAI Responses、Anthropic、Azure、GenAI）在 `json_schema` / `function_calling` / 多模态场景下的行为，对外屏蔽各家 API 差异。
2. **CN 模型链路上线**：新增 `PlaudModerationPlugin`（ADR-020）补齐国内 Provider 的合规链路，并通过 `feature/cn_model_hub_v1/v2/v3` 多轮灰度，把 CN model hub 完整推到主环境。
3. **可观测性升级**：plaud-summary 接入 model-hub SDK 的 OTelPlugin 指标，并把 Hub fallback 指标迁移到 `FallbackLifecycleHook`（ADR-016），统一回退（fallback）路径的埋点入口。

## 🚀 主要进展 (Key Achievements)

- **结构化输出 & 多模态**：
  - 实现 capability-aware 的 `json_schema → function_calling` 自动回退（ADR-021），消费方无需感知 Provider 是否支持 json_schema。
  - Anthropic Provider 落地 synthetic tool 翻译，覆盖 `json_schema` 形态的 `response_format`，并对异常 `tool_calls` 增加告警。
  - Responses API 扁平化 `json_schema` 结构，新增 `_inline_json_schema_refs` 解决 `$ref` 递归引用问题。
  - Azure / openai-compatible 通道支持把 image_url 内联为 base64，打通 Responses API 上的多模态输入（ADR-017）。
- **CN Provider 合规与灰度**：
  - 落地 `PlaudModerationPlugin`（ADR-020），统一接入国内 Provider 的内容安全检查。
  - 通过三轮迭代（cn_model_hub_v1 → v2 → v3）将 CN model hub 灰度到主环境，覆盖 ap-northeast-1、us-west-2、CN 多区域配置。
- **可观测性 & 稳定性**：
  - plaud-summary 启用 model-hub SDK 的 `OTelPlugin`，向链路新增 SDK 级别指标。
  - 将 Hub fallback 指标迁移到 `FallbackLifecycleHook`（ADR-016），收敛 fallback 埋点入口。
  - 对 image_url 加载失败显式标记为 400，避免无意义的重试与熔断（circuit break）。
  - `with_structured_output` 的 `function_calling` 默认值对齐 `BaseChatOpenAI`（ADR-018）。
- **维度笔记数据模型**：plaud-summary 中将 `KeyDataPoint` / `Classified` / `Dedup` 的 `unit` 字段改为可空，并同步 bump model-hub SDK（ADR-018）。

## 📝 详细提交记录 (Git Log Summary)

### plaud-model-hub

- **结构化输出与回退**
  - `feat(structured-output)`: capability-aware json_schema → function_calling fallback（ADR-021）
  - `fix(anthropic)`: implement synthetic tool translation for json_schema response_format
  - `fix(anthropic)`: warn when synthetic tool path returns anomalous tool_calls
  - `fix(responses)`: flatten json_schema structure for Responses API compatibility
  - `feat(genai)`: implement `_inline_json_schema_refs` for handling `$ref` in JSON schemas
  - `fix(genai)`: update stack handling in `_walk` for improved reference resolution
  - `fix(langchain)`: align `with_structured_output` function_calling defaults with `BaseChatOpenAI`（ADR-018）
- **多模态与 Provider 适配**
  - `feat(openai-compatible)`: support multimodal on Responses API + Azure image inline（ADR-017）
  - `feat(openai-compatible)`: refactor image URL handling to inline base64 conversion
  - `feat(plugins)`: introduce `PlaudModerationPlugin` for CN providers（ADR-020）
  - `fix(providers)`: mark `image_url` load failure as 400 to skip retry and circuit break
- **工程清理**
  - `fix(chat_model)`: improve model reference splitting for clarity and maintainability
  - `docs`: address PlaudModerationPlugin review notes / structured output PR checklist / ADR-021 changelog

### plaud-summary

- `feat(hub)`: enable model-hub SDK OTelPlugin metrics
- `feat`: migrate Hub fallback metric to `FallbackLifecycleHook`（ADR-016）
- `fix(dimension-notes)`: make `KeyDataPoint` / `Classified` / `Dedup` unit nullable（ADR-018）
- `chore(deps)`: 持续 bump model-hub-core / model-hub-sdk（`50b4acf` → `af1f3cd` → `fcdb443`）
- `fix model hub error`

### deploy

- `feature/summary_model_v4 → v8`：在 ap-northeast-1、us-west-2 多区域逐步推进 summary 模型版本，覆盖 image tag `95caa61 → 0308aa4 → 5c3ea6e → 37c1547 → 7eca408 → 7941bc0`。
- `feature/cn_model_hub_v1 → v3`：CN model hub 三轮灰度，image tag `b17ee98 → c3ba6b1 → ad85160 → af63af7` 推到 `values-pre.yaml` 与 `values-main.yaml`。
- `chore`: plaud-community-hub 同步更新到 `7941bc0`。

## 🔜 下周计划

### 本周成果跟进

- 跟进 ADR-021 上线后的 fallback 命中率与失败分布，确认是否需要进一步调整 capability 探测策略。
- 观察 CN model hub v3 在主环境的稳定性与合规拦截数据，沉淀 `PlaudModerationPlugin` 的运行手册。

### ADR 遗留问题清单（model-hub 侧，按优先级）

过去两周（ADR-011 → ADR-022）在文档里**明确承认是问题、但当时刻意不修**、且需要在 plaud-model-hub 仓库内改动的事项汇总，按严重程度排列。已剔除 plaud-summary 应用侧改动（`on_error` 日志补全、业务侧反模式清理、OTel pipeline 全局 View 排查、应用侧 timeout / 流式总预算、业务侧绕 hub 治理等）。

#### P0 — 生产已踩坑或正在持续踩

| # | 问题 | 来源 |
| --- | --- | --- |
| 1 | **fallback 链跨 region 单点**：`summary:gpt-5` 与 `summary:gpt-5.5` 两个 endpoint 都在 `azure-sweden-central`，Sweden 区整区抖动时 fallback 链整条挂。解法：在 `prod-model-config.yaml` 把 `azure-eu-germany-gpt-5` / `azure-eu-west-gpt-5` 的 `enabled` 改为 `true`。 | ADR-016 §6.1 |
| 2 | **CN 流式路径无 `content_filter` 检测**：`PlaudModerationPlugin.after_response` 只覆盖非流式；流式接入 CN 模型时需要在 `Plugin.wrap_async_stream` 钩子加 chunk-level `finish_reason=content_filter` 检测。 | ADR-020 §7.2 |

#### P1 — 治理缺失，已有 workaround 但持续吃技术债

| # | 问题 | 来源 |
| --- | --- | --- |
| 1 | **流式 Responses API 整体未支持**：业务用 `reasoning` / builtin tools / Responses 多模态都被迫降级到非流式；流式 unified chat 命中 Responses-only 字段时直接抛 `ProviderError`。 | ADR-011 §8 / ADR-013-azure §3 / ADR-017 §5 |
| 2 | **Bedrock 不在默认 `json_schema` 范围**：Bedrock provider 需新增 `response_format=json_schema → toolConfig` 转换，否则业务方接入需显式 `method="function_calling"` opt-out。 | ADR-019 §9 |
| 3 | **SSRF 兜底机制空缺**：当前 image_url 全部来自自家 S3 / CloudFront 可信，但 hub 没有 URL 白名单机制；业务一旦扩外部输入立刻退化成 P0。 | ADR-015 §3 |
| 4 | **结构化输出 / 降级 OTel 指标缺失**：SDK OTel span 需新增 `structured_output.method` / `structured_output.downgraded: bool` / `structured_output.downgrade_reason`（枚举：`capability_declared_false` / `none`）属性，便于统计 json_schema vs function_calling 分布、`strict` 占比、降级来源。 | ADR-019 §9 + ADR-021 §8 |

#### P2 — 设计完整性 / 影响有限

| # | 问题 | 来源 |
| --- | --- | --- |
| 1 | `aggregated_response.usage` 跨 endpoint failover 时只含最后一个 endpoint 的 token，计费 / 统计不准。 | ADR-013-mid 已知局限 |
| 2 | `reasoning_delta` 不累积进 `partial_content`，resume 模式下 reasoning_delta 丢失。 | ADR-013-mid 已知局限 |
| 3 | resume 模式下"已 yield + 续写"拼接处可能重复或风格突兀；结构化输出场景应禁用 resume。 | ADR-013-mid 已知局限 |
| 4 | **流式 + fallback 语义未设计**：中途切 model 时已 yield 出去的 chunk 怎么处理是独立设计议题。 | ADR-016 §6.3 |
| 5 | `_invoke_*` 出口未加 `mypy --strict` / runtime assert 卡死"kwarg 字典不允许 value=None"的不变式，无法防御同质 bug 复发。 | ADR-012 §7 |
| 6 | image_url fetch 异步路径仍是阻塞 `requests`；`ainvoke` 启用时需改 `httpx.AsyncClient`。 | ADR-015 §3 |
| 7 | 海外 provider 的 `content_filter` 等价语义未统一（OpenAI `content_filter` / Anthropic `refusal` / Gemini `SAFETY` 各自一套）；统一方案是 `ModelResponse.safety_flagged: bool`，独立架构议题。 | ADR-020 §7.3 |
| 8 | 新接入 OpenAI 兼容代理（LiteLLM / OpenRouter / 方舟类）上的第三方模型缺标准化的 capability 标定流程。 | ADR-021 §8 |

#### P3 — 占位 / 等触发条件

| # | 问题 | 来源 |
| --- | --- | --- |
| 1 | Engine 路由按 `provider.supported_request_types` 校验（image / audio / moderation 当前在 provider 层 fail fast，路由层未拦）。 | ADR-009 后续 |
| 2 | DashScope `BadRequestError` 子分类入 SDK；等 CN 接入方 ≥3 个再抽 `classify_dashscope_error` 辅助函数。 | ADR-020 §7.1 |
| 3 | 等 openai SDK 把 `/responses` 排除出 deployment 改写规则，Azure 即可同时兼容 deployment-path 与 v1 surface。 | ADR-013-azure §3 |
| 4 | `JunkStreamPlugin` Phase 1/2 边界与 `TTFTFailoverPlugin._has_payload()` 函数耦合；扩展 payload 定义时两处同步修改。 | ADR-014-post 已知局限 |
| 5 | 两个 watchdog plugin 并存时先 raise 者胜出（实际极低概率）。 | ADR-014-post 已知局限 |
| 6 | MIME magic-bytes 嗅探（当前 Content-Type → fallback jpeg 已兜底）。 | ADR-015 §3 / ADR-017 §5 |
| 7 | Responses 路径上 audio / video 多模态转换。 | ADR-017 §5 |
| 8 | `PlaudModerationPlugin` contextvar 之外的灰度模型注入（可加 `inject_decision` 回调，等差异接入方出现再做）。 | ADR-020 §7.4 |
| 9 | OTel histogram → ExponentialHistogram 迁移（AWS Managed Prometheus / Grafana 支持成熟后评估）。 | ADR-022 §6.3 |
| 10 | SDK 未来新增 `first_token_latency_ms` 等 latency 指标时复用 / 扩展本周引入的 boundary 常量。 | ADR-022 §6.4 |

