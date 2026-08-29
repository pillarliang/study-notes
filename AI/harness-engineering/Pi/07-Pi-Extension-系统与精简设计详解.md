可以。理解 Pi 的扩展系统，最重要的是先纠正一个容易产生的误解：

> **Pi 的“精简”不是功能少，也不是整个项目代码少，而是 Agent 内层主循环承担的职责很少。**
>
> 具体产品能力没有消失，而是被推到了 `pi-coding-agent` 产品层、Extension、Skill、Provider、宿主和执行环境中。

你可以把 Pi 理解成一个**微内核式 Agent Harness**：里面保留稳定机制，外面通过扩展组合产品能力。

相关笔记可以对照：

- [[study-notes/AI/harness-engineering/Pi/official-doc/08-Extensions-扩展编写]]
- [[study-notes/AI/harness-engineering/Pi/01-Pi-自上而下整体架构]]
- [[study-notes/AI/harness-engineering/Pi/06-pi-设计艺术-全书精要]]

---

# 一、先看 Pi 到底把什么叫做“核心”

Pi 大体可以理解成四层：

```text
L4 交互与宿主
   TUI / Print / JSON / RPC / SDK
                ↓
L3 pi-coding-agent 产品运行层
   Session / Compaction / Tools / ResourceLoader / Extensions
                ↓
L2 pi-agent-core Agent 内核
   Agent + agentLoop + tool pipeline + AgentEvent
                ↓
L1 pi-ai 模型协议层
   Provider / Model / Message / Stream
                ↓
             外部 LLM
```

## 1. `pi-ai`：统一模型差异

这一层负责把 OpenAI、Anthropic、Google 等不同模型 API 收敛成统一协议：

```text
统一 Context
    ↓
Provider Adapter
    ↓
厂商 API
    ↓
统一 AssistantMessageEvent
```

上层不需要到处写：

```typescript
if (provider === "openai") {
  // ...
} else if (provider === "anthropic") {
  // ...
}
```

模型协议变化被限制在 provider adapter 里。

---

## 2. `pi-agent-core`：保持 Agent Loop 简单

内层真正的主循环，概念上接近下面这样：

```typescript
while (true) {
  const assistant = await callModel(context);
  context.messages.push(assistant);

  const toolCalls = getToolCalls(assistant);

  if (toolCalls.length === 0) {
    break;
  }

  const results = await executeTools(toolCalls);

  context.messages.push(...results);
}
```

真实实现还会处理：

- 流式输出；
- tool schema 校验；
- steering 和 follow-up 队列；
- AbortSignal；
- 工具并行执行；
- tool result 顺序；
- AgentEvent；
- 错误转换；
- terminate 规则。

但它仍然尽量不负责：

- TUI 怎么显示；
- 会话文件怎么存；
- 是否需要审批；
- 什么命令危险；
- 要不要有 plan mode；
- 如何加载 Skill；
- 如何发现项目规则；
- 如何压缩历史；
- 企业审计怎么做；
- 是否需要 GitHub、数据库或浏览器工具。

这就是“精简”的第一个核心含义：

> **Loop 只负责推进“模型 → 工具 → 模型”闭环，而不负责决定所有产品策略。**

---

## 3. `pi-coding-agent`：完整产品壳

真正让 Pi 成为日常可用 coding agent 的，是这一层：

- `AgentSession`
- `SessionManager`
- `ResourceLoader`
- Extension Runner
- Compaction
- 系统提示词装配
- 内置工具
- Retry
- TUI/RPC/SDK 接入

所以准确地说：

> Pi 不是一个功能简陋的玩具 Agent，而是“薄内核 + 完整产品层 + 可编程扩展面”。

---

# 二、Extension 系统在架构里处于什么位置

Extension 主要位于 `pi-coding-agent` 产品运行层。

它不是直接把代码塞进 `agentLoop`，而是在 Agent 运行过程中的一系列稳定检查点上注册行为。

可以把它想象成：

```text
Agent 主链
   │
   ├── 输入到达
   │      └── Extension input handlers
   │
   ├── Agent 开始
   │      └── Extension before_agent_start handlers
   │
   ├── 调用 Provider
   │      └── Extension provider handlers
   │
   ├── 模型请求工具
   │      └── Extension tool_call handlers
   │
   ├── 工具执行完成
   │      └── Extension tool_result handlers
   │
   ├── Turn 完成
   │      └── Extension turn_end handlers
   │
   └── Session 关闭
          └── Extension session_shutdown handlers
```

