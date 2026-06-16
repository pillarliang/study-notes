# LangChain Agent 开发指南（create_agent 与 Deep Agents）

> 基于 LangChain v1.x 官方文档（截至 2026.03）。
> 最新版本：langchain v1.1.0 (2025.11) / langgraph v1.x / deepagents v0.4 (2026.02)。
>
> 本指南从底层积木讲到上层成品：`create_agent` 是 LangChain v1.0 推出的生产级 Agent 构建函数（取代旧的 `langgraph.prebuilt.create_react_agent`），核心机制是 **Middleware**；`create_deep_agent` 则是在 `create_agent` 之上预装了一整套 Middleware、文件系统、子 Agent 与默认提示的开箱即用封装。两者共享同一套底层概念，掌握 `create_agent` 后，Deep Agents 只是"已经替你配好"。

![[langchain-agent-map.png|720]]

---

## 1. 定位与选型

### 1.1 三层抽象

LangChain Agent 体系是自底向上叠加的三层，上层复用下层的全部能力：

```
┌──────────────────────────────────────────────────────┐
│  create_deep_agent  —— 成品：预装 Middleware 栈 +       │
│    文件系统 + Skills + Memory + 默认系统提示            │
│  ┌────────────────────────────────────────────────┐   │
│  │  create_agent —— 积木：Middleware（核心）+ State │   │
│  │    + Tool + 循环控制 + 结构化输出 + Runtime      │   │
│  │  ┌──────────────────────────────────────────┐   │   │
│  │  │  LangGraph Runtime —— 底座：               │   │   │
│  │  │    持久化执行 / 流式输出 / Graph 循环       │   │   │
│  │  └──────────────────────────────────────────┘   │   │
│  └────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

`create_deep_agent` 本质上等于 `create_agent` + 一组预配置。需要完全掌控行为，用 `create_agent`；想快速搭出功能丰富的 Agent，用 `create_deep_agent`。

### 1.2 选型建议

| 需求 | 选择 |
| --- | --- |
| 简单 Agent，需要细粒度控制 | `create_agent` |
| 完全自定义的工作流（非标准循环） | 直接用 LangGraph |
| 复杂多步骤任务（规划、文件系统、子 Agent、记忆）开箱即用 | `create_deep_agent` |

### 1.3 从 create_react_agent 迁移

| 变更项 | create_react_agent（旧） | create_agent（新） |
| --- | --- | --- |
| **导入路径** | `langgraph.prebuilt` | `langchain.agents` |
| **系统提示** | `prompt` 参数 | `system_prompt` 参数，支持动态 prompt（middleware） |
| **Pre-model hook** | 自定义 | middleware `before_model` |
| **Post-model hook** | 自定义 | middleware `after_model` |
| **自定义 State** | TypedDict | TypedDict，可通过 `state_schema` 或 middleware 定义 |
| **模型切换** | 预绑定模型 | middleware 动态选择，不支持预绑定 |
| **工具错误处理** | 内置 | middleware `wrap_tool_call` |
| **Structured output** | `(prompt, Schema)` 元组 | `ToolStrategy` / `ProviderStrategy` |
| **Streaming node 名** | `"agent"` | `"model"` |
| **运行时上下文** | `config["configurable"]` | `context` 参数依赖注入 |

```python
# 旧写法
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(model="gpt-4o", tools=tools, prompt="...")

# 新写法
from langchain.agents import create_agent
agent = create_agent(model="gpt-4o", tools=tools, system_prompt="...")
```

---

## 2. Middleware：create_agent 的核心机制

Middleware 在 Agent 执行的不同阶段插入自定义逻辑，是 `create_agent` 区别于旧 API 的核心特性。Deep Agents 的几乎所有能力（规划、压缩、子 Agent、审批）都是预装的 Middleware（见 §8.3）。

### 2.1 Hooks

| Hook | 执行时机 | 用途 |
| --- | --- | --- |
| `before_agent` | Agent 开始运行前 | 加载记忆、验证输入 |
| `before_model` | 每次 LLM 调用前 | 更新 prompt、裁剪消息 |
| `wrap_model_call` | 包裹 LLM 调用 | 拦截和修改请求/响应 |
| `wrap_tool_call` | 包裹 Tool 调用 | 拦截和修改工具执行 |
| `after_model` | 每次 LLM 响应后 | 验证输出、应用 guardrails |
| `after_agent` | Agent 完成后 | 保存结果、清理资源 |

### 2.2 动态 Prompt

通过 middleware 实现基于上下文的动态系统提示，可读取 `runtime.context`（静态配置）或 `runtime.store`（持久化数据）：

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.agents.middleware import dynamic_prompt, ModelRequest

@dataclass
class Context:
    user_role: str
    deployment_env: str

@dynamic_prompt
def context_aware_prompt(request: ModelRequest) -> str:
    user_role = request.runtime.context.user_role
    env = request.runtime.context.deployment_env
    message_count = len(request.messages)

    base = "You are a helpful assistant."
    if user_role == "admin":
        base += "\nYou have admin access."
    if env == "production":
        base += "\nBe extra careful with data modifications."
    if message_count > 10:
        base += "\nThis is a long conversation - be extra concise."
    return base

agent = create_agent(
    model="gpt-4.1",
    tools=[...],
    middleware=[context_aware_prompt],
    context_schema=Context,
)
```

从 Store 读取用户偏好：

```python
@dynamic_prompt
def store_aware_prompt(request: ModelRequest) -> str:
    user_id = request.runtime.context.user_id
    store = request.runtime.store
    user_prefs = store.get(("preferences",), user_id)

    base = "You are a helpful assistant."
    if user_prefs:
        style = user_prefs.value.get("communication_style", "balanced")
        base += f"\nUser prefers {style} responses."
    return base
```

### 2.3 内置 Middleware

#### TodoListMiddleware

为 Agent 提供任务规划和追踪能力。自动注入 `write_todos` 工具，无需配置参数，适用于复杂多步骤任务。

```python
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(model="gpt-4o", tools=[...], middleware=[TodoListMiddleware()])
```

#### SummarizationMiddleware

自动压缩对话历史，防止超出上下文窗口。

