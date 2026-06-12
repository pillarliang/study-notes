# Sigma-Agent 完整数据流与架构（通俗版）

> 目标：让一个 Redis 小白也能看懂从"用户在浏览器说一句话"到"sandbox 跑完返回结果"的完整链路。
>
> 适用范围：sigma-agent 项目当前架构（2026-06 时点）。

---

## 一、整体角色（5 个核心服务）

```
┌────────────┐
│  浏览器     │  WebSocket
└──────┬─────┘
       │
       ▼
┌────────────────────┐
│  agent-service     │  Master Agent 大脑：意图判断、决定派活
│  (多副本)           │
└──┬──────────┬──────┘
   │          │  HTTP
   ▼          ▼
┌─────────┐ ┌────────────────────┐
│ context │ │  worker-service    │  调度中心：开工位、传命令、收结果（自己不执行）
│ service │ │  (多副本)           │
└─────────┘ └──┬─────────────────┘
                │  HTTP POST
                ▼
            ┌────────────────────────┐
            │  Sandbox（隔离容器）    │  Worker Agent 执行体
            │  • E2B / Docker /      │   - Python sigma-worker 进程
            │    AgentCore           │   - claude/deep/pi adapter
            └─────┬──────────────────┘
                  │  所有外网访问
                  ▼
            ┌────────────────────────┐
            │  sandbox-gateway       │  保安岗：替 sandbox 拿密钥代发
            │                        │   - LLM Proxy
            │                        │   - 出网过滤
            └────┬───────────────────┘
                 ▼
            外部世界（Claude API、S3、第三方 API）
```

### 1.1 先理清命名：一条指挥链上的三个角色

整套系统最绕人的地方只有一个——"worker"这个词被复用在了三个完全不同的东西上。在看任何数据流之前，先把这三者钉死，后面就不会再乱。

把系统想象成一个接活干活的工作室，从决策到落地是一条清晰的指挥链：

- **Master Agent**——大脑，住在 agent-service 里。直接对接用户，听懂诉求，判断"这件事要不要派出去、派给谁、是新开一摊还是接着之前那摊"。它只做决策，自己不动手。
- **worker-service**——**sandbox 的调度器：开一个 sandbox，让里面的 Worker Agent 干活**。它是一个常驻的 Go 后端服务（代码在 `services/worker-service/`），不关心任务内容，只运营三件事：开工位（创建 sandbox 容器）、把 Master 派下来的活传进去、把产出的结果收回来。流量大时可以多开副本。
- **Worker Agent**——真正干活的工人，对应代码里的 sigma-worker（一个 Python 程序，在 `agents/sigma-worker/`）。它住在 worker-service 给它开的工位（sandbox 容器）里，真正去调 LLM、调搜索等工具，一步步把任务做完。

**为什么偏偏卡在这里**：管理者叫 worker-**service**，执行者叫 Worker **Agent**，名字几乎一样，角色却正好相反——一个发号施令，一个埋头干活。只要记住"带 service 的是调度的、带 Agent 的是干活的"，混乱立刻解开。

为消除歧义，本文约定：单独出现"worker"一律指 worker-service（调度中心）；干活的执行体一律写全称 Worker Agent 或 sigma-worker。


| 名字                  | 是什么                                                 | 跑在哪                      | 角色                           |
| ------------------- | --------------------------------------------------- | ------------------------ | ---------------------------- |
| **Master Agent**    | agent-service 里的编排大脑                                | K8s Pod                  | 决策：派不派、派给谁                   |
| **worker-service**  | 一个 Go 后端服务，多副本（记作 worker-1 / worker-3 / w7…）        | K8s Pod                  | 调度：开工位、传命令、收结果，自己不执行         |
| **Worker Agent**    | sandbox 容器里的执行体，即 sigma-worker（Python 进程 + adapter） | E2B / Docker / AgentCore | 执行：真正跑 LLM、调工具               |
| **EKS worker node** | K8s 集群的机器节点                                         | ——                       | 只是承载上面那些 Pod 的物理机，与业务无关，看到忽略 |


**一句话**：Master 决定派活，worker-service 开工位并把活传进去，Worker Agent 在工位里真正干完。三者不在同一台机器上——worker-service 与 Worker Agent 之间靠 HTTP（那个 `https://sb-xyz.e2b.app` 地址）跨网络通信，全程没有任何中间代理进程。

### 1.2 服务跑在哪（部署拓扑）

```
K8s (EKS) 集群
  ├── agent-service Pod   (多副本)  ← Master 大脑
  ├── worker-service Pod  (多副本)  ← 调度引擎
  ├── context-service Pod
  ├── sandbox-gateway Pod           ← 出网保安
  └── Redis                         ← 公共中转站

E2B / AgentCore / Docker  (另一套基础设施，不在上面的集群里)
  └── sandbox 容器        ← Worker Agent 跑在这里
```

