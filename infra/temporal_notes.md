# Temporal 学习笔记

---

## 一、定位与本质

### 一句话定位

**Temporal 是「持久化工作流编排引擎」，核心承诺：业务代码一定会执行完，即使中间发生任何故障。**

### Temporal 是什么 / 不是什么


| 维度   | Temporal           | Celery / Airflow（任务调度框架） |
| ---- | ------------------ | ------------------------ |
| 核心承诺 | 业务逻辑一定执行完          | 任务被执行一次                  |
| 状态   | 整个工作流级（事件历史持久化）    | 任务级                      |
| 失败恢复 | 从中断处继续整个流程         | 重试该任务                    |
| 长任务  | 天然支持（数月）           | 不适合                      |
| 编排   | 直接写代码（顺序、并发、循环、条件） | DAG 或链式                  |


### 包含调度，但不止于调度

Temporal 把传统任务调度框架的能力全部包含进来（定时触发、并发控制、速率限制、失败重试、DAG 编排），但目标不在「调度」本身——调度是为了实现「代码一定执行完」承诺所采用的手段。

类比：**数据库包含锁机制，但不叫「锁框架」；Temporal 包含调度机制，但不叫「调度框架」。** 锁是手段，存数据是目标；调度是手段，业务流程可靠完成才是目标。

### 实现路径

将业务流程的每一步执行结果持久化到 Server 数据库，故障恢复时通过**重放事件历史**重建运行时状态，而非依赖 Worker 内存。

---

## 二、系统架构：三个独立进程

### 2.1 三进程职责

```
┌──────────────────┐                    ┌──────────────────┐
│  Temporal Server │                    │      Worker      │
│  (调度 + 存储)    │ ◄── gRPC 长轮询──── │   (执行业务代码)   │
│                  │                    │                  │
│  :7233  gRPC     │                    │  注册:            │
│  :8233  Web UI   │                    │  - Workflow 类    │
│                  │                    │  - Activity 函数  │
└────────▲─────────┘                    └──────────────────┘
         │
         │ gRPC
         │
┌────────┴─────────┐
│     Client       │
│  (发起请求)       │
│                  │
│  启动 Workflow   │
│  发送 Signal     │
│  查询 Query      │
└──────────────────┘
```


| 进程                  | 职责                                  | 不做什么           |
| ------------------- | ----------------------------------- | -------------- |
| **Temporal Server** | 任务调度、事件持久化、Signal 路由、定时器管理、限流强制     | 不执行任何业务代码      |
| **Worker**          | 执行 Workflow / Activity 代码、轮询任务、上报结果 | 不存储任何状态（完全无状态） |
| **Client**          | 启动 Workflow、发送 Signal、查询 Query      | 不执行业务逻辑、不参与调度  |


### 2.2 关键约束：Client 永不直连 Worker

```
Client ─── ❌ 直接通信 ❌ ─── Worker
   │                             │
   │  所有交互必须经过 Server       │
   ▼                             ▲
       Temporal Server  ─────────►
```

强约束的意义：Worker 挂了、Worker 横向扩展、Worker 漂移到别的节点，Client 完全无感。Client 只需告诉 Server「我要启动这个 Workflow」，剩下交给 Server 调度。

### 2.3 生产部署示例（以 Plaud 为例）

**「名字里有 temporal 不代表它就是 Temporal Server」** —— 最常见的误解。


| 名称                             | 实际身份                                 | 镜像/代码来源                               |
| ------------------------------ | ------------------------------------ | ------------------------------------- |
| `plaud-temporal`（deploy 仓库子目录） | **Temporal Server**                  | 官方镜像 `temporalio/server`              |
| `plaud-temporal-backend`       | **Worker**（业务团队自写 Go）                | 自建业务镜像，注册自定义 Workflow                 |
| `plaud-file`                   | **Client**（启动 Workflow）              | 自建业务镜像，使用 `go.temporal.io/sdk/client` |
| `plaud-project-summary`        | **下游 HTTP 服务 + Client**（发 Signal 回调） | 自建业务镜像，使用 `temporalio.client`         |
| `plaud-api`                    | 普通业务网关（与 Temporal 无关）                | 仅转发 HTTP                              |


#### 区分依据

1. **是否用官方 Server 镜像启动**：是 → Server；否 → 业务进程
2. **业务进程导入的 SDK 类**：
  - 仅 `client.Client` / `Client.Dial(...)` → **Client 角色**
  - `worker.Worker` 并 `RegisterWorkflow(...)` → **Worker 角色**

#### 部署拓扑

