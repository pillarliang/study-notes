# 单文件 Ask 完整流程文档

## 1. 总体架构概览

单文件 Ask 是一个基于 **LangGraph** 的 RAG（Retrieval Augmented Generation）管道，针对单个文件/笔记内容回答用户问题。采用 SSE（Server-Sent Events）流式协议返回结果。

**核心技术栈**: FastAPI + LangGraph StateGraph + LangChain + PostgreSQL Checkpointer + Cohere Rerank

**Graph 拓扑结构**:

```
START ──┬── filter_copied ── contextualize_q ──┬── check_query ────────┬── retriever ── rerank ── qa ── END
        │                                      ├── prepare_context ────┤
        │                                      └── intent_recognition ─┘
        │
        └── process_metadata ──────────────────────────────────────────────── END (独立并行)
```

---

## 2. API 入口层

**文件**: `api/query.py`

### 2.1 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `question` | `str` | 是 | 用户问题 |
| `file_id` | `str` | 是 | 文件 ID |
| `note_id` | `str` | 否 | 笔记 ID |
| `transcription` | `str` | 否 | 转写文本（若不提供则从 DB 加载） |
| `recommend_question_category` | `str` | 否 | 推荐问题类别 |
| `skills` | `object` | 否 | 技能配置 |
| `deep_thinking` | `bool` | 否 | 是否启用深度思考 |
| `show_thinking` | `bool` | 否 | 是否展示思考过程 |
| `system_prompt` | `str` | 否 | 自定义系统 prompt |

### 2.2 请求处理流程

```
POST/GET /v2/ask
  → get_current_user()          # 鉴权
  → enrich_custom_skill()       # 富化自定义技能
  → rag_query.ask_v2(is_global=False)  # 分发到单文件模式
  → EventSourceResponse         # SSE 流式返回
```

---

## 3. Client 初始化层

### 3.1 ask_v2 分发

**文件**: `ask/rag_query.py:83`

```python
client = SingleFileAskClient(
    user_id=user_id,
    ref_doc_id=ref_doc_id,       # file_id
    original_text=original_text, # 转写文本
    note_id=note_id,
    note_content=note_content,
    language="auto",
    ...
)
return await client.ask(question=question)
```

### 3.2 SingleFileAskClient 特性

**文件**: `ask/client/single_file_ask_client.py`

| 属性 | 值 |
|------|-----|
| `storage_type` | `StorageType.POSTGRES` |
| `thread_id` | `{user_id}_{note_id or ref_doc_id}` |
| `is_single_file_ask` | `True` |
| `should_contextualize_query()` | `False` (不做查询改写) |
| `should_emit_stream_events()` | `False` (不发 retriever/rerank stream) |
| `rerank_k` | AWS 配置 `SINGLE_FILE_RERANK_K` |
| `session_id` | `""` (空字符串) |
| `retriever_strategy` | `SingleFileRetrieverStrategy` |
| `reference_formatter` | `SingleFileReferenceFormatter` (时间戳格式) |

### 3.3 类继承关系

```
BaseAskClient (ABC)
  ├── SingleFileAskClient   ← 本文档重点
  ├── GlobalAskClient
  └── ProjectAskClient
```

---

## 4. Graph State 数据结构

**文件**: `ask/client/ask_state.py:26`

### 4.1 核心 State (TypedDict)

```python
class State(TypedDict):
    input: str                              # 用户问题
    chat_history: Sequence[BaseMessage]     # 用户可见的对话历史 (HumanMessage + AIMessage)
    raw_chat_history: Sequence[BaseMessage] # 模型可见的完整历史 (含 ToolMessage)
    context: str | list[Document]           # 检索到的文档列表 / 格式化后的字符串
    answer: str                             # LLM 回答
    language: str                           # 检测到的语言
    current_time: str                       # 当前时间
    self_check_query: SelfCheckQuery        # 语言检测结果
    summary: str = ""                       # 笔记内容摘要
    transcription: str = ""                 # 转写文本
    intents: list[str] = []                 # 检测到的意图列表
    tool_calls: list[dict] = []             # 当前轮工具调用记录
```

### 4.2 ClientContext (SingleFileClientContext)

```python
class SingleFileClientContext(BaseClientContext):
    file_id: Optional[str]                  # 文件 ID
    note_id: Optional[str]                  # 笔记 ID
    note_ask_auto_summary: Optional[str]    # 自动摘要内容

class BaseClientContext(TypedDict):
    user_id: str
    is_global: bool                         # False for 单文件
    thread_id: Optional[str]
    message_id: Optional[str]               # UUID, 每次回答唯一
    skills: Optional[ClientSkill]           # 技能配置
    deep_thinking: bool
    show_thinking: bool
    app_version: Optional[tuple]
    trigger_failure: Optional[str]
    user_timezone: int
    runtime_state: Optional[State]          # QA 节点用，格式化后的 state
    session_metadata: Optional[dict]
    user_account_settings: UserAccountSettings      # {memory_enabled: bool}
    ask_related_user_personalization: AskRelatedUserPersonalization  # 记忆/个性化内容
```

