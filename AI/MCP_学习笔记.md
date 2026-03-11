# MCP (Model Context Protocol) 学习笔记

> MCP 是由 Anthropic 推出的**开放协议**，旨在标准化 LLM 应用与外部数据源、工具之间的交互方式。
> 可以把它理解为 **AI 领域的 USB-C**——一个统一的接口标准。
>
> 官方文档：https://modelcontextprotocol.io

---

## 1. 为什么需要 MCP？

传统方式下，每个 AI 应用都需要为不同的数据源和工具编写定制化的集成代码，形成 M×N 的组合爆炸：

```
AI 应用 1 ──→ 数据源 A
AI 应用 1 ──→ 数据源 B
AI 应用 2 ──→ 数据源 A    # 重复造轮子
AI 应用 2 ──→ 数据源 B    # 重复造轮子
```

MCP 将其简化为 M+N：

```
AI 应用 1 ──┐              ┌──→ MCP Server A (数据源 A)
            ├── MCP 协议 ──┤
AI 应用 2 ──┘              └──→ MCP Server B (数据源 B)
```

---

## 2. 整体架构

### 2.1 三种角色

```
┌─────────────────────────────────────────┐
│            MCP Host (AI 应用)            │
│  例如 Claude Desktop / VS Code / IDE    │
│                                         │
│  ┌─────────────┐  ┌─────────────┐       │
│  │ MCP Client 1│  │ MCP Client 2│  ...  │
│  └──────┬──────┘  └──────┬──────┘       │
└─────────┼────────────────┼──────────────┘
          │ 专用连接        │ 专用连接
          ▼                ▼
   ┌─────────────┐  ┌─────────────┐
   │ MCP Server A│  │ MCP Server B│
   │ (本地/远程)  │  │ (本地/远程)  │
   └─────────────┘  └─────────────┘
```

| 角色 | 职责 | 示例 |
|------|------|------|
| **Host** | AI 应用本身，协调和管理多个 Client | Claude Desktop, VS Code, Claude Code |
| **Client** | 维护与某个 Server 的连接，获取上下文 | Host 内部为每个 Server 创建的客户端实例 |
| **Server** | 暴露 Tool / Resource / Prompt 的服务端 | 文件系统 Server、数据库 Server、Sentry Server |

### 2.2 协议分层

MCP 由两层组成：

```
┌──────────────────────────────────────┐
│         数据层 (Data Layer)           │
│  JSON-RPC 2.0 协议                    │
│  • 生命周期管理（握手、能力协商）        │
│  • 服务端原语：Tool / Resource / Prompt │
│  • 客户端原语：Sampling / Elicitation   │
│  • 通知 & 进度追踪                     │
├──────────────────────────────────────┤
│        传输层 (Transport Layer)        │
│  • stdio（本地子进程）                  │
│  • Streamable HTTP（远程/生产环境）     │
└──────────────────────────────────────┘
```

---

## 3. 连接生命周期

MCP 是**有状态协议**，连接需要经过完整的生命周期管理：

```
Client                          Server
  │                                │
  │──── initialize (请求) ────────→│   1. 发送协议版本 + 客户端能力
  │←─── initialize (响应) ────────│   2. 返回服务端能力
  │──── initialized (通知) ──────→│   3. 确认初始化完成
  │                                │
  │        ═══ 正常通信 ═══         │   4. 双向请求/响应/通知
  │                                │
  │──── 断开连接 ─────────────────→│   5. 关闭
```

### 能力协商 (Capability Negotiation)

初始化时双方声明各自支持的能力，后续通信仅使用双方都支持的功能：

```json
// 客户端声明的能力
{
  "capabilities": {
    "sampling": {},        // 支持 LLM 采样
    "elicitation": {}      // 支持用户交互请求
  }
}

// 服务端声明的能力
{
  "capabilities": {
    "tools": { "listChanged": true },     // 支持工具 + 变更通知
    "resources": { "subscribe": true },    // 支持资源 + 订阅
    "prompts": { "listChanged": true }     // 支持提示词模板 + 变更通知
  }
}
```

---

## 4. 三大核心原语 (Server Primitives)

### 4.1 Tool（工具）

> **模型控制 (Model-controlled)**：LLM 根据上下文自动发现和调用。

Tool 让 LLM 能够**执行操作**——查询数据库、调用 API、执行计算等。

#### 发现与调用流程