```python
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model="gpt-4o-mini",          # 用于生成摘要的模型
            trigger=("tokens", 170000),   # 触发条件
            keep=("messages", 6),         # 保留策略
        ),
    ],
)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `model` | str \| BaseChatModel | 用于生成摘要的模型 |
| `trigger` | ContextSize | 触发摘要的条件 |
| `keep` | ContextSize | 摘要后保留多少内容 |

`ContextSize` 三种格式：

```python
("tokens", 170000)    # 基于 token 数量
("messages", 20)      # 基于消息数量
("fraction", 0.8)     # 基于模型上下文窗口比例
```

> [!warning] 废弃参数
> `max_tokens_before_summary` 和 `messages_to_keep` 已废弃，改用 `trigger` 和 `keep`。

> [!note] v0.4 变更（2026.02）
> Summarization 改为在 model node 中通过 `wrap_model_call` 触发，**保留完整消息历史**于 graph state 中，token 计数更准确。当模型抛出 `ContextOverflowError` 时会自动触发（当前支持 `langchain-anthropic` 和 `langchain-openai`）。此外提供 **Summarization Tool Middleware**，允许 Agent 在合适时机主动触发摘要，而非固定 token 阈值。

#### PatchToolCallsMiddleware

处理悬挂的 tool calls（deepagents 包提供）。当 AI 发出 `tool_call` 但没有对应的 ToolMessage 响应时（用户中断、超时等），消息历史会不完整，导致后续报错。

```python
from deepagents.middleware.patch_tool_calls import PatchToolCallsMiddleware

agent = create_agent(model="gpt-4o", tools=[...], middleware=[PatchToolCallsMiddleware()])
```

工作原理：在 `before_agent` 阶段遍历所有消息，检查每个 AI 消息的 `tool_calls`，若找不到匹配 `tool_call_id` 的 ToolMessage，自动补一条"已取消"占位。

```
AI Message (tool_calls: [{id: "abc"}])
    → 查找后续是否有 tool_call_id == "abc" 的 ToolMessage
    → 没找到 → 插入: ToolMessage("Tool call xxx was cancelled...")
```

#### SubAgentMiddleware

通过 `task` 工具实现主 Agent 向子 Agent 委派任务，核心价值是**上下文隔离（Context Quarantine）**：子 Agent 在独立上下文中执行，完成后只返回单条结果消息，大量中间步骤不污染主 Agent 的上下文窗口。

```
Main Agent
  │
  ├─ task(description="分析文件...", subagent_type="research")
  │     └─ SubAgent（独立上下文，独立工具集）
  │          ├─ Tool 1 → ...
  │          ├─ Tool 2 → ...
  │          └─ 返回最终结果（单条 ToolMessage）
  │
  └─ 继续主流程（上下文保持干净）
```

**基本原理**：SubAgentMiddleware 初始化时做两件事——① 注入 `task` 工具，其 description 中列出所有可用子 Agent 的 `name` + `description` 供 LLM 选择；② 每次 `wrap_model_call` 时把 `TASK_SYSTEM_PROMPT`（委派指南）追加到主 Agent 的 system message 末尾。LLM 据此自行决定何时调用 `task(description, subagent_type)`。

**参数**：

```python
SubAgentMiddleware(
    default_model=model,              # 子 Agent 默认模型
    default_tools=[],                 # general-purpose 子 Agent 的默认工具
    subagents=[...],                  # 自定义子 Agent 列表
    default_middleware=[              # 所有子 Agent 共享的默认中间件
        PatchToolCallsMiddleware(),
    ],
    general_purpose_agent=True,       # 是否包含通用子 Agent（默认 True）
    task_description=None,            # 自定义 task 工具描述（支持 {available_agents} 占位符）
)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `default_model` | str \| BaseChatModel | 子 Agent 使用的默认模型 |
| `default_tools` | Sequence[BaseTool] | 通用子 Agent 的默认工具集 |
| `default_middleware` | list[AgentMiddleware] \| None | 应用于所有子 Agent 的中间件 |
| `subagents` | list[SubAgent \| CompiledSubAgent] | 自定义子 Agent 列表 |
| `general_purpose_agent` | bool | 是否自动创建通用子 Agent（默认 True） |
| `task_description` | str \| None | 自定义 task 工具的描述文本 |

**定义子 Agent 的两种方式**：

```python
# 方式 1：字典定义（SubAgent TypedDict）—— 简单场景，中间件自动创建 Agent
{
    "name": "weather",
    "description": "获取城市天气的子 Agent",
    "system_prompt": "使用 get_weather 工具获取天气信息",
    "tools": [get_weather],
    "model": "gpt-4.1",       # 可选：覆盖默认模型
    "middleware": [],          # 可选：额外中间件
}

# 方式 2：预编译定义（CompiledSubAgent）—— 复杂场景，传入自定义 runnable
from deepagents.middleware.subagents import CompiledSubAgent
from langchain.agents import create_agent

subagent_runnable = create_agent(
    model=model,
    system_prompt="你是 Map-Reduce 文档合成器...",
    tools=[extract_sections, merge_sections, assemble_document],
    middleware=[PatchToolCallsMiddleware()],
    state_schema=SummaryState,         # 与主 Agent 共享同一 State Schema
)

map_reduce_subagent = CompiledSubAgent(
    name="map-reduce-synthesizer",
    description="处理大内容的 Map-Reduce 子 Agent...",
    runnable=subagent_runnable,
)
```

`runnable` 也可以是自定义编译后的 LangGraph。

> [!important] CompiledSubAgent 约束
> runnable 的 state schema **必须包含 `messages` key**。子 Agent 完成后，`messages` 列表的最后一条消息会被提取为 ToolMessage 返回给主 Agent。

**State 共享机制**：子 Agent 与主 Agent 通过共享 State 传数据，但排除 `messages`、`todos`、`structured_response` 三个 key。

```python
# 委派时：主 Agent 的 state（排除上述 key）传给子 Agent
subagent_state = {k: v for k, v in runtime.state.items() if k not in _EXCLUDED_STATE_KEYS}
subagent_state["messages"] = [HumanMessage(content=description)]

# 返回时：子 Agent 的 state 更新（排除上述 key）合并回主 Agent，messages 只保留最后一条
state_update = {k: v for k, v in result.items() if k not in _EXCLUDED_STATE_KEYS}
```

这意味着：主 Agent 写入的字段子 Agent 可读；子 Agent 写入的字段主 Agent 在其完成后可读；但子 Agent 的完整多轮对话历史**不会**回传，只返回最终一条消息。

**`general_purpose_agent` 的取舍**：

| 值 | 行为 |
| --- | --- |
| `True`（默认） | 自动创建通用子 Agent，拥有与主 Agent 相同的 tools，主要用于上下文隔离 |
| `False` | 只用 `subagents` 中定义的自定义子 Agent，避免 LLM 自由委派 |

