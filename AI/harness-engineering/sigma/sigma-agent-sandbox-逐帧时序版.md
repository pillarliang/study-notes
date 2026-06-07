# Sigma-Agent Sandbox 交互逐帧时序版

> **配合文档**：[sigma-agent-完整数据流与架构-通俗版](sigma-agent-完整数据流与架构-通俗版.md)（架构总览）+ 本文档（逐帧时序）。
>
> **本文档解决的痛点**：知道有这些组件、知道有 4 条通道，但**时序对不上、不知道哪一帧谁在动**。
>
> **读法**：
> - 第 1 节 **全帧速览表**：30 秒扫一眼，知道总共多少帧、每帧是谁动。
> - 第 3 节 **逐帧详解**：每帧 = 一张卡片，固定 5 栏：`时间 / 主语 / 代码位置 / 状态变化 / 为什么这么设计`。
> - 第 5 节 **时序 FAQ**：把"看上去顺序对不上"的 6 个常见错觉一次性掰开。

---

## 1. 全帧速览表

> 选定的请求：用户在浏览器输入 **"帮我搜下今天北京的天气"**（这是 spawn 阶段，全新 agent）。

| # | T(ms) | 谁在动 | 一句话动作 | 主要状态变化 |
|---|---|---|---|---|
| 0 | T+0 | 浏览器 | WebSocket 帧投出 `chat.input` | — |
| 1 | T+1 | agent-service | Master AgentRunner 收帧 → 入对话历史 | — |
| 2 | T+1 | agent-service | Master 调 Claude API（流式） | — |
| 3 | T+300 | Anthropic | 返回 `tool_use: agent.spawn(...)` | — |
| 4 | T+301 | agent-service | Tool registry 解析 → HTTP POST `/api/v1/agents` | — |
| 5 | T+302 | worker-service | `spawnAgent` handler 收到，生成 agent_id | — |
| 6 | T+303 | worker-service | `manager.Spawn` 保存 meta + 启动 supervisor goroutine | Redis: `meta:status=spawning` |
| 6.5 | T+304 | worker-service | **HTTP 202 立刻返回 master**（不等 sandbox） | — |
| 7 | T+305 | worker-service | supervisor 派 goroutine `runAgent` | — |
| 8 | T+306 | worker-service | `assembleBootConfig` 组装 bootConfig | — |
| 9 | T+307~2000 | E2B / Docker | `sandbox.Create` 真正起容器 | sandbox URL 诞生 |
| 10 | T+2001 | worker-service | `compareAndSetRunFields` 写 sandbox_endpoint | Redis: `meta:sandbox_endpoint=...` |
| 11 | T+2002 | worker-service | `InvokeStream` 启动 sandbox 主流程 | — |
| 12 | T+2003 | worker-service | `enqueueInitialStartCommand` 投首条命令 | — |
| 13 | T+2003 | Redis | `enqueue_cmd.lua` 原子执行 5 件事 | Redis: `cmd` stream 新增 entry |
| 14 | T+2004 | worker-service | Dispatcher goroutine `run()` 被唤醒 | — |
| 15 | T+2004 | worker-service | `XReadGroup BLOCK` 拿到 entry | Pending list +1 |
| 16 | T+2005 | worker-service | `dispatchEntry` 读 meta → 拿 sandbox_endpoint | — |
| 17 | T+2006 | worker-service | HTTP `POST {sandbox}/commands` | — |
| 18 | T+2010 | sandbox | `/commands` handler 去重 + 按 action 分流 | — |
| 19 | T+2011 | sandbox | `user_input` 入主队列 | — |
| 20 | T+2011 | sandbox | **HTTP 200 立刻返回 worker**（不等执行） | — |
| 21 | T+2012 | worker-service | 收 200 → `XACK` | Pending list 移除 |
| 22 | T+2012 | sandbox | 主队列消费协程取出 → 调内部 `/invocations` | — |
| 23 | T+2013 | sandbox | `/invocations` 按 engine 选 adapter（claude → 复用 `_persistent_adapter`） | — |
| 24 | T+2014 | sandbox | adapter 调 Anthropic（base URL 已被改写到 gateway） | — |
| 25 | T+2014 | sandbox-gateway | 鉴权 + 限流 + 替换 API Key + 转发 | — |
| 26 | T+2300 | Anthropic | 流式吐第一个 token | — |
| 27 | T+2300 | sandbox | token → AgentEvent → 入出站事件缓冲 | — |
| 28 | T+2330 | sandbox | 触发 flush（30ms 到了）→ 批量 POST `/agents/{id}/events` | — |
| 29 | T+2331 | worker-service | events handler `IngestEvent` 处理 | — |
| 30 | T+2332 | Redis | `append_event.lua` 原子写 events stream | Redis: `events` stream 新增多条 |
| 31 | T+2333 | worker-service | `broadcastEvent` 走 Pub/Sub | Redis: `PUBLISH sigma.output.{sid}` |
| 32 | T+2334 | agent-service | Master 订阅器收到 → WebSocket 转给浏览器 | — |
| 33 | T+2335 | 浏览器 | 显示第一个 token | — |

**之后**：26→33 循环到 LLM 生成完毕，整个流程 sandbox 处于 idle，等下一条命令。

---

## 2. 角色卡（看不清谁是谁时翻这里）