### 4.3 ClientSkill

```python
class ClientSkill(TypedDict):
    name: str                               # UUID (自定义) 或 string ID (官方)
    source: Literal["ask", "action", "keep_asking"]
    type: Literal["official", "custom"]     # 可选
    args: Dict[str, str]                    # 自定义技能: {"prompt": "..."}
```

---

## 5. Graph 编译与流式执行

### 5.1 Graph 编译

**文件**: `ask/client/rag_client_v2.py:213`

```python
async def ask(self, question):
    # 1. 创建所有节点实例
    self.nodes = rag_client_v2_nodes.patch_nodes(client_instance=self, question=question)

    # 2. 构建 StateGraph
    workflow = StateGraph(state_schema=State)

    # 3. 添加节点
    workflow.add_node("prepare_context", ...)
    workflow.add_node("filter_copied", ...)
    workflow.add_node("contextualize_q", ...)
    workflow.add_node("check_query", ...)
    workflow.add_node("intent_recognition", ...)
    workflow.add_node("retriever", ...)
    workflow.add_node("rerank", ...)
    workflow.add_node("qa", ...)
    workflow.add_node("process_metadata", ...)

    # 4. 定义边（执行顺序）
    # START → filter_copied → contextualize_q ─┬→ check_query ────────┬→ retriever → rerank → qa → END
    #                                           ├→ prepare_context ────┤
    #                                           └→ intent_recognition ─┘
    # START → process_metadata → END  （独立并行分支）
```

### 5.2 Graph 输入数据

```python
config = {
    "configurable": {"thread_id": "{user_id}_{note_id}", "is_global": False},
    "file_id": ref_doc_id,
    "user_id": user_id,
    "note_id": note_id or ref_doc_id,
    "callbacks": trace_callbacks,
    "metadata": {"langfuse_tags": ["note_ask", RUNENV]}
}

input = {
    "input": question,          # "这段对话讲了什么？"
    "language": "auto",
    "current_time": "2026-03-13 10:30:00"
}
```

### 5.3 流式输出机制

**文件**: `ask/client/rag_client_v2.py:609`

采用 **后台任务 + asyncio.Queue** 模式，HTTP 断连后生成仍继续：

```python
async def _stream_with_checkpointer(...):
    # 1. 立即 yield chat_metadata 事件
    yield StreamProtocol.create_chat_metadata_event(message_id=..., session_id=...)

    # 2. 创建 Queue 和后台任务
    queue = asyncio.Queue(maxsize=500)
    background_task = asyncio.create_task(self._background_generation_task(..., queue))

    # 3. 从 queue 消费事件并 yield
    while True:
        event = await queue.get()
        if event is None: break  # 生成完成
        yield event
```

**stream_mode**: `["updates", "messages", "custom"]`
- `updates`: 节点状态更新
- `messages`: LLM 流式 token (AIMessageChunk)
- `custom`: 自定义事件 (retriever_info, rerank_info 等)

**Checkpointer**: `PlaudAsyncPostgresSaver` + `durability="exit"` (仅在 graph 结束时持久化)

---

## 6. 节点详解

### 6.1 Node 1: FilterCopied

**文件**: `ask/graph/rag_client_v2/nodes/filter_copied.py`

**功能**: 过滤无效消息

**输入 State**:
```python
{
    "chat_history": [HumanMessage(...), AIMessage(...), ...]  # 来自 checkpointer 恢复的历史
}
```

**处理逻辑**:
1. 移除 `metadata.is_copied == True` 的消息
2. 移除 `content` 为空的消息
3. 截断到最后 100 条消息

**输出 State**:
```python
{
    "chat_history": [...]  # 过滤后的消息列表，最多 100 条
}
```

---

### 6.2 Node 2: ContextualizeQuery

**文件**: `ask/graph/rag_client_v2/nodes/contextualize_q.py`

**功能**: 多轮对话中的**指代消解和查询改写**（**单文件模式下跳过**）

**设计目的**: 处理多轮对话中指代不清的问题。例如：

```text
用户第1轮: "这段对话的主题是什么？"
AI回答: "主要讨论了Q1销售策略"
用户第2轮: "那参会的人有哪些？"   ← "那"指代第1轮的对话
```

如果直接拿"那参会的人有哪些？"去检索，检索器不知道"那"指什么。`contextualize_q` 会用 LLM 将其改写为：**"这段关于Q1销售策略的对话中，参会的人有哪些？"**

