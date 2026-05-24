# Plaud Summary 迁移 Model Hub 技术方案

## 0、本次审核结论

不能直接作为 Phase 1 开发单，必须先修正以下阻塞项：

1. **Model Hub SDK 多模态未就绪**：参考仓库当前 `ChatModelHub.with_config(...)` 已修复 LangChain Runnable 语义，非流式 token metadata 也已返回；但 `ChatModelHub._get_message_content()` 仍会把 `HumanMessage(content=[text, image_url])` 压成纯文本，`docs/decisions/007-multimodal-content-passthrough.md` 还处于“提议中”。因此图片 mark 不能进 Hub 灰度，除非先合入并发布 ADR-007 对应代码。
2. **直接 `EndpointDispatcher` 调用点比方案列出的多**：主 summary 入口之外，mark note、persona、self-check、enable-result、re-act、model-name-review、trans-compress、ai-router、knowledge short、summary card 都有直接 dispatch。若目标是“覆盖所有 chat 模型”，Phase 1 必须把这些调用点分级处理；否则只能把目标降级为“主 summary 主链路 + embedding”。
3. **Gemini 3 旧名兼容漏了**：当前 `LLMName.GEMINI_3_PRO` 仍是 `gemini-3-pro`，而 Hub YAML 是 `summary:gemini-3.1-pro`。进入 Hub 判断前必须把 `gemini-3-pro` 归一化到 `gemini-3.1-pro`，否则前端旧值会绕回旧 Dispatcher。
4. **Hub 路径 token/cost 需要代码级保护**：当前 `basic_runnable_summary.py` 会在回调后无条件调用 `calculate_llm_cost(endpoint=...)`。HubEndpoint 如果映射到旧 `ModelName`，会被本地价格表估算成本，违背“Hub 路径不算旧 cost”。必须显式跳过本地 cost 计算，并保留 token 字段。
5. **本地 Hub YAML 含敏感配置，不应入库**：`conf/dev-endpoints-model-hub.yaml` 当前是未跟踪文件且包含内联凭证默认值。生产前必须改为环境变量/Secret Manager 引用，并把本地明文配置加入 ignore 或只保留脱敏样例。

## 一、目标

把 `plaud-summary` 里走 `EndpointDispatcher` 拿 endpoint 的请求按一个全局比例切到 Model Hub。覆盖所有 chat 模型 + AUTO + Embedding。

**核心思路**:`get_endpoint_for_llm` 入口处直接用 logical 模型字符串交给 Hub。**Hub yaml 是唯一真理源** —— yaml 里登记了就接管,没登记就走旧路径。代码里不维护任何"哪些模型走 Hub"的硬编码列表。

CN 模型本期不动(依赖 `ChatPlaudAI` 内容安全豁免,等 Hub 侧补 plugin 后再迁)。

> 关键不变量:**一次调用要么旧 Dispatcher 选 endpoint,要么 Hub 选 endpoint,绝不两层都选。**

---

## 二、为什么是"入口分流",不是"`Endpoint.llm` 套娃"

**套娃做法**:保留 `Endpoint` 类,把它内部的 `self.llm` 从 `AzureChatOpenAI / ChatBedrock / ChatVertex...` 换成 `ChatModelHub(...)`。

**为什么不行**:旧 Dispatcher 在前面已经把模型 + endpoint 选完了,Hub 永远只能拿到 `summary:gpt-5` 这种已经定死的 logical model,看不到 `summary:auto`。后果:

1. Hub 的 `summary:auto`(yaml 把模型选择 + endpoint 选择压扁成一张加权表)用不上 —— `dispatch_auto_model` 已经把模型选完了
2. 未来 Hub 侧的跨模型能力(`fallback_model` 降级链、按 prompt 自动降级到 flash 等)全都用不上
3. 双层路由:旧 `_pool` 按 hits 挑了 Sweden,Hub 又按 yaml 权重挑了 Germany,第一层白选;熔断归因错位

**入口分流(本方案)**:走 Hub 的请求从 `get_endpoint_for_llm` 直接拐走,不进 `EndpointDispatcher`。两条路状态隔离 —— HubEndpoint 不进 `_pool`,旧 ExceptionMonitor 看不到它;Hub 内部的熔断/限流也不知道旧 `_pool` 的存在。

---

## 三、落地

### 3.1 `hub_endpoint.py`(新增,~60 行)

`ChatModelHub` / `EmbeddingsModelHub` 的鸭子类型壳。下游会直接读 `endpoint.model_name` / `endpoint.type` / `endpoint.is_global_fallback` / `endpoint.get_model_name()` 等旧 `Endpoint` 字段,所以 HubEndpoint 不能只暴露 `.llm` / `.name`。

先给旧枚举加一个 Hub 类型:

```python
# plaud_summary/plaud/endpoint_manager/llm_config.py
class EndpointType(Enum):
    ...
    HUB = "hub"
```

```python
# plaud_summary/plaud/endpoint_manager/hub_endpoint.py
from dataclasses import dataclass
from model_hub_sdk.integrations.langchain import ChatModelHub, EmbeddingsModelHub
from plaud_summary.plaud.endpoint_manager.llm_config import EndpointType, ModelName

@dataclass
class HubEndpoint:
    """Hub 路径的 Endpoint 鸭子类型。

    刻意不继承 Endpoint —— Endpoint 上的 tpm/rpm/hits/_pool 等字段对 Hub 没意义。
    一个实例只服务一种用途:chat(llm 非空)或 embedding(embedding_llm 非空)。
    """
    name: str                                          # 形如 "hub:summary:gpt-5"
    logical_model: str                                 # Hub yaml 中的 logical model
    model_name: ModelName = ModelName.UNKNOWN          # 兼容旧下游直接访问
    type: EndpointType = EndpointType.HUB              # 兼容旧 provider 类型判断
    region: str = "hub"
    is_global_fallback: bool = False
    is_reasoning: bool = False
    supports_json_format: bool = True
    is_hub: bool = True

    llm: ChatModelHub | None = None
    embedding_llm: EmbeddingsModelHub | None = None

    def get_model_name(self) -> ModelName:
        return self.model_name

    def get_model_name_str(self) -> str:
        return self.logical_model

    def get_name(self) -> str:
        return self.name

    def get_region(self) -> str:
        return self.region
```

