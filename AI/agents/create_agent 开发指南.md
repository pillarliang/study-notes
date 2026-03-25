# Create Agent 开发指南

> 基于 LangChain v1.x 官方文档（截至 2026.03），总结 `create_agent` 的完整使用方法和最佳实践。
> 当前最新版本：langchain v1.1.0 (2025.11) / langgraph v1.x / deepagents v0.4 (2026.02)
>
> `create_agent` 是 LangChain v1.0 推出的生产级 Agent 构建函数，取代了之前 `langgraph.prebuilt.create_react_agent`。它基于 LangGraph 的 Graph Runtime 构建，核心特性是 **Middleware（中间件）** 架构。

## 0. 从 create_react_agent 迁移

| 变更项 | create_react_agent (旧) | create_agent (新) |
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

本文档总结了使用 LangChain/LangGraph 开发 Agent 的核心概念和最佳实践。

## 1. Middleware 中间件

Middleware 是 `create_agent` 的核心特性，用于在 agent 执行的不同阶段插入自定义逻辑。

### 1.1 Middleware Hooks

| Hook | 执行时机 | 用途 |
| --- | --- | --- |
| `before_agent` | Agent 开始运行前 | 加载记忆、验证输入 |
| `before_model` | 每次 LLM 调用前 | 更新 prompt、裁剪消息 |
| `wrap_model_call` | 包裹 LLM 调用 | 拦截和修改请求/响应 |
| `wrap_tool_call` | 包裹 Tool 调用 | 拦截和修改工具执行 |
| `after_model` | 每次 LLM 响应后 | 验证输出、应用 guardrails |
| `after_agent` | Agent 完成后 | 保存结果、清理资源 |

### 1.2 动态 Prompt（Dynamic Prompt）

通过 middleware 实现基于上下文的动态系统提示：

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

**也可以从 Store 读取用户偏好**：

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

### 1.3 内置 Middleware

### TodoListMiddleware

为 agent 提供任务规划和追踪能力：

```python
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[TodoListMiddleware()],
)
```

- 自动提供 `write_todos` 工具
- 无需配置参数
- 适用于复杂多步骤任务

### SummarizationMiddleware

自动压缩对话历史，防止超出上下文窗口：

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

**参数说明：**

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `model` | str | BaseChatModel | 用于生成摘要的模型 |
| `trigger` | ContextSize | 触发摘要的条件 |
| `keep` | ContextSize | 摘要后保留多少内容 |

**ContextSize 格式：**

```python
("tokens", 170000)    # 基于 token 数量
("messages", 20)      # 基于消息数量
("fraction", 0.8)     # 基于模型上下文窗口比例
```

> 注意：max_tokens_before_summary 和 messages_to_keep 参数已废弃，请使用 trigger 和 keep。

**v0.4 变更（2026.02）**：在 deepagents 中，Summarization 已改为在 model node 中通过 `wrap_model_call` 触发，保留完整消息历史，token 计数更准确。当模型抛出 `ContextOverflowError` 时会自动触发（当前支持 `langchain-anthropic` 和 `langchain-openai`）。
>

### PatchToolCallsMiddleware

处理悬挂的 tool calls（deepagents 包提供）：

```python
from deepagents.middleware.patch_tool_calls import PatchToolCallsMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[PatchToolCallsMiddleware()],
)
```

**问题背景**：当 AI 发出 tool_call 但没有对应的 ToolMessage 响应时（如用户中断、超时等），会导致消息历史不完整。

**工作原理**：
1. 在 `before_agent` 阶段遍历所有消息
2. 检查每个 AI 消息的 tool_calls
3. 如果找不到对应的 ToolMessage，自动补充一个

```
AI Message (tool_calls: [{id: "abc"}])
    ↓
查找后续是否有 tool_call_id == "abc" 的 ToolMessage
    ↓
没找到 → 插入: ToolMessage("Tool call xxx was cancelled...")
```

### SubAgentMiddleware

通过 `task` 工具实现主 Agent 向子 Agent 委派任务，核心价值：**上下文隔离**。

```python
from deepagents.middleware.subagents import SubAgentMiddleware, CompiledSubAgent
from langchain.agents import create_agent
```

#### 基本原理

SubAgentMiddleware 在初始化时做两件事：

1. **注入 `task` 工具**：自动创建一个 `task` 工具并加入主 Agent 的工具列表。`task` 工具的 description 中包含所有可用子 Agent 的 `name` + `description`，供 LLM 选择委派目标。
2. **追加 System Prompt**：在每次 `wrap_model_call` 时，将 `TASK_SYSTEM_PROMPT`（task 工具使用指南：何时委派、子 Agent 生命周期、并行使用等）追加到主 Agent 的 system message 末尾。

主 Agent 的 LLM 看到 system prompt 中的指导 + 工具列表中的 `task` 工具描述后，自行决定何时调用 `task(description, subagent_type)` 委派任务。子 Agent 在独立上下文中执行，完成后返回单条结果消息。

```
Main Agent
  │
  ├─ task(description="分析文件...", subagent_type="research")
  │     │
  │     └─ SubAgent (独立上下文，独立工具集)
  │          ├─ Tool 1 → ...
  │          ├─ Tool 2 → ...
  │          └─ 返回最终结果 (单条 ToolMessage)
  │
  └─ 继续主流程
```

#### SubAgentMiddleware 参数

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

#### 定义子 Agent 的两种方式

**方式 1：字典定义（SubAgent TypedDict）**

适合简单场景，中间件自动创建 Agent：

```python
agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        SubAgentMiddleware(
            default_model="claude-sonnet-4-6",
            default_tools=[],
            subagents=[
                {
                    "name": "weather",
                    "description": "获取城市天气的子 Agent",
                    "system_prompt": "使用 get_weather 工具获取天气信息",
                    "tools": [get_weather],
                    "model": "gpt-4.1",         # 可选：覆盖默认模型
                    "middleware": [],             # 可选：额外中间件
                }
            ],
        )
    ],
)
```