```
┌─────────────────────────── K8s 集群 ───────────────────────────┐
│                                                                │
│  ┌──────────────────────┐         ┌─────────────────────────┐  │
│  │  Temporal Server      │         │  Worker 进程             │  │
│  │  (Helm Chart 部署)    │ ◄────── │  (plaud-temporal-backend)│  │
│  │                       │  gRPC   │                          │  │
│  │  镜像:                │ 长轮询  │  连 Server               │  │
│  │  - temporalio/server  │         │  注册自写 Workflow:      │  │
│  │  - temporalio/ui      │         │   - OCRWorkflow          │  │
│  │  - temporalio/admin   │         │   - OverviewWorkflow     │  │
│  │                       │         │   - ProjectNoteWorkflow  │  │
│  │  :7233 gRPC           │         │   - TranscriptCompress…  │  │
│  │  :8233 Web UI         │         └─────────────────────────┘  │
│  └──────────────────────┘                                       │
│             ▲                                                   │
│             │ gRPC                                              │
│             │                                                   │
│  ┌──────────┴─────────────┐    ┌────────────────────────────┐   │
│  │  Client 进程            │    │  下游 HTTP + Client 进程   │   │
│  │  (plaud-file)           │    │  (plaud-project-summary)   │   │
│  │                         │    │                            │   │
│  │  启动 Workflow          │    │  接收 Activity HTTP 请求   │   │
│  │  (ExecuteWorkflow)      │    │  调 LLM 生成结果           │   │
│  │                         │    │  通过 Signal 回调 Workflow │   │
│  └─────────────────────────┘    └────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

#### 类比：MySQL 部署

- **Temporal Server** ≈ MySQL 数据库本体（官方 `mysql/mysql-server`）
- **Worker** ≈ 用 `Client.Dial(...)` 连 MySQL 的业务后端
- **Client** ≈ 另一个用 `Client.Dial(...)` 连 MySQL 的服务

没人会把业务后端叫做「MySQL 本体」，同理 `plaud-temporal-backend` 也不是 Temporal Server。

---

## 三、六大核心概念

### 3.1 Workflow —— 流程编排器

**是什么**：用代码定义业务流程执行顺序的类。

**核心约束：必须确定性。** Workflow 会被反复重放（详见第四节），相同输入必须产生相同执行路径。

**最小语法骨架（Python）：**

```python
from temporalio import workflow

@workflow.defn                          # 1. 类装饰器（必须，仅 1 个）
class MyWorkflow:
    def __init__(self):
        self._state = False              # 2. 状态变量（重放时由历史恢复）

    @workflow.run                        # 3. 主入口（必须，仅 1 个）
    async def run(self, input: dict) -> dict:
        result = await workflow.execute_activity(
            do_something,
            args=[input],
            start_to_close_timeout=timedelta(seconds=30),
        )
        await workflow.wait_condition(lambda: self._state)
        return {"result": result}

    @workflow.signal                     # 4. Signal 处理器（可选，可多个）
    async def receive(self, data: dict):
        self._state = True

    @workflow.query                      # 5. Query 处理器（可选，可多个，只读）
    def get_state(self) -> bool:
        return self._state
```


| 组成部分               | 必须？ | 数量       | 说明                         |
| ------------------ | --- | -------- | -------------------------- |
| `@workflow.defn`   | 必须  | 1 个类装饰器  | 标记类为 Workflow 定义           |
| `@workflow.run`    | 必须  | 有且仅有 1 个 | Workflow 主入口               |
| `__init__`         | 可选  | 0–1 个    | 初始化状态，有 Signal/Query 时通常需要 |
| `@workflow.signal` | 可选  | 0–N 个    | 接收外部异步数据                   |
| `@workflow.query`  | 可选  | 0–N 个    | 只读查询状态，不写历史                |


**确定性约束示例：**

```python
# ❌ 禁止：非确定性 API
if random.random() > 0.5: ...     # 重放结果不同
if datetime.now().hour > 12: ...  # 重放时时间已变

# ✅ 正确：用 Temporal 的确定性 API
if await workflow.random() > 0.5: ...
now = workflow.now()
```

**Workflow 内导入第三方模块的特殊处理：**

```python
with workflow.unsafe.imports_passed_through():
    from my_module import some_function
```

原因：Workflow 代码会被重放，普通 import 的副作用会被重复执行。

---

### 3.2 Activity —— 实际干活的函数

**是什么**：执行具体 I/O 的函数（HTTP、DB、文件读写）。Workflow 禁止直接做 I/O，所有副作用必须封装为 Activity。


| 对比项    | Workflow | Activity   |
| ------ | -------- | ---------- |
| 确定性    | 必须       | 不要求        |
| I/O 操作 | 禁止       | 允许         |
| 失败恢复   | 事件重放     | 重新执行       |
| 运行时间   | 可数月      | 应尽量短，必须设超时 |


**最小语法骨架：**

```python
from temporalio import activity

@activity.defn
async def call_external_service(workflow_id: str, data: str) -> str:
    """实际执行 HTTP 调用，返回 task_id。"""
    async with aiohttp.ClientSession() as session:
        async with session.post(url, json={"workflow_id": workflow_id, "data": data}) as resp:
            return await resp.text()
```

**在 Workflow 中调用 Activity 的完整签名：**

```python
result = await workflow.execute_activity(
    call_external_service,                      # Activity 函数引用
    args=[workflow_id, data],                   # 参数列表
    start_to_close_timeout=timedelta(seconds=30),   # 必须：单次执行超时
    schedule_to_close_timeout=timedelta(minutes=30), # 可选：含排队的总超时
    retry_policy=RetryPolicy(                   # 可选：重试策略
        maximum_attempts=3,
        initial_interval=timedelta(seconds=1),
        maximum_interval=timedelta(seconds=10),
        backoff_coefficient=2.0,
    ),
)
```

---

### 3.3 Worker —— 执行引擎

**是什么**：长时间运行的进程，从 Task Queue 拉取任务并执行注册过的 Workflow / Activity 代码。

**核心特性：完全无状态。** Worker 随时可崩溃重启，任意 Worker 都能接手任意 Workflow，因为状态全部在 Server。

**最小语法骨架：**

```python
from temporalio.client import Client
from temporalio.worker import Worker

async def main():
    client = await Client.connect("localhost:7233")

    worker = Worker(
        client,
        task_queue="my-task-queue",     # 监听哪个队列
        workflows=[MyWorkflow],         # 注册能处理的 Workflow 类
        activities=[call_external_service],  # 注册能执行的 Activity 函数
    )

    await worker.run()                  # 无限循环：长轮询 → 执行 → 上报
