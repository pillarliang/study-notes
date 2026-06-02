

## 第零站：启动阶段（流程图省略了这一段）

正式跟着七节点走之前，必须先讲一段流程图里没画的事：**启动阶段**。它在所有七步发生之前一次性完成，把后续循环要用的"运行环境"准备好。

### 它跟运行阶段的关系

```text
┌─────────────────────────────────────────┐
│ 启动阶段（流程图省略）                  │
├─────────────────────────────────────────┤
│ 1. 用户/SDK 调 createAgentSession()     │
│ 2. 创建 ResourceLoader                  │
│ 3. ResourceLoader.reload()              │
│    ├─ loadSkills          (skill 元信息) │
│    ├─ loadProjectContextFiles (AGENTS)  │
│    └─ buildSystemPrompt → 拼成大字符串  │
│ 4. 创建 Agent 实例                       │
│ 5. agent.state.systemPrompt = 上面那串  │
│ 6. 返回 AgentSession 句柄                │
└─────────────────────────────────────────┘
       ↓ AgentSession 被 CLI/TUI/SDK 存起来
       ↓ 等用户输入到达
┌─────────────────────────────────────────┐
│ 运行阶段(流程图 + 笔记后面要讲的)        │
├─────────────────────────────────────────┤
│ ─── 第一站：Product 入口条 ───          │
│ 7. 用户输入到达 → session.prompt(text)  │
│ ...                                      │
└─────────────────────────────────────────┘
```

启动阶段做的事**只发生一次**（除非显式 reload），结果在整个会话期间反复被用。

### 启动阶段干了三件事

#### 一、构造 systemPrompt

`buildSystemPrompt()` 把以下内容按顺序拼成一段大字符串，存到 `agent.state.systemPrompt`：

1. 基础提示词（自动发现的 `.pi/system.md` 或用户传入）
2. 工具列表 + 使用指南（按可用工具集动态生成）
3. `<project_context>` 块——包含项目根目录的 `AGENTS.md` / `CLAUDE.md` 内容
4. `<available_skills>` 块——所有可被 model 自主调用的 skill 元信息（name / description / location）
5. 追加段（自动发现的 `.pi/system-append.md`）
6. 当前日期 + 工作目录

整个会话期间 systemPrompt 通常不变。每轮调模型时，第四站组装 `Context` 直接从 `agent.state.systemPrompt` 取这一份。

#### 二、加载 skill 元信息（不读正文）

`loadSkills()` 扫描三类来源：

| 来源 | 路径 |
| --- | --- |
| 用户级 | `agentDir/skills/` |
| 项目级 | `cwd/.config/skills/` |
| 扩展提供 | 调用方显式传入 |

每个目录递归找 `SKILL.md`，**只解析 frontmatter 拿元信息**（name、description、disableModelInvocation），不读 skill 文件正文。

元信息渲染进 systemPrompt 的 `<available_skills>` 块。model 看到目录后按需调 `read` 工具去拿正文——这就是上一节讲的 "skill 自主发现路径"。

#### 三、加载项目上下文

`loadProjectContextFiles()` 自动发现并读取项目根目录的 `AGENTS.md` / `CLAUDE.md`，内容塞进 systemPrompt 的 `<project_context>` 块。这跟 skill 不同——项目上下文是**直接读全文嵌入**，因为它通常较短且每个 session 都需要。

### 为什么流程图不画这部分

流程图聚焦于**循环阶段**——用户输入到达后，从 Product 入口到模型返回再到工具执行的反复活跃过程。启动阶段是一次性的设置工作，画进循环图会让视觉重点失焦。

但理解整个系统不能漏掉这一段——否则 `systemPrompt`、`<available_skills>`、`<project_context>` 这些每轮都会送给模型的东西，会显得"凭空出现"。它们其实在启动阶段就准备好了。

### 一个直觉对照