关键认知：**sandbox 不在 K8s 集群里**。worker-service 只是调云平台 SDK 远程开一个容器，所以它和 sandbox 之间的一切都得走 HTTP + Redis，不能靠本地内存或共享磁盘。这是全篇很多设计（地址记 Redis、Dual-POST、gateway 代发）的根本前提。

### 1.3 Redis 在本项目的角色（兼比喻词对照）

**Redis 是连接所有服务的中转站**：服务之间从不直接对话，一律往 Redis 里写、再从 Redis 里读来协作。它在本项目存三类东西：

- **状态**——每个 agent 的"档案卡"：它是谁、sandbox 在哪、进行到第几步。
- **队列**——待派发的命令、待消费的事件，两个"邮箱"：存下来不怕丢、可重读。
- **广播**——发给在线前端的实时事件流，一个"喇叭"：发完即焚，只有在线的人收得到。

这几类东西后文用比喻词指代，这里一次性绑定到真实 key：


| 比喻说法        | 真实 Redis 对象                   | 类型      |
| ----------- | ----------------------------- | ------- |
| 档案卡         | `sigma:wkr:agent:{id}:meta`   | Hash    |
| 任务邮箱        | `sigma:wkr:agent:{id}:cmd`    | Stream  |
| 事件邮箱        | `sigma:wkr:agent:{id}:events` | Stream  |
| 实时广播频道 / 喇叭 | `sigma.output.{session_id}`   | Pub/Sub |


---

## 二、Redis 速通（看懂后面的命令）

### 2.1 Redis 是什么

内存里的超大 key-value 仓库。所有数据存在 RAM 里，所以快得离谱（微秒级读写）。

### 2.2 Key 命名约定

Redis 的 key 就是一个字符串。冒号 `:` **没有任何语法含义**，纯粹是行业约定，方便人按层级阅读：

```
sigma:wkr:agent:abc123:cmd
└┬──┘ └┬─┘ └┬──┘ └┬──┘ └┬─┘
项目  服务  类型  ID    用途
```

读法："sigma 项目的 worker 服务下，agent abc123 的命令队列"。等价于文件路径 `/sigma/wkr/agent/abc123/cmd`。

### 2.3 数据类型


| 类型          | 像什么              | 常用命令                                   |
| ----------- | ---------------- | -------------------------------------- |
| String      | 单个值              | `SET k v` / `GET k`                    |
| Hash        | 一张表格（字段+值）       | `HSET k field value` / `HGET k field`  |
| Set         | 一个集合（去重）         | `SADD k member` / `SISMEMBER k member` |
| **Stream**  | **一个持久化队列 / 日志** | `XADD` / `XREADGROUP` / `XACK`         |
| **Pub/Sub** | **实时广播频道（不持久化）** | `PUBLISH` / `SUBSCRIBE`                |


### 2.4 Stream 是什么——通俗版

把 Stream 想象成**一个永不删除的群聊**：

- 谁都可以往群里发消息（XADD）
- 每条消息有一个**全局唯一的递增 ID**（entry ID，如 `1714720000000-0`，前半段是毫秒时间戳）
- 消息按发送顺序排好，永远不变
- 想读的人可以从任意位置开始读

### 2.5 三个 Stream 核心命令

**XADD** — 追加一条消息：

```
XADD sigma:wkr:agent:abc123:cmd  *  action user_input  content "..."
     └─────── 哪个 stream ──────┘  │  └──────── 字段+值 ──────────┘
                                   └─ * 表示让 Redis 自动分配 entry ID
```

**XREADGROUP** — 多个消费者协调读（分单）：

```
XREADGROUP GROUP agent-dispatch CONSUMER worker-1
           STREAMS sigma:wkr:agent:abc123:cmd  >
```

`>` = "给我没人读过的新消息"。Consumer Group 保证**同一条消息只发给组内一个消费者**。worker-1 拿到一条后，这条进入 **pending list**（"已发出，等 ACK"）。

**XACK** — 确认处理完毕：

```
XACK sigma:wkr:agent:abc123:cmd agent-dispatch 1714720000300-0
```

告诉 Redis"这条我处理完了"，从 pending list 移除。如果 worker-1 挂了没 ACK，30 秒后其他 worker 可以 `XCLAIM` 抢过来重试。

### 2.6 Stream vs Pub/Sub


|           | Stream     | Pub/Sub    |
| --------- | ---------- | ---------- |
| 持久化？      | ✅ 是        | ❌ 否（发完就丢）  |
| 离线订阅者能收到？ | ✅ 可重读      | ❌ 错过就错过    |
| 用途        | 可靠的命令/事件队列 | 实时广播给在线订阅者 |


事件**同时**走两条：Stream 是"账本"，Pub/Sub 是"实时喇叭"。

---

## 三、完整端到端数据流（用真实请求走一遍）

**场景**：用户在浏览器输入"帮我搜下今天北京的天气"，按下回车。

### 阶段 A：用户输入 → Master 决策

