# Temporal 学习笔记

---

## 一、Temporal 解决什么问题？

### 没有 Temporal 的世界

假设你要实现一个业务流程：「调用 AI 服务处理数据 → 等待处理完成 → 将结果存入数据库 → 通知用户」。

传统做法会遇到以下问题：


| 问题        | 具体场景             | 后果                  |
| --------- | ---------------- | ------------------- |
| **进程崩溃**  | 服务器重启，内存中的流程状态丢失 | 任务卡死，用户永远收不到结果      |
| **网络故障**  | 调用 AI 服务超时       | 不知道该重试还是放弃，可能重复处理   |
| **长时间等待** | AI 处理需要 10 分钟    | 线程/连接被长时间占用，资源耗尽    |
| **状态追踪**  | 需要知道每个任务进行到哪一步   | 要自己建状态表、写状态机，复杂且易出错 |


### Temporal 的答案

Temporal 把上述问题抽象为一个核心承诺：**你的代码逻辑一定会执行完，即使中间发生任何故障。**

它的实现方式是：把每一步执行的结果持久化到数据库，故障恢复时通过「重放历史」重建状态，而非依赖内存。

---

## 二、系统架构：三个独立进程

Temporal 系统由三个独立进程组成，各司其职：

```
┌─────────────────┐                    ┌─────────────────┐
│  Temporal Server │                    │     Worker      │
│  (调度 + 存储)    │ ◄── gRPC 轮询 ──── │  (执行业务代码)   │
│                  │                    │                  │
│  端口 7233: gRPC  │                    │  注册:            │
│  端口 8233: Web UI│                    │  - Workflow 类    │
│                  │                    │  - Activity 函数  │
└────────▲─────────┘                    └─────────────────┘
         │
         │ gRPC 连接
         │
┌────────┴─────────┐
│     Client       │
│  (发起请求)       │
│                  │
│  启动 Workflow    │
│  发送 Signal     │
│  查询 Query      │
└─────────────────┘
```


| 进程                  | 职责                         | 不做什么       | 本项目对应文件                                                     |
| ------------------- | -------------------------- | ---------- | ----------------------------------------------------------- |
| **Temporal Server** | 任务调度、状态存储、Signal 路由、定时器管理  | 不执行任何业务代码  | `docker compose up` 或 `temporal server start-dev`           |
| **Worker**          | 执行 Workflow 和 Activity 代码  | 不存储状态（无状态） | `s5_worker.py`                                              |
| **Client**          | 启动 Workflow、发送 Signal、查询状态 | 不执行业务逻辑    | `s1_starter.py`、`s4_external_service.py`、`signal_sender.py` |


**关键理解：** 所有进程都通过同一行代码连接到 Server：

```python
client = await Client.connect("localhost:7233")
```

但角色不同——Worker 用它来轮询任务和上报结果，Client 用它来提交任务和发送 Signal。

---

## 三、六大核心概念

### 3.1 Workflow —— 流程编排器

**是什么：** 用代码定义业务流程的执行顺序。它是一个 Python 类，描述「先做 A，再做 B，如果 B 失败就做 C」。

**核心约束：必须确定性。** 因为 Workflow 会被反复重放（后面详解），相同输入必须产生相同的执行路径。

```python
# ❌ 禁止：非确定性代码
if random.random() > 0.5:        # 每次重放结果不同
    await activity_a()
if datetime.now().hour > 12:     # 重放时时间变了
    await activity_b()

# ✅ 正确：用 Temporal 提供的确定性 API
if await workflow.random() > 0.5:
    await activity_a()
```

**怎么写：** 参见 `s2_workflows.py`

```python
from temporalio import workflow

@workflow.defn                          # 1. 用装饰器标记类
class ExternalServiceWorkflow:
    def __init__(self):
        self._result_received = False   # 2. 状态变量（会通过重放恢复）

    @workflow.run                        # 3. 主入口（必须，有且仅有一个）
    async def run(self, input: WorkflowInput) -> WorkflowOutput:
        # 通过 Activity 执行实际 I/O
        task_id = await workflow.execute_activity(
            send_task_to_external_service,
            args=[workflow_id, input.data],
            start_to_close_timeout=timedelta(seconds=30),
        )
        # 等待外部回调
        await workflow.wait_condition(lambda: self._result_received)
        return WorkflowOutput(...)

    @workflow.signal                     # 4. Signal 处理器（可选，可多个）
    async def receive_result(self, result: TaskResult):
        self._task_result = result
        self._result_received = True     # 触发 wait_condition

    @workflow.query                      # 5. Query 处理器（可选，只读）
    def get_status(self) -> str:
        return "Completed" if self._result_received else "Waiting"
```