每个 Extension 只挂自己关心的节点。

例如：

- 权限扩展只监听 `tool_call`；
- 审计扩展监听 `tool_call`、`tool_result`；
- Prompt 扩展监听 `before_agent_start`；
- 成本监控监听 provider 和 turn 相关事件；
- GitHub 扩展注册几个 tool 和 slash command；
- 状态栏扩展监听 model、turn、token usage；
- 自定义模型扩展注册 provider。

于是主循环不需要知道这些产品功能的存在。

---

# 三、一个 Extension 是怎么被加载起来的

以你笔记记录的版本为准，一个 Extension 本质上是一个 TypeScript 模块：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // 在这里注册功能
}
```

加载过程可以拆成五步。

## 第一步：发现 Extension

Pi 会从几个来源发现扩展：

```text
~/.pi/agent/extensions/*.ts
~/.pi/agent/extensions/*/index.ts

项目目录：
.pi/extensions/*.ts
.pi/extensions/*/index.ts
```

也可以显式指定：

```bash
pi -e ./my-extension.ts
```

或者写进 `settings.json`：

```json
{
  "extensions": [
    "/path/to/my-extension.ts"
  ]
}
```

还可以通过 Pi package 分发：

```json
{
  "packages": [
    "npm:@foo/bar@1.0.0",
    "git:github.com/user/repo@v1"
  ]
}
```

---

## 第二步：用 jiti 直接加载 TypeScript

Pi 使用 `jiti` 加载 TypeScript 文件，所以普通 Extension 通常不需要自己先跑：

```bash
tsc
```

也不需要把它编译成 JavaScript 再安装。

这使得扩展开发体验很直接：

```text
写一个 .ts 文件
    ↓
放进 .pi/extensions/
    ↓
/reload
    ↓
立即生效
```

这也是“精简”的另一层含义：

> 扩展开发没有复杂的插件构建协议，普通 TypeScript 模块就是插件。

当然，复杂扩展仍然可以使用目录、`package.json`、npm 依赖和多文件结构。

---

## 第三步：执行默认导出的工厂函数

Pi 加载模块后，会调用默认导出的工厂函数：

```typescript
export default function (pi: ExtensionAPI) {
  // 注册阶段
}
```

如果它是异步函数，Pi 会等待初始化完成：

```typescript
export default async function (pi: ExtensionAPI) {
  const models = await fetchModels();

  pi.registerProvider("local", {
    // ...
  });
}
```

也就是说，Extension 初始化阶段可以：

- 读取配置；
- 查询本地服务；
- 动态获取模型列表；
- 初始化数据库连接；
- 注册工具；
- 注册事件 handler；
- 注册命令和 UI。

但要注意：它运行在 Pi 主进程里，拥有主进程的系统权限。

---

## 第四步：把能力注册进 Extension Runner

工厂函数拿到的 `pi`，可以理解成一个注册中心加运行时控制接口。

例如：

```typescript
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", handler);
  pi.registerTool(tool);
  pi.registerCommand("hello", command);
  pi.registerProvider("local", provider);
  pi.registerShortcut("ctrl+x", shortcut);
}
```

Extension Runner 会维护类似下面的注册信息：

```text
event handlers
registered tools
registered commands
registered providers
registered renderers
registered shortcuts
registered flags
```

主程序到了相应生命周期节点，再调用这些注册项。

因此它不是靠 Extension 修改 Pi 源码，而是靠：

> **稳定注册协议 + 稳定生命周期事件 + 运行时装配。**

---

## 第五步：运行时触发 handler

例如扩展注册了：

```typescript
pi.on("tool_call", async (event, ctx) => {
  // ...
});
```

当模型请求工具时，Pi 会在工具管道中的相应检查点调用它。

简化过程是：

```text
模型返回 tool call
    ↓
找到工具
    ↓
处理、校验参数
    ↓
触发 Extension tool_call
    ├── 放行
    ├── 阻止
    └── 修改参数
    ↓
执行工具
    ↓