```

一个 Worker 可以注册多种 Workflow 和 Activity，注册的是「能力」。实际执行什么取决于队列里有什么。

**Go SDK 等价：**

```go
w := worker.New(client, "my-task-queue", worker.Options{
    MaxConcurrentActivityExecutionSize:     20,
    MaxConcurrentWorkflowTaskExecutionSize: 10,
})
w.RegisterWorkflow(MyWorkflow)
w.RegisterActivity(CallExternalService)
w.Start()
```

---

### 3.4 Task Queue —— 任务通道

**是什么**：连接 Server 和 Worker 的逻辑通道。Worker 通过 Task Queue 名称决定监听哪些任务。

**关键规则：**

- 一个 Worker 只能监听一个 Task Queue
- 多个 Worker 可监听同一个 Task Queue（水平扩展）
- 启动 Workflow 时指定的 Task Queue **必须与 Worker 监听的完全一致**，否则任务永远堆积无人处理

**为什么不用 Kafka？** Temporal 用「数据库 + 长轮询」实现 Task Queue，因为：

1. **同步匹配**：有 Worker 等着就直接给它，不用写入再读出
2. **精确一次**：`SELECT ... FOR UPDATE` 保证任务不会被两个 Worker 同时拿到
3. **事务原子**：写历史事件 + 创建下一个 Task 在同一个数据库事务里

---

### 3.5 Signal —— 向 Workflow 发送数据

**是什么**：外部向正在运行的 Workflow 发送数据的机制。单向、异步、持久化。

**使用场景**：外部服务处理完成后通知 Workflow、人工审批输入、流程干预。

**发送 Signal 所需信息：**


| 信息          | 是否必需    | 来源                                  |
| ----------- | ------- | ----------------------------------- |
| Workflow ID | 必需      | 启动 Workflow 时指定的 `id` 参数            |
| Signal 名称   | 必需      | Workflow 类中 `@workflow.signal` 方法名  |
| Signal 数据   | 看定义     | Signal 处理器的参数                       |
| Task Queue  | **不需要** | Signal 不经过 Task Queue，由 Server 直接路由 |


**Signal 流转路径：**

```
发送方 ──(workflow_id)──► Temporal Server ──(路由)──► 目标 Workflow
                          不经过 Task Queue
```

**三种发送场景：**

```python
# 场景 1：刚启动 Workflow，用 handle 直接发
handle = await client.start_workflow(MyWorkflow.run, id=wid, task_queue=tq)
await handle.signal(MyWorkflow.receive, "Hello")

# 场景 2：已运行的 Workflow，用 workflow_id 拿 handle 后发
handle = client.get_workflow_handle(workflow_id)
await handle.signal("receive", data)              # Signal 名用字符串

# 场景 3：外部服务发 Signal（先连接 Server）
client = await Client.connect("localhost:7233")
handle = client.get_workflow_handle(workflow_id)  # workflow_id 从 HTTP 请求体获取
await handle.signal("receive", data)
```

**在 Workflow 中接收 Signal：**

```python
@workflow.signal
async def receive(self, data: dict):
    self._state = data
    self._received = True
```

**等待 Signal 触发的条件：**

```python
# 无超时等待
await workflow.wait_condition(lambda: self._received)

# 带超时
await asyncio.wait_for(
    workflow.wait_condition(lambda: self._received),
    timeout=300,
)
```

`wait_condition` 不会阻塞 Signal 接收——等待期间 Signal 正常入队执行。

---

### 3.6 Query —— 只读查询 Workflow 状态

**是什么**：同步查询 Workflow 当前状态。只读、不影响执行、不写入历史。

**定义和调用：**

```python
# 定义（在 Workflow 类内）
@workflow.query
def get_status(self) -> str:
    return "completed" if self._done else "running"

# 调用（外部）
handle = client.get_workflow_handle(workflow_id)
status = await handle.query("get_status")
# 或类型安全方式
status = await handle.query(MyWorkflow.get_status)
```

Query 与 Signal 的关键区别：


| 维度  | Signal              | Query    |
| --- | ------------------- | -------- |
| 方向  | 写入（修改状态）            | 读取       |
| 持久化 | 记录到事件历史             | 不记录      |
| 同步性 | 异步（fire-and-forget） | 同步（等返回值） |


---

## 四、核心机制：重建不是恢复

这是理解 Temporal 最关键的一点。

### 常见误解 vs 实际情况


|          | 描述                                                 |
| -------- | -------------------------------------------------- |
| ❌ 误解（恢复） | Workflow 启动后驻留内存，遇到 `await` 暂停，Activity 完成后从原位置继续  |
| ✅ 实际（重建） | 每次有新事件时，Worker 创建新的 Workflow 实例，从头执行代码，通过历史快进到当前位置 |


### 生命周期示意

```
Workflow Task #1               Workflow Task #2
   │                                │
   ▼                                ▼
┌──────────┐                  ┌──────────┐
│ 新建实例 │                  │ 新建实例 │  ← 可能不同 Worker
│ 执行代码 │                  │ 从头执行 │
│ 到 await │                  │ 重放历史 │
│ 产生命令 │                  │ 继续到下 │
│ 实例销毁 │ ← 内存释放        │ 一个 await│
└──────────┘                  │ 实例销毁 │ ← 又销毁
                              └──────────┘