**Workflow 类中各方法的必要性：**

| 组成部分 | 必须？ | 数量限制 | 说明 |
| --- | --- | --- | --- |
| `@workflow.defn` | **必须** | 1 个类装饰器 | 标记类为 Workflow 定义 |
| `@workflow.run` | **必须** | 有且仅有 1 个 | Workflow 主入口，相当于 `main()` |
| `__init__` | 可选 | 0~1 个 | 初始化状态变量，有 Signal/Query 时通常需要 |
| `@workflow.signal` | 可选 | 0~多个 | 接收外部异步数据，需要外部输入时才用 |
| `@workflow.query` | 可选 | 0~多个 | 只读查询状态，不影响执行、不记录到历史 |

> **最小合法 Workflow** 只需要 `@workflow.defn` + `@workflow.run`。Signal 和 Query 都是按需添加的能力——纯顺序执行的 Workflow（启动 → 执行若干 Activity → 返回结果）完全不需要它们。

**Workflow 中导入模块的特殊处理：**

```python
# Workflow 代码会被重放，普通 import 的模块副作用会被重复执行
# 必须用 imports_passed_through() 包裹
with workflow.unsafe.imports_passed_through():
    from s3_activities import send_task_to_external_service
    from s6_shared import TaskResult
```

---

### 3.2 Activity —— 实际干活的函数

**是什么：** 执行具体 I/O 操作的函数——HTTP 请求、数据库读写、文件操作等。

**为什么需要它：** Workflow 禁止直接做 I/O（因为要确定性），所有副作用都必须封装在 Activity 中。


| 对比项    | Workflow | Activity  |
| ------ | -------- | --------- |
| 确定性    | 必须确定性    | 可以非确定性    |
| I/O 操作 | 禁止       | 允许        |
| 失败恢复   | 通过事件重放   | 通过重新执行    |
| 运行时间   | 可以持续数月   | 应尽量短（有超时） |


**怎么写：** 参见 `s3_activities.py`

```python
from temporalio import activity

@activity.defn                          # 用装饰器标记函数
async def send_task_to_external_service(
    workflow_id: str, data: str
) -> str:
    task_id = str(uuid.uuid4())

    async with aiohttp.ClientSession() as session:
        async with session.post(url, json={
            "task_id": task_id,
            "workflow_id": workflow_id,  # ⚠️ 关键：传给外部服务用于 Signal 回调
            "data": data,
        }) as response:
            if response.status == 200:
                return task_id
            raise Exception(f"HTTP {response.status}")
```

**怎么在 Workflow 中调用 Activity：**

```python
task_id = await workflow.execute_activity(
    send_task_to_external_service,       # Activity 函数引用
    args=[workflow_id, input.data],      # 参数列表
    start_to_close_timeout=timedelta(seconds=30),  # 超时时间（必须设置）
    retry_policy=RetryPolicy(            # 重试策略（可选）
        maximum_attempts=3,              # 最多 3 次（1 原始 + 2 重试）
        initial_interval=timedelta(seconds=1),
        maximum_interval=timedelta(seconds=10),
    ),
)
```

---

### 3.3 Worker —— 执行引擎

**是什么：** 一个长时间运行的进程，从 Task Queue 拉取任务并执行 Workflow/Activity 代码。

**核心特性：无状态。** Worker 随时可以崩溃重启，任何 Worker 都能接手任何 Workflow，因为状态在 Server 端。

**怎么写：** 参见 `s5_worker.py`

```python
from temporalio.client import Client
from temporalio.worker import Worker

async def main():
    # 1. 连接 Temporal Server
    client = await Client.connect("localhost:7233")

    # 2. 创建 Worker 实例
    worker = Worker(
        client,
        task_queue=TASK_QUEUE,               # 监听哪个队列
        workflows=[                          # 注册能处理的 Workflow 类型
            ExternalServiceWorkflow,
            SimpleSignalWorkflow,
        ],
        activities=[                         # 注册能执行的 Activity 函数
            send_task_to_external_service,
            process_result,
        ],
    )

    # 3. 运行（无限循环：轮询 → 执行 → 上报结果）
    await worker.run()
```

**一个 Worker 可以注册多种 Workflow 和 Activity。** 注册的是「能力」——我能处理哪些类型。实际执行什么取决于 Client 往队列里提交了什么。

**多 Worker 声明式配置（生产模式）：**