**实现逻辑**（`contextualize_q.py:37-46`）:
```python
contextualize_q = RunnableBranch(
    (
        # 条件1: 没有聊天历史 OR 不需要改写 → 直接返回原问题
        lambda x: not x.get("chat_history", False)
                  or not self.client_instance.should_contextualize_query(),
        (lambda x: x["input"]),   # ← 单文件走这条路
    ),
    # 条件2: 有历史且需要改写 → 调用 LLM 改写
    prompt | llm_lite | StrOutputParser(),
)
```

**单文件行为**: `SingleFileAskClient.should_contextualize_query()` 返回 `False`，**永远走第一个分支，直接返回原始 input，不调用 LLM**。

**单文件跳过的原因**: 单文件模式下 retriever 是把全文直接传给 QA，不做语义向量检索，所以查询改写没有意义——不需要拿改写后的 query 去做向量匹配。而 Global 模式需要用 query 去 Zilliz 做向量检索，指代不清会严重影响检索质量，所以 Global 模式下会启用。

**输入 State**:
```python
{
    "input": "这段对话讲了什么？",
    "chat_history": [...],
}
```

**输出 State** (单文件):
```python
{
    "input": "这段对话讲了什么？"  # 原样返回，不改写
}
```

---

### 6.3 Node 3a: CheckQuery (与 3b、3c 并行)

**文件**: `ask/graph/rag_client_v2/nodes/check_query.py`

**功能**: 检测用户问题的语言

**执行条件**: `contextualize_q` 完成后，与 `prepare_context`、`intent_recognition` **并行执行**

**处理逻辑**:
1. 若 `language != "auto"`，直接返回
2. 若有自定义 skill，拼接 skill prompt 前 200 字符参与检测
3. 若有 app_language 且为官方 skill，使用 app_language
4. 调用 `detect_language_by_llm()` 检测语言

**输出 State**:
```python
{
    "language": "Chinese"  # 或 "English", "Japanese" 等
}
```

---

### 6.4 Node 3b: PrepareContext (与 3a、3c 并行)

**文件**: `ask/graph/rag_client_v2/nodes/prepare_context.py`

**功能**: 并行加载用户记忆和文件数据

**两个并行子任务**:

#### 子任务 A: prepare_memory_content()
1. 检查 `ask_memory` feature flag
2. 若有 skill 激活 → 禁用记忆
3. 调用 MemoryOS API:
   - `get_personalization()` → content_focus, custom_instruction, memory_enable
   - 若 memory_enabled:
     - `get_profile()` → 用户画像
     - `list_preferences()` → 阅读偏好
     - `list_projects()` → 近期关注项目
     - `list_inputs()` → 用户输入记忆
4. 组装 memory_content:
```python
{
    "memory": "user profile:\n  ...\nuser reading preference:\n  ...",
    "input_memory": "...",
    "input_memory_exceed_limit": False,
    "has_memory_content": True
}
```

#### 子任务 B: _load_mode_data()
并行加载三项数据:
```python
(language, original_text), note_content, auto_summary = await asyncio.gather(
    get_language_and_text(),       # 语言检测 + 转写文本
    get_note(),                     # 笔记内容
    get_note_ask_auto_summary()    # 自动摘要
)
```

- `get_language_and_text()`: 若有 original_text 则检测语言，否则从 DB 查询
- `get_note()`: 若有 note_id 但无 note_content，从 DB 查询
- `get_note_ask_auto_summary()`: 获取自动摘要笔记内容
- 清除 `![PLAUD NOTE](...)` 标记

**输出 State**: `{}` (不修改 state，数据存入 client_context)

---

### 6.5 Node 3c: IntentRecognition (与 3a、3b 并行)

**文件**: `ask/graph/rag_client_v2/nodes/intent_recognition.py`

**功能**: 识别用户意图

**可识别意图**:
| 意图 | 说明 |
|------|------|
| `generate_image` | 生成图片/思维导图/流程图 |
| `write_email` | 写邮件 |
| `extract_todos` | 提取待办事项 |
| `extract_conclusions` | 提取结论/摘要 |
| `extract_insights` | 提取关键洞察 |
| `composite_intent` | 复合意图（多意图同时高置信） |
| `no_category` | 无明确意图 |

**跳过条件**:
- 有 `recommend_question_category` → 直接返回 `[]`
- 有官方 skill（非 UUID）→ 返回 `[]`

**模型**: lite 模型 + structured output (`IntentRecognitionOutput`)

**超时**: 7 秒

**输出 State**:
```python
{
    "intents": ["write_email"]  # 置信度 >= 90 的意图列表，或空列表
}
```

---

### 6.6 Node 4: Retriever

**文件**: `ask/graph/rag_client_v2/nodes/retriever.py`

**前置条件**: check_query + prepare_context + intent_recognition **全部完成**

**功能**: 检索相关文档