**方式 2：预编译定义（CompiledSubAgent）**

适合复杂场景，可以传入自定义 LangGraph 或 `create_agent` 实例作为 runnable：

```python
from deepagents.middleware.subagents import CompiledSubAgent

# 方式 2a：用 create_agent 构建 runnable
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

# 方式 2b：用自定义 LangGraph 构建 runnable
from langgraph.graph import StateGraph

workflow = StateGraph(...)
# ... 构建自定义图 ...
custom_graph = workflow.compile()

custom_subagent = CompiledSubAgent(
    name="custom-workflow",
    description="自定义工作流子 Agent",
    runnable=custom_graph,        # state schema 必须包含 'messages' key
)
```

> **注意**：CompiledSubAgent 的 runnable 的 state schema **必须包含 `messages` key**。子 Agent 完成后，`messages` 列表中的最后一条消息会被提取为 ToolMessage 返回给主 Agent。

#### State 共享机制

子 Agent 与主 Agent 通过 **共享 State** 传递数据：

```python
# 委派时：主 Agent 的 state（排除 messages/todos/structured_response）传给子 Agent
subagent_state = {k: v for k, v in runtime.state.items() if k not in _EXCLUDED_STATE_KEYS}
subagent_state["messages"] = [HumanMessage(content=description)]

# 返回时：子 Agent 的 state 更新（排除 messages/todos/structured_response）合并回主 Agent
state_update = {k: v for k, v in result.items() if k not in _EXCLUDED_STATE_KEYS}
# messages 只保留最后一条作为 ToolMessage 返回
```

**排除的 State Key**：`messages`、`todos`、`structured_response`

这意味着：
- 主 Agent 写入 `structure_planner` → 子 Agent 可通过 state 读取
- 子 Agent 写入 `extracted_sections` → 主 Agent 在子 Agent 完成后可通过 state 读取
- 子 Agent 的完整对话历史（多轮 tool 调用）**不会**回传给主 Agent，只返回最终一条消息

#### general_purpose_agent 参数

| 值 | 行为 |
| --- | --- |
| `True`（默认） | 自动创建一个通用子 Agent，拥有与主 Agent 相同的 tools，主要用于上下文隔离 |
| `False` | 不创建通用子 Agent，只使用 `subagents` 中定义的自定义子 Agent |

设置为 `False` 的场景：当你只想让主 Agent 使用特定的预定义子 Agent，避免 LLM 自由委派。

#### 子 Agent 不被调用的排查

如果主 Agent 自己执行任务而不委派给子 Agent：

1. **让 description 更具体**：明确描述子 Agent 擅长什么
2. **在主 Agent system prompt 中强调委派**：
   ```python
   system_prompt = """...你的指令...

   重要：对于复杂任务，使用 task() 工具委派给子 Agent。
   这能保持上下文干净并提高结果质量。"""
   ```

#### 实际项目用法示例

以下是本项目中的完整用法（条件性创建子 Agent）：

```python
from deepagents.middleware.subagents import SubAgentMiddleware
from agent.subagents.map_reduce import create_map_reduce_subagent

# 根据内容大小决定是否使用子 Agent
total_size = get_total_content_size(file_infos)
use_map_reduce = total_size > settings.agent.content.small_content_threshold

subagents = []
if use_map_reduce:
    # 大内容 → 创建 Map-Reduce 子 Agent
    map_reduce_subagent = create_map_reduce_subagent(user_strategy, model)
    subagents.append(map_reduce_subagent)
else:
    # 小内容 → 给主 Agent 加 one-pass 工具
    tools.append(generate_few_file_summary)

# 配置中间件栈
middleware = [
    AgentLoggingMiddleware(agent_name="main"),
    TodoListMiddleware(),
    SubAgentMiddleware(
        default_model=model,
        default_tools=[],
        subagents=subagents,
        default_middleware=[
            AgentLoggingMiddleware(agent_name="subagent"),
            PatchToolCallsMiddleware(),
        ],
        general_purpose_agent=False,    # 禁用通用子 Agent，只用预定义的
    ),
    SummarizationMiddleware(...),
    PatchToolCallsMiddleware(),
]

agent = create_agent(
    model=model,
    tools=tools,                         # 主 Agent 工具（不含子 Agent 工具）
    system_prompt=system_prompt,
    middleware=middleware,
    state_schema=SummaryState,           # 主 Agent 和子 Agent 共享
).with_config({"recursion_limit": 1000})
```

#### 使用 SubAgent 的优势

| 优势 | 说明 |
| --- | --- |
| **上下文隔离** | 子 Agent 的中间 tool 调用不会污染主 Agent 的上下文窗口 |
| **并行执行** | 多个子 Agent 可以并发运行 |
| **专业化** | 子 Agent 可以有不同的工具集、模型和配置 |
| **Token 效率** | 大量中间步骤被压缩为一条结果消息 |
| **工具隔离** | 子 Agent 独有的工具不在主 Agent 的工具列表中，避免误调用 |

### HumanInTheLoopMiddleware

为敏感工具调用添加人工审批，基于 LangGraph 的 `interrupt` 机制：

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
                "send_email": True,                                          # 需要审批
                "delete_database": {
                    "description": "Please review before deleting",
                    "allowed_decisions": ["approve", "reject"],              # 不允许编辑
                },
                "search": False,                                             # 自动通过
            }
        ),
    ],
    checkpointer=InMemorySaver(),  # HITL 必须使用 checkpointer
)

config = {"configurable": {"thread_id": "some_id"}}

# Agent 会在敏感工具前暂停
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Send an email to the team"}]},
    config=config,
)

# 恢复执行：approve / edit / reject
result = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config,
)
```

**决策类型**：

| 决策 | 说明 |
| --- | --- |
| `approve` | 按原样执行 |
| `edit` | 修改参数后执行 |
| `reject` | 拒绝执行，附带反馈 |

### LLMToolSelectorMiddleware

用 LLM 智能选择相关工具，适用于工具数量 10+ 的场景：

```python
from langchain.agents import create_agent
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