```
┌───────────┐  ① WebSocket 帧
│  浏览器    │  {"type": "chat.input", "content": "帮我搜下今天北京的天气"}
└──────┬────┘
       ▼
┌──────────────────────────────────────────────────────────────┐
│  agent-service (Master Agent Runner)                          │
│                                                              │
│  ② 收到 chat.input 事件                                       │
│  ③ 把消息塞进 Master 对话历史                                 │
│  ④ 调 Claude API                                              │
│  ⑤ Claude 返回 tool_use:                                      │
│      agent.spawn(title="天气查询", instruction="搜索北京天气...")│
│      └─ 这就是"意图判断"——LLM 在推理里完成                    │
└──────────────────┬───────────────────────────────────────────┘
                   │  HTTP POST /api/v1/agents
                   │  body: {session_id, title, instruction}
                   │  ⚠️ 无 engine 字段
                   ▼
```

**关键认知**：

- 没有独立的"意图判断层"——Master Agent 里的 LLM（Claude）推理本身就是意图判断
- Master 只能选"派 / 不派 / 调谁"，**完全不选执行引擎类型**

### 阶段 B：worker-service 创建 sandbox（只在 spawn 时）

```
┌────────────────────────────────────────────────────────┐
│  worker-service (LB 随机选了 worker-3)                  │
│                                                        │
│  ⑥ 立即:  生成 agent_id="abc123"，回 master            │
│                                                        │
│  ⑦ 异步:  调 E2B SDK 创建 sandbox  ★ sandbox 诞生 ★    │
│           拿到 URL: https://sb-xyz.e2b.app             │
│           把 URL 和 status=running 写进 abc123 档案卡   │
│                                                        │
│     首发: 往 abc123 的任务邮箱投第一封信               │
│           action=user_input, content=<instruction>     │
└──────────────────────────────┬─────────────────────────┘
                               ▼
```

**三件事就够了**：

1. **创建 sandbox，把它的 URL 记进档案卡**
  abc123 在 Redis 里有张"档案卡"（一个 Hash）。worker 调 E2B 拿到 sandbox 的 URL，写进 `sandbox_endpoint` 字段——后面 Dispatcher 派命令就靠它找地址。同时把 `status` 写成 `running`，让前端/监控能查到 agent 已就绪。
2. **把第一条命令投进任务邮箱**
  这封信只有两个关键字段：`action=user_input`（用户输入类，区别于 interrupt/cancel），`content=<instruction>`（就是 master 派活给的指令）。
3. **投信用一段 Lua 脚本一次做完**
  投信其实附带去重、分配序号等几步杂事，项目用一段 Lua 脚本让 Redis **原子**地一口气做完，避免并发下序号乱序。细节不重要，记住"投信是原子的"即可。

### 阶段 C：Dispatcher 派命令给 sandbox（所有命令都走这里）

```
┌────────────────────────────────────────────────────────────────────┐
│  Dispatcher goroutine (每个 worker 副本 × 每个活跃 agent 各一个)    │
│                                                                    │
│  while true:                                                       │
│    ⑧ 以"分单消费者"身份盯着 abc123 的任务邮箱                       │
│       （我是 agent-dispatch 组里的 w3:abc123 这个消费者）          │
│       拿一条还没人读过的新信; 没信就阻塞等                          │
│                                                                    │
│    ⑨ 从 abc123 的档案卡读出 sandbox_endpoint 字段                   │
│       → "https://sb-xyz.e2b.app"                                   │
│                                                                    │
│    ⑩ POST {sandbox_endpoint}/commands         ★ 发命令时刻 ★        │
│       body: {command_id, seq, action, content, ref_id}             │
│                                                                    │
│       三种命令进同一管道但分流处理:                                  │
│       ┌────────────────────────────────────────────────────┐       │
│       │ • action=user_input  → 用户的话（含 spawn 首条）    │       │
│       │ • action=interrupt   → 打断当前 LLM                │       │
│       │ • action=cancel      → 终止 agent                  │       │
│       └────────────────────────────────────────────────────┘       │
│                                                                    │
│    ⑪ 收到 sandbox 2xx → 告诉 Redis 这封信处理完了                   │
│       （从 pending list 移除，Redis 不再认为它"未完成"）            │
└────────────────────────────────────────────────────────────────────┘
```

**Consumer Group 的妙处**：10 个 worker 副本都订阅同一个 Stream，但 Redis 保证**同一条 entry 只发给一个 consumer**。所以同一 agent 的命令永远只有一个副本在派送，天然不会重复 POST。

### 阶段 D + E：Sandbox 收命令 + 选 adapter