```python
@dataclass
class WorkerDef:
    name: str
    task_queue: str
    workflows: list[type] = field(default_factory=list)
    activities: list[Any] = field(default_factory=list)

WORKER_DEFS = [
    WorkerDef(name="ocr", task_queue="ocr-tasks", workflows=[OCRWorkflow], ...),
    WorkerDef(name="overview", task_queue="overview-tasks", workflows=[OverviewWorkflow], ...),
]

# 用 asyncio.gather 并发运行多个 Worker
await asyncio.gather(*[create_worker(d).run() for d in WORKER_DEFS])
```

---

### 3.4 Task Queue —— 任务通道

**是什么：** 连接 Server 和 Worker 的逻辑通道。Worker 通过 Task Queue 名称决定「我处理哪些任务」。

**关键点：**

- 一个 Worker 只能监听一个 Task Queue
- 多个 Worker 可以监听同一个 Task Queue（水平扩展）
- Task Queue 名称在启动 Workflow 和注册 Worker 时必须一致

**代码中的使用（三处必须一致）：**

```python
# s6_shared.py —— 统一定义
TASK_QUEUE = "external-service-task-queue"

# s5_worker.py —— Worker 监听
worker = Worker(client, task_queue=TASK_QUEUE, ...)

# s1_starter.py —— Client 提交
await client.execute_workflow(..., task_queue=TASK_QUEUE)
```

**为什么不用 Kafka？** Temporal 用「数据库 + 长轮询」实现 Task Queue，原因：

1. **同步匹配**：有 Worker 等着就直接给它，不用写入再读出
2. **精确一次**：数据库事务（`SELECT ... FOR UPDATE`）保证任务不会被两个 Worker 同时拿到
3. **原子性**：写历史事件和创建新任务在同一个事务中，不会出现「历史记录了完成，但下一步没触发」的情况

---

### 3.5 Signal —— 向 Workflow 发送数据

**是什么：** 外部向正在运行的 Workflow 发送数据的机制。单向、异步。

**使用场景：** 外部服务处理完成后通知 Workflow、用户输入审批结果、手动干预流程等。

**发送 Signal 必须知道什么：**


| 信息              | 是否必需    | 从哪获取                                  |
| --------------- | ------- | ------------------------------------- |
| **Workflow ID** | 必需      | Client 启动 Workflow 时指定的 `id` 参数       |
| **Signal 名称**   | 必需      | Workflow 类中 `@workflow.signal` 方法的函数名 |
| **Signal 数据**   | 看定义     | Signal 处理器的参数                         |
| Task Queue      | **不需要** | Signal 不经过 Task Queue，直接由 Server 路由   |


**Signal 的流转路径：**

```
发送方 ──(workflow_id)──► Temporal Server ──(路由)──► 目标 Workflow
                          不经过 Task Queue
```

**怎么发送 Signal（三种场景）：**

```python
# 场景 1：Client 拿着 handle 发送（刚启动的 Workflow）
handle = await client.start_workflow(SimpleSignalWorkflow.run, id=wf_id, task_queue=TASK_QUEUE)
await handle.signal(SimpleSignalWorkflow.send_message, "Hello")

# 场景 2：通过 workflow_id 发送（已运行的 Workflow，不需要知道类型）
handle = client.get_workflow_handle(workflow_id)        # 只需要 workflow_id
await handle.signal("receive_result", result)           # Signal 名称用字符串

# 场景 3：外部服务发送 Signal（先连接 Server，再获取 handle）
client = await Client.connect("localhost:7233")         # 外部服务也要连 Temporal Server
handle = client.get_workflow_handle(workflow_id)         # 用 HTTP 请求中携带的 workflow_id
await handle.signal("receive_result", result)
```

**怎么在 Workflow 中接收 Signal：**

```python
@workflow.signal
async def receive_result(self, result: TaskResult):
    self._task_result = result
    self._result_received = True    # 修改状态，触发 wait_condition
```

**怎么等待 Signal：**

```python
# 方式 1：无超时等待
await workflow.wait_condition(lambda: self._result_received)

# 方式 2：带超时等待
await asyncio.wait_for(
    workflow.wait_condition(lambda: self._result_received),
    timeout=120,  # 秒
)
```

`wait_condition` 不会阻塞 Signal 的接收——等待期间 Signal 正常入队并执行。

---

### 3.6 Query —— 只读查询 Workflow 状态

**是什么：** 同步查询 Workflow 当前状态的机制。只读，不影响执行，不记录到历史。

**怎么定义：**

```python
@workflow.query
def get_status(self) -> str:
    return "Completed" if self._result_received else "Waiting"
```

**怎么调用：**