触发 Extension tool_result
    ↓
生成 ToolResultMessage
    ↓
交回模型
```

一个典型的危险命令审批扩展是：

```typescript
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    if (
      event.toolName === "bash" &&
      event.input.command?.includes("rm -rf")
    ) {
      if (!ctx.hasUI) {
        return {
          block: true,
          reason: "Headless mode does not allow interactive approval",
        };
      }

      const ok = await ctx.ui.confirm(
        "危险命令",
        `是否允许执行：${event.input.command}?`
      );

      if (!ok) {
        return {
          block: true,
          reason: "User rejected the command",
        };
      }
    }
  });
}
```

这个功能完全不需要在 `agentLoop` 里增加：

```typescript
if (command.includes("rm -rf")) {
  // 弹窗
}
```

这正是 Extension 系统的主要价值。

---

# 四、Extension API 实际上提供了哪些“扩展面”

可以把它们分成六类。

# 1. 注册新的能力

## 注册 Tool

```typescript
pi.registerTool({
  name: "get_issue",
  label: "Get Issue",
  description: "Read an issue from the project tracker",
  parameters: Type.Object({
    id: Type.String(),
  }),

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    const issue = await loadIssue(params.id);

    return {
      content: [
        {
          type: "text",
          text: JSON.stringify(issue, null, 2),
        },
      ],
      details: {
        issueId: params.id,
      },
    };
  },
});
```

注册完成后，这个工具会进入 Agent 的 tool registry，也会作为工具 schema 提供给模型。

模型不需要知道它来自内置代码还是 Extension。

从 Agent Loop 的视角看，两者都是：

```typescript
interface AgentTool {
  name: string;
  description: string;
  parameters: Schema;
  execute(...): Promise<ToolResult>;
}
```

这是一种很重要的“同构扩展”：

> 扩展工具和内置工具走同一条执行管道，而不是走旁路。

因此它们都能获得：

- tool call 事件；
- 参数处理；
- AbortSignal；
- 流式更新；
- tool result；
- Session 持久化；
- TUI 渲染；
- 后续 LLM 继续推理。

Extension 甚至可以注册同名工具，覆盖 `read`、`bash`、`edit` 等内置工具。

---

## 注册 Provider

```typescript
pi.registerProvider("local-openai", {
  baseUrl: "http://localhost:1234/v1",
  apiKey: "LOCAL_OPENAI_API_KEY",
  api: "openai-completions",
  models: [
    // ...
  ],
});
```

这样可以接入：

- 本地模型服务；
- 企业内部模型网关；
- OpenAI-compatible 服务；
- 自定义鉴权；
- 动态模型目录。

它不是要求 Agent Loop 理解新 provider，而是把新 provider 接入已有的模型协议层。

---

## 注册 Slash Command

```typescript
pi.registerCommand("hello", {
  description: "Say hello",

  handler: async (args, ctx) => {
    ctx.ui.notify(`Hello ${args || "world"}!`, "info");
  },
});
```

输入：

```text
/hello Drew
```

会直接进入这个 command handler，而不是先交给 LLM。

Command 适合：

- 确定性操作；
- Session 管理；
- 配置切换；
- 显示状态；
- 执行不需要模型推理的工作。

---

# 2. 观察和干预生命周期

Extension 的第二种能力是监听事件。

事件大致覆盖了以下范围。

## Resource 生命周期

```text
resources_discover
```

可以贡献：

- Skill 路径；
- Prompt Template 路径；
- Theme 路径；
- 其他项目资源。

---

## Session 生命周期

```text
session_start
session_before_switch
session_before_fork
session_before_compact
session_before_tree
session_compact
session_tree
session_shutdown
```

这使扩展可以实现：

- Session 启动恢复状态；
- 切换前检查；
- 阻止某些 Session 操作；
- Compaction 前保存信息；
- Fork 后更新状态；
- 关闭前释放资源。

---

## Agent 生命周期

```text
before_agent_start
agent_start
turn_start
message_start
message_update
message_end
turn_end
agent_end
```

其中比较关键的是 `before_agent_start`。

它可以：

- 注入消息；
- 修改 system prompt；
- 根据 cwd 或项目状态增加规则；
- 根据模型改变提示策略；
- 为这次 run 加入动态上下文。

---

## 模型请求生命周期

```text
context
before_provider_request
after_provider_response
```

它们处在不同抽象层级：

- `context`：修改本次要给模型看的消息投影；
- `before_provider_request`：接触更接近 provider 的请求 payload；
- `after_provider_response`：查看 HTTP 状态、headers 等响应元数据。

这里要特别注意：

> `context` 修改的是本次模型调用看到的上下文投影，不一定修改持久 Session 历史。

这适合：

- 临时脱敏；
- 过滤某些消息；
- 注入即时环境信息；
- 针对不同模型转换上下文；
- 做缓存或追踪标记。

---

## Tool 生命周期

```text
tool_call
tool_result
tool_execution_start
tool_execution_update
tool_execution_end
```

`tool_call` 可以：

- block；
- 给出阻止原因；
- 修改参数；
- 弹出人工确认。

`tool_result` 可以：

- 截断超长结果；
- 删除敏感数据；
- 转换结果结构；
- 添加审计信息；
- 给错误补充可恢复建议。

多个 `tool_result` handler 会按加载顺序链式处理：

```text
原始结果
  ↓
