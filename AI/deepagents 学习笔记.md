# Deep Agents 学习笔记

> 基于 LangChain 官方文档（截至 2026.03），总结 `deepagents` 库的核心概念、架构和使用方法。
> 当前最新版本：deepagents v0.4 (2026.02) / langchain v1.1.0 / langgraph v1.x

## 1. 概述

### 1.1 什么是 Deep Agents

`deepagents` 是一个构建在 LangChain 核心组件之上的 **"Agent Harness"（Agent 套件）**，使用 LangGraph Runtime 实现持久化执行、流式输出、Human-in-the-Loop 等生产级特性。

**核心理念**：与其他 Agent 框架相同的 tool calling 循环，但内置了实用工具和能力：

- **任务规划**（TodoListMiddleware）
- **文件系统**（虚拟 / 本地 / 持久化）
- **子 Agent 委派**（SubAgentMiddleware）
- **长期记忆**（AGENTS.md / Store）
- **上下文压缩**（SummarizationMiddleware）

### 1.2 Deep Agents vs LangChain vs LangGraph

| 特性 | LangChain (`create_agent`) | LangGraph | Deep Agents (`create_deep_agent`) |
| --- | --- | --- | --- |
| **定位** | Agent 构建块 | 底层编排框架 | 高级 Agent 套件 |
| **短期记忆** | Short-term memory | Short-term memory | StateBackend |
| **长期记忆** | Long-term memory | Long-term memory | Long-term memory |
| **Skills** | Multi-agent skills | - | Skills (progressive disclosure) |
| **子 Agent** | Multi-agent subagents | Subgraphs | Subagents |
| **HITL** | HITL middleware | Interrupts | `interrupt_on` 参数 |
| **流式输出** | Agent Streaming | Streaming | Streaming |

**选择建议**：

- **简单 Agent** → `create_agent`
- **自定义工作流** → LangGraph
- **复杂多步骤任务**（规划、文件系统、子 Agent、记忆） → `create_deep_agent`

### 1.3 组成

| 组件 | 说明 |
| --- | --- |
| **Deep Agents SDK** | Python/JS 库，用于构建可处理任意任务的 Agent |
| **Deep Agents CLI** | 基于 SDK 构建的终端编程 Agent |

---

## 2. 快速开始

### 2.1 安装

```bash
# SDK
pip install deepagents

# CLI（选择 Provider）
uv tool install 'deepagents-cli[anthropic]'
uv tool install 'deepagents-cli[ollama,groq]'

# ACP 集成
pip install deepagents-acp
```

### 2.2 最小示例

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="claude-sonnet-4-6",
    system_prompt="You are a helpful research assistant.",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What is LangGraph?"}]
})
```

### 2.3 带 Subagent 的研究 Agent

```python
import os
from typing import Literal
from tavily import TavilyClient
from deepagents import create_deep_agent

tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

def internet_search(
    query: str,
    max_results: int = 5,
    topic: Literal["general", "news", "finance"] = "general",
    include_raw_content: bool = False,
):
    """Run a web search."""
    return tavily_client.search(
        query,
        max_results=max_results,
        include_raw_content=include_raw_content,
        topic=topic,
    )

research_subagent = {
    "name": "research-agent",
    "description": "Used to research more in depth questions",
    "system_prompt": "You are a great researcher",
    "tools": [internet_search],
    "model": "claude-sonnet-4-6",
}