**单文件策略** (`SingleFileRetrieverStrategy`):
```python
# 不做语义向量检索，直接将全文转为 Document
docs = generate_all_docs(
    self.client_instance.original_text,
    file_id=self.client_instance.ref_doc_id,
    page_content_has_timestamp=True
)
```

**generate_all_docs 详细过程** (`ask/ask_utils.py:72-77`):

这个函数**不做分块（chunking）**，而是将整篇转写文本包进**一个 Document**：

1. 将整篇 `original_text` 包进一个 `Document(page_content=original_text)`
2. 调用 `parse_transcript()` 逐行解析原始格式（如 `[123-456][Speaker1]你好`），提取时间戳和说话人到 metadata
3. 将 `page_content` 重新格式化（保留时间戳，因为 `page_content_has_timestamp=True`）
4. 最终返回**一个 Document 列表（通常只有 1 个元素）**

原始转写文本格式：
```text
[83000-105000][Speaker1] 今天我们讨论一下Q1的销售策略
[105000-120000][Speaker2] 好的，我先汇报一下数据
```

解析后的 Document：
```python
Document(
    page_content="[83000-105000][Speaker1]今天我们讨论一下Q1的销售策略\n[105000-120000][Speaker2]好的，我先汇报一下数据",
    metadata={
        "ref_doc_id": "file_123",
        "doc_type": "transcribe",
        "speakers": ["Speaker1", "Speaker2"],  # 提取的说话人列表
        "start_times": [83000, 105000],         # 提取的开始时间（毫秒）
        "end_times": [105000, 120000],          # 提取的结束时间（毫秒）
        "reserved_int": 0,
        "reserved_varchar": "",
        "reserved_array_varchar": [],
        "reserved_array_int": [],
    }
)
```

> **关键点**: 单文件模式下不做语义向量检索、不做文本切块，直接把全文当作一个整体传下去。这与 Global 模式不同——Global 模式会去 Zilliz 做向量检索取回多个相关片段。

**后处理**:
1. `_filter_context_documents()`: 验证 file_id 和 note_id 在数据库中存在
2. 设置 `summary` = `single_ask_summary_processor(note_content)`
3. 设置 `note_ask_auto_summary` = 处理后的自动摘要

**输出 State**:
```python
{
    "context": [Document(...)],  # 通常只有 1 个 Document（整篇全文）
    "summary": "会议讨论了Q1业绩..."  # 笔记摘要
}
```

---

### 6.7 Node 5: RerankDocuments

**文件**: `ask/graph/rag_client_v2/nodes/rerank.py`

**功能**: 对检索结果进行 Cohere 重排序

**处理逻辑** (`rerank.py:36`):

```python
if docs and len(docs) > k and settings.rerank_enabled:
    reranked_docs = rerank_text(query, docs, k)
else:
    reranked_docs = docs  # ← 单文件通常走这里
```

**单文件模式下实际跳过**: 因为 `generate_all_docs` 通常只返回 **1 个 Document**，而 `rerank_k` 一般 > 1，所以 `len(docs) > k` 为 `False`，**rerank 节点直接原样返回，不做任何处理**。

> **为什么还保留这个节点**: Graph 的拓扑是 Global/SingleFile/Project 三种模式共用的。Global 模式下从 Zilliz 检索回几十个片段，rerank 才真正发挥作用——使用 AWS Bedrock 的 Cohere Rerank v3.5 模型对文档按相关性重排序并取 top-k。

**输出 State**:
```python
{
    "context": [Document(...)]  # 单文件：原样返回
}
```

---

### 6.8 Node 6: QA (核心生成节点)

**文件**: `ask/graph/rag_client_v2/nodes/qa.py`

**功能**: 调用 LLM 生成最终回答

#### 6.8.1 System Prompt 选择优先级

按优先级从高到低依次判断，**命中即停**（`qa.py:72-148`）：

```text
1. 自定义 system_prompt (API 参数传入)
2. 自定义 skill prompt → ASK_SYSTEM_PROMPT_WITH_SKILLS.format(skills_part=...)
3. 官方 skill prompt → official_skills_prompts.{skill_name}
4. 意图匹配 prompt (仅当没有 skill 时才看 intents):
   - write_email → REWRITE_FORMAT_MAIL_PROMPT_3_0
   - extract_todos → EXTRACT_TODOS_PROMPT_TEMPLATE_3_0
   - extract_conclusions → EXTRACT_CONCLUSIONS_PROMPT_TEMPLATE_3_0
   - extract_insights → EXTRACT_INSIGHTS_PROMPT_TEMPLATE
   - key_insights → KEY_INSIGHTS_PROMPT
   - relationship_dynamics → RELATIONSHIP_DYNAMICS_PROMPT
   - subtext → SUBTEXT_PROMPT
5. 默认兜底 → Langfuse 管理的 note_ask_system_prompt (jinja2)
```

**各层级详细说明**:

