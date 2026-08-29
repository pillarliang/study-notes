# Summary 模块化 Pipeline 与区域化 Prompt 架构设计

> 状态：目标架构设计稿  
> 日期：2026-07-24  
> 适用范围：假设从零开发 Summary 业务，不受当前遗留架构和迁移兼容要求限制

![[assets/summary-modular-pipeline-architecture.png|663]]

图中的蓝色纵向箭头表示业务数据流。右侧绿色虚线区域是请求级可观测性作用域。它能读取任务元信息，但不参与 Prompt 选择、模型选择或任何业务判断。

## 一、结论

采用一个**模块化单体**。应用入口根据 `scenario` 找到对应的 Summary 模块，再由模块执行自己的 Pipeline。

每个模块可以自行决定：

- Pipeline 有哪些步骤；
- 步骤执行顺序；
- 是否进行长文本切分；
- 是否并发调用模型；
- 使用几个 Prompt；
- 是否生成标题、执行合并或结果校验；
- 是否进行业务级重试和结果修复。

所有模块只统一以下边界：

- 输入 `SummaryRequest`；
- 输出 `SummaryResult`；
- 模块注册和场景路由；
- Region 到 Prompt Edition 的启动期转换；
- 请求级超时、合规检查、日志和异常格式；
- 模型调用的基础设施能力。

任务数据严格分为三类：

1. **业务请求**：承载业务身份和业务输入，必须通过 `SummaryRequest` 显式传递；
2. **任务执行上下文**：只保存安全的关联元信息，通过 `ContextVar` 在当前任务内自动传播；
3. **模块局部状态**：只保存 Pipeline 中间结果，由模块内部的类型化对象管理。

OpenTelemetry 负责描述一次执行链路。`summary_id` 负责关联同一业务任务的多次执行。两者同时保留，不能互相替代。

第一版不建设通用工作流引擎、DAG、远程 Prompt 平台或动态插件系统。各模块的 Pipeline 直接使用 Python 编写。

---

## 二、业务目标

用户提交一段录音转写并选择摘要类型，系统根据摘要类型执行对应的处理流程，最终生成符合用户需求的笔记。

不同摘要类型本身就有不同流程，例如：

| 摘要类型           | 典型流程                                 |
| -------------- | ------------------------------------ |
| Dairy          | 渲染 Prompt → 单次 LLM 调用 → 格式化          |
| Meeting        | 文本切分 → 分段摘要 → 合并 → 标题 → 校验           |
| User Custom    | 读取用户要求 → 安全处理 → LLM 调用 → Markdown 修复 |
| Dimension Note | 提取特定维度 → 结构校验 → 输出                   |

因此，业务目标不是让所有场景服从一条统一 Pipeline，而是：

> 在共享基础设施和统一输入输出契约之上，让每个摘要模块独立表达自己的业务流程。

---

## 三、总体架构

请求路径可以简化为：

```text
SummaryRequest
  → SummaryApplication
  → 绑定任务执行上下文并创建 Root Span
  → ModuleRegistry.get(request.scenario)
  → module.run(request)
  → 模块自己的 Pipeline
  → SummaryResult
```

---

## 四、核心设计边界

### 4.1 外层统一负责什么

`SummaryApplication` 负责所有模块都必须遵守的请求级行为：

1. 绑定只读任务执行上下文并创建 Root Span；
2. 校验 `scenario`、`content`、`language`；
3. 执行统一的内容合规检查；
4. 通过 `ModuleRegistry` 找到模块；
5. 设置请求超时；
6. 记录请求级日志、耗时和错误；
7. 把模块异常转换成统一业务错误；
8. 返回统一的 `SummaryResult`。

外层不参与具体摘要步骤，也不知道模块使用了几个 Prompt 或调用了几次模型。

### 4.2 模块内部负责什么

每个模块负责自己的业务流程：

- 选择模块内部的 Prompt；
- 决定是否切分文本；
- 决定模型调用次数和先后顺序；
- 决定哪些步骤并行执行；
- 决定如何合并结果；
- 决定输出校验和业务级重试；
- 组装最终 `SummaryResult`。

### 4.3 共享能力负责什么

共享能力是模块可以调用的小工具，而不是固定 Pipeline：