```

### 类比


|      | 类比                                |
| ---- | --------------------------------- |
| ❌ 恢复 | 游戏暂停后继续（内存状态还在）                   |
| ✅ 重建 | 看录像快进（关游戏 → 重开 → 靠存档快进到原进度 → 继续玩） |


### 为什么这样设计

1. **无状态 Worker**：Worker 随时崩溃重启，任意 Worker 接手都正确
2. **状态在 Server**：事件历史就是「真相」，不依赖 Worker 内存
3. **超长运行**：Workflow 可跑数月而不占任何 Worker 内存槽位

---

## 五、两种 Task 与执行时间线

Worker 内部有两套独立处理器，处理两种 Task：


| 维度         | Workflow Task（「通知」）    | Activity Task（「任务」） |
| ---------- | ---------------------- | ------------------- |
| Server 发什么 | 事件历史                   | 函数名 + 参数            |
| Worker 做什么 | 重放代码、决策下一步             | 调用 Activity 函数      |
| 产出         | 命令（例如「请调度 X Activity」） | 执行结果                |
| 耗时         | 毫秒级                    | 可能很长（秒到分钟）          |


**触发 Workflow Task 的事件：**


| 触发事件             | Server 的动作       |
| ---------------- | ---------------- |
| Workflow 刚启动     | 通知 Worker 决定第一步  |
| Activity 完成 / 失败 | 通知 Worker 决定下一步  |
| 收到 Signal        | 通知 Worker 处理这个信号 |
| Timer 到期         | 通知 Worker 继续执行   |


### 完整执行时间线示例

```
Workflow Task #1   Activity Task #1   Workflow Task #2   等待 Signal...
┌──────────┐       ┌──────────┐       ┌──────────┐
│ 执行到   │       │ HTTP POST│       │ 重放 #1  │   ⏳ Workflow 暂停
│ await    │ ────► │ 外部服务 │ ────► │ 到 wait  │      状态在 Server
│ activity │       │          │       │ condition│
└──────────┘       └──────────┘       └──────────┘
                                                       … 外部处理中 …

Signal 到达 ──► Workflow Task #3   Activity Task #2   Workflow Task #4
                ┌──────────┐       ┌──────────┐       ┌──────────┐
                │ 重放#1#2 │       │ process  │       │ 重放全部 │
                │ 到 await │ ────► │ result   │ ────► │ return   │
                │ activity │       │          │       │ 完成！   │
                └──────────┘       └──────────┘       └──────────┘
```

### 每次 Workflow Task 拿到的历史

```
Task #1: [ WorkflowExecutionStarted ]
  → 从头执行，遇到第一个 await，生成 ScheduleActivityTask 命令

Task #2: [ Started, ActivityScheduled, ActivityCompleted{result} ]
  → 重放：第一个 await 有历史结果，跳过
  → 继续到 wait_condition，暂停

Task #3: [ …前面事件…, SignalReceived{data} ]
  → 重放：前面 await 跳过，Signal 修改状态
  → wait_condition 满足，继续到第二个 await

Task #4: [ …所有事件…, ActivityCompleted{final_result} ]
  → 重放：所有 await 跳过
  → 执行到 return，Workflow 完成
```

---

## 六、Client API：触发 Workflow

### 6.1 两种启动方式


| 方式                 | 行为              | 返回值            | 适用场景                 |
| ------------------ | --------------- | -------------- | -------------------- |
| `execute_workflow` | 启动 **并阻塞等待** 完成 | Workflow 返回值   | 同步调用                 |
| `start_workflow`   | 仅启动，**立即返回**    | WorkflowHandle | 需要后续交互（Signal/Query） |


`execute_workflow` 本质等同于 `start_workflow()` + `handle.result()`。

### 6.2 两种调用风格

```python
# ✅ 风格 1：类型安全（同进程内，能 import Workflow 类）
handle = await client.start_workflow(
    MyWorkflow.run,                    # 直接引用方法
    input_data,
    id=workflow_id,
    task_queue="my-tasks",
)

# ✅ 风格 2：字符串（跨服务调用，无法 import）
handle = await client.start_workflow(
    "MyWorkflow",                      # Workflow Type 名称
    payload,                           # dict 形式
    id=workflow_id,
    task_queue="my-tasks",
)
```


| 场景      | 推荐风格 | 原因               |
| ------- | ---- | ---------------- |
| 同代码库    | 类型安全 | IDE 补全、参数校验、重构安全 |
| 跨服务/跨语言 | 字符串  | 无法 import，靠类型名约定 |


字符串方式传的 Workflow Type 名称必须和对方 Worker 注册的名称**完全一致**。

### 6.3 完整参数参考

```python
handle = await client.start_workflow(
    # ═══ 必须参数 ═══
    workflow,                          # Workflow 引用或字符串类型名
    arg,                               # 传给 @workflow.run 的参数
    # 或 args=[a, b],                  # 多参数用 args 列表
    id="workflow-id",                  # 全局唯一，建议映射业务实体
    task_queue="task-queue",           # 必须与 Worker 注册一致

    # ═══ 超时参数 ═══
    execution_timeout=timedelta(...),  # 整体超时（含重试 + ContinueAsNew）
    run_timeout=timedelta(...),        # 单次 Run 超时
    task_timeout=timedelta(...),       # 单次 Workflow Task 超时

    # ═══ 重试 & ID 策略 ═══
    retry_policy=RetryPolicy(...),     # Workflow 级重试
    id_reuse_policy=...,               # 同 ID 冲突处理:
        # ALLOW_DUPLICATE / ALLOW_DUPLICATE_FAILED_ONLY
        # REJECT_DUPLICATE / TERMINATE_IF_RUNNING

    # ═══ 可观测性 ═══
    memo={"key": "value"},             # Web UI 可见元数据
    search_attributes=...,             # Visibility API 过滤字段

    # ═══ 高级 ═══
    cron_schedule="0 * * * *",         # 定时调度
    start_delay=timedelta(minutes=5),  # 延迟启动
    request_eager_start=True,          # 跳过一次轮询延迟
)
```

**核心四参数：**


| 参数             | 必须？ | 说明                  |
| -------------- | --- | ------------------- |
| `workflow`     | 必须  | Workflow 类引用或字符串类型名 |
| `arg` / `args` | 看定义 | 传给 `@workflow.run`  |
| `id`           | 必须  | 全局唯一                |
| `task_queue`   | 必须  | 必须与 Worker 注册一致     |


### 6.4 Workflow Type vs Workflow ID

```
Workflow Type = 类 / 模板         如 ProjectNoteWorkflow
Workflow ID   = 对象 / 实例       如 "task-abc123-project-xxx"