Extension A 脱敏
  ↓
Extension B 截断
  ↓
Extension C 加审计元数据
  ↓
最终 ToolResultMessage
```

需要注意你笔记中特别标出的风险：

> `tool_call` 中修改 `event.input` 后，参数不会再次自动校验。

所以扩展改写参数时，必须自己保证类型和结构正确。否则会把不符合工具约定的数据直接送到 `execute()`。

---

# 3. 主动控制 Agent

Extension 不只是被动监听，它也可以主动向运行中的 Agent 注入消息。

例如：

```typescript
pi.sendMessage(message, {
  deliverAs: "steer",
});
```

主要投递方式可以理解成：

- `steer`：尽快影响当前进行中的工作；
- `followUp`：当前工作准备结束时继续追加任务；
- `nextTurn`：放到后续 turn。

这允许外部事件进入 Agent：

```text
文件发生变化
CI 运行结束
用户从另一个 UI 发来补充信息
后台任务完成
另一个扩展产生结果
```

然后 Extension 可以把消息注入 Agent，而不需要重写主循环。

---

# 4. 扩展 TUI

Extension 可以：

- 弹出选择框；
- 请求确认；
- 打开输入框；
- 打开编辑器；
- 显示通知；
- 设置状态栏；
- 设置 widget；
- 替换 footer；
- 修改标题；
- 注册快捷键；
- 注册自动补全；
- 自定义 Tool 渲染；
- 自定义消息渲染。

例如：

```typescript
if (ctx.hasUI) {
  const environment = await ctx.ui.select(
    "选择部署环境",
    ["development", "staging", "production"]
  );
}
```

这里 `ctx.hasUI` 很重要。

因为同一个 `AgentSession` 可能运行在：

- Interactive TUI；
- Print 模式；
- JSON 模式；
- RPC；
- SDK；
- Headless 自动化环境。

在 Headless 环境里，不应该默认弹出确认框并永远等待用户。

所以扩展通常要明确设计无 UI 策略：

```typescript
if (!ctx.hasUI) {
  return { block: true, reason: "Approval requires interactive UI" };
}
```

或者通过配置决定自动允许/拒绝。

---

# 5. 持久化状态

Pi 的 Session 不是简单的线性数组，而是一个带 `parentId` 的 JSONL entry tree。

这意味着 Session 可能出现：

```text
A → B → C
     ├── D → E
     └── F → G