```python
handle = client.get_workflow_handle(workflow_id)
status = await handle.query("get_status")          # 字符串方式
# 或
status = await handle.query(ExternalServiceWorkflow.get_status)  # 类型安全方式
```

---

## 四、核心机制：重建而非恢复

这是理解 Temporal 最关键的一点。

### 常见误解 vs 实际情况

- **❌ 误解（恢复）：** Workflow 启动后一直在内存中运行，遇到 `await` 暂停，等 Activity 完成后继续
- **✅ 实际（重建）：** 每次有新事件时，Worker 创建新的 Workflow 实例，从头执行代码，通过历史快进到当前位置

### 生命周期

```
Workflow Task #1                      Workflow Task #2
    │                                      │
    ▼                                      ▼
┌──────────┐                          ┌──────────┐
│ 新建实例  │                          │ 新建实例  │  ← 可能是同一个 Worker
│ 执行代码  │                          │ 从头执行  │    也可能是不同 Worker
│ 到 await │                          │ 重放历史  │
│ 产生命令  │                          │ 继续到下  │
│ 实例销毁  │ ← 没了，内存释放          │ 一个await│
└──────────┘                          │ 实例销毁  │ ← 又没了
                                      └──────────┘
```

### 类比

- **❌ 恢复** = 游戏暂停后继续（内存中的状态还在）
- **✅ 重建** = 看录像带快进（关掉游戏 → 重新打开 → 靠存档快进到之前进度 → 继续玩）

### 为什么这样设计？

1. **无状态 Worker**：Worker 随时崩溃重启，任意 Worker 都能接手
2. **状态在 Server**：事件历史就是「真相」，不依赖 Worker 内存
3. **超长运行**：Workflow 可以跑数月而不占用 Worker 内存

---

## 五、Workflow Task vs Activity Task

Worker 内部有两套独立的处理器，处理两种不同的任务：


| 维度         | Workflow Task（「通知」）     | Activity Task（「任务」） |
| ---------- | ----------------------- | ------------------- |
| Server 发什么 | 事件历史                    | 函数名 + 参数            |
| Worker 做什么 | 重放代码，决策下一步              | 调用 Activity 函数      |
| 产出         | 命令（如「请调度 XXX Activity」） | 执行结果                |
| 耗时         | 毫秒级                     | 可能很长（秒、分钟）          |


**什么时候产生 Workflow Task？**


| 触发事件         | Server 的动作      |
| ------------ | --------------- |
| Workflow 刚启动 | Worker 你来决定第一步  |
| Activity 完成  | Worker 你来决定下一步  |
| 收到 Signal    | Worker 你来处理这个信号 |
| Timer 到期     | Worker 你来继续执行   |


### 完整执行时间线

以 `ExternalServiceWorkflow` 为例：

```
Workflow Task #1    Activity Task #1    Workflow Task #2    等待 Signal...
┌──────────┐        ┌──────────┐        ┌──────────┐
│ 执行到    │        │ HTTP POST│        │ 重放 #1  │        ⏳ Workflow 暂停
│ await    │──────► │ 外部服务  │──────► │ 到 wait  │        状态持久化在 Server
│ activity │        │          │        │ condition│
└──────────┘        └──────────┘        └──────────┘
                                                            ... 外部服务处理中 ...

Signal 到达 ──►  Workflow Task #3    Activity Task #2    Workflow Task #4
                 ┌──────────┐        ┌──────────┐        ┌──────────┐
                 │ 重放 #1#2│        │ process  │        │ 重放全部  │
                 │ 到 await │──────► │ result   │──────► │ return   │
                 │ activity │        │          │        │ 完成！    │
                 └──────────┘        └──────────┘        └──────────┘
```

### 每次 Workflow Task 拿到的历史

```
Task #1: [ WorkflowExecutionStarted ]
  → 从头执行，遇到第一个 await，生成 ScheduleActivityTask 命令

Task #2: [ ...Started, ActivityScheduled, ActivityCompleted{result} ]
  → 重放：第一个 await 有历史结果，跳过
  → 继续到 wait_condition，暂停

Task #3: [ ...前面的事件..., SignalReceived{data} ]
  → 重放：前面的 await 跳过，Signal 修改状态
  → wait_condition 满足，继续到第二个 await activity

Task #4: [ ...所有事件..., ActivityCompleted{final_result} ]
  → 重放：所有 await 跳过
  → 执行到 return，Workflow 完成
```

---

## 六、两个实战案例

### 案例 1：ExternalServiceWorkflow（HTTP + Signal 异步回调）

