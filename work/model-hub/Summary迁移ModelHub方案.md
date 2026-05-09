# Plaud Summary 迁移 Model Hub 技术方案

> 目标：在不影响线上稳定性的前提下，将 `plaud-summary` 的 LLM 调用层从自研 `EndpointDispatcher` 渐进迁移到 Model Hub，支持**按百分比放量**、**按模型粒度切换**、**CN 模型延后改造**。

---

## 一、问题与目标

### 1.1 现状

- `plaud-summary` 通过 `EndpointDispatcher`（[endpoint.py](../../../../../Documents/work/plaud-summary/plaud_summary/plaud/endpoint_manager/endpoint.py)）维护多个 LLM 池：`_pool`、`_o3_pool`、`_claude_sonnet_4_5_pool`、`_gcp_claude_*_pool` 等。
- 每个 `Endpoint` 持有具体厂商 SDK：`ChatOpenAI / AzureChatOpenAI / ChatBedrock / ChatBedrockConverse / ChatVertexGemini / ChatAnthropicVertex / ChatPlaudAI`。
- Dispatcher 实际生效的能力：加权分发（`hits`）、`ExceptionMonitor` 熔断、global fallback、按 `ModelName` 路由。
- ⚠️ `Endpoint.tpm/rpm` 字段以及 `_check_endpoint` 中的 RPM/TPM 检查是**死代码**：[endpoint.py:435](../../../../../Documents/work/plaud-summary/plaud_summary/plaud/endpoint_manager/endpoint.py#L435) 函数首行 `return True` 直接短路；`tpm/rpm` 仅在 `__str__` 中被打印。本方案不再考虑限流维度，相关字段在 Phase 4 清理。

### 1.2 目标能力（Model Hub 提供）

加权随机 / 优先级 / 会话粘性、滑动窗口熔断、Token Bucket 限流（**Hub 引入后才有真正的限流**）、配置热更新、统一可观测性（Langfuse / OTel）。

### 1.3 核心约束

| 约束 | 说明 |
|---|---|
| **不双层路由** | 一次调用要么走旧路径（Dispatcher 选 Endpoint），要么走新路径（Hub 选 endpoint），**不能两层都走** |
| **可放量回滚** | 必须支持任意百分比切换，按 ModelName 维度独立配置 |
| **CN 延后** | 豆包/通义/Kimi/DeepSeek 依赖 `ChatPlaudAI` 的内容安全检测，第一阶段不动 |
| **观测对齐** | 新旧路径的 cost / latency / 错误率必须能并排对比 |

---

## 二、架构方案

### 2.1 关键决策：避免双层路由

**采用"二选一"分流，而不是"包一层"。**

方案一： ❌：保留 `Endpoint`，让每个 Endpoint 内部 `self.llm = ChatModelHub(...)`。这样 Dispatcher 选完 Endpoint 后，Hub 又会基于其配置再选一次 endpoint，造成：
- 权重失效（外层挑了 A，内层可能又转 B）
- 熔断冲突（两层都在熔断同一个 key）
- 故障归因困难（不知道是哪层选的）

方案二： ✅：**入口分流**。在 `get_endpoint_for_llm` 之前判断当前请求该走哪条路径：

```
                   ┌─→ [旧] EndpointDispatcher → Endpoint(具体厂商SDK).invoke()
                   │           ↑ 完整保留 hits 加权 / ExceptionMonitor 熔断
请求 → 路由决策 ────┤
                   │
                   └─→ [新] HubAdapter → ChatModelHub(app_id, model).invoke()
                               ↑ 路由 / 熔断 / 限流全部由 Hub 负责
                               ↑ 旧 Dispatcher 完全不参与
```

**关键不变量**：进入新路径的请求，**绝不**调用 `EndpointDispatcher.dispatch_endpoint()` / `ExceptionMonitor`。新路径的统计、熔断、限流统一委托给 Hub，避免任何状态串污染。

### 2.2 新路径的最小 Endpoint Shim

下游业务代码（`call_note.py`、`mark_note_summary.py` 等几十处）大量使用 `endpoint.llm`、`endpoint.region`、`endpoint.is_reasoning`、`endpoint.supports_json_format`、`endpoint.get_model_name_str()`。短期内不可能全改。

设计一个 `HubEndpoint` shim，**鸭子类型**兼容现有 `Endpoint` 接口：

```python
# plaud_summary/plaud/endpoint_manager/hub_endpoint.py
class HubEndpoint(Endpoint):
    """Model Hub 路径下的 Endpoint shim。

    保持与原 Endpoint 相同的鸭子接口（llm / name / region / type /
    is_reasoning / supports_json_format / get_model_name_str），让下游
    业务代码无感切换。但内部 self.llm 是 ChatModelHub，所有路由/熔断
    /限流由 Hub 负责，不进入 EndpointDispatcher 的统计。
    """
    def __init__(self, model_name: ModelName, *, app_id: str = "summary",
                 logical_model: str, is_reasoning: bool = False,
                 supports_json_format: bool = True):
        super().__init__(
            name=f"hub:{logical_model}",
            region="hub",
            type=_infer_type(model_name),
            llm_main=ChatModelHub(app_id=app_id, model=logical_model),
            tpm=0, rpm=0,                # 兼容父类签名，不参与任何逻辑（旧字段已是死代码）
            is_reasoning=is_reasoning,
            model_name=model_name,
            supports_json_format=supports_json_format,
        )
        self._is_hub = True              # 用于 isinstance 替代判断
```

**为什么继承 `Endpoint` 而不是 duck-only**：现有代码有 `isinstance(endpoint.llm, ChatOpenAI)` 这类判断（详见 §3.4），保留继承让 `endpoint.llm` 检查走分支前先检查 `endpoint._is_hub`。

### 2.3 路由层（Rollout 决策）

**唯一入口**：[summary.py:2254 `get_endpoint_for_llm`](../../../../../Documents/work/plaud-summary/plaud_summary/summary.py#L2254)。在该函数最前面注入分流逻辑：

```python
# plaud_summary/plaud/endpoint_manager/hub_rollout.py
import hashlib
from plaud_summary.plaud.endpoint_manager.llm_config import ModelName
from plaud_summary import config   # 你们的 appconfig 入口，按实际名字替换

# ModelName → Hub 逻辑模型名（CN 模型不登记 → 永远走旧路径）
HUB_MODEL_MAP: dict[ModelName, str] = {
    ModelName.GPT_4_1:           "summary:gpt-4.1",
    ModelName.O3:                "summary:o3",
    ModelName.O4_MINI:           "summary:o4-mini",
    ModelName.CLAUDE_SONNET_4_5: "summary:claude-sonnet-4-5",
    ModelName.GEMINI_2_5_PRO:    "summary:gemini-2.5-pro",
}

def _rollout_ratio(model_name: ModelName) -> float:
    """从 appconfig 读取该模型走 Hub 的比例（0.0 ~ 1.0）。

    appconfig 示例 (yaml/json/远程配置中心皆可)：

        model_hub_rollout:
          GPT_4_1: 0.1
          O3: 0.0
          CLAUDE_SONNET_4_5: 0.5

    缺省（未配置）= 0，即仍走旧路径。
    """
    raw = config.get("model_hub_rollout", {}) or {}
    try:
        return float(raw.get(model_name.value, 0.0))
    except (TypeError, ValueError):
        return 0.0

def should_use_hub(model_name: ModelName, sticky_key: str) -> bool:
    """决定该任务此次调用是否走 Hub。

    - 未登记到 HUB_MODEL_MAP 的模型（如 CN）一律返回 False。
    - sticky_key 用 summary_id（或 file_id）保证同任务粘性，避免
      一次总结里前半截走旧后半截走新。
    - 哈希取模做确定性灰度，便于复现 / 排查。
    """
    if model_name not in HUB_MODEL_MAP:
        return False
    ratio = _rollout_ratio(model_name)
    if ratio <= 0:
        return False
    if ratio >= 1:
        return True
    bucket = int(hashlib.md5(f"{model_name.value}:{sticky_key}".encode()).hexdigest()[:8], 16) % 10000
    return bucket < int(ratio * 10000)
```

**调整放量比例不需要改代码**：直接改 appconfig（或配置中心）的 `model_hub_rollout` 字段，下次请求生效。


### 2.4 改造点：`get_endpoint_for_llm`

```python
# summary.py 内
def get_endpoint_for_llm(content, llm, scenario, language, summary_id=None):
    # ... 既有逻辑：解析 model_name ...
    model_name = _resolve_model_name(llm, scenario, language)

    # 分流：决定本次走哪条路径
    if should_use_hub(model_name, sticky_key=str(summary_id or "")):
        ep = build_hub_endpoint(model_name)              # HubEndpoint
        logger.info(f"rollout=hub, summary_id={summary_id}, model={model_name.value}")
        return ep

    # 旧路径不变
    return EndpointDispatcher().dispatch_endpoint(
        content=content, will_use_model=model_name, summary_id=summary_id,
    )
```

`build_hub_endpoint` 单例缓存 `HubEndpoint` 即可（`ChatModelHub` 是无状态客户端）：

```python
@functools.lru_cache(maxsize=64)
def build_hub_endpoint(model_name: ModelName) -> HubEndpoint:
    logical = HUB_MODEL_MAP[model_name]
    return HubEndpoint(
        model_name=model_name,
        logical_model=logical,
        is_reasoning=model_name in {ModelName.O3, ModelName.O4_MINI},
        supports_json_format=_lookup_json_support(model_name),
    )
```

---

## 三、改造影响面清单

### 3.1 上游：路由 / 配置

| 文件 | 改动 | 风险 |
|---|---|---|
| `summary.py:get_endpoint_for_llm` | 入口加 `should_use_hub` 分流 | 低 |
| `endpoint_manager/hub_rollout.py` | **新增** | 低 |
| `endpoint_manager/hub_endpoint.py` | **新增**，HubEndpoint shim | 低 |
| `pyproject.toml` | 加 `model-hub-core` / `model-hub-sdk` 依赖 | 低 |
| `config.yaml`（Hub） | **新增**，定义 `summary:*` 逻辑模型 → 实际 endpoint 映射 | 中（首次配置易错） |

### 3.2 下游：直接读 `endpoint.llm` 的代码

调用 `.invoke / .ainvoke / .with_structured_output / .bind_tools / LLMChain(llm=...)` —— `ChatModelHub` 都兼容，**不需要改**。覆盖：

- [`mark_note/mark_note_processor.py`](../../../../../Documents/work/plaud-summary/plaud_summary/mark_note/mark_note_processor.py)
- [`mark_note/mark_note_summary.py`](../../../../../Documents/work/plaud-summary/plaud_summary/mark_note/mark_note_summary.py)
- [`mark_note/mark_note_post_processor.py`](../../../../../Documents/work/plaud-summary/plaud_summary/mark_note/mark_note_post_processor.py)
- `plaud/{call_note,meeting_seminar_note,meeting_consult_note,education_*,sales_bant_note,reasoning_one_shot_note,overview_chunk_note}.py`
- `trans_compress/service.py`、`features/persona/*.py`

### 3.3 下游：`isinstance(endpoint.llm, ChatXxx)` 判断

`ChatModelHub` 是统一类型，原 `isinstance` 分支会全部走 `else`。需要在新路径下用 `endpoint._is_hub` 短路：

| 位置 | 当前逻辑 | 改造 |
|---|---|---|
| [endpoint.py:266](../../../../../Documents/work/plaud-summary/plaud_summary/plaud/endpoint_manager/endpoint.py#L266) `isinstance(..., ChatOpenAI)` | 用于 `_pool` 内部判断 | 旧路径独占，HubEndpoint 不进 `_pool`，**无需改** |
| [endpoint.py:423](../../../../../Documents/work/plaud-summary/plaud_summary/plaud/endpoint_manager/endpoint.py#L423) `_filter_endpoint` 排除 Bedrock | 同上 | 同上 |
| [check_failure_handler.py:130-136](../../../../../Documents/work/plaud-summary/plaud_summary/plaud/check_failure_handler.py#L130-L136) callback 选择 | Hub 路径需要单独分支 | **必改**：先判 `getattr(endpoint, "_is_hub", False)`，是则用 Hub 提供的 callback / Langfuse 抓取 token；否则走原分支 |

### 3.4 Token / 成本统计

Hub 路径下 `langchain_community.callbacks.get_openai_callback()` 抓不到 token（Hub 走自己的 client）。两个选项：

- **A**：用 `model_hub_sdk` 暴露的 callback / Langfuse 数据反查；
- **B**：放量初期 Hub 路径的 cost 由 Hub Langfuse 看板统计，旧 cost 表不写。`tokens_cost` 字段在 Hub 路径置 0，并在日志中标记 `path=hub` 方便对账。

推荐先 B 后 A，避免一次性吞太多。

### 3.5 ChatPlaudAI 的内容安全检测

`ChatPlaudAI` 在 OpenAI 协议响应上叠加内容安全检测/重试（豆包/通义/Kimi/DeepSeek 都依赖）。本期**不动 CN 模型**，所以 `ChatPlaudAI` 对应的 ModelName（`DOUBAO_*` / `QWEN_*` / `KIMI_*` / `DEEPSEEK_*`）不进 `HUB_MODEL_MAP`，`should_use_hub` 直接返回 `False`，自然继续走旧路径。

CN 后续接入时，方案二选一：
1. 把内容安全检测做成 Model Hub 的 plugin（在 Hub 侧统一拦截）；
2. 用 `wrap_openai` 风格 wrapper 在 SDK 层注入，保留 `ChatPlaudAI` 的 hook 含义。

---

## 四、放量阶段规划

### Phase 0 — 准备（无功能变更，1～2 天）

- 依赖：`uv add model-hub-core model-hub-sdk`（按 readme 指定 git 源）
- 写出 `config.yaml`（Hub 配置）骨架，先放 1 个模型 1 个 endpoint，跑通本地 demo
- 在 dev 环境跑通 `ChatModelHub("summary:gpt-4.1").invoke("ping")`
- **不改主代码**

### Phase 1 — 接入分流骨架（0% 流量，1 天）

- 新增 `hub_rollout.py` / `hub_endpoint.py` / `build_hub_endpoint`
- 改 `get_endpoint_for_llm` 加分流判断
- appconfig 中 `model_hub_rollout` 全部缺省 / 0
- 上线后**实际行为不变**，只验证：HubEndpoint 能正确构造 / 分流逻辑分支被正确跳过 / 无导入错误

### Phase 2 — Canary 1% → 10% → 50% → 100%（每档观察 24～72 小时）

调比例 = 改 appconfig 中 `model_hub_rollout.<MODEL>` 的小数值，**不需要发版**。按以下顺序灰度，**每改一档都看监控**：

| 档位 | 模型 | 来源 |
|---|---|---|
| 0.01 | `GPT_4_1` | 风险最小、流量最大、对比基准 |
| 0.1 | `GPT_4_1` | |
| 0.5 | `GPT_4_1` | |
| 1.0 | `GPT_4_1` | 切换完成 |
| 0.01→1.0 | `O3` / `O4_MINI` | reasoning 模型独立爬坡 |
| 0.01→1.0 | `CLAUDE_SONNET_4_5` | Bedrock + GCP 两路 endpoint，Hub 配置要覆盖 |
| 0.01→1.0 | `GEMINI_2_5_PRO` | Vertex 凭据 / location 处理 |

### Phase 3 — 解耦旧 Pool（per-model 100% 后）

某模型 100% 切到 Hub 后：

- 删除其 `_parse_xx_endpoints` 函数 / `_xx_pool`
- 清理对应的 `_filter_endpoint` 分支

### Phase 4 — 死代码清理（可前置，独立 PR）

不依赖 Hub 改动，建议**先于 Phase 1 落地**以缩小后续 diff：

- 移除 `Endpoint.tpm/rpm` 字段
- 删除 `_check_endpoint` 整个函数
- 删除 `_rpm_tc_data` / `_tpm_tc_data` / `_pre_minute` 状态
- 清理 `_check_endpoint` 的所有调用点（`_dispatch_weighted_pool` / `_dispatch_gemini_endpoint_hits` / `_dispatch_endpoint` 等处的 `if self._check_endpoint(...)` 判断直接去掉）

### Phase 5 — CN 模型迁移（独立项目）

待 `ChatPlaudAI` 内容安全检测在 Hub 侧落地后，再按 Phase 1~3 重复一轮。**本方案不覆盖此阶段细节。**

### Phase 6 — Dispatcher 退场

所有非 CN 模型迁完后，`EndpointDispatcher` 仅服务 CN。CN 迁完后整个类移除，`Endpoint` shim 也可以拆掉（业务代码改为直接调用 `ChatModelHub`）。

---

## 五、可观测性 / 回滚

### 5.1 关键指标（按 path=old/hub × model_name 切面）

- **业务**：每分钟请求数、p50/p95/p99 latency、错误率、cost/请求、token/请求
- **质量**：输出长度分布（避免 Hub 路径返回截断）、JSON 解析失败率（`with_structured_output` 重试次数）
- **Hub 内部**：熔断打开次数、限流命中次数、fallback 触发次数（Hub 自带 metrics）

### 5.2 回滚策略

- **秒级回滚**：appconfig 中 `model_hub_rollout.<MODEL>` 改回 `0` 即全部回到旧路径，无需代码发布
- **粒度**：单模型回滚不影响其他模型
- **粘性**：同一 `summary_id` 在重试间不变更分流决策，避免一次任务横跨两条路径

### 5.3 日志规范

`get_endpoint_for_llm` 必须打印 `summary_id` / `model_name` / `path=old|hub` / `endpoint_name`，方便 grep 复盘。

---

## 六、风险矩阵

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| Hub 配置写错（密钥 / endpoint 漏写） | 中 | 单模型不可用 | Phase 0 dev 环境跑通后再上 prod；Phase 2 首档 1% 灰度提前暴露 |
| `with_structured_output` 在 ChatModelHub 上行为不同 | 中 | JSON 解析失败 | Phase 2 1% 档位重点观察成功率，必要时换 prompt 兼容写法 |
| Hub Langfuse 双写 cost（Hub 路径在 Hub 看板，旧路径在自有看板） | 高 | 短期对账复杂 | 接受，提供切面看板；Phase 4 完成后旧看板下线 |
| Hub 熔断粒度（按 endpoint）与旧 Dispatcher 不一致 | 中 | 切流期间错误率波动 | 先小流量看数据，必要时调 Hub 熔断阈值 |
| `endpoint.region` 在下游业务用作日志/计费 tag，Hub 路径返回 `"hub"` | 低 | tag 失真 | Hub 提供"实际选中的 endpoint"元数据 → 在 HubEndpoint 中透传 |
| `ChatPlaudAI` 的内容安全检测意外被 CN 模型外的代码依赖 | 低 | 重构遗漏 | 全局 grep `ChatPlaudAI` 锁定调用面（已确认仅 CN parse 函数使用） |
| 双层路由意外发生（误把 HubEndpoint 加进 `_pool`） | 低 | 路由失效 | `EndpointDispatcher.add_endpoint` 加断言：`assert not getattr(ep, "_is_hub", False)` |

---

## 七、决策清单（开工前需要敲定）

1. **Hub 配置存放位置**：内嵌 yaml / 配置中心 / S3？影响热更新方式。
2. **app_id**：建议固定 `"summary"`；如要按业务子线再细分（mark_note / call_note / persona）需要在 §2.4 `build_hub_endpoint` 入参里加 scenario。
3. **环境隔离**：`env=dev/staging/prod`，对应不同 yaml；CI 怎么注入？
4. **Langfuse 项目**：与现有 trace 体系合并 or 新建 `summary-hub` 项目？
5. **放量配置开关位置**：环境变量 / 配置中心。建议配置中心，避免发布。
6. **是否需要离线 replay 对账**：本方案默认不做。如需做，建议在每档灰度后一周内挑取一定数量的 summary_id 在测试环境用 Hub 路径重跑做内容对比。

---

## 八、首版改动文件清单（Phase 1 实际落盘）

```
新增:
  plaud_summary/plaud/endpoint_manager/hub_rollout.py
  plaud_summary/plaud/endpoint_manager/hub_endpoint.py
  config/model_hub.yaml          (Hub 配置)

修改:
  plaud_summary/summary.py       (+10 行：get_endpoint_for_llm 入口分流)
  plaud_summary/plaud/check_failure_handler.py
                                 (+5 行：_is_hub 短路)
  pyproject.toml                 (+2 依赖)

不动:
  EndpointDispatcher / parse_*_endpoints / _create_generic_llm
  下游业务代码（call_note / mark_note 等几十处）
  CN 模型相关 parse_* 函数
```

---

## 九、回答最初的疑虑

> "如果保留 `_pool + _model_pools`，每个 Endpoint 又包一个 ChatModelHub，就会变成调度器选 endpoint → Hub 再选一次。"

**本方案不会出现此问题**，原因：

1. **HubEndpoint 不进 `_pool`**，也不进 `_model_pools`。它由 `build_hub_endpoint` 直接构造，绕过 Dispatcher。
2. **入口二选一**：`get_endpoint_for_llm` 要么返回旧 `Endpoint`（来自 `_pool`），要么返回 `HubEndpoint`，互斥。
3. **Hub 配置中只列实际 endpoint**，不去引用旧 Dispatcher 的池子，所以 Hub 内部的路由是独立闭环。
4. **`ExceptionMonitor` 只在旧路径调用**：`HubEndpoint` 不参与 `_dispatch_endpoint` / `_check_endpoint`，自然不进入旧统计。

放量百分比（X%）的语义就是：**X% 的任务从入口就去走新路径，剩下 (100-X)% 从入口去走旧路径。两条路径的状态完全隔离。**

CN 模型在 `HUB_MODEL_MAP` 中不登记，`should_use_hub` 直接返回 `False`，永远走旧路径，与 §5 的快速回滚机制互不影响。