**第 1 层 — 自定义 system_prompt**: 这是 API 请求参数 `system_prompt` 字段，调用方可以在请求 body 里传一段自定义 prompt 文本，完全覆盖所有默认逻辑。**普通用户使用 App 时前端不会传此参数**（传 `None`），它是给内部测试、B 端集成、或前端特殊功能场景预留的接口能力。

**第 2-3 层 — Skills**: Skills 是**用户在 App 界面上主动点选**的功能按钮（如"生成信息图"、"提取待办"），通过 API 请求参数 `skills` 传入。Skill 直接决定使用哪个预置的 system prompt。

**第 4 层 — 意图匹配**: 仅当没有 skill 时才会看 `intents` 结果。意图识别只影响 prompt 模板的选择（让 LLM 以特定风格回答），**不会直接触发工具调用**。

**第 5 层 — 默认兜底**: 大多数普通提问（"这段对话讲了什么"、"帮我总结一下"）不会触发意图识别的 90 分阈值，所以**大部分请求走的都是 Langfuse 默认 prompt**。

#### 6.8.1.1 Skills 与 IntentRecognition 的关系

**Skills 和 Intents 是两条互斥的路径，Skill 优先级始终高于 Intent**：

| 场景 | IntentRecognition 行为 | QA prompt 选择 |
| --- | --- | --- |
| 有官方 Skill（非 UUID） | 直接跳过，返回 `[]` | 用 Skill 预置 prompt |
| 有自定义 Skill（UUID） | 会执行（拼接 skill prompt 前 200 字辅助判断），但结果不被使用 | Skill 判断在 intent 之前，Skill prompt 优先 |
| 没有 Skill | 正常执行，结果用于选择 prompt 模板 | 按 intent 匹配或走默认 |

#### 6.8.1.2 image_generator 工具 vs generate_infographic Skill

这是**同一个功能的两种触发方式**，最终都调用同一个 `image_generator()` 函数生成图片：

| 对比项 | `image_generator`（工具） | `generate_infographic`（官方 Skill） |
| --- | --- | --- |
| **是什么** | 注册在 ToolRegistry 里的 LangChain Tool | 定义在 `official_skills_definition.py` 里的官方 Skill |
| **触发方式** | LLM 在运行时自主决定是否调用 | 用户在 App 界面主动点选 |
| **机制** | Agent 工具列表中包含该工具，LLM "可能会"调用 | Skill 对应的预置 prompt 会引导 LLM "一定会"调用 image_generator 工具 |
| **最终执行** | 调用 `image_generator()` → 图片生成 API → S3 → 图片 URL | 调用 `image_generator()` → 图片生成 API → S3 → 图片 URL |

#### 6.8.2 QA Chain 构造

```python
# 文档格式化
state_formatter = RunnablePassthrough.assign(
    context=partial(format_docs_for_qa, is_global=False)
)
# context 从 list[Document] 变为格式化的字符串

# Prompt 模板
qa_prompt = ChatPromptTemplate.from_messages([
    MessagesPlaceholder("raw_chat_history"),
    ("human", QA_HUMAN_PROMPT_WITH_META_INSTRUCTION),  # Global区域
    # 或 ("human", "{input}\n\n以下为额外系统提示..."),     # CN区域
])

# Agent (支持工具调用)
agent = create_agent(
    llm,
    tools=ToolRegistry.get_tools(tools_list),  # memory_update, image_generator 等
    middleware=[
        qa_system_prompt,              # 动态 system prompt
        ask_qa_direct_tool_call,       # 直接工具调用
        tool_call_wrapper,             # 工具调用包装
        tool_wrapper_handle_tool_direct_return,
        ToolCallLimitMiddleware(...),  # 工具调用次数限制
        ToolRetryMiddleware(...),      # 工具调用重试
        AgentMiddleware.from_fallback_models(...)  # 模型降级
    ],
    checkpointer=False,
)

qa_chain = qa_prompt | agent.bind(context=client_context)
```

#### 6.8.3 执行与重试

- 最多重试 3 次
- `BaseAskException` 和 `CancelledError` 不重试直接抛出

#### 6.8.4 输出 State

```python
{
    "chat_history": [
        HumanMessage("这段对话讲了什么？", metadata={"language": "Chinese", "intents": [...]}),
        AIMessage("这段对话主要讨论了...", id="uuid-xxx", metadata={
            "time": "2026-03-13T10:30:00+00:00",
            "memory": {"used": False},
            "thinking": "让我分析一下...",       # 若有 deep_thinking
            "tool_calls": [...],                 # 若使用了工具
            "rerank_info": {...}                 # rerank 统计
        })
    ],
    "raw_chat_history": [
        HumanMessage("这段对话讲了什么？"),
        AIMessage(tool_calls=[...]),            # 若有工具调用
        ToolMessage(content="...", name="memory_update"),
        AIMessage("这段对话主要讨论了...")
    ],
    "answer": "这段对话主要讨论了...",            # 格式化后的最终答案
    "tool_calls": [{"name": "memory_update", ...}]
}
```