**优势**：减少 token 消耗、提升模型专注度和准确性。

### Provider-specific Middleware

| Provider | 可用中间件 |
| --- | --- |
| **Anthropic** | Prompt caching, bash tool, text editor, memory, file search |
| **AWS** | Prompt caching (Bedrock) |
| **OpenAI** | Content moderation |

**OpenAI Content Moderation 示例**：

```python
from langchain.agents import create_agent
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

### 1.4 自定义 Middleware

```python
from langchain.agents.middleware import AgentMiddleware, AgentState
from langgraph.runtime import Runtime
from typing import Any

class MyMiddleware(AgentMiddleware):
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        # 在 LLM 调用前执行
        messages = state["messages"]
        # ... 处理逻辑
        return {"messages": modified_messages}  # 返回更新，或 None 不更新
```

**函数式快捷方式**：

```python
from langchain.agents.middleware import before_model, after_model

@before_model
def log_before(state: AgentState, runtime: Runtime) -> dict | None:
    print(f"Processing request for user: {runtime.context.user_name}")
    return None

@after_model
def log_after(state: AgentState, runtime: Runtime) -> dict | None:
    print(f"Completed request for user: {runtime.context.user_name}")
    return None

agent = create_agent(
    model="gpt-4.1",
    tools=[...],
    middleware=[log_before, log_after],
)
```

---

## 2. State 状态管理

### 2.1 State 定义

```python
from typing import Annotated, TypedDict
from langgraph.graph import add_messages

class SummaryState(TypedDict):
    # 使用 add_messages reducer：新消息会追加
    messages: Annotated[list, add_messages]

    # 普通字段：直接覆盖
    files_relationship: str
    plan_structure: str
```

### 2.2 Reducer 行为

| 字段类型 | 更新行为 |
| --- | --- |
| `Annotated[list, add_messages]` | **追加**到现有列表 |
| 普通字段 | **覆盖**原值 |

### 2.3 强制覆盖 messages

使用 `Overwrite` 可以覆盖整个列表：

```python
from langgraph.types import Overwrite

return {"messages": Overwrite([msg1, msg2, msg3])}  # 完全替换
```

### 2.4 Context vs State

| 概念 | 说明 | 可变性 | 生命周期 |
| --- | --- | --- | --- |
| `context_schema` | 静态配置（如用户ID、数据库连接） | 不可变 | 单次运行 |
| `state_schema` | 动态状态（如对话历史、中间结果） | 可变 | 单次运行 |
| Store | 持久化数据 | 可变 | 跨会话 |

---

## 3. Tool

### 3.1 Agent Tool 调用全流程

#### 流程概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Agent 执行循环                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐       │
│   │  User    │ ───▶ │  Model   │ ───▶ │  Tools   │ ───▶ │  Model   │ ───▶  │
│   │  Input   │      │  Node    │      │  Node    │      │  Node    │       │
│   └──────────┘      └──────────┘      └──────────┘      └──────────┘       │
│        │                 │                 │                 │              │
│        ▼                 ▼                 ▼                 ▼              │
│   HumanMessage      AIMessage         ToolMessage       AIMessage          │
│                    (tool_calls)                        (最终回复/更多tool)   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 完整 Messages 流转示例

假设用户请求："请分析这些文件并生成摘要"

```python
# ==================== 第 1 轮：用户输入 ====================
messages = [
    HumanMessage(content="请分析这些文件并生成摘要", id="human_1")
]

# ==================== 第 2 轮：Model 决定调用 tool ====================
# Model Node 执行后，返回带 tool_calls 的 AIMessage
messages = [
    HumanMessage(content="请分析这些文件并生成摘要", id="human_1"),
    AIMessage(
        content="",  # 调用工具时 content 通常为空
        id="ai_1",
        tool_calls=[
            {
                "id": "call_abc123",           # tool_call_id，用于关联响应
                "name": "analyze_file_relationships",
                "args": {"user_strategy": "SUMMARY"}
            }
        ]
    )
]

# ==================== 第 3 轮：Tool 执行并返回结果 ====================
# Tools Node 执行 tool，返回 ToolMessage
messages = [
    HumanMessage(content="请分析这些文件并生成摘要", id="human_1"),
    AIMessage(content="", id="ai_1", tool_calls=[{"id": "call_abc123", ...}]),
    ToolMessage(
        content="File relationships analyzed. Found 3 main themes.",
        tool_call_id="call_abc123",  # 必须匹配 AIMessage 中的 tool_call id
        name="analyze_file_relationships"  # 可选，用于调试
    )
]

# ==================== 第 4 轮：Model 继续调用下一个 tool ====================
messages = [
    HumanMessage(content="请分析这些文件并生成摘要", id="human_1"),
    AIMessage(content="", id="ai_1", tool_calls=[{"id": "call_abc123", ...}]),
    ToolMessage(
        content="File relationships analyzed. Found 3 main themes.",
        tool_call_id="call_abc123",  # 必须匹配 AIMessage 中的 tool_call id
        name="analyze_file_relationships"  # 可选，用于调试
    ),
    AIMessage(
        content="",
        id="ai_2",
        tool_calls=[{"id": "call_def456", "name": "generate_plan_structure", ...}]
    )
]

# ==================== 第 5 轮：第二个 Tool 执行 ====================
messages = [
    HumanMessage(content="请分析这些文件并生成摘要", id="human_1"),
    AIMessage(content="", id="ai_1", tool_calls=[{"id": "call_abc123", ...}]),
    ToolMessage(
        content="File relationships analyzed. Found 3 main themes.",
        tool_call_id="call_abc123",  # 必须匹配 AIMessage 中的 tool_call id
        name="analyze_file_relationships"  # 可选，用于调试
    ),
    AIMessage(
        content="",
        id="ai_2",
        tool_calls=[{"id": "call_def456", "name": "generate_plan_structure", ...}]
    ),
    ToolMessage(content="Structure plan created.", tool_call_id="call_def456")
]