> [!tip] 子 Agent 不被调用时的排查
> ① 让子 Agent 的 `description` 更具体，明确它擅长什么；② 在主 Agent 的 `system_prompt` 中强调"复杂任务用 `task()` 委派，以保持上下文干净并提高质量"。

**使用 SubAgent 的优势**：

| 优势 | 说明 |
| --- | --- |
| 上下文隔离 | 子 Agent 的中间 tool 调用不污染主 Agent 上下文 |
| 并行执行 | 多个子 Agent 可并发运行 |
| 专业化 | 子 Agent 可有不同的工具集、模型和配置 |
| Token 效率 | 大量中间步骤压缩为一条结果消息 |
| 工具隔离 | 子 Agent 独有的工具不在主 Agent 工具列表中，避免误调用 |

#### HumanInTheLoopMiddleware

为敏感工具调用添加人工审批，基于 LangGraph 的 `interrupt` 机制。**必须配合 checkpointer 使用**。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command

agent = create_agent(
    model="gpt-4.1",
    tools=[search_tool, send_email_tool, delete_database_tool],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": True,                                 # 需要审批
                "delete_database": {
                    "description": "Please review before deleting",
                    "allowed_decisions": ["approve", "reject"],     # 不允许编辑
                },
                "search": False,                                    # 自动通过
            }
        ),
    ],
    checkpointer=InMemorySaver(),
)

config = {"configurable": {"thread_id": "some_id"}}

# Agent 会在敏感工具前暂停
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Send an email to the team"}]},
    config=config,
)

# 恢复执行
result = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config,
)
```

| 决策 | 说明 |
| --- | --- |
| `approve` | 按原样执行 |
| `edit` | 修改参数后执行 |
| `reject` | 拒绝执行，附带反馈 |

工具内也可以直接调用 `interrupt()` 主动暂停，等待人工输入：

```python
from langgraph.types import interrupt

@tool
def request_approval(action_description: str) -> str:
    """请求人工审批。"""
    approval = interrupt({
        "type": "approval_request",
        "action": action_description,
        "message": f"Please approve or reject: {action_description}",
    })
    if approval.get("approved"):
        return f"Action '{action_description}' was APPROVED."
    return f"Action '{action_description}' was REJECTED. Reason: {approval.get('reason')}"
```

#### LLMToolSelectorMiddleware

用一个更小的模型智能筛选相关工具，适用于工具数量 10+ 的场景，减少 token 消耗、提升专注度。

```python
from langchain.agents.middleware import LLMToolSelectorMiddleware

agent = create_agent(
    model="gpt-4.1",
    tools=[tool1, tool2, tool3, tool4, tool5, ...],
    middleware=[
        LLMToolSelectorMiddleware(
            model="gpt-4.1-mini",       # 用于选择的模型（更小更快）
            max_tools=3,                 # 最多选择 N 个工具
            always_include=["search"],   # 始终包含的工具
        ),
    ],
)
```

#### Provider 专属 Middleware

| Provider | 可用中间件 |
| --- | --- |
| **Anthropic** | Prompt caching, bash tool, text editor, memory, file search |
| **AWS** | Prompt caching (Bedrock) |
| **OpenAI** | Content moderation |

OpenAI 内容审核示例：

```python
from langchain.agents.middleware import OpenAIModerationMiddleware

agent = create_agent(
    model="openai:gpt-4.1",
    tools=[search_tool],
    middleware=[
        OpenAIModerationMiddleware(
            model="openai:gpt-4.1",
            moderation_model="omni-moderation-latest",
            check_input=True,
            check_output=True,
            exit_behavior="end",
        ),
    ],
)
```

### 2.4 自定义 Middleware

继承 `AgentMiddleware` 并实现对应 hook，或用函数式装饰器：

```python
from langchain.agents.middleware import AgentMiddleware, AgentState, before_model, after_model
from langgraph.runtime import Runtime
from typing import Any

# 类形式
class MyMiddleware(AgentMiddleware):
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        messages = state["messages"]
        # ... 处理逻辑
        return {"messages": modified_messages}  # 返回更新，或 None 不更新

# 函数式快捷方式
@before_model
def log_before(state: AgentState, runtime: Runtime) -> dict | None:
    print(f"Processing request for user: {runtime.context.user_name}")
    return None

@after_model
def log_after(state: AgentState, runtime: Runtime) -> dict | None:
    print(f"Completed request for user: {runtime.context.user_name}")
    return None

agent = create_agent(model="gpt-4.1", tools=[...], middleware=[log_before, log_after])
```

---

## 3. State 状态管理

### 3.1 State 定义与 Reducer

State 用 TypedDict 定义。字段的更新行为由 **reducer** 决定：

```python
from typing import Annotated, TypedDict
from langgraph.graph import add_messages

class SummaryState(TypedDict):
    messages: Annotated[list, add_messages]   # 使用 add_messages reducer：追加
    files_relationship: str                    # 普通字段：覆盖
    plan_structure: str
```

| 字段类型 | 更新行为 |
| --- | --- |
| `Annotated[list, add_messages]` | **追加**到现有列表 |
| 普通字段 | **覆盖**原值 |

### 3.2 强制覆盖 messages

默认 `messages` 是追加的；需要完全替换整个列表时用 `Overwrite`：

```python
from langgraph.types import Overwrite

return {"messages": Overwrite([msg1, msg2, msg3])}  # 完全替换
```

> Context / State / Store 三者的区别与生命周期见 §7.1，是理解数据如何在 Agent 中流动的关键。

---

## 4. Tool

### 4.1 Tool 调用全流程

Agent 执行是 Model 与 Tools 节点之间的循环，直到 Model 不再请求调用工具：

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  User    │ ─▶ │  Model   │ ─▶ │  Tools   │ ─▶ │  Model   │ ─▶ ...
│  Input   │    │  Node    │    │  Node    │    │  Node    │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │
HumanMessage    AIMessage       ToolMessage     AIMessage
                (tool_calls)                    (最终回复 / 更多 tool)
```

以"请分析这些文件并生成摘要"为例，messages 列表逐轮增长（下面只标每轮**新增**的消息）：