| 共享能力 | 职责 | 不负责 |
| --- | --- | --- |
| `PromptCatalog` | 加载、缓存、校验 Prompt | 决定业务流程 |
| `LLMClient` | 调用逻辑模型，处理基础设施错误 | 决定业务级重试 |
| `TextSplitter` | 按 Token 切分文本 | 决定是否需要切分 |
| `OutputValidator` | 提供结构、长度等校验函数 | 决定校验失败后的业务动作 |
| Metrics | 记录步骤耗时、Token、成功失败 | 改变模块执行逻辑 |

### 4.4 三类数据必须分开

状态设计的核心不是创建一个无所不包的 `state`，而是根据数据的职责选择传播方式：

| 数据类别 | 典型内容 | 传播方式 | 生命周期 |
| --- | --- | --- | --- |
| 业务请求 | `summary_id`、`file_id`、`scenario`、`content`、`language` | 通过 `SummaryRequest` 显式传递 | 整个摘要任务 |
| 任务执行上下文 | `summary_id`、`file_id`、`scenario`、`request_id` | 入口绑定到 `ContextVar`，日志与 Trace 自动读取 | 当前一次执行 |
| 模块局部状态 | chunks、分段结果、合并结果、业务重试次数 | 模块内部类型化对象 | 当前模块调用 |

业务模块只能从 `SummaryRequest` 读取业务身份和会影响结果的数据。任务执行上下文只服务于日志和 Trace 关联，不能成为业务判断的隐式输入。模块局部状态不得写入 `ContextVar`，否则并发分支会共享可变对象，数据流也会失去可读性。

> [!important] `summary_id` 与 `trace_id` 的语义不同
> `summary_id` 是稳定业务身份，可跨重试、重新入队和多次执行保持不变。`trace_id` 标识某一次执行链路，重新消费通常产生新的值。排查问题时先用 `summary_id` 找到全部执行历史，再用 `trace_id` 展开其中一次调用链。

---

## 五、建议目录结构

采用按业务场景纵向组织的目录。一个模块相关的代码、Prompt 和测试尽量放在一起。

```text
plaud_summary/
  summary/
    core/
      contracts.py             # SummaryRequest、SummaryResult、SummaryModule
      application.py           # 统一用例入口
      registry.py              # scenario → module
      errors.py                # 统一业务错误

    observability/
      context.py               # 只读任务执行上下文及绑定作用域
      logging.py               # 自动注入任务关联字段
      tracing.py               # Root Span、步骤 Span 与属性规范
      metrics.py               # 请求级和步骤级低基数指标

    shared/
      prompts.py               # PromptCatalog
      llm.py                   # LLMClient 协议
      text_splitter.py         # 可复用切分工具
      validation.py            # 可复用结果校验

    modules/
      dairy/
        module.py              # DairyModule
        pipeline.py            # Dairy 具体流程
        prompts/
          manifest.yaml
          cn/
            summary.md
          global/
            summary.md
        tests/
          test_pipeline.py

      meeting/
        module.py              # MeetingModule
        pipeline.py            # Meeting 具体流程
        prompts/
          manifest.yaml
          cn/
            summary.md
            merge.md
            title.md
          global/
            summary.md
            merge.md
            title.md
        tests/
          test_pipeline.py

      user_custom/
        module.py
        pipeline.py
        prompts/
        tests/
```

这样新增一个模块时，不需要修改其他模块内部代码。

---

## 六、统一输入输出与任务上下文

### 6.1 业务输入输出契约

模块之间只统一输入输出，不统一内部步骤。

```python
from dataclasses import dataclass, field
from types import MappingProxyType
from typing import Any, Mapping, Protocol


@dataclass(frozen=True, slots=True)
class SummaryRequest:
    """摘要模块的统一输入。"""

    summary_id: str
    file_id: str
    scenario: str
    content: str
    language: str
    options: Mapping[str, Any] = field(default_factory=dict)

    def __post_init__(self) -> None:
        """复制并冻结场景参数，避免调用方在执行期间修改。"""

        object.__setattr__(
            self,
            "options",
            MappingProxyType(dict(self.options)),
        )


@dataclass(frozen=True, slots=True)
class SummaryResult:
    """摘要模块的统一输出。"""

    markdown: str
    title: str | None = None
    extra: Mapping[str, Any] = field(default_factory=dict)

    def __post_init__(self) -> None:
        """复制并冻结扩展结果，避免返回后被意外修改。"""

        object.__setattr__(
            self,
            "extra",
            MappingProxyType(dict(self.extra)),
        )


class SummaryModule(Protocol):
    """所有摘要模块必须实现的最小协议。"""

    @property
    def scenario(self) -> str:
        """返回模块负责的摘要场景。"""
        ...

    async def run(self, request: SummaryRequest) -> SummaryResult:
        """执行模块自己的摘要流程。"""
        ...
```