配套调整:

- `endpoint.type == EndpointType.GEMINI` 这类 provider 专属逻辑对 Hub 不适用;Hub 路径应走 `getattr(endpoint, "is_hub", False)` 分支。
- structured output 白名单如果按 `endpoint.type` 判断,要加入 `EndpointType.HUB`,或改成能力判断: `hasattr(endpoint.llm, "with_structured_output")`。

### 3.2 Model Hub YAML 配置源和热更新

生产环境的 Hub YAML 不从本地 `conf/dev-endpoints-model-hub.yaml` 读;本地文件只用于开发 / smoke。线上应从 AWS AppConfig 的 Model Hub 配置 profile 读取。

这里有两个容易混淆的 AppConfig 读取路径:

- `plaud_library_python.configpkg.config.get_aws_config`:当前 `plaud-summary` 主配置读取入口,由 `init_aws_config(...)` 初始化,后台 watchdog 热更新。`summary_hub_rollout` 应继续放在这个主配置里。
- `model_hub_sdk.config.AppConfigConfigStore`:Model Hub SDK 自带的 AppConfigData client。`plaud-project-summary/common/llm.py` 里就是显式 new 这个 store 后传给 `ChatModelHub`,它不是 `get_aws_config` 那套全局配置。

如果照搬 `plaud-project-summary` 的 `AppConfigConfigStore`,还要额外解决 refresh:SDK 的 `get_config()` 返回内存快照,不会自动调用 `refresh()`;除非每次重建 store,否则长生命周期 `ChatModelHub` 看不到 YAML 变更。

本项目建议不要直接采用 SDK 默认 AppConfig client,而是复用 / 泛化当前已有的 `plaud_summary.llm_profile_config` 模式:用 `AppConfigSessionClient` 加载 Model Hub YAML profile,再封装成 Model Hub SDK 的 `ConfigStore`,统一传给 `ChatModelHub` / `EmbeddingsModelHub`。这样 Hub YAML 删除条目、改权重都能随 AppConfig profile watchdog 热更新,不被进程内永久缓存卡住。

新增 `hub_config.py` 负责三件事:

- `get_model_hub_config_store()` 返回共享的 Model Hub `ConfigStore`。
- `get_model_hub_env()` 返回当前 AppConfig env,避免 `ChatModelHub` 默认 `prod`。
- `hub_models()` 从 `config_store.get_config().models` 当前快照派生 `summary:*` logical model 集合。它必须 fail closed:配置未初始化、解析失败、模型不存在、`ModelConfig.enabled=False`、或该模型没有任何启用 endpoint 时,都返回不接管,让上层走旧 Dispatcher。这样 Hub YAML 删除条目 / 禁用模型才能作为模型级回滚手段。

边界要求:

- 不对 Hub YAML 再调用 `plaud_library_python.configpkg.config.init_aws_config(...)`;该函数维护全局 `_aws_config`,继续只服务主配置和 `get_aws_config(...)`。
- 不直接复用 `plaud_summary.llm_profile_config.init_llm_config(...)` 加载 Hub YAML;该函数会触发 `refresh_model_index()`,是旧 `llm.yaml model_registry` 的副作用。
- `hub_config.py` 自己持有独立 `AppConfigSessionClient`、token、watchdog 和解析后的 Model Hub `ConfigStore`;它只影响 `ChatModelHub` / `EmbeddingsModelHub`,不覆盖原主配置读取。
- 需要在 `main.py` 和 `server/api.py` 初始化主配置、`llm_profile` 之后显式初始化 Hub 配置,例如 `init_model_hub_config(service_name, run_env, model_hub_profile, aws_region)`。本地调试沿用 `LOCAL_MODE/LOCAL_CONFIG_FILE` 同目录查找,但只读取脱敏后的 Model Hub YAML。
- `ConfigStore.refresh()` 不能只更新原始 dict;必须重新走 `model_hub_core.config.parser.ConfigParser(default_app_id="summary", default_env=get_model_hub_env())` 解析并替换内存中的 `ModelHubConfig`,同时递增 `version`。

> 注意:当前 `plaud_library_python.configpkg.aws_config.AppConfigSessionClient` 默认 `polling_seconds=120`,并用这个值作为 `RequiredMinimumPollIntervalInSeconds`。所以这里的“热更新”是免重启生效,不是严格秒级生效;具体 SLA 取决于 AppConfig 返回的下一次轮询间隔。

### 3.3 `hub_rollout.py`(新增,~90 行,**唯一的决策点**)

**前置:Hub yaml 命名已对齐 `llm.yaml frontend_models` 的目标字符串。**`conf/dev-endpoints-model-hub.yaml` 已把 4 个 logical key 改名(`summary:claude-sonnet-4-5` → `summary:claude-sonnet-4.5`、`summary:gemini-3.1-pro-preview` → `summary:gemini-3.1-pro` 等)。但当前代码枚举里仍保留 `LLMName.GEMINI_3_PRO = "gemini-3-pro"`,旧 APP / 旧调用方可能继续传旧值,所以必须保留旧名归一化。

旧 APP / 前端可能还会传过期 `LLMName`。这类兼容不放进 Hub 接管模型清单,而是在进入 Hub 判断前做一次**旧名归一化**:把过期 `LLMName` 转成 Hub yaml 中存在的 logical model。这个映射只服务客户端兼容,不表示"哪些模型走 Hub"。