| 类比 | 启动阶段 | 第一站及以后 |
| --- | --- | --- |
| 餐馆 | 装修、印菜单、招厨师、备食材 | 客人进门点菜、上菜 |
| 浏览器 WebSocket | `new WebSocket(url)` 建连接 | `ws.send(msg)` 发消息 |
| HTTP server | `app.listen(port)` 启动 | `req` 进来处理 |

`AgentSession` 必须**先被造出来**，才能**被调用**。

---

## 第一站：Product 入口条

最顶上那条 Product 入口层，列着 CLI、TUI、RPC daemon、SDK embed、Web/IDE UI——这五种入口看起来八竿子打不着，但它们都被一条注释收口：

> 所有入口都收敛到 `AgentSession.prompt(text)`，入口不感知 Loop 存在。

这句话其实埋了整张图最重要的设计哲学。一个 agent 系统的入口形态会无穷变化——明天加个微信机器人入口、后天加个 Slack bot——但底层那台引擎不能跟着变。所以中间一定要有一个"插座"，让任何入口插上来，行为一致。

这个插座就叫 **AgentSession**。它是 Runtime 层暴露出来的一个对象，长得很普通：调它的 `prompt(text)` 方法，告诉它要做什么，它就开始驱动后续整个循环。入口完全不需要知道有 Loop、有 Hook、有 convertToLlm——只需要传一段文本进去，订阅事件出来。

关于这个词后面会专门展开。先记住一件事：图里所有 Product 入口和所有内核环节之间，**只通过 AgentSession 这一座桥连接**。

---

## 第二站：进入循环 · 用户目标

文本一旦送进 AgentSession，就跨过 Product 边界，进入概念循环的第一步：用户目标。

这一步很短，几乎只是一个语义标记，表示"这次对话的起点"。它的工作不是处理输入，而是声明"从这里开始，所有环节都不再关心用户从哪进来的"。CLI 还是 Web，到了这里之后没有区别。

这种"丢掉入口信息"的动作是有意的。如果让循环知道"现在是 TUI 在调我"，循环就开始为 TUI 写特殊逻辑，下一个新入口接进来就要改循环。所以这一步的存在像一道门，门内是纯净的 agent 世界，门外是产品世界。

---

## 第三站：构造上下文

### 它在循环里被两条路径触发

图上有两条箭头指向这一步：

- **用户目标 → 构造上下文**：用户刚说了新内容
- **工具结果回写 → 构造上下文**：上一轮的 `toolResult` 刚被 append 完，要再调一次模型

不管哪条路进来，要解决的问题都是同一个。

### 本质

**把会话当前的消息全集准备好，交给下一步调模型。**

到这一步的时候，新消息已经被前一步 append 进 `agent.state.messages` 了（用户输入路径加了一条 user message；工具回写路径加了 assistant + toolResult）。这一步本身**不再追加消息**，只做三件事：

1. 拿当前完整的 `AgentMessage[]`（SessionManager 在内存里维护着）
2. **如果是从用户输入路径来的**，检查 token 是否超出上下文窗口——超了就先做 **Compaction**（把旧消息压成一段摘要 entry 追加进去，原始历史保留）。工具回写路径跳过这一步（同一轮已检查过）
3. 跑 `transformContext` hook，让扩展最后改一次消息列表（每轮都跑）

### 两条路径的差别一览

| 差别 | 用户输入路径 | 工具回写路径 |
| --- | --- | --- |
| 谁负责 append 新消息 | `AgentSession.prompt()` | 第八站工具结果回写 |
| 是否检查压缩 | 是 | 否 |
| 是否做输入预处理 | 是 | 不需要 |

"输入预处理"包括：slash 命令探测、`input` hook 改写、skill/template 展开。它们都在 `AgentSession.prompt()` 里完成，可以理解为构造上下文之前的开胃菜，不是这一步本身的工作。

### 输出

一份 `AgentMessage[]`，原样交给下一站，由 `convertToLlm` 塌缩成模型能看懂的格式。

---

## 第四站：请求大模型

消息列表准备好，下一步就是把它送进模型。这是全图最复杂的环节，因为这里要跨过两道边界：从 Kernel 跨到 pi-ai，再从 pi-ai 跨到远端 API。