```

用户可以：

- Fork；
- 回到旧节点；
- 切换分支；
- 做 Compaction；
- 恢复历史 Session。

所以扩展不能只把状态存在一个普通全局变量里：

```typescript
let approvedCount = 0;
```

因为切换分支后，这个值未必属于当前分支。

更符合 Pi 架构的做法是把状态写进 Session entry 或 tool result `details`：

```typescript
return {
  content: [{ type: "text", text: "Approved" }],
  details: {
    approved: true,
    policyVersion: 3,
  },
};
```

恢复时沿当前 branch 遍历：

```typescript
pi.on("session_start", async (_event, ctx) => {
  const branch = ctx.sessionManager.getBranch();

  for (const entry of branch) {
    // 根据当前路径上的 entry 重建扩展状态
  }
});
```

它的好处是：

- Fork 后状态自动分叉；
- 切换分支时不会串状态；
- Session 恢复后可以重建；
- Compaction 和历史记录仍能解释状态来源。

这其实是一种轻量的事件溯源思路：

> 不把隐藏的可变内存当成事实，而是让状态跟随 Session tree 的持久记录。

`pi.appendEntry()` 也可以保存扩展自己的 custom entry，而且不会自动进入 LLM context，适合保存：

- 审计信息；
- UI 状态；
- 外部任务 ID；
- 扩展版本；
- 缓存索引；
- 非模型上下文数据。

---

# 6. 扩展之间通信

Extension 之间可以通过事件总线通信：

```typescript
pi.events.on("my-extension:task-finished", handler);

pi.events.emit("my-extension:task-finished", {
  taskId: "123",
});
```

这样多个扩展可以松耦合协作：

```text
CI Extension
    ↓ 发出 ci:finished
Notification Extension
    ↓ 显示通知
Audit Extension
    ↓ 写入审计记录
Agent Bridge Extension
    ↓ 注入 follow-up 消息
```

它们不需要互相直接 import，也不需要进入 Agent Loop。

---

# 五、一次用户请求是怎么穿过扩展系统的

假设用户输入：

```text
帮我修改 README 并运行测试
```

完整链路可以简化为：

```text
1. 用户输入
   ↓
2. Extension slash command 检查
   ↓
3. Extension input 事件
   - 可以改写输入
   - 可以完全处理
   - 可以继续传递
   ↓
4. Skill 展开
   ↓
5. Prompt Template 展开
   ↓
6. 检查 Session / Compaction
   ↓
7. Extension before_agent_start
   - 注入消息
   - 修改 system prompt
   ↓
8. 构造本次模型 Context
   ↓
9. Extension context
   - 非破坏性调整本次 messages
   ↓
10. Extension before_provider_request
   ↓
11. Provider 调用 LLM
   ↓
12. message_start / update / end
   ↓
13. 模型产生 read 工具调用
   ↓
14. Extension tool_call
   - 放行、阻止或改写
   ↓
15. 执行 read
   ↓
16. Extension tool_result
   - 脱敏、截断、改写
   ↓
17. ToolResultMessage 写回上下文
   ↓
18. 模型继续产生 edit / bash
   ↓
19. 重复工具管道
   ↓
20. turn_end
   ↓
21. agent_end
```

你可以看到，Extension 几乎能接触整个产品生命周期，但它仍然没有要求 Agent Loop 为每个具体功能增加专门分支。

---

# 六、Pi 的“精简”具体精简在哪里

## 1. 精简的是内核责任，不是功能总量

Pi 没有试图让 `agentLoop` 同时负责：

```text
模型调用
工具权限
Session 存储
上下文压缩
TUI
Provider 认证
项目规则
Skill
审计
多 Agent 编排
后台任务
企业权限
```

它把这些问题分层处理。

因此“精简”应该理解为：

> 每层只拥有少数明确责任，特别是内层不会不断吸收产品功能。

---

## 2. 新功能通常不需要修改主循环

假设要新增“生产环境命令必须审批”。

一种不够克制的架构会修改 Tool Executor：

```typescript
if (environment === "production") {
  await showApprovalDialog();
}
```

以后又增加：

```typescript
if (commandTouchesDatabase) { ... }
if (userRole === "intern") { ... }
if (timeIsAfterHours) { ... }
if (repoIsProtected) { ... }
```

主执行链很快会堆满业务分支。

Pi 的做法是：

```typescript
pi.on("tool_call", productionApprovalPolicy);
pi.on("tool_call", databasePolicy);
pi.on("tool_call", rolePolicy);
pi.on("tool_call", protectedRepoPolicy);
```

Agent Loop 只知道：

```text
执行 beforeToolCall handlers
```

但不知道“生产环境”“实习生”“数据库”分别是什么。

---

## 3. 使用少数统一协议连接能力

Pi 反复复用几个稳定抽象：

```text
Message
AgentEvent
AgentTool
ToolResult
Provider
SessionEntry
Extension handler
```

例如所有工具，不管来自哪里，都走同一协议：

```text
内置 read
Extension 的 GitHub tool
Extension 的数据库 tool
远端执行 tool
覆盖版 bash
```

都进入同一条：

```text
tool call
  → 参数处理
  → hook
  → execute
  → result
  → message