```
┌────────────────────────────────────────────────────────────────────┐
│  Sandbox (E2B 上的隔离容器)                                         │
│                                                                    │
│  ⑫ POST /commands handler (sigma-worker Python 进程):              │
│       • 按 command_id 去重                                          │
│       • 按 action 分流:                                             │
│         ┌─ user_input  → 入主队列 (main_queue)                     │
│         ├─ interrupt   → asyncio.Event.set()  (立即生效)            │
│         └─ cancel      → asyncio.Event.set() + state=stopped       │
│       • 立即返回 200（handler 不阻塞等处理）                        │
│                                                                    │
│  ⑬ 主队列消费协程 (asyncio) 取出 user_input → 调内部 /invocations  │
│                                                                    │
│  ⑭ /invocations handler (main.py:962):                              │
│       engine = body.get("engine") or "claude"  ★ adapter 选择时刻 ★ │
│                                                                    │
│       if engine == "deepagents":                                   │
│           adapter = DeepAdapter(...)         ← 每次新建             │
│       elif engine == "pi":                                         │
│           adapter = PiAdapter()              ← 每次新建             │
│       else:  # claude                                              │
│           if _persistent_adapter 还活着:                           │
│               adapter = _persistent_adapter ← 复用!避免重启 CLI   │
│           else:                                                    │
│               adapter = ClaudeAdapter(...)                         │
│               _persistent_adapter = adapter                        │
└────────────────────────────────────────────────────────────────────┘
```

**关于 engine 的两个事实**：

- **当前生产链路所有 agent 都跑 claude**：master 根本不传 engine，一路空着传到 sandbox，最后兜底成 `claude`；deepagents/pi 是预留扩展点，没人用。
- **一个 sandbox 整个生命周期只用一种 engine**：要切 engine 得关掉 agent → spawn 新的。

### 阶段 F：Adapter 执行 + 流式 LLM 调用

```
┌────────────────────────────────────────────────────────────────────┐
│  ClaudeAdapter（或 DeepAdapter / PiAdapter）                        │
│                                                                    │
│  ⑮ 调 Anthropic Claude API（流式）                                  │
│      ⚠️ 路径: Adapter → sandbox-gateway → Anthropic                 │
│      (sandbox 内没有 API Key, gateway 替它加)                       │
│                                                                    │
│  ⑯ Claude 流式吐 token / tool_use:                                  │
│      "我"   → AgentEvent(type=text, delta)                          │
│      "来"   → AgentEvent(type=text, delta)                          │
│      tool_use(web_search, "北京天气") → AgentEvent(type=tool_use)   │
│      ...                                                           │
│      每个 event 进 OutboundEventQueue (内存)                        │
└────────────────────────────────────────────────────────────────────┘
                               │
                               │ 触发条件: 50 条 / 64KB / 30ms 任一满足
                               ▼
```

### 阶段 G：事件回流 → Redis → Master → 浏览器

```
┌────────────────────────────────────────────────────────────────────┐
│  ⑰ Sandbox 批量 POST /api/v1/agents/abc123/events                   │
│     body: {events: [50 条]}                                         │
└──────────────────────────────┬─────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  worker-service (LB 任意副本，比如 worker-7)                        │
│                                                                    │
│  ⑱ events handler 用原子脚本批量入库:                                │
│      • 检查 event_id 在去重集合里有没有（防重复）                   │
│      • 把事件追加到 abc123 的"事件邮箱"（持久化, 可回放）            │
│      • 更新档案卡里的 last_event_seq 字段                           │
│                                                                    │
│  ⑲ 同时往"实时广播频道"发一份:                                       │
│      广播到 session s1 的频道（sigma.output.s1）                    │
│      内容是事件的 JSON                                              │
│      （只有在线订阅者收得到, 不持久化）                              │
└──────────────────────────────┬─────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  agent-service (回到 Master Agent)                                  │
│                                                                    │
│  ⑳ Master AgentRunner 一开始就订阅了 session s1 的广播频道           │
│     每收到一条事件:                                                 │
│       - 视情况整合到 Master 对话历史                                │
│       - 通过 WebSocket 转发给浏览器                                 │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ WebSocket
                               │ {"type":"agent.event","data":{...}}
                               ▼
┌───────────┐  浏览器实时显示 token 一个一个吐出来:
│  浏览器    │  "我来帮你搜索..." → 显示 web_search 工具调用动画
│           │  → 显示搜索结果 → 显示总结
└───────────┘
```

---

## 四、Sandbox I/O 全景（黑盒视图）

> 前面阶段 B–G 把 sandbox 的输入输出散落在端到端流程里。这一章把 sandbox 当成黑盒，把它对外的所有通道收敛到一处。

### 4.1 原理：sandbox 对外只有 4 条通道

把 sandbox 看成一个隔离的进程容器：

- 跑在 E2B / Docker / AgentCore 上，对外**只暴露 HTTP 端点**
- **没有持久磁盘、没有共享内存**；进程内的状态（持久适配器、对话上下文）随容器消失
- **内部不持有任何外部 API 密钥**；所有出网必须经 sandbox-gateway 替它加密钥

由此推出 sandbox 的 I/O 只有 **4 条通道，2 入 2 出**：