启动时需要：  Type + ID + Task Queue
发送 Signal：  只需要 ID（Server 已记录 Type）
```

### 6.5 实战：跨服务启动 ProjectNoteWorkflow（Plaud 真实代码）

ProjectNoteWorkflow 定义在 `plaud-temporal-backend`（Go），由 `plaud-file`（Go）的业务层启动。

**Go SDK 等价 API：**

```go
import "go.temporal.io/sdk/client"

// 启动方代码（plaud-file 中的封装）
func StartProjectNoteWorkflow(ctx context.Context, input *ProjectNoteWorkflowInput) error {
    workflowOptions := client.StartWorkflowOptions{
        ID:        input.TaskID,                  // 业务 task_id 作为 workflow_id
        TaskQueue: "plaud-project-note-tasks",
    }
    _, err := temporalClient.ExecuteWorkflow(
        ctx,
        workflowOptions,
        "ProjectNoteWorkflow",                    // Workflow Type 字符串
        input,                                    // 跨服务 payload
    )
    return err
}
```

**payload 结构（双方约定 schema）：**

```go
type ProjectNoteWorkflowInput struct {
    TaskID        string                  // 复用作 workflow_id
    ProjectID     string
    UserID        string
    WorkspaceID   string
    MemberID      string
    Overview      string
    SourceList    []ProjectNoteSourceItem
    Strategy      string
    Language      string
    DownstreamURL string  // 下游 AI 服务地址（Workflow 内的 Activity 会 POST 到这里）
    CallbackURL   string  // Workflow 完成后回调启动方的地址
}
```

**关键点：**

- **跨语言透明**：启动方 Go，Worker Go，下游 AI 服务 Python——三方仅靠「Workflow Type + Task Queue + payload schema」约定
- **payload 用 dict / struct**：跨服务无法共享类型定义，靠协议文档
- **业务 ID 复用为 workflow_id**：方便 Signal 回调、日志关联、Web UI 检索

---

## 七、`start_workflow` 底层流程

### 7.1 调的是 Server，不是 Worker

```
Client (plaud-file)                        Worker (plaud-temporal-backend)
    │                                              │
    │ ExecuteWorkflow(...)                         │
    │ ＝ 发一次 gRPC                                │
    │   "StartWorkflowExecution"                   │
    │                                              │
    │  ❌ 没有直接连接 ❌                            │
    │                                              │
    ▼                                              ▲
┌──────────────────────────────────────────────────┴───┐
│              Temporal Server                          │
│  - 接收 StartWorkflowExecution gRPC                  │
│  - 数据库事务里：写 workflow_execution + 入队 Task   │
│  - 返回 workflow_id / run_id                          │
└──────────────────────────────────────────────────────┘
```

### 7.2 Server 入队机制

`start_workflow` 这一行 SDK 代码做的事**就是发一次 gRPC**，不操作任何队列。Server 收到后在一个数据库事务里完成三件事：

1. 写 `workflow_execution` 记录（状态、Type、Input 等）
2. 写第一条事件 `WorkflowExecutionStarted`
3. 在对应 Task Queue 表里创建一个 Workflow Task

类比数据库：


| Temporal                  | 数据库类比                     |
| ------------------------- | ------------------------- |
| Client 调 `start_workflow` | 应用执行 `INSERT INTO orders` |
| Server 写记录 + 创建 Task      | DB 引擎写 B+ 树 + 刷 WAL       |
| Worker 长轮询拉任务             | 另一个 reader 查待处理订单         |


**应用代码不直接动磁盘**——同理，Client 不直接动队列。

### 7.3 Worker 长轮询机制

**长轮询（Long Polling）= 请求发出后服务端不立刻返回，挂住等到有任务或超时再回。**

```
时间轴：
        Worker                                Server
t=0     │ PollActivityTaskQueue (gRPC)        │
        │ ─────────────────────────────────► │  请求挂起
        │                                    │  ⏳ 等任务…
t=15s   │                                    │  ✨ 收到新任务！
        │ ◄─────── 立刻返回 Task ──────────  │
        │                                    │
        │ 执行 Activity (10 min)              │
        │ ──── 上报结果 ─────────────────►   │
        │                                    │
        │ 再次发起 Poll                       │
        │ ─────────────────────────────────► │  又挂起
        │                                    │  ⏳ 没任务…
t=10m+  │ ◄────── 60s 超时空返回 ──────────  │  避免连接长期阻塞
        │                                    │
        │ 立刻再发一次 Poll                   │