`options` 用来承载少量确实存在的场景参数，例如是否保留时间戳。不要把所有历史参数一次性塞入该对象；只有出现真实消费者时才增加字段或类型。

### 6.2 只读任务执行上下文

任务执行上下文是一次执行的关联元信息，不是业务数据容器：

```python
from contextlib import contextmanager
from contextvars import ContextVar
from dataclasses import dataclass
from collections.abc import Iterator


@dataclass(frozen=True, slots=True)
class SummaryExecutionContext:
    """当前摘要任务的安全关联元信息。"""

    summary_id: str
    file_id: str
    scenario: str
    request_id: str | None = None


current_summary_context: ContextVar[
    SummaryExecutionContext | None
] = ContextVar("current_summary_context", default=None)


@contextmanager
def bind_summary_context(
    context: SummaryExecutionContext,
) -> Iterator[None]:
    """在当前执行作用域内绑定任务上下文。"""

    token = current_summary_context.set(context)
    try:
        yield
    finally:
        current_summary_context.reset(token)
```

上下文必须满足以下约束：

- 使用不可变类型，避免并发分支修改共享状态；
- 只保存可安全写入日志和 Span 的低体积元信息；
- 不保存 transcript、Prompt、模型输出、数据库连接或 Service 对象；
- 每次绑定都保存 `Token`，并在 `finally` 中恢复旧值；
- 没有上下文时，日志组件应正常输出，而不是阻断业务。

---

## 七、应用入口与模块路由

### 7.1 ModuleRegistry

```python
class ModuleRegistry:
    """保存摘要场景和模块的对应关系。"""

    def __init__(self, modules: list[SummaryModule]) -> None:
        self._modules = {
            module.scenario: module
            for module in modules
        }

    def get(self, scenario: str) -> SummaryModule:
        """取得负责指定场景的摘要模块。"""

        try:
            return self._modules[scenario]
        except KeyError as exc:
            raise UnsupportedScenarioError(scenario) from exc
```

第一版使用显式注册，不扫描 Python 包，也不动态加载插件：

```python
registry = ModuleRegistry(
    modules=[
        DairyModule(llm_client, prompt_catalog),
        MeetingModule(llm_client, prompt_catalog, text_splitter),
        UserCustomModule(llm_client, prompt_catalog),
    ]
)
```

显式注册容易阅读、测试和排查，也能在服务启动时发现重复的 `scenario`。

### 7.2 SummaryApplication

```python
from opentelemetry.trace import Tracer


class SummaryApplication:
    """摘要业务的统一应用入口。"""

    def __init__(
        self,
        registry: ModuleRegistry,
        request_guard: RequestGuard,
        tracer: Tracer,
    ) -> None:
        self._registry = registry
        self._request_guard = request_guard
        self._tracer = tracer

    async def generate(
        self,
        request: SummaryRequest,
        *,
        request_id: str | None = None,
    ) -> SummaryResult:
        """在统一执行作用域内运行对应摘要模块。"""

        execution_context = SummaryExecutionContext(
            summary_id=request.summary_id,
            file_id=request.file_id,
            scenario=request.scenario,
            request_id=request_id,
        )

        with bind_summary_context(execution_context):
            with self._tracer.start_as_current_span(
                "summary.generate"
            ) as span:
                span.set_attribute("summary.id", request.summary_id)
                span.set_attribute("file.id", request.file_id)
                span.set_attribute("summary.scenario", request.scenario)

                self._request_guard.validate(request)
                module = self._registry.get(request.scenario)
                return await module.run(request)
```

`SummaryApplication` 是任务上下文和 Root Span 的唯一绑定点。任务上下文先于业务校验绑定，因此校验失败日志也能关联任务。HTTP、Kafka 或内部 RPC Adapter 在调用前恢复 W3C Trace Context，并把 `request_id` 传给该入口。业务模块仍显式接收 `SummaryRequest`，不会从 `ContextVar` 偷读业务数据。

超时、请求级指标和异常转换也放在这个作用域外围，但不要把模块内部步骤搬到应用层。

---

## 八、模块自定义 Pipeline

### 8.1 Dairy：简单单次调用