```text
                  ┌────────────────────────────────┐
启动配置 ────────▶│                                │
(boot 一次性)     │           Sandbox              │
                  │                                │
命令请求 ────────▶│   • 主任务队列                 │
POST /commands    │   • 持久适配器实例             │
(运行时持续)      │   • 出站事件缓冲               │
                  │                                │
                  └──┬──────────────────────────┬──┘
                     │                          │
                     │ 事件回流(批量上行)        │ 外部 HTTP
                     │ POST /agents/.../events   │ (LLM / 工具 / 对象存储)
                     ▼                          ▼
                worker-service          sandbox-gateway
                                              │
                                              ▼
                                        外部世界
```

**这 4 条通道归成两条互不相干的链路**，分清它们就不会再混：


| 链路       | 谁跟谁                            | 走哪几条通道             | 干啥                      |
| -------- | ------------------------------ | ------------------ | ----------------------- |
| **控制链路** | worker-service ↔ sandbox       | 启动配置 + 命令下行 + 事件上行 | 派活、收结果（管调度）             |
| **出网链路** | sandbox → sandbox-gateway → 外部 | 外部 HTTP            | 调 LLM、调工具（gateway 代加密钥） |


两条链路**互不交叉**：worker-service 不碰出网（不经手密钥、不转发 LLM 响应），gateway 也不碰调度（不传命令、不收事件）。所以 sandbox 调 LLM 是它自己经 gateway 出去拿结果，worker-service 在这条路上没有任何角色。

### 4.2 入口 1：启动配置（boot，一次性）

- **触发时机** —— worker-service supervisor 协程异步创建 sandbox 时
- **传递方式** —— 通过 E2B SDK 在创建容器时注入（环境变量 / 启动参数）
- **频次** —— 整个 agent 生命周期**仅一次**

**核心字段**（worker-service 在创建 sandbox 时一次性塞进去，决定这个 Worker Agent 是什么样、能干什么）：

- **worker 类型 `engine`** —— `claude` / `deepagents` / `pi`，决定 sandbox 里用哪套 adapter 干活。一旦设定，整个 sandbox 生命周期不可变，想换得关掉重开。当前生产线全是 claude。
- **具体大模型 `model`** —— 如 `claude-sonnet-4-6`，决定 Worker Agent 调哪个 LLM。
- **系统提示词 `system_prompt`** —— 决定这个工人怎么思考、怎么干活（worker 的"人格/行为准则"）。
- **工具清单 `mcp_servers`** —— 能调哪些工具（搜索、文件等，外加自动塞进去的 artifact 工具）。
- **出网通道 `gateway_url` + `agent_token`** —— 所有外网请求改指向 sandbox-gateway，带 token 才放行（密钥不进 sandbox）。
- **身份/代际 `agent_id`、`session_id`、`runtime_epoch`** —— 标明是谁、哪次会话、第几代 runtime。

### 4.3 入口 2：命令请求（运行时持续）

- **HTTP 端点** —— `POST {sandbox_endpoint}/commands`
- **调用方** —— worker-service 的 Dispatcher 协程（每个 agent 一个）
- **中转** —— Redis Stream `sigma:wkr:agent:{id}:cmd`
- **响应模式** —— 立即返回 200（不阻塞等执行）

**请求体字段**：

- `**command_id`**（string）—— 命令唯一 ID，sandbox 据此去重
- `**seq`**（int）—— 全局递增序号，保证顺序
- `**action**`（enum）—— `user_input` / `interrupt` / `cancel`
- `**content**`（string）—— `action=user_input` 时携带的文本内容
- `**ref_id**`（string）—— 工具批准/拒绝场景下指向具体的 tool_use ID

**sandbox 内部分流逻辑**：

- `**user_input`** —— 入主任务队列，由消费协程取出调内部 `/invocations`
- `**interrupt`** —— 设置异步事件信号，立即打断当前 LLM 流式生成
- `**cancel**` —— 设置异步事件信号 + 把 sandbox 状态置为已停止

**Dispatcher 怎么找到"对应的" sandbox**：每个 agent 与一个 sandbox 一对一绑定，地址就存在它档案卡（`...:meta`）的 `sandbox_endpoint` 字段——创建时 worker 写进去，派单前 Dispatcher 读出来，关闭时清掉。

关键在于**地址只存 Redis、不进进程内存**：所以无论哪个 worker 副本抢到命令，读的都是同一张卡、拿到同一个地址。这就是"worker 无状态"的具体含义——sandbox 的位置不绑定在任何一个 worker 进程上。

### 4.4 出口 1：事件回流（批量上行）

- **HTTP 端点** —— `POST {worker-service}/api/v1/agents/{agent_id}/events`
- **调用方** —— sandbox 内的出站事件缓冲 flush 协程
- **模式** —— 批量上行（不是逐条）

**触发批量发送的条件**（任一满足即触发）：

- 累积 ≥ 50 条事件
- 累积 ≥ 64 KB
- 距离上次发送 ≥ 30 ms

