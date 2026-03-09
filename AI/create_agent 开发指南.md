# Create agent 开发指南

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

### 1.2 内置 Middleware

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

### 1.3 自定义 Middleware

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

## 参考资料

- [LangChain Middleware 文档](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [LangGraph State 管理](https://docs.langchain.com/oss/python/langchain/context-engineering)
- [Deep Agents Middleware](https://docs.langchain.com/oss/python/deepagents/middleware)