```python
# 第 1 轮 · 用户输入
+ HumanMessage(content="请分析这些文件并生成摘要", id="human_1")

# 第 2 轮 · Model 决定调用 tool（content 通常为空）
+ AIMessage(id="ai_1", content="", tool_calls=[
      {"id": "call_abc123", "name": "analyze_file_relationships", "args": {"user_strategy": "SUMMARY"}}
  ])

# 第 3 轮 · Tools Node 执行，返回结果
+ ToolMessage(
      content="File relationships analyzed. Found 3 main themes.",
      tool_call_id="call_abc123",      # 必须匹配 AIMessage 中的 tool_call id
      name="analyze_file_relationships" # 可选，用于调试
  )

# 第 4 轮 · Model 继续调用下一个 tool
+ AIMessage(id="ai_2", content="", tool_calls=[
      {"id": "call_def456", "name": "generate_plan_structure", "args": {...}}
  ])

# 第 5 轮 · 第二个 tool 执行
+ ToolMessage(content="Structure plan created.", tool_call_id="call_def456")

# 第 6 轮 · Model 生成最终回复（无 tool_calls → 循环结束）
+ AIMessage(id="ai_3", content="根据分析，这些文件主要包含 3 个主题：...", tool_calls=[])
```

**循环终止条件**：

```python
AIMessage(content="", tool_calls=[{...}])         # 有 tool_calls → 继续执行 Tools Node
AIMessage(content="最终答案...", tool_calls=[])    # 空 tool_calls → 退出循环
AIMessage(content="最终答案...")                   # tool_calls 字段不存在也视为结束
```

**并行 Tool 调用**：Model 可在一次响应中请求多个 tool，Tools Node 并行执行，返回多条 ToolMessage（每条对应一个 `tool_call_id`）：

```python
AIMessage(content="", tool_calls=[
    {"id": "call_001", "name": "analyze_file_A", "args": {...}},
    {"id": "call_002", "name": "analyze_file_B", "args": {...}},
])
ToolMessage(content="File A result", tool_call_id="call_001")
ToolMessage(content="File B result", tool_call_id="call_002")
```

| 阶段 | Message 类型 | 关键字段 | 说明 |
| --- | --- | --- | --- |
| 用户输入 | `HumanMessage` | `content` | 用户的原始请求 |
| Model 调用工具 | `AIMessage` | `tool_calls` | 包含 `id`, `name`, `args` |
| Tool 返回结果 | `ToolMessage` | `tool_call_id` | **必须**匹配对应的 `tool_calls[].id` |
| Model 最终回复 | `AIMessage` | `content`，无 `tool_calls` | 空数组或缺失 = 结束循环 |

### 4.2 ToolMessage 与消息累积

`ToolMessage` 是 Tool 执行后返回给 Model 的"回执"：Model 用 `AIMessage.tool_calls` 说"请执行这个工具"，Tool 用 `ToolMessage` 说"执行完了，结果是……"。

它有两种创建途径：

```python
# 1. LangGraph 自动创建（tool 返回字符串时）
@tool
def my_tool() -> str:
    return "分析完成"
# 内部自动包装为：ToolMessage(content="分析完成", tool_call_id=<自动填充>)

# 2. 开发者显式创建（tool 返回 Command 时）
@tool
def my_tool(runtime: ToolRuntime) -> Command:
    return Command(update={"messages": [
        ToolMessage(content="分析完成", tool_call_id=runtime.tool_call_id)  # 必须手动指定
    ]})
```

显式创建能确保 `tool_call_id` 正确关联，避免 Model 误以为工具未完成而重复调用——这正是无限循环的常见根因（见 §5.2）：

```python
# 正常：ID 匹配 → Model 收到结果 → 决定结束
AIMessage(tool_calls=[{"id": "call_1"}]) → ToolMessage(tool_call_id="call_1") → AIMessage("完成")

# 异常：ID 丢失/不匹配 → Model 认为未完成 → 再次调用 → 无限循环
AIMessage(tool_calls=[{"id": "call_1"}]) → ToolMessage(tool_call_id="???") → 重复调用...
```

> [!warning] Context 膨胀
> ToolMessage 持续累积会撑大上下文。两个对策：① `SummarizationMiddleware` 自动压缩历史；② ToolMessage 只回简短摘要，详细结果写入 state（见 §4.3）。

### 4.3 Tool 的返回值

三种返回方式，按"是否需要更新 state / 控制流程"选择：

```python
# 方式 1：返回字符串（自动包装成 ToolMessage）
def my_tool(...) -> str:
    return "分析完成"

# 方式 2：显式返回 ToolMessage
def my_tool(...) -> ToolMessage:
    return ToolMessage(content="分析完成", tool_call_id=runtime.tool_call_id)

# 方式 3：返回 Command（可同时更新 state、控制跳转）
def my_tool(...) -> Command:
    return Command(update={
        "my_field": result,             # 更新 state 字段（覆盖）
        "messages": [ToolMessage(...)], # 追加消息（add_messages reducer）
    })
```

| 需求 | 推荐方式 |
| --- | --- |
| 只返回结果给 LLM | `str` 或 `ToolMessage` |
| 返回结果 + 更新 state | `Command` |
| 控制执行流程（跳转节点） | `Command` with `goto` |

### 4.4 在 Tool 间共享数据

**推荐写入 State**：详细结果存 state，ToolMessage 只回简短确认，其他 tool 通过 `runtime.state` 读取。

```python
from langgraph.types import Command
from langchain_core.messages import ToolMessage

@tool
def collect_file_snapshot(file_id: str, runtime: ToolRuntime[Context, CustomState]) -> Command:
    snapshot = _get_snapshot(file_id, runtime.context)
    return Command(update={
        "file_snapshots": {file_id: snapshot},   # 写入 state
        "messages": [ToolMessage(
            content=f"已收集文件 {file_id} 的快照",  # 给模型的只是简短确认
            tool_call_id=runtime.tool_call_id,
        )],
    })
```

**需要跨 session 持久化时写入 Store**：

```python
@tool
def collect_file_snapshot(file_id: str, runtime: ToolRuntime) -> Command:
    snapshot = _get_snapshot(file_id)
    runtime.store.put(("snapshots",), file_id, {"data": snapshot})  # 持久化存储
    return f"已收集文件 {file_id} 的快照"
```

> State 与 Store 的生命周期差异见 §7.1。

---

## 5. 循环控制与稳定性

### 5.1 recursion_limit

Agent 循环到"模型不再请求任何工具"才正常结束；否则到 `recursion_limit` 强制终止。

```python
agent = create_agent(...).with_config({"recursion_limit": 1000})
```