# ==================== 第 6 轮：Model 生成最终回复 ====================
# Model 认为任务完成，返回不带 tool_calls 的 AIMessage → 循环结束
messages = [
    ...前面的消息...,
    AIMessage(
        content="根据分析，这些文件主要包含3个主题：...",
        id="ai_3",
        tool_calls=[]  # 空 = 循环结束
    )
]
```

#### 循环终止条件

```python
# 循环继续：AIMessage 包含 tool_calls
AIMessage(content="", tool_calls=[{...}])  # → 继续执行 Tools Node

# 循环结束：AIMessage 不包含 tool_calls
AIMessage(content="最终答案...", tool_calls=[])  # → 退出循环，返回结果
AIMessage(content="最终答案...")  # tool_calls 字段不存在也视为结束
```

#### 并行 Tool 调用

Model 可以在一次响应中请求调用多个 tool：

```python
# Model 同时请求调用两个 tool
AIMessage(
    content="",
    tool_calls=[
        {"id": "call_001", "name": "analyze_file_A", "args": {...}},
        {"id": "call_002", "name": "analyze_file_B", "args": {...}},
    ]
)

# Tools Node 并行执行后，返回多个 ToolMessage（每个必须有对应的 tool_call_id）
ToolMessage(content="File A result", tool_call_id="call_001")
ToolMessage(content="File B result", tool_call_id="call_002")
```

#### 关键点总结

| 阶段 | Message 类型 | 关键字段 | 说明 |
| --- | --- | --- | --- |
| 用户输入 | `HumanMessage` | `content` | 用户的原始请求 |
| Model 调用工具 | `AIMessage` | `tool_calls` | 包含 `id`, `name`, `args` |
| Tool 返回结果 | `ToolMessage` | `tool_call_id` | **必须**匹配对应的 `tool_calls[].id` |
| Model 最终回复 | `AIMessage` | `content`, 无 `tool_calls` | 无 tool_calls 或空数组 = 结束循环 |

### 3.2 ToolMessage 与消息累积机制

#### ToolMessage 的本质

`ToolMessage` 是 **Tool 执行后返回给 Model 的结果**，是 Tool 和 Model 之间的"回执"：

- Model 通过 `AIMessage.tool_calls` 说："请帮我执行这个工具"
- Tool 通过 `ToolMessage` 说："执行完了，结果是..."

#### 谁创建 ToolMessage？

有两种情况：

```python
# 1. LangGraph 自动创建（返回字符串时）
@tool
def my_tool() -> str:
    return "分析完成"
# LangGraph 内部自动包装成：ToolMessage(content="分析完成", tool_call_id=<自动填充>)

# 2. 开发者显式创建（返回 Command 时）
@tool
def my_tool(runtime: ToolRuntime) -> Command:
    return Command(
        update={
            "messages": [
                ToolMessage(
                    content="分析完成",
                    tool_call_id=runtime.tool_call_id,  # 必须手动指定
                )
            ]
        }
    )
```

#### 为什么推荐显式创建？

| 方式 | 优点 | 风险 |
| --- | --- | --- |
| 返回字符串（自动创建） | 简单 | `tool_call_id` 可能在某些情况下关联失败 |
| 返回 Command（显式创建） | 可控、可同时更新 state | 代码稍多 |

**显式创建可以确保 `tool_call_id` 正确关联**，避免 Model 认为工具未完成而重复调用。

#### 消息累积示例

ToolMessage 会累积在 messages 中，作为 model 的 context：

```python
messages = [
    HumanMessage("请总结这些文件"),
    AIMessage(tool_calls=[{"name": "analyze_file_relationships", "id": "call_1"}]),
    ToolMessage("File relationships analyzed...", tool_call_id="call_1"),  # 第1次 tool 结果
    AIMessage(tool_calls=[{"name": "generate_plan_structure", "id": "call_2"}]),
    ToolMessage("Structure plan created...", tool_call_id="call_2"),  # 第2次 tool 结果
    AIMessage("根据分析，总结如下..."),  # 最终回复（无 tool_calls = 结束）
]
```

#### 无限循环的根本原因

```python
# 正常流程
AIMessage(tool_calls=[{"id": "call_1"}])
    ↓
ToolMessage(tool_call_id="call_1")  # ✅ ID 匹配，Model 收到结果
    ↓
AIMessage(content="完成")  # Model 决定结束

# 问题流程
AIMessage(tool_calls=[{"id": "call_1"}])
    ↓
ToolMessage(tool_call_id="???")  # ❌ ID 不匹配或丢失
    ↓
Model 认为 tool 未完成，再次调用 → 无限循环...
```

#### Context 膨胀问题

随着 tool 调用增多，messages 会持续累积，导致 **context 膨胀**。解决方案：

1. **SummarizationMiddleware**：自动压缩历史消息
2. **ToolMessage 内容精简**：只返回关键信息，详细结果写入 state

```python
# ✅ 推荐：详细结果写入 state，ToolMessage 只返回摘要
return Command(
    update={
        "full_analysis": detailed_result,  # 完整结果存 state
        "messages": [
            ToolMessage(
                content="Analysis complete. 3 themes identified.",  # 简短摘要
                tool_call_id=runtime.tool_call_id,
            )
        ],
    }
)
```

### 3.3 Tool 的返回值

#### 三种返回方式

```python
# 方式 1：返回字符串（自动包装成 ToolMessage）
def my_tool(...) -> str:
    return "分析完成"

# 方式 2：显式返回 ToolMessage
def my_tool(...) -> ToolMessage:
    return ToolMessage(
        content="分析完成",
        tool_call_id=runtime.tool_call_id,
    )

# 方式 3：返回 Command（可同时更新 state）
def my_tool(...) -> Command:
    return Command(
        update={
            "my_field": result,           # 更新 state 字段
            "messages": [ToolMessage(...)], # 追加消息
        }
    )