```
LLM ←→ Client ←→ Server

1. Client → Server:  tools/list          # 发现可用工具
2. Server → Client:  [{name, description, inputSchema}, ...]
3. LLM 决定调用某个工具
4. Client → Server:  tools/call          # 执行工具
5. Server → Client:  {content: [...]}    # 返回结果
6. Client → LLM:     处理结果
```

#### 工具定义示例

```json
{
  "name": "get_weather",
  "description": "获取城市天气信息",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": { "type": "string", "description": "城市名称" }
    },
    "required": ["city"]
  }
}
```

#### Python 实现

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("WeatherServer")

@mcp.tool()
def get_weather(city: str) -> str:
    """获取城市天气信息。"""
    return f"{city}: 晴，25°C"
```

#### 工具返回值类型

| 类型 | 说明 |
|------|------|
| `TextContent` | 纯文本 |
| `ImageContent` | Base64 编码的图片 |
| `AudioContent` | Base64 编码的音频 |
| `ResourceLink` | 资源链接引用 |
| `EmbeddedResource` | 内嵌资源数据 |

#### 错误处理

两种错误机制：
- **协议错误**：JSON-RPC 标准错误（未知工具、参数无效等）
- **执行错误**：工具结果中 `isError: true`（API 失败、业务逻辑错误等）

---

### 4.2 Resource（资源）

> **应用驱动 (Application-driven)**：由 Host 应用决定何时、如何使用。

Resource 为 LLM 提供**上下文数据**——文件内容、数据库 Schema、配置信息等。

#### 两种资源形式

| 形式 | 说明 | 示例 |
|------|------|------|
| **静态资源** | 固定 URI，直接列举 | `config://settings` |
| **资源模板** | 含参数的 URI 模板 | `users://{user_id}/profile` |

#### 发现与读取流程

```
1. Client → Server:  resources/list               # 列举静态资源
2. Client → Server:  resources/templates/list      # 列举资源模板
3. Client → Server:  resources/read {uri}          # 读取资源内容
4. Client → Server:  resources/subscribe {uri}     # 订阅资源变更（可选）
5. Server → Client:  notifications/resources/updated  # 变更通知
```

#### Python 实现

```python
# 静态资源
@mcp.resource("config://app-settings")
def get_settings() -> str:
    """获取应用配置。"""
    return '{"debug": true, "version": "1.0"}'

# 动态资源模板
@mcp.resource("users://{user_id}/profile")
def get_user(user_id: str) -> str:
    """根据 ID 获取用户信息。"""
    return f'{{"id": "{user_id}", "name": "张三"}}'
```

#### 常见 URI Scheme

| Scheme | 用途 |
|--------|------|
| `https://` | Web 资源（客户端可直接获取） |
| `file://` | 类文件系统资源 |
| `git://` | Git 版本控制 |
| 自定义 Scheme | 如 `config://`、`db://` 等 |

---

### 4.3 Prompt（提示词模板）

> **用户控制 (User-controlled)**：由用户主动选择和触发（如斜杠命令）。

Prompt 提供**可复用的交互模板**，结构化 LLM 的输入。

#### 发现与使用流程

```
1. Client → Server:  prompts/list                    # 列举可用模板
2. Client → Server:  prompts/get {name, arguments}   # 获取渲染后的模板
3. Server → Client:  {messages: [...]}               # 返回消息列表
```

#### Python 实现

```python
@mcp.prompt()
def code_review(code: str, focus: str = "质量") -> str:
    """生成代码审查提示词。"""
    return f"请审查以下代码，重点关注{focus}：\n\n```\n{code}\n```"
```

#### Prompt 消息中可包含的内容

- 纯文本 (`TextContent`)
- 图片 (`ImageContent`)
- 音频 (`AudioContent`)
- 内嵌资源 (`EmbeddedResource`)

---

### 4.4 三大原语对比

| 维度 | Tool | Resource | Prompt |
|------|------|----------|--------|
| **控制方** | LLM 自动调用 | 应用程序决定 | 用户主动触发 |
| **类比** | POST 端点 | GET 端点 | 预设模板 |
| **用途** | 执行操作 | 提供数据 | 结构化交互 |
| **发现方法** | `tools/list` | `resources/list` | `prompts/list` |
| **使用方法** | `tools/call` | `resources/read` | `prompts/get` |
| **动态更新** | `tools/list_changed` | `resources/list_changed` | `prompts/list_changed` |

---

## 5. 客户端原语 (Client Primitives)

除了服务端原语，MCP 还定义了客户端可以暴露给服务端的能力：

### 5.1 Sampling（采样）