| 设置 | 适用 |
| --- | --- |
| 默认值（25） | 简单任务 |
| 100–500 | 一般多步骤任务 |
| 1000 | 复杂长流程任务 |

超限抛出 `GraphRecursionError: Recursion limit of N reached without hitting a stop condition.`

### 5.2 无限循环排查

遇到 `GraphRecursionError` 通常是 Agent 无法正常结束循环。按以下顺序排查：

| 步骤 | 检查项 | 方法 |
| --- | --- | --- |
| 1 | 哪个 tool 被重复调用 | LangSmith trace 或打印日志 |
| 2 | 该 tool 所有 return 路径 | 确保都返回带 `tool_call_id` 的响应 |
| 3 | 缓存/条件分支的返回类型 | 保持一致 |
| 4 | system_prompt | 是否有明确的结束指导 |
| 5 | tool 调用次数限制 | 见 §5.3 |
| 6 | RemainingSteps 兜底 | 见下文 |

**根因 1：返回值缺少 tool_call_id**——返回纯字符串在某些情况下 id 关联失败，改为显式返回带 `tool_call_id` 的 ToolMessage（见 §4.2）。

**根因 2：返回类型不一致**——缓存命中返回 `str`、正常返回 `Command`，会让 Model 行为异常。保持所有分支返回类型一致：

```python
@tool
def analyze(runtime: ToolRuntime[MyState]) -> Command:
    if runtime.state.get("result"):       # 缓存命中也返回 Command
        return Command(update={"messages": [
            ToolMessage(content="Already analyzed. Using cached result.",
                        tool_call_id=runtime.tool_call_id)
        ]})
    result = do_analysis()
    return Command(update={
        "result": result,
        "messages": [ToolMessage(content="Analysis done.", tool_call_id=runtime.tool_call_id)],
    })
```

**根因 3：Prompt 缺少结束指导**——在 system_prompt 中明确步骤和终止条件：

```python
system_prompt = """
你是一个文档分析助手。请按以下步骤工作：
1. 调用 analyze_files 分析文件关系（仅调用一次）
2. 调用 generate_plan 生成计划（仅调用一次）
3. 调用 create_summary 生成最终摘要
4. 完成后直接输出结果，不要再调用任何工具

重要：每个工具最多调用一次。如果工具返回"已完成"或"使用缓存"，直接进入下一步。
"""
```

**根因 4：tool 被重复调用**——见 §5.3 的调用次数限制。

**根因 5：复杂流程意外回到已执行步骤**——用 `RemainingSteps` 兜底，剩余步数不足时提前返回：

```python
from langgraph.managed import RemainingSteps

class MyState(TypedDict):
    messages: Annotated[list, add_messages]
    remaining_steps: RemainingSteps   # Graph 级别，自动注入剩余步数

@tool
def my_tool(runtime: ToolRuntime[MyState]) -> Command:
    if runtime.state.get("remaining_steps", 100) < 10:
        return Command(update={"messages": [
            ToolMessage(content="Approaching limit. Returning current result.",
                        tool_call_id=runtime.tool_call_id)
        ]})
    ...  # 正常执行
```

### 5.3 Tool 调用次数限制

LLM 可能"忘记"已调用过某工具而重复调用，造成 token 浪费、延迟增加、结果不一致。在 State 中记录调用次数、执行前检查是否超限。

**① State 加计数字段**（用 `merge_dicts` reducer 合并）：

```python
from typing import Annotated
from langchain.agents import AgentState

def merge_dicts(left: dict, right: dict) -> dict:
    """合并两个字典，右侧优先。"""
    return {**(left or {}), **(right or {})}

class MyState(AgentState):
    tool_call_counts: Annotated[dict, merge_dicts]  # {"tool_name": count}
```

**② 统一的限制器模块**：

```python
# agent/core/tool_limiter.py
DEFAULT_MAX_CALLS = 2

def check_tool_limit(runtime, tool_name: str, max_calls: int = DEFAULT_MAX_CALLS) -> str | None:
    """检查工具是否超过调用次数限制。超限返回错误消息，否则返回 None。"""
    counts = runtime.state.get("tool_call_counts", {})
    current = counts.get(tool_name, 0)
    if current >= max_calls:
        return f"Error: Tool '{tool_name}' has reached its limit ({max_calls} calls)."
    return None

def increment_tool_count(tool_name: str, current_counts: dict) -> dict:
    """返回更新后的计数字典（用于 state 更新）。"""
    return {tool_name: current_counts.get(tool_name, 0) + 1}
```

**③ 在 Tool 中使用**：超限时返回已有结果而非报错，确保工作流能继续。

```python
from agent.core.tool_limiter import check_tool_limit, increment_tool_count

@tool
def generate_summary(runtime: ToolRuntime[MyState]) -> Command:
    """生成摘要。注意：此工具最多调用 2 次。"""
    tool_name = "generate_summary"

    if check_tool_limit(runtime, tool_name):              # 超限 → 返回已有结果
        existing = runtime.state.get("final_summary", "")
        msg = (f"Tool call limit reached. Returning existing result:\n\n{existing}"
               if existing else "Error: limit reached and no existing result.")
        return Command(update={"messages": [
            ToolMessage(content=msg, tool_call_id=runtime.tool_call_id)
        ]})

    result = do_work()                                    # 正常执行
    return Command(update={
        "final_summary": result,
        "tool_call_counts": increment_tool_count(tool_name, runtime.state.get("tool_call_counts", {})),
        "messages": [ToolMessage(content="Summary generated", tool_call_id=runtime.tool_call_id)],
    })
```

> [!tip] 最佳实践
> ① 在 docstring 中写明调用次数限制，让 LLM 知道；② 超限返回已有结果而非错误；③ 用统一的限制器模块，别每个工具重复实现；④ 生成类工具 2 次足够（允许一次重试），查询类工具可放宽。

**RemainingSteps vs 调用次数限制**：

| 特性 | RemainingSteps | Tool 调用次数限制 |
| --- | --- | --- |
| 作用范围 | 整个 Graph（所有 node） | 单个 Tool |
| 计数方式 | 所有 super-step 计数 | 只计特定 tool |
| 用途 | 全局递归保护、兜底 | 精确控制单个 tool |
| 推荐场景 | 复杂流程的安全网 | 已知某 tool 可能被重复调用 |

---

## 6. Structured Output

### 6.1 概览