---

### 6.9 Node 7: ProcessMetadata (独立并行)

**文件**: `ask/graph/rag_client_v2/nodes/process_metadata.py`

**功能**: 重复问题检测（与主流程独立并行执行）

**执行条件**:
- 仅单文件 ask
- 无 skill 激活
- 有非空问题

**处理逻辑**:
1. 调用 `check_and_track_duplicate(session, user_id, question)`
2. 若检测到重复 → 通过 `stream_writer` 发送通知事件

**输出**: 不修改 State，仅可能发送 custom 事件:
```json
{
    "event": "metadata",
    "data": "{\"type\": \"NOTIFY_SAVE_QUERY_AS_SKILL\", \"message\": \"You've asked similar questions before...\"}"
}
```

---

## 7. SSE 流式协议

**文件**: `ask/streaming_protocol.py`

所有事件均为 `dict` 格式：`{"event": "type", "data": "json_string"}`

### 7.1 事件类型与顺序

```
1. chat_metadata   ← 立即返回，包含 message_id
2. thinking        ← (可选) deep_thinking 模式下的推理过程
3. answer          ← 多次，LLM 流式 token
4. metadata        ← (可选) memory 使用通知 / 重复问题通知
5. reference       ← (可选) 引用来源，解析自回答中的标签
6. media           ← (可选) 图片等媒体
7. tool_call       ← (可选) 工具调用事件
8. end / error     ← 结束或错误
```

### 7.2 各事件数据格式

**chat_metadata**:
```json
{
    "event": "chat_metadata",
    "data": "{\"message_id\":\"uuid-xxx\",\"timestamp\":\"10:30:00\",\"session_id\":\"\"}"
}
```

**thinking**:
```json
{
    "event": "thinking",
    "data": "{\"content\":\"让我分析一下这段对话的主要内容...\"}"
}
```

**answer**:
```json
{"event": "answer", "data": "{\"content\":\"这段对话主要\"}"}
{"event": "answer", "data": "{\"content\":\"讨论了三个议题：\"}"}
```

**reference** (单文件格式 — 时间戳引用):
```json
{
    "event": "reference",
    "data": "{\"type\":\"transcribe\",\"file_id\":\"abc123\",\"start_time\":83000}"
}
```

**metadata** (记忆使用通知):
```json
{
    "event": "metadata",
    "data": "{\"type\":\"memory\",\"data\":{\"used\":true}}"
}
```

**metadata** (重复问题通知):
```json
{
    "event": "metadata",
    "data": "{\"type\":\"NOTIFY_SAVE_QUERY_AS_SKILL\",\"message\":\"You've asked similar questions before...\"}"
}
```

**error**:
```json
{
    "event": "error",
    "data": "{\"id\":-100,\"code\":500,\"message\":\"Unexpected error occurred\",\"error_data\":{}}"
}
```

**end**:
```json
{"event": "end", "data": "{}"}
```

---

## 8. 完整数据流转图

```text
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ API Layer                                                                            │
│  Request: {question, file_id, note_id, transcription, skills, deep_thinking}        │
│  → SingleFileAskClient.ask(question)                                                 │
│  ← SSE Stream: chat_metadata → thinking* → answer* → reference* → end              │
└─────────────────────────────┬───────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────────────────────┐
│ Graph Input                                                                          │
│  {"input": "问题", "language": "auto", "current_time": "2026-03-13 10:30:00"}       │
│  + Checkpointer 恢复的 chat_history                                                  │
└─────────────────────────────┬───────────────────────────────────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   filter_copied     │  过滤空/copied 消息，截断到 100 条
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  contextualize_q    │  多轮对话指代消解（单文件跳过，原样返回）
                    └──┬──────┬──────┬───┘
                       │      │      │          ← 三路并行
           ┌───────────▼┐ ┌──▼──────▼──┐ ┌────▼────────────┐
           │ check_query │ │prepare_ctx │ │intent_recognition│
           │→language:   │ │→memory:    │ │→intents:        │
           │  "Chinese"  │ │  {profile, │ │  ["write_email"]│
           │             │ │   prefs}   │ │  或 []          │
           │             │ │→load_data: │ │                 │
           │             │ │  text+note │ │                 │
           └──────┬──────┘ └─────┬──────┘ └──────┬──────────┘
                  │              │                │
                  └──────────────┼────────────────┘
                                 │          ← 三路汇聚
                    ┌────────────▼───────────┐
                    │       retriever         │
                    │  全文 → 1个 Document    │  （单文件：不分块、不向量检索）
                    │  + filter + summary     │
                    └────────────┬───────────┘
                                 │
                    ┌────────────▼───────────┐
                    │       rerank           │  （单文件：实际跳过，仅 1 个 doc < rerank_k）
                    └────────────┬───────────┘
                                 │
                    ┌────────────▼───────────┐
                    │         qa             │
                    │  format_docs → string  │
                    │  选择 system_prompt:   │
                    │    自定义 > skill >    │
                    │    intent > 默认       │
                    │  agent(llm + tools)    │
                    │  → 流式 answer tokens  │
                    │  → reference 解析      │
                    │  → chat_history 更新   │
                    └────────────┬───────────┘
                                 │
                    ┌────────────▼───────────┐
                    │         END            │
                    │  Checkpointer 持久化   │
                    │  → PostgreSQL          │
                    └────────────────────────┘
```