请求大模型旁边那个大块的实现框里分了六步，前三步由 Kernel 干，后三步由 pi-ai 干。

**第一步：transformContext**。可选的 hook，给 Runtime 最后一次改消息的机会，仍然在 Kernel 类型体系里。

**第二步：convertToLlm**。这一步被特意拎出来用粗框标了★，因为它是整张图最关键的一道关卡。

为什么关键？因为 Runtime 喜欢往消息里塞自定义类型——`bashExecution` 记录了一次 bash 执行、`branchSummary` 是一段会话分支的摘要、`compactionSummary` 是压缩后的历史摘要。这些类型在 Kernel/Runtime 内部流通没问题，但模型不认识。模型只认识三种 role：user、assistant、toolResult。

所以总得有人把"花花绿绿的 AgentMessage"塌缩成"模型认识的 Message"。这就是 `convertToLlm` 的工作。它由 Runtime 通过 `config.convertToLlm` 注入给 Kernel，Kernel 只知道"调模型前一定要先跑一次这个函数"，但不知道里面具体怎么转。

这是一种典型的"控制反转"：Kernel 暴露一个插口，Runtime 把转换逻辑插进去。Kernel 因此保持纯粹，Runtime 想加新的自定义类型也只需要更新这个函数。

**第三步：组装 llmContext**。把转换后的 messages、系统提示、可用工具打包成 `Context` 对象，准备交给 pi-ai。

到这里 Kernel 的工作就结束了，下面三步交给 pi-ai 层。

**第四步：streamSimple 派发**。pi-ai 收到 `Context`，根据 `model.api` 字符串去 `api-registry` 查对应的 provider adapter。这是个简单但很重要的查表动作——它意味着"换个模型只需要换字符串"，业务代码不用动。

**第五步：Provider Adapter 归一化**。每家模型供应商的 API 都不一样：工具格式不一样、流式协议不一样、错误结构不一样、thinking 块怎么表达不一样、StopReason 名字不一样、Usage 怎么算也不一样。Adapter 的工作就是把这六件事翻译成 pi-ai 的统一格式。这一层做的事情看起来很 boring，但它是整套架构能"换模型不改代码"的根本原因。

**第六步：真正发请求**。HTTP/SSE 出去，流式响应回来。这里有一条铁律：**错误编码进 stream event，不 throw**。也就是说，无论模型 API 返回什么错误，Adapter 都把它转成一个 `error` 类型的事件塞进流里。这样上游 Kernel 永远能用同一种方式消费流，不需要写 try/catch。

请求大模型这一步最终的产物是一连串流式事件，每个 delta 都携带 `partial: AssistantMessage` 累积快照——UI 可以直接渲染这个快照而不用自己拼增量。

---

## 第五站：分叉判断

模型回完话，进入那个菱形判断节点，问一个问题——模型这次输出里有没有 toolCall？

判断逻辑简单到一行：

```ts
assistant.content.some(b => b.type === 'toolCall')
```

但这一行决定了循环的命运：

- 如果模型只输出了文本，没有工具请求，本轮结束，走返回用户的分支；
- 如果模型请求了工具，循环还得继续，走执行工具的分支。

这就是 agent 和普通 chatbot 的根本区别——agent 能根据模型自己的判断决定继续干活还是停下，而不是固定问一句答一句。

---

## 第六站：返回给用户

走文本分支后，进入返回用户这一步。它的工作是三层接力，每层做一件事。

Kernel 干第一棒：emit `turn_end` / `agent_end` 事件。Kernel 自己不知道事件给谁、怎么显示，它只负责"广播一声"。

Runtime 干第二棒：`AgentSession` 把事件转发给所有订阅者——TUI 要渲染、日志系统要落盘、扩展要观察、远程 RPC 客户端要推送。这就是前面说的"AgentSession 是事件分发中枢"的具体体现。