```

这比为每一种工具类型建立一套独立运行机制更精简。

---

## 4. 机制和策略分离

Pi 核心提供的是机制：

```text
我可以在工具执行前调用 handler
我可以把 Session 写成树
我可以把 Provider 统一成流式事件
我可以注册一个新 Tool
我可以投递 steer/follow-up 消息
```

具体策略由产品或扩展决定：

```text
哪些命令必须审批
什么情况自动 Compaction
哪些文件不能修改
工具输出如何脱敏
哪些模型允许使用
是否允许无人值守执行
```

机制通常稳定，策略变化频繁。

把策略放进核心，会让核心不断变化；把策略放到 Extension，核心接口可以保持稳定。

---

## 5. 不把所有高级功能固化为内核概念

很多 Agent 产品会把下面这些都设计成核心模式：

- plan mode；
- sub-agent；
- MCP；
- permission popup；
- browser mode；
- workflow；
- review mode；
- cost monitor。

Pi 的倾向是：如果可以由以下机制表达，就先不增加新的核心状态机分支：

```text
Tool
Extension
Skill
Provider
Session entry
SDK/RPC 宿主
```

例如 sub-agent 可以被表达为：

```text
Extension 注册 subagent tool
    ↓
tool 内部启动另一个 AgentSession 或进程
    ↓
结果作为 ToolResult 返回
```

主 Agent Loop 不需要认识“sub-agent”这种特殊概念。

这里不是说 Pi 生态中不能有这些功能，而是：

> 它们不一定要成为所有用户、所有宿主都必须承担的内核语义。

---

## 6. 资源按需加载，减少上下文负担

Skill 也体现了同一种精简思想。

Pi 通常不会在启动时把所有 Skill 全文塞进 system prompt，而是只提供：

```text
name + description
```

任务匹配时，模型再读取对应 `SKILL.md`。

所以精简的不只是代码路径，还有模型上下文：

```text
启动时只加载索引
    ↓
匹配时按需读取正文
```

这避免系统提示词随着 Skill 数量线性膨胀。

---

## 7. 多个宿主复用同一个运行对象

Interactive、RPC、SDK、Print/JSON 不各自实现一遍 Agent Loop。

它们复用同一个产品运行对象和事件协议：

```text
TUI ─┐
RPC ─┼─→ AgentSession → Agent → agentLoop
SDK ─┤
JSON ┘
```

这也是架构精简：

> 宿主只是不同输入输出方式，而不是四套 Agent 实现。

---

# 七、但 Pi 并不是所有地方都“简单”

这一点很重要。

## 1. Extension API 本身并不小

从你笔记的 API 面积看，它已经覆盖：

- 工具；
- 命令；
- Provider；
- Session；
- 消息；
- Context；
- Provider payload；
- TUI；
- 快捷键；
- 渲染器；
- 持久化；
- Resource discovery；
- 事件总线。

所以不能说 Pi 的扩展系统“功能简陋”。

更准确地说：

> 扩展 API 很宽，但它被放在产品层，没有让 Agent Loop 本身变成一个巨型框架。

---

## 2. 复杂度没有消失，只是被重新放置

例如权限系统没有进入 Loop，并不表示权限问题不存在。

它被交给：

- Extension 作者；
- 产品装配者；
- OS sandbox；
- 容器或 microVM；
- 企业执行环境。

同样：

- Provider 兼容性转移到 `pi-ai`；
- Session 复杂度转移到 `SessionManager`；
- 长上下文复杂度转移到 Compaction；
- UI 复杂度转移到 TUI；
- 产品定制复杂度转移到 Extension。

所以 Pi 的理念不是“消灭复杂度”，而是：

> **把复杂度放到拥有它的那一层，防止所有复杂度汇聚到主循环。**

---

# 八、Extension 不是安全沙箱

Pi Extension 与 VS Code 那种相对隔离的插件宿主不能完全等同。

Extension 是同进程 TypeScript 代码，通常拥有：

- 文件系统权限；
- 网络权限；
- 环境变量；
- 子进程权限；
- 当前用户权限；
- Pi 进程内的可用资源。

也就是说，它理论上可以：

```typescript
import fs from "node:fs";

