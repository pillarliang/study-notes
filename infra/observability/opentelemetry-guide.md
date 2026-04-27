# OpenTelemetry 完整指南

> 本笔记按"基础概念 → 数据流 → SDK 实操 → 深度专题 → 排障 → 实战案例"组织，覆盖 OpenTelemetry 在 Python 服务中的从原理到生产的完整链路。相关部署侧内容见 [[09-k8s-observability]]。

## 全文摘要

**OpenTelemetry（OTel）** 是 CNCF 托管的开源可观测性框架，由 OpenTracing + OpenCensus 于 2019 年合并而来，提供统一的遥测数据标准（OTLP 协议）。

**三大支柱**：

- **Traces**：以 `trace_id` 串联的 Span 树，记录请求跨服务的完整调用路径，通过 W3C `traceparent` header 传播上下文。
- **Metrics**：Counter/Histogram/Gauge 等聚合数值，支撑监控告警与容量规划。
- **Logs**：离散事件记录，通过 `trace_id` / `span_id` 与 Traces 关联。

**典型后端**：Traces 用 Jaeger / Tempo / Elasticsearch；Metrics 用 Prometheus / Mimir / VictoriaMetrics；Logs 用 ELK / Loki / ClickHouse。Grafana 做统一可视化。

**完整链路**：应用 SDK 内累计 → BatchProcessor 批处理 → OTLP Exporter（gRPC / HTTP）→ OTel Collector 接收/处理/路由 → 后端存储 → Grafana 查询渲染。

**关键设计原则**：Span 名描述操作而非实现；高基数（user_id、trace_id 等）放 Trace attributes 或日志，不放 Metric label；Histogram 一律显式配 `explicit_bucket_boundaries_advisory`；生产环境采样率 10–20%；Batch + 异步导出避免阻塞业务线程。

**常见踩坑**：Metrics 的 Cumulative / Delta temporality 选错会导致断崖；Grafana 的 `histogram_quantile` 和 Metabase / SQL 的 `PERCENTILE_CONT` 永远对不上（桶插值误差）；asyncio 后台任务默认丢失 OTel 上下文，需手动 `attach`。

---

## 目录