```

**长轮询 vs 普通轮询：**


| 维度    | 长轮询          | 普通轮询         |
| ----- | ------------ | ------------ |
| 空闲时   | 一个连接挂着等      | 不断 ping，浪费请求 |
| 新任务到达 | 立即返回（毫秒延迟）   | 最差等到下次 ping  |
| 资源占用  | 极低（gRPC 长连接） | 高（频繁建/断）     |


### 7.4 PollActivityTaskQueue 是 SDK 内置，不需要自己实现

业务代码只写两件事：

1. `worker.New(...)` 创建 Worker
2. `RegisterWorkflow(...)` / `RegisterActivity(...)` 告诉 SDK 自己能处理哪些函数

**剩下的全部由 SDK 干**：起 Poller goroutine、发 `PollWorkflowTaskQueue` / `PollActivityTaskQueue` gRPC、收到任务后反射调用注册函数、上报结果。


| 概念                  | 业务代码写                           | SDK 帮忙做                                    |
| ------------------- | ------------------------------- | ------------------------------------------ |
| HTTP 服务             | 路由 + handler                    | TCP accept、HTTP 解析、路由匹配                    |
| ORM                 | `User.find(id)`                 | 连接池、SQL 拼接、网络包、反序列化                        |
| **Temporal Worker** | **Workflow / Activity 函数 + 注册** | `**PollActivityTaskQueue` gRPC、任务派发、结果上报** |


---

## 八、任务调度与限流

### 8.1 调度本质

**Task Queue + 长轮询匹配**，不是预排时间表。Server 把待办 Task 放进数据库队列，Worker 主动拉取。Server 从不主动 push。

「调度策略」=

- **派发策略**：Server 决定何时把 Task 从队列里放给 Poller
- **并发控制**：Worker 端决定同时跑多少个
- **速率控制**：Server / Worker 联合决定每秒启多少个

### 8.2 worker.Options 旋钮全集


| 旋钮                                       | 作用                                     | 控制粒度            |
| ---------------------------------------- | -------------------------------------- | --------------- |
| `MaxConcurrentActivityExecutionSize`     | 单 Worker 同时执行多少 Activity               | Worker 本地槽位     |
| `MaxConcurrentWorkflowTaskExecutionSize` | 单 Worker 同时处理多少 Workflow 决策            | Worker 本地槽位     |
| `MaxConcurrentActivityTaskPollers`       | 单 Worker 开几个 Activity Poller goroutine | Worker 本地       |
| `MaxConcurrentWorkflowTaskPollers`       | 单 Worker 开几个 Workflow Poller goroutine | Worker 本地       |
| `WorkerActivitiesPerSecond`              | 本 Worker 每秒最多启多少 Activity              | Worker 本地限速     |
| `TaskQueueActivitiesPerSecond`           | 整条 Task Queue 每秒最多启多少 Activity         | **Server 全局限速** |


`TaskQueueActivitiesPerSecond` 是最强的旋钮——多 Worker 部署时总量也不会超过此值。

### 8.3 五层限流闸门

```
   N 个 Workflow 启动
        │
        ▼
┌───────────────────────────────┐
│ ① Server 入队（不可控）        │  ← worker.Options 管不到
│   start_workflow 永远立刻入队  │     这是 Client 的事
└───────────────┬───────────────┘
                ▼
┌───────────────────────────────┐
│ ② Worker 拉取 Activity Task    │  ← ✅ worker.Options 控
│   - MaxConcurrentActivity      │     「水龙头流量上限」
│     ExecutionSize              │
└───────────────┬───────────────┘
                ▼
┌───────────────────────────────┐
│ ③ Activity 启动速率            │  ← ✅ worker.Options 控
│   - WorkerActivitiesPerSecond  │     「水龙头速率上限」
│   - TaskQueueActivitiesPer     │
│     Second（全局）              │
└───────────────┬───────────────┘
                ▼
┌───────────────────────────────┐
│ ④ Activity 内 HTTP 调用        │  ← ❌ worker.Options 管不到
│   超时由 ActivityOptions 控    │     行为级配置，非 Worker 级
└───────────────┬───────────────┘
                ▼
┌───────────────────────────────┐
│ ⑤ 下游服务内部资源调用         │  ← ❌ 跨进程，需要业务代码兜底
│   （如 LLM API、DB 写入）       │     Semaphore / TokenBucket
└───────────────────────────────┘
```

### 8.4 实战剧本：LLM RPM/TPM 限流场景

**需求**：前端瞬间发起 100 个项目摘要请求，下游 LLM 有 RPM/TPM 上限，不能并发轰炸。

#### 第一阶段：100 个请求被瞬间「吞下」

100 个 HTTP 请求进 `plaud-api` → 转发到 `plaud-file`。`plaud-file` 做完幂等校验和落库后，**100 次调用 `ExecuteWorkflow(...)` 几乎瞬间返回**——只是发一次 gRPC，Server 收到后写记录 + 入队。

100 条 HTTP 请求**几秒内全部 200 OK 返回前端**，用户感受不到任何排队压力。

```
此时 Server 数据库：
- workflow_execution 表：100 条 RUNNING 记录
- 队列 plaud-project-note-tasks：100 个 Workflow Task 待派发
```

#### 第二阶段：Worker 拉走 Workflow Task，产生 100 个 Activity Task

Worker 早已挂着长轮询。100 个 Workflow Task 一进队列，Worker 的 Workflow Poller 立刻拉取并重放代码：

```
进入 ProjectNoteWorkflow
  → 校验输入
  → await execute_activity(调下游 AI 服务)
                          ↑
                  产生「请调度此 Activity」命令
  → 暂停，等结果
