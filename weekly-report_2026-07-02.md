# 工作报告 - 2026-07-02

## 📅 本周（2026-06-26 ~ 2026-07-02）工作摘要

本周工作重心有两条主线：一是 **Model Gateway 链路建设**——plaud-model-hub 侧新增 Anthropic Gateway 静态密钥认证模式（ADR-028 提案到落地，覆盖 Vertex 和 Bedrock），plaud-summary 侧接入 gateway 并修复 summary 结果中实际输出 model 的记录（PR #659）；二是 **多区域部署配置扩展**，为 eu-central-1、ap-southeast-1 等区域补齐 pre-lane、AppConfig 与健康探针配置，并完成 plaud-summary 相关服务的发布与配置修正。

## 🚀 主要进展 (Key Achievements)

- **功能开发**:
  - 为 `anthropic_vertex` 和 `anthropic_bedrock` 新增 Gateway 静态服务密钥（static service key）认证模式，并撰写 ADR-028 记录设计决策（plaud-model-hub）。
  - plaud-summary 接入 model gateway：升级 model-hub-core / model-hub-sdk 依赖，新增 `summary_id_from_trace` 从 Hub trace_id 反解 summary_id，并补充 Bedrock / Azure OpenAI / Vertex Claude 经 Plaud Caddy gateway 调用的 demo notebooks。
  - 为多区域（含 eu-central-1、ap-southeast-1）新增 pre-lane 与 AppConfig 部署配置，更新 liveness/readiness 探针（deploy）。
- **问题修复**:
  - 修复 summary 结果 endpoint/model 字段：恢复记录 Hub 实际使用的模型，而非请求时指定的模型名（plaud-summary，PR #659）。
  - 修正 gateway 模式的启用条件：以解析后的 credentials_type 为准做门控，并强制要求 base_url，避免配置缺失时误入 gateway 路径（plaud-model-hub）。
  - 修正 plaud-summary-task 的 `APPCONFIG_SERVICE_NAME` 配置，并补充 preview 环境文档。
- **发布与调优**:
  - plaud-summary 完成 cn-northwest-1 区域镜像更新发布（image tag `1c09ac6`）。
  - 调大 drill.py 的 `BATCH_SIZE` 与 `SAMPLE_LIMIT`，提升压测请求处理吞吐。

### Model Gateway 收口 — 关键技术点（ADR-026 / 027 / 028）

- **目标架构**：LLM 出口统一收口到 plaud caddy gateway——业务侧不再持有云厂商凭证（GCP service account / AWS IRSA），改用 gateway 下发的静态 service key，由 gateway 注入真实凭证后转发；同时坚持继续经 model-hub provider 调用而非绕过 hub 直连 gateway，完整保留加权路由、fallback、circuit breaker、adaptive weight、rate limit、Langfuse 上报等治理能力。
- **Bedrock 注入方案（ADR-027）**：`AnthropicBedrock` SDK 必在客户端本地做 SigV4 签名且签名会重写 `Authorization` 头，因此 service key 经 `x-api-key` 头注入（SigV4 不触碰该头），并喂占位 ak/sk 满足签名前提；由 gateway 注入真实 AWS 凭证并重新完成 SigV4 后转发 Bedrock。核心改动约 3 行，业务侧与运行环境的 AWS 凭证彻底解耦。
- **Vertex 系注入方案（ADR-026 / 028）**：`google-genai` 与 `AnthropicVertex` SDK 均无条件用 `credentials.token` 覆盖 `Authorization` 且没有字符串 token 入口，唯一干净解法是构造静态 Credentials 壳类（`token` = service key，永不刷新）；`anthropic_vertex` 作为第二个 GCP 形态使用者出现时，把壳类抽到共享模块，消除跨 provider 耦合。
- **工程护栏**：gateway 模式下强制要求 `base_url`（validator 显式校验，防漏配打到公网端点导致鉴权失败）；client 缓存键纳入 service key 的 sha256 指纹（防换 key 撞缓存复用旧 client，防明文 key 进日志）；凭证自动推断优先级 `gateway_api_key` > `service_account_*` > `adc`，显式 `credentials_type` 可覆盖。
- **兼容与可靠性**：改动纯增量零回归——未配 `gateway_api_key` 的 provider 走原有 SA / IRSA / ADC 路径行为零变化；gateway 是新增共享故障域，关键 model 配「gateway 主 + 直连备」多 endpoint，靠熔断 + fallback 兜底；staging 实测非流式 / 流式均 200。
- **改造成本分级**：同批的 `azure_openai` 走 gateway 零代码改动（openai SDK 天然把 `api_key` 作为 `api-key` 头发出），纯配置迁移，不另立 ADR。

## 📝 详细提交记录 (Git Log Summary)

### plaud-model-hub

- feat: `anthropic_vertex` / `anthropic_bedrock` 新增 gateway 静态服务密钥认证模式
- fix: gateway 模式以解析后的 credentials_type 门控；gateway 模式下强制要求 base_url
- docs: 新增 ADR-028（anthropic_vertex gateway static key auth）

### deploy

- feat: 新增 eu-central-1、ap-southeast-1 的 pre-lane 配置及多区域 AppConfig 设置
- feat: 更新多区域 liveness/readiness 探针配置
- refactor: pre.yml 所有区域 lane 统一切换到 main
- chore: plaud-summary cn-northwest-1 镜像更新至 `1c09ac6`

### plaud-summary

- feat: 恢复 summary 结果 endpoint/model 记录 Hub 实际使用的模型（PR #659）
- feat: 新增 `summary_id_from_trace`，从 Hub trace_id 反解 summary_id
- feat: 新增 Bedrock / Azure OpenAI / Vertex Claude 经 Plaud Caddy gateway 的 demo notebooks
- chore: 升级 model-hub-core / model-hub-sdk 依赖至最新
- fix: drill.py 调大 `BATCH_SIZE` 与 `SAMPLE_LIMIT`

### 其他

- **plaud-summary-task**: 修正 `APPCONFIG_SERVICE_NAME`，补充 preview 环境文档

## 🔜 下周计划

- [ ] 验证 Anthropic Gateway 静态密钥模式在各环境的实际调用链路
- [ ] 跟进 eu-central-1 / ap-southeast-1 pre-lane 的部署验证
- [ ] 观察 plaud-summary 新镜像在 cn-northwest-1 的运行情况