`create_agent` 通过 `response_format` 参数支持结构化输出。Agent 完成所有 tool 调用后，最终响应被强制转换为指定 schema，结果存在 `state["structured_response"]`。

```python
def create_agent(
    ...
    response_format: Union[
        ToolStrategy[StructuredResponseT],
        ProviderStrategy[StructuredResponseT],
        type[StructuredResponseT],      # 自动选择最佳策略
        None,
    ]
)
```

### 6.2 两种策略

| 策略 | 原理 | 适用 |
| --- | --- | --- |
| `ToolStrategy` | 通过人工 tool calling 生成结构化输出 | 所有支持 tool calling 的模型 |
| `ProviderStrategy` | 使用 Provider 原生结构化输出 | OpenAI / Anthropic / xAI 等支持的 Provider |

直接传入 schema 类型时自动选择：支持原生结构化输出 → `ProviderStrategy`，否则 → `ToolStrategy`。

### 6.3 Schema 类型

支持 Pydantic Model、dataclass、TypedDict、JSON Schema：

```python
# Pydantic Model
from pydantic import BaseModel, Field
class ContactInfo(BaseModel):
    """联系人信息。"""
    name: str = Field(description="姓名")
    email: str = Field(description="邮箱")

# dataclass
from dataclasses import dataclass
@dataclass
class ContactInfo:
    name: str
    email: str

# TypedDict
from typing_extensions import TypedDict
class ContactInfo(TypedDict):
    name: str
    email: str

# JSON Schema
contact_schema = {
    "type": "object",
    "properties": {"name": {"type": "string"}, "email": {"type": "string"}},
    "required": ["name", "email"],
}
```

### 6.4 显式指定策略与错误处理

```python
from langchain.agents.structured_output import ToolStrategy, ProviderStrategy

# ToolStrategy 兼容性最好；默认 handle_errors=True，输出不符合 schema 时自动反馈让模型重试
agent = create_agent(model="gpt-4.1-mini", tools=[search_tool],
                     response_format=ToolStrategy(ContactInfo))

# ProviderStrategy 更可靠，但需 Provider 支持
agent = create_agent(model="gpt-4.1", response_format=ProviderStrategy(ContactInfo))

# 支持多种可能的输出类型
from typing import Union
agent = create_agent(model="gpt-4.1", tools=[],
                     response_format=ToolStrategy(Union[ContactInfo, EventDetails]))
```

获取结果：

```python
result = agent.invoke({"messages": [{"role": "user", "content": "Extract: John Doe, john@example.com"}]})
print(result["structured_response"])
# ContactInfo(name='John Doe', email='john@example.com')
```

> [!note] 子 Agent 的结构化输出
> 子 Agent 也可设 `response_format` 验证输出，但**结构化对象本身不会返回给父 Agent**——需要在 ToolMessage 中显式包含结构化数据才能回传。

---

## 7. Runtime Context

### 7.1 Context / State / Store

理解三者的区别是掌握数据流动的关键：

| 概念 | 说明 | 可变性 | 生命周期 | 访问方式 |
| --- | --- | --- | --- | --- |
| `context_schema` | 静态配置（用户 ID、数据库连接） | 不可变 | 单次运行 | `runtime.context` |
| `state_schema` | 动态状态（对话历史、中间结果） | 可变 | 单次运行 | `runtime.state` |
| Store | 持久化数据（用户偏好、长期记忆） | 可变 | 跨会话 | `runtime.store` |

### 7.2 在 Tool 中访问 Runtime

`runtime` 是**保留参数名**，按"名字 + 类型"注入（不靠位置，对模型不可见）。参数必须命名为 `runtime`、类型标注 `ToolRuntime`（可带泛型 `ToolRuntime[Context, State]`）。

```python
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime

@dataclass
class Context:
    user_id: str

@tool
def get_user_location(runtime: ToolRuntime[Context]) -> str:
    """获取用户位置。"""
    user_id = runtime.context.user_id
    return "Florida" if user_id == "1" else "SF"

agent = create_agent(model="claude-sonnet-4-6", tools=[get_user_location], context_schema=Context)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "Where am I?"}]},
    context=Context(user_id="1"),   # 运行时传入 context
)
```

> [!tip] runtime 参数位置随意
> 既可放第一个也可放最后。唯一约束来自 Python 语法：若某业务参数有默认值而 `runtime` 没有，`runtime` 需排在带默认值的参数之前。另外 `config` 同样是保留名，勿用作普通参数。

### 7.3 用 Store 实现长期记忆

```python
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-4.1",
    tools=[...],
    middleware=[store_aware_prompt],
    context_schema=Context,
    store=InMemoryStore(),
    checkpointer=InMemorySaver(),
)

# 用 thread_id 实现对话记忆
config = {"configurable": {"thread_id": "conversation_1"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Hello"}]},
    config=config,
    context=Context(user_id="1"),
)
```

---

## 8. Deep Agents：开箱即用的封装

### 8.1 定位与组成

`deepagents` 是构建在 LangChain 之上的 **"Agent Harness"（Agent 套件）**，用 LangGraph Runtime 实现持久化执行、流式输出、HITL 等生产级特性。核心理念是"相同的 tool calling 循环，但内置了实用能力"：任务规划、文件系统、子 Agent 委派、长期记忆、上下文压缩。

| 组件 | 说明 |
| --- | --- |
| **Deep Agents SDK** | Python/JS 库，用于构建可处理任意任务的 Agent |
| **Deep Agents CLI** | 基于 SDK 构建的终端编程 Agent |

安装：

```bash
pip install deepagents                          # SDK
uv tool install 'deepagents-cli[anthropic]'     # CLI（选择 Provider）
pip install deepagents-acp                       # ACP 集成
```