agent = create_deep_agent(
    model="claude-sonnet-4-6",
    subagents=[research_subagent],
    name="main-agent",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Research the latest AI agent frameworks"}]
})
```

---

## 3. 配置选项

### 3.1 create_deep_agent 签名

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

### 3.2 参数总览

| 参数 | 说明 |
| --- | --- |
| `name` | Agent 名称，用于追踪和日志 |
| `model` | 模型标识符或实例 |
| `tools` | 自定义工具列表 |
| `system_prompt` | 系统提示（会追加到内置 prompt 之后） |
| `middleware` | 额外的自定义中间件 |
| `subagents` | 自定义子 Agent 列表 |
| `backend` | 文件系统后端 |
| `interrupt_on` | Human-in-the-Loop 工具审批配置 |
| `skills` | Skill 目录路径列表 |
| `memory` | AGENTS.md 文件路径列表 |
| `checkpointer` | 状态持久化（HITL 必需） |
| `store` | 持久化 KV 存储 |

---

## 4. 内置 Middleware

Deep Agent 默认包含以下中间件（无需手动配置）：

| Middleware | 说明 |
| --- | --- |
| `TodoListMiddleware` | 任务规划和追踪 |
| `FilesystemMiddleware` | 文件系统操作（读、写、导航） |
| `SubAgentMiddleware` | 子 Agent 委派 |
| `SummarizationMiddleware` | 对话历史压缩 |
| `AnthropicPromptCachingMiddleware` | Anthropic 模型的 prompt 缓存优化 |
| `PatchToolCallsMiddleware` | 修复中断的 tool call 消息历史 |

**条件性加载的中间件**（使用对应功能时自动启用）：

| Middleware | 触发条件 |
| --- | --- |
| `MemoryMiddleware` | 提供 `memory` 参数时 |
| `SkillsMiddleware` | 提供 `skills` 参数时 |
| `HumanInTheLoopMiddleware` | 提供 `interrupt_on` 参数时 |

此外，LangChain 还提供额外的预构建中间件（retry、fallback、PII 检测等），以及 **Summarization Tool Middleware**（允许 Agent 在合适时机主动触发摘要，而非固定 token 阈值）。

### v0.4 Summarization 重要变更（2026.02）

v0.4 对对话历史摘要机制做了重大改进：

| 变更项 | 旧行为 | v0.4 新行为 |
| --- | --- | --- |
| **触发位置** | 独立步骤 | 在 model node 中通过 `wrap_model_call` 触发 |
| **消息历史** | 摘要后丢弃原始消息 | **保留完整消息历史**在 graph state 中 |
| **Token 计数** | 近似估算 | 更准确的 token 计数 |
| **自动触发** | 仅基于阈值 | 当模型抛出 `ContextOverflowError` 时**自动触发** |

**ContextOverflowError 自动触发**：当前支持 `langchain-anthropic` 和 `langchain-openai`。

### v0.4 其他重要变更

**新 Sandbox 集成包**：

- `langchain-modal` — Modal sandbox
- `langchain-daytona` — Daytona sandbox
- `langchain-runloop` — Runloop sandbox

**OpenAI Responses API 默认启用**：`"openai:"` 前缀的模型字符串默认使用 Responses API：

```python
from langchain.chat_models import init_chat_model

agent = create_deep_agent(
    model=init_chat_model(
        "openai:...",
        use_responses_api=True,
        store=False,
        include=["reasoning.encrypted_content"],
    )
)
```

---

## 5. Backends（文件系统后端）

Deep Agent 通过虚拟文件系统管理上下文。Agent 可以读写文件来存储中间结果、计划和记忆。

### 5.1 可用后端

| Backend | 说明 | 持久化 |
| --- | --- | --- |
| `StateBackend`（默认） | 存储在 LangGraph State 中的临时文件系统 | 仅单个 thread |
| `FilesystemBackend` | 本地机器文件系统 | 本地磁盘 |
| `StoreBackend` | LangGraph Store 持久化 | 跨 thread |
| `LocalShellBackend` | 文件系统 + shell 执行 | 本地磁盘 |
| `CompositeBackend` | 路由器，不同路径指向不同后端 | 混合 |
| Sandbox | 隔离环境（Modal / Daytona / Deno / VFS） | 取决于 sandbox |

### 5.2 使用示例

```python
# 默认：StateBackend（临时）
agent = create_deep_agent()

# 本地文件系统
from deepagents.backends import FilesystemBackend
agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="/Users/user/project"),
)

# 持久化 Store
from deepagents.backends import StoreBackend
agent = create_deep_agent(
    backend=lambda rt: StoreBackend(rt),
    store=InMemoryStore(),
)

# 本地 Shell（文件系统 + 命令执行，谨慎使用）
from deepagents.backends import LocalShellBackend
agent = create_deep_agent(
    backend=LocalShellBackend(root_dir=".", env={"PATH": "/usr/bin:/bin"}),
)
```

### 5.3 CompositeBackend（路由器）

不同路径指向不同后端，最大灵活性：

```python
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

composite_backend = lambda rt: CompositeBackend(
    default=StateBackend(rt),           # 默认：临时存储
    routes={
        "/memories/": StoreBackend(rt), # /memories/ 路径：持久化
    },
)