让 Server **请求 Client 调用 LLM** 生成文本——Server 无需自己持有 API Key。

```
Server → Client:  sampling/createMessage   # 请求 LLM 生成
Client → 用户:    审批请求（人在回路）
Client → LLM:     转发请求
LLM → Client:     返回生成结果
Client → 用户:    审批响应
Client → Server:  返回结果
```

关键特性：
- Server 可通过 `modelPreferences` 建议模型（hints + 优先级），但**最终选择权在 Client**
- 必须有**人在回路 (Human-in-the-loop)**，用户可审批/修改/拒绝

### 5.2 Elicitation（用户交互请求）

让 Server 请求 Client **向用户获取额外信息**或确认操作。

### 5.3 Logging（日志）

让 Server 向 Client 发送**调试和监控日志**。

---

## 6. 传输方式 (Transport)

### 6.1 stdio（标准输入输出）

```
Client ──(stdin/stdout)──→ Server 子进程
```

- Client 以**子进程**方式启动 Server
- 通过 stdin/stdout 交换 JSON-RPC 消息，每条消息以换行分隔
- 适用场景：**本地通信**，零网络开销
- stderr 可用于日志输出

### 6.2 Streamable HTTP（推荐用于生产环境）

```
Client ──(HTTP POST/GET)──→ Server HTTP 端点
                            (例如 /mcp)
```

Server 提供一个统一的 HTTP 端点（如 `https://example.com/mcp`），同时支持 POST 和 GET：

| 方法 | 用途 |
|------|------|
| **POST** | 客户端发送 JSON-RPC 消息（请求/通知/响应） |
| **GET** | 客户端打开 SSE 流，监听服务端主动推送 |

响应方式：
- `Content-Type: application/json` → 单个 JSON 响应
- `Content-Type: text/event-stream` → SSE 流式响应（可包含中间消息）

#### 会话管理

```
1. Client POST InitializeRequest
2. Server 返回 InitializeResponse + Mcp-Session-Id 头
3. Client 后续请求都携带 Mcp-Session-Id 头
4. Client 结束时 DELETE 端点以关闭会话
```

#### 安全注意事项

- **必须**验证 `Origin` 头，防止 DNS 重绑定攻击
- 本地运行时**应**绑定 `127.0.0.1`，而非 `0.0.0.0`
- **应**实现认证机制（推荐 OAuth）

### 6.3 传输方式对比

| 维度 | stdio | Streamable HTTP |
|------|-------|-----------------|
| 部署方式 | 本地子进程 | 独立 HTTP 服务 |
| 网络开销 | 无 | 有 |
| 多客户端 | 通常单客户端 | 支持多客户端 |
| 适用场景 | 桌面应用、CLI 工具 | 远程服务、生产环境 |
| 认证 | 进程级隔离 | HTTP 标准认证 |

---

## 7. 通知机制 (Notifications)

MCP 支持**实时通知**，无需轮询即可感知变化：

```json
// 通知是没有 id 字段的 JSON-RPC 消息（不需要响应）
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

常见通知：

| 通知 | 触发条件 |
|------|----------|
| `notifications/tools/list_changed` | 可用工具列表变化 |
| `notifications/resources/list_changed` | 可用资源列表变化 |
| `notifications/resources/updated` | 某个已订阅资源内容更新 |
| `notifications/prompts/list_changed` | 可用提示词模板列表变化 |

---

## 8. Python SDK 使用指南

### 8.1 安装

```bash
pip install "mcp[cli]"
# 或
uv add "mcp[cli]"
```

### 8.2 服务端（FastMCP 高级 API）

```python
from mcp.server.fastmcp import FastMCP

# 创建服务器
mcp = FastMCP(
    "MyServer",
    stateless_http=True,   # 无状态模式，便于水平扩展
    json_response=True,    # 返回 JSON 而非 SSE 流
)

# 注册工具
@mcp.tool()
def add(a: int, b: int) -> int:
    """两数相加。"""
    return a + b

# 注册资源
@mcp.resource("config://settings")
def get_settings() -> str:
    """获取配置信息。"""
    return '{"version": "1.0"}'

# 注册提示词模板
@mcp.prompt()
def summarize(text: str) -> str:
    """文本摘要模板。"""
    return f"请总结以下内容：\n\n{text}"

# 启动（Streamable HTTP）
if __name__ == "__main__":
    mcp.run(transport="streamable-http")
    # 默认监听 http://127.0.0.1:8000/mcp