```text
读取 summary Prompt
  → 填充 content、language
  → 调用 summary-default 逻辑模型
  → 清理 Markdown
  → 返回 SummaryResult
```

```python
class DairyModule:
    """日记摘要模块。"""

    scenario = "dairy"

    def __init__(
        self,
        llm_client: LLMClient,
        prompts: PromptCatalog,
    ) -> None:
        self._llm_client = llm_client
        self._prompts = prompts

    async def run(self, request: SummaryRequest) -> SummaryResult:
        """执行日记摘要的单次调用流程。"""

        template = self._prompts.get("dairy", "summary")
        prompt = template.format(
            content=request.content,
            language=request.language,
        )
        markdown = await self._llm_client.generate(
            logical_model="summary-default",
            prompt=prompt,
        )
        return SummaryResult(markdown=normalize_markdown(markdown))
```

### 8.2 Meeting：长文本并行处理

```text
判断文本长度
  ├─ 短文本 → 直接摘要
  └─ 长文本 → 切分 → 并行分段摘要 → 合并
                                  ↓
                              生成标题
                                  ↓
                              校验输出
```

```python
class MeetingModule:
    """会议摘要模块。"""

    scenario = "meeting"

    async def run(self, request: SummaryRequest) -> SummaryResult:
        """执行会议摘要的分段、合并和标题流程。"""

        chunks = self._text_splitter.split(request.content)

        if len(chunks) == 1:
            markdown = await self._summarize(chunks[0], request.language)
        else:
            partial_results = await asyncio.gather(
                *[
                    self._summarize(chunk, request.language)
                    for chunk in chunks
                ]
            )
            markdown = await self._merge(
                partial_results,
                request.language,
            )

        title = await self._generate_title(markdown, request.language)
        self._validate(markdown)

        return SummaryResult(
            markdown=markdown,
            title=title,
        )
```

### 8.3 User Custom：用户自定义指令

```text
读取用户指令
  → 校验长度和危险占位符
  → 拼接系统约束
  → 调用模型
  → 修复 Markdown
  → 返回结果
```

这个模块可以完全不使用 Meeting 的切分、合并和标题流程。模块之间不通过继承共享流程，只复用确实通用的小工具。

### 8.4 复杂模块使用局部 Pipeline State

简单模块直接使用局部变量。只有复杂模块确实需要共享多个中间结果时，才定义模块专属的 State：

```python
@dataclass(slots=True)
class MeetingPipelineState:
    """Meeting Pipeline 在单次调用内产生的中间结果。"""

    chunks: tuple[str, ...]
    partial_results: list[str] = field(default_factory=list)
    markdown: str | None = None
    title: str | None = None
```

这个对象只在 `MeetingModule.run()` 及其私有方法之间流转。它不进入 `ContextVar`，不暴露给其他模块，也不作为通用 Pipeline 框架的基础类。

> [!warning] 不要创建万能 State
> 把请求参数、执行元信息、Service 对象和中间结果全部塞进同一个字典，会让依赖关系变成隐式读写。结果是字段来源无法追踪，并发分支容易互相污染，测试也只能构造庞大的伪 State。

---

## 九、Prompt 的区域化管理

### 9.1 Region 只在启动期转换一次

```python
def resolve_prompt_edition(aws_region: AwsRegion) -> PromptEdition:
    """将部署区域转换成 Prompt 内容版本。"""

    if aws_region is AwsRegion.CN_NORTHWEST_1:
        return PromptEdition.CN
    return PromptEdition.GLOBAL
```

组合根负责创建 `PromptCatalog`：

```python
edition = resolve_prompt_edition(settings.aws_region)
prompt_catalog = PromptCatalog.load_all(
    modules_root=Path("summary/modules"),
    edition=edition,
)
```

之后模块只调用：

```python
self._prompts.get("meeting", "summary")
```

模块不知道当前是 CN 还是 global，也不读取 Region。

### 9.2 每个模块维护自己的 Manifest

例如 `modules/meeting/prompts/manifest.yaml`：

```yaml
prompts:
  summary:
    variables:
      - content
      - language

  merge:
    variables:
      - batch_results
      - language

  title:
    variables:
      - markdown
      - language
```

CN 和 global 必须提供 Manifest 中声明的全部文件：

```text
meeting/prompts/
  manifest.yaml
  cn/
    summary.md
    merge.md
    title.md
  global/
    summary.md
    merge.md
    title.md
```

### 9.3 启动时全量校验