Runtime 同时干第三棒：`SessionManager.appendMessage` 把消息写进 JSONL。这里有个很重要的设计原则：**消息是事实来源，事件只是投影**。事件丢了可以重发，消息丢了就丢了。所以持久化的是消息，不是事件流。

最后 Product 干第四棒：TUI 或 Web UI 按事件流式渲染给用户看。

---

## 第七站：执行工具

如果分叉判断走工具分支，就进入执行工具这一步。这是三方协作——Kernel 编排、Runtime 加 Hook、本地副作用真正执行。

本地副作用在图上被单独标成一个角色，就是为了强调一件事：**工具执行是发生在本地的副作用**，跟 pi-ai 的"远端 HTTP 调用"是两种完全不同的边界。混淆这两者是新手最常犯的错误——pi-ai 是"出门跟模型说话"，tool.execute 是"在家干活"，把它们标成不同角色就是在提醒这件事。

执行工具的实现块里分了四步。

**第一步：校验**（Kernel）。按 name 找工具，用 TypeBox schema 校验参数。如果工具不存在或参数非法，Kernel **不抛异常**，而是直接造一个 `isError: true` 的 ToolResult 返回。这是不变量 I3 的体现：循环永远能跑下去。

**第二步：beforeToolCall hook**（Runtime）。这是企业可控性的关键钩子。Runtime 可以在这里拦截、改写、要求人工确认。常见用法：路径保护（禁止写 `/etc/`）、命令黑名单（禁止 `rm -rf`）、二次确认（每次写文件都问一下用户）。Hook 可以返回 allow、block、rewrite、confirm 四种结果。

**第三步：tool.execute**（本地副作用）。真正动手的地方。可能是读写本地文件、跑 shell 命令、调远端 MCP、甚至嵌套调用另一个 agent。注意一个细节：**pi-ai 层的 Tool 类型不带 `execute` 方法**，只有 Kernel 层的 `AgentTool` 才有。因为 pi-ai 只描述工具给模型看，不负责执行——执行是 Kernel 域的职责。

**第四步：afterToolCall**（Runtime）。执行完了再过一道 Hook，可以截断超长输出、脱敏敏感信息、甚至设置 `terminate=true` 强制结束本次 agent 运行。

整个执行工具流程贯穿一条铁律：**任何失败都转成 `isError=true` 的 ToolResult，进程绝不崩**。这是 agent 系统能长跑的基础。

---

## 第八站：工具结果回写

执行工具干完活，进入工具结果回写。这一步把工具的执行结果包装成消息，写回上下文。

第一步组装 `ToolResultMessage`，结构很简单：

```ts
{
  role: "toolResult",
  toolCallId,        // 对应哪次调用
  toolName,
  content,
  isError,
  timestamp
}
```

第二步 append 到 `context.messages`，emit `tool_execution_end`。这里有个容易忽视的细节：**回写顺序严格按 LLM 原始 toolCall 顺序**——如果模型一次返回了三个工具调用，结果就必须按 1、2、3 的顺序回写，不能因为某个工具跑得快就先写。否则模型会困惑。

第三步还是 `SessionManager.appendMessage`，把这条 toolResult 也持久化到 JSONL。

工具结果回写之后，整个循环画了一个大圈回到构造上下文：

> 新一轮：`messages += assistant + toolResult`

新一轮的构造上下文会看到比上一轮多出两条消息（上一轮模型的回复 + 这一轮的工具结果），然后再次走请求大模型 → 分叉判断的流程，直到某一轮模型不再请求工具，走返回用户的分支结束。

---

## 中场总结 · 循环走完了

到这里七步的旅程结束了。回头看一眼，整个循环干的事情其实非常朴素——**让模型不断"想 → 做 → 看结果 → 再想"**。但这个朴素的循环要在生产环境跑起来，就需要前面讲的所有工程封装：分层、Hook、归一化、持久化、错误转化、事件广播。

下面回头讲两个最重要的"边界"。它们不是节点，但理解它们比理解节点更重要。

---

## 边界一：模型边界（convertToLlm）

这是 Kernel 和 pi-ai 之间的边界，在图右下角的协议 ABI 块里被画成了一条塌缩条。