agent = create_deep_agent(
    backend=composite_backend,
    store=InMemoryStore(),
)
```

---

## 6. Subagents（子 Agent）

### 6.1 核心概念

子 Agent 通过 `task` 工具实现任务委派，核心价值是 **上下文隔离（Context Quarantine）**：

```
Main Agent
  │
  ├─ task("Research AI frameworks", subagent_type="research-agent")
  │     └─ SubAgent（独立上下文）
  │          ├─ Tool 1 → ...
  │          ├─ Tool 2 → ...
  │          └─ 返回最终结果（单条消息）
  │
  └─ 继续主流程（上下文保持干净）
```

### 6.2 通用子 Agent

Deep Agent 默认自带一个 **general-purpose subagent**：

- 与主 Agent 共享系统提示
- 拥有所有相同工具
- 使用相同模型（除非覆盖）
- 继承主 Agent 的 skills

**主要用途**：上下文隔离 —— 将复杂任务委派给它，获取简洁结果，避免中间步骤膨胀主 Agent 上下文。

### 6.3 自定义子 Agent

```python
from deepagents import create_deep_agent

# 字典定义（简单场景）
research_subagent = {
    "name": "research-agent",
    "description": "Used to research in-depth questions",
    "system_prompt": "You are a great researcher",
    "tools": [internet_search],
    "model": "claude-sonnet-4-6",
    "middleware": [],                    # 可选
}

agent = create_deep_agent(
    model="claude-sonnet-4-6",
    subagents=[research_subagent],
)
```

**使用 CompiledSubAgent（复杂场景）**：

```python
from langchain.agents import create_agent
from deepagents.middleware.subagents import CompiledSubAgent

# 用 create_agent 构建
subagent_runnable = create_agent(
    model="claude-sonnet-4-6",
    system_prompt="You are a document synthesizer",
    tools=[extract_sections, merge_sections],
)

compiled_subagent = CompiledSubAgent(
    name="synthesizer",
    description="Process and synthesize documents",
    runnable=subagent_runnable,
)

agent = create_deep_agent(
    model="claude-sonnet-4-6",
    subagents=[compiled_subagent],
)
```

### 6.4 Subagent 结构化输出

子 Agent 支持 `response_format` 来验证输出：

```python
subagent = create_agent(
    model="claude-sonnet-4-6",
    tools=[...],
    response_format=AnalysisSchema,
)
```

> 注意：结构化对象本身不会返回给父 Agent。使用结构化输出时，需要在 ToolMessage 中包含结构化数据。

---

## 7. Human-in-the-Loop

### 7.1 基本配置

通过 `interrupt_on` 参数为每个工具配置审批策略：

```python
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()  # HITL 必须使用 checkpointer

agent = create_deep_agent(
    model="claude-sonnet-4-6",
    tools=[delete_file, read_file, send_email],
    interrupt_on={
        "delete_file": True,                                        # 默认：approve/edit/reject
        "read_file": False,                                         # 无需审批
        "send_email": {"allowed_decisions": ["approve", "reject"]}, # 不允许编辑
    },
    checkpointer=checkpointer,
)
```

### 7.2 审批流程

```
Agent → Check{需要审批？}
  ├─ No → 直接执行
  └─ Yes → 暂停等待人工决策
            ├─ approve → 执行
            ├─ edit → 修改后执行
            └─ reject → 取消
```

### 7.3 在 Tool 中直接中断

子 Agent 的工具可以直接调用 `interrupt()` 暂停执行：

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
    else:
        return f"Action '{action_description}' was REJECTED. Reason: {approval.get('reason')}"
```

---

## 8. Skills（技能）

### 8.1 概念

Skills 是包含指令、脚本和资源的目录，为 Agent 提供专业能力。核心特性是 **Progressive Disclosure（渐进式加载）**：

- 启动时只读取每个 SKILL.md 的 frontmatter（name + description）
- 仅当 Agent 判断某个 skill 与当前任务相关时，才加载完整内容
- 减少 token 消耗和启动上下文压力

### 8.2 Skill 目录结构

```
skills/
├── web-research/
│   ├── SKILL.md           # 必需：指令和元数据
│   ├── search_template.py # 可选：脚本
│   └── docs/              # 可选：参考文档
├── code-review/
│   └── SKILL.md
```

### 8.3 使用 Skills

```python
# 使用 FilesystemBackend
from deepagents import create_deep_agent
from deepagents.backends.filesystem import FilesystemBackend
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="/Users/user/project"),
    skills=["/Users/user/project/skills/"],
    checkpointer=MemorySaver(),
)

# 使用 StateBackend（需要手动注入 skill 文件）
from deepagents.backends.utils import create_file_data

skills_files = {
    "/skills/langgraph-docs/SKILL.md": create_file_data(skill_content),
}

agent = create_deep_agent(
    skills=["/skills/"],
    checkpointer=MemorySaver(),
)

result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "What is langgraph?"}],
        "files": skills_files,     # 注入到 StateBackend
    },
    config={"configurable": {"thread_id": "12345"}},
)
```