服务启动时只加载当前 Edition，但要校验所有已注册模块：

1. Manifest 能被解析；
2. 当前 Edition 的文件全部存在；
3. Prompt 模板能被编译；
4. 实际变量集合与 Manifest 完全一致；
5. 不允许缺失 Prompt 后静默回退到另一个 Edition。

Prompt 总量在当前业务规模下不值得通过按文件延迟 import 增加复杂度。启动时全量失败比在某个用户请求中首次失败更容易运维。

---

## 十、模型调用边界

模块依赖 `LLMClient`，但不直接依赖某个 Provider 或物理 Endpoint：

```python
class LLMClient(Protocol):
    """向摘要模块提供稳定的逻辑模型调用。"""

    async def generate(
        self,
        logical_model: str,
        prompt: str,
    ) -> str:
        """调用逻辑模型并返回文本结果。"""
        ...
```

职责边界：

| 层级 | 负责的重试 |
| --- | --- |
| `LLMClient` / Model Hub | 超时、429、Endpoint Failover 等基础设施重试 |
| Summary 模块 | 输出过短、格式不合法、业务校验失败等业务级重试 |

模块可以指定 `summary-default`、`summary-reasoning` 等逻辑模型，但不应该选择具体云厂商 Endpoint。

`LLMClient.generate()` 不接收仅用于日志的 `summary_id`、`file_id` 或 `trace_id`。实现层从当前任务上下文和 OTel Context 自动取得关联信息。只有幂等键、模型路由约束等真正影响调用语义的数据，才通过类型明确的调用参数显式传递。

---

## 十一、错误与可观测性

可观测性的目标是回答三个不同问题：

1. 这个业务任务经历过哪些执行？
2. 某次执行经过了哪些步骤和外部调用？
3. 某类场景的整体成功率和耗时是否异常？

这三个问题分别由业务标识、Trace 和 Metrics 回答，不能混用。

### 11.1 标识符各司其职

| 标识 | 含义 | 生命周期 | 主要用途 |
| --- | --- | --- | --- |
| `summary_id` | 摘要业务任务 | 跨重试、重新入队和多次执行稳定 | 聚合同一任务的全部日志与执行历史 |
| `file_id` | 输入文件业务身份 | 跟随文件生命周期 | 关联文件相关任务，不替代 `summary_id` |
| `request_id` | 一次入口请求或消息投递 | 每次 HTTP 请求或消息投递 | 排查入口、网关和消费问题 |
| `trace_id` | 一次端到端执行链路 | 每次执行生成或从上游传播 | 展开跨模块、跨服务调用链 |
| `span_id` | Trace 中的一个步骤 | 单个操作 | 定位具体 Pipeline 步骤或外部调用 |

查询顺序是固定的：先按 `summary_id` 找到该任务的执行记录，再选中某个 `trace_id` 查看完整链路，最后通过 `span_id` 定位步骤。

### 11.2 每条日志自动关联任务上下文

业务代码不应该为了日志而层层传递 `summary_id`。Logging Filter 在生成 `LogRecord` 时读取 `ContextVar`，统一补齐字段：

```python
class SummaryContextFilter(logging.Filter):
    """为当前任务日志注入安全关联字段。"""

    def filter(self, record: logging.LogRecord) -> bool:
        """上下文存在时补齐字段；不存在时保持日志可用。"""

        context = current_summary_context.get()
        if context is None:
            record.summary_id = ""
            record.file_id = ""
            record.scenario = ""
            record.request_id = ""
            return True

        record.summary_id = context.summary_id
        record.file_id = context.file_id
        record.scenario = context.scenario
        record.request_id = context.request_id or ""
        return True
```

Filter 挂在应用的日志 Handler 上，而不是逐个业务 Logger 配置。这样模块、共享能力和第三方调用适配层产生的日志都会经过同一注入路径。

日志 Formatter 同时读取当前 OpenTelemetry Span 的 `trace_id` 和 `span_id`。业务代码只需记录事件本身：

```python
logger.info(
    "meeting merge completed",
    extra={
        "event": "summary.pipeline.step.completed",
        "step": "merge",
        "duration_ms": duration_ms,
    },
)
```

最终日志自动包含：

```json
{
  "event": "summary.pipeline.step.completed",
  "summary_id": "summary-123",
  "file_id": "file-456",
  "scenario": "meeting",
  "request_id": "request-789",
  "trace_id": "...",
  "span_id": "...",
  "step": "merge",
  "duration_ms": 842
}
```