这是最核心的模式：Workflow 发起请求 → 外部服务异步处理 → 通过 Signal 回调结果。

#### 完整数据流（10 步）

```
s1_starter.py                  Temporal Server              s5_worker.py
    │                               │                           │
    │  1. execute_workflow()         │                           │
    │ ─────────────────────────────► │  2. Workflow Task ──────► │
    │                               │                           │
    │                               │  3. Activity Task ◄────── │ (execute_activity)
    │                               │ ─────────────────────────►│
    │                               │                           │
    │                               │               s3_activities.py
    │                               │                    │
    │                               │                    │ 4. HTTP POST
    │                               │                    │    (携带 workflow_id)
    │                               │                    ▼
    │                               │            s4_external_service.py
    │                               │                    │
    │                               │                    │ 5. 返回 "accepted"
    │                               │                    │
    │                               │                    │ 6. 后台异步处理...
    │                               │ ◄──────────────────│ 7. Signal (workflow_id)
    │                               │                    │
    │                               │  8. Workflow Task ──────► │ (wait_condition 满足)
    │                               │                           │
    │                               │  9. Activity Task ──────► │ (process_result)
    │                               │                           │
    │  10. 返回 WorkflowOutput      │ ◄─────────────────────── │ (return)
    │ ◄──────────────────────────── │                           │
```

#### 外部服务怎么发 Signal（关键步骤）

参见 `s4_external_service.py`，外部服务需要做三件事：

```python
# 第 1 步：启动时连接 Temporal Server（只需一次）
temporal_client = await Client.connect("localhost:7233")

# 第 2 步：用 workflow_id 获取 handle（workflow_id 从 HTTP 请求中获取）
handle = temporal_client.get_workflow_handle(workflow_id)

# 第 3 步：发送 Signal
await handle.signal("receive_result", TaskResult(
    task_id=task_id,
    status=TaskStatus.COMPLETED.value,
    result=processed_result,
))
```

为什么外部服务知道 `workflow_id`？因为 Activity 发 HTTP 请求时把它放在了请求体里：

```python
# s3_activities.py 中
request_data = {
    "task_id": task_id,
    "workflow_id": workflow_id,   # ⚠️ 这就是回调的「钥匙」
    "data": data,
}
```

#### 为什么用 Signal 而不是 HTTP 回调？

1. **可靠**：Signal 由 Temporal 保证送达，即使 Worker 暂时不可用
2. **持久化**：Signal 会记录到 Workflow 历史，不会丢失
3. **简化架构**：Workflow 不需要暴露 HTTP 端点来接收回调
4. **类型安全**：Signal 参数有类型检查

---

### 案例 2：SimpleSignalWorkflow（基础 Signal 交互）

一个更简单的示例，演示多 Signal + Query 的基本用法。

参见 `s2_workflows.py` 中的 `SimpleSignalWorkflow`：

```python
@workflow.defn
class SimpleSignalWorkflow:
    def __init__(self):
        self._messages: list[str] = []
        self._should_exit: bool = False

    @workflow.signal
    async def send_message(self, message: str):   # Signal 1：接收消息
        self._messages.append(message)

    @workflow.signal
    async def exit(self):                          # Signal 2：退出信号
        self._should_exit = True

    @workflow.query
    def get_messages(self) -> list[str]:            # Query：查询消息
        return self._messages.copy()               # 返回副本，避免外部修改

    @workflow.run
    async def run(self) -> list[str]:
        await workflow.wait_condition(lambda: self._should_exit)
        return self._messages
```

**Client 端交互流程：** 参见 `s1_starter.py`

```python
# 1. 启动 Workflow（不等待完成）
handle = await client.start_workflow(SimpleSignalWorkflow.run, id=wf_id, task_queue=TASK_QUEUE)

# 2. 发送多个消息 Signal
await handle.signal(SimpleSignalWorkflow.send_message, "First message")
await handle.signal(SimpleSignalWorkflow.send_message, "Second message")

# 3. Query 查询当前状态
messages = await handle.query(SimpleSignalWorkflow.get_messages)

# 4. 发送退出 Signal
await handle.signal(SimpleSignalWorkflow.exit)

# 5. 等待结果
result = await handle.result()
```

---

## 七、Client API：触发 Workflow 执行

### 7.1 两种启动方式


| 方式                 | 行为              | 返回值            | 适用场景                 |
| ------------------ | --------------- | -------------- | -------------------- |
| `execute_workflow` | 启动 **并阻塞等待** 完成 | Workflow 返回值   | 同步调用，需要立即拿到结果        |
| `start_workflow`   | 仅启动，**立即返回**    | WorkflowHandle | 需要后续交互（Signal/Query） |