最小示例：

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="claude-sonnet-4-6",
    system_prompt="You are a helpful research assistant.",
)
result = agent.invoke({"messages": [{"role": "user", "content": "What is LangGraph?"}]})
```

### 8.2 create_deep_agent 签名

```python
create_deep_agent(
    name: str | None = None,
    model: str | BaseChatModel | None = None,
    tools: Sequence[BaseTool | Callable | dict[str, Any]] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: list[AgentMiddleware] | None = None,
    subagents: list[SubAgent | CompiledSubAgent] | None = None,
    backend: Backend | Callable | None = None,
    interrupt_on: dict[str, bool | dict] | None = None,
    skills: list[str] | None = None,
    memory: list[str] | None = None,
    checkpointer: BaseCheckpointSaver | None = None,
    store: BaseStore | None = None,
) -> CompiledStateGraph
```

| 参数 | 说明 |
| --- | --- |
| `name` | Agent 名称，用于追踪和日志 |
| `model` | 模型标识符或实例 |
| `tools` | 自定义工具列表 |
| `system_prompt` | 系统提示（**追加**到内置 prompt 之后） |
| `middleware` | 额外的自定义中间件 |
| `subagents` | 自定义子 Agent 列表（定义方式同 §2.3） |
| `backend` | 文件系统后端（见 §8.4） |
| `interrupt_on` | HITL 工具审批配置（等价于装配 `HumanInTheLoopMiddleware`，语义同 §2.3） |
| `skills` | Skill 目录路径列表（见 §8.5） |
| `memory` | AGENTS.md 文件路径列表（见 §8.6） |
| `checkpointer` | 状态持久化（HITL 必需） |
| `store` | 持久化 KV 存储 |

带子 Agent 的研究 Agent：

```python
import os
from typing import Literal
from tavily import TavilyClient
from deepagents import create_deep_agent

tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

def internet_search(query: str, max_results: int = 5,
                    topic: Literal["general", "news", "finance"] = "general"):
    """Run a web search."""
    return tavily_client.search(query, max_results=max_results, topic=topic)

research_subagent = {
    "name": "research-agent",
    "description": "Used to research more in depth questions",
    "system_prompt": "You are a great researcher",
    "tools": [internet_search],
    "model": "claude-sonnet-4-6",
}

agent = create_deep_agent(model="claude-sonnet-4-6", subagents=[research_subagent], name="main-agent")
result = agent.invoke({"messages": [{"role": "user", "content": "Research the latest AI agent frameworks"}]})
```

### 8.3 内置 Middleware 栈

Deep Agent 默认装好以下中间件，无需手动配置——它们正是 §2.3 中逐个讲过的积木：

| Middleware | 说明 | 详见 |
| --- | --- | --- |
| `TodoListMiddleware` | 任务规划和追踪 | §2.3 |
| `FilesystemMiddleware` | 文件系统操作（读、写、导航） | §8.4 |
| `SubAgentMiddleware` | 子 Agent 委派 | §2.3 |
| `SummarizationMiddleware` | 对话历史压缩 | §2.3 |
| `AnthropicPromptCachingMiddleware` | Anthropic 模型的 prompt 缓存优化 | §2.3 |
| `PatchToolCallsMiddleware` | 修复中断的 tool call 消息历史 | §2.3 |

按需自动启用的中间件：

| Middleware | 触发条件 |
| --- | --- |
| `MemoryMiddleware` | 提供 `memory` 参数时 |
| `SkillsMiddleware` | 提供 `skills` 参数时 |
| `HumanInTheLoopMiddleware` | 提供 `interrupt_on` 参数时 |

`interrupt_on` 的配置语义与 §2.3 的 `HumanInTheLoopMiddleware` 完全一致：

```python
agent = create_deep_agent(
    model="claude-sonnet-4-6",
    tools=[delete_file, read_file, send_email],
    interrupt_on={
        "delete_file": True,                                        # approve/edit/reject
        "read_file": False,                                         # 无需审批
        "send_email": {"allowed_decisions": ["approve", "reject"]}, # 不允许编辑
    },
    checkpointer=MemorySaver(),
)
```

> [!note] OpenAI Responses API（v0.4）
> `"openai:"` 前缀的模型字符串默认使用 Responses API。如需自定义，用 `init_chat_model("openai:...", use_responses_api=True, store=False, include=["reasoning.encrypted_content"])`。

### 8.4 Backends（文件系统后端）

Deep Agent 通过虚拟文件系统管理上下文——Agent 读写文件来存中间结果、计划和记忆。

| Backend | 说明 | 持久化 |
| --- | --- | --- |
| `StateBackend`（默认） | 存在 LangGraph State 中的临时文件系统 | 仅单个 thread |
| `FilesystemBackend` | 本地机器文件系统 | 本地磁盘 |
| `StoreBackend` | LangGraph Store 持久化 | 跨 thread |
| `LocalShellBackend` | 文件系统 + shell 执行 | 本地磁盘 |
| `CompositeBackend` | 路由器，不同路径指向不同后端 | 混合 |
| Sandbox | 隔离环境（Modal / Daytona / Deno / VFS） | 取决于 sandbox |

```python
# 默认 StateBackend（临时）
agent = create_deep_agent()

# 本地文件系统
from deepagents.backends import FilesystemBackend
agent = create_deep_agent(backend=FilesystemBackend(root_dir="/Users/user/project"))

# 持久化 Store
from deepagents.backends import StoreBackend
agent = create_deep_agent(backend=lambda rt: StoreBackend(rt), store=InMemoryStore())

# 本地 Shell（文件系统 + 命令执行，谨慎使用）
from deepagents.backends import LocalShellBackend
agent = create_deep_agent(backend=LocalShellBackend(root_dir=".", env={"PATH": "/usr/bin:/bin"}))
```

`CompositeBackend` 按路径路由到不同后端，灵活度最高：

```python
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

composite_backend = lambda rt: CompositeBackend(
    default=StateBackend(rt),               # 默认：临时存储
    routes={"/memories/": StoreBackend(rt)},# /memories/ 路径：持久化
)
agent = create_deep_agent(backend=composite_backend, store=InMemoryStore())
```

> [!tip] v0.4 新增 Sandbox 集成包
> `langchain-modal`（Modal）、`langchain-daytona`（Daytona）、`langchain-runloop`（Runloop）。

### 8.5 Skills（技能）

Skills 是包含指令、脚本和资源的目录，为 Agent 提供专业能力。核心是 **Progressive Disclosure（渐进式加载）**：启动时只读每个 `SKILL.md` 的 frontmatter（name + description），仅当 Agent 判断某 skill 与当前任务相关时才加载完整内容，以省 token、减轻启动上下文压力。

目录结构：

```
skills/
├── web-research/
│   ├── SKILL.md           # 必需：指令和元数据
│   ├── search_template.py # 可选：脚本
│   └── docs/              # 可选：参考文档
└── code-review/
    └── SKILL.md
```

```python
# 配 FilesystemBackend 时直接给目录路径
from deepagents.backends.filesystem import FilesystemBackend
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="/Users/user/project"),
    skills=["/Users/user/project/skills/"],
    checkpointer=MemorySaver(),
)

# 用默认 StateBackend 时需手动注入 skill 文件
from deepagents.backends.utils import create_file_data