日志中不写入 transcript、Prompt 正文、模型完整输入输出、密钥或用户隐私字段。

### 11.3 Trace 描述一次执行

`SummaryApplication.generate()` 创建 Root Span。模块只为具有诊断价值的步骤创建 Child Span，不为每个函数机械建 Span。

Meeting 的典型层级如下：

```text
summary.generate
└─ summary.module.meeting
   ├─ summary.split
   ├─ summary.chunk (并行多个)
   │  └─ llm.generate
   ├─ summary.merge
   │  └─ llm.generate
   ├─ summary.title
   │  └─ llm.generate
   └─ summary.validate
```

Span 可以记录以下安全属性：

- `summary.id`、`file.id`、`summary.scenario`；
- `summary.step`、`summary.chunk_count`、`summary.attempt`；
- `prompt.edition`、`prompt.key`；
- `llm.logical_model`、`llm.input_tokens`、`llm.output_tokens`；
- `error.type`。

`summary_id` 是 Span 属性，不是 `trace_id` 的替代品，也不要把它拼进 `trace_id`。

### 11.4 并发与跨边界传播

同一事件循环中的 `asyncio.create_task()` 和 `asyncio.gather()` 会复制当前 Context，因此每个分段任务都能读取同一个只读任务上下文，同时拥有自己的 Child Span。

跨边界时遵循明确规则：

| 边界 | 传播方式 |
| --- | --- |
| 同一事件循环 | 自动继承 `ContextVar` 与当前 OTel Context |
| 线程池 | 使用 `contextvars.copy_context()` 包装提交任务 |
| 新进程 | 不继承内存上下文；通过 IPC 消息重建 |
| HTTP / RPC | 使用 W3C `traceparent` 传播 Trace；业务请求继续显式携带业务 ID |
| Kafka / Queue | 消息体携带 `summary_id`、`file_id`、`scenario`；Header 携带 `traceparent` |

消费者收到消息后先恢复 OTel Context，再绑定新的 `SummaryExecutionContext`。正常异步消费可以延续上游 Trace。任务被重新入队并开始一次新的独立执行时，创建新 Trace，并用 Span Link 关联上一次执行；`summary_id` 保持不变。

### 11.5 Metrics 只使用低基数维度

请求级指标由应用层统一记录：

- 请求总数与成功率；
- 总耗时；
- Token 与模型调用次数；
- 最终状态和错误类型。

模块为重要步骤记录耗时、Token 和状态，例如 `split`、`summarize_chunk`、`merge`、`generate_title`、`validate_output`。

Metrics Label 只使用稳定、低基数字段：

- `scenario`；
- `step`；
- `status`；
- `error_type`；
- `logical_model`；
- `prompt_key`；
- `prompt_edition`。

> [!danger] 禁止高基数 Label
> `summary_id`、`file_id`、`request_id`、`trace_id` 和用户 ID 不进入 Metrics Label。它们属于日志或 Trace，否则会导致时序数量持续膨胀。`prompt_key` 只能来自启动时注册的有限集合。

### 11.6 错误分类保持稳定

全系统只保留少量稳定错误类型：

| 错误 | 说明 |
| --- | --- |
| `UnsupportedScenarioError` | 请求了未注册场景 |
| `PromptConfigurationError` | Prompt 文件、Manifest 或变量不合法 |
| `ModelInvocationError` | 模型调用最终失败 |
| `OutputValidationError` | 模型返回结果不满足模块要求 |
| `SummaryTimeoutError` | 整个摘要请求超时 |

模块可以在异常中附带安全的错误上下文，但由应用层统一转换、记录 Span 状态并生成对外错误。不要为每个模块创建一套平行异常体系。

---

## 十二、测试策略

### 12.1 Prompt 测试

- 每个已注册模块在 CN/global 都有完整 Prompt；
- Prompt 文件变量与 Manifest 一致；
- CN/global 可以在同一测试进程分别构造；
- 缺失文件、额外变量和非法模板都会失败；
- 模块不直接读取 Region。

### 12.2 模块 Pipeline 测试

使用 Fake `LLMClient`，不调用真实模型：

- Dairy 只调用一次模型；
- Meeting 短文本不进入 merge；
- Meeting 长文本会并发处理所有分段；
- 分段摘要完成后才执行 merge；
- merge 完成后执行 title 和校验；
- 业务校验失败时按模块规则重试或抛错。