```

#### 选择指南

| 需求 | 推荐方式 |
| --- | --- |
| 只返回结果给 LLM | `str` 或 `ToolMessage` |
| 返回结果 + 更新 state | `Command` |
| 控制执行流程（跳转节点） | `Command` with `goto` |

#### Command 详解

```python
from langgraph.types import Command
from langchain_core.messages import ToolMessage

def analyze_files(runtime: Runtime, ...) -> Command:
    result = do_analysis()

    return Command(
        update={
            # 更新 state 中的自定义字段
            "files_relationship": result,

            # 追加 ToolMessage 到消息历史
            "messages": [
                ToolMessage(
                    content=f"分析完成:{summary}",
                    tool_call_id=runtime.tool_call_id,
                )
            ],
        },
        # 可选：跳转到指定节点
        # goto="next_node",
    )
```

**Command.update 中的字段行为**：
- `messages`：遵循 `add_messages` reducer，**追加**
- 其他字段：**覆盖**

---

## 4. Agent 配置

### 4.1 创建 Agent

```python
from langchain.agents import create_agent

agent = create_agent(
    model=model,
    tools=tools,
    system_prompt="你是一个助手...",
    middleware=[
        TodoListMiddleware(),
        SummarizationMiddleware(...),
        PatchToolCallsMiddleware(),
    ],
    context_schema=MyContext,
    state_schema=MyState,
).with_config({"recursion_limit": 1000})
```

### 4.2 recursion_limit 与 Agent 循环机制

#### Agent 执行循环

Agent 执行是一个循环过程，直到满足**停止条件**：

```
Model call (请求调用 tool)
    → Tool execution (执行 tool)
    → ToolMessage (返回结果给 model)
    → Model call (model 根据结果决定下一步)
    → ... 循环 ...
    → Finish (model 不再请求调用任何 tool)
```

**停止条件**：

1. **正常结束**：模型不再请求调用任何工具（发出 finish 信号）
2. **强制终止**：达到 `recursion_limit` 限制

#### 配置 recursion_limit

```python
.with_config({"recursion_limit": 1000})
```

| 设置 | 说明 |
| --- | --- |
| 默认值（25） | 保守，适合简单任务 |
| 100-500 | 中等，适合一般多步骤任务 |
| 1000 | 宽松，适合复杂长流程任务 |

**超过限制会抛出**：

```
GraphRecursionError: Recursion limit of 1000 reached without hitting a stop condition.
For troubleshooting, visit: https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT
```

#### 无限循环问题排查与解决

当遇到 `GraphRecursionError` 时，通常是因为 Agent 无法正常结束循环。以下是常见原因和对应的解决方案。

##### 原因 1：Tool 返回值缺少 tool_call_id

Model 发出 tool call 后，期望收到带有匹配 `tool_call_id` 的 ToolMessage。如果 tool 返回纯字符串，LangGraph 虽然会自动包装，但在某些情况下可能导致 id 关联失败。

```python
# ❌ 问题：返回纯字符串
@tool
def my_tool(runtime: ToolRuntime) -> str:
    return "done"  # 可能导致 tool_call_id 关联问题

# ✅ 解决：显式返回带 tool_call_id 的 ToolMessage
@tool
def my_tool(runtime: ToolRuntime) -> Command:
    return Command(
        update={
            "messages": [
                ToolMessage(content="done", tool_call_id=runtime.tool_call_id)
            ]
        }
    )
```

##### 原因 2：缓存检查时返回值类型不一致

Tool 内部做缓存检查时，首次调用和缓存命中时的返回类型不一致，会导致 model 行为异常。

```python
# ❌ 问题：返回类型不一致
@tool
def analyze(runtime: ToolRuntime[MyState]) -> str:
    if runtime.state.get("result"):
        return runtime.state.get("result")  # 返回字符串

    result = do_analysis()
    return Command(update={...})  # 返回 Command

# ✅ 解决：保持返回类型一致
@tool
def analyze(runtime: ToolRuntime[MyState]) -> Command:
    if runtime.state.get("result"):
        return Command(
            update={
                "messages": [
                    ToolMessage(
                        content="Already analyzed. Using cached result.",
                        tool_call_id=runtime.tool_call_id,
                    )
                ]
            }
        )

    result = do_analysis()
    return Command(
        update={
            "result": result,
            "messages": [
                ToolMessage(content="Analysis done.", tool_call_id=runtime.tool_call_id)
            ]
        }
    )
```

##### 原因 3：Prompt 缺少明确的结束指导

Model 不知道何时应该停止调用 tool，持续循环。

```python
# ✅ 解决：在 system_prompt 中明确工作流程和结束条件
system_prompt = """
你是一个文档分析助手。请按以下步骤工作：

1. 调用 analyze_files 分析文件关系（仅调用一次）
2. 调用 generate_plan 生成计划（仅调用一次）
3. 调用 create_summary 生成最终摘要
4. 完成后直接输出结果，不要再调用任何工具

重要：每个工具最多调用一次。如果工具返回"已完成"或"使用缓存"，直接进入下一步。
"""
```

##### 原因 4：Tool 被重复调用（无调用次数限制）

LLM 可能"忘记"已经调用过某个工具，或认为需要重试。

解决方案：创建 Tool 调用限制器。首先，在 State 中添加计数字段：

```python
from typing import Annotated

def merge_dicts(left: dict, right: dict) -> dict:
    """合并字典，右侧优先。"""
    return {**(left or {}), **(right or {})}

class MyState(TypedDict):
    messages: Annotated[list, add_messages]
    tool_call_counts: Annotated[dict, merge_dicts]  # {"tool_name": count}
```

然后，创建限制器工具函数：

```python
# agent/core/tool_limiter.py

DEFAULT_MAX_CALLS = 2