**请求体结构**：`{ events: [event1, event2, ...] }`

**单个 event 的核心字段**：

- `**event_id`** —— 事件唯一 ID，worker 端用于去重
- `**type`** —— `text` / `tool_use` / `tool_result` / `status_change` 等
- `**delta**` —— type=text 时的增量文本
- `**tool_name` / `tool_input**` —— type=tool_use 时的工具调用信息
- `**seq**` —— 事件序号
- `**timestamp**` —— 产生时间

**worker 收到后的两路写入**：

1. **持久化**：追加到 Redis Stream `sigma:wkr:agent:{id}:events`（支持重放）
2. **实时广播**：PUBLISH 到 Pub/Sub 频道 `sigma.output.{session_id}`（在线订阅者实时收）

### 4.5 出口 2：外部 HTTP 调用（经 gateway 出网）

**核心原则：sandbox 永远拿不到 API key。** gateway 是个透明代理，夹在 sandbox 和外部服务中间，去程替它加 key、回程原样送回结果：

```
sandbox 要调 LLM
   │  发请求（只带 agent_token，不带 key）
   ▼
sandbox-gateway   ① 验 token   ② 加上真正的 API key   ③ 转发
   │
   ▼
api.anthropic.com
   │  返回生成结果，原路穿回 gateway（gateway 不改内容）
   ▼
sandbox  ← 收到的就是 LLM 生成结果本身（流式 token / tool_use），
            跟直连 Anthropic 拿到的一模一样
```

密钥从头到尾只在 gateway 一侧，sandbox 手里始终只有"结果"。

- **触发方** —— sandbox 内的 adapter 或工具实现
- **必经节点** —— sandbox-gateway（强制）
- **鉴权方式** —— sandbox 出网请求带 `agent_token`，gateway 验证后替换为目标服务密钥

**典型外部调用**：

- `**api.anthropic.com`** —— Claude API 流式生成，gateway 替它加 Anthropic API Key
- **Tavily / Serper / Exa** —— 搜索类工具调用，gateway 替它加第三方搜索 API Key
- **对象存储（S3 等）** —— 上传/下载文件，gateway 替它加云厂商临时凭证
- **各 MCP 服务端点** —— 工具协议调用，gateway 替它加对应 MCP 鉴权 token

**这种设计的收益**：

- 密钥泄露面**收敛到 gateway 一个组件**，sandbox 被攻破也无 key 可偷
- gateway 可以在中间统一做**出网过滤、速率限制、调用审计**
- 切换 LLM 厂商或工具厂商时，sandbox 代码不动，改 gateway 配置即可

### 4.6 不存在的通道（防混淆）

为了避免误解，列出 sandbox **没有** 的输入输出方式：

- **worker → sandbox 反向 WebSocket / gRPC stream** —— ❌ 用 Dual-POST 拆成两条短连接（见第十一章）
- **sandbox 直连 Redis** —— ❌ sandbox 与 Redis 之间隔着 worker-service，所有交互走 HTTP
- **共享文件系统 / 共享卷** —— ❌ 容器隔离，本地写文件随容器销毁
- **直接调外部 API（绕过 gateway）** —— ❌ sandbox 进程内无任何外部密钥，绕过即失败
- **Master Agent 直接调 sandbox** —— ❌ Master 只能通过 worker-service 间接派命令

### 4.7 一条请求里 4 条通道的交错时序

以"帮我搜下今天北京的天气"为例，标记每一步走的是哪条通道：

```text
T0  worker 创建 sandbox、注入启动配置             〔入口1 · boot 一次性〕
     │
T1  把用户指令派给 sandbox                        〔入口2 · 命令〕
     │
T2  sandbox 经 gateway 调 LLM                     〔出口2 · 出网〕
     │
T3  LLM 流式吐字，先进出站缓冲                     （暂存缓冲，不上行）
     │
T4  缓冲攒够 30ms，批量回传事件                    〔出口1 · 事件〕
     │
T5  LLM 要调搜索工具，事件继续上行                 〔出口1 · 事件〕
     │
T6  sandbox 经 gateway 调搜索服务                  〔出口2 · 出网〕
     │
T7  搜索结果作为工具结果上行                       〔出口1 · 事件〕
     │
T8  LLM 据结果生成总结，文字持续上行               〔出口1 · 事件〕
     │
T9  生成结束，sandbox 闲置等下一条命令             〔回到入口2 等待〕
```

整个过程 sandbox 只通过这 4 条通道与外界交互，没有任何额外耦合。这也是 sandbox 能跨 E2B / Docker / AgentCore 三种运行时复用同一份 Python 代码的根本原因。

---

## 五、Redis Key 总清单

这次请求执行完，Redis 里大概有这些 key：