```python
# plaud_summary/plaud/endpoint_manager/hub_rollout.py
import functools
import hashlib
from model_hub_sdk.integrations.langchain import ChatModelHub, EmbeddingsModelHub
from plaud_library_python.configpkg.config import get_aws_config
from plaud_summary.logger import logger
from plaud_summary.plaud.endpoint_manager.endpoint import EndpointDispatcher
from plaud_summary.plaud.endpoint_manager.hub_config import (
    get_model_hub_config_store,
    get_model_hub_env,
    hub_models,
)
from plaud_summary.plaud.endpoint_manager.hub_endpoint import HubEndpoint
from plaud_summary.plaud.endpoint_manager.llm_config import ModelName

# 从热更新后的 Hub YAML 当前快照读出所有 summary app 下登记的 logical 模型名。
# 当前 Hub v2 yaml 使用扁平 key: models.summary:gpt-5 / models.summary:auto。
# yaml 是单一真理源:加 / 移除模型只改 yaml,代码零改动。
def _hub_models() -> set[str]:
    return hub_models()

_HUB_LOGICAL_TO_MODEL_NAME = {
    # 只为旧下游字段兼容服务,不是 Hub 接管清单。未映射的 Hub logical model 返回 UNKNOWN。
    "gpt-5": ModelName.GPT_5,
    "gpt-5.2": ModelName.GPT_5_2,
    "gpt-5.5": ModelName.GPT_5_5,
    "gemini-2.5-pro": ModelName.GEMINI_2_5_PRO,
    "gemini-2.5-flash": ModelName.GEMINI_2_5_FLASH,
    "gemini-3.1-pro": ModelName.GEMINI_3_PRO,          # 旧代码里按 GEMINI_3_PRO 兼容
    "gemini-3-flash": ModelName.GEMINI_3_FLASH,
    "claude-sonnet-4.5": ModelName.CLAUDE_SONNET_4_5,
    "claude-sonnet-4.6": ModelName.CLAUDE_SONNET_4_5,   # 旧代码里按 4.5 兼容
}

_MODEL_NAME_TO_HUB_LOGICAL = {
    # 只为旧代码以 ModelName 调用 route_model_endpoint 时找到 Hub logical model。
    ModelName.GPT_5: "gpt-5",
    ModelName.GPT_5_2: "gpt-5.2",
    ModelName.GPT_5_5: "gpt-5.5",
    ModelName.GEMINI_2_5_PRO: "gemini-2.5-pro",
    ModelName.GEMINI_2_5_FLASH: "gemini-2.5-flash",
    ModelName.GEMINI_3_PRO: "gemini-3.1-pro",
    ModelName.GEMINI_3_FLASH: "gemini-3-flash",
    ModelName.CLAUDE_SONNET_4_5: "claude-sonnet-4.5",
}

def _model_name_for_logical(logical_model: str) -> ModelName:
    return _HUB_LOGICAL_TO_MODEL_NAME.get(logical_model, ModelName.UNKNOWN)

def _logical_for_model_name(model_name: ModelName) -> str:
    return _MODEL_NAME_TO_HUB_LOGICAL.get(model_name, model_name.value)

def _ratio() -> float:
    raw = get_aws_config("summary_hub_rollout", 0.0)
    try:
        return max(0.0, min(1.0, float(raw)))
    except (TypeError, ValueError):
        return 0.0

def _hit(key: str) -> bool:
    r = _ratio()
    if r <= 0: return False
    if r >= 1: return True
    return int(hashlib.md5(key.encode()).hexdigest()[:8], 16) % 10000 < int(r * 10000)

def try_hub(llm: str, summary_id: str | None, force: bool = False) -> HubEndpoint | None:
    """统一入口:任意前端 logical 字符串(含 "auto")。
    yaml 里登记 + (force 或任务级 sticky 命中) → 返回 HubEndpoint;否则 None,上层走旧路径。
    """
    if llm not in _hub_models():
        return None
    if not force:
        # 本期灰度单位是 summary 任务,summary_id 是唯一 sticky key。
        # 没有 summary_id 的后台/工具调用默认不参与 Hub 灰度,避免所有无 id 调用共用空 key。
        if not summary_id:
            return None
        if not _hit(f"chat:{llm}:{summary_id}"):
            return None
    logger.info(f"hub_rollout hit, summary_id={summary_id}, llm={llm}")
    return _chat(llm)

@functools.lru_cache(maxsize=64)
def _chat(llm: str) -> HubEndpoint:
    # 不传 session_id: Hub 内部每次调用按 YAML 权重重新选择 actual model / endpoint。
    # summary_id 只用于“是否进入 Hub”的任务级灰度,不用于 Hub 内部 endpoint 粘性。
    return HubEndpoint(
        name=f"hub:summary:{llm}",
        logical_model=llm,
        model_name=_model_name_for_logical(llm),
        llm=ChatModelHub(
            app_id="summary",
            env=get_model_hub_env(),
            model=llm,
            config_store=get_model_hub_config_store(),
        ),
    )

def route_model_endpoint(
    content,
    model_name: ModelName,
    summary_id=None,
    force_hub: bool = False,
    fallback_endpoint: HubEndpoint | None = None,
    **dispatch_kwargs,
):
    """ModelName 入口:Hub 命中则返回 HubEndpoint,否则走旧 dispatcher。

    force_hub=True 用于 Hub 路径的外层业务重试:如果目标模型不在 Hub yaml,
    也不能回落旧 dispatcher,只能返回 fallback_endpoint 或 None。
    """
    logical_model = _logical_for_model_name(model_name)
    if (ep := try_hub(logical_model, summary_id, force=force_hub)):
        return ep
    if force_hub:
        logger.warning(
            f"hub forced retry miss, summary_id={summary_id}, model_name={model_name.value}, logical_model={logical_model}"
        )
        return fallback_endpoint
    return EndpointDispatcher.get_instance().dispatch(
        content, model_name=model_name, summary_id=summary_id, **dispatch_kwargs
    )

def try_hub_embedding(sticky: str | None = None) -> HubEndpoint | None:
    if "text-embedding" not in _hub_models():
        return None
    # embedding 也按任务灰度。没有 summary_id/sticky key 的后台脚本本期不进 Hub。
    if not sticky:
        return None
    if not _hit(f"emb:{sticky}"):
        return None
    return _emb()

@functools.lru_cache(maxsize=1)
def _emb() -> HubEndpoint:
    return HubEndpoint(
        name="hub:summary:text-embedding",
        logical_model="text-embedding",
        embedding_llm=EmbeddingsModelHub(
            app_id="summary",
            env=get_model_hub_env(),
            model="text-embedding",
            config_store=get_model_hub_config_store(),
        ),
    )
```

### 3.3.1 Model Hub SDK 前置修复验收

结合 `/Users/liangzhu/Documents/work/plaud-model-hub` 当前代码审核,Model Hub SDK 的三个前置项状态如下:

