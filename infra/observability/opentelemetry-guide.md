# OpenTelemetry 完整指南

> 本文档详细介绍 OpenTelemetry 的核心概念、完整流程、使用方法，以及如何在新项目中集成。

> **全文摘要**
>
> **OpenTelemetry（OTel）** 是 CNCF 托管的开源可观测性框架，由 OpenTracing + OpenCensus 于 2019 年合并而来，提供统一的遥测数据标准。
>
> **三大支柱：**
> - **Traces（链路追踪）**：以 Trace → Span 树形结构记录请求的完整调用路径，通过 Context Propagation（W3C traceparent header）实现跨服务关联
> - **Metrics（指标）**：Counter（计数）、Histogram（分布）、Gauge（瞬时值）等聚合数值，用于监控告警与容量规划
> - **Logs（日志）**：离散事件记录，可通过 TraceId/SpanId 与链路追踪关联
>
> **后端存储（按支柱分类）：**
> | 支柱 | 典型存储方案 | 说明 |
> |------|------------|------|
> | Traces | Jaeger (Cassandra/ES/ClickHouse)、Tempo (对象存储)、Elasticsearch | 文档/对象存储，按 traceId 检索 |
> | Metrics | Prometheus (TSDB)、Thanos/Mimir (长期存储)、VictoriaMetrics | 时序数据库，按时间戳+label 聚合 |
> | Logs | ELK/EFK (Elasticsearch)、Loki (只索引 label)、ClickHouse | 全文检索/列存 |
>
> **Langfuse 存储架构（LLM 可观测性平台）：** PostgreSQL (OLTP 事务数据) + **ClickHouse (OLAP 存储 Traces/Observations/Scores)** + Redis (缓存/队列) + S3 (原始事件/多模态附件)。写入流程：SDK 批量发送 → Web Server 先写 S3 + Redis 队列 → Worker 异步消费写入 ClickHouse。
>
> **架构流程：** Application SDK → BatchProcessor → Exporter → OTel Collector（接收/处理/路由） → 后端存储（Jaeger/Prometheus/Loki） → Grafana 可视化
>
> **Python SDK 核心组件：**
> - `TracerProvider` + `BatchSpanProcessor` + `OTLPSpanExporter`（Traces）
> - `MeterProvider` + `PeriodicExportingMetricReader` + `OTLPMetricExporter`（Metrics）
> - 通过 `tracer.start_as_current_span()` 上下文管理器或装饰器创建 Span
>
> **当前项目实践：** 使用 `plaud_library_python.init_observability` 初始化；`@trace_span` 装饰器自动创建 Span 和 Metrics；`LLMMonitorCallbackHandler` 回调监控 LLM 调用的延迟、token 用量和错误
>
> **关键最佳实践：**
> - Span 命名描述操作（非实现细节），高基数值放 Attributes 而非名称
> - Metrics 标签保持低基数，避免 user_id 等导致时间序列爆炸
> - 采样率：开发 100% → 生产 10%-20%
> - 使用 Batch 处理 + 异步导出，避免影响主线程性能
>
> **与 Langfuse 的关系：** Langfuse 的数据模型（Trace → Span/Generation）本质上复用了 OTel 的 Trace 树形结构，是分布式追踪概念在 LLM 可观测性领域的特化应用。近期版本已支持原生 OTLP 协议集成。

---

## 目录