`execute_workflow` 本质等同于 `start_workflow()` + `handle.result()`。

---

### 7.2 两种调用风格：类型安全 vs 字符串

Temporal Python SDK 支持两种方式指定要启动的 Workflow：

```python
# ✅ 风格 1：类型安全（同一 Worker 进程内，能 import Workflow 类）
handle = await client.start_workflow(
    ExternalServiceWorkflow.run,     # 直接引用 Workflow 类的 @workflow.run 方法
    input_data,                       # 单个参数直接传
    id=workflow_id,
    task_queue=TASK_QUEUE,
)

# ✅ 风格 2：字符串（跨服务调用，无法 import 对方的 Workflow 类）
handle = await client.start_workflow(
    "TranscriptCompressWorkflow",    # Workflow Type 名称（字符串）
    payload,                          # 通常是 dict，对方 Workflow 自行解析
    id=workflow_id,
    task_queue="plaud-transcript-compress-tasks",
)
```

**什么时候用哪种？**

| 场景 | 推荐风格 | 原因 |
| --- | --- | --- |
| 同一代码库，能 import Workflow 类 | 类型安全 | IDE 补全、参数校验、重构安全 |
| 跨服务/跨语言（如 Python → Go） | 字符串 | 无法 import 对方代码，只能约定类型名称 |

> **注意：** 字符串方式传的 Workflow Type 名称必须和对方 Worker 注册的名称**完全一致**。

---

### 7.3 完整参数参考

`start_workflow` / `execute_workflow` 支持的完整参数：

```python
handle = await client.start_workflow(
    # ═══ 必须参数 ═══
    workflow,                          # Workflow 引用或字符串类型名
    arg,                               # 传给 @workflow.run 的参数（单参数）
    # 或 args=[arg1, arg2],            # 多参数用 args 列表
    id="workflow-id",                  # Workflow ID（全局唯一，建议映射业务实体 ID）
    task_queue="task-queue-name",      # Worker 监听的 Task Queue

    # ═══ 超时参数 ═══
    execution_timeout=timedelta(...),  # 整个 Workflow 执行的总超时（含重试 + ContinueAsNew）
    run_timeout=timedelta(...),        # 单次 Workflow Run 的超时
    task_timeout=timedelta(...),       # 单次 Workflow Task（决策步骤）的超时

    # ═══ 重试 & ID 策略 ═══
    retry_policy=RetryPolicy(...),     # Workflow 级别重试策略
    id_reuse_policy=                   # ID 冲突处理策略:
        WorkflowIDReusePolicy.ALLOW_DUPLICATE,
        # ALLOW_DUPLICATE: 允许（前一个已完成/失败/取消时）
        # ALLOW_DUPLICATE_FAILED_ONLY: 仅前一个失败/取消时允许
        # REJECT_DUPLICATE: 拒绝，抛 WorkflowAlreadyStartedError
        # TERMINATE_IF_RUNNING: 终止正在运行的同 ID workflow

    # ═══ 可观测性 ═══
    memo={"key": "value"},             # 附加元数据（可在 Web UI 中查看）
    search_attributes=...,             # 搜索属性（用于 Visibility API 过滤）

    # ═══ 高级选项 ═══
    cron_schedule="0 * * * *",         # Cron 定时调度表达式
    start_delay=timedelta(minutes=5),  # 延迟启动
    request_eager_start=True,          # 请求立即调度（减少一次轮询延迟）
    rpc_timeout=timedelta(seconds=10), # gRPC 调用本身的超时
)
```

**最常用的核心四参数：**

| 参数 | 必须？ | 说明 |
| --- | --- | --- |
| `workflow` | **必须** | Workflow 类引用或字符串类型名 |
| `arg` / `args` | 看定义 | 传给 `@workflow.run` 的参数 |
| `id` | **必须** | Workflow ID，建议映射业务实体 |
| `task_queue` | **必须** | 必须和 Worker 注册的一致 |

---

### 7.4 实际项目示例：跨服务触发 Workflow

以下是本项目中**触发 Go 服务的 TranscriptCompressWorkflow** 的完整代码（跨服务场景）：