### 12.3 应用层测试

- `scenario` 能路由到正确模块；
- 未注册场景返回统一错误；
- 模块异常被转换成统一业务错误；
- 请求级超时和 Root Span 对所有模块生效；
- 每次执行结束后都会恢复旧的 `ContextVar`；
- 两个并发任务的 `summary_id` 不会互相污染；
- 模块只从 `SummaryRequest` 读取业务数据。

### 12.4 可观测性测试

- 上下文存在时，每条日志自动包含 `summary_id`、`file_id` 和 `scenario`；
- 没有上下文的启动日志和后台日志仍可正常输出；
- Child Span 继承 Root Span，并包含稳定的业务属性；
- `asyncio.gather()` 中的分段任务共享任务元信息，但拥有独立 Span；
- Queue 消费者能从消息体恢复业务 ID，从 Header 恢复 Trace Context；
- Metrics 不包含任何高基数 Label。

---

## 十三、新增模块的步骤

新增一个 `sales-bant` 模块时：

1. 创建 `modules/sales_bant/`；
2. 定义 `SalesBantModule.run()`；
3. 按业务需要直接编写 Pipeline；
4. 增加 Prompt Manifest；
5. 增加 CN/global Prompt 文件；
6. 增加 Pipeline 单元测试；
7. 在组合根显式注册模块。

不需要：

- 修改其他模块；
- 给中央 Prompt Service 增加 `resolve_sales_bant()`；
- 给通用 Pipeline 增加大量条件分支；
- 创建新的 loader 函数；
- 修改 Region 判断逻辑。

---

## 十四、当前阶段明确不做

为避免过度设计，第一版不实现：

- 通用 DAG 或工作流 DSL；
- `Step`、`Node`、`Edge` 等抽象框架；
- 运行时动态安装模块；
- Python 包自动扫描；
- 独立 Prompt 微服务；
- Langfuse 等远程 Prompt 的请求时读取；
- Prompt 灰度发布平台；
- 任意场景之间的 Pipeline 继承体系；
- Prompt 缺失时跨 Region 静默回退；
- 保存所有业务数据和中间结果的全局万能 State；
- 用 `trace_id` 替代 `summary_id`，或把业务 ID 编码进 `trace_id`。

当至少三个模块出现完全相同的真实步骤时，再把该步骤提取为共享函数或小组件。不要提前抽象一个通用 Pipeline 框架。

---

## 十五、与 ADR-6 的关系

ADR-6 可以继续作为现有系统迁移 Dairy Prompt 的局部方案。它解决的是导入期 Region 判断和新旧调用方兼容问题。

如果从零建设，目标架构做以下调整：

| ADR-6 当前做法 | 从零设计 |
| --- | --- |
| `ReasoningPromptService.resolve_dairy()` | `DairyModule.run()` 自己组织流程 |
| 每个 Prompt 编写 loader | `PromptCatalog` 根据 Manifest 加载文件 |
| Prompt 返回裸字符串 | 模块取得自己命名空间下的 Prompt |
| 只迁移 Reasoning 中的 Dairy | 每个业务场景是独立模块 |
| 兼容旧 Prompt 和可空 Service | 不保留双路径 |
| 按需 lazy import | 启动时加载并校验当前 Edition |

仍然保留 ADR-6 中两个正确原则：

1. Region 只在组合根转换，业务模块不读取 Region；
2. Prompt 模板变量必须经过严格校验。

---

## 十六、推荐实施顺序

如果用该设计启动新业务，建议按以下顺序交付：

1. 定义 `SummaryRequest`、`SummaryResult` 和 `SummaryModule`；
2. 实现只读 `SummaryExecutionContext`、日志注入和 Root Span；
3. 实现 `ModuleRegistry` 和 `SummaryApplication`；
4. 实现本地 `PromptCatalog` 及启动校验；
5. 用 Dairy 实现最简单的单调用模块；
6. 用 Meeting 验证自定义长文本 Pipeline 与并发上下文隔离；
7. 接入统一错误、超时、步骤 Span 和低基数指标；
8. 再逐个增加真实业务模块。

完成 Dairy 和 Meeting 后即可验证这套架构的两个核心能力：

- 简单模块不会被通用框架拖累；
- 复杂模块可以完全控制自己的 Pipeline；
- 每条任务日志都能通过 `summary_id` 关联，并可通过 `trace_id` 展开一次执行链路。
