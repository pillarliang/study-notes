# MCP (Model Context Protocol) 学习笔记

> MCP 是由 Anthropic 推出的**开放协议**，旨在标准化 LLM 应用与外部数据源、工具之间的交互方式。
> 可以把它理解为 **AI 领域的 USB-C**——一个统一的接口标准。
>
> 官方文档：[https://modelcontextprotocol.io](https://modelcontextprotocol.io)

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


| 角色         | 职责                               | 示例                                   |
| ---------- | -------------------------------- | ------------------------------------ |
| **Host**   | AI 应用本身，协调和管理多个 Client           | Claude Desktop, VS Code, Claude Code |
| **Client** | 维护与某个 Server 的连接，获取上下文           | Host 内部为每个 Server 创建的客户端实例           |
| **Server** | 暴露 Tool / Resource / Prompt 的服务端 | 文件系统 Server、数据库 Server、Sentry Server |


### 2.2 协议分层

MCP 由两层组成，类比"写信"和"寄信"：

- **数据层**：定义消息的**内容格式**（信怎么写）—— 使用 JSON-RPC 2.0
- **传输层**：定义消息的**传递通道**（信怎么寄）—— 使用 HTTP 或 stdio

同一条 JSON-RPC 消息，既可以通过 HTTP 发送（远程），也可以通过 stdin/stdout 发送（本地），**消息内容完全不变**。

```
┌──────────────────────────────────────┐
│    数据层 (Data Layer) —— 消息内容     │
│  JSON-RPC 2.0 格式                    │
│  • 生命周期管理（握手、能力协商）        │
│  • 服务端原语：Tool / Resource / Prompt │
│  • 客户端原语：Sampling / Elicitation   │
│  • 通知 & 进度追踪                     │
├──────────────────────────────────────┤
│    传输层 (Transport Layer) —— 传递通道 │
│  • stdio（本地子进程的 stdin/stdout）    │
│  • Streamable HTTP（HTTP POST/GET）    │
└──────────────────────────────────────┘
```

### 2.3 底层通信格式：JSON-RPC 2.0

MCP 所有消息都基于 [JSON-RPC 2.0](https://www.jsonrpc.org/) 格式。你不需要手动构造这些 JSON（SDK 封装好了），但理解它有助于看懂协议的运作方式。

JSON-RPC 只有**三种消息**：

**请求 (Request)**——有 `id` 字段，发出后**等待对方回复**：

```json
{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}
```

**响应 (Response)**——也有 `id` 字段，`id` 和请求一一对应，用于**配对问答**：

```json
{"jsonrpc": "2.0", "id": 1, "result": {"tools": [...]}}
```

> 为什么需要 `id`？因为可以同时发多个请求，响应可能乱序到达。
> 比如你同时发了 `id=1`（列工具）和 `id=2`（调工具），Server 可能先回 `id=2` 再回 `id=1`，靠 `id` 匹配。

**通知 (Notification)**——**没有 `id`**，发完就完，**不需要也不会收到回复**：

```json
{"jsonrpc": "2.0", "method": "notifications/tools/list_changed"}
```

简单总结：**请求/响应是打电话（一问一答），通知是发短信（说完就完）。**

> 使用 Python SDK 时你不需要手动构造这些 JSON，SDK 全部封装好了。
> 这里介绍 JSON-RPC 是为了理解后面章节中"请求""响应""通知"这些术语的含义。

---

## 3. 连接生命周期

MCP 是**有状态协议**，连接需要经过完整的生命周期管理：

```
Client                          Server
  │                                │
  │──── initialize ──────────────→│   1. 【请求】发送协议版本 + 客户端能力（有 id，等回复）
  │←─── initialize result ───────│   2. 【响应】返回服务端能力（id 对应上面的请求）
  │──── initialized ─────────────→│   3. 【通知】确认初始化完成（无 id，不等回复）
  │                                │
  │        ═══ 正常通信 ═══         │   4. 双向请求/响应/通知
  │                                │
  │──── 断开连接 ─────────────────→│   5. 关闭
```

在 Python SDK 中，这整个过程（含下面的能力协商）被封装在一行代码里：

```python
await session.initialize()
# 背后自动完成了：握手 3 步 + 能力协商，全部在这一次调用中
```

### 能力协商 (Capability Negotiation)

能力协商**不是额外的步骤**，它就发生在上面第 1、2 步的 `initialize` 请求和响应中——双方在握手消息里各自声明"我支持什么"。

之后的通信**只能使用双方都支持的功能**。比如：Server 声明支持 `tools`，但没声明 `prompts`，那 Client 就不应该调用 `prompts/list`。

```json
// 客户端声明的能力（"我能帮你做这些"）
{
  "capabilities": {
    "sampling": {},        // 我可以帮你调 LLM
    "elicitation": {}      // 我可以帮你问用户
  }
}

// 服务端声明的能力（"我能提供这些"）
{
  "capabilities": {
    "tools": { "listChanged": true },     // 我有工具，而且工具列表变化时会通知你
    "resources": { "subscribe": true },    // 我有资源，你可以订阅变更
    "prompts": { "listChanged": true }     // 我有提示词模板
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


| 类型                 | 说明           |
| ------------------ | ------------ |
| `TextContent`      | 纯文本          |
| `ImageContent`     | Base64 编码的图片 |
| `AudioContent`     | Base64 编码的音频 |
| `ResourceLink`     | 资源链接引用       |
| `EmbeddedResource` | 内嵌资源数据       |


#### 错误处理

两种错误机制：

- **协议错误**：JSON-RPC 标准错误（未知工具、参数无效等）
- **执行错误**：工具结果中 `isError: true`（API 失败、业务逻辑错误等）

---

### 4.2 Resource（资源）

> **应用驱动 (Application-driven)**：由 Host 应用决定何时、如何使用。

Resource 为 LLM 提供**上下文数据**——文件内容、数据库 Schema、配置信息等。

#### 两种资源形式


| 形式       | 说明          | 示例                          |
| -------- | ----------- | --------------------------- |
| **静态资源** | 固定 URI，直接列举 | `config://settings`         |
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


| Scheme     | 用途                      |
| ---------- | ----------------------- |
| `https://` | Web 资源（客户端可直接获取）        |
| `file://`  | 类文件系统资源                 |
| `git://`   | Git 版本控制                |
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


| 维度       | Tool                 | Resource                 | Prompt                 |
| -------- | -------------------- | ------------------------ | ---------------------- |
| **控制方**  | LLM 自动调用             | 应用程序决定                   | 用户主动触发                 |
| **类比**   | POST 端点              | GET 端点                   | 预设模板                   |
| **用途**   | 执行操作                 | 提供数据                     | 结构化交互                  |
| **发现方法** | `tools/list`         | `resources/list`         | `prompts/list`         |
| **使用方法** | `tools/call`         | `resources/read`         | `prompts/get`          |
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
- 必须有 Human-in-the-loop，用户可审批/修改/拒绝

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


| 方法       | 用途                          |
| -------- | --------------------------- |
| **POST** | 客户端发送 JSON-RPC 消息（请求/通知/响应） |
| **GET**  | 客户端打开 SSE 流，监听服务端主动推送       |


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


| 维度   | stdio       | Streamable HTTP |
| ---- | ----------- | --------------- |
| 部署方式 | 本地子进程       | 独立 HTTP 服务      |
| 网络开销 | 无           | 有               |
| 多客户端 | 通常单客户端      | 支持多客户端          |
| 适用场景 | 桌面应用、CLI 工具 | 远程服务、生产环境       |
| 认证   | 进程级隔离       | HTTP 标准认证       |


---

## 7. 通知机制 (Notifications)

MCP 支持**实时通知**，无需轮询即可感知变化。

通知就是 2.3 节介绍的第三种 JSON-RPC 消息——**没有 `id`，不需要回复**，Server 单方面告知 Client：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
  // 注意：没有 "id" 字段
}
```

常见通知：


| 通知                                     | 触发条件        |
| -------------------------------------- | ----------- |
| `notifications/tools/list_changed`     | 可用工具列表变化    |
| `notifications/resources/list_changed` | 可用资源列表变化    |
| `notifications/resources/updated`      | 某个已订阅资源内容更新 |
| `notifications/prompts/list_changed`   | 可用提示词模板列表变化 |


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

## 11. Tool 调用与流式输出的关系

这里有两个层面容易混淆，需要先区分清楚：

```
传输层（Streamable HTTP）：消息怎么在网络上走 → SSE 是传输管道
协议层（JSON-RPC/MCP）：  消息的语义         → tools/call 的 result 必须完整
```

**结论先行：** `tools/call` 的**结果内容**不能流式拆分（协议层约束），但传输这个结果的管道本身（SSE）可以同时承载进度通知。

### 11.1 `tools/call` 是请求-响应模式（协议层）

`tools/call` 是标准的 JSON-RPC 请求-响应（第 2.3 节的"打电话，一问一答"）：

```json
// 请求（有 id，等回复）
{"jsonrpc": "2.0", "id": 42, "method": "tools/call",
 "params": {"name": "web_search", "arguments": {"query": "..."}}}

// 响应（id=42 配对，一次性返回完整结果）
{"jsonrpc": "2.0", "id": 42, "result": {"content": [...], "isError": false}}
```

`id=42` 的请求只能对应**一个**完整的 `id=42` 响应，不能把 result 的内容拆碎分多次返回——这是 JSON-RPC **协议语义**层面的约束，与传输方式无关。

### 11.2 Streamable HTTP 传输 `tools/call` 的实际过程（传输层）

虽然 result 必须完整，但 **SSE 确实是传输 `tools/call` 响应的管道**。客户端发一个 POST，服务端用 SSE stream 回应，stream 里可以包含多个 event：

```
Client                                  Server
  |                                        |
  |── POST /mcp (tools/call, id=42) ──→   |
  |                                        | 工具执行中...
  |←── SSE: notifications/progress(10%) ──|  ← 进度通知（无 id）
  |←── SSE: notifications/progress(80%) ──|  ← 进度通知（无 id）
  |                                        | 执行完毕
  |←── SSE: {"id":42,"result":{完整内容}} ─|  ← 唯一的配对响应
  |←── SSE stream 关闭 ───────────────── |
```

对应的 SSE 报文如下：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progress":10,"total":100}}

data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progress":80,"total":100}}

data: {"jsonrpc":"2.0","id":42,"result":{"content":[{"type":"text","text":"搜索结果..."}],"isError":false}}
```

关键点：

- SSE 里**有且仅有一个** `id=42` 的 result event（配对响应）
- 其余 event 都是**没有 id 的 notification**（进度更新、列表变更等）
- **tool 的输出内容本身没有流式**——不存在 `result_chunk` 这种拆分机制

```
SSE 能做的：在工具跑完之前，通过 notifications/progress 推送进度
SSE 不能做的：把 tool 的 result 内容拆成多块逐步返回
```

### 11.3 三种能力对比

| 能力 | 支持？ | 机制 |
|------|--------|------|
| Tool 输出内容分块流式 | ❌ 不支持 | 协议层禁止拆分 result |
| 工具运行中的进度更新 | ✅ 支持 | `notifications/progress`（通过 SSE 传输） |
| 调用工具前后 LLM 文本流式 | ✅ 支持 | LLM API 层，与 MCP 无关 |

### 11.4 流式发生在 Client 侧的 LLM 调用

在典型的 Agent 架构中（如第 9 节），流式输出发生在 **Client 侧调用 LLM** 的环节，而不是 MCP tool 返回结果的环节：

```
用户提问
  ↓
LLM 流式生成 ──→ "我需要调用 web_search"     ← ✅ 流式（LLM API）
  ↓
session.call_tool("web_search", {...})        ← ❌ 非流式（MCP 协议，阻塞等待）
  ↓ 等待完整结果
tool_output = "搜索结果..."
  ↓
LLM 流式生成 ──→ "根据搜索结果，..."           ← ✅ 流式（LLM API）
  ↓
最终回答流式展示给用户
```

对应代码：

```python
# ✅ 流式：LLM 生成文本
async for chunk in llm.astream(messages):
    print(chunk.content, end="", flush=True)

# ❌ 非流式：MCP tool 调用，等完整结果
result = await session.call_tool(tool_name, arguments=tool_args)
tool_output = result.content[0].text
```

### 11.5 Server 内部调 LLM 的 Tool

有些 MCP Tool 在 Server 内部调用 LLM（例如 `draft_email` 用 LLM 把笔记润色为邮件）。这种情况下：

- Server 内部即使用流式调 LLM，也**无法流式返回给 Client**（受 `tools/call` 协议限制）
- 所以 Server 内部通常直接用非流式调用，等 LLM 完整生成后一次性返回
- 这是合理的设计选择，不是缺陷

### 11.6 长时间任务的替代方案：Task Polling

对于耗时较长的 tool（如深度研究），可以用**任务轮询**模式绕开阻塞：

```
1. Client → tools/call("deep_research", {query: "..."})
2. Server → 立即返回 {taskId: "task-abc123"}         # 不阻塞
3. Client → tasks/get({taskId: "task-abc123"})        # 轮询
4. Server → {status: "running"}
5. Client → tasks/get({taskId: "task-abc123"})        # 再次轮询
6. Server → {status: "completed", result: {...}}      # 完成
```

本质是把一个长操作拆成"启动"和"查询结果"两次独立的请求-响应。

> **注意：** 上面的 `tasks/get` 是伪代码示意，MCP 协议**没有内置**异步任务机制。两步都是普通的 `session.call_tool()`，"启动"和"查询"是 Server 端自定义的两个工具名称。

#### Python 实现示例

**Server 端（异步执行 + 内存状态存储）：**

```python
import asyncio, uuid, json
from mcp.server import Server

app = Server("research-server")
task_store: dict = {}  # 简单内存存储

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "start_deep_research":
        task_id = str(uuid.uuid4())
        task_store[task_id] = {"status": "running", "result": None}
        # 后台异步执行，立即返回 task_id，不阻塞
        asyncio.create_task(_run_research(task_id, arguments["query"]))
        return [{"type": "text", "text": task_id}]

    elif name == "get_task_status":
        task = task_store.get(arguments["task_id"], {"status": "not_found"})
        return [{"type": "text", "text": json.dumps(task)}]

async def _run_research(task_id: str, query: str):
    await asyncio.sleep(10)  # 模拟耗时操作
    task_store[task_id] = {"status": "completed", "result": f"{query} 的研究结果..."}
```

**Client 端（轮询直到完成）：**

```python
import asyncio, json
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def run_with_polling():
    async with stdio_client(StdioServerParameters(command="python", args=["server.py"])) as (r, w):
        async with ClientSession(r, w) as session:
            await session.initialize()

            # 第 1 步：启动任务，立即拿到 task_id
            result = await session.call_tool("start_deep_research", {"query": "MCP 协议"})
            task_id = result.content[0].text
            print(f"任务已启动，task_id = {task_id}")

            # 第 2 步：轮询直到完成
            while True:
                status_result = await session.call_tool("get_task_status", {"task_id": task_id})
                task = json.loads(status_result.content[0].text)

                if task["status"] == "completed":
                    print(f"完成！结果：{task['result']}")
                    break
                elif task["status"] == "running":
                    print("还在跑，3 秒后再查...")
                    await asyncio.sleep(3)
                else:
                    print(f"异常状态：{task['status']}")
                    break
```

**伪代码与实际方法的对应关系：**

| 笔记伪代码 | 实际 Python 方法 |
|------------|-----------------|
| `tools/call("deep_research")` | `session.call_tool("start_deep_research", {...})` |
| `tasks/get({taskId})` | `session.call_tool("get_task_status", {"task_id": ...})` |

### 11.7 总结

| 环节 | 是否流式 | 原因 |
|------|---------|------|
| Client 侧 LLM → 用户 | ✅ 可流式 | LLM API 支持 streaming |
| tools/call 结果内容 | ❌ 不可拆分 | JSON-RPC 协议层：result 必须完整 |
| tools/call 运行中进度 | ✅ 可推送 | `notifications/progress` 通过 SSE 传输 |
| Streamable HTTP 传输 tools/call | SSE 管道 | POST → SSE stream（含 notifications + 最终 result） |
| MCP Server 内部 LLM 调用 | 无意义 | 即使内部流式，也无法流式返回给 Client |
| 长时间任务 | 非阻塞 | Task polling 模式替代 |

---

## 12. 相关生态


| 项目                                                                       | 说明                |
| ------------------------------------------------------------------------ | ----------------- |
| [MCP 规范](https://spec.modelcontextprotocol.io)                           | 协议完整规范            |
| [Python SDK](https://github.com/modelcontextprotocol/python-sdk)         | Python 官方 SDK     |
| [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) | TypeScript 官方 SDK |
| [MCP Inspector](https://github.com/modelcontextprotocol/inspector)       | 调试和测试工具           |
| [MCP Servers](https://github.com/modelcontextprotocol/servers)           | 官方参考 Server 实现    |
| [Awesome MCP Servers](https://github.com/punkpeye/awesome-mcp-servers)   | 社区 Server 合集      |