```python
from common.config import get_settings
from common.temporal import get_temporal_client

settings = get_settings()
compress_config = settings.services.transcript_compress

# 1. 构造 Workflow ID（建议包含业务 ID + 随机后缀防重复）
workflow_id = f"compress-{file_info.file_id}-{uuid.uuid4().hex[:8]}"

# 2. 构造 payload（跨服务用 dict，双方约定 schema）
payload = {
    "schema_version": "1.0",
    "request_id": workflow_id,
    "tenant_id": compress_config.tenant_id,
    "user_id": user_id,
    "file_id": file_info.file_id,
    "transcript_s3_uri": file_info.primary_content_path,
    "content_type": _get_file_type(file_info.source_type),
    "downstream_url": compress_config.downstream_url,  # 完成后的回调地址
}

# 3. 获取 Temporal Client（单例，复用连接）
client = await get_temporal_client()

# 4. 启动 Workflow（字符串方式，因为 Workflow 定义在 Go 服务中）
handle = await client.start_workflow(
    compress_config.workflow_type,      # "TranscriptCompressWorkflow"
    payload,                            # dict 作为参数
    id=workflow_id,                     # 唯一 ID
    task_queue=compress_config.task_queue,  # "plaud-transcript-compress-tasks"
)

# 5. 等待完成（带超时保护）
await asyncio.wait_for(handle.result(), timeout=180)
```

**这段代码做了什么：**

```
Python 服务                    Temporal Server              Go Worker
    │                               │                          │
    │  start_workflow()             │                          │
    │  type="Transcript..."        │                          │
    │  queue="plaud-transcript..." │                          │
    │ ─────────────────────────►   │  派发到对应 queue ──────► │
    │                               │                          │ 执行压缩...
    │  handle.result() 阻塞等待     │                          │
    │  (timeout=180s)               │  ◄── 完成上报 ────────── │
    │ ◄──────────── 返回结果 ────── │                          │
```

**关键点：**

- **跨语言透明**：Python Client 启动，Go Worker 执行，Temporal Server 居中调度。双方通过 **Workflow Type 名称** + **Task Queue 名称** 约定即可，无需共享代码
- **payload 是 dict**：跨服务时用 dict/JSON 传参，而非 dataclass，因为无法共享类型定义
- **超时保护**：`asyncio.wait_for(handle.result(), timeout=180)` 防止对方服务未部署时无限阻塞
- **配置外置**：Workflow Type、Task Queue、downstream_url 等参数从 AppConfig 读取，不同 region 可独立配置

---

### 7.5 Workflow Type vs Workflow ID

```
Workflow Type = 类/模板      如 ExternalServiceWorkflow
Workflow ID   = 对象/实例    如 "external-service-workflow-abc123"

启动时需要：  Type + ID + Task Queue
发送 Signal： 只需要 ID（Server 已记录 Type）
```

**对应 CLI 命令：**


| Python SDK                    | CLI 参数                                     |
| ----------------------------- | ------------------------------------------ |
| `ExternalServiceWorkflow.run` | `--type ExternalServiceWorkflow`           |
| `id=workflow_id`              | `--workflow-id xxx`                        |
| `task_queue=TASK_QUEUE`       | `--task-queue external-service-task-queue` |
| `arg=input_data`              | `--input '{...}'`                          |


---

## 八、错误处理

### Activity 重试

Activity 失败时 Temporal 会自动重试，通过 `RetryPolicy` 控制：

```python
retry_policy=RetryPolicy(
    maximum_attempts=3,                      # 最多 3 次
    initial_interval=timedelta(seconds=1),   # 首次重试等 1 秒
    maximum_interval=timedelta(seconds=10),  # 间隔上限 10 秒
    # backoff_coefficient=2.0,               # 退避系数（默认 2.0，每次翻倍）
    # non_retryable_error_types=[],          # 不重试的错误类型
)
```

### 不可重试的业务错误

```python
from temporalio.exceptions import ApplicationError

# non_retryable=True → 立即失败，不触发重试
raise ApplicationError(
    "External service callback timeout",
    non_retryable=True,
)
```

**区分两种错误：**

- **可重试**（网络抖动、临时故障）：让 Temporal 自动重试
- **不可重试**（业务超时、数据错误）：用 `ApplicationError(non_retryable=True)` 立即失败

---

## 九、Namespace 隔离

**是什么：** 类似 Kubernetes Namespace，在同一个 Temporal Server 上提供逻辑隔离。

```
┌──────────────────────────────────────────────────┐
│                Temporal Server                     │
├────────────────┬────────────────┬────────────────┤
│  "default"     │  "production"  │  "staging"     │
│  (开发测试)     │  (生产环境)     │  (预发布)       │
└────────────────┴────────────────┴────────────────┘
```

- 不同 Namespace 的 Workflow **完全隔离**，互不可见
- 同名 Workflow ID 在不同 Namespace 中可以共存
- 不指定时使用 `"default"` Namespace
- 一个 Namespace 内可以有多个 Task Queue 和多种 Workflow Type