def check_tool_limit(
    runtime: ToolRuntime,
    tool_name: str,
    max_calls: int = DEFAULT_MAX_CALLS,
) -> str | None:
    """检查工具是否超过调用次数限制。超限返回错误消息，否则返回 None。"""
    counts = runtime.state.get("tool_call_counts", {})
    current = counts.get(tool_name, 0)

    if current >= max_calls:
        return f"Tool '{tool_name}' limit reached ({max_calls} calls)."
    return None

def increment_tool_count(tool_name: str, current_counts: dict) -> dict:
    """返回更新后的计数字典。"""
    return {tool_name: current_counts.get(tool_name, 0) + 1}
```

最后，在 Tool 中使用：

```python
from agent.core.tool_limiter import check_tool_limit, increment_tool_count

@tool
def generate_summary(runtime: ToolRuntime[MyState]) -> Command:
    """生成摘要。注意：此工具最多调用 2 次。"""

    tool_name = "generate_summary"

    # 1. 检查调用限制
    if error := check_tool_limit(runtime, tool_name):
        return Command(
            update={
                "messages": [
                    ToolMessage(content=error, tool_call_id=runtime.tool_call_id)
                ]
            }
        )

    # 2. 正常执行
    result = do_work()

    # 3. 返回结果并增加计数
    return Command(
        update={
            "summary": result,
            "tool_call_counts": increment_tool_count(
                tool_name, runtime.state.get("tool_call_counts", {})
            ),
            "messages": [
                ToolMessage(content="Done", tool_call_id=runtime.tool_call_id)
            ]
        }
    )
```

> 更多详细说明和最佳实践，参见 [6. Tool 调用限制](#6-tool-调用限制) 章节。

##### 原因 5：复杂流程中的意外循环

在多步骤工作流中，控制流可能意外回到已执行的步骤。

```python
# ✅ 解决：使用 RemainingSteps 检测并提前退出
from langgraph.managed import RemainingSteps

class MyState(TypedDict):
    messages: Annotated[list, add_messages]
    remaining_steps: RemainingSteps  # Graph 级别，自动注入剩余步数

@tool
def my_tool(runtime: ToolRuntime[MyState]) -> Command:
    remaining = runtime.state.get("remaining_steps", 100)

    if remaining < 10:
        # 剩余步数不足，提前返回
        return Command(
            update={
                "messages": [
                    ToolMessage(
                        content="Approaching limit. Returning current result.",
                        tool_call_id=runtime.tool_call_id,
                    )
                ]
            }
        )

    # 正常执行
    ...
```

##### 排查清单

遇到无限循环时，按以下顺序排查：

| 步骤 | 检查项 | 排查方法 |
| --- | --- | --- |
| 1 | 查看 trace 中哪个 tool 被重复调用 | 使用 LangSmith 或打印日志 |
| 2 | 检查该 tool 的所有 return 路径 | 确保都返回带 `tool_call_id` 的响应 |
| 3 | 检查缓存/条件分支的返回类型 | 保持一致性 |
| 4 | 检查 system_prompt | 是否有明确的结束指导 |
| 5 | 考虑添加 tool 调用次数限制 | 参考 6.2 节 |
| 6 | 考虑使用 RemainingSteps | 作为兜底保护 |

##### RemainingSteps vs Tool 调用限制

| 特性 | RemainingSteps | Tool 调用次数限制 |
| --- | --- | --- |
| **作用范围** | 整个 Graph（所有 node） | 单个 Tool |
| **计数方式** | 所有 super-step 计数 | 只计特定 tool 调用 |
| **用途** | 全局递归保护、兜底 | 精确控制单个 tool |
| **推荐场景** | 复杂流程的安全网 | 已知某 tool 可能被重复调用 |

### 4.3 其他常用配置

```python
.with_config({
    "recursion_limit": 1000,
    "configurable": {
        "thread_id": "abc123",      # 用于 checkpointer 持久化
    },
    "tags": ["production"],         # 用于 LangSmith 追踪
    "metadata": {"user_id": "xxx"}, # 自定义元数据
})
```

---

## 5. 完整示例

```python
from typing import Annotated, TypedDict
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware, SummarizationMiddleware
from langchain_core.messages import ToolMessage
from langgraph.graph import add_messages
from langgraph.types import Command
from langgraph.runtime import Runtime
from deepagents.middleware.patch_tool_calls import PatchToolCallsMiddleware

# 1. 定义 State
class MyState(TypedDict):
    messages: Annotated[list, add_messages]
    analysis_result: str

# 2. 定义 Tool
def analyze_data(runtime: Runtime, data: str) -> Command:
    result = f"Analyzed:{data}"
    return Command(
        update={
            "analysis_result": result,
            "messages": [
                ToolMessage(
                    content=f"Analysis complete:{result[:50]}...",
                    tool_call_id=runtime.tool_call_id,
                )
            ],
        }
    )

# 3. 创建 Agent
agent = create_agent(
    model="gpt-4o",
    tools=[analyze_data],
    system_prompt="你是一个数据分析助手。",
    middleware=[
        TodoListMiddleware(),
        SummarizationMiddleware(
            model="gpt-4o-mini",
            trigger=("tokens", 100000),
            keep=("messages", 10),
        ),
        PatchToolCallsMiddleware(),
    ],
    state_schema=MyState,
).with_config({"recursion_limit": 500})

# 4. 运行 Agent
result = agent.invoke({
    "messages": [{"role": "user", "content": "请分析这些数据..."}]
})
```

---

## **在其他 Tool/Agent 中共享数据**

### 方案 1：**写入 State（推荐）**

```python
from langgraph.types import Command
from langchain.messages import ToolMessage

@tool
def collect_file_snapshot(
    file_id: str,
    runtime: ToolRuntime[ProjectSummaryContext, CustomState]
) -> Command:
    snapshot = _get_snapshot(file_id, runtime.context)

    return Command(update={
        # 写入 state，其他 tool 可以通过 runtime.state 访问
        "file_snapshots": {file_id: snapshot},
        # 给模型的只是简短确认
        "messages": [
            ToolMessage(
                content=f"已收集文件 {file_id} 的快照",
                tool_call_id=runtime.tool_call_id
            )
        ]
    })