| Key 模式                                  | 类型                  | 存什么                              | 谁写 / 谁读                                           |
| --------------------------------------- | ------------------- | -------------------------------- | ------------------------------------------------- |
| `sigma:wkr:agent:abc123:meta`           | Hash                | agent 状态、sandbox endpoint、当前 seq | worker-service 写，worker-service / agent-service 读 |
| `sigma:wkr:agent:abc123:cmd`            | **Stream**          | 待下发的命令队列                         | worker-service handler 写，Dispatcher 读             |
| `sigma:wkr:agent:abc123:cmd_dedup`      | Set                 | 已 enqueue 的 command_id（去重）       | enqueue_cmd.lua 维护                                |
| `sigma:wkr:agent:abc123:cmd:cmd-uuid-1` | Hash                | 单条命令的状态投影（供查询）                   | worker-service 写，master 查询用                       |
| `sigma:wkr:agent:abc123:events`         | **Stream**          | sandbox 上行的事件流                   | sandbox 经 HTTP 上行，worker-service 写入，持久化用          |
| `sigma.output.s1`                       | **Pub/Sub channel** | 实时事件广播（不持久化）                     | worker-service 发布，agent-service 订阅                |


---

## 六、命令派发时机全景

所有命令都经 Dispatcher → POST /commands，差别只在 sandbox 端怎么处理：


| 时机         | 触发动作                                          | 走的命令 action            | sandbox 端处理          |
| ---------- | --------------------------------------------- | ---------------------- | -------------------- |
| Spawn 时第一条 | master 调 `agent.spawn`                        | `user_input`           | 入主队列                 |
| 后续对话       | master 调 `agent.invoke`                       | `user_input`           | 入主队列                 |
| 批准/拒绝工具调用  | master 调 `agent.invoke(input_action=approve)` | `user_input`（带 ref_id） | 入主队列                 |
| 用户点停止      | master 调 `agent.interrupt`                    | `interrupt`            | 异步信号，立即生效            |
| agent 关闭   | master 调 cancel 或超时                           | `cancel`               | 异步信号 + state=stopped |


---

## 七、后续轮次（用户继续说话）的简化路径

第一次 spawn 之后，后续每条用户消息**不再创建新 sandbox**，而是 invoke 已有的：

```
浏览器输入"再详细点"
    ↓ WebSocket
agent-service Master LLM 决定:
  agent.invoke(agent_id="abc123", instruction="再详细点")
    ↓ HTTP POST /api/v1/agents/abc123/invoke
worker-service 用原子脚本投递命令:
  往 abc123 的任务邮箱投一封信:
    action = user_input
    content = "再详细点"
    ↓
Dispatcher 从邮箱里拿到这封信 → POST sandbox /commands
    ↓
sandbox 主队列消费 → /invocations → 复用 _persistent_adapter ★
    ↓
Claude 继续在原对话上下文吐 token（因为 _persistent_adapter 状态保留）
    ↓
事件回流（同前）
```

**关键**：`_persistent_adapter` 保留了**整个 Claude Code 适配器的运行状态**（包括 MCP 连接、对话历史、工作目录），所以"再详细点"能接上"今天北京的天气"的上文。

---

## 八、外部服务的使用方式

把"用第三方"分成 4 类：


| 类别                       | 含义                     | 项目里有吗                         |
| ------------------------ | ---------------------- | ----------------------------- |
| **A 类：LLM Provider**     | 调 API 拿文本生成            | ✅ Anthropic Claude            |
| **B 类：Tool Provider**    | 作为工具暴露给 LLM 调用         | ✅ Tavily/Serper/Exa 搜索、MCP 服务 |
| **C 类：Runtime Provider** | 提供 sandbox 容器，我们代码在里面跑 | ✅ E2B、AWS AgentCore、Docker    |
| **D 类：外部 Agent Service** | **把整个 agent 行为外包给第三方** | ❌ **没有**                      |


**关键区分**：AWS Bedrock AgentCore 在这里是用作 **sandbox 容器（C 类）**，**不是用作 AWS 的 Agent 服务（D 类）**。我们的 Python sigma-worker 跑在 AgentCore 的 microVM 里，agent 行为是项目自己实现的。

**如果未来要接入 OpenAI Assistants 这类 D 类服务**，路径是加一个新 engine + 新 adapter：

```python
def make_adapter(engine: str, *, agent_id: str):
    if engine == "deepagents": return DeepAdapter(...)
    if engine == "pi":         return PiAdapter()
    if engine == "openai":     return OpenAIAssistantAdapter(...)   # 假想
    return ClaudeAdapter(...)
```

`OpenAIAssistantAdapter` 只负责把外部 agent 的事件翻译成项目的 `AgentEvent` 格式回流。**目前没人实现这种 adapter。**

---

## 九、整条链路的核心设计哲学

记住三句话：

1. **服务之间不直接调用，通过 Redis 当邮箱**
  master 不直接发命令给 sandbox。它写 Redis Stream，worker Dispatcher 自己来取。这样 worker 任意副本可以重启、扩容、容灾，master 完全不感知。