---

## 9. 完整流程文字描述

### 9.1 入口

单文件 Ask 的入口是 `POST/GET /v2/ask`，鉴权后创建 `SingleFileAskClient`，调用 `client.ask(question)` 构建并执行 LangGraph。Graph 启动后分为**主分支**和**独立并行分支**两条路径。

### 9.2 主分支

**Step 1 — filter_copied**

过滤历史消息中标记为 `is_copied` 或内容为空的消息，截断到最近 100 条。

**Step 2 — contextualize_q**

多轮对话查询改写节点。设计目的是处理"那个怎么样"这类指代不清的问题，用 LLM 改写为完整独立的问句。**单文件模式下跳过**（因为不做语义向量检索，改写无意义），直接返回原始问题。

**Step 3 — 三路并行**

以下三个节点同时启动（Graph 拓扑固定，不会根据运行时参数跳过节点，但节点内部可能提前 return）：

- **check_query** — 用 LLM 检测问题语言（如 Chinese/English）。
- **prepare_context** — 并行加载两组数据：MemoryOS 用户记忆（画像/偏好/项目）+ 文件数据（转写文本/笔记内容/自动摘要）。
- **intent_recognition** — 从预定义的 6 个意图中匹配用户问题，详见下方说明。

> **intent_recognition 详细说明**：
>
> 用 LLM + structured output 对用户问题打分，判断它属于哪个预定义意图（`generate_image`、`write_email`、`extract_todos`、`extract_conclusions`、`extract_insights`、`composite_intent`），每个意图返回 0-100 的置信度分数，只保留 >= 90 分的。识别结果在后续 QA 节点中用于选择 system prompt 模板。
>
> **有官方 Skill 时（name 为可读字符串如 `"image_generator"`）**：节点内部第一行就判断，直接 `return {"intents": []}` 不调用 LLM。因为用户已通过按钮明确表达意图，不需要 AI 再猜。
>
> **有自定义 Skill 时（name 为 UUID 如 `"a1b2c3d4-..."`）**：意图识别**正常执行**，会调用 LLM。还会把自定义 skill 的 prompt 前 200 字符拼接到用户问题后面一起送去做意图分类（为了更准确判断）。但识别出的结果**不会被 QA 使用**——因为 QA 的 `get_system_prompt()` 中 Skill 判断在 Intent 之前，有 Skill 就直接 return 用 Skill prompt 了。所以自定义 Skill 时意图识别属于"做了但白做"。
>
> **没有 Skill 时**：正常执行意图识别，结果在 QA 节点中用于选择 prompt 模板。
>
> **如何区分官方/自定义 Skill**：前端传入的 `skills.name` 如果是可读字符串 ID 就是官方 Skill，如果是 UUID 就是自定义 Skill。API 层的 `enrich_custom_skill()` 用 `is_uuid_regex()` 判断并打上 `type = "official"/"custom"` 标记，自定义 Skill 还会从数据库查出对应的 prompt 填入 `args.prompt`。

**Step 4 — retriever**

三路汇聚后执行。单文件模式下**不做向量检索、不做文本切块**，而是把整篇转写文本通过 `generate_all_docs()` 解析为一个 Document（提取时间戳、说话人到 metadata），直接作为上下文传递。

**Step 5 — rerank**

Cohere 重排序节点。因为只有 1 个 Document 且小于 rerank_k 阈值，**实际跳过**，原样传递。

**Step 6 — qa（核心生成节点）**

分为两个阶段：

**(A) System Prompt 选择**（按优先级从高到低，命中即停）：

1. **自定义 system_prompt** — API 请求参数 `system_prompt` 字段。调用方可在请求 body 里直接传一段 prompt 文本覆盖所有默认逻辑。这不是用户在 App 里输入的问题，而是给内部测试、B 端集成、或前端特殊功能场景预留的接口能力，普通用户使用 App 时前端不传此参数。
2. **Skill prompt** — Skills 是用户在 App 界面上**主动点选**的功能按钮，通过 API 参数 `skills` 传入。包含两种子类型，都在同一优先级，命中任一即 return：
   - **官方 Skill**（name 为可读字符串如 `"image_generator"`）→ 使用代码中预置的 `official_skills_prompts.{skill_name}`
   - **自定义 Skill**（name 为 UUID）→ 使用数据库中查出的用户自己编写的 prompt，塞进 `ASK_SYSTEM_PROMPT_WITH_SKILLS` 包装模板。若数据库中 prompt 为空，则不命中，继续往下走到意图匹配或默认。