```

### 方案2：**写入 Store（跨 session 持久化）**

```python
@tool
def collect_file_snapshot(
    file_id: str,
    runtime: ToolRuntime
) -> Command:
    snapshot = _get_snapshot(file_id)

    # 写入 store（持久化存储）
    runtime.store.put(("snapshots",), file_id, {"data": snapshot})

    return f"已收集文件 {file_id} 的快照"
```

---

## 6. Tool 调用限制

### 6.1 问题背景

在 Agent 工作流中，LLM 可能会**重复调用同一个 Tool**，导致以下问题：

| 问题 | 影响 |
| --- | --- |
| Token 浪费 | 重复的 LLM 调用消耗大量 token 成本 |
| 延迟增加 | 不必要的调用延长整体执行时间 |
| 结果不一致 | 多次调用可能产生不同结果，造成混乱 |

**常见原因**：

1. **LLM 决策的不确定性**：LLM 可能"忘记"已经调用过某个工具，或认为需要再次调用以获得更好结果
2. **Prompt 指令不够明确**：系统提示词中没有明确限制工具调用次数
3. **错误重试逻辑**：Agent 在遇到问题时可能尝试重新调用工具
4. **复杂流程中的循环**：在多步骤工作流中，控制流可能意外回到已执行的步骤

### 6.2 解决方案

通过在 State 中记录工具调用次数，并在工具执行前检查是否超限。

#### 6.2.1 定义 State 字段

```python
from typing import Annotated
from langchain.agents import AgentState

def merge_dicts(left: dict, right: dict) -> dict:
    """Merge two dicts, with right taking precedence."""
    if left is None:
        left = {}
    if right is None:
        right = {}
    return {**left, **right}

class MyState(AgentState):
    # ... 其他字段 ...
    tool_call_counts: Annotated[dict, merge_dicts]  # {"tool_name": count}
```

#### 6.2.2 创建限制器工具

```python
# agent/core/tool_limiter.py

DEFAULT_MAX_CALLS = 2

def check_tool_limit(
    runtime: "ToolRuntime[SummaryState]",
    tool_name: str,
    max_calls: int = DEFAULT_MAX_CALLS,
) -> str | None:
    """检查工具是否超过调用次数限制。

    Returns:
        超限时返回错误消息字符串，否则返回 None
    """
    tool_call_counts = runtime.state.get("tool_call_counts", {})
    current_count = tool_call_counts.get(tool_name, 0)

    if current_count >= max_calls:
        return (
            f"Error: Tool '{tool_name}' has reached its maximum call limit ({max_calls}). "
            f"This tool has already been called {current_count} times."
        )
    return None


def increment_tool_count(tool_name: str, current_counts: dict) -> dict:
    """增加工具调用计数。

    Returns:
        包含更新后计数的字典（用于 state 更新）
    """
    new_count = current_counts.get(tool_name, 0) + 1
    return {tool_name: new_count}
```

#### 6.2.3 在 Tool 中使用

```python
from agent.core.tool_limiter import check_tool_limit, increment_tool_count

@tool
def generate_summary(runtime: ToolRuntime[MyState]) -> str:
    """生成摘要。最多调用 2 次。"""

    # 1. 检查调用限制 - 超限时返回已有结果
    tool_name = "generate_summary"
    if check_tool_limit(runtime, tool_name):
        existing_result = runtime.state.get("final_summary", "")
        if existing_result:
            return f"Tool call limit reached. Returning existing result:\n\n{existing_result}"
        return "Error: Tool call limit reached and no existing result found."

    # 2. 正常执行工具逻辑
    result = do_something()

    # 3. 返回结果并增加计数
    return Command(
        update={
            "final_summary": result,
            "tool_call_counts": increment_tool_count(
                tool_name, runtime.state.get("tool_call_counts", {})
            ),
            "messages": [
                ToolMessage(
                    content="Summary generated",
                    tool_call_id=runtime.tool_call_id,
                )
            ],
        }
    )
```

### 6.3 行为说明

| 调用次数 | 行为 |
| --- | --- |
| 第 1 次 | 正常执行 LLM 调用，结果写入 state，计数 +1 |
| 第 2 次 | 正常执行 LLM 调用，结果写入 state，计数 +1 |
| 第 3+ 次 | **不执行 LLM**，返回 state 中已有的结果 |

### 6.4 与其他防重复方案对比

| 方案 | 实现方式 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **检查结果是否存在** | `if state.get("result"): return state["result"]` | 简单 | 只能调用一次，无法重试 |
| **布尔标记** | `if state.get("tool_completed"): return` | 简单 | 只能调用一次，无法重试 |
| **计数限制（推荐）** | 本方案 | 可配置次数，灵活 | 需要额外代码 |

### 6.5 最佳实践

1. **在 docstring 中说明限制**：让 LLM 知道工具有调用次数限制

   ```python
   """Generate summary. Note: This tool can be called at most 2 times per workflow."""
   ```

2. **超限时返回已有结果**：而不是返回错误，确保工作流能继续

3. **使用统一的限制器模块**：避免在每个工具中重复实现

4. **合理设置限制次数**：
   - 生成类工具（如摘要生成）：2 次足够（允许一次重试）
   - 查询类工具（如文件分析）：可能需要更多次

---

## 7. Structured Output（结构化输出）

### 7.1 概览

`create_agent` 通过 `response_format` 参数支持结构化输出。Agent 在完成所有 tool 调用后，最终响应会被强制转换为指定的 schema 格式，结果存储在 `state["structured_response"]` 中。

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

### 7.2 两种策略

| 策略 | 原理 | 适用场景 |
| --- | --- | --- |
| `ToolStrategy` | 通过人工 tool calling 生成结构化输出 | 所有支持 tool calling 的模型 |
| `ProviderStrategy` | 使用 Provider 原生结构化输出 | OpenAI / Anthropic / xAI 等支持的 Provider |

**自动选择**：直接传入 schema 类型时，LangChain 会根据模型 profile 自动选择：
- 支持原生结构化输出 → `ProviderStrategy`
- 其他 → `ToolStrategy`

### 7.3 Schema 类型支持

支持 Pydantic Model、dataclass、TypedDict、JSON Schema 四种格式：

```python
# 方式 1：Pydantic Model
from pydantic import BaseModel, Field