2. **Stream 是持久化的可靠邮箱，Pub/Sub 是实时的喇叭**
  Stream 用于命令/事件这种不能丢的；Pub/Sub 用于实时推送给在线的人。两条路互补。
3. **所有"做了什么"都靠 Redis 记录，内存里的状态只是性能优化**
  worker 进程死了，Redis 里的 meta、stream、pending list 还在。新副本起来读 Redis 就接着干，这就是"无状态"的真正含义。

---

## 十、worker-service 重启时命令会丢吗？

短答：**不会丢，但可能延迟送达或短暂重复**。


| worker-service 死亡时机   | 命令在哪                        | 结果                                           |
| --------------------- | --------------------------- | -------------------------------------------- |
| handler 跑一半（极少）       | 还没进 Redis                   | master 收 HTTP 错误，自行重试                        |
| XADD 后、Dispatcher 取出前 | Redis Stream 里              | 任何副本下次取出即可（延迟 1-2s）                          |
| Dispatcher 取出、POST 前  | Pending list 里              | XCLAIM 30s 后接管重发                             |
| POST 进行中              | Pending list + 可能已到 sandbox | XCLAIM 重发，sandbox 按 command_id 去重，**不会重复执行** |
| POST 成功未 XACK         | Pending list + 已到 sandbox   | 同上                                           |
| SIGTERM 优雅关闭          | grace period 内处理完           | 几乎全部 XACK 完才退出，无感                            |


**三个不变量保证可靠性**：

1. 命令一旦写进 Redis Stream 就死不了（Redis 持久化 + Stream 是 append-only）
2. Pending list 不会"忘记"被取走未 ACK 的命令（Consumer Group 硬保证）
3. sandbox 按 command_id 去重（即便同一命令被重发 N 次，也只执行一次）

---

## 十一、为什么用 Dual-POST 而不是 WebSocket

**WebSocket 的致命问题**：连接粘在某一个 worker 副本上。worker 重启 = 所有 sandbox 断连；多副本扩容 = 要做 sticky 路由。

**Dual-POST**：双向拆成两条独立的短连接，每条都是无状态的：worker 任何副本都能接、任何副本都能发，挂掉随时换。代价是多 2ms 延迟，但 LLM 本身就要等 300-800ms，根本看不出来。

**类比**：

- WebSocket 就像跟客服建了私人微信，那个客服离职你就联系不上了
- Dual-POST 就像每次都打 400 客服电话，谁接都行，下次再打是另一个人也无所谓——因为你的工单号（Redis Stream）记着所有上下文

---

## 十二、命令速查表


| 你看到这种代码                                                         | 它在干什么                                   |
| --------------------------------------------------------------- | --------------------------------------- |
| `HSET sigma:wkr:agent:abc123:meta status running`               | 把 abc123 的 status 字段设为 running          |
| `HGET sigma:wkr:agent:abc123:meta sandbox_endpoint`             | 读出 abc123 的 sandbox URL                 |
| `HINCRBY ...:meta next_command_seq 1`                           | next_command_seq 字段自增 1 并返回新值           |
| `SADD ...:cmd_dedup cmd-uuid-1`                                 | 把 cmd-uuid-1 加进去重集合                     |
| `SISMEMBER ...:cmd_dedup cmd-uuid-1`                            | 检查 cmd-uuid-1 在不在（返回 0/1）               |
| `XADD ...:cmd * cmd "{...}" seq 1`                              | 往命令 stream 追加一条，字段 cmd=JSON，seq=1       |
| `XREADGROUP GROUP agent-dispatch CONSUMER w3 STREAMS ...:cmd >` | 我是 group agent-dispatch 里的 w3，给我没人读的新消息 |
| `XACK ...:cmd agent-dispatch 1714720000300-0`                   | 告诉 Redis 这条处理完了，从 pending list 移除       |
| `XCLAIM ...:cmd agent-dispatch w7 30000 1714720000300-0`        | 把这条 entry 抢过来归 w7（必须 idle > 30s）        |
| `PUBLISH sigma.output.s1 "{...}"`                               | 往 channel s1 广播一条                       |
| `SUBSCRIBE sigma.output.s1`                                     | 订阅这个 channel，有新消息就推给我                   |


---

## 十三、一句话总览

> Master Agent 决定"派活给谁"（spawn / invoke / interrupt），命令通过 Lua 原子脚本进 Redis Stream，worker Dispatcher 在所有副本里通过 Consumer Group 抢单，POST 给对应 sandbox 的 /commands。Sandbox 启动时 engine 就定了，每次收到命令时按 engine 找对应 adapter（claude 复用、deep/pi 新建），adapter 调 Claude API 流式生成（走 sandbox-gateway 拿密钥），token 反向以 batch POST 形式回流 Redis（Stream 持久化 + Pub/Sub 广播），Master 订阅 Pub/Sub 实时推给浏览器。整条链路没有任何"必须是某个副本来处理某个 agent"的强绑定，所以 worker 可以随便重启、随便扩缩容。