3. **意图匹配 prompt** — 仅当没有 Skill 时才看 `intents` 结果，通过 `match` 匹配不同意图对应不同的专用 prompt 模板（如 `write_email` → 邮件撰写 prompt，`extract_todos` → 待办提取 prompt）。意图识别只影响 prompt 选择，**不会直接触发工具调用**。注意：**并非所有意图都有对应的 prompt 分支**，如 `generate_image` 没有对应 `case`，会走 `case _: pass` 落到默认。
4. **Langfuse 默认 prompt** — 以上都没命中时的兜底。大多数普通提问（"这段对话讲了什么"）不会触发意图识别的 90 分阈值，所以**大部分请求走的都是默认 prompt**。

**(B) LLM 生成**

选好 prompt 后，将 Document 格式化为文本字符串注入 prompt，通过 Agent（LLM + memory_update / image_generator 等工具）流式生成回答。工具调用由 LLM Agent 在运行时自主决定，不受意图识别结果直接控制。

### 9.3 Skills 与 Intents 的关系

两者是**完全不同的来源、互斥的路径**：

| 维度 | Skills | Intents |
| --- | --- | --- |
| **来源** | 用户在 App 界面**主动点选** | 系统自动用 LLM 分析问题文本识别 |
| **传入方式** | API 参数 `skills` | intent_recognition 节点输出 |
| **优先级** | 高（QA 中先判断 Skill，命中直接 return） | 低（仅当没有 Skill 时才看） |
| **有官方 Skill 时** | intent_recognition 节点内部直接返回 `[]`，不调 LLM | 结果不产生 |

**以"生图"为例对比两种触发路径**：

| 触发方式 | 流程 |
| --- | --- |
| 用户**点了"生成信息图"Skill 按钮** | → Skill prompt 引导 LLM **一定会**调用 `image_generator` 工具 |
| 用户**只打字"帮我画个思维导图"** | → 意图识别出 `generate_image`，但无对应 prompt 分支，走默认 prompt → LLM Agent **可能会**自主调用 `image_generator` 工具 |

两种方式殊途同归，最终都调用同一个 `image_generator()` 函数（调图片生成 API → 上传 S3 → 返回图片 URL）。

### 9.4 独立并行分支

**process_metadata** — 与主分支同时启动。检测重复问题，命中则发送"保存为 Skill"的通知。不修改 State，不阻塞主流程。

### 9.5 流式输出与持久化

流式输出通过后台任务 + asyncio.Queue 推送 SSE 事件（chat_metadata → thinking → answer → reference → end），HTTP 断连后生成仍继续。整轮对话结束后 State 通过 PostgreSQL Checkpointer 持久化，供下轮多轮对话恢复。

---

## 10. 关键文件索引

| 组件 | 文件路径 |
|------|----------|
| API 入口 | `api/query.py` |
| 请求分发 | `ask/rag_query.py` |
| 单文件 Client | `ask/client/single_file_ask_client.py` |
| 基础 Client | `ask/client/base_ask_client.py` |
| Graph 编译 & 流式 | `ask/client/rag_client_v2.py` |
| State 定义 | `ask/client/ask_state.py` |
| 节点工厂 | `ask/graph/rag_client_v2/nodes/__init__.py` |
| FilterCopied | `ask/graph/rag_client_v2/nodes/filter_copied.py` |
| ContextualizeQuery | `ask/graph/rag_client_v2/nodes/contextualize_q.py` |
| CheckQuery | `ask/graph/rag_client_v2/nodes/check_query.py` |
| PrepareContext | `ask/graph/rag_client_v2/nodes/prepare_context.py` |
| IntentRecognition | `ask/graph/rag_client_v2/nodes/intent_recognition.py` |
| Retriever | `ask/graph/rag_client_v2/nodes/retriever.py` |
| Rerank | `ask/graph/rag_client_v2/nodes/rerank.py` |
| QA | `ask/graph/rag_client_v2/nodes/qa.py` |
| ProcessMetadata | `ask/graph/rag_client_v2/nodes/process_metadata.py` |
| 流式协议 | `ask/streaming_protocol.py` |
| 检索策略 | `ask/client/strategies/retriever_strategy.py` |
| 引用格式化 | `ask/client/strategies/formatter_strategy.py` |
| PG Checkpointer | `ask/memory/postgres_saver_async.py` |