- [一、基础概念](#一基础概念)
- [二、Traces：分布式追踪](#二traces分布式追踪)
- [三、Metrics：聚合指标](#三metrics聚合指标)
- [四、数据流：从代码到看板](#四数据流从代码到看板)
- [五、Python SDK 实操指南](#五python-sdk-实操指南)
- [六、当前项目实现参考（项目特定）](#六当前项目实现参考项目特定)
- [七、深度专题（深入）](#七深度专题深入)
- [八、最佳实践](#八最佳实践)
- [九、常见问题与调试](#九常见问题与调试)
- [十、实战案例：asyncio 后台任务的上下文传播（深入）](#十实战案例asyncio-后台任务的上下文传播深入)
- [附录](#附录)

---

## 一、基础概念

### 1.1 什么是 OpenTelemetry

OpenTelemetry（简称 OTel）是 CNCF（云原生计算基金会）托管的开源可观测性框架，提供一套统一的 API、SDK 和工具，用于在应用中生成、收集和导出遥测数据。

历史背景：

```
OpenTracing (分布式追踪 API) + OpenCensus (Google 指标/追踪库)
                       ↓ 2019 年合并
                  OpenTelemetry
```

OTel 解决的根本问题是**多种监控工具不兼容、数据孤岛、厂商锁定**。在它之前，每接一个新后端（Jaeger、Datadog、New Relic）都要换一套埋点代码；OTel 之后，业务代码只面对统一的 API，后端切换只换 Exporter 配置。


| 旧时代痛点                    | OTel 解决方案                  |
| ------------------------ | -------------------------- |
| 多种监控工具不兼容                | 统一的数据格式（OTLP 协议）           |
| 厂商锁定                     | 厂商中立，可切换后端                 |
| 手动埋点繁琐                   | 自动检测（Auto-instrumentation） |
| Traces/Metrics/Logs 数据孤岛 | 通过 trace_id 关联             |


### 1.2 三大支柱

可观测性（Observability）的三大支柱回答的是不同的问题：

```
┌─────────────────────────────────────────────────────────────┐
│                    Observability（可观测性）                  │
├───────────────────┬───────────────────┬─────────────────────┤
│      Traces       │     Metrics       │       Logs          │
│    （链路追踪）     │     （指标）       │      （日志）        │
├───────────────────┼───────────────────┼─────────────────────┤
│ 请求在系统中的       │ 聚合的数值数据      │ 离散的事件记录       │
│ 完整路径            │                  │                     │
├───────────────────┼───────────────────┼─────────────────────┤
│ 用途：定位延迟、     │ 用途：监控健康状态   │ 用途：调试、审计     │
│ 分析调用链          │ 告警、容量规划      │ 错误排查             │
└───────────────────┴───────────────────┴─────────────────────┘
```

三者各有专长，**不应互相替代**：

- **Metrics 负责"面"**：聚合趋势，回答"系统现在健康吗？"
- **Traces 负责"点"**：单次请求的完整路径，回答"这一次请求慢在哪？"
- **Logs 负责"细节"**：文本上下文，回答"那一刻代码到底干了什么？"

### 1.3 SDK 组件分层

OTel 的 Python 实现分为三层：API（业务代码面对的接口）、SDK（具体实现）、Exporter（导出到后端）。这种分层让"业务代码不依赖具体后端"成为可能：

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Application                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                         API Layer                            │    │
│  │   trace.get_tracer()    metrics.get_meter()                 │    │
│  └─────────────────────────────┬───────────────────────────────┘    │
│                                │                                     │
│  ┌─────────────────────────────▼───────────────────────────────┐    │
│  │                         SDK Layer                            │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │    │
│  │  │TracerProvider│  │MeterProvider │  │LoggerProvider│       │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │    │
│  │         │                 │                 │                │    │
│  │  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐       │    │
│  │  │  Processors  │  │   Readers    │  │  Processors  │       │    │
│  │  │ (Batch/Simple│  │ (Periodic)   │  │              │       │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │    │
│  │         │                 │                 │                │    │
│  │  ┌──────▼─────────────────▼─────────────────▼───────┐       │    │
│  │  │                    Exporters                      │       │    │
│  │  │  OTLP | Jaeger | Zipkin | Prometheus | Console   │       │    │
│  │  └──────────────────────────────────────────────────┘       │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

各层职责：

- **API**：定义 `Tracer`、`Meter` 等接口，业务代码只导入 API 层；不引入 SDK 时调用全部是 no-op，零开销。
- **SDK**：具体实现 Provider、Processor（批量/采样）、Reader（聚合周期）。
- **Exporter**：把内部数据结构序列化后通过具体协议（OTLP gRPC / HTTP / Jaeger / Prometheus）发出去。

### 1.4 OTel 与周边系统的关系


| 系统                           | 关系                                                                                                                           |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Prometheus**               | OTel Metrics 的典型后端。Prometheus v2.47+ 起原生支持 OTLP receiver，可不经过 Collector 直接接收 OTLP；早期版本必须经 Collector 转换。                      |
| **Jaeger / Tempo**           | OTel Traces 的典型后端。Tempo 走对象存储，成本最低；Jaeger 元数据靠 Cassandra/ES/ClickHouse。                                                      |
| **Grafana**                  | 统一可视化层。Prometheus、Loki、Tempo 都是 Grafana 的数据源，看板里用 PromQL/LogQL/TraceQL 查询。                                                   |
| **Langfuse**（LLM 可观测性平台）     | 数据模型 Trace → Span/Generation 本质上复用 OTel 的 Trace 树形结构，是分布式追踪在 LLM 场景的特化。新版已支持原生 OTLP 协议接入，OTel 可直接把 LLM 调用的 Span 推到 Langfuse。 |
| **OpenTracing / OpenCensus** | OTel 的前身，2019 年合并，已停止维护。                                                                                                     |


---

## 二、Traces：分布式追踪

### 2.1 Span 与 Trace 的关系

**Span 是 Trace 的基本组成单元。** 一个 Span 代表一个有明确开始和结束时间的**操作**——可以是一次函数调用、一次 HTTP 请求、一次数据库查询等。Span 不等于函数：它是要观测的任意一段操作。

**Trace 是一组共享同一个 `trace_id` 的 Span 集合**，通过 `parent_span_id` 组成树形结构，完整描述一个请求从入口到结束的调用路径。

```
一个请求的完整 Trace（所有 Span 共享同一个 trace_id）：

trace_id: abc123

Span A: API Gateway (parent: 无 → 这是 Root Span)
├── Span B: Auth Service (parent: A)
├── Span C: Business Logic (parent: A)
│   ├── Span D: DB Query (parent: C)
│   └── Span E: Cache Lookup (parent: C)
└── Span F: Response Serialize (parent: A)

时间轴视图：
Span A ██████████████████████████████████████████████
Span B   ████
Span C        ██████████████████████████
Span D          ████████
Span E                    ██████
Span F                                      ████
```

> **关键认知：** Trace 本身并不是独立的数据结构——后端存储的是一个个 Span，它们通过共同的 `trace_id` 被"串"成一条 Trace，通过 `parent_span_id` 被"组"成一棵树。

### 2.2 Span 的关键字段

每个 Span 携带以下核心标识：


| 字段                        | 作用                               | 示例                    |
| ------------------------- | -------------------------------- | --------------------- |
| `trace_id`                | 全局唯一，标识整条链路，同一 Trace 下所有 Span 共享 | `abc123...` (128-bit) |
| `span_id`                 | 当前 Span 的唯一标识                    | `def456...` (64-bit)  |
| `parent_span_id`          | 指向父 Span，Root Span 该字段为空         | `aaa789...`           |
| `name`                    | 操作名称                             | `HTTP GET /api/users` |
| `start_time` / `end_time` | 起止时间戳                            | 用于计算耗时                |
| `attributes`              | 键值对元数据                           | `http.method=GET`     |
| `events`                  | Span 内的时间点事件                     | 异常发生、重试开始             |
| `links`                   | 跨 Trace 的关联                      | 批处理任务关联多个原始请求         |
| `status`                  | 操作结果                             | `OK` / `ERROR`        |


### 2.3 Span Status 与 Span Kind

**Status** 表示操作结果：


| 状态      | 含义         |
| ------- | ---------- |
| `UNSET` | 默认状态，未明确设置 |
| `OK`    | 操作成功完成     |
| `ERROR` | 操作出错       |


**Kind** 描述 Span 在分布式调用中的角色，影响后端展示方式（例如 Jaeger 会按 SERVER/CLIENT 配对绘制跨服务调用线）：


| Kind       | 描述       | 示例           |
| ---------- | -------- | ------------ |
| `INTERNAL` | 内部操作（默认） | 内部函数调用       |
| `SERVER`   | 服务端处理请求  | HTTP 服务器处理请求 |
| `CLIENT`   | 客户端发起请求  | HTTP 客户端调用   |
| `PRODUCER` | 消息生产者    | Kafka 生产消息   |
| `CONSUMER` | 消息消费者    | Kafka 消费消息   |


### 2.4 Context Propagation：跨服务串联

Span 之间通过 **Context Propagation** 跨进程传递 `trace_id` 和 `parent_span_id`，最常见的方式是 **W3C `traceparent` HTTP header**。

```
traceparent: 00-<trace_id>-<parent_span_id>-<flags>
             │   │            │               │
             │   │            │               └─ 采样标志（01=采样）
             │   │            └─ 当前 Span ID，下游服务以此为 parent
             │   └─ 整条链路的 Trace ID
             └─ 版本号

示例：
Service A 发起请求，header 中携带：
  traceparent: 00-abc123...-spanA...-01

Service B 收到后：
  1. 解析出 trace_id=abc123, parent_span_id=spanA
  2. 创建新 Span（trace_id=abc123, parent=spanA, span_id=spanB）
  3. 继续往下游传播
```

跨服务流转图：

```
Service A                          Service B
┌─────────────────────┐           ┌─────────────────────┐
│ Span: handle_request│           │ Span: process_data  │
│ trace_id: abc123    │──HTTP────▶│ trace_id: abc123    │
│ span_id: 001        │  Headers  │ span_id: 002        │
│                     │           │ parent_id: 001      │
└─────────────────────┘           └─────────────────────┘
```

主流传播格式：

- **W3C Trace Context**（标准，推荐）
- **B3**（Zipkin 格式）
- **Jaeger**（Uber 格式）

异步场景下（线程、`asyncio.create_task`）上下文不会自动传播，需要手动 `attach`，详见 [10. 实战案例](#十实战案例asyncio-后台任务的上下文传播深入)。

### 2.5 Span 生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                        Span Lifecycle                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Start ──▶ Set Attributes ──▶ Add Events ──▶ End           │
│    │              │                │            │           │
│    ▼              ▼                ▼            ▼           │
│ 记录开始时间   添加键值对属性   记录重要事件   计算持续时间   │
│ 生成 span_id  http.method     exception      设置状态       │
│ 设置 parent   db.statement    retry.start   导出数据       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.6 Logs 与 Traces 关联

OTel 提供 `LoggingInstrumentor`，自动把 `trace_id` / `span_id` 注入到日志记录中，使日志和 Trace 在后端可以双向跳转。

```python
import logging
from opentelemetry import trace

logger = logging.getLogger(__name__)

def process_request():
    span = trace.get_current_span()
    ctx = span.get_span_context()

    logger.info(
        "Processing request",
        extra={
            "trace_id": format(ctx.trace_id, '032x'),
            "span_id": format(ctx.span_id, '016x'),
        }
    )
```

> **重要区分：日志注入 vs Span Attributes**
>
> `LoggingInstrumentor` 自动注入到日志中的**只有** `otelTraceID`、`otelSpanID`、`otelTraceSampled` 三个字段。
>
> 通过 `span.set_attribute("task_id", "xxx")` 设置的**自定义 Span Attributes 不会出现在日志中**——它们只存在于 Trace 后端（Jaeger/Tempo）。两者数据流向不同：
>
> ```
> LoggingInstrumentor 注入  →  日志记录（CloudWatch/Loki）
>                               ↑ 只有 trace_id, span_id, trace_sampled
>
> span.set_attribute()      →  Span 数据（Jaeger/Tempo）
>                               ↑ 自定义属性只在这里可见
> ```
>
> 业务字段（如 `task_id`）若要在日志里也能 `grep`，必须通过 `logger.info("msg", extra={"task_id": "xxx"})` 或 `ContextVar` 显式传入，OTel 不会自动同步。详见 [10.3](#103-contextvar--日志格式化器覆盖-trace_id) 的覆盖方案。

---

## 三、Metrics：聚合指标

### 3.1 核心概念：预聚合数据

Metrics 是**预聚合的数值数据**——与 Traces（离散事件）和 Logs（文本记录）最大的区别在于：它在采集端就做了数学运算（求和、计数、分桶），存储的是统计结果而非原始事件。这是它在存储成本和查询速度上有天然优势的根本原因。

```
原始事件流（高基数）         聚合后的 Metric（低基数）
─────────────────────       ─────────────────────────
req 1: 200ms, POST /api     http_request_duration_seconds{method="POST", endpoint="/api"}
req 2: 150ms, POST /api       count = 3
req 3: 300ms, POST /api       sum   = 0.65
                               bucket_le_0.5 = 2
                               bucket_le_1.0 = 3
```

### 3.2 Instrument 类型全景

OTel 定义 6 种 Instrument，分为同步（业务代码主动调用）和异步（SDK 按周期回调）两类：


| 类型                          | 同步/异步 | 单调性  | 典型场景   | 举例             |
| --------------------------- | ----- | ---- | ------ | -------------- |
| **Counter**                 | 同步    | 只增   | 累计计数   | 请求总数、已处理消息数    |
| **UpDownCounter**           | 同步    | 可增可减 | 当前计数   | 活跃连接数、队列长度变化   |
| **Histogram**               | 同步    | —    | 值的分布   | 请求延迟、响应体大小     |
| **ObservableCounter**       | 异步    | 只增   | 系统级累计量 | CPU time、页面错误数 |
| **ObservableUpDownCounter** | 异步    | 可增可减 | 系统级瞬时量 | 内存使用量、线程数      |
| **ObservableGauge**         | 异步    | 可增可减 | 瞬时采样值  | CPU 温度、当前房间人数  |


> **同步 vs 异步的分工：**
>
> - **同步 Instrument**：在业务代码中主动调用 `.add()` / `.record()`，适合跟业务事件绑定（每次请求结束记一笔）。
> - **异步 Instrument**：注册回调函数，SDK 按采集周期自动调用，适合采集系统状态（CPU、内存等"无明确事件"的瞬时量）。

### 3.3 聚合方式与 Temporality

#### 3.3.1 默认聚合方式

SDK 在导出前会对原始测量值做聚合，不同 Instrument 默认聚合方式不同：


| Instrument                              | 默认聚合                        | 导出的数据                    |
| --------------------------------------- | --------------------------- | ------------------------ |
| Counter / ObservableCounter             | **Sum**（累计）                 | 单个递增数值                   |
| UpDownCounter / ObservableUpDownCounter | **Sum**（非单调）                | 可增减的数值                   |
| Histogram                               | **ExplicitBucketHistogram** | count、sum、min、max + 各桶计数 |
| ObservableGauge                         | **LastValue**               | 最近一次采样值                  |


#### 3.3.2 Temporality（时间性）

Metrics 导出有两种时间语义，**选错会导致数据错误**：

```
Cumulative（累计型）：每次导出从 0 开始的总量
  T1: 100    T2: 250    T3: 400    ← 始终是总计数

Delta（增量型）：每次导出只报告上一周期的增量
  T1: 100    T2: 150    T3: 150    ← 每个周期新增的量
```


| 后端                              | 推荐 Temporality | 原因                                |
| ------------------------------- | -------------- | --------------------------------- |
| **Prometheus**                  | Cumulative     | Prometheus 自己用 `rate()` 算速率，需要累计值 |
| **OTLP → ClickHouse / Datadog** | Delta          | 避免重启归零问题，服务端做累加                   |


> **常见坑：** Prometheus 只支持 Cumulative，如果配错成 Delta，图表会出现断崖式下跌。

### 3.4 Attributes 与时间序列基数（重点）

Metrics 的存储成本几乎全部由"时间序列基数"决定。这一节是**最容易翻车也最关键的部分**。

#### 3.4.1 什么是时间序列

**时间序列（Time Series）** 是 TSDB（时序数据库）中的最小存储单元——一个指标名 + 一组固定的 label 值，随时间不断追加 `(时间戳, 数值)` 数据点：

```
一条时间序列 = 指标名{label="固定值"} + 按时间追加的数据点

http_requests_total{method="GET", status="200"}
  → T1: 10,  T2: 15,  T3: 23,  T4: 30 ...    ← 不断追加，独立存储

http_requests_total{method="GET", status="500"}
  → T1: 0,   T2: 1,   T3: 1,   T4: 3  ...    ← 这是另一条独立的时间序列

http_requests_total{method="POST", status="200"}
  → T1: 5,   T2: 8,   T3: 12,  T4: 18 ...    ← 又一条独立的时间序列
```

> 可以把每条时间序列想象成**数据库里的一张独立的表**，各自不断 append 新行。

#### 3.4.2 Attributes 如何产生时间序列

Attributes 是 Metric 的**维度标签**（类似 SQL 中的 GROUP BY 字段），每组唯一的 attribute 组合会生成一条独立的时间序列：

```
# 2 个 method × 3 个 endpoint × 2 个 status = 12 条时间序列
http_requests_total{method="GET",  endpoint="/api",    status="200"}
http_requests_total{method="GET",  endpoint="/api",    status="500"}
http_requests_total{method="POST", endpoint="/api",    status="200"}
...
```

label 决定了能按什么维度切片查询（就像 SQL 中 WHERE + GROUP BY）：

```
按 method 维度聚合（合并所有 status）：
  rate(http_requests_total{method="GET"})

按 status 维度聚合（合并所有 method）：
  rate(http_requests_total{status="500"})

全部聚合：
  rate(http_requests_total)                  ← 合并所有时间序列
```

**基数（Cardinality）** 指某个 Attribute/Label **可能取值的数量**。每一组唯一的 label 组合都会产生一条独立的时间序列，因此基数直接决定存储和查询压力。

#### 3.4.3 低基数 vs 高基数


| 维度              | 低基数                                                                                | 高基数                                                       |
| --------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------- |
| **定义**          | 取值范围小且有限                                                                           | 取值范围大或无限                                                  |
| **典型值数量**       | 几个 ~ 几百个                                                                           | 几万 ~ 无限                                                   |
| **例子**          | `method`(GET/POST/PUT/DELETE)、`status_code`(200/400/500)、`region`(us-east/eu-west) | `user_id`(100万+)、`request_id`(每请求唯一)、`email`、`session_id` |
| **是否适合做 label** | 适合                                                                                 | 不适合                                                       |


#### 3.4.4 为什么高基数会导致灾难

核心公式：

```
时间序列总数 = label_1 取值数 × label_2 取值数 × ... × label_N 取值数
```

具体例子：

```
http_requests_total{method, endpoint, status_code}

低基数方案：
  method:      4 种 (GET/POST/PUT/DELETE)
  endpoint:   10 个
  status_code:  5 种 (200/400/401/404/500)
  → 4 × 10 × 5 = 200 条时间序列 ✅

混入一个高基数 label：
  + user_id: 100,000 个用户
  → 4 × 10 × 5 × 100,000 = 2 亿条时间序列 💥
```

**2 亿条时间序列意味着：**

- Prometheus 每条时间序列约占 1-2 KB 内存 → 单这一个指标就要 **200-400 GB 内存**
- 每次查询需要扫描/聚合 2 亿条序列 → **查询超时**
- 持久化到磁盘的 block 膨胀 → **磁盘 I/O 打满**

#### 3.4.5 判断基数的经验法则

```
问自己：这个 label 的取值是否会随业务增长而无限增长？

  "method"      → 永远就那几个         → ✅ 低基数，放心用
  "status_code" → HTTP 状态码有限      → ✅ 低基数
  "region"      → 部署区域有限         → ✅ 低基数
  "endpoint"    → API 数量可控         → ⚠️ 中等基数，注意不要包含路径参数
  "customer_id" → 随业务增长           → ❌ 高基数
  "trace_id"    → 每请求唯一           → ❌ 无限基数，绝对不能用
```

**特别注意 URL 路径参数的陷阱：**

```
❌ 错误：endpoint="/api/users/12345"    ← user ID 嵌入路径，无限基数
✅ 正确：endpoint="/api/users/{id}"     ← 参数化后变成低基数
```

#### 3.4.6 高基数需求的正确解法


| 需求            | 错误做法                           | 正确做法                      |
| ------------- | ------------------------------ | ------------------------- |
| 统计每个用户的请求数    | Metric label 加 `user_id`       | 用 Traces 按 user_id 查询聚合   |
| 定位某个慢请求       | Histogram label 加 `request_id` | 用 Exemplar 关联到 Trace      |
| 按 IP 分析流量     | Counter label 加 `client_ip`    | 日志/流量分析系统（ELK、ClickHouse） |
| 统计 Top N 热门商品 | Metric label 加 `product_id`    | 日志聚合 或 OLAP 数据库           |


> **一句话总结：Metrics 负责"面"（聚合趋势），Traces 负责"点"（单次请求），Logs 负责"细节"（文本上下文）。高基数分析是 Traces 和 Logs 的职责，不要让 Metrics 来扛。**
>
> **经验法则：** 单个 Metric 的时间序列数应控制在 **< 1000**。

### 3.5 Histogram：桶设计与 Exemplars

#### 3.5.1 桶设计的基本理念

Histogram 在采集端就把样本按"边界"分桶，存的是每个桶的累计计数，**不保留原始样本值**。因此桶边界设计直接决定后续 P95/P99 估算的精度（详见 [7.1](#71-histogram-分位数读数陷阱p95--p99)）。

```python
# 场景：统计请求延迟分布
latency_histogram = meter.create_histogram(
    name="http_request_duration_seconds",
    description="HTTP request latency",
    unit="s",
    explicit_bucket_boundaries_advisory=[
        0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0
    ]
)

latency_histogram.record(0.235, {"endpoint": "/api/summary"})
```

可视化效果：

```
延迟分布示例 (单位: 秒)
┌────────────────────────────────────────────────────────────┐
│  Bucket      │ Count │ 可视化                              │
├──────────────┼───────┼─────────────────────────────────────┤
│  ≤ 0.01s     │  150  │ ████████████████                   │
│  ≤ 0.05s     │  320  │ ██████████████████████████████████ │
│  ≤ 0.1s      │  180  │ ███████████████████                │
│  ≤ 0.5s      │   45  │ ████                               │
│  ≤ 1.0s      │   12  │ █                                  │
│  > 1.0s      │    3  │                                    │
└──────────────┴───────┴─────────────────────────────────────┘
```

> 新建 Histogram **务必显式配 `explicit_bucket_boundaries_advisory`**，OTel Python 默认桶（5、10、20…10000）只覆盖毫秒到百秒量级的通用 HTTP 延迟，对秒级以下或长任务都不合适，详见 [7.1.3](#713-opentelemetry-python-默认桶不适合所有场景)。

#### 3.5.2 Exemplars：从 Metrics 跳到 Traces

Exemplar 允许在 Metric 数据点上附加一个 Trace 的采样引用，实现从聚合指标钻取到具体请求：

```
Histogram bucket [0.5s, 1.0s] 命中了 12 次
  └─ Exemplar: trace_id=abc123, span_id=def456, value=0.78s
     → 点击可跳转到该慢请求的完整 Trace
```

这是 Grafana 中"从 Metrics 面板点击跳转到 Jaeger/Tempo 查看 Trace"的底层机制。

### 3.6 三种 Instrument 的代码示例

```python
from opentelemetry import metrics

meter = metrics.get_meter(__name__)

# Counter - 累计计数（如总请求数）
request_counter = meter.create_counter(
    name="http_requests_total",
    description="Total number of HTTP requests",
    unit="1"
)
request_counter.add(1, {
    "method": "POST",
    "endpoint": "/api/summary",
    "status_code": "200"
})

# Histogram - 值的分布（如延迟、字节数）
latency_histogram = meter.create_histogram(
    name="http_request_duration_seconds",
    unit="s",
    explicit_bucket_boundaries_advisory=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)
latency_histogram.record(0.235, {"endpoint": "/api/summary"})

# UpDownCounter - 可增可减（如活跃连接）
active_connections = meter.create_up_down_counter(
    name="active_connections",
    unit="1"
)
active_connections.add(1)   # 连接建立
active_connections.add(-1)  # 连接关闭
```

异步 Instrument 通过回调函数采集（适用于"无明确事件"的瞬时量）：

```python
import psutil

# 异步 Gauge - CPU 使用率
def get_cpu_usage(options):
    yield metrics.Observation(
        psutil.cpu_percent(),
        {"host": "server-1"}
    )

cpu_gauge = meter.create_observable_gauge(
    name="system_cpu_usage_percent",
    unit="%",
    callbacks=[get_cpu_usage]
)
```

---

## 四、数据流：从代码到看板

这一章把数据从产生到展示的完整链路串起来。前面 2、3 章讲的是"数据本身长什么样"，本章讲"数据怎么从应用到达 Grafana"。

### 4.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Your Services                                   │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  Service A  │───▶│  Service B  │───▶│  Service C  │───▶│  Service D  │  │
│  │ (Frontend)  │    │   (API)     │    │  (Worker)   │    │    (DB)     │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│         │                  │                  │                  │          │
│         │    OTel SDK      │    OTel SDK      │    OTel SDK      │          │
│         └────────┬─────────┴────────┬─────────┴────────┬─────────┘          │
│                  │                  │                  │                     │
└──────────────────┼──────────────────┼──────────────────┼─────────────────────┘
                   │                  │                  │
                   │   OTLP (gRPC)    │                  │
                   └────────┬─────────┴────────┬─────────┘
                            │                  │
                            ▼                  ▼
              ┌──────────────────────────────────────────┐
              │           OTel Collector                  │
              │  ┌────────────────────────────────────┐  │
              │  │            Receivers               │  │
              │  │   OTLP | Jaeger | Zipkin | etc.   │  │
              │  └─────────────────┬──────────────────┘  │
              │                    │                     │
              │  ┌─────────────────▼──────────────────┐  │
              │  │           Processors               │  │
              │  │  Batch | Filter | Transform | etc. │  │
              │  └─────────────────┬──────────────────┘  │
              │                    │                     │
              │  ┌─────────────────▼──────────────────┐  │
              │  │            Exporters               │  │
              │  │  OTLP | Jaeger | Prometheus | etc. │  │
              │  └────────────────────────────────────┘  │
              └─────────────────┬────────────────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
  ┌────────────┐        ┌────────────┐        ┌────────────┐
  │   Jaeger   │        │ Prometheus │        │   Loki     │
  │  (Traces)  │        │ (Metrics)  │        │  (Logs)    │
  └──────┬─────┘        └──────┬─────┘        └──────┬─────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               ▼
                        ┌────────────┐
                        │  Grafana   │
                        │ (可视化)    │
                        └────────────┘
```

### 4.2 直接导出 vs Collector 模式

#### 直接导出（Simple）

```
Application ─────────────────────────────────────▶ Backend
            (OTLP/Jaeger/Zipkin/Prometheus)
```

- **优点**：链路简单，延迟低。
- **缺点**：应用承担所有导出负载；切换后端要改应用代码；多后端需要配多个 Exporter。

#### Collector 模式（推荐）

```
Application ──▶ OTel Collector ──▶ Multiple Backends
            OTLP            (处理、路由、转换)
```

- **优点**：
  - 应用与后端解耦（切换后端只改 Collector 配置）
  - Collector 统一做采样、过滤、敏感字段脱敏
  - 支持同时导出到多个后端（一份数据投到 Jaeger + Datadog）
  - Collector 可水平扩展，业务侧无感

> 部署侧的 Collector 配置（含 DaemonSet 模式、与 Tempo 集成等）见 [[09-k8s-observability]] 第四节。

### 4.3 请求追踪流程（生产侧）

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 1: 请求进入 API Gateway                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 生成 trace_id (如果没有)                                                  │
│ • 创建根 Span: "HTTP POST /api/summary"                                    │
│ • 设置 Attributes: http.method, http.url, http.user_agent                  │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ 传播上下文 (traceparent header)
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 2: 业务服务处理                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 提取上下文，创建子 Span: "process_summary"                                │
│ • 记录业务属性: summary_id, scenario, language                             │
│ • 可能创建更多子 Span                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ 调用外部服务
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 3: LLM 调用                                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 创建子 Span: "llm_invoke"                                                 │
│ • 记录: model_name, input_tokens, output_tokens                            │
│ • 记录异常（如果有）                                                        │
│ • 同时记录 Metrics: latency, token_usage, request_count                    │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ 返回结果
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 4: 结束追踪                                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 逐层结束 Span                                                             │
│ • 设置最终状态: OK 或 ERROR                                                 │
│ • 数据通过 Processor 批量处理                                               │
│ • 通过 Exporter 导出到 Collector                                            │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 5: 数据存储与可视化                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│ Collector 接收数据 → 处理 → 导出到:                                         │
│ • Jaeger/Tempo (Traces 存储)                                               │
│ • Prometheus (Metrics 存储)                                                │
│ • Grafana (统一可视化)                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 指标收集流程（生产侧）

```
代码中记录指标
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 1: 创建指标                                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│ meter = metrics.get_meter("my_service")                                    │
│ counter = meter.create_counter("requests_total")                           │
│ histogram = meter.create_histogram("request_latency")                      │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 2: 记录数据点                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ counter.add(1, {"status": "success"})                                      │
│ histogram.record(0.235, {"endpoint": "/api"})                              │
│ • 数据暂存在内存中                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    │ 定期导出 (默认 60s)
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 3: MetricReader 聚合                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 按时间窗口聚合数据                                                        │
│ • Counter: 计算累计值                                                       │
│ • Histogram: 计算分布桶                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 4: 导出到后端                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ OTLPMetricExporter → Collector → Prometheus                                │
│                                                                            │
│ Prometheus 存储后可用于:                                                    │
│ • Grafana Dashboard                                                        │
│ • 告警规则 (AlertManager)                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.5 Grafana 看板渲染流程（消费侧）

Prometheus 只是时序数据库，本身不画图。看板由 Grafana 在浏览器打开后实时拉数据渲染：

```
浏览器打开 Dashboard
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 1: Grafana 解析面板查询                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 读取 Panel 配置：datasource、PromQL 表达式、时间范围、刷新间隔            │
│ • 替换模板变量：${env}、${service} → 实际值                                 │
│ • 计算 step：(end - start) / maxDataPoints                                  │
│   1h 窗口 / 1170px → step ≈ 3s                                             │
│   7d 窗口 / 1170px → step ≈ 600s                                           │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 2: HTTP 请求 Prometheus                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 时序图 → /api/v1/query_range?query=...&start=...&end=...&step=...        │
│ • Stat 单值 → /api/v1/query?query=...&time=...                             │
│ • 表格 → instant 或 range，按面板类型决定                                   │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 3: Prometheus 执行 PromQL                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 在 [start, end] 区间，每隔 step 计算一次表达式                            │
│ • rate(metric[5m]) 等带 range 选择器的函数：每个采样点向前看 5m 窗口        │
│ • 返回 JSON：每条 series（label 组合）一组 (timestamp, value) 数组         │
└─────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 4: Grafana 渲染                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ • 按 Panel 类型（Time series / Heatmap / Stat / Table…）画图                │
│ • Legend、单位、阈值色、Transformation 都在这一步                           │
│ • 按刷新间隔（如 30s）整轮重跑 Step 1~4                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

几个初次使用容易踩的点：

- **step 跟着面板宽度自动变**：同一个 query，把窗口从 1h 拉到 24h，step 会从秒级变到分钟级，曲线会"变平"——不是数据变了，是采样粒度粗了。
- **range vector 必须和 rate / increase / sum_over_time 等函数搭配**：`rate(http_requests_total[5m])` 中的 `[5m]` 是回看窗口，长度需要 ≥ scrape interval × 4 才稳。
- **Instant query 只取一个点**：Stat 面板显示的"当前值"其实是 Grafana 选定时刻往前看一小段的结果，不是真正的"现在"。
- **Auto-refresh ≠ 实时**：刷新间隔最快通常 5s，每次刷新都会整轮重跑全部 PromQL，刷新太频繁会拖垮 Prometheus。
- **时区**：Grafana 默认按浏览器时区渲染轴标签，Prometheus 内部一律 UTC。鼠标悬停看到的"11:00"和后端真实存储的时间戳要按 UTC 换算回去。
- **模板变量 → 多 query**：`${service}` 选了多个值时，Grafana 会展开成多个并发 query，结果按 series 拼回一张图。

> Grafana 的 `histogram_quantile` 与 SQL 的精确分位数为什么对不上，详见 [7.1](#71-histogram-分位数读数陷阱p95--p99)；和 Metabase / Redshift 在 counter 上也对不齐的根因，详见 [7.2](#72-grafanaprometheus-vs-metaberedshiftsql-范式对比)。

---

## 五、Python SDK 实操指南

### 5.1 依赖安装

```bash
# 核心包
pip install opentelemetry-api opentelemetry-sdk

# Exporters
pip install opentelemetry-exporter-otlp          # OTLP (推荐)
pip install opentelemetry-exporter-jaeger        # Jaeger
pip install opentelemetry-exporter-prometheus    # Prometheus

# Auto-instrumentation (自动检测)
pip install opentelemetry-instrumentation-fastapi
pip install opentelemetry-instrumentation-requests
pip install opentelemetry-instrumentation-sqlalchemy
pip install opentelemetry-instrumentation-redis
pip install opentelemetry-instrumentation-logging
```

### 5.2 初始化模板（生产可复用）

下面是一个可以直接放到新项目里的 `observability.py`，覆盖 Resource 定义、Trace + Metric Provider、控制台调试导出、采样率、优雅关闭。

```python
"""OpenTelemetry 初始化模块"""

from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import (
    PeriodicExportingMetricReader,
    ConsoleMetricExporter,
)
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased
import logging

logger = logging.getLogger(__name__)


def init_observability(
    service_name: str,
    service_version: str = "1.0.0",
    environment: str = "dev",
    otlp_endpoint: str = None,
    trace_sampling_rate: float = 1.0,
    enable_console_export: bool = False,
    metric_export_interval_ms: int = 60000,
):
    """初始化 OpenTelemetry。

    Args:
        service_name: 服务名称
        service_version: 服务版本
        environment: 运行环境 (dev/staging/prod)
        otlp_endpoint: OTLP Collector 端点
        trace_sampling_rate: 追踪采样率 (0.0-1.0)
        enable_console_export: 是否输出到控制台（调试用）
        metric_export_interval_ms: 指标导出间隔（毫秒）
    """

    # 1. 定义服务资源（所有 Span/Metric 都会带上这些标识）
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": environment,
    })

    # 2. 配置 Traces
    sampler = TraceIdRatioBased(trace_sampling_rate)
    trace_provider = TracerProvider(resource=resource, sampler=sampler)

    if enable_console_export:
        trace_provider.add_span_processor(
            BatchSpanProcessor(ConsoleSpanExporter())
        )
    if otlp_endpoint:
        trace_provider.add_span_processor(
            BatchSpanProcessor(
                OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
            )
        )
    trace.set_tracer_provider(trace_provider)

    # 3. 配置 Metrics
    metric_readers = []
    if enable_console_export:
        metric_readers.append(
            PeriodicExportingMetricReader(
                ConsoleMetricExporter(),
                export_interval_millis=metric_export_interval_ms,
            )
        )
    if otlp_endpoint:
        metric_readers.append(
            PeriodicExportingMetricReader(
                OTLPMetricExporter(endpoint=otlp_endpoint, insecure=True),
                export_interval_millis=metric_export_interval_ms,
            )
        )
    if metric_readers:
        meter_provider = MeterProvider(
            resource=resource,
            metric_readers=metric_readers,
        )
        metrics.set_meter_provider(meter_provider)

    logger.info(
        f"OpenTelemetry initialized: service={service_name}, "
        f"endpoint={otlp_endpoint}, sampling_rate={trace_sampling_rate}"
    )


def shutdown_observability():
    """关闭 OpenTelemetry（应在进程优雅退出时调用，避免最后一批数据丢失）。"""
    trace_provider = trace.get_tracer_provider()
    if hasattr(trace_provider, 'shutdown'):
        trace_provider.shutdown()

    meter_provider = metrics.get_meter_provider()
    if hasattr(meter_provider, 'shutdown'):
        meter_provider.shutdown()
```

### 5.3 Traces 写法

#### 方式一：上下文管理器（推荐）

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_order(order_id: str):
    with tracer.start_as_current_span("process_order") as span:
        # 设置属性
        span.set_attribute("order.id", order_id)
        span.set_attribute("order.type", "standard")

        # 嵌套 Span
        with tracer.start_as_current_span("validate_order") as child_span:
            validate_result = validate(order_id)
            child_span.set_attribute("validation.result", validate_result)

        # 记录事件
        span.add_event("order_validated", {"order_id": order_id})

        try:
            result = execute_order(order_id)
            span.set_status(trace.StatusCode.OK)
            return result
        except Exception as e:
            span.set_status(trace.StatusCode.ERROR, str(e))
            span.record_exception(e)
            raise
```

#### 方式二：装饰器（适合大量函数批量插桩）

```python
from opentelemetry import trace
from functools import wraps

def traced(span_name: str = None):
    """追踪装饰器。"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            tracer = trace.get_tracer(__name__)
            name = span_name or func.__name__
            with tracer.start_as_current_span(name) as span:
                span.set_attribute("code.function", func.__qualname__)
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    span.record_exception(e)
                    span.set_status(trace.StatusCode.ERROR)
                    raise
        return wrapper
    return decorator

@traced("fetch_user_data")
def get_user(user_id: str):
    return db.query(User).filter_by(id=user_id).first()
```

### 5.4 Metrics 写法

参见 [3.6](#36-三种-instrument-的代码示例)。下面给一个**业务里同时使用 Counter + Histogram + UpDownCounter** 的完整模式：

```python
def handle_request(request):
    start_time = time.time()

    active_connections.add(1)
    try:
        response = process(request)
        request_counter.add(1, {
            "method": request.method,
            "status": "success"
        })
        return response
    except Exception:
        request_counter.add(1, {
            "method": request.method,
            "status": "error"
        })
        raise
    finally:
        active_connections.add(-1)
        latency_histogram.record(
            time.time() - start_time,
            {"endpoint": request.path}
        )
```

### 5.5 通用工具：装饰器与 MetricsHelper（进阶）

下面是一个把 Traces + Metrics 工具沉淀到新项目里的模板，可放在 `otel_utils.py`。

`traced` 装饰器同时支持同步和异步函数：

```python
"""OpenTelemetry 工具函数"""

import time
import functools
import asyncio
from typing import Optional, Dict, Any
from opentelemetry import trace, metrics


def traced(
    span_name: str = None,
    attributes: Dict[str, Any] = None,
    record_exception: bool = True,
):
    """同步/异步通用追踪装饰器。

    Usage:
        @traced("process_order")
        def process_order(order_id: str): ...
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            tracer = trace.get_tracer(__name__)
            name = span_name or func.__name__
            with tracer.start_as_current_span(name) as span:
                span.set_attribute("code.function", func.__qualname__)
                span.set_attribute("code.filepath", func.__code__.co_filename)
                if attributes:
                    for key, value in attributes.items():
                        span.set_attribute(key, value)
                try:
                    result = func(*args, **kwargs)
                    span.set_status(trace.StatusCode.OK)
                    return result
                except Exception as e:
                    span.set_status(trace.StatusCode.ERROR, str(e))
                    if record_exception:
                        span.record_exception(e)
                    raise

        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            tracer = trace.get_tracer(__name__)
            name = span_name or func.__name__
            with tracer.start_as_current_span(name) as span:
                span.set_attribute("code.function", func.__qualname__)
                if attributes:
                    for key, value in attributes.items():
                        span.set_attribute(key, value)
                try:
                    result = await func(*args, **kwargs)
                    span.set_status(trace.StatusCode.OK)
                    return result
                except Exception as e:
                    span.set_status(trace.StatusCode.ERROR, str(e))
                    if record_exception:
                        span.record_exception(e)
                    raise

        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return wrapper
    return decorator
```

`MetricsHelper` 把 Counter / Histogram 的"create-once、record-many"模式封装起来，避免重复样板代码：

```python
class MetricsHelper:
    """指标助手类。"""

    def __init__(self, meter_name: str):
        self.meter = metrics.get_meter(meter_name)
        self._counters = {}
        self._histograms = {}

    def counter(self, name, value=1, attributes=None, description=""):
        if name not in self._counters:
            self._counters[name] = self.meter.create_counter(
                name=name, description=description, unit="1",
            )
        self._counters[name].add(value, attributes or {})

    def histogram(self, name, value, attributes=None, description="",
                  unit="ms", buckets=None):
        if name not in self._histograms:
            kwargs = {"name": name, "description": description, "unit": unit}
            if buckets:
                kwargs["explicit_bucket_boundaries_advisory"] = buckets
            self._histograms[name] = self.meter.create_histogram(**kwargs)
        self._histograms[name].record(value, attributes or {})

    def timed(self, name, attributes=None, unit="s"):
        return TimedContext(self, name, attributes, unit)


class TimedContext:
    """计时上下文管理器，自动把 with 块的耗时记成 histogram。"""

    def __init__(self, helper, name, attributes, unit):
        self.helper = helper
        self.name = name
        self.attributes = attributes or {}
        self.unit = unit
        self.start_time = None

    def __enter__(self):
        self.start_time = time.time()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        duration = time.time() - self.start_time
        if self.unit == "ms":
            duration *= 1000
        self.attributes["status"] = "error" if exc_type else "success"
        self.helper.histogram(self.name, duration, self.attributes, unit=self.unit)
        return False
```

### 5.6 FastAPI 集成示例

把上面所有件拼起来：

```python
from fastapi import FastAPI, Request
from contextlib import asynccontextmanager
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

from observability import init_observability, shutdown_observability
from otel_utils import traced, MetricsHelper
from config import load_config


config = load_config()
metrics_helper = MetricsHelper("my_service")


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    otel_config = config.get("otel", {})
    init_observability(
        service_name=otel_config.get("service_name", "my-service"),
        service_version=otel_config.get("service_version", "1.0.0"),
        environment=config.get("environment", "dev"),
        otlp_endpoint=otel_config.get("otlp_endpoint"),
        trace_sampling_rate=otel_config.get("trace_sampling_rate", 1.0),
        enable_console_export=otel_config.get("enable_console_export", False),
    )

    # 自动检测 FastAPI（自动给每个请求创建 root span）
    FastAPIInstrumentor.instrument_app(app)

    yield

    # Shutdown：必须调用，否则最后一批 batch 可能丢
    shutdown_observability()


app = FastAPI(lifespan=lifespan)


@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    """请求指标中间件。"""
    with metrics_helper.timed(
        "http_request_duration_seconds",
        attributes={"method": request.method, "path": request.url.path},
        unit="s",
    ):
        response = await call_next(request)
        metrics_helper.counter(
            "http_requests_total",
            attributes={
                "method": request.method,
                "path": request.url.path,
                "status": str(response.status_code),
            },
        )
        return response


@app.post("/api/process")
@traced("api_process")
async def process_data(data: dict):
    span = trace.get_current_span()
    span.set_attribute("data.size", len(str(data)))
    result = await do_processing(data)
    return {"status": "success", "result": result}
```

配套的 `config.yaml`：

```yaml
otel:
  enabled: true
  service_name: "my-new-service"
  service_version: "1.0.0"

  # OTLP Collector 端点
  otlp_endpoint: "http://otel-collector:4317"

  # 采样率 (0.0-1.0)
  # 开发环境: 1.0 (100%)
  # 生产环境: 0.1-0.5 (10%-50%)
  trace_sampling_rate: 1.0

  # 调试模式：输出到控制台
  enable_console_export: false

  # 指标导出间隔（毫秒）
  metric_export_interval_ms: 60000
```

### 5.7 OTel Collector 部署（docker-compose）

本地起一套完整环境（OTel Collector + Jaeger + Prometheus + Grafana）：

```yaml
# docker-compose.yaml
version: '3.8'

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus metrics
    depends_on:
      - jaeger
      - prometheus

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14250:14250"  # gRPC

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana

volumes:
  grafana-storage:
```

`otel-collector-config.yaml`：

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  # 内存限制：超过阈值时拒绝新数据，避免 OOM
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  # 属性处理：统一注入环境标识
  attributes:
    actions:
      - key: environment
        value: production
        action: upsert

exporters:
  # Jaeger (Traces)
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  # Prometheus (Metrics)
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: myapp

  # 调试输出（新版 Collector 已改名为 debug）
  debug:
    verbosity: detailed

extensions:
  health_check:
  pprof:
  zpages:

service:
  extensions: [health_check, pprof, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger, debug]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

> 生产环境的 K8s DaemonSet 部署模式、与 Tempo 集成等更复杂的配置见 [[09-k8s-observability]] 第四节「Traces——OTel Collector + Tempo」。

---

## 六、当前项目实现参考（项目特定）

> 本节内容来自当前业务代码（`plaud_summary` 系列服务），仅作为"在真实项目里这些组件长什么样"的参考。新项目集成请直接参照 [第五章](#五python-sdk-实操指南)。

### 6.1 初始化封装

**文件**: `plaud_summary/obser.py`

```python
from plaud_library_python import init_observability

def init_observer(service_name: str, run_env: str):
    otel_conf = config.get_otel_conf()

    init_observability(
        service_name,
        service_version="latest",
        environment=run_env,
        enable_console_export=otel_conf.enable_console_export,
        otlp_endpoint=otel_conf.otlp_endpoint,
        trace_sampling_rate=otel_conf.trace_sampling_rate,
    )
```

**配置项** (`aws_appconfig/dev.yaml`)：

```yaml
otel:
  enable_otel_exporter: true
  otlp_endpoint: "http://dev-olcc-prometheus-proxy.plaud.ai:4317"
  trace_sampling_rate: 1.0      # 100% 采样
  enable_console_export: false  # 生产关闭
```

### 6.2 @trace_span 装饰器

**文件**: `tools/decorators.py`

```python
@trace_span(
    trace_name="summary_service",
    span_name="generate_summary",
    with_meter=True,
    option={
        "latency_unit": "s",
        "explicit_bucket_boundaries_advisory": [0.1, 0.5, 1.0, 5.0, 10.0]
    }
)
def generate_summary(content: str):
    # 自动创建 Span
    # 自动记录 Counter 和 Latency Histogram
    pass
```

### 6.3 LLMMonitorCallbackHandler

**文件**: `plaud_summary/callbacks/timing_callback.py`

```python
class LLMMonitorCallbackHandler(BaseCallbackHandler):
    """LLM 调用监控。"""

    def on_llm_start(self, serialized, prompts, **kwargs):
        # 记录开始时间
        # 估算 input tokens
        # 记录模型信息
        pass

    def on_llm_end(self, response, **kwargs):
        # 计算延迟
        # 提取 token 使用量
        # 记录 metrics: counter, latency, tokens
        pass

    def on_llm_error(self, error, **kwargs):
        # 记录错误
        # 统计错误类型
        pass
```

使用：

```python
callback = LLMMonitorCallbackHandler(
    summary_id="xxx",
    scenario="meeting"
)
result = llm.invoke(prompt, config={"callbacks": [callback]})
```

### 6.4 Summary Task Metrics

**文件**: `plaud_summary/summary_task_metrics.py`

```python
# Token 桶边界配置（按真实 token 分布设计，详见 7.1.4 桶设计原则）
TOKEN_BUCKETS = [0, 1024, 5120, 10240, 25600, 51200, 76800, 102400, 204800]

def record_summary_task_metrics(summary_id, result, llm, scenario, language, spend):
    """记录总结任务的所有监控指标。"""

    attributes = {
        "final_use_model": result.get("result", {}).get("endpoint", ""),
        "scenario": scenario.lower(),
    }

    summary_task_counter(1, attributes)                                     # 计数器
    summary_task_latency(spend, attributes)                                  # 延迟直方图
    summary_task_token_usage(input_tokens, "input", attributes)              # Token 使用量
    summary_task_token_usage(output_tokens, "output", attributes)
    summary_task_token_usage(total_tokens, "total", attributes)
```

---

## 七、深度专题（深入）

> 本章三个专题相互独立，但都围绕"Metrics 在生产环境的真实表现"——它们是把笔记从"会用"提升到"踩过坑"的关键内容。

### 7.1 Histogram 分位数读数陷阱（P95 / P99）

> **一句话**：Grafana 上看到的 `histogram_quantile(...)` 永远是**估算值**，不是精确 P95/P99；它和 Metabase / SQL 里 `PERCENTILE_CONT(0.95)` 算出来的数一定有偏差。这不是 bug，是直方图的先天特性。

#### 7.1.1 为什么有偏差

Prometheus 存 Histogram **只存每个桶的累计计数**，**不保留原始样本值**。例如某 1 小时内 100 个任务的延迟分布：


| 桶 `le` | 累计计数 |
| ------ | ---- |
| ≤ 150s | 85   |
| ≤ 175s | 92   |
| ≤ 200s | 96   |
| ≤ 300s | 100  |


- **SQL 的精确算法**：有全部 100 个原始值，排序后取第 95 名，假设真实是 **179.2s**。
- **PromQL `histogram_quantile(0.95)` 的算法**：只能看到上表，知道第 95 名"在 (175, 200] 这桶里"。它**假设这 4 个任务（93–96 名）均匀分布在 175~200 之间**，线性插值：
  ```
  P95 ≈ 175 + (200 - 175) × (95 - 92) / (96 - 92)
      = 175 + 25 × 3/4 = 193.75s
  ```
  **误差 ~14s**。桶越稀疏 / 越靠尾部，误差越大。

#### 7.1.2 精度取决于桶边界密度


| 区间        | 桶边界                          | 插值误差上界 |
| --------- | ---------------------------- | ------ |
| 30~50s    | 若只有 30, 50                   | ±20s   |
| 30~50s    | 若有 30, 35, 40, 45, 50        | ±5s    |
| 200~1000s | 若只有 200, 300, 500, 750, 1000 | 百秒级    |


**在实际分布集中的区间加密桶**，尾部可以稀疏。反过来：桶很多但都落在没有样本的区域，也没用。

#### 7.1.3 OpenTelemetry Python 默认桶不适合所有场景

默认桶（不显式配置 `explicit_bucket_boundaries_advisory` 时）：

```
5, 10, 20, 30, 50, 75, 100, 125, 150, 175, 200, 300, 500, 750, 1000, 1500, 2000, 2500, 3000, 4000, 5000, 7500, 10000
```

- **适合**：毫秒~百秒量级的通用 HTTP 请求延迟。
- **不适合**：
  - 秒级以下（例如 DB 查询 P95 = 50ms，全落进 ≤ 5s 这一个桶，完全看不到分布）。
  - 长任务（例如 AI summary 任务典型 30~300s，尾部 500 之后只有 4 个桶 `500/750/1000/1500`，P99 几乎全靠猜）。

#### 7.1.4 桶设计建议

```python
# 示例：AI 总结任务，典型 30~300s，尾部 ~1800s
_summary_task_latency = meter.create_histogram(
    name="summary_task_latency",
    unit="s",
    explicit_bucket_boundaries_advisory=[
        5, 10, 20, 30, 45, 60, 90, 120,
        150, 180, 210, 240, 270, 300,   # 密集覆盖主分布
        360, 420, 480, 600, 900, 1200, 1800, 3600,
    ],
)
```

原则：

1. **先画一次真实分布**（用 SQL 直方图 / Grafana `histogram_quantile` 查几个分位数）。
2. **主分布区间每 10~30% 一个桶**，P95/P99 所在区间尤其要密。
3. **桶数量控制在 15~30**（每多一个桶 = 每个 label 组合多一条 series，基数成本线性增长——见 [3.4](#34-attributes-与时间序列基数重点)）。
4. 加桶是**破坏性变更**：Prometheus 里已有历史数据的 series 会出现"缺桶"或"多桶"，`histogram_quantile` 跨这个时间点会算错。如需切换，最好新起指标名或标注 version label。

#### 7.1.5 小抄

- 看见 P95/P99 先问一句"这是 histogram 估的还是 SQL 算的"。
- Grafana 上 P95 抖几秒，别急着查业务，先看桶够不够密。
- 新建 Histogram **一律显式配 `explicit_bucket_boundaries_advisory`**，别信默认。
- 需要精确分位数 → OpenTelemetry 侧开启 **Exponential Histogram**（指数桶，自适应分布；目前已成为新项目推荐）；或直接用 SQL。

### 7.2 Grafana(Prometheus) vs Metabase(Redshift/SQL) 范式对比

7.1 节聚焦 histogram 分位数差异。但即便是单纯的 counter 计数（"今天 11 点跑了多少任务"），两个系统也常对不上。根因是两套范式完全不同。

#### 7.2.1 范式差异


| 维度    | Metabase + Redshift | Grafana + Prometheus                          |
| ----- | ------------------- | --------------------------------------------- |
| 数据模型  | 行级事实表，每个事件一行        | 时序聚合，label 组合 → 时间桶 → 数值                      |
| 写入路径  | 业务代码 INSERT，事务保证    | SDK 累计 → 60s 批量导出 → Collector → Prometheus 拉取 |
| 写入时机  | 每个事件一次              | 进程内累加，60s 一批；Prometheus 每 15~60s scrape 一次    |
| 数据完整性 | 完整事件流，可补录           | 进程崩溃前未导出的批次会丢；scrape 间隔内的细节被聚合                |
| 时间粒度  | 任意精确时间戳             | 受 scrape interval + step 限制，秒级以下基本不可见         |
| 基数承受  | 行级，几十亿行可查           | label 笛卡尔积，每条 series 都吃内存                     |
| 历史回填  | 支持（INSERT 历史时间戳）    | 不支持，超过 lookback 窗口拒收                          |
| 查询语义  | SQL，事件级聚合精确         | PromQL，区间外推 + 桶插值，估算                          |
| 适用场景  | 业务报表、对账、漏斗、留存       | 实时监控、告警、SLO、同环比                               |


#### 7.2.2 同一时间点为什么对不齐

同一指标（例如"成功完成的总结任务数"）、同一时间点（例如 11:00），两边数字不一致，按以下顺序排查：

1. **时间窗口语义不同**
  - Metabase：`WHERE created_at >= '2026-04-27 11:00' AND < '12:00'`，左闭右开 1 小时整。
  - Grafana：`increase(summary_task_total[1h])` 在 11:00 这个数据点，统计的是 `(10:00, 11:00]`，不是"11 点这一个小时"。
  - 同样问"11 点的数"，两套语义差了 1 小时。
2. **时区**：Metabase 通常按用户/问题级时区渲染（常见 America/Los_Angeles 或 Asia/Shanghai），Grafana 按浏览器时区，Prometheus 存 UTC。差几小时通常是这里。
3. **样本集不同**
  - Metric 只在代码显式 `record()` / `add()` 的分支里 +1。`if result_should_log:` 之类过滤会让 metric 少于 DB 行数。
  - Retry：Counter 通常每次重试都计一次；DB 一般只插最终结果一行。
  - 失败 / 中断 / 超时 / moderation 拦截：DB 多半会留行，metric 视实现可能完全不计。
4. **Scrape / 导出丢点**
  - SDK 默认 60s 批量导出，进程在导出前被 kill → 这一批样本丢。
  - Pod 在两次 scrape 之间存在又消失（短任务、HPA 缩容，相关机制见 [[12-k8s-pod-graceful-shutdown]]）→ Prometheus 完全没看见。
  - Collector 到 Prometheus 之间网络抖动 → remote_write 重试失败的部分会丢。
5. **Counter 重启外推**：进程重启后 counter 从 0 开始，Prometheus 检测到回退按 reset 处理，会把"重启前最后一次值 → 0"这段补成 delta。新进程在第一次 scrape 之前已经处理的请求会被漏算。
6. `**rate()` / `increase()` 的边界外推**：Prometheus 用区间内实际样本做线性外推到端点，区间端点附近误差几个百分点是常态。
7. **Histogram 桶插值**：见 [7.1](#71-histogram-分位数读数陷阱p95--p99)，分位数估算误差。
8. **数据源分片**：多 region 部署时每个 region 一套 Prometheus；Grafana 里 `${datasource}` 变量只指一个，Metabase 的 Redshift 通常是全局合并表。
9. **去重 / 唯一约束**：DB 有 unique key，重复事件被吞；Counter 不去重，每次调用都 +1。

#### 7.2.3 实操原则

- **对账、汇报数字、KPI** → 一律以 SQL（Metabase）为准。Prometheus 不参与对账。
- **告警、容量、趋势、SLO、值班看板** → 一律以 Grafana / PromQL 为准。SQL 跑不出秒级粒度，也跑不动每 30s 一次。
- **看到两边数字不一致先别 debug 业务逻辑**，先按上面 1~9 排查，90% 的情况是窗口 / 时区 / 样本集差异。
- **不要试图让两个数字完全相等**。设计指标时就接受"近似"，把精确数字留给数仓。

### 7.3 Cumulative vs Delta：选错就断崖

3.3.2 节已经介绍过两种 Temporality。这里补充实际部署时的踩坑模式：


| 配置                                | 后端             | 现象                                                                         |
| --------------------------------- | -------------- | -------------------------------------------------------------------------- |
| Cumulative + Prometheus（默认）       | Prometheus     | ✓ 正常                                                                       |
| **Delta + Prometheus**            | Prometheus     | **断崖式下跌**：Prometheus 把 delta 当 cumulative，每次新批次 < 上次累计值 → 触发 reset 检测，曲线归零 |
| Delta + ClickHouse / Datadog      | OTLP-native 后端 | ✓ 正常                                                                       |
| Cumulative + ClickHouse / Datadog | OTLP-native 后端 | 后端需要在 ingest 时计算 delta，可能产生重复或丢失                                           |


**经验法则**：导出端选 Cumulative 还是 Delta，**完全取决于直接接收方是谁**，不取决于"哪个更先进"。Prometheus 永远 Cumulative。

---

## 八、最佳实践

### 8.1 Span 命名规范

```python
# 好的命名 - 描述操作而非实现细节
"HTTP GET /api/users"
"DB query users"
"Cache get user:123"
"LLM invoke gpt-4"

# 不好的命名 - 太笼统或包含高基数值
"request"
"process"
"HTTP GET /api/users/12345"  # 用户ID应作为属性
```

### 8.2 Attributes 最佳实践

```python
# 使用语义约定 (Semantic Conventions)
span.set_attribute("http.method", "POST")
span.set_attribute("http.url", "https://api.example.com/users")
span.set_attribute("http.status_code", 200)
span.set_attribute("db.system", "mysql")
span.set_attribute("db.statement", "SELECT * FROM users WHERE id = ?")

# 避免高基数属性（会导致存储爆炸）
# 不要: span.set_attribute("user.email", "user@example.com")
# 应该: span.set_attribute("user.id", "12345")  # 使用 ID
```

完整列表见 [附录 A](#a-语义约定参考) 或 [https://opentelemetry.io/docs/specs/semconv/](https://opentelemetry.io/docs/specs/semconv/)。

### 8.3 采样策略


| 环境  | 采样率    | 原因       |
| --- | ------ | -------- |
| 开发  | 100%   | 便于调试     |
| 测试  | 100%   | 完整追踪     |
| 预发布 | 50%    | 平衡性能和可见性 |
| 生产  | 10-20% | 高流量下避免开销 |


```python
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ParentBased

# 自适应采样（基于 trace_id 的尾部采样）
sampler = ParentBased(
    root=TraceIdRatioBased(0.1),  # 10% 采样
)
```

`ParentBased` 的好处：上游已经决定采样的 trace，下游一律继承（不会出现"半截 trace"）。

### 8.4 Metrics 标签设计

```python
# 低基数标签（推荐）
attributes = {
    "method": "POST",           # 有限值：GET, POST, PUT, DELETE
    "status": "success",        # 有限值：success, error
    "endpoint": "/api/summary", # 路由模板，不含参数
    "scenario": "meeting",      # 业务枚举
}

# 避免高基数标签
# 不要包含：user_id, request_id, timestamp, 具体错误消息
```

详细解释见 [3.4](#34-attributes-与时间序列基数重点)。

### 8.5 性能与批量参数

```python
# 1. 批量处理：减少导出次数，提高吞吐
BatchSpanProcessor(
    exporter,
    max_queue_size=2048,        # 队列大小
    schedule_delay_millis=5000, # 导出间隔
    max_export_batch_size=512,  # 批次大小
)

# 2. 异步导出：默认就是异步，不会阻塞主线程

# 3. 合理设置导出间隔
PeriodicExportingMetricReader(
    exporter,
    export_interval_millis=60000,  # 1 分钟，生产环境推荐
)

# 4. Collector 侧的内存限制
# memory_limiter:
#   limit_mib: 512
#   spike_limit_mib: 128
```

---

## 九、常见问题与调试

### 9.1 数据未导出

排查清单：

1. **Endpoint 是否正确**
  ```python
   logger.info(f"OTLP endpoint: {otlp_endpoint}")
  ```
2. **网络是否通畅**
  ```bash
   curl -v http://otel-collector:4317
  ```
3. **是否调用了 set_tracer_provider**
  ```python
   trace.set_tracer_provider(provider)  # 必须调用！
  ```
4. **启用控制台导出调试**
  ```python
   init_observability(enable_console_export=True)
  ```
5. **进程退出前是否调用 `shutdown_observability()`**：BatchProcessor 默认 5s 才导出一次，进程立刻退出会丢最后一批。

### 9.2 Span 未关联

**原因**: 上下文未正确传播（线程、asyncio Task）。

线程场景：

```python
# 错误：在新线程中丢失上下文
import threading

def worker():
    tracer.start_span("child")  # 不会关联到父 span

# 正确：传递上下文
from opentelemetry import context

def worker(ctx):
    context.attach(ctx)
    with tracer.start_as_current_span("child"):
        pass

ctx = context.get_current()
thread = threading.Thread(target=worker, args=(ctx,))
```

asyncio 场景的完整解决方案见 [10. 实战案例](#十实战案例asyncio-后台任务的上下文传播深入)。

### 9.3 指标基数爆炸

**症状**: Prometheus 内存持续增长。

**原因**: 标签包含高基数值（详见 [3.4.4](#344-为什么高基数会导致灾难)）。

```python
# 问题代码
histogram.record(latency, {"user_id": user_id})  # 每个用户一个时间序列

# 修复
histogram.record(latency, {"user_type": user.type})  # 使用低基数分类
```

### 9.4 调试命令

```bash
# 查看 Collector 日志
docker logs otel-collector -f

# 检查 Prometheus 目标
curl http://localhost:9090/api/v1/targets

# 查看 Jaeger 服务
curl http://localhost:16686/api/services

# 检查 OTLP 健康状态
curl http://localhost:13133/health
```

### 9.5 常用查询

**Prometheus (PromQL)**：

```promql
# 请求速率 (QPS)
rate(http_requests_total[5m])

# P99 延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 错误率
sum(rate(http_requests_total{status="error"}[5m]))
/ sum(rate(http_requests_total[5m]))
```

**Jaeger**：

```
# 查找慢请求
service=my-service AND duration>1s

# 查找错误
service=my-service AND error=true
```

---

## 十、实战案例：asyncio 后台任务的上下文传播（深入）

> **场景**：FastAPI 端点收到请求后，通过 `asyncio.create_task` 将耗时处理调度为后台任务。由于 `asyncio.create_task` **不会自动传播 OTel 上下文**，后台任务中的所有日志和 Span 都会丢失原始请求的 `trace_id`，导致无法通过 `task_id` 关联到完整的 Trace 链路。
>
> 本案例的业务背景与 Temporal 后台任务相似（参考 [[temporal_notes]]），但本节聚焦的是纯 asyncio 场景下的 OTel 上下文丢失。

### 10.1 问题分析

```
HTTP POST /temporal/project-summary/process
  │
  ├─ FastAPIInstrumentor 自动创建 Request Span (含 trace_id)
  │
  ├─ endpoint handler 返回 202 ──→ Request Span 结束
  │
  └─ schedule_background_task(process_temporal_summary_task(...))
       │
       └─ asyncio.create_task(coro)  ← 这里丢失了 OTel 上下文!
            │
            └─ 后台协程中所有日志的 trace_id = "0" (无法关联)
```

**根因**：OTel Python SDK 使用 `contextvars.ContextVar` 存储当前 Span/Context。虽然 `asyncio.create_task` 会复制 Python 的 `contextvars.Context`，但 OTel 的 context 层在其之上有一套独立的管理机制，不保证在新的 asyncio Task 中自动可用。

### 10.2 三层绑定方案

#### 10.2.1 第一层：OTel 上下文传播（核心）

在 `schedule_background_task` 中，创建 asyncio Task 之前捕获当前 OTel 上下文快照，并在后台协程中恢复：

```python
from opentelemetry import context as otel_context

def schedule_background_task(coro):
    # 在创建 asyncio.Task 之前，拍一个当前 OTel 上下文的快照
    ctx = otel_context.get_current()

    async def _run_with_context():
        # 在后台协程开始时，把快照恢复到当前协程的 ContextVar 中
        token = otel_context.attach(ctx)
        try:
            await coro
        finally:
            otel_context.detach(token)

    task = asyncio.create_task(_run_with_context())
    return task
```

修改后的数据流：

```
HTTP Request
  │
  ├─ Request Span [trace_id=abc123]
  │
  └─ schedule_background_task
       │
       ├─ ctx = get_current()         ← 快照：保存 trace_id=abc123
       │
       └─ asyncio.create_task(_run_with_context)
            │
            ├─ attach(ctx)            ← 恢复：trace_id=abc123 生效
            │
            └─ process_temporal_summary_task
                 │  所有日志自动带 trace_id=abc123
                 └─ ...
```

#### 10.2.2 第二层：Span Attribute 绑定 task_id

在 **HTTP 端点**中，将 `task_id` 写入 FastAPI 自动创建的 Request Span 的 attributes：

```python
from opentelemetry import trace

def set_span_attribute(key: str, value: str) -> None:
    span = trace.get_current_span()
    if span.is_recording():
        span.set_attribute(key, value)

# 在端点 handler 中
set_span_attribute("task_id", request.task_id)
```

在 **后台任务**中，创建一个新的子 Span，它继承了第一层传播过来的 `trace_id`，同时携带 `task_id` 等业务字段：

```python
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span(
    "process_temporal_summary_task",
    attributes={
        "task_id": task_id,
        "project_id": request.project_id,
        "user_id": request.user_id,
        "strategy": request.strategy or "auto",
        "file_count": len(request.source_list),
    },
):
    # 整个后台处理逻辑...
    pass
```

#### 10.2.3 第三层：日志显式记录绑定关系

在端点和后台任务入口处，同时打印 `task_id` 和 `trace_id`：

```python
def get_current_trace_id() -> str | None:
    span = trace.get_current_span()
    ctx = span.get_span_context()
    if ctx and ctx.trace_id:
        return format(ctx.trace_id, "032x")
    return None

# 端点日志
logger.info(
    "Received summary request: task_id=%s, trace_id=%s",
    request.task_id,
    get_current_trace_id(),
)

# 后台任务日志
logger.info(
    "[%s] Starting async processing... (trace_id=%s)",
    task_id,
    get_current_trace_id(),
)
```

即使不去 tracing 后端，只查 CloudWatch 日志，`grep task_id` 也能直接看到对应的 `trace_id`。

#### 10.2.4 最终效果

```
┌─ Trace abc123 ──────────────────────────────────────────────┐
│                                                              │
│  [Span] POST /temporal/project-summary/process               │
│         attributes: {task_id: "xxx"}                         │
│         |                                                    │
│         └─[Span] process_temporal_summary_task               │
│                  attributes: {task_id: "xxx",                │
│                               project_id: "...",             │
│                               file_count: 5, ...}           │
│                  |                                           │
│                  ├─ (子 span: HTTP calls, S3, LLM...)        │
│                  └─ ...                                      │
└──────────────────────────────────────────────────────────────┘
```

三条查询路径互通：

- **task_id → trace_id**：查日志 `grep task_id=xxx`，看到 `trace_id=abc123`
- **trace_id → task_id**：在 Jaeger/Tempo 中打开 trace `abc123`，span attributes 里有 `task_id`
- **后台日志自动关联**：因为 OTel 上下文已传播，`LoggingInstrumentor` 自动在后台任务的每一条日志中注入 `trace_id` 和 `span_id`

### 10.3 ContextVar + 日志格式化器覆盖 trace_id（进阶）

> **动机**：10.2 实现了 OTel `trace_id` 的传播和 `task_id` 的 Span Attribute 绑定，但日志中的 `trace_id` 字段仍然是 OTel 自动生成的 32 位十六进制值（如 `480691584f702b103110318efdcca515`），无法直接通过业务 `task_id` 搜索日志。本节介绍如何让日志的 `trace_id` 字段直接显示为 `task_id`。

#### 10.3.1 问题

```
# 修改前：trace_id 是 OTel 自动生成的值，与 task_id 无直接关联
{"level": "INFO", "message": "Loaded agent prompt...", "trace_id": "480691584f702b103110318efdcca515", "span_id": "7c4ef9a26fd11193"}

# 修改后：trace_id 直接就是业务 task_id
{"level": "INFO", "message": "Loaded agent prompt...", "trace_id": "my-temporal-task-123", "span_id": "7c4ef9a26fd11193"}
```

#### 10.3.2 方案

核心思路：通过 Python 标准库 `contextvars` 在异步调用链中传递 `task_id`，在 JSON 日志格式化器中用它覆盖 OTel 注入的 `trace_id`。

```
HTTP handler (temporal.py)
  ├─ set_current_task_id(task_id)    ← 最早设置，覆盖请求处理阶段所有日志
  ├─ logger.info("Received...")      ← trace_id = task_id ✓
  ├─ logger.warning("Rejecting...")  ← trace_id = task_id ✓
  └─ schedule_background_task()
       ├─ asyncio.create_task(...)   ← 复制当前 context（已含 task_id）
       ├─ logger.info("scheduled...") ← trace_id = task_id ✓
       └─ background task
            └─ process_temporal_summary_task()
                 ├─ set_current_task_id(task_id)  ← 冗余但无害
                 ├─ run_summary_agent()
                 │    ├─ ensure_file_summaries()   ← asyncio.gather 继承 context ✓
                 │    ├─ detect_language()          ✓
                 │    └─ agent.ainvoke()            ✓
                 │         ├─ prompt loading        ✓
                 │         └─ mapreduce gather      ← asyncio.gather 继承 context ✓
                 └─ send_workflow_signal()          ✓
```

> **关键：必须在 HTTP handler 入口处设置 `set_current_task_id`**，而不是仅在后台任务中设置。原因：
>
> 1. handler 中的日志（如 "Received summary request"、"Rejecting task"）在 `schedule_background_task` 之前执行，若不提前设置则这些日志仍使用 OTel 自动生成的 trace_id
> 2. `asyncio.create_task()` 在创建时复制当前 context 快照——若此时 `_current_task_id` 尚未设置，后台任务会继承 `None`
> 3. `process_temporal_summary_task` 内部的 `set_current_task_id` 调用变为冗余保险（同一个值设置两次无副作用）

#### 10.3.3 实现代码

**Step 1：定义 ContextVar**（写在 `common/logging.py`）：

```python
import contextvars

_current_task_id: contextvars.ContextVar[str | None] = contextvars.ContextVar(
    "_current_task_id", default=None
)

def set_current_task_id(task_id: str) -> contextvars.Token[str | None]:
    """设置当前上下文的 task_id，后续日志的 trace_id 字段将使用该值。"""
    return _current_task_id.set(task_id)

def get_current_task_id() -> str | None:
    """获取当前上下文中的 task_id。"""
    return _current_task_id.get()
```

**Step 2：在日志格式化器中覆盖 trace_id**（同样在 `common/logging.py`，扩展自定义 `EKSJsonFormatter`）：

```python
class EKSJsonFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)
        # ... 基础字段 ...

        # 先注入 OTel trace context（原有逻辑）
        if hasattr(record, "otelTraceID") and record.otelTraceID != "0":
            log_record["trace_id"] = record.otelTraceID

        # 当 task_id 存在时，覆盖 trace_id
        task_id = _current_task_id.get(None)
        if task_id:
            log_record["trace_id"] = task_id
```

**Step 3：在所有入口点设置 ContextVar**

Temporal HTTP handler（`api/routers/temporal.py`）——**最关键，必须在 handler 最开头设置**：

```python
async def process_project_summary(request):
    set_current_task_id(request.task_id)    # ← handler 入口
    set_span_attribute("task_id", request.task_id)
    logger.info("Received summary request...")  # trace_id = task_id ✓
    # ...
    schedule_background_task(process_temporal_summary_task(request))
```

Temporal 后台任务（`api/services/summary_task.py`）——冗余保险：

```python
async def process_temporal_summary_task(request):
    task_id = request.task_id
    set_current_task_id(task_id)   # ← 冗余但确保后台任务内也有值
    # ...
```

Multifile 入口（`api/routers/multifile/router.py`）：

```python
def run_multifile_summary_in_background(task_id, ...):
    set_current_task_id(task_id)
    # ...
```

#### 10.3.4 为什么 ContextVar 在异步场景下有效


| 场景                    | ContextVar 行为           | task_id 是否可见 |
| --------------------- | ----------------------- | ------------ |
| `await` 调用链           | 同一协程，直接共享               | 可见           |
| `asyncio.gather`      | 在当前 Task 内执行，共享 context | 可见           |
| `asyncio.create_task` | 创建时复制当前 context 的快照     | 可见（继承副本）     |
| `threading.Thread`    | **不继承**，需要手动传递          | 不可见（需额外处理）   |


`process_temporal_summary_task` 通过 `schedule_background_task` 以 `asyncio.create_task` 执行。`set_current_task_id()` 在 HTTP handler 中调用（早于 `create_task`），因此整个调用链（包括 Agent 内部代码、LLM 回调、prompt 加载等）都能读取到 `task_id`。

### 10.4 关键注意事项


| 要点                          | 说明                                                                |
| --------------------------- | ----------------------------------------------------------------- |
| OTel 未初始化时的安全性              | `get_current()` 返回默认空 context，`attach/detach` 为 no-op，不会报错        |
| 子 Span 可以超出父 Span 生命周期      | HTTP 请求返回 202 后父 Span 结束，后台子 Span 仍共享同一 `trace_id`——这在分布式追踪中是标准行为 |
| `set_span_attribute` 的防御性检查 | 通过 `span.is_recording()` 判断，`INVALID_SPAN` 返回 `False`，安全跳过        |
| 与 `threading.Thread` 的区别    | 线程场景同理，需要手动 `get_current()` + `attach()`，参见 [9.2](#92-span-未关联)   |


> **总结**：核心原理就是 **get → attach → detach** 三步，确保异步/多线程环境中 OTel 上下文不丢失。在此基础上通过 Span Attribute 将业务 ID（如 task_id）与 trace_id 双向绑定，并通过 ContextVar 把 task_id 注入到日志的 `trace_id` 字段，实现端到端可追踪。

---

## 附录

### A. 语义约定参考


| 领域        | 属性                 | 示例                                                 |
| --------- | ------------------ | -------------------------------------------------- |
| HTTP      | `http.method`      | GET, POST                                          |
| HTTP      | `http.status_code` | 200, 404                                           |
| HTTP      | `http.url`         | [https://api.example.com](https://api.example.com) |
| DB        | `db.system`        | mysql, postgresql                                  |
| DB        | `db.statement`     | SELECT * FROM users                                |
| RPC       | `rpc.system`       | grpc, aws-api                                      |
| Messaging | `messaging.system` | kafka, rabbitmq                                    |


完整列表: [https://opentelemetry.io/docs/specs/semconv/](https://opentelemetry.io/docs/specs/semconv/)

### B. 常用 Exporter


| Exporter        | 用途      | 协议          |
| --------------- | ------- | ----------- |
| OTLP            | 通用（推荐）  | gRPC/HTTP   |
| Jaeger          | Traces  | Thrift/gRPC |
| Zipkin          | Traces  | HTTP        |
| Prometheus      | Metrics | HTTP (pull) |
| Console / Debug | 调试      | stdout      |


### C. 相关资源

- [OpenTelemetry 官网](https://opentelemetry.io/)
- [Python SDK 文档](https://opentelemetry-python.readthedocs.io/)
- [语义约定](https://opentelemetry.io/docs/specs/semconv/)
- [Collector 配置](https://opentelemetry.io/docs/collector/configuration/)
- 部署侧实践：[[09-k8s-observability]]
- 后台任务背景：[[temporal_notes]]
- Pod 优雅退出与 metric 丢失：[[12-k8s-pod-graceful-shutdown]]