class ContactInfo(BaseModel):
    """联系人信息。"""
    name: str = Field(description="姓名")
    email: str = Field(description="邮箱")
    phone: str = Field(description="电话")

agent = create_agent(
    model="gpt-4.1",
    response_format=ContactInfo,  # 自动选择 ProviderStrategy
)

# 方式 2：dataclass
from dataclasses import dataclass

@dataclass
class ContactInfo:
    """联系人信息。"""
    name: str
    email: str
    phone: str

# 方式 3：TypedDict
from typing_extensions import TypedDict

class ContactInfo(TypedDict):
    """联系人信息。"""
    name: str
    email: str
    phone: str

# 方式 4：JSON Schema
contact_schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "email": {"type": "string"},
    },
    "required": ["name", "email"],
}
```

### 7.4 显式指定策略

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy, ProviderStrategy

# 显式使用 ToolStrategy（兼容性最好）
agent = create_agent(
    model="gpt-4.1-mini",
    tools=[search_tool],
    response_format=ToolStrategy(ContactInfo),
)

# 显式使用 ProviderStrategy（更可靠，但需要 Provider 支持）
agent = create_agent(
    model="gpt-4.1",
    response_format=ProviderStrategy(ContactInfo),
)
```

### 7.5 错误处理

`ToolStrategy` 默认启用 `handle_errors=True`，当模型生成不符合 schema 的输出时，会自动反馈错误让模型重试：

```python
# 多个可能的输出类型
from typing import Union

agent = create_agent(
    model="gpt-4.1",
    tools=[],
    response_format=ToolStrategy(Union[ContactInfo, EventDetails]),
)
```

### 7.6 获取结果

```python
result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract: John Doe, john@example.com"}]
})

print(result["structured_response"])
# ContactInfo(name='John Doe', email='john@example.com', phone='')
```

---

## 8. Runtime Context（运行时上下文）

### 8.1 Context vs State vs Store

| 概念 | 说明 | 可变性 | 生命周期 |
| --- | --- | --- | --- |
| `context_schema` | 静态配置（用户ID、数据库连接） | 不可变 | 单次运行 |
| `state_schema` | 动态状态（对话历史、中间结果） | 可变 | 单次运行 |
| Store | 持久化数据（用户偏好、长期记忆） | 可变 | 跨会话 |

### 8.2 在 Middleware 和 Tool 中访问 Runtime

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

agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[get_user_location],
    context_schema=Context,
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "Where am I?"}]},
    context=Context(user_id="1"),
)
```

### 8.3 使用 Store 实现长期记忆

```python
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import InMemorySaver

store = InMemoryStore()
checkpointer = InMemorySaver()

agent = create_agent(
    model="gpt-4.1",
    tools=[...],
    middleware=[store_aware_prompt],
    context_schema=Context,
    store=store,
    checkpointer=checkpointer,
)

# 使用 thread_id 实现对话记忆
config = {"configurable": {"thread_id": "conversation_1"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Hello"}]},
    config=config,
    context=Context(user_id="1"),
)
```

---

## 9. 完整生产级示例

```python
from dataclasses import dataclass
from typing import Annotated, TypedDict

from langchain.agents import create_agent
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware,
    SummarizationMiddleware,
    TodoListMiddleware,
)
from langchain.agents.structured_output import ToolStrategy
from langchain.chat_models import init_chat_model
from langchain.tools import tool, ToolRuntime
from langchain_core.messages import ToolMessage
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import add_messages
from langgraph.types import Command


# 1. 定义 Context
@dataclass
class AppContext:
    user_id: str
    env: str = "production"


# 2. 定义 State
class AppState(TypedDict):
    messages: Annotated[list, add_messages]
    analysis_result: str


# 3. 定义 Response Schema
@dataclass
class AnalysisResponse:
    summary: str
    confidence: float
    key_findings: list[str]


# 4. 定义 Tool
@tool
def analyze_data(runtime: ToolRuntime[AppContext, AppState], query: str) -> Command:
    """分析数据并返回结果。"""
    result = f"Analysis for {query}: found 3 patterns"
    return Command(
        update={
            "analysis_result": result,
            "messages": [
                ToolMessage(
                    content=f"Analysis complete: {result[:50]}...",
                    tool_call_id=runtime.tool_call_id,
                )
            ],
        }
    )


# 5. 配置模型
model = init_chat_model("claude-sonnet-4-6", temperature=0)

# 6. 创建 Agent
agent = create_agent(
    model=model,
    tools=[analyze_data],
    system_prompt="你是一个数据分析助手。完成分析后输出结构化结果。",
    middleware=[
        TodoListMiddleware(),
        SummarizationMiddleware(
            model="gpt-4o-mini",
            trigger=("tokens", 100000),
            keep=("messages", 10),
        ),
        HumanInTheLoopMiddleware(
            interrupt_on={"analyze_data": False},
        ),
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

---

## 参考资料

- [LangChain Agents 文档](https://docs.langchain.com/oss/python/langchain/agents)
- [LangChain Middleware 文档](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [LangChain Context Engineering](https://docs.langchain.com/oss/python/langchain/context-engineering)
- [LangChain Structured Output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [LangChain Human-in-the-Loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
- [LangChain Guardrails](https://docs.langchain.com/oss/python/langchain/guardrails)
- [从 create_react_agent 迁移](https://docs.langchain.com/oss/python/migrate/langchain-v1)
- [Deep Agents Middleware](https://docs.langchain.com/oss/python/deepagents/middleware)