---

## 十、项目文件结构与启动顺序

### 文件职责


| 文件                       | 角色          | 核心职责                                               |
| ------------------------ | ----------- | -------------------------------------------------- |
| `s6_shared.py`           | 共享定义        | Task Queue 名称、数据结构（TaskResult、WorkflowInput 等）     |
| `s2_workflows.py`        | Workflow 定义 | ExternalServiceWorkflow + SimpleSignalWorkflow     |
| `s3_activities.py`       | Activity 定义 | HTTP 发送 + 结果处理                                     |
| `s4_external_service.py` | 模拟外部服务      | HTTP 接收请求 → 异步处理 → **连接 Temporal Server 发 Signal** |
| `s5_worker.py`           | Worker      | 注册 Workflow/Activity，轮询 Task Queue                 |
| `s1_starter.py`          | Client      | 启动 Workflow，发送 Signal，查询状态                         |
| `signal_sender.py`       | 调试工具        | 手动发送 Signal/Query 的 CLI                            |


### 启动顺序

```bash
# 1. Temporal Server（依赖项，必须先启动）
temporal server start-dev     # 开发模式，端口 7233(gRPC) + 8233(Web UI)

# 2. 外部服务（HTTP 服务器，等待接收请求）
python s4_external_service.py  # localhost:8080

# 3. Worker（连接 Server，开始轮询任务）
python s5_worker.py

# 4. Client（启动 Workflow，触发整个流程）
python s1_starter.py           # 选择示例 1 或 2
```

### 三处连接同一个 Server，角色不同


| 文件                       | 代码                                 | 角色                   |
| ------------------------ | ---------------------------------- | -------------------- |
| `s1_starter.py`          | `Client.connect("localhost:7233")` | 提交 Workflow、发 Signal |
| `s5_worker.py`           | `Client.connect("localhost:7233")` | 轮询 Task Queue、上报结果   |
| `s4_external_service.py` | `Client.connect("localhost:7233")` | 获取 handle 发 Signal   |


---

## 十一、与生产项目的映射

Demo（Python）对应生产项目 `plaud-temporal-backend`（Go）的核心模式——**异步外部服务回调**。

### 角色映射


| Demo 文件                  | 生产项目                      | 说明           |
| ------------------------ | ------------------------- | ------------ |
| `s2_workflows.py`        | `internal/workflow/*.go`  | Workflow 编排  |
| `s3_activities.py`       | `internal/activity/*.go`  | Activity 实现  |
| `s5_worker.py`           | `main.go` + `registry.go` | Worker 注册与启动 |
| `s6_shared.py`           | `internal/types/*.go`     | 共享数据结构       |
| `s1_starter.py`          | 上游业务服务（不在仓库内）             | 外部 Client    |
| `s4_external_service.py` | 下游处理服务 + Notifier（不在仓库内）  | 外部回调         |


### 生产项目比 Demo 多了什么


| 特性       | 说明                                                           |
| -------- | ------------------------------------------------------------ |
| 上游回调     | 多了 `CallbackUpstream` Activity，主动通知上游服务                      |
| 多 Worker | `registry.go` 管理多个 Worker（OCR、Overview、TranscriptCompress 等） |
| 可观测性     | OpenTelemetry + Prometheus + 结构化日志                           |
| 配置管理     | AWS AppConfig + 多环境配置                                        |
| TLS      | 支持 mTLS，适配 Temporal Cloud                                    |
| 优雅关闭     | 监听 SIGINT/SIGTERM，等运行中的 Activity 完成再退出                       |
| 统一信号格式   | Signal Payload 统一为 `{code, message, data}`                   |


### 数据流对比

```
Demo（单体演示）:
  Client ──► Workflow ──► Activity ──HTTP──► External Service
                ▲                                  │
                └──────── Signal callback ─────────┘

生产（微服务）:
  上游服务 ──► Workflow ──► Activity ──HTTP──► 下游服务
                 ▲                                │
                 └──── Signal (via Notifier) ─────┘
                 │
                 └──► CallbackUpstream ──► 上游服务
```

---

## 十二、一句话总结

> **Temporal 的本质**：把业务流程的每一步持久化到数据库，通过事件重放实现「代码一定会执行完」的承诺。
>
> **Workflow 是重建不是恢复**：每次新建实例、从头执行、靠历史快进到当前位置。
>
> **外部服务回调的关键**：Activity 发请求时带上 `workflow_id` → 外部服务处理完后用 `workflow_id` 获取 handle → 通过 handle 发 Signal 回调。