- [1. 什么是 OpenTelemetry](#1-什么是-opentelemetry)
- [2. 核心概念](#2-核心概念)
- [3. 架构与数据流](#3-架构与数据流)
- [4. 三大支柱详解](#4-三大支柱详解)
- [5. 完整工作流程](#5-完整工作流程)
- [6. Python SDK 使用指南](#6-python-sdk-使用指南)
- [7. 当前项目实现分析](#7-当前项目实现分析)
- [8. 新项目集成步骤](#8-新项目集成步骤)
- [9. 最佳实践](#9-最佳实践)
- [10. 常见问题与调试](#10-常见问题与调试)
- [附录](#附录)

---

## 1. 什么是 OpenTelemetry

### 1.1 定义

OpenTelemetry（简称 OTel）是一个开源的**可观测性框架**，由 CNCF（云原生计算基金会）托管。它提供了一套统一的 API、SDK 和工具，用于生成、收集和导出遥测数据。

### 1.2 发展历史

```
OpenTracing + OpenCensus = OpenTelemetry (2019年合并)
```

- **OpenTracing**: 分布式追踪的标准 API
- **OpenCensus**: Google 开源的指标和追踪库
- **OpenTelemetry**: 统一两者，成为可观测性的行业标准

### 1.3 为什么需要 OpenTelemetry

| 问题 | OTel 解决方案 |
|------|---------------|
| 多种监控工具不兼容 | 统一的数据格式（OTLP） |
| 厂商锁定 | 厂商中立，可切换后端 |
| 手动埋点繁琐 | 自动检测（Auto-instrumentation） |
| 数据孤岛 | Traces、Metrics、Logs 关联 |

---

## 2. 核心概念

### 2.1 可观测性三大支柱

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

### 2.2 关键术语

#### Traces（链路追踪）

| 术语 | 定义 | 示例 |
|------|------|------|
| **Trace** | 一个请求的完整调用链 | 用户请求 → API → DB → 返回 |
| **Span** | Trace 中的一个操作单元 | "查询数据库" 这个操作 |
| **SpanContext** | Span 的上下文信息 | trace_id, span_id, flags |
| **Attributes** | Span 的键值对属性 | `http.method=POST` |
| **Events** | Span 中的时间点事件 | 异常发生、重试开始 |
| **Links** | Span 之间的关联 | 批处理任务关联原始请求 |

```
Trace (trace_id: abc123)
├── Span A: HTTP Request (span_id: 001, parent: none)
│   ├── Span B: Auth Check (span_id: 002, parent: 001)
│   └── Span C: Business Logic (span_id: 003, parent: 001)
│       ├── Span D: DB Query (span_id: 004, parent: 003)
│       └── Span E: Cache Lookup (span_id: 005, parent: 003)
```

#### Metrics（指标）

| 类型                | 定义       | 使用场景         |
| ----------------- | -------- | ------------ |
| **Counter**       | 只增不减的累计值 | 请求总数、错误数     |
| **UpDownCounter** | 可增可减的值   | 当前连接数、队列深度   |
| **Histogram**     | 值的分布统计   | 请求延迟、响应大小    |
| **Gauge**         | 瞬时值（异步）  | CPU 使用率、内存占用 |

#### Logs（日志）

| 字段 | 定义 |
|------|------|
| **Timestamp** | 日志时间 |
| **Severity** | 严重级别 (DEBUG/INFO/WARN/ERROR) |
| **Body** | 日志内容 |
| **Attributes** | 结构化的上下文信息 |
| **TraceId/SpanId** | 关联的追踪信息 |

### 2.3 组件架构

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

---

## 3. 架构与数据流

### 3.1 整体架构

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

### 3.2 数据流详解

#### 直接导出模式（Simple）

```
Application ─────────────────────────────────────▶ Backend
            (OTLP/Jaeger/Zipkin/Prometheus)
```

**优点**: 简单、延迟低
**缺点**: 应用负载大、无法统一处理

#### Collector 模式（推荐）

```
Application ──▶ OTel Collector ──▶ Multiple Backends
            OTLP            (处理、路由、转换)
```

**优点**:
- 应用与后端解耦
- 统一处理（采样、过滤、增强）
- 支持多后端同时导出
- 可水平扩展

### 3.3 上下文传播（Context Propagation）

分布式追踪的核心是跨服务传递上下文：

```
Service A                          Service B
┌─────────────────────┐           ┌─────────────────────┐
│ Span: handle_request│           │ Span: process_data  │
│ trace_id: abc123    │──HTTP────▶│ trace_id: abc123    │
│ span_id: 001        │  Headers  │ span_id: 002        │
│                     │           │ parent_id: 001      │
└─────────────────────┘           └─────────────────────┘

HTTP Headers:
traceparent: 00-abc123-001-01
tracestate: vendor=value
```

**传播格式**:
- **W3C Trace Context** (标准，推荐)
- **B3** (Zipkin 格式)
- **Jaeger** (Uber 格式)

---

## 4. 三大支柱详解

### 4.1 Traces 深入

#### Trace 与 Span 的关系

**Span 是 Trace 的基本组成单元。** 一个 Span 代表一个有明确开始和结束时间的**操作**，可以是一次函数调用、一次 HTTP 请求、一次数据库查询等。Span 不等于函数——它是你选择观测的任意一段操作。

**Trace 是一组共享同一个 `trace_id` 的 Span 集合，** 通过 `parent_span_id` 组成树形结构，完整描述一个请求从入口到结束的调用路径。

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

##### 关键字段

每个 Span 携带以下核心标识：

| 字段 | 作用 | 示例 |
|------|------|------|
| `trace_id` | 全局唯一，标识整条链路，同一 Trace 下所有 Span 共享 | `abc123...` (128-bit) |
| `span_id` | 当前 Span 的唯一标识 | `def456...` (64-bit) |
| `parent_span_id` | 指向父 Span，Root Span 该字段为空 | `aaa789...` |
| `name` | 操作名称 | `HTTP GET /api/users` |
| `start_time` / `end_time` | 起止时间戳 | 用于计算耗时 |
| `attributes` | 键值对元数据 | `http.method=GET` |
| `status` | 操作结果 | `OK` / `ERROR` |

##### 跨服务如何串联（Context Propagation）

Span 之间通过 **Context Propagation** 跨进程传递 `trace_id` 和 `parent_span_id`，最常见的方式是 W3C `traceparent` HTTP header：

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

> **总结：** Trace 本身并不是一个独立的数据结构——后端存储的是一个个 Span，它们通过共同的 `trace_id` 被"串"成一条 Trace，通过 `parent_span_id` 被"组"成一棵树。

#### Span 生命周期

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

#### Span 状态

| 状态 | 含义 |
|------|------|
| `UNSET` | 默认状态，未明确设置 |
| `OK` | 操作成功完成 |
| `ERROR` | 操作出错 |

#### Span Kind

| Kind | 描述 | 示例 |
|------|------|------|
| `INTERNAL` | 内部操作（默认） | 内部函数调用 |
| `SERVER` | 服务端处理请求 | HTTP 服务器处理请求 |
| `CLIENT` | 客户端发起请求 | HTTP 客户端调用 |
| `PRODUCER` | 消息生产者 | Kafka 生产消息 |
| `CONSUMER` | 消息消费者 | Kafka 消费消息 |

### 4.2 Metrics 深入

#### 核心概念

Metrics 是**预聚合的数值数据**，与 Traces（离散事件）和 Logs（文本记录）最大的区别在于：它在采集端就做了数学运算（求和、计数、分桶），存储的是统计结果而非原始事件。这使得它在存储成本和查询速度上有天然优势。

```
原始事件流（高基数）         聚合后的 Metric（低基数）
─────────────────────       ─────────────────────────
req 1: 200ms, POST /api     http_request_duration_seconds{method="POST", endpoint="/api"}
req 2: 150ms, POST /api       count = 3
req 3: 300ms, POST /api       sum   = 0.65
                               bucket_le_0.5 = 2
                               bucket_le_1.0 = 3
```

#### Instrument 类型全景

OTel 定义了 6 种 Instrument，分为同步（调用时立即记录）和异步（回调时采集）两大类：

| 类型 | 同步/异步 | 单调性 | 典型场景 | 举例 |
|------|----------|--------|---------|------|
| **Counter** | 同步 | 只增 | 累计计数 | 请求总数、已处理消息数 |
| **UpDownCounter** | 同步 | 可增可减 | 当前计数 | 活跃连接数、队列长度变化 |
| **Histogram** | 同步 | — | 值的分布 | 请求延迟、响应体大小 |
| **ObservableCounter** | 异步 | 只增 | 系统级累计量 | CPU time、页面错误数 |
| **ObservableUpDownCounter** | 异步 | 可增可减 | 系统级瞬时量 | 内存使用量、线程数 |
| **ObservableGauge** | 异步 | 可增可减 | 瞬时采样值 | CPU 温度、当前房间人数 |

> **同步 vs 异步：**
> - **同步 Instrument**：在业务代码中主动调用 `.add()` / `.record()`，适合跟业务事件绑定
> - **异步 Instrument**：注册回调函数，SDK 按采集周期自动调用，适合采集系统状态（CPU、内存等）

#### 聚合方式（Aggregation）

SDK 在导出前会对原始测量值做聚合，不同 Instrument 默认聚合方式不同：

| Instrument | 默认聚合 | 导出的数据 |
|-----------|---------|-----------|
| Counter / ObservableCounter | **Sum** (累计) | 单个递增数值 |
| UpDownCounter / ObservableUpDownCounter | **Sum** (非单调) | 可增减的数值 |
| Histogram | **ExplicitBucketHistogram** | count、sum、min、max + 各桶计数 |
| ObservableGauge | **LastValue** | 最近一次采样值 |

#### Temporality（时间性）

Metrics 导出有两种时间语义，**选错会导致数据错误**：

```
Cumulative（累计型）：每次导出从 0 开始的总量
  T1: 100    T2: 250    T3: 400    ← 始终是总计数
  
Delta（增量型）：每次导出只报告上一周期的增量
  T1: 100    T2: 150    T3: 150    ← 每个周期新增的量
```

| 后端                            | 推荐 Temporality | 原因                             |
| ----------------------------- | -------------- | ------------------------------ |
| **Prometheus**                | Cumulative     | Prometheus 自己做 rate() 计算，需要累计值 |
| **OTLP → ClickHouse/Datadog** | Delta          | 避免重启归零问题，服务端做累加                |

> ⚠️ **常见坑：** Prometheus 只支持 Cumulative，如果配错成 Delta，图表会出现断崖式下跌。

#### Attributes（标签）与基数控制

##### 什么是时间序列

**时间序列（Time Series）** 是 TSDB 中的最小存储单元——一个指标名 + 一组固定的 label 值，随时间不断追加 `(时间戳, 数值)` 数据点：

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

##### Attributes 如何产生时间序列

Attributes 是 Metric 的**维度标签**（类似 SQL 中的 GROUP BY 字段），每组唯一的 attribute 组合会生成一条独立的时间序列：

```
# 2 个 method × 3 个 endpoint × 2 个 status = 12 条时间序列
http_requests_total{method="GET",  endpoint="/api",    status="200"}
http_requests_total{method="GET",  endpoint="/api",    status="500"}
http_requests_total{method="POST", endpoint="/api",    status="200"}
...
```

label 决定了你能按什么维度切片查询（就像 SQL 中 WHERE + GROUP BY）：

```
按 method 维度聚合（合并所有 status）：
  rate(http_requests_total{method="GET"})

按 status 维度聚合（合并所有 method）：
  rate(http_requests_total{status="500"})

全部聚合：
  rate(http_requests_total)                  ← 合并所有时间序列
```

**基数（Cardinality）** 指某个 Attribute/Label **可能取值的数量**。每一组唯一的 label 组合都会产生一条独立的时间序列，因此基数直接决定存储和查询压力。

##### 低基数 vs 高基数

| 维度 | 低基数 | 高基数 |
|------|--------|--------|
| **定义** | 取值范围小且有限 | 取值范围大或无限 |
| **典型值数量** | 几个 ~ 几百个 | 几万 ~ 无限 |
| **例子** | `method`(GET/POST/PUT/DELETE)、`status_code`(200/400/500)、`region`(us-east/eu-west) | `user_id`(100万+)、`request_id`(每请求唯一)、`email`、`session_id` |
| **是否适合做 label** | 适合 | 不适合 |

##### 为什么高基数会导致灾难

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

##### 判断基数的经验法则

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

##### 高基数需求的正确解法

| 需求 | 错误做法 | 正确做法 |
|------|---------|---------|
| 统计每个用户的请求数 | Metric label 加 `user_id` | 用 Traces 按 user_id 查询聚合 |
| 定位某个慢请求 | Histogram label 加 `request_id` | 用 Exemplar 关联到 Trace |
| 按 IP 分析流量 | Counter label 加 `client_ip` | 日志/流量分析系统（ELK、ClickHouse） |
| 统计 Top N 热门商品 | Metric label 加 `product_id` | 日志聚合 或 OLAP 数据库 |

> **一句话总结：Metrics 负责"面"（聚合趋势），Traces 负责"点"（单次请求），Logs 负责"细节"（文本上下文）。高基数分析是 Traces 和 Logs 的职责，不要让 Metrics 来扛。**
>
> **经验法则：** 单个 Metric 的时间序列数应控制在 **< 1000**。

#### Exemplars：Metrics 与 Traces 的桥梁

Exemplar 允许在 Metric 数据点上附加一个 Trace 的采样引用，实现从聚合指标钻取到具体请求：

```
Histogram bucket [0.5s, 1.0s] 命中了 12 次
  └─ Exemplar: trace_id=abc123, span_id=def456, value=0.78s
     → 点击可跳转到该慢请求的完整 Trace
```

这是 Grafana 中"从 Metrics 面板点击跳转到 Jaeger/Tempo 查看 Trace"的底层机制。

#### Counter 示例

```python
# 场景：统计 API 请求数
request_counter = meter.create_counter(
    name="http_requests_total",
    description="Total HTTP requests",
    unit="1"
)

# 使用
request_counter.add(1, {
    "method": "POST",
    "endpoint": "/api/summary",
    "status_code": "200"
})
```

#### Histogram 示例

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

# 使用
latency_histogram.record(0.235, {"endpoint": "/api/summary"})
```

#### Histogram Buckets 设计

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

### 4.3 Logs 与 Traces 关联

```python
import logging
from opentelemetry import trace

# 在日志中自动注入 trace 信息
logger = logging.getLogger(__name__)

def process_request():
    span = trace.get_current_span()
    ctx = span.get_span_context()

    # 日志会包含 trace_id 和 span_id
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
> `LoggingInstrumentor` 自动注入到日志中的**只有** `otelTraceID`、`otelSpanID`、`otelTraceSampled` 三个字段，用于实现日志与 trace 的关联。
>
> 通过 `span.set_attribute("task_id", "xxx")` 设置的**自定义 Span Attributes 不会出现在日志中**，它们只存在于 trace 后端（Jaeger/Tempo）。两者的数据流向不同：
>
> ```
> LoggingInstrumentor 注入  →  日志记录（CloudWatch/Loki）
>                               ↑ 只有 trace_id, span_id, trace_sampled
>
> span.set_attribute()      →  Span 数据（Jaeger/Tempo）
>                               ↑ 自定义属性只在这里可见
> ```
>
> 如果需要在日志中也包含业务属性，必须通过 `logger.info("msg", extra={"task_id": "xxx"})` 或 `ContextVar` 等机制显式传入，OTel 不会自动帮你同步。

---

## 5. 完整工作流程

### 5.1 请求追踪流程

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

### 5.2 指标收集流程

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

---

## 6. Python SDK 使用指南

### 6.1 安装依赖

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
```

### 6.2 基础初始化

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION


def init_otel(service_name: str, otlp_endpoint: str):
    """初始化 OpenTelemetry"""

    # 1. 定义服务资源
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: "1.0.0",
        "deployment.environment": "production",
    })

    # 2. 配置 Traces
    trace_provider = TracerProvider(resource=resource)
    trace_provider.add_span_processor(
        BatchSpanProcessor(
            OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True)
        )
    )
    trace.set_tracer_provider(trace_provider)

    # 3. 配置 Metrics
    metric_reader = PeriodicExportingMetricReader(
        OTLPMetricExporter(endpoint=otlp_endpoint, insecure=True),
        export_interval_millis=60000,
    )
    meter_provider = MeterProvider(
        resource=resource,
        metric_readers=[metric_reader]
    )
    metrics.set_meter_provider(meter_provider)
```

### 6.3 Traces 使用

#### 方式一：上下文管理器

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

#### 方式二：装饰器

```python
from opentelemetry import trace
from functools import wraps

def traced(span_name: str = None):
    """追踪装饰器"""
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

# 使用
@traced("fetch_user_data")
def get_user(user_id: str):
    return db.query(User).filter_by(id=user_id).first()
```

### 6.4 Metrics 使用

```python
from opentelemetry import metrics

meter = metrics.get_meter(__name__)

# Counter - 请求计数
request_counter = meter.create_counter(
    name="http_requests_total",
    description="Total number of HTTP requests",
    unit="1"
)

# Histogram - 延迟分布
latency_histogram = meter.create_histogram(
    name="http_request_duration_seconds",
    description="HTTP request latency in seconds",
    unit="s",
    explicit_bucket_boundaries_advisory=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

# UpDownCounter - 活跃连接数
active_connections = meter.create_up_down_counter(
    name="active_connections",
    description="Number of active connections",
    unit="1"
)

# 使用示例
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
    except Exception as e:
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

### 6.5 异步回调指标

```python
from opentelemetry import metrics
import psutil

meter = metrics.get_meter(__name__)

# 异步 Gauge - CPU 使用率
def get_cpu_usage(options):
    yield metrics.Observation(
        psutil.cpu_percent(),
        {"host": "server-1"}
    )

cpu_gauge = meter.create_observable_gauge(
    name="system_cpu_usage_percent",
    description="Current CPU usage percentage",
    unit="%",
    callbacks=[get_cpu_usage]
)

# 异步 Counter - 累计处理字节数
def get_bytes_processed(options):
    yield metrics.Observation(
        get_total_bytes_from_somewhere(),
        {"type": "inbound"}
    )

bytes_counter = meter.create_observable_counter(
    name="network_bytes_total",
    description="Total bytes processed",
    unit="bytes",
    callbacks=[get_bytes_processed]
)
```

---

## 7. 当前项目实现分析

### 7.1 初始化配置

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

**配置项** (`aws_appconfig/dev.yaml`):

```yaml
otel:
  enable_otel_exporter: true
  otlp_endpoint: "http://dev-olcc-prometheus-proxy.plaud.ai:4317"
  trace_sampling_rate: 1.0      # 100% 采样
  enable_console_export: false  # 生产关闭
```

### 7.2 Trace Span 装饰器

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

### 7.3 LLM 监控回调

**文件**: `plaud_summary/callbacks/timing_callback.py`

```python
class LLMMonitorCallbackHandler(BaseCallbackHandler):
    """LLM 调用监控"""

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

**使用**:

```python
callback = LLMMonitorCallbackHandler(
    summary_id="xxx",
    scenario="meeting"
)
result = llm.invoke(prompt, config={"callbacks": [callback]})
```

### 7.4 Summary Task Metrics

**文件**: `plaud_summary/summary_task_metrics.py`

```python
# Token 桶边界配置
TOKEN_BUCKETS = [0, 1024, 5120, 10240, 25600, 51200, 76800, 102400, 204800]

def record_summary_task_metrics(summary_id, result, llm, scenario, language, spend):
    """记录总结任务的所有监控指标"""

    attributes = {
        "final_use_model": result.get("result", {}).get("endpoint", ""),
        "scenario": scenario.lower(),
    }

    # 计数器
    summary_task_counter(1, attributes)

    # 延迟直方图
    summary_task_latency(spend, attributes)

    # Token 使用量
    summary_task_token_usage(input_tokens, "input", attributes)
    summary_task_token_usage(output_tokens, "output", attributes)
    summary_task_token_usage(total_tokens, "total", attributes)
```

---

## 8. 新项目集成步骤

### 8.1 Step 1: 添加依赖

**requirements.txt**:

```
# OpenTelemetry Core
opentelemetry-api>=1.20.0
opentelemetry-sdk>=1.20.0

# Exporters
opentelemetry-exporter-otlp>=1.20.0

# Auto-instrumentation (按需)
opentelemetry-instrumentation-fastapi>=0.41b0
opentelemetry-instrumentation-requests>=0.41b0
opentelemetry-instrumentation-sqlalchemy>=0.41b0
opentelemetry-instrumentation-redis>=0.41b0
opentelemetry-instrumentation-logging>=0.41b0
```

### 8.2 Step 2: 创建初始化模块

**observability.py**:

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
    """
    初始化 OpenTelemetry

    Args:
        service_name: 服务名称
        service_version: 服务版本
        environment: 运行环境 (dev/staging/prod)
        otlp_endpoint: OTLP Collector 端点
        trace_sampling_rate: 追踪采样率 (0.0-1.0)
        enable_console_export: 是否输出到控制台（调试用）
        metric_export_interval_ms: 指标导出间隔（毫秒）
    """

    # 1. 创建 Resource
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": environment,
    })

    # 2. 配置 Traces
    sampler = TraceIdRatioBased(trace_sampling_rate)
    trace_provider = TracerProvider(resource=resource, sampler=sampler)

    # 添加 Exporters
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
    """关闭 OpenTelemetry（优雅退出时调用）"""
    trace_provider = trace.get_tracer_provider()
    if hasattr(trace_provider, 'shutdown'):
        trace_provider.shutdown()

    meter_provider = metrics.get_meter_provider()
    if hasattr(meter_provider, 'shutdown'):
        meter_provider.shutdown()
```

### 8.3 Step 3: 创建工具函数

**otel_utils.py**:

```python
"""OpenTelemetry 工具函数"""

import time
import functools
from typing import Optional, Dict, Any
from opentelemetry import trace, metrics


def traced(
    span_name: str = None,
    attributes: Dict[str, Any] = None,
    record_exception: bool = True,
):
    """
    追踪装饰器

    Usage:
        @traced("process_order")
        def process_order(order_id: str):
            pass
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            tracer = trace.get_tracer(__name__)
            name = span_name or func.__name__

            with tracer.start_as_current_span(name) as span:
                # 设置基础属性
                span.set_attribute("code.function", func.__qualname__)
                span.set_attribute("code.filepath", func.__code__.co_filename)

                # 设置自定义属性
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

        import asyncio
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return wrapper
    return decorator


class MetricsHelper:
    """指标助手类"""

    def __init__(self, meter_name: str):
        self.meter = metrics.get_meter(meter_name)
        self._counters = {}
        self._histograms = {}

    def counter(
        self,
        name: str,
        value: int = 1,
        attributes: Dict[str, str] = None,
        description: str = "",
    ):
        """记录计数"""
        if name not in self._counters:
            self._counters[name] = self.meter.create_counter(
                name=name,
                description=description,
                unit="1",
            )
        self._counters[name].add(value, attributes or {})

    def histogram(
        self,
        name: str,
        value: float,
        attributes: Dict[str, str] = None,
        description: str = "",
        unit: str = "ms",
        buckets: list = None,
    ):
        """记录直方图"""
        if name not in self._histograms:
            kwargs = {
                "name": name,
                "description": description,
                "unit": unit,
            }
            if buckets:
                kwargs["explicit_bucket_boundaries_advisory"] = buckets
            self._histograms[name] = self.meter.create_histogram(**kwargs)
        self._histograms[name].record(value, attributes or {})

    def timed(
        self,
        name: str,
        attributes: Dict[str, str] = None,
        unit: str = "s",
    ):
        """计时上下文管理器"""
        return TimedContext(self, name, attributes, unit)


class TimedContext:
    """计时上下文"""

    def __init__(self, helper: MetricsHelper, name: str, attributes: dict, unit: str):
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
        self.helper.histogram(
            self.name,
            duration,
            self.attributes,
            unit=self.unit,
        )
        return False
```

### 8.4 Step 4: 配置文件

**config.yaml**:

```yaml
# OpenTelemetry 配置
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

### 8.5 Step 5: FastAPI 集成示例

**main.py**:

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

    # 自动检测 FastAPI
    FastAPIInstrumentor.instrument_app(app)

    yield

    # Shutdown
    shutdown_observability()


app = FastAPI(lifespan=lifespan)


@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    """请求指标中间件"""
    with metrics_helper.timed(
        "http_request_duration_seconds",
        attributes={
            "method": request.method,
            "path": request.url.path,
        },
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
    """示例 API"""
    from opentelemetry import trace

    span = trace.get_current_span()
    span.set_attribute("data.size", len(str(data)))

    # 业务逻辑...
    result = await do_processing(data)

    return {"status": "success", "result": result}
```

### 8.6 Step 6: 部署 OTel Collector

**docker-compose.yaml**:

```yaml
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

**otel-collector-config.yaml**:

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

  # 内存限制
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  # 属性处理
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

  # 调试输出
  logging:
    loglevel: debug

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
      exporters: [jaeger, logging]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

---

## 9. 最佳实践

### 9.1 Span 命名规范

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

### 9.2 Attributes 最佳实践

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

### 9.3 采样策略

| 环境 | 采样率 | 原因 |
|------|--------|------|
| 开发 | 100% | 便于调试 |
| 测试 | 100% | 完整追踪 |
| 预发布 | 50% | 平衡性能和可见性 |
| 生产 | 10-20% | 高流量下避免开销 |

```python
# 自适应采样（基于 trace_id 的尾部采样）
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ParentBased

sampler = ParentBased(
    root=TraceIdRatioBased(0.1),  # 10% 采样
)
```

### 9.4 Metrics 标签设计

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

### 9.5 性能考虑

```python
# 1. 批量处理
BatchSpanProcessor(
    exporter,
    max_queue_size=2048,        # 队列大小
    schedule_delay_millis=5000, # 导出间隔
    max_export_batch_size=512,  # 批次大小
)

# 2. 异步导出
# 默认已是异步，不会阻塞主线程

# 3. 合理设置导出间隔
PeriodicExportingMetricReader(
    exporter,
    export_interval_millis=60000,  # 1分钟，生产环境推荐
)

# 4. 内存限制
# memory_limiter:
#   limit_mib: 512
#   spike_limit_mib: 128
```

### 9.6 Histogram 分位数读数陷阱（P95 / P99）

> **一句话**：Grafana 上看到的 `histogram_quantile(...)` 永远是**估算值**，不是精确 P95/P99；它和 Metabase/SQL 里 `PERCENTILE_CONT(0.95)` 算出来的数一定有偏差。这不是 bug，是直方图的先天特性。

#### 9.6.1 为什么有偏差

Prometheus 存 Histogram **只存每个桶的累计计数**，**不保留原始样本值**。例如某 1 小时内 100 个任务的延迟分布：

| 桶 `le` | 累计计数 |
|---|---|
| ≤ 150s | 85 |
| ≤ 175s | 92 |
| ≤ 200s | 96 |
| ≤ 300s | 100 |

- **SQL 的精确算法**：有全部 100 个原始值，排序后取第 95 名，假设真实是 **179.2s**。
- **PromQL `histogram_quantile(0.95)` 的算法**：只能看到上表，知道第 95 名"在 (175, 200] 这桶里"。它**假设这 4 个任务（93–96 名）均匀分布在 175~200 之间**，线性插值：

  ```
  P95 ≈ 175 + (200 - 175) × (95 - 92) / (96 - 92)
      = 175 + 25 × 3/4 = 193.75s
  ```

  **误差 ~14s**。桶越稀疏 / 越靠尾部，误差越大。

#### 9.6.2 精度取决于桶边界密度

| 区间 | 桶边界 | 插值误差上界 |
|---|---|---|
| 30~50s | 若只有 30, 50 | ±20s |
| 30~50s | 若有 30, 35, 40, 45, 50 | ±5s |
| 200~1000s | 若只有 200, 300, 500, 750, 1000 | 百秒级 |

**在实际分布集中的区间加密桶**，尾部可以稀疏。反过来：桶很多但都落在没有样本的区域，也没用。

#### 9.6.3 OpenTelemetry Python 默认桶不适合所有场景

默认桶（不显式配置 `explicit_bucket_boundaries_advisory` 时）：

```
5, 10, 20, 30, 50, 75, 100, 125, 150, 175, 200, 300, 500, 750, 1000, 1500, 2000, 2500, 3000, 4000, 5000, 7500, 10000
```

- **适合**：毫秒~百秒量级的通用 HTTP 请求延迟。
- **不适合**：
  - 秒级以下（例如 DB 查询 P95 = 50ms，全落进 ≤ 5s 这一个桶，完全看不到分布）。
  - 长任务（例如 AI summary 任务典型 30~300s，尾部 500 之后只有 4 个桶 `500/750/1000/1500`，P99 几乎全靠猜）。

#### 9.6.4 桶设计建议

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
3. **桶数量控制在 15~30**（每多一个桶 = 每个 label 组合多一条 series，基数成本线性增长）。
4. 加桶是**破坏性变更**：Prometheus 里已有历史数据的 series 会出现"缺桶"或"多桶"，`histogram_quantile` 跨这个时间点会算错。如需切换，最好新起指标名或标注 version label。

#### 9.6.5 与 SQL / Metabase 对比的正确姿势

两个工具**不会对得上，也不需要对得上**。正确使用方式：

| 需求 | 用哪个 |
|---|---|
| 实时告警、P95 趋势、同环比、模型/节点分布 | **Grafana / Prometheus** |
| 精确业务报表、漏斗、对外汇报的"绝对数字" | **Metabase / SQL** |
| 审计、对账 | **SQL**（histogram_quantile 不能用来对账） |

对比时若数值不一致，**按下面的顺序排查**（大多数情况都是 1 或 2）：

1. **时间窗边界不同**：Metabase 的"11 点"通常是 `[11:00, 12:00)`；Grafana 的"11 点数据点"是 `increase(..[1h])` 在 11:00 为终点，覆盖 `[10:00, 11:00]`——差了整整一小时。
2. **样本集不同**：
   - metric 只在任务成功完成时 record（受代码中 `if result_should_log:` 之类分支控制）。
   - retry 在 counter/histogram 里每次都 +1 一个样本，DB 一行可能只算一次。
   - DB 通常还含失败、moderation 拦截、异常中断等 metric 不计的记录。
3. **桶插值误差**（本节主题）：几秒到几十秒的偏差属正常，桶越稀越大。
4. **多数据源分片**：Metabase 常查全局库；Grafana 的 `${datasource}` 变量只指向一个 region Prometheus。
5. **`increase()` 本身是外推**，区间端点有几个百分点的线性估算误差。

#### 9.6.6 小抄

- 看见 P95/P99 先问一句"这是 histogram 估的还是 SQL 算的"。
- Grafana 上 P95 抖几秒，别急着查业务，先看桶够不够密。
- 新建 Histogram **一律显式配 `explicit_bucket_boundaries_advisory`**，别信默认。
- 需要精确分位数 → OpenTelemetry 侧开启 **Exponential Histogram**（或直接用 SQL）。

---

## 10. 常见问题与调试

### 10.1 数据未导出

**检查清单**:

1. **Endpoint 是否正确**
   ```python
   # 检查日志
   logger.info(f"OTLP endpoint: {otlp_endpoint}")
   ```

2. **网络是否通畅**
   ```bash
   # 测试连接
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

### 10.2 Span 未关联

**原因**: 上下文未正确传播

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

### 10.3 指标基数爆炸

**症状**: Prometheus 内存持续增长
**原因**: 标签包含高基数值

```python
# 问题代码
histogram.record(latency, {"user_id": user_id})  # 每个用户一个时间序列

# 修复
histogram.record(latency, {"user_type": user.type})  # 使用低基数分类
```

### 10.4 调试命令

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

### 10.5 常用查询

**Prometheus (PromQL)**:

```
# 请求速率 (QPS)
rate(http_requests_total[5m])

# P99 延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 错误率
sum(rate(http_requests_total{status="error"}[5m]))
/ sum(rate(http_requests_total[5m]))
```

**Jaeger**:

```
# 查找慢请求
service=my-service AND duration>1s

# 查找错误
service=my-service AND error=true
```

---

## 附录

### A. 语义约定参考

| 领域 | 属性 | 示例 |
|------|------|------|
| HTTP | `http.method` | GET, POST |
| HTTP | `http.status_code` | 200, 404 |
| HTTP | `http.url` | https://api.example.com |
| DB | `db.system` | mysql, postgresql |
| DB | `db.statement` | SELECT * FROM users |
| RPC | `rpc.system` | grpc, aws-api |
| Messaging | `messaging.system` | kafka, rabbitmq |

完整列表: https://opentelemetry.io/docs/specs/semconv/

### B. 常用 Exporter

| Exporter | 用途 | 协议 |
|----------|------|------|
| OTLP | 通用（推荐） | gRPC/HTTP |
| Jaeger | Traces | Thrift/gRPC |
| Zipkin | Traces | HTTP |
| Prometheus | Metrics | HTTP (pull) |
| Console | 调试 | stdout |

### C. 相关资源

- [OpenTelemetry 官网](https://opentelemetry.io/)
- [Python SDK 文档](https://opentelemetry-python.readthedocs.io/)
- [语义约定](https://opentelemetry.io/docs/specs/semconv/)
- [Collector 配置](https://opentelemetry.io/docs/collector/configuration/)

---

### D. 实践案例：asyncio 后台任务的 OTel 上下文传播与 task_id 绑定

> **场景**：FastAPI 端点收到请求后，通过 `asyncio.create_task` 将耗时处理调度为后台任务。由于 `asyncio.create_task` **不会自动传播 OTel 上下文**，后台任务中的所有日志和 Span 都会丢失原始请求的 `trace_id`，导致无法通过 `task_id` 关联到完整的 Trace 链路。

#### D.1 问题分析

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

#### D.2 解决方案：三层绑定

##### 第一层：OTel 上下文传播（核心）

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

##### 第二层：Span Attribute 绑定 task_id

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
```

##### 第三层：日志显式记录绑定关系

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

#### D.3 最终效果

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

#### D.4 关键注意事项

| 要点 | 说明 |
|------|------|
| OTel 未初始化时的安全性 | `get_current()` 返回默认空 context，`attach/detach` 为 no-op，不会报错 |
| 子 Span 可以超出父 Span 生命周期 | HTTP 请求返回 202 后父 Span 结束，后台子 Span 仍共享同一 `trace_id`——这在分布式追踪中是标准行为 |
| `set_span_attribute` 的防御性检查 | 通过 `span.is_recording()` 判断，`INVALID_SPAN` 返回 `False`，安全跳过 |
| 与 `threading.Thread` 的区别 | 线程场景同理，需要手动 `get_current()` + `attach()`，参见本文档 10.2 节 |

> **总结**：核心原理就是 **get → attach → detach** 三步，确保异步/多线程环境中 OTel 上下文不丢失。在此基础上通过 Span Attribute 将业务 ID（如 task_id）与 trace_id 双向绑定，实现端到端可追踪。

#### D.5 进阶：使用 task_id 覆盖日志 trace_id

> **动机**：D.2 - D.4 实现了 OTel trace_id 的传播和 task_id 的 Span Attribute 绑定，但日志中的 `trace_id` 字段仍然是 OTel 自动生成的 32 位十六进制值（如 `480691584f702b103110318efdcca515`），无法直接通过业务 `task_id` 搜索日志。本节介绍如何让日志的 `trace_id` 字段直接显示为 `task_id`。

##### 问题

```
# 修改前：trace_id 是 OTel 自动生成的值，与 task_id 无直接关联
{"level": "INFO", "message": "Loaded agent prompt...", "trace_id": "480691584f702b103110318efdcca515", "span_id": "7c4ef9a26fd11193"}

# 修改后：trace_id 直接就是业务 task_id
{"level": "INFO", "message": "Loaded agent prompt...", "trace_id": "my-temporal-task-123", "span_id": "7c4ef9a26fd11193"}
```

##### 方案：contextvars.ContextVar + 日志格式化器覆盖

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
> 1. handler 中的日志（如 "Received summary request"、"Rejecting task"）在 `schedule_background_task` 之前执行，若不提前设置则这些日志仍使用 OTel 自动生成的 trace_id
> 2. `asyncio.create_task()` 在创建时复制当前 context 快照——若此时 `_current_task_id` 尚未设置，后台任务会继承 `None`
> 3. `process_temporal_summary_task` 内部的 `set_current_task_id` 调用变为冗余保险（同一个值设置两次无副作用）

##### 实现代码

**Step 1：定义 ContextVar**（`common/logging.py`）

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

**Step 2：在日志格式化器中覆盖 trace_id**（`common/logging.py` - `EKSJsonFormatter`）

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
    ...
    schedule_background_task(process_temporal_summary_task(request))
```

Temporal 后台任务（`api/services/summary_task.py`）——冗余保险：

```python
async def process_temporal_summary_task(request):
    task_id = request.task_id
    set_current_task_id(task_id)   # ← 冗余但确保后台任务内也有值
    ...
```

Multifile 入口（`api/routers/multifile/router.py`）：

```python
def run_multifile_summary_in_background(task_id, ...):
    set_current_task_id(task_id)
    ...
```

##### 为什么 ContextVar 在异步场景下有效

| 场景 | ContextVar 行为 | task_id 是否可见 |
|------|-----------------|-----------------|
| `await` 调用链 | 同一协程，直接共享 | 可见 |
| `asyncio.gather` | 在当前 Task 内执行，共享 context | 可见 |
| `asyncio.create_task` | 创建时复制当前 context 的快照 | 可见（继承副本） |
| `threading.Thread` | **不继承**，需要手动传递 | 不可见（需额外处理） |

在本项目中，`process_temporal_summary_task` 通过 `schedule_background_task` 以 `asyncio.create_task` 执行。`set_current_task_id()` 在 HTTP handler 中调用（早于 `create_task`），因此整个调用链（包括 Agent 内部代码、LLM 回调、prompt 加载等）都能读取到 `task_id`。

> **与 D.2 三层绑定的关系**：D.5 是对第三层（日志绑定）的增强。原方案通过手动 `[{task_id}]` 前缀和显式打印 `trace_id` 实现关联；新方案通过 ContextVar 自动将 `task_id` 注入到每一条日志的 `trace_id` 结构化字段中，无需修改任何业务代码中的 logger 调用。