向下看：Kernel 用 `AgentMessage`——一种可扩展的超集类型，允许 Runtime 注册任意自定义消息。
向上看：pi-ai 用 `Message`——固定三种 role 的 ABI，模型唯一认识的格式。

这两种类型之间必须有一道"翻译"，就是 convertToLlm。塌缩条上写着"Kernel 层注入，每次调 model 前必跑"——这句话有两个重点：第一，由 Kernel 调度，所以位置稳定；第二，由 Runtime 注入，所以行为可扩展。

为什么不直接让 Kernel 用 `Message` 算了，省得转？因为那样 Runtime 就没法表达"这是一段被压缩的历史""这是一次 bash 执行的记录"了。失去了自定义类型，会话树、压缩摘要、扩展数据就全都没了存放的地方。所以宁可加一道翻译，也要保留 Kernel 的可扩展性。

这是工程上很常见的取舍：**在两个稳定层之间加一道转换函数，比硬把两边对齐成同一种类型代价更小**。

---

## 边界二：副作用边界（tool.execute）

这是 Kernel 和本地副作用之间的边界。

模型不能直接动文件、跑命令、调 API。模型只能"请求"工具——也就是输出一个 toolCall block。真正能动手的只有 `tool.execute`，而它被 Kernel 编排、Runtime Hook、schema 校验三层守门。

这个设计的核心目的不是性能，是**可控性**。企业想限制 agent 能干什么、能动哪些文件、要不要人工确认，全靠这条边界上的 Hook。如果让模型直接动手，这一切就都谈不上。

另一个常被忽视的好处是**可观测性**。所有副作用都走同一个出口，意味着想加审计日志、想接监控、想做回放，只需要在这个出口插一个 hook，不用满世界找散落的 `fs.writeFile`。

---

## 重头戏 · 再聊 AgentSession

讲完七步循环和两条边界，回头看 Product 入口条上方那行字：

> 所有入口都收敛到 `AgentSession.prompt(text)`

现在应该能看明白这句话的分量了。

### 它是什么

`AgentSession` 是 Runtime 层的一个对象，代表"一次正在进行中的 agent 对话"。可以类比成浏览器里的 `WebSocket`、数据库里的 `Connection`——都是"代表一段连接生命周期的把手"。

它本身不是消息、不是 Loop、不是状态机，而是一个**句柄**：上层产品拿着它发指令、订阅事件、查询状态。

### 它解决两件事

**第一件，把多种入口塌缩成一个调用面。**

CLI、TUI、RPC、SDK、Web 各种入口都只需要做一件事——`session.prompt(text)`。它们完全不需要知道 Kernel 长什么样、Loop 怎么转、convertToLlm 是干嘛的。Kernel 内部 API 改了，五种入口一行不用动。

设想没有 AgentSession 会怎样：每个入口都得自己装配 SessionManager、注册 ToolRegistry、订阅 AgentEvent、处理 abort 信号——同样的胶水代码写五遍，下次 Kernel API 升级要改五个地方。AgentSession 就是把这堆"每个入口都要写"的胶水代码收敛到一个对象里。

**第二件，当事件分发中枢。**

Kernel emit 一个事件，可能同时有 TUI 要渲染、日志系统要落盘、扩展要观察、远程 RPC 客户端要推送。让 Kernel 自己管订阅列表会污染它的纯粹性——Kernel 不应该知道"哦原来有个日志系统在听我"。AgentSession 把"订阅管理"这件麻烦事拢在 Runtime 层里：Kernel 只 emit 一次，AgentSession 广播给所有订阅者。

### 它跟 SessionManager 是什么关系

这俩名字像，但完全不一样：

- `AgentSession` 是**运行时句柄**，活的对象，代表"这次对话怎么跑"
- `SessionManager` 是**持久化管理器**，管 JSONL 文件，代表"消息怎么存"

一个 AgentSession 会用到 SessionManager（在返回用户和工具结果回写时写消息），但它们职责完全不同。可以这样理解：**AgentSession 是一次对话的"运行时实例"，SessionManager 是所有对话的"数据库"**。