```

### 8.3 客户端

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client

async def main():
    async with streamable_http_client("http://localhost:8000/mcp") as (
        read_stream, write_stream, _
    ):
        async with ClientSession(read_stream, write_stream) as session:
            # 初始化连接（协议握手 + 能力协商）
            await session.initialize()

            # 列举并调用工具
            tools = await session.list_tools()
            result = await session.call_tool("add", arguments={"a": 1, "b": 2})

            # 列举并读取资源
            resources = await session.list_resources()
            content = await session.read_resource("config://settings")

            # 列举并获取提示词模板
            prompts = await session.list_prompts()
            prompt = await session.get_prompt("summarize", arguments={"text": "..."})

asyncio.run(main())
```

### 8.4 服务器生命周期管理 (Lifespan)

使用 `lifespan` 管理服务器级别的资源（如数据库连接）：

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from dataclasses import dataclass
from mcp.server.fastmcp import Context, FastMCP

class Database:
    @classmethod
    async def connect(cls) -> "Database":
        return cls()
    async def disconnect(self) -> None:
        pass
    def query(self, sql: str) -> str:
        return f"Result: {sql}"

@dataclass
class AppContext:
    db: Database

@asynccontextmanager
async def app_lifespan(server: FastMCP) -> AsyncIterator[AppContext]:
    """服务器启动时初始化，关闭时清理。"""
    db = await Database.connect()
    try:
        yield AppContext(db=db)
    finally:
        await db.disconnect()

mcp = FastMCP("MyApp", lifespan=app_lifespan)

@mcp.tool()
def query_db(sql: str, ctx: Context) -> str:
    """在工具中访问生命周期上下文。"""
    db = ctx.request_context.lifespan_context.db
    return db.query(sql)
```

---

## 9. 与 LLM 集成的完整流程

这是 MCP 在实际 AI 应用中的典型使用模式：

```
用户提问
   │
   ▼
┌──────────┐     tools/list      ┌──────────┐
│ AI 应用   │───────────────────→│MCP Server│
│ (Host)   │←───────────────────│          │
│          │   工具列表           │          │
│  ┌─────┐ │                     │          │
│  │ LLM │ │  LLM 决定调用工具    │          │
│  └──┬──┘ │                     │          │
│     │    │     tools/call      │          │
│     ▼    │───────────────────→│          │
│  处理结果 │←───────────────────│          │
│          │   工具执行结果       │          │
│  ┌─────┐ │                     │          │
│  │ LLM │ │  基于结果生成回答    │          │
│  └─────┘ │                     └──────────┘
└──────────┘
   │
   ▼
最终回答返回给用户
```

核心代码模式（伪代码）：

```python
# 1. 连接 MCP Server，获取工具
tools = await session.list_tools()
llm_tools = convert_to_llm_format(tools)

# 2. 用户提问 → LLM（携带工具定义）
response = llm.chat(messages, tools=llm_tools)

# 3. 如果 LLM 决定调用工具
if response.tool_calls:
    for call in response.tool_calls:
        # 通过 MCP 协议调用工具
        result = await session.call_tool(call.name, call.arguments)
        messages.append(tool_result(result))

    # 4. 将工具结果返回给 LLM，生成最终回答
    final_response = llm.chat(messages)
```

---

## 10. 安全最佳实践

### 服务端

- **验证**所有工具输入参数
- 实现**访问控制**和**速率限制**
- **净化**工具输出，防止注入攻击
- 验证所有资源 URI

### 客户端

- 工具调用前**提示用户确认**（人在回路）
- 调用前**展示工具输入**，防止数据泄露
- **验证**工具结果后再传给 LLM
- 实现**超时**和**日志审计**

### 传输安全

- 验证 `Origin` 头防止 DNS 重绑定
- 本地服务绑定 `127.0.0.1`
- 远程服务使用 HTTPS + OAuth 认证

---

## 11. 相关生态

| 项目 | 说明 |
|------|------|
| [MCP 规范](https://spec.modelcontextprotocol.io) | 协议完整规范 |
| [Python SDK](https://github.com/modelcontextprotocol/python-sdk) | Python 官方 SDK |
| [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) | TypeScript 官方 SDK |
| [MCP Inspector](https://github.com/modelcontextprotocol/inspector) | 调试和测试工具 |
| [MCP Servers](https://github.com/modelcontextprotocol/servers) | 官方参考 Server 实现 |
| [Awesome MCP Servers](https://github.com/punkpeye/awesome-mcp-servers) | 社区 Server 合集 |