```

100 个 Workflow Task 全跑完上述决策（毫秒级），队列里**多出 100 个 Activity Task**。Workflow 此时全部进入「等 Activity」状态，**不占任何 Worker 内存**——状态在 Server。

#### 第三阶段：限流闸门按节奏放行 Activity

Worker 配置：

- `MaxConcurrentActivityExecutionSize = 20`：本 Worker 最多同时跑 20 个 Activity
- `TaskQueueActivitiesPerSecond = 0.5`：全队列每秒最多启 0.5 个 Activity

效果：

```
t=0s    ──► 第 1 个 Activity 出发（槽位 1/20）
t=2s    ──► 第 2 个 Activity 出发
…
t=38s   ──► 第 20 个 Activity 出发（槽位 20/20 占满）
t=40s   ──► 第 21 个待发，但槽位满 → 继续在 Server 队列里等
```

剩下 80 个 Activity Task **躺在 Server 数据库里等**，Web UI 上可见「Pending」。**0 CPU / 0 内存占用**——只是几条数据库记录。

#### 第四阶段：LLM 慢慢消化，槽位轮转

20 个 Activity 同时 HTTP POST 到下游 AI 服务。下游内部用 `asyncio.Semaphore(3)` 做兜底，只让 3 个 LLM 调用并发。

5 分钟后第一个完成 → 下游通过 Signal 回传 → Worker 收到 Signal Task → 重放 Workflow → 走完剩余代码 → Activity 槽位释放。

槽位一释放，Server **立刻派下一个等待中的 Activity Task** 过来。

```
t=0s            100 个任务全部「已接受」
t=0–38s         头 20 个 Activity 陆续出发
t=5min          第一批完成，新的 Activity 顶上
t=25min         100 个全部完成
```

整个 25 分钟里：

- **前端从未感到压力**——所有请求秒回
- **LLM 从未被压爆**——任意时刻 ≤20 HTTP / ≤3 LLM
- **没有任务丢失**——状态在 Server 数据库
- **没有人维护待处理列表**——Server 自己就是

#### 异常处理（全自动）


| 情况                      | Temporal 自愈方式                                        |
| ----------------------- | ---------------------------------------------------- |
| 某个 Activity HTTP 返回 500 | 根据 `RetryPolicy` 自动重试 3 次                            |
| Worker 进程崩溃             | Server 端 Lease 超时（默认 10s）→ Task 自动回队列 → 别的 Worker 接手 |
| 下游服务整体宕机                | 失败的 Activity 走重试 → 队列里其他 Task 继续等                    |
| 突然再来 200 个请求            | Server 数据库再写 200 条 → 照样按 0.5/s 派发 → 前端依然秒回           |


#### 关键洞察

Temporal 在该场景的核心价值不是「调度」本身，而是**把「待办任务」彻底持久化为数据库记录**。

传统做法需要手写：

- 一张 `pending_tasks` 表
- 一个定时器扫表派发
- 一个并发计数器
- 一个重试队列
- 一套 Worker 注册和心跳

Temporal 把这些做成基础设施：


| 业务需要的能力 | Temporal 实现方式              |
| ------- | -------------------------- |
| 任务永不丢失  | 数据库事务 + 事件历史               |
| 请求秒回    | `start_workflow` 只发一次 gRPC |
| 削峰填谷    | Task Queue + 并发槽位 + 速率限流   |
| 失败自动重试  | `RetryPolicy`              |
| 单点故障自愈  | Worker 无状态 + 重建模式          |
| 任务进度可观察 | Web UI 显示每个 Workflow 状态    |
| 跨服务回调可靠 | Signal 持久化在事件历史            |


业务代码因此可以像本地函数一样写分布式流程：

```go
result := callDownstreamAI(...)
signal := waitForSignal(...)
callbackUpstream(...)
```

---

## 九、错误处理

### 9.1 Activity 重试

Activity 失败时 Temporal 自动重试，通过 `RetryPolicy` 控制：

```python
retry_policy=RetryPolicy(
    maximum_attempts=3,                      # 最多 3 次（1 原始 + 2 重试）
    initial_interval=timedelta(seconds=1),   # 首次重试等 1 秒
    maximum_interval=timedelta(seconds=10),  # 间隔上限
    backoff_coefficient=2.0,                 # 退避系数（默认 2.0）
    non_retryable_error_types=["ValueError"],# 不重试的错误类型
)
```

### 9.2 不可重试的业务错误

```python
from temporalio.exceptions import ApplicationError