| 角色 | 进程位置 | 一句话职责 | 关键代码入口 |
|---|---|---|---|
| **浏览器** | 用户机器 | WebSocket 收发 chat 帧 | — |
| **agent-service Master AgentRunner** | Go 进程，K8s pod | 跑 LLM 决定派活；订阅 sandbox 事件回流 | `services/agent-service/internal/...` |
| **worker-service spawn handler** | Go 进程，K8s pod | 收 master 的 spawn HTTP 请求，立即返回 agent_id | [handler.go:102](services/worker-service/internal/handler/handler.go#L102) |
| **worker-service AgentManager** | Go 进程，与 handler 同进程 | 管理 agent 元数据 + 启动 supervisor | [manager.go:683](services/worker-service/internal/agentmanager/manager.go#L683) |
| **worker-service supervisor 协程** | goroutine | 异步创建 sandbox、组装 bootConfig、写 endpoint | [manager.go:1391](services/worker-service/internal/agentmanager/manager.go#L1391) |
| **worker-service Dispatcher 协程** | goroutine（每个 agent 一个） | 从 Redis Stream 抢命令 → POST sandbox `/commands` → XACK | [dispatcher.go:343](services/worker-service/internal/agentmanager/dispatcher.go#L343) |
| **worker-service events handler** | Go HTTP handler | 接收 sandbox 上行事件批量 → 入 Redis + Pub/Sub | [events.go:63](services/worker-service/internal/handler/events.go#L63) |
| **sandbox**（容器） | E2B / Docker / AgentCore 内的 microVM | 跑 Python sigma-worker 进程，**对外只 HTTP** | — |
| **sigma-worker Python 主进程** | sandbox 内 | FastAPI server，端口 8080 | [main.py:1086](agents/sigma-worker/src/sigma_worker/main.py#L1086) |
| **sigma-worker `/commands` handler** | sandbox 内 | 去重 + 按 action 分流（input/interrupt/cancel） | [main.py:1011](agents/sigma-worker/src/sigma_worker/main.py#L1011), [transport_http.py:573](agents/sigma-worker/src/sigma_worker/transport_http.py#L573) |
| **sigma-worker `/invocations` handler** | sandbox 内 | 按 engine 选 adapter（claude 复用 `_persistent_adapter`） | [main.py:900](agents/sigma-worker/src/sigma_worker/main.py#L900), [driver.py:26](agents/sigma-worker/src/sigma_worker/driver.py#L26) |
| **ClaudeAdapter** | sandbox 内 | 调 Anthropic SDK 流式生成 + 工具调用 | [claude_adapter.py:223](agents/sigma-worker/src/sigma_worker/claude_adapter.py#L223) |
| **HTTPTransport 出站缓冲** | sandbox 内 | 攒 50/64KB/30ms 批量上行事件 | [transport_http.py:356](agents/sigma-worker/src/sigma_worker/transport_http.py#L356) |
| **sandbox-gateway** | Go 进程，K8s pod | 替 sandbox 加密钥 + 出网过滤 + 鉴权限流 | [main.go:273](services/sandbox-gateway/cmd/server/main.go#L273) |
| **Redis** | 独立实例 | 所有命令/事件/状态的中转站 | — |

---

## 3. 逐帧详解（spawn 阶段，T+0 ~ T+2335ms）

> 每帧固定格式：
>
> - **时间 / 主语 / 代码位置 / Redis 变化 / 为什么这么设计**

---

### 阶段 A：Master 决策（Frame 0-3）

#### Frame 0 — 浏览器发出 chat 帧

- **时间**：T+0
- **主语**：浏览器 JS
- **代码**：用户前端代码（sigmaweb / Flutter）
- **Redis**：无
- **动作**：WebSocket 帧 `{"type": "chat.input", "content": "帮我搜下今天北京的天气"}`
- **为什么是 WebSocket 不是 HTTP**：浏览器 ↔ agent-service 之间是**唯一**的长连接位置——因为浏览器需要实时收 token 流。worker ↔ sandbox 之间反而是 HTTP（详见 FAQ §5.6）。

#### Frame 1-3 — Master LLM 推理 = 意图判断

- **时间**：T+1 ~ T+300
- **主语**：agent-service Master AgentRunner
- **代码**：[services/agent-service/internal/agentrunner](services/agent-service/internal/)
- **Redis**：无
- **动作**：
  1. 收 WebSocket 帧，把用户消息塞进 Master 对话历史
  2. 调 Claude API（流式），Master 自己就是一个 Claude 实例
  3. Claude 返回 `tool_use: agent.spawn(title="天气查询", instruction="搜索北京天气...")`
- **关键认知**：**没有"意图判断模块"**，意图判断就是 Master 的 LLM 推理本身。Master 看到用户输入，自己决定要不要派活、调哪个工具。

---

### 阶段 B：worker spawn handler（Frame 4-6.5）

#### Frame 4 — HTTP POST 到 worker

- **时间**：T+301
- **主语**：agent-service tool registry
- **代码**：tool registry 解析 `agent.spawn` → 转 HTTP
- **请求**：`POST /api/v1/agents`，body `{session_id, title, instruction}`
- **Redis**：无
- **为什么走 HTTP 而不是直接函数调用**：agent-service 和 worker-service 是**两个独立 K8s 部署**，必须走网络。这也是为什么 worker 可以独立扩容/重启而不影响 master。

#### Frame 5 — spawnAgent handler 验证参数

- **时间**：T+302
- **主语**：worker-service HTTP handler
- **代码**：[services/worker-service/internal/handler/handler.go:102-132](services/worker-service/internal/handler/handler.go#L102-L132) (`spawnAgent`)
- **动作**：验证 session_id / instruction 非空，调 `manager.Spawn(...)` 拿 agent_id
- **Redis**：无

#### Frame 6 — manager.Spawn 写 meta 并启动 supervisor

- **时间**：T+303
- **主语**：worker-service AgentManager
- **代码**：[services/worker-service/internal/agentmanager/manager.go:683-837](services/worker-service/internal/agentmanager/manager.go#L683-L837)
- **动作**：
  1. 生成 agent_id（snowflake 格式的 hex string）
  2. 写 meta hash 到 Redis（status=spawning, runtime_epoch=N）
  3. 把 run 对象塞进 `m.agents` map（**进程内**索引，只是性能优化）
  4. **第 806 行**：`m.startSupervisor(run)` 启动 supervisor goroutine
- **Redis 变化**：
  ```
  HSET sigma:wkr:agent:<id>:meta
      status spawning
      runtime_epoch 1
      session_id <sid>
      ...
  ```
- **为什么 status 先标 spawning**：Master 可能立即又来下一条命令，看到 spawning 就知道"先等等"。

#### Frame 6.5 — HTTP 202 立即返回 master

- **时间**：T+304
- **主语**：worker-service handler
- **动作**：返回 `{agent_id: "abc123"}`，HTTP 202 Accepted
- **关键设计**：**handler 不等 sandbox 创建完！** sandbox 创建可能要 100ms~5s（取决于 E2B 是否要冷启动），handler 等不起。
- **Master 收到 agent_id 后**：把这个 ID 记下来，后续要 invoke / interrupt 都用这个 ID。但 master 不知道 sandbox 起没起来——它**不需要知道**，只管发命令进 Redis。

---

### 阶段 C：supervisor 协程异步起 sandbox（Frame 7-12）

> ⚠️ 从这里开始，时序"对不上"的根源就出现了：**handler 已经返回**，但下面这些事还在异步进行。Master 可能已经在等 sandbox 起来；如果 Master 这时候就发第二条命令，会被 enqueue 到 stream 里等 sandbox 起来后由 Dispatcher 一起派。

#### Frame 7 — supervisor 派 runAgent goroutine

- **时间**：T+305
- **主语**：worker-service supervisor
- **代码**：[manager.go:825](services/worker-service/internal/agentmanager/manager.go#L825) `go m.runAgent(...)`

#### Frame 8 — assembleBootConfig 组装启动配置

- **时间**：T+306
- **主语**：worker-service runAgent goroutine
- **代码**：[manager.go:1410-1425](services/worker-service/internal/agentmanager/manager.go#L1410-L1425) `m.assembleBootConfig`
- **bootConfig 核心字段**：
  ```
  agent_id       = "abc123"
  engine         = "" (空 → sandbox 内兜底成 "claude")
  gateway_url    = "https://sandbox-gateway.internal:8090"
  agent_token    = JWT(HS256 by agentauth lib)
  session_id     = "s1"
  callback_url   = "https://worker-service.internal:8084"
  system_prompt  = "<master 给的 instruction>"
  ```
- **为什么 engine 留空**：当前生产链路全部走 claude，master 不传 engine，这一层空字符串透传到 sandbox 内被 `body.get("engine") or "claude"` 兜底。架构上是预留扩展点。

#### Frame 9 — sandbox.Create（真正起容器）★ 时间大头 ★

- **时间**：T+307 ~ T+2000（**这一帧最久**，热启 100ms / 冷启 2-5s）
- **主语**：worker-service runAgent goroutine
- **代码**：[manager.go:1445](services/worker-service/internal/agentmanager/manager.go#L1445) `run.sandbox.Create(ctx, ...)`
- **依赖**：E2B SDK / Docker / AgentCore（三选一）
- **完成时**：拿到一个 sandbox session，包含 `Endpoint = "https://sb-xyz.e2b.app"`
- **为什么这里这么慢**：要拉镜像（首次）/起容器/起 Python 进程/Python import 重量级依赖。这就是为什么前面 handler 不能同步等。

#### Frame 10 — sandbox endpoint 写入 Redis meta

- **时间**：T+2001
- **代码**：[manager.go:1475-1478](services/worker-service/internal/agentmanager/manager.go#L1475-L1478)
- **Redis 变化**：
  ```
  HSET sigma:wkr:agent:abc123:meta
      sandbox_endpoint "https://sb-xyz.e2b.app"
      sandbox_phase    "running"
  ```
- **关键作用**：从这一刻起，**Dispatcher 协程才能读到 sandbox URL**。前面如果 Dispatcher 试图派单，会读到空 endpoint 然后等。

#### Frame 11 — InvokeStream 启动 sandbox 主流程

- **时间**：T+2002
- **代码**：[manager.go:1544](services/worker-service/internal/agentmanager/manager.go#L1544) `run.sandbox.InvokeStream(...)`
- **作用**：触发 sandbox 内的 Python 主进程进入"等命令"循环

#### Frame 12 — enqueueInitialStartCommand 投首条命令

- **时间**：T+2003
- **代码**：[manager.go:2705-2721](services/worker-service/internal/agentmanager/manager.go#L2705-L2721)
- **动作**：把 master 当初给的 instruction 包装成 `action=user_input` 命令，塞进 cmd stream

---

### 阶段 D：首发命令入队（Frame 13-15）

#### Frame 13 — enqueue_cmd.lua 原子执行 5 件事 ★ 重点 ★

- **时间**：T+2003（一次 Lua 调用 < 1ms）
- **主语**：Redis（Lua 脚本在 Redis 进程里跑）
- **代码**：[services/worker-service/internal/agentstore/lua/enqueue_cmd.lua](services/worker-service/internal/agentstore/lua/enqueue_cmd.lua)
- **5 件事**（按 Lua 文件行号）：
  1. **第 27-31 行**：检查 runtime_epoch（防止 sandbox 被换掉后旧命令乱投）
  2. **第 34-37 行**：`SISMEMBER cmd_dedup` 看 command_id 重不重
  3. **第 42 行**：`HINCRBY meta next_command_seq 1` 拿单调递增编号
  4. **第 64 行**：`XADD cmd stream * cmd <JSON> seq <N>` 入队
  5. **第 66-75 行**：`SADD cmd_dedup` + `HSET cmd:<id>` 写状态投影
- **Redis 变化**：
  ```
  XADD sigma:wkr:agent:abc123:cmd  *  cmd '{...}'  seq 1
  → 返回 entry ID 例如 1714720000300-0
  SADD sigma:wkr:agent:abc123:cmd_dedup cmd-uuid-1
  HSET sigma:wkr:agent:abc123:cmd:cmd-uuid-1 sid=... seq=1 envelope=...
  ```
- **为什么必须 Lua 原子**：5 步分开执行的话，假如第 2 步刚分配 seq=5，第 3 步还没 XADD，另一个并发 enqueue 拿到 seq=6 并先 XADD 完成，结果 stream 里 seq=6 在 seq=5 前面，Dispatcher 就看到乱序。Lua 让 5 步在 Redis 单线程里**不可分割**。

#### Frame 14 — Dispatcher goroutine 被唤醒

- **时间**：T+2004
- **主语**：worker-service Dispatcher 协程（每个活跃 agent 一个）
- **代码**：[dispatcher.go:343-389](services/worker-service/internal/agentmanager/dispatcher.go#L343-L389) `run()` 主循环
- **背景**：这个 goroutine 一直在跑：
  ```go
  for {
      drainOwnPending(...)   // 处理自己 pending list 里的（崩溃恢复用）
      claimStale(...)         // 抢别人挂掉的（30s 没 ACK 才抢）
      XReadGroup BLOCK 5s    // 阻塞等新消息
      → dispatchEntry(...)
  }
  ```
- **触发**：刚才 Frame 13 的 XADD 让 BLOCK 立即返回

#### Frame 15 — XReadGroup 拿到 entry

- **时间**：T+2004
- **代码**：[dispatcher.go:355-361](services/worker-service/internal/agentmanager/dispatcher.go#L355-L361)
- **Redis 行为**：
  ```
  XREADGROUP GROUP agent-dispatch CONSUMER worker-3:abc123
             STREAMS sigma:wkr:agent:abc123:cmd  >
  ```
  这条命令保证：**多个 worker 副本同时 XREADGROUP 同一 stream，每个 entry 只发给其中一个**。entry 进入这个 consumer 的 **pending list**（"已发出，等 ACK"）。
- **为什么用 Consumer Group**：如果不用，多个 worker 副本会重复派单。Consumer Group 是 Redis 内置的"分单去重"机制。

---

### 阶段 E：Dispatcher 派单（Frame 16-21）

#### Frame 16 — 读 meta 拿 sandbox endpoint

- **时间**：T+2005
- **代码**：[dispatcher.go:385+](services/worker-service/internal/agentmanager/dispatcher.go#L385) `dispatchEntry`
- **Redis 操作**：
  ```
  HGET sigma:wkr:agent:abc123:meta sandbox_endpoint
  → "https://sb-xyz.e2b.app"
  ```
- **关键**：Dispatcher **每次派单都重新读** sandbox_endpoint，不缓存在进程内存里。这是为什么 worker 副本可以随便重启——位置信息全在 Redis。

#### Frame 17 — POST sandbox /commands

- **时间**：T+2006
- **请求**：
  ```http
  POST https://sb-xyz.e2b.app/commands
  Content-Type: application/json
  {
    "command_id": "cmd-uuid-1",
    "seq": 1,
    "action": "user_input",
    "content": "搜索北京天气",
    "ref_id": null
  }
  ```
- **为什么 POST 而不是 WebSocket**：详见 §5.6 FAQ "为什么 Dual-POST"。

#### Frame 18-20 — sandbox 内 /commands 处理

- **时间**：T+2010~2011
- **代码**：
  - 路由：[main.py:1011](agents/sigma-worker/src/sigma_worker/main.py#L1011) `POST /commands`
  - 去重 + 分流：[transport_http.py:128-149](agents/sigma-worker/src/sigma_worker/transport_http.py#L128-L149) `_CommandDedup` + [transport_http.py:573-619](agents/sigma-worker/src/sigma_worker/transport_http.py#L573-L619) `handle_inbound_command`
- **三段动作**：
  1. **去重**：`_CommandDedup` 是 LRU OrderedDict（容量 1024）。同一 command_id 重复来，直接返回 `{accepted: True, duplicate: True}`，**不入主队列**。
  2. **分流**：按 action：
     - `input` → `_handle_input_invocation()` 把 InputEvent 放进 asyncio.Queue（**主队列**）
     - `interrupt` → asyncio.Event.set()（BaseRunner 监听这个 event，立即打断当前 LLM 流）
     - `cancel` → 同上 + 状态置 stopped
  3. **立即返回 200**：handler 不等执行，立即给 worker Dispatcher 回 200
- **为什么 handler 立即返回**：跟 spawn handler 一样的设计哲学——handler 只负责"接收 + 入队"，执行是消费协程的事。

#### Frame 21 — Dispatcher 收 200 → XACK

- **时间**：T+2012
- **代码**：[dispatcher.go:667](services/worker-service/internal/agentmanager/dispatcher.go#L667) `XAck`
- **Redis 操作**：
  ```
  XACK sigma:wkr:agent:abc123:cmd  agent-dispatch  1714720000300-0
  ```
- **效果**：这条 entry 从 pending list 移除，Redis 不再认为"未完成"。
- **失败处理**：如果 sandbox 返回非 2xx，Dispatcher 也会 XACK（已经到 sandbox 了，重发也没用），但会记日志告警。

---

### 阶段 F：sandbox 内消费 + adapter 选择（Frame 22-25）

#### Frame 22 — 主队列消费协程取 input

- **时间**：T+2012
- **代码**：[main.py:1033-1080](agents/sigma-worker/src/sigma_worker/main.py#L1033-L1080) `_handle_input_invocation`
- **动作**：从 asyncio.Queue 拿一个 InputEvent，准备调 `/invocations`
- **关键**：这里 **不是真的 HTTP 调** `/invocations`，是**直接函数调用**（同进程）。`/invocations` 路由本身也接受 HTTP，但 sandbox 内部走直调。

#### Frame 23 — /invocations 入口：engine 选择

- **时间**：T+2013
- **代码**：[main.py:900](agents/sigma-worker/src/sigma_worker/main.py#L900), [main.py:962](agents/sigma-worker/src/sigma_worker/main.py#L962)
- **核心代码**：
  ```python
  engine = body.get("engine", "claude")
  ```
- **重要事实**：master 传过来的 engine 是空字符串 / None → 兜底成 `"claude"`。当前生产链路**全部走 claude adapter**。

#### Frame 24 — adapter 工厂选实例

- **时间**：T+2013
- **代码**：[driver.py:26-46](agents/sigma-worker/src/sigma_worker/driver.py#L26-L46)
- **三个分支**：
  | engine 值 | adapter | 复用策略 |
  |---|---|---|
  | `"deepagents"` | `DeepAdapter` | 每次新建 |
  | `"pi"` | `PiAdapter` | 每次新建 |
  | 其他（默认 `"claude"`） | `ClaudeAdapter` | **复用 `_persistent_adapter`**，避免重启 Claude Code CLI |
- **`_persistent_adapter` 的关键**：
  - 是 sandbox 内的**全局变量**
  - 保留整个 Claude Code 的运行状态（MCP 连接、对话历史、工作目录）
  - 跨多次 `/invocations` 调用复用
  - 通过 `is_alive()` 检查存活
- **为什么 claude 要复用**：Claude Code CLI 启动一次要 200~500ms（加载工具、连 MCP）。每次新建会让用户每条消息都多等 500ms。

#### Frame 25 — adapter 调 Anthropic API（经 gateway）★ 重点 ★

- **时间**：T+2014
- **代码**：[main.py:485-523](agents/sigma-worker/src/sigma_worker/main.py#L485-L523) `_setup_gateway_env`
- **关键机制**：sandbox 启动时已经设了环境变量：
  ```
  ANTHROPIC_BASE_URL = "https://sandbox-gateway.internal:8090"
  ANTHROPIC_API_KEY  = <agent_token>  # JWT，不是真的 Anthropic key
  HTTPS_PROXY        = <gateway_proxy_url>
  ```
- **Claude SDK 行为**：SDK 看到 `ANTHROPIC_BASE_URL`，把 `POST /v1/messages` 发到 gateway 而不是 `api.anthropic.com`。

---

### 阶段 G：sandbox-gateway 替 sandbox 加密钥（Frame 26）

#### Frame 26 — gateway 鉴权 + 替换 key + 转发

- **时间**：T+2014~2015
- **主语**：sandbox-gateway 进程
- **代码**：
  - 入口：[main.go:273-306](services/sandbox-gateway/cmd/server/main.go#L273-L306)
  - LLM proxy：[llmproxy/handler.go:111-216](services/sandbox-gateway/internal/llmproxy/handler.go#L111-L216)
  - 鉴权：[auth/auth.go:79-118](services/sandbox-gateway/internal/auth/auth.go#L79-L118)
- **5 步骤**：
  1. **鉴权**：从 `Authorization: Bearer <agent_token>` 解出 agent_id，Redis 双向查询 token → AgentContext
  2. **限流**：Redis Lua 脚本检查 RPM（llm 默认 15 RPM）
  3. **路由选择**：根据 agent 的 BackendType（vertex / direct）
  4. **替换密钥**：
     - direct：把请求头 `x-api-key` 换成真正的 `ANTHROPIC_API_KEY`
     - vertex：换 OAuth2 token，URL 换到 Vertex AI 端点
  5. **转发**：流式代理上行 token
- **设计收益**：
  - sandbox 进程内**没有任何真实 API key**，即使被注入也偷不到
  - 切换 LLM 厂商（Anthropic ↔ Vertex）只改 gateway 配置，sandbox 代码不动
  - 出网审计、限流、配额都收敛在 gateway

---

### 阶段 H：事件回流（Frame 27-33）

#### Frame 27 — Anthropic 流式吐 token

- **时间**：T+2300（LLM 首 token 通常 200-500ms）
- **主语**：Anthropic API
- **响应**：SSE 流，每个事件可能是：
  - `content_block_delta` → 一段文本增量
  - `tool_use` → 工具调用请求

#### Frame 28 — sandbox 把 LLM 事件转 AgentEvent 入缓冲

- **时间**：T+2300
- **代码**：[claude_adapter.py:223+](agents/sigma-worker/src/sigma_worker/claude_adapter.py#L223), [transport_http.py:356-376](agents/sigma-worker/src/sigma_worker/transport_http.py#L356-L376) `send_event`
- **流程**：
  1. ClaudeAdapter 内的 MessageDispatcher 把 Anthropic SSE 事件转成统一的 `AgentEvent` 结构
  2. 调 `send_event()` 非阻塞入队 `_events` deque（容量 10000）
  3. 入队立即 `_flush_event.set()`，触发 flusher loop 看一眼是否到批量阈值

#### Frame 29 — 触发批量 flush

- **时间**：T+2330（30ms 到了）
- **代码**：[transport_http.py:378-404](agents/sigma-worker/src/sigma_worker/transport_http.py#L378-L404) `_flusher_loop`
- **触发条件**（任一满足）：
  - 累积 ≥ 50 条事件
  - 累积 ≥ 64 KB
  - 距上次发送 ≥ 30 ms
- **为什么要批量**：
  - LLM 一条流式响应可能有几百个 delta token
  - 一条一发的话，HTTP overhead 比 payload 还大
  - 30ms 是用户感知不到的延迟，但能合并 10~50 条事件
- **POST 客户端**：[transport_http.py:459-510](agents/sigma-worker/src/sigma_worker/transport_http.py#L459-L510)，httpx 调 `POST {WORKER_SERVICE_URL}/agents/{agent_id}/events`
- **失败重试**：1s → 2s → 4s → ... → 30s 指数退避

#### Frame 30 — worker events handler 收批量

- **时间**：T+2331
- **代码**：[events.go:63-153](services/worker-service/internal/handler/events.go#L63-L153) `Post`
- **动作**：
  1. 读 meta 拿 `expectedEpoch`（防止过期 sandbox 还在乱发）
  2. 调 `manager.IngestEvent(...)` 处理批量
- **代码**：[manager.go:1742-1834](services/worker-service/internal/agentmanager/manager.go#L1742-L1834) `IngestEvent`

#### Frame 31 — append_event.lua 原子入库

- **时间**：T+2332
- **代码**：[services/worker-service/internal/agentstore/lua/append_event.lua](services/worker-service/internal/agentstore/lua/append_event.lua)
- **Redis 操作**（按 Lua 行号）：
  - **第 41-44 行**：epoch 检查
  - **第 46-56 行**：event_seq 去重（防 sandbox 重传）
  - **第 58 行**：`XADD events stream` 持久化
  - **第 63 行**：写 `last_ingest_ts`（用作 sandbox heartbeat）
  - **第 69-80 行**：更新 `current_phase`（如 `running` → `awaiting_tool`）
- **Redis 变化**：
  ```
  XADD sigma:wkr:agent:abc123:events * type text delta "我" seq 1
  XADD sigma:wkr:agent:abc123:events * type text delta "来" seq 2
  ...
  ```
- **为什么用 Stream 而不是 List**：Stream 支持精确从某个 ID 开始重读，前端断线重连可以从最后一个 seq 续读。

#### Frame 32 — Pub/Sub 广播实时事件

- **时间**：T+2333
- **代码**：[manager.go:1808](services/worker-service/internal/agentmanager/manager.go#L1808) `broadcastEvent`
- **Redis 操作**：
  ```
  PUBLISH sigma.output.s1 '{...event JSON...}'
  ```
- **关键区别**：
  - `XADD events stream` 是**持久化日志**（断线后可重读）
  - `PUBLISH` 是**实时广播**（不持久化，离线订阅者收不到）
- **两条路并行**：Stream 保证不丢，Pub/Sub 保证实时。

#### Frame 33 — Master 订阅器收到 → 转发浏览器

- **时间**：T+2334
- **主语**：agent-service Master AgentRunner
- **代码**：agent-service 启动 session 时就 `SUBSCRIBE sigma.output.{session_id}`
- **动作**：每收到一条事件，包成 `{"type": "agent.event", "data": {...}}` 通过 WebSocket 推给浏览器

#### Frame 34 — 浏览器渲染

- **时间**：T+2335
- **主语**：浏览器 JS
- **动作**：解析 AgentEvent → 在聊天界面追加 token

---

## 4. 第二次请求：invoke 简化路径

> 用户继续输入"再详细点"。**不再创建 sandbox**，复用现有的。

| # | T(ms) | 谁在动 | 动作 | 跟 spawn 的差别 |
|---|---|---|---|---|
| 0 | T+0 | 浏览器 | WebSocket 帧 `chat.input` | 同 |
| 1 | T+1 | agent-service Master | Claude 推理 → `tool_use: agent.invoke(agent_id="abc123", instruction="再详细点")` | tool 变成 invoke |
| 2 | T+300 | agent-service | HTTP `POST /api/v1/agents/abc123/invoke` | **不是 /agents** |
| 3 | T+301 | worker-service | [commands.go:95-249](services/worker-service/internal/handler/commands.go#L95-L249) `Invoke` handler | 不创建 sandbox |
| 4 | T+302 | worker-service | 读 meta 检查 sandbox 还活着（liveness）| spawn 没这步 |
| 5 | T+302 | worker-service | 如果 phase=paused，唤醒 sandbox | spawn 时是新的 |
| 6 | T+303 | worker-service | `h.store.EnqueueCommand(...)` 走 enqueue_cmd.lua | **同一个 Lua 脚本！** |
| 7 | T+303 | Redis | XADD cmd stream + 5 件原子事 | 同 spawn |
| 8 | T+304 | worker-service | Dispatcher 抢到 → POST sandbox `/commands` | 同 |
| 9 | T+305 | sandbox | `/commands` 收 input → 入主队列 | 同 |
| 10 | T+306 | sandbox | 消费协程 → `/invocations` | 同 |
| 11 | T+307 | sandbox | engine=claude → **复用 `_persistent_adapter`** ★ | 不重建 adapter |
| 12 | T+308 | sandbox | adapter 在原对话上下文上加用户消息 → 调 Claude | Claude 看到完整上文 |
| ... | ... | ... | 后续事件回流同 spawn 阶段 H | 同 |

**关键差别**：
1. **不再过 supervisor**，不再创建 sandbox（省了 100ms~5s）
2. **不再 assembleBootConfig**（sandbox 已经在跑）
3. **`_persistent_adapter` 复用**让对话上下文连续
4. invoke handler 多了一步"唤醒 sandbox"（如果 sandbox 在 idle 中被自动暂停过）

---

## 5. 时序 FAQ（"对不上"的 6 个常见错觉）

### 5.1 spawn handler 立即返回，但 sandbox 还没创建好，第一条命令哪儿去了？

**答**：命令在 spawn 流程**最后**才 enqueue（Frame 12），那时 sandbox 已经创建完（Frame 9-11）且 endpoint 已经写进 Redis（Frame 10）。也就是说：

```
T+302 handler 返回 master  ──┐
                             ├─→ 这俩并行
T+307~2000 创建 sandbox ────┘
T+2001 写 endpoint
T+2003 enqueue 首条命令  ←─ 一定在 sandbox 起来之后
```

**Master 怎么办**：master 拿到 agent_id 后**不需要等 sandbox**，它可以立即做别的事。如果 master 这时候立刻又发 invoke，invoke handler 会读 meta 看到 sandbox_phase 还没到 running，要么等要么直接 enqueue 进 stream（命令在 stream 里排队等 dispatcher 派）。

### 5.2 Dispatcher 怎么知道 sandbox 的 URL？sandbox 还没起来时它怎么办？

**答**：每次派单**实时读 Redis** `HGET meta sandbox_endpoint`（Frame 16），**不缓存**。

- sandbox 还没起来 → endpoint 是空 → Dispatcher 看到空就**等**（XReadGroup 没拿到不会派单，等到 Frame 12 enqueue 才有命令）
- 实际上 Dispatcher 是 spawn 时就启动的，但因为还没有命令在 stream 里，它一直 BLOCK 在 XReadGroup 上

**worker 副本无关性**：worker-3 创建了 sandbox 并写了 endpoint，worker-7 重启后接管 dispatcher，**读同一个 Redis hash 拿到同一个 endpoint**——这就是"无状态"。

### 5.3 事件回流跟命令派发是异步的，浏览器看到的顺序是怎么保证的？

**答**：浏览器看到的顺序**只由事件回流的 seq 决定**，跟命令派发完全解耦。

- 用户发"问题 A" → 命令 A enqueue（seq=10）
- 命令 A 到 sandbox → Claude 生成 → 事件流 e1, e2, e3...
- 事件流的 event_id 在 sandbox 内单调递增
- 浏览器按 event_id 顺序渲染

如果用户在"问题 A"还没完时发"问题 B"：
- 命令 B enqueue（seq=11）排在 A 后面
- Dispatcher 派完 A 才派 B（同 stream 顺序保证）
- sandbox 主队列也是 FIFO，B 等 A 跑完才执行
- 浏览器先看到 A 的所有 token，再看到 B

**例外**：interrupt 是异步信号，绕过主队列直接 set Event，立即打断 A，但 A 已经吐出的 token 仍然回流到浏览器（只是 A 不再继续）。

### 5.4 interrupt 明明在主任务后面 enqueue，为什么能"立即"打断？

**答**：`/commands` handler 按 action **分流**：

```python
if action == "input":
    queue.put_nowait(event)        # 进主队列 FIFO
elif action == "interrupt":
    interrupt_event.set()          # 异步信号，BaseRunner 立即看到
elif action == "cancel":
    interrupt_event.set()
    state = "stopped"
```

interrupt 是 `asyncio.Event`，BaseRunner 主循环在每个 LLM stream 步骤之间都 check 一次，看到 set 就抛出 InterruptedError，立即停止当前生成。

**所以"先后入队"是 stream 层面的，但 sandbox 内分流后走不同管道，interrupt 不排队**。

### 5.5 `_persistent_adapter` 是怎么跨调用保活的？sandbox 进程会不会被销毁？

**答**：

- **`_persistent_adapter` 是 sandbox 内 Python 进程的全局变量**，只要 Python 进程活着就在。
- **Python 进程什么时候销毁**：
  - sandbox 容器被销毁（agent 关闭 / 超时 / E2B 配额回收）
  - Python 主进程崩溃
- **正常会话期间**：Python 进程一直跑，`_persistent_adapter` 一直在，跨多次 `/invocations` 调用都复用同一份。
- **`is_alive()` 兜底**：每次进 `/invocations` 都检查一下 adapter 是否还活着，挂了就重建。

**sandbox 容器有空闲超时**：长时间没命令，容器会被回收。下次有命令时 worker-service 会重建 sandbox（agent 状态从 paused → running 的过程）。但这种情况 adapter 必然要重建，`_persistent_adapter` 也跟着丢。

### 5.6 为什么不用 WebSocket，要拆成 Dual-POST？

**答**：见旧文档第十一章。简化版：

- **WebSocket 致命问题**：连接粘在某一个 worker 副本上。worker 重启 = 所有 sandbox 断连。
- **Dual-POST 优势**：
  - worker → sandbox（POST /commands）和 sandbox → worker（POST /events）是两条独立短连接
  - 每个 worker 副本无状态，任何副本都能收发
  - 重启随时可换，sandbox 完全不感知
- **代价**：每条消息多 2~5ms 网络往返
- **为什么 2~5ms 不重要**：LLM 首 token 就要 200-500ms，2ms 几乎不可见

---

## 6. 一张总图：所有帧的时间轴

```
T+0     ┃ 浏览器 WebSocket
T+1     ┃ Master agent-service 收帧
T+1     ┃ Master 调 Claude API
T+300   ┃         Claude 返回 tool_use(agent.spawn)
T+301   ┃ HTTP POST /api/v1/agents
T+302   ┃ worker-service spawnAgent handler
T+303   ┃   manager.Spawn 写 meta + 启动 supervisor
T+304   ┃ ←── HTTP 202 立刻返回 master ────
T+305   ┃ (异步) supervisor 派 runAgent goroutine
T+306   ┃         assembleBootConfig
T+307   ┃         sandbox.Create  ━━━━━━━━━━━━━━━━━━━━━━━━━┓
        ┃                                                  ┃
        ┃                              最慢的一步 (100ms~5s) ┃
        ┃                                                  ┃
T+2000  ┃         sandbox 容器起来  ━━━━━━━━━━━━━━━━━━━━━━━━┛
T+2001  ┃         compareAndSetRunFields: 写 sandbox_endpoint
T+2002  ┃         InvokeStream 启动 sandbox 主流程
T+2003  ┃         enqueueInitialStartCommand
T+2003  ┃ Redis  enqueue_cmd.lua: 5 件事原子完成
T+2004  ┃ Dispatcher XReadGroup 拿到 entry
T+2005  ┃         HGET meta sandbox_endpoint
T+2006  ┃         POST sandbox /commands
T+2010  ┃ sandbox /commands handler 去重 + 分流
T+2011  ┃         user_input 入主队列
T+2011  ┃ ←── HTTP 200 立刻返回 worker ────
T+2012  ┃ Dispatcher XACK
T+2012  ┃ sandbox  主队列消费 → /invocations
T+2013  ┃         engine="claude" → 复用 _persistent_adapter
T+2014  ┃         ClaudeAdapter 调 Anthropic（base url 改写到 gateway）
T+2015  ┃ gateway 鉴权 + 限流 + 替 API Key + 转发
T+2300  ┃ Anthropic 流式吐 token (LLM 首 token ~200-300ms)
T+2300  ┃ sandbox  AgentEvent 入缓冲
T+2330  ┃         flush 触发（30ms 到）→ POST /events 批量
T+2331  ┃ worker  events handler IngestEvent
T+2332  ┃ Redis   append_event.lua 持久化
T+2333  ┃         PUBLISH sigma.output.{sid}
T+2334  ┃ agent-service Master 订阅器收到
T+2335  ┃ 浏览器 显示 token

[之后 T+2300 ~ T+N 循环：token / tool_use / tool_result 不断回流]
```

---

## 7. 用这份文档自检学没学懂

读完后试着不看文档回答：

1. ✅ master 调 spawn 时，handler 是同步等 sandbox 起来再返回，还是立即返回？为什么？
2. ✅ Dispatcher 在 sandbox 还没起来时已经在跑吗？它在干啥？
3. ✅ enqueue_cmd.lua 要原子完成 5 件事，分开做会出什么问题？
4. ✅ Consumer Group 在多 worker 副本场景下解决了什么问题？
5. ✅ sandbox 内 `_persistent_adapter` 跨调用复用，复用了什么状态？
6. ✅ interrupt 命令和 user_input 命令都进同一个 stream，为什么 interrupt 能"立即"生效？
7. ✅ 事件批量 flush 的 3 个触发条件是什么？为什么要批量？
8. ✅ sandbox 内调 Claude API，URL 被改写到了哪里？密钥是谁加的？
9. ✅ Redis Stream 和 Pub/Sub 同时用，分别承担什么职责？
10. ✅ worker 副本随机重启，为什么不会丢命令、不会乱序、不会重复执行？

回答不出来的题对应去看哪几帧 / FAQ 哪一节，已在题号旁边的章节里标注好。