| 项 | 当前状态 | 结论 |
|---|---|---|
| `ChatModelHub.with_config(...)` 保持 LangChain Runnable 语义 | 已修复,参考仓库最近提交包含 `fix/chat-model-with-config-semantics` | 可验收,但 summary 侧仍要补回归测试 |
| 非流式响应 token metadata | `AIMessage.usage_metadata` 与 `ChatResult.llm_output.token_usage` 已写入 | 可验收,但 summary 侧要确认 callback 聚合结果 |
| 多模态 `HumanMessage(content=[text,image_url])` 透传 | 未修复。当前 `_get_message_content()` 仍只拼 text,丢弃 `image_url`;ADR-007 还是“提议中”文档 | Phase 0 阻塞。图片 mark 不得进 Hub |

SDK 修复完成后,`plaud-summary` 侧**不再做兼容绕行**。验收标准仍是:

1. `ChatModelHub.with_config(...)` 必须保持 LangChain Runnable 语义,不能吞掉 `run_name` / `metadata` / `callbacks` / `tags` 等运行配置。
2. `HumanMessage(content=[{"type": "text", ...}, {"type": "image_url", ...}])` 必须原样透传到 Model Hub/Core/Provider,不能被压扁成纯文本。
3. `ChatModelHub` 非流式响应必须带 token 元数据(PR #17 已提供): `AIMessage.usage_metadata.input_tokens / output_tokens / total_tokens`,以及 `ChatResult.llm_output.token_usage.prompt_tokens / completion_tokens / total_tokens`。本项目没有 streaming 总结链路,因此本期只验收非流式 token 统计;stream chunk 是否带 usage 不作为本次迁移阻塞项。

`plaud-summary` 仍然必须做三件事:

- 升级并锁定已修复的 `model-hub-sdk` 版本,不要使用未固定版本。
- 增加接入验收测试:主 summary 链路验证 `with_config(run_name=..., metadata=...)` 不丢;图片 mark 链路验证 `image_url` payload 到达 Hub transport/provider mock。
- 灰度前本地跑通四条路径:`ChatModelHub("gpt-5")` / `ChatModelHub("auto")` / `EmbeddingsModelHub("text-embedding")` / 图片 mark 多模态调用;同时确认 summary 结果里的 `tokens.prompt_tokens / completion_tokens / total_tokens` 在 Hub 路径非 0。

结论:SDK 修复后,summary 侧不需要改业务调用写法,但需要升级依赖和补验证。多模态未修复前,图片 mark 不进 Hub 灰度;若使用未锁定或早于 `with_config` 修复的版本,`run_name` / metadata / callback 相关观测不能作为可信数据。token 统计优先复用 `ChatModelHub` 已返回的 `usage_metadata` / `llm_output.token_usage`;如果现有 `get_*_callback()` 不能自动聚合这些字段,只补一个 Hub 专用轻量 callback/wrapper 从 LangChain 结果里累加 token,不再回退到旧 provider 类型判断。

**关键设计**:

- `try_hub(llm, summary_id)` 一个函数搞定所有 chat 入口(显式模型 + AUTO 都走它,AUTO 也是 yaml 里的一个 logical 条目 `summary:auto`)
- 灰度单位是 **summary 任务**:只有传入 `summary_id` 的调用才参与 Hub 灰度;同一个 `summary_id` 要么全程走 Hub,要么全程走旧 Dispatcher。
- `summary_id` **只用于是否进入 Hub 的任务级灰度**,不传给 `ChatModelHub.session_id`。Hub 内部 `summary:auto` 的 actual model 选择、多 endpoint 的 region/endpoint 选择,每次调用都按 Hub YAML 权重重新随机,这是预期行为。
- `summary:auto` 的模型分布以 Hub YAML 为准。本次迁移允许顺手调整 AUTO 内部模型 mix;旧 Dispatcher 的 `model_auto_dispatch_config` 不再作为 Hub 路径 AUTO 分布的真理源。
- **没有任何 Hub 接管模型清单或路由元数据表** —— 加新模型 = Hub yaml 加一条 entry,代码不改一行。代码里只允许保留两类兼容映射:旧客户端 `LLMName` → Hub logical model;Hub logical model ↔ 旧 `ModelName` 字段兼容。
- Hub 模型清单从共享 `ConfigStore` 的当前快照派生,不做永久 `lru_cache`;Hub YAML profile 热更新后,删条目 / 改权重不需要重启进程。

### 3.3.2 AUTO 目标分布

Hub 路径的 `summary:auto` 不追求与旧 Dispatcher 的 AUTO 分布等价,而是以 Hub YAML 中 `summary:auto.endpoints[*].weight` 为目标分布。也就是说,如果某个 endpoint 在旧 `model_auto_dispatch_config` 中 hits=0,但在 Hub YAML 中 `enabled: true` 且 `weight > 0`,它就会拿到 Hub AUTO 流量;这是本次模型 mix 调整的一部分。

当前 `conf/dev-endpoints-model-hub.yaml` 的 `summary:auto` 目标分布为:

| actual model | weight | Hub AUTO 占比 |
|---|---:|---:|
| `gemini-2.5-pro` | 66 | 64.7% |
| `gpt-5` | 34 | 33.3% |
| `gpt-4.1` | 1 | 1.0% |
| `o3` | 1 | 1.0% |

因此 YAML 注释必须与配置一致:不要写“`gpt-4.1 / o3` hits=0,禁用占位”。应明确它们是 Hub AUTO 新增的小流量探索模型。`gpt-5.1` 是否进入 Hub AUTO 也以目标分布为准;如果本期不放量,可以不配置或配置为 `enabled: false`。

### 3.4 入口改造:统一路由入口 + 重点调用点

所有持有 `EndpointDispatcher` 的入口原则上都同一种改法 —— 先问 Hub,Hub miss 再走旧路径。但不要在业务文件里散落重复判断,建议在 `hub_rollout.py` 暴露统一 helper,业务侧只调用 helper:

```python
endpoint = route_model_endpoint(
    content, model_name=ModelName.GPT_5, summary_id=summary_id
)
```

这样 Phase 1 改的是“所有直接 dispatcher 调用点改用统一 helper”,而不是每处手写一段 walrus。

#### `summary.py:get_endpoint_for_llm`(主入口)

[summary.py:2256](../../../../../Documents/work/plaud-summary/plaud_summary/summary.py#L2256) 入口最顶端加一行:

```python
from plaud_summary.plaud.endpoint_manager.hub_rollout import try_hub

_LEGACY_HUB_LLM_ALIASES = {
    LLMName.GPT_4O_OLD.value: ModelName.GPT_5.value,  # "openai" 旧值
    LLMName.GPT_4O.value: ModelName.GPT_5.value,
    LLMName.GPT_4_1.value: ModelName.GPT_5.value,
    LLMName.GEMINI_3_PRO.value: "gemini-3.1-pro",      # 当前枚举旧值仍是 "gemini-3-pro"
}

def _normalize_hub_llm(llm: str) -> str:
    """把旧客户端仍可能传入的过期 LLMName 归一化为 Hub yaml 中的 logical model。"""
    return _LEGACY_HUB_LLM_ALIASES.get(llm, llm)

def get_endpoint_for_llm(content, llm, scenario, language, summary_id=None):
    # 先做旧客户端兼容归一化,再问 Hub。注意:这里不能先初始化 EndpointDispatcher。
    llm = _normalize_hub_llm(llm)

    # ★ Hub 统一接管(含 "auto"、含未在 LLMName 枚举的 logical 字符串如 "gemini-3.1-pro")
    if (ep := try_hub(llm, summary_id)):
        return ep

    # ↓↓↓ 以下是旧 dispatcher 全部分支,保持不变 ↓↓↓
    ...
```

拿到 endpoint 后的业务调用面(`endpoint.llm.invoke / .ainvoke / .with_structured_output / .bind_tools`)保持不改;但所有直接 `EndpointDispatcher.dispatch(...)` 的入口要改到 `route_model_endpoint(...)`,否则灰度覆盖不完整。

#### `plaud/utils.py`(embedding 入口)

[`utils.py:192 create_embeddings`](../../../../../Documents/work/plaud-summary/plaud_summary/plaud/utils.py#L192) / [`utils.py:202 create_embeddings_endpoint`](../../../../../Documents/work/plaud-summary/plaud_summary/plaud/utils.py#L202) 各加一行:

```python
from plaud_summary.plaud.endpoint_manager.hub_rollout import try_hub_embedding

def create_embeddings(summary_id: str, text: str = "") -> OpenAIEmbeddings:
    if (ep := try_hub_embedding(sticky=summary_id)): return ep.embedding_llm   # ★
    return EndpointDispatcher.get_instance().dispatch_embedding(len(text or "")).embedding_llm

def create_embeddings_endpoint(summary_id: str, text: str = ""):
    if (ep := try_hub_embedding(sticky=summary_id)): return ep                  # ★
    return EndpointDispatcher.get_instance().dispatch_embedding(len(text or ""))
```

> `zilliz_region_sync.py` / `community/search/milvus_clients.py` 是后台脚本,直接调 `dispatch_embedding()` 绕开 helper,本期不动。

#### `basic_runnable_summary.py` 外层业务重试(必须处理)

Model Hub 内部会做 provider/endpoint/fallback 层面的重试,但 `basic_runnable_summary.py` 还有一层 summary 业务重试:LLM 调用成功后,解析失败、结果过短、业务校验失败等异常也会进入这里。

因此 Hub 首次命中后,外层业务重试不能再调用旧 `EndpointDispatcher`。否则同一个 summary 任务会变成“第一次 Hub,第二次旧 dispatcher”,违反关键不变量。

改法:在 `except` 里先判断当前 endpoint 是否 Hub:

```python
cur_endpoint = self.endpoint
is_hub_path = getattr(cur_endpoint, "is_hub", False)

if is_hub_path:
    # Hub 路径:不做 Gemini regional -> global 的旧 dispatcher fallback。
    # 外层业务重试可选择继续使用同一 logical model,或用 _select_retry_model()
    # 得到的新 ModelName 再通过 route_model_endpoint(force_hub=True) 继续走 Hub。
    if get_aws_config("USE_NEW_RETRY", False):
        selected_model = self._select_retry_model()
        self.endpoint = route_model_endpoint(
            text,
            model_name=selected_model,
            summary_id=self.summary_id,
            force_hub=True,
            fallback_endpoint=cur_endpoint,
        )
    else:
        self.endpoint = (
            try_hub(cur_endpoint.logical_model, self.summary_id, force=True)
            or cur_endpoint
        )
else:
    # ↓↓↓ 保留原旧 dispatcher 重试逻辑 ↓↓↓
    dispatcher = EndpointDispatcher.get_instance()
    ...
```

关键点:

- Hub 暴露异常不等于 Hub 有问题;它可能是所有 fallback 都失败,也可能是业务层后处理异常。
- Hub endpoint 的外层业务重试必须继续经由 Hub 分流 helper,不能直接 `dispatcher.dispatch(...)`。
- 旧 Gemini regional 429 → global endpoint 逻辑只适用于旧 `EndpointType.GEMINI`;Hub 路径交给 Hub 的 endpoint failover / fallback / adaptive weight。

#### `knowledge_base_note.py` / `poster_generator.py`(自持 dispatcher)

这两个文件自己 `EndpointDispatcher.get_instance()`,绕过 `get_endpoint_for_llm` 直接调 `dispatch(...)`。这里不要再手写 walrus,统一改成调用 `route_model_endpoint(...)`:

```python
# knowledge_base_note.py:1003 / 1120 / 1167 三处
endpoint = route_model_endpoint(
    content=self.text, model_name=model, summary_id=self.summary_id
)

# poster_generator.py:1445
if model_name_enum is not None:
    endpoint = route_model_endpoint(
        content="", model_name=model_name_enum, summary_id=self.summary_id, ...
    )
```

> **关键**:Phase 1 上线前必须 `grep -rn "dispatcher\.dispatch\|EndpointDispatcher" plaud_summary/` 兜底核对所有直接调用点。主 summary 链路和外层业务重试必须改到统一 helper;非主链路可列入 Phase 2,但 Phase 3 删旧 dispatcher 前必须清零这些调用点。

本次审核实际跑 `rg -n "EndpointDispatcher|get_instance\\(\\)|\\.dispatch\\(|dispatch_auto_model\\(|dispatch_embedding\\(|dispatch_by_endpoint_prefixes\\(" plaud_summary -S` 后,需要补充以下调用点分级:

**Phase 1 必须处理(会影响线上 summary/mark 主链路或同一任务重试不变量):**

- `plaud_summary/summary.py:618` mark note 子链路直接 dispatch;若图片多模态 SDK 未修复,这里必须显式保持旧路径,不能误进 Hub。
- `plaud_summary/mark_note/mark_note_processor.py:51,238` 和 `plaud_summary/mark_note/mark_note_post_processor.py:53` 是 mark note 独立链路,其中图片 mark 依赖 `image_url` 多模态,SDK 修复前不进 Hub。
- `plaud_summary/mark_note/mark_note_summary.py` 内部大量 `self.endpoint.llm` 调用不需要重选 endpoint,但入口 endpoint 来源必须和 mark note 主任务一致。
- `plaud_summary/plaud/basic_runnable_summary.py:769,1210` 超出普通 retry 分支的 Claude timeout fallback、标题重试也会重新 dispatch。Hub 路径必须继续走 `route_model_endpoint(force_hub=True)` 或复用当前 HubEndpoint,不能回旧 Dispatcher。
- `plaud_summary/plaud/knowledge_notes/knowledge_base_note.py:1003,1014,1120,1167` 和 `knowledge_base_note_short.py:403,462` 有自己的 step fallback / Gemini global retry,需要按 `is_hub` 分支处理,否则 knowledge pipeline 会混用 Hub 与旧 Dispatcher。

**Phase 1 需二选一:接入 helper,或明确标注“不纳入本期覆盖范围”:**

- `plaud_summary/chains/self_check_chain.py`
- `plaud_summary/chains/enable_result_chain.py`
- `plaud_summary/chains/re_act_chain.py`
- `plaud_summary/chains/model_name_review_chain.py`
- `plaud_summary/chains/compare_results.py`
- `plaud_summary/features/persona/persona_service.py`
- `plaud_summary/trans_compress/service.py`
- `plaud_summary/plaud/ai_router_note.py`
- `plaud_summary/services/summary_card/poster_generator.py`

**Phase 1 不处理但要登记为 Phase 2/Phase 3 清理项:**

- `tests/`、`scripts/`、`frontend/`、`community/` 下的手工测试和离线脚本。
- `config_handler.py` 的 `load_localfile(...)` 属于旧 Dispatcher 配置加载入口,只在 Phase 3 删除旧 Dispatcher 时处理。

如果产品目标坚持“覆盖所有 chat 模型 + AUTO + Embedding”,上述 Phase 1 调用点不能留白;如果为了降低风险,建议把目标文案改成“主 summary 主链路 + AUTO + embedding 首批接入,mark/persona/trans-compress 等 chat 子链路 Phase 2 接入”。

---

## 四、放量与回滚

### 4.1 放量

```
summary_hub_rollout: 0.0    # 默认全部走旧路径
                  → 0.01    # 1% 灰度
                  → 0.1
                  → 0.5
                  → 1.0     # 全切 Hub
```

`summary_hub_rollout` 是 AWS 配置中心主配置里的 float key,通过 `plaud_library_python.configpkg.config.get_aws_config` 读取,随主配置 watchdog 热更新生效。灰度按 `summary_id` 做任务级 sticky hash —— 同一 summary 任务重试期间路径不变。没有 `summary_id` 的调用本期默认不进入 Hub;后续要迁后台/工具链路时,必须为该链路显式定义稳定任务 key(如 `task_id` / `file_id` / `mark_note_id`)。

注意:这里的 sticky 只决定“这个任务是否进入 Hub”。进入 Hub 后,`summary:auto` 的 actual model 选择和多 endpoint 的 endpoint/region 选择不做 `summary_id` 粘性,每次 LLM 调用按 Hub YAML 权重重新选择。

### 4.2 回滚

`summary_hub_rollout = 0.0` → 主配置被 watchdog 拉到后,全量新请求回旧路径。当前 AppConfig client 默认约 120s 轮询,因此不要承诺严格秒级。

### 4.3 单独控制某个模型

**改 Hub yaml**:把该 logical 条目从 `models.summary:<logical_model>` 删掉(或注释)→ Model Hub profile 被 watchdog 刷新后,`_hub_models()` 不再包含它 → 该模型只走旧路径,不影响其他模型。

> **设计取舍**:放量比例(运行时旋钮)在主 AppConfig;模型登记在 Hub yaml profile。代码里**不复读一份模型清单**,避免代码与 yaml 不一致这种类别的 bug。

---

## 五、阶段计划

| Phase | 内容 | 时长 |
|---|---|---|
| **0** | ① 锁定已修复 LangChain adapter 的 `model-hub-sdk` 版本。② 本地 dev 跑通四条路径(`ChatModelHub("gpt-5") / ("auto") / EmbeddingsModelHub("text-embedding")` / 图片 mark 多模态),并确认非流式 `ChatModelHub` 返回的 token 能落到 summary 结果 `tokens`。③ **阻塞**:参考仓库当前多模态仍会丢 `image_url`,必须先合入并发布 ADR-007 对应代码;否则图片 mark 从 Phase 1 范围剔除。④ **已完成但需复核旧名映射**:Hub yaml 命名对齐前端字符串(`summary:claude-sonnet-4.5` / `summary:claude-sonnet-4.6` / `summary:gemini-3.1-pro` / `summary:gemini-3-flash`),同时代码中 `gemini-3-pro` 要归一化为 `gemini-3.1-pro`。⑤ 验证所有线上活跃模型(`gpt-5/5.2/5.5`、`claude-sonnet-4.5/4.6`、`gemini-2.5-pro/3.1-pro`、`auto`、`text-embedding`)在 Hub yaml 中已登记且 `fallback_model` 降级链已配齐。⑥ 确认 `summary:auto` 目标分布与 YAML 注释一致,并在监控中能按 actual model / endpoint_id 切分。⑦ 脱敏 `conf/dev-endpoints-model-hub.yaml`,移除明文默认凭证,并确保本地配置不会被误提交 | 1~2 天 |
| **1** | 落 3 个新文件 + `EndpointType.HUB` + 主入口/embedding/自持 dispatcher/外层业务重试改造;补齐本次审核列出的直接 dispatcher 调用点分级处理;Hub 路径跳过本地 cost 计算;`summary_hub_rollout=0.0` 上线;验证导入与构造,行为不变 | 1~2 天 |
| **2** | 灰度爬坡 0.01 → 0.1 → 0.5 → 1.0,每档 24~72h;按 `logical_model` / `actual_model` / `endpoint_id` 切面看监控 | 1~2 周 |
| **3** | 100% 后,**一次性删干净旧路径**:`EndpointDispatcher` 整个类、`_pool` / 各 `_xx_pool` / `_parse_*_endpoints` / `_create_generic_llm` / `dispatch` / `dispatch_auto_model` / `dispatch_embedding` / `dispatch_by_endpoint_prefixes` / `_build_model_pools`;AWS Secrets `*_endpoints` JSON 全部下线;`llm.yaml model_registry` 字段变为前端展示用 | 2~3 PR |
| **4**(独立)| CN 模型迁移(等 Hub 侧落 `ChatPlaudAI` 内容安全豁免 plugin) | 后续评估 |

---

## 六、可观测性

### 6.1 指标切面

按 `path=old|hub` × `logical_model` 切:QPS / p50,p95,p99 latency / 错误率 / token。cost 先不作为灰度阻塞指标,只保留旧路径现有口径;Hub 路径成本以后由 Model Hub 统一提供。对 `logical_model=auto` 必须额外按 Hub 返回的 `actual_model` / `endpoint_id` 切,否则看不到本次 AUTO 模型 mix 调整带来的质量、延迟、token 消耗变化。

### 6.2 Token / Cost 归属

- **旧路径**:现有 `tokens_cost` 与 Langfuse 统计不变。
- **Hub 路径 token**:继续保留 `tokens_cost` 返回结构,至少回填 `prompt_tokens / completion_tokens / total_tokens / user_content_tokens / final_output_tokens`。LLM token 来源优先使用 `ChatModelHub` 非流式返回的 `AIMessage.usage_metadata` 或 `ChatResult.llm_output.token_usage`;`reasoning_tokens / cached_tokens` 在 Hub SDK 未显式返回前填 0。
- **Hub 路径 cost**:不调用 Summary 旧 `calculate_llm_cost(endpoint=HubEndpoint, ...)`,不使用本地价格表估算 `ChatModelHub` 成本。`total_cost_in_usd` 暂填 0 或 None,并在日志 / 监控属性里标记 `path=hub`、`cost_source=not_calculated`。
- 这样上游依赖 `tokens_cost` 读取 token 的用户不受影响;依赖 cost 的报表在灰度期按 old / hub 分开看。后续如果需要准确 Hub cost,由 Model Hub 基于最终 `actual_model + endpoint_id` 统一返回。

当前 `basic_runnable_summary.py` 的实际代码会在 callback 后无条件执行:

```python
calculate_llm_cost(endpoint=self.endpoint, tokens_cost=self.tokens_cost, update_tokens_cost=True)
```

这行必须改成:

```python
if getattr(self.endpoint, "is_hub", False):
    self.tokens_cost.total_cost = 0
    # 监控属性中标记 cost_source=not_calculated
else:
    calculate_llm_cost(endpoint=self.endpoint, tokens_cost=self.tokens_cost, update_tokens_cost=True)
```

否则 HubEndpoint 的 `model_name` 兼容字段会让旧价格表产生一个看似真实但实际归属错误的成本。

### 6.3 必打日志

`get_endpoint_for_llm` 每次返回前打 `summary_id / llm / path=old|hub / endpoint_name`。Hub Langfuse / Hub telemetry 侧必须能查到 `logical_model / actual_model / endpoint_id`,用于验证 `summary:auto` 实际分布。summary 结果日志里保留 `tokens.prompt_tokens / completion_tokens / total_tokens`,Hub 路径额外带 `cost_source=not_calculated`,避免把 0 成本误解成真实免费。

---

## 七、不变量

1. **HubEndpoint 不进旧 `_pool`** —— HubEndpoint 只由 `try_hub` / `route_model_endpoint` 返回,不追加到 `EndpointDispatcher._pool` / `_model_pools` / `_embedding_pool`;如后续新增统一 add helper,必须加 `assert not getattr(ep, "is_hub", False)`。
2. **同 `summary_id` 在重试间路径不变** —— 本期灰度单位是 summary 任务;chat / embedding 入口都用 `summary_id` 做“是否进入 Hub”的 sticky key。没有 `summary_id` 的调用默认走旧路径。
3. **Hub 内部不做 summary_id 粘性** —— `ChatModelHub` / `EmbeddingsModelHub` 本期不传 `session_id`;Hub 内部 actual model / endpoint / region 每次调用按 YAML 权重重新选择。
4. **Hub 路径的外层业务重试不回落旧 Dispatcher** —— `basic_runnable_summary.py` 捕获异常后,如果 `cur_endpoint.is_hub=True`,只能继续经由 Hub 分流 helper 选 endpoint;不能直接 `EndpointDispatcher.get_instance().dispatch(...)`。
5. **Hub 路径保留 token,不算旧 cost** —— `path=hub` 时继续返回 `tokens_cost` 的 token 字段,但 `total_cost_in_usd` 不走 Summary 本地价格表;不做旧 provider callback/cost 双写。token 来自 `ChatModelHub` 非流式 `usage_metadata` / `llm_output.token_usage`,如现有 callback 不能聚合则补 Hub 专用轻量 callback/wrapper。
6. **Hub yaml 是单一真理源** —— 加/移除一个模型只改 yaml,代码零改动;**不**在代码里维护"哪些模型走 Hub"的并行清单。
7. **AUTO 分布以 Hub YAML 为准** —— Hub 路径的 `summary:auto` 不复刻旧 `model_auto_dispatch_config`;本次迁移允许通过 YAML 调整 AUTO 模型 mix。
8. **AUTO 接受时段调权降级** —— 旧 `dispatch_config.hits_schedule` / `pt_hits_scale_config` 在 Hub 没有原生支持。如果生产里这两个机制实际在用且不可丢,先在 Hub 侧补"按时段调权"插件再开 AUTO 灰度。

---

## 八、本期不做

| 项 | 原因 |
|---|---|
| CN 模型(豆包/通义/Kimi/DeepSeek) | 依赖 `ChatPlaudAI` 内容安全豁免,等 Hub 侧 plugin |
| `dispatch_auto_model` 内的 CN 区域兜底 | 与 CN 迁移绑定;CN 集群不开 `summary_hub_rollout` 即可 |
| 后台脚本 embedding(`zilliz_region_sync.py` / `milvus_clients.py`) | 直接调 `dispatch_embedding()` 绕开 helper;Phase 3 单独 PR |
| 图片 mark Hub 灰度(在 ADR-007 未发布前) | 当前 `ChatModelHub` 会丢 `image_url`;SDK 修复并验收前必须保持旧路径 |
| `Endpoint.tpm/rpm` / `_check_endpoint` 死代码清理 | 与本迁移无依赖,独立 PR |
| Hub Langfuse / 自有 Langfuse 双写 cost | 复杂度大于收益,灰度期接受报表两块看 |
| Summary 本地计算 Hub cost | Hub 才知道最终 `actual_model / endpoint_id / fallback`;本期只保留 token,成本后续由 Hub 统一提供 |
| `hits_schedule` / `pt_hits_scale_config` 时段调权 | Hub 无原生支持;接受降级(详见 §七 不变量 8) |

---

## 九、已拍板与待拍板

已拍板:

1. **灰度单位按任务** —— `summary_id` 是本期唯一 sticky key;没有 `summary_id` 的调用默认不进 Hub。
2. **Hub 内部不做任务粘性** —— 不给 `ChatModelHub` / `EmbeddingsModelHub` 传 `session_id`;Hub 内部每次调用按 YAML 权重选择 actual model / endpoint。
3. **AUTO 分布以 Hub YAML 为准** —— 本次迁移允许顺手调整 `summary:auto` 模型 mix,不要求复刻旧 `model_auto_dispatch_config`。

仍需拍板:

1. **首发入口**:chat + AUTO + embedding 同时开灰度还是分批?建议**同时开** —— 共用 `summary_hub_rollout` 一个旋钮,起步 0.01 时三类各承担 1%,监控按 `kind` 切面看,任一异常整体回滚。分批需要拆三个独立比例,复杂度上升。
2. **`hits_schedule` / `pt_hits_scale_config` 现在生产里到底有没有在生效?** 占位 → AUTO 直接迁;在用 → 先在 Hub 侧补时段调权 plugin,本期可先只开 chat + embedding。
3. **首发是否包含 mark/persona/trans-compress 等子链路?** 如果继续宣称“覆盖所有 chat 模型”,这些直接 dispatch 调用点必须进 Phase 1;如果不想扩大风险,目标需要改成“主 summary 主链路首发”,并把这些子链路列为 Phase 2。
4. **图片 mark 是否等 SDK 多模态修复后再进灰度?** 建议是。否则必须在 route helper 增加 `allow_hub=False` 或独立 mark 开关,确保图片链路固定旧路径。

---

## 十、改动文件清单

```text
新增:
  plaud_summary/plaud/endpoint_manager/hub_config.py       (Model Hub ConfigStore + AppConfig profile 热更新适配)
  plaud_summary/plaud/endpoint_manager/hub_endpoint.py     (~60 行)
  plaud_summary/plaud/endpoint_manager/hub_rollout.py      (~90 行)

修改:
  plaud_summary/summary.py                                 (旧名归一化 + 主入口 Hub 分流)
  plaud_summary/plaud/basic_runnable_summary.py             (Hub 路径外层业务重试不回落旧 dispatcher)
  plaud_summary/plaud/utils.py                             (+2 行 walrus + 1 行 import)
  plaud_summary/plaud/knowledge_notes/knowledge_base_note.py  (直接 dispatch 改 route_model_endpoint)
  plaud_summary/plaud/knowledge_notes/knowledge_base_note_short.py  (同上,或明确 Phase 2)
  plaud_summary/services/summary_card/poster_generator.py  (直接 dispatch 改 route_model_endpoint)
  plaud_summary/mark_note/mark_note_processor.py            (SDK 多模态修复前固定旧路径;修复后接 route helper)
  plaud_summary/mark_note/mark_note_post_processor.py       (同上)
  plaud_summary/chains/self_check_chain.py                  (接 route helper 或 Phase 2)
  plaud_summary/chains/enable_result_chain.py               (接 route helper 或 Phase 2)
  plaud_summary/chains/re_act_chain.py                      (接 route helper 或 Phase 2)
  plaud_summary/chains/model_name_review_chain.py           (接 route helper 或 Phase 2)
  plaud_summary/features/persona/persona_service.py         (接 route helper 或 Phase 2)
  plaud_summary/trans_compress/service.py                   (接 route helper 或 Phase 2)
  plaud_summary/plaud/ai_router_note.py                     (接 route helper 或 Phase 2)
  plaud_summary/plaud/endpoint_manager/llm_config.py       (+EndpointType.HUB)
  plaud_summary/chains/form_llm_chain.py                   (structured output 类型白名单加 HUB 或改能力判断)
  main.py / server/api.py                                  (初始化 Model Hub ConfigStore profile)
  requirements.txt / setup.py                              (+锁定已修复版本的 model-hub-sdk)
  tests 或 smoke 脚本                                      (with_config 运行配置 + 图片 mark 多模态 + Hub token 回填验收)

外部配置:
  主 AppConfig 新增 key: summary_hub_rollout (float, 默认 0.0)
  主 AppConfig 新增/确认 key: model_hub_profile (如 dev-endpoints-model-hub.yaml;也可复用既有命名)
  Model Hub YAML AppConfig profile 注册所有线上模型 + auto + text-embedding
  Model Hub YAML 不保留明文默认凭证;本地 dev 文件只允许脱敏占位或环境变量引用

不动:
  已拿到 endpoint 后的业务调用代码(call_note / meeting_* / splitters 等)
  EndpointDispatcher / parse_*_endpoints / _create_generic_llm     (Phase 3 整体删)
  dispatch / dispatch_auto_model / dispatch_embedding              (Phase 3 整体删)
  dispatch_by_endpoint_prefixes / generic_endpoints                (Phase 3 整体删)
  CN 模型相关(Phase 4)
```