fs.readFileSync(process.env.HOME + "/.ssh/id_rsa");
```

Extension API 对它的约束主要是接口和生命周期约束，不是安全边界。

因此必须区分：

```text
tool_call hook ＝ 动作准入机制
OS sandbox     ＝ 物理权限限制
```

前者可以阻止正常工具管道中的危险调用，但不能限制恶意 Extension 直接调用 Node.js API。

所以：

> 只能安装可信 Extension；不可信代码应该放进独立进程、容器或受控执行环境。

这也是 Pi 选择“同进程扩展”的代价：

- 优点：调用简单、开销低、集成深；
- 缺点：没有真正的权限隔离。

---

# 九、什么时候用 Extension，什么时候不用

可以用下面这个判断顺序。

## 只是想教 Agent 怎么做一类任务

用 **Skill**。

例如：

- 如何做安全审计；
- 如何发布 npm package；
- 如何写研究报告；
- 如何排查数据库问题。

它是说明书，不需要执行运行时代码。

---

## 想复用一段用户请求

用 **Prompt Template**。

例如：

```text
检查当前分支的改动，按严重程度输出问题
```

它只是一段可复用输入。

---

## 想长期告诉 Agent 项目规则

用项目 Context File，例如 `AGENTS.md`。

例如：

- 测试命令；
- 代码规范；
- 禁止修改的目录；
- 架构约定。

---

## 想改变运行时行为

用 **Extension**。

例如：

- 新增 Tool；
- 拦截 Bash；
- 接入外部系统；
- 自定义 TUI；
- 改写 Context；
- 监听 Session；
- 注册 Provider；
- 做权限控制。

---

## 想分发一组能力

用 **Pi Package**。

Package 可以携带：

- Extension；
- Skill；
- Prompt Template；
- Theme。

Package 是分发容器，不是另一套运行时。

---

## 想从自己的应用中运行 Pi

用 **SDK** 或 **RPC**。

- Node/TypeScript 应用内直接嵌入，优先考虑 SDK；
- 非 Node 宿主或需要进程隔离，考虑 RPC。

---

# 十、用一个具体例子理解“薄核心、厚扩展”

假设你要做一个公司内部 Agent，要求：

1. 接内部模型网关；
2. 访问 Jira；
3. 修改生产配置前必须审批；
4. 所有工具结果脱敏；
5. 底部显示当前项目和 token 用量；
6. Session 恢复后继续关联 Jira 任务。

在 Pi 里可以这样组合：

```text
registerProvider
    → 接内部模型

registerTool
    → jira_get_issue
    → jira_add_comment

tool_call
    → 检查是否修改生产配置
    → 弹出审批

tool_result
    → 删除 token、密码、密钥

ctx.ui.setFooter / setWidget
    → 显示项目和 token

appendEntry / tool result details
    → 保存 Jira issue ID

session_start
    → 沿当前 branch 重建 Jira 上下文
```

整个过程中，`agentLoop` 仍然只做：

```text
请求模型
执行模型请求的工具
把结果交回模型
直到模型停止
```

这就是 Pi 所说的 minimal harness 最直观的体现。

---

# 十一、最后用三句话总结

第一句：

> **Pi 的扩展系统本质上是“同进程 TypeScript 模块 + 注册 API + 生命周期事件 + 统一运行协议”。**

第二句：

> **Pi 的精简不在于功能少，而在于 Agent Loop 不吸收产品策略；工具、审批、UI、Provider、Skill 和工作流尽量在外层组合。**

第三句：

> **这种设计没有消灭复杂度，而是把复杂度分层：内核获得稳定性和可嵌入性，扩展作者则承担安全、状态恢复、兼容性和产品策略的责任。**

所以，评价 Pi 是否“精简”，不要只看 Extension API 有多少，也不要只看仓库总行数；应该看：

```text
新增一个产品功能时，
是否必须修改 Agent 主循环？
```

在 Pi 里，大多数情况下答案是：

```text
不需要。
注册一个 Tool、handler、Provider、Skill 或宿主即可。
```

这才是它真正精简的地方。