skills_files = {"/skills/langgraph-docs/SKILL.md": create_file_data(skill_content)}
agent = create_deep_agent(skills=["/skills/"], checkpointer=MemorySaver())
result = agent.invoke(
    {"messages": [{"role": "user", "content": "What is langgraph?"}], "files": skills_files},
    config={"configurable": {"thread_id": "12345"}},
)
```

CLI 中的 skill 发现路径（优先级从低到高）：

```bash
deepagents skill create test-skill   # 创建 skill

~/.deepagents/<agent_name>/skills/
~/.agents/skills/
.deepagents/skills/
.agents/skills/
```

### 8.6 Memory（记忆）

Memory 通过 `AGENTS.md` 文件提供持久化上下文，**始终加载进系统提示**——这是它与 Skills 渐进式加载的根本区别。

| 维度 | Skills | Memory |
| --- | --- | --- |
| 用途 | 按需加载的专业能力 | 始终可用的持久上下文 |
| 加载方式 | Progressive disclosure | 始终注入系统提示 |
| 格式 | SKILL.md（命名目录） | AGENTS.md 文件 |
| 分层 | 用户 → 项目（后者覆盖） | 用户 → 项目（合并） |
| 适用场景 | 任务特定的大量指令 | 项目惯例、用户偏好 |

```python
from deepagents.backends import FilesystemBackend
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="/Users/user/project"),
    memory=["./AGENTS.md"],
    checkpointer=MemorySaver(),
)
```

Memory 适合放：编码风格与惯例、用户偏好、项目指南、领域知识、Agent 交互中学到的模式。

### 8.7 CLI / ACP / LangSmith

**ACP 集成**——Deep Agent 可通过 ACP（Agent Communication Protocol）暴露为服务：

```python
import asyncio
from acp import run_agent
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver
from deepagents_acp.server import AgentServerACP

async def main() -> None:
    agent = create_deep_agent(system_prompt="You are a helpful coding assistant",
                              checkpointer=MemorySaver())
    server = AgentServerACP(agent)
    await run_agent(server)

if __name__ == "__main__":
    asyncio.run(main())
```

**LangSmith 追踪**——设 `LANGSMITH_API_KEY` 环境变量即自动启用；子 Agent 的 metadata 含 `lc_agent_name`，便于区分不同 Agent 的 trace。

---

## 9. 完整生产级示例

```python
from dataclasses import dataclass
from typing import Annotated, TypedDict

from langchain.agents import create_agent
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware, SummarizationMiddleware, TodoListMiddleware,
)
from langchain.agents.structured_output import ToolStrategy
from langchain.chat_models import init_chat_model
from langchain.tools import tool, ToolRuntime
from langchain_core.messages import ToolMessage
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import add_messages
from langgraph.types import Command


# 1. Context（静态配置）
@dataclass
class AppContext:
    user_id: str
    env: str = "production"


# 2. State（动态状态）
class AppState(TypedDict):
    messages: Annotated[list, add_messages]
    analysis_result: str


# 3. Response Schema（结构化输出）
@dataclass
class AnalysisResponse:
    summary: str
    confidence: float
    key_findings: list[str]


# 4. Tool（写 state + 简短回执）
@tool
def analyze_data(runtime: ToolRuntime[AppContext, AppState], query: str) -> Command:
    """分析数据并返回结果。"""
    result = f"Analysis for {query}: found 3 patterns"
    return Command(update={
        "analysis_result": result,
        "messages": [ToolMessage(
            content=f"Analysis complete: {result[:50]}...",
            tool_call_id=runtime.tool_call_id,
        )],
    })


# 5. 模型
model = init_chat_model("claude-sonnet-4-6", temperature=0)

# 6. 组装 Agent
agent = create_agent(
    model=model,
    tools=[analyze_data],
    system_prompt="你是一个数据分析助手。完成分析后输出结构化结果。",
    middleware=[
        TodoListMiddleware(),
        SummarizationMiddleware(model="gpt-4o-mini", trigger=("tokens", 100000), keep=("messages", 10)),
        HumanInTheLoopMiddleware(interrupt_on={"analyze_data": False}),
    ],
    context_schema=AppContext,
    state_schema=AppState,
    response_format=ToolStrategy(AnalysisResponse),
    checkpointer=InMemorySaver(),
).with_config({"recursion_limit": 500})

# 7. 运行
config = {"configurable": {"thread_id": "session_1"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "请分析最近的销售趋势"}]},
    config=config,
    context=AppContext(user_id="user_123"),
)
print(result["structured_response"])
# AnalysisResponse(summary='...', confidence=0.85, key_findings=['...'])
```

同一份配置若用 `create_deep_agent`，则 `TodoListMiddleware`、`SummarizationMiddleware`、`SubAgentMiddleware`、`PatchToolCallsMiddleware` 等都已默认装好，只需关注业务 tool、`subagents`、`interrupt_on` 与 `backend`。

---

## 参考资料

**create_agent / 核心概念**

- [LangChain Agents](https://docs.langchain.com/oss/python/langchain/agents)
- [LangChain Middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [LangChain Tools](https://docs.langchain.com/oss/python/langchain/tools)
- [Context Engineering](https://docs.langchain.com/oss/python/langchain/context-engineering)
- [Structured Output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [Human-in-the-Loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
- [Runtime](https://docs.langchain.com/oss/python/langchain/runtime)
- [从 create_react_agent 迁移](https://docs.langchain.com/oss/python/migrate/langchain-v1)

**Deep Agents**

- [Deep Agents Overview](https://docs.langchain.com/oss/python/deepagents/overview)
- [Quickstart](https://docs.langchain.com/oss/python/deepagents/quickstart)
- [Customization](https://docs.langchain.com/oss/python/deepagents/customization)
- [Backends](https://docs.langchain.com/oss/python/deepagents/backends)
- [Subagents](https://docs.langchain.com/oss/python/deepagents/subagents)
- [Skills](https://docs.langchain.com/oss/python/deepagents/skills)
- [Human-in-the-Loop](https://docs.langchain.com/oss/python/deepagents/human-in-the-loop)
- [CLI](https://docs.langchain.com/oss/python/deepagents/cli/overview)
- [Trace Deep Agents (LangSmith)](https://docs.langchain.com/langsmith/trace-deep-agents)
- [Frameworks, Runtimes, and Harnesses](https://docs.langchain.com/oss/python/concepts/products)