### 它跟 Agent / Loop / Kernel / Runtime 是什么关系

这几个词容易被误以为是平级的，其实它们**不在同一个抽象层级**。先把每个词的本体说清楚：

| 名字 | 本体 |
| --- | --- |
| **Runtime** | 一个**包**（`pi-coding-agent`） |
| **Kernel** | 一个**包**（`pi-agent-core` / 源码里叫 `packages/agent`） |
| **`runAgentLoop`** | Kernel 包里的一个**纯函数**——喂一份消息 + config + emit 进去，跑完返回新消息，自己不持有状态 |
| **`Agent`** | Kernel 包里的一个**类**——状态机/句柄，持有 state、listeners、queue、所有 hook 字段，method 内部调 `runAgentLoop()` |
| **`AgentSession`** | Runtime 包里的一个**类**——包住一个 `Agent` 实例，给上层提供友好 API |

所以"Runtime / Kernel / Loop 三个并列"这种说法严格讲是错的。真正的依赖关系是：**Runtime 用 Kernel，Kernel 里包含 Loop**。

调用链从外到里：

```text
Product
  ↓ session.prompt(text)
AgentSession (Runtime 包)        ← 处理用户输入、扩展、压缩
  ↓ agent.continue()
Agent (Kernel 包的类)             ← 持有状态、hook、队列
  ↓ runAgentLoop(prompts, ctx, config, emit)
runAgentLoop (Kernel 包的函数)    ← 纯函数循环:调模型 → 执行工具 → 写回
  ↓ streamFn(...)
pi-ai                             ← 跨网络跟模型说话
```

可以用汽车类比把它们一一对上：

| 概念 | 类比 |
| --- | --- |
| `runAgentLoop` | 发动机（纯机械，喂油转动） |
| `Agent` | 整车（发动机 + 变速箱 + 仪表盘 + 油门，作为整体出厂） |
| `AgentSession` | 出租车前台（接客户、调度司机、记账） |
| Runtime | 出租车公司（前台 + 整车 + 调度规则） |
| Kernel | 汽车厂（出整车 + 出发动机零件） |

关键洞察：**依赖方向永远向下**。发动机不知道车是出租还是私家，整车不知道哪家公司在用它。同理——`runAgentLoop` 不知道 `Agent` 类的存在，`Agent` 不知道 `AgentSession` 包它，Kernel 整体不知道 Runtime 长啥样。

为什么要拆这么多层？每层对应一个不同的变化频率：

- `runAgentLoop` 几乎不变（agent 循环逻辑本身稳定）
- `Agent` 偶尔变（增减 hook 字段时）
- `AgentSession` 较常变（加新入口、加新扩展事件）
- Runtime 经常变（工具、压缩策略、扩展系统持续迭代）

如果不分层，改一个工具就要动到循环，整套系统没法维护。

---

## 收尾 · 一张图，四个视角

整张架构图浓缩了四个视角。读图的时候在脑子里轮流切换这四个视角，整张图就活了：

- **循环视角**：这次对话该做什么？回答的是流程问题。
- **分层视角**：每一步谁来做？回答的是分工问题。
- **边界视角**：跨层的时候要小心什么？回答的是约束问题。
- **AgentSession 视角**：上层产品怎么用？回答的是接口问题。

把这四个视角分别想清楚，再去看实际代码（无论是 Pi 还是任何类似的 agent 框架），就不容易被表面的复杂度劝退。所有 agent 框架本质都在解同一个问题——**怎么让一个无状态的模型在有状态的世界里持续工作**——只是切层方式略有不同。Pi 给出的答案是"七步循环 + 三层分工 + 两条边界 + AgentSession 收口"。

读完这张图，下一步可以继续看 [[22-从零到一搭建-Agent-完整技术文档]]，那里有更细的代码骨架；或者读 [[06-Sessions-会话树]]，把"会话为什么是树而不是数组"这条线索单独拉出来理解。