raise ApplicationError(
    "External service callback timeout",
    non_retryable=True,                      # 立即失败，不触发重试
)
```

**区分两种错误：**


| 类型   | 例子                  | 处理方式                                   |
| ---- | ------------------- | -------------------------------------- |
| 可重试  | 网络抖动、临时 503、超时      | 让 Temporal 自动重试                        |
| 不可重试 | 业务校验失败、数据错误、永久性配置错误 | `ApplicationError(non_retryable=True)` |


### 9.3 Workflow 级失败

Workflow 整体也可以失败（return 异常）。Workflow 级 `RetryPolicy` 控制整个 Workflow 是否重试。绝大多数场景只需要 Activity 级重试。

---

## 十、Namespace 隔离

**是什么**：类似 Kubernetes Namespace，在同一个 Temporal Server 上提供逻辑隔离。

```
┌──────────────────────────────────────────────────┐
│              Temporal Server                       │
├────────────────┬────────────────┬────────────────┤
│  "default"     │  "production"  │  "staging"     │
│  (开发测试)    │  (生产环境)    │  (预发布)      │
└────────────────┴────────────────┴────────────────┘
```


| 特性    | 说明                                              |
| ----- | ----------------------------------------------- |
| 隔离性   | 不同 Namespace 的 Workflow 完全隔离，互不可见               |
| ID 复用 | 同名 Workflow ID 在不同 Namespace 中可共存               |
| 默认值   | 未指定时使用 `"default"`                              |
| 内部结构  | 一个 Namespace 内可有多个 Task Queue 和多种 Workflow Type |


Plaud 生产使用 `backend-ai` 等业务 namespace 隔离不同业务线的工作流。

---

## 十一、Plaud 生产全景

### 11.1 服务清单与角色


| 服务                       | 语言     | Temporal 角色                       | 关键代码位置                                                                                      |
| ------------------------ | ------ | --------------------------------- | ------------------------------------------------------------------------------------------- |
| `plaud-temporal`（部署仓库）   | –      | **Server**（官方镜像）                  | `deploy/plaud-temporal/temporal/values.yaml`                                                |
| `plaud-temporal-backend` | Go     | **Worker**                        | `main.go`、`internal/workflow/*.go`、`internal/activity/*.go`、`internal/registry/registry.go` |
| `plaud-api`              | Python | 业务网关（与 Temporal 无关）               | `api/project/note/create_task.py`                                                           |
| `plaud-file`             | Go     | **Client**（启动 Workflow）           | `pkg/temporal/client.go`、`services/project/note_task.go`                                    |
| `plaud-project-summary`  | Python | **下游 HTTP + Client**（发 Signal 回调） | `api/routers/temporal.py`、`api/services/result_handlers.py`、`common/temporal.py`            |


### 11.2 完整调用链（以 ProjectNote 为例）

```
前端
  │ POST /v1/project/create-task/{project_id}
  ▼
plaud-api
  │ 鉴权、灰度判断
  │ forward_to_plaud_file()
  ▼
plaud-file (Go)
  │ 业务校验、落库
  │ ExecuteWorkflow("ProjectNoteWorkflow", payload, queue="plaud-project-note-tasks")
  │ ↓ gRPC ↓
  ▼
Temporal Server
  │ 写 workflow_execution + 创建 Workflow Task
  │ HTTP 200 返回前端（链路前段秒回）
  │ ↑ 长轮询 ↑
  ▼
plaud-temporal-backend (Worker)
  │ 拉取 Workflow Task → 重放 ProjectNoteWorkflow 代码
  │ → 调度 CallProjectNoteAIActivity
  │ → 拉取 Activity Task → 发 HTTP POST
  │ ↓ HTTP ↓
  ▼
plaud-project-summary (Python)
  │ 接收 /api/temporal/project-summary/process
  │ 返回 202 accepted
  │ 后台跑 AI 摘要（5-10 min，含 LLM 限流）
  │ 通过 Signal "project-note-result" 回传结果
  │ ↑ gRPC Signal ↑
  ▼
plaud-temporal-backend (Worker)
  │ 收到 Signal → 重放 Workflow → 继续到下一步
  │ → 调度 CallbackUpstream Activity
  │ ↓ HTTP POST ↓
  ▼
plaud-file
  │ /v1/project/note-callback
  │ 更新 task 状态、通知用户
```

### 11.3 三个关键设计


| 设计                                     | 实现                                                      |
| -------------------------------------- | ------------------------------------------------------- |
| **Workflow Type + Task Queue 作为跨服务契约** | 启动方 Go、Worker Go、下游 Python——三方语言不同，仅靠字符串名约定             |
| **业务 ID 复用为 workflow_id**              | `task_id` 同时作 Workflow ID，便于日志、Signal 路由、Web UI 检索      |
| **Signal 而非 HTTP 回调**                  | 下游处理完后用 Signal 而非 HTTP 把结果送回 Workflow，靠 Server 保证持久化和送达 |


### 11.4 Worker 注册（plaud-temporal-backend）

`internal/registry/registry.go` 声明式注册多个 Worker：

```go
var Workers = []WorkerDef{
    {Name: "ocr",                  TaskQueue: "plaud-ocr-tasks",
     MaxConcurrentActivities: 20,  Register: ...OCRWorkflow},
    {Name: "overview",             TaskQueue: "plaud-overview-tasks",
     MaxConcurrentActivities: 20,  Register: ...OverviewWorkflow},
    {Name: "project_note",         TaskQueue: "plaud-project-note-tasks",
     MaxConcurrentActivities: 20,  Register: ...ProjectNoteWorkflow},
    {Name: "transcript-compress",  TaskQueue: "plaud-transcript-compress-tasks",
     MaxConcurrentActivities: 20,  Register: ...TranscriptCompressWorkflow},
}
```

每个 Workflow 一个独立 Task Queue —— 不同业务的限流和并发互不影响。

### 11.5 Demo 与生产的映射

如果对照官方 Demo（Python s1–s6）理解生产：


| Demo 文件                     | 生产映射                                     |
| --------------------------- | ---------------------------------------- |
| `temporal server start-dev` | `deploy/plaud-temporal/`（官方 Helm Chart）  |
| `s5_worker.py`              | `plaud-temporal-backend`（多 Worker 注册）    |
| `s2_workflows.py`           | `internal/workflow/*.go`                 |
| `s3_activities.py`          | `internal/activity/*.go`                 |
| `s1_starter.py`             | `plaud-file`（启动方）                        |
| `s4_external_service.py`    | `plaud-project-summary`（下游 + Signal 回调方） |
| `s6_shared.py`              | `internal/types/*.go`（双方契约由协议文档约定）       |


---

## 十二、一句话总结

> **Temporal 的本质**：把业务流程每一步执行结果持久化到 Server 数据库，通过事件重放实现「代码一定执行完」的承诺。
>
> **三进程的约束**：Client 永不直连 Worker，所有交互经 Server；Worker 完全无状态。
>
> **重建不是恢复**：每次新建 Workflow 实例、从头执行代码、靠历史快进到当前位置。
>
> **start_workflow 的本质**：发一次 gRPC 请求给 Server，Server 在数据库事务里完成写记录 + 入队；Worker 通过长轮询拉任务执行。
>
> **任务调度即限流**：`worker.Options` 控本地并发与速率，`TaskQueueActivitiesPerSecond` 控全局速率，配合下游业务级 Semaphore 形成多层闸门。
>
> **跨服务回调的关键**：Activity 调下游时携带 `workflow_id` → 下游处理完后用 `workflow_id` 拿 handle → 通过 Signal 回传结果。