### 8.4 CLI 中的 Skills

```bash
# 创建 skill
deepagents skill create test-skill

# skill 发现路径（优先级从低到高）
~/.deepagents/<agent_name>/skills/
~/.agents/skills/
.deepagents/skills/
.agents/skills/
```

---

## 9. Memory（记忆）

### 9.1 概念

Memory 通过 `AGENTS.md` 文件为 Agent 提供持久化上下文，**始终加载到系统提示中**（与 Skills 的渐进式加载不同）。

### 9.2 Skills vs Memory

| 维度 | Skills | Memory |
| --- | --- | --- |
| **用途** | 按需加载的专业能力 | 始终可用的持久上下文 |
| **加载方式** | Progressive disclosure | 始终注入系统提示 |
| **格式** | SKILL.md（命名目录） | AGENTS.md 文件 |
| **分层** | 用户 → 项目（后者覆盖） | 用户 → 项目（合并） |
| **适用场景** | 任务特定的大量指令 | 项目惯例、用户偏好 |

### 9.3 使用 Memory

```python
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="/Users/user/project"),
    memory=["./AGENTS.md"],            # AGENTS.md 文件路径
    checkpointer=MemorySaver(),
)
```

**Memory 的用途**：

- 编码风格和惯例
- 用户偏好
- 项目指南
- 领域知识
- Agent 在交互中学到的模式

---

## 10. ACP 集成

Deep Agent 可以通过 ACP（Agent Communication Protocol）暴露为服务：

```python
import asyncio
from acp import run_agent
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver
from deepagents_acp.server import AgentServerACP

async def main() -> None:
    agent = create_deep_agent(
        system_prompt="You are a helpful coding assistant",
        checkpointer=MemorySaver(),
    )
    server = AgentServerACP(agent)
    await run_agent(server)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 11. LangSmith 追踪

Deep Agents 原生支持 LangSmith 追踪，用于调试和监控：

- 设置 `LANGSMITH_API_KEY` 环境变量即可自动启用
- 子 Agent 的 metadata 中包含 `lc_agent_name`，便于区分不同 Agent 的 trace
- 支持自定义 trace 配置

---

## 12. Deep Agents 与 create_agent 的关系

```
┌──────────────────────────────────────────────────┐
│                  Deep Agents                      │
│  ┌─────────────────────────────────────────────┐  │
│  │              create_deep_agent               │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │           create_agent                  │  │  │
│  │  │  ┌──────────────────────────────────┐   │  │  │
│  │  │  │        LangGraph Runtime         │   │  │  │
│  │  │  └──────────────────────────────────┘   │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  │  + Built-in Middleware Stack                  │  │
│  │  + Filesystem Tools                           │  │
│  │  + Planning Tools                             │  │
│  │  + Skills / Memory                            │  │
│  │  + Default System Prompt                      │  │
│  └─────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

**总结**：`create_deep_agent` 本质上是在 `create_agent` 之上预配置了一整套 middleware、工具和系统提示。如果你需要完全控制 Agent 行为，使用 `create_agent`；如果你想快速构建功能丰富的 Agent，使用 `create_deep_agent`。

---

## 参考资料

- [Deep Agents Overview](https://docs.langchain.com/oss/python/deepagents/overview)
- [Deep Agents Quickstart](https://docs.langchain.com/oss/python/deepagents/quickstart)
- [Deep Agents Customization](https://docs.langchain.com/oss/python/deepagents/customization)
- [Deep Agents Backends](https://docs.langchain.com/oss/python/deepagents/backends)
- [Deep Agents Subagents](https://docs.langchain.com/oss/python/deepagents/subagents)
- [Deep Agents Skills](https://docs.langchain.com/oss/python/deepagents/skills)
- [Deep Agents Human-in-the-Loop](https://docs.langchain.com/oss/python/deepagents/human-in-the-loop)
- [Deep Agents CLI](https://docs.langchain.com/oss/python/deepagents/cli/overview)
- [Trace Deep Agents (LangSmith)](https://docs.langchain.com/langsmith/trace-deep-agents)
- [Frameworks, Runtimes, and Harnesses](https://docs.langchain.com/oss/python/concepts/products)
