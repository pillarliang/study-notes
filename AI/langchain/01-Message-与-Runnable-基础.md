# LangChain Message 与 Runnable 基础

> 范围:Message content 的合法形态、与 OpenAI 原生格式的对应关系、Runnable 的 `with_config` 用法。

---

## 1. 原理:Message content 是「松类型」的

LangChain 的 `Message` 把多模态数据塞进一个 `content` 字段。这个字段的类型设计上是**松类型**(loosely-typed),目的是同时兼容两种世界:

- **provider-native 格式** — 直接放 OpenAI / Anthropic 原生那套结构,不做翻译;
- **LangChain 标准格式** — 自家定义的 content block,跨 provider 通用。

由此可推出 `HumanMessage(content=...)` 接受三种形态:

| 形态 | 写法 | 适用场景 |
|---|---|---|
| 字符串 | `HumanMessage("hello")` | 纯文本,最常见 |
| `list[dict]`,provider-native | `[{"type": "image_url", "image_url": {...}}]` | 直接对接某个 provider |
| `list[ContentBlock]`,LangChain 标准 | 通过 `content_blocks=` 传入 | 想要 provider 无关 |

### LangChain 标准 content block 的 `type` 全集

```python
# 协议层面所有合法 type(并非每种都适合 HumanMessage)
{"text", "reasoning",
 "image", "audio", "video", "file", "text-plain",
 "tool_call", "tool_call_chunk", "invalid_tool_call",
 "server_tool_call", "server_tool_call_chunk", "server_tool_result",
 "non_standard"}
```

各 block 的字段骨架(节选自官方 `Content block reference`):

| `type` | 必要字段 | 可选字段 |
|---|---|---|
| `text` | `text: str` | `annotations`, `extras` |
| `image` | `url` **或** `base64` | `mime_type`(base64 时必填)、`id` |
| `audio` | 同 image | 同 image |
| `video` | 同 image | 同 image |
| `file` | 同 image | 同 image(MIME 例:`application/pdf`) |
| `text-plain` | `text`, `mime_type` | — |
| `reasoning` | `reasoning: str` | `extras`(如 Anthropic 的 `signature`) |

### `HumanMessage` 语义上常用的子集

虽然 content 字段是松类型,可以塞任何 block,但语义上 `HumanMessage` 代表「用户输入」,只有这几类合理:

- `text`
- `image`
- `audio`
- `video`
- `file`
- `text-plain`

`reasoning` / `tool_call*` / `server_tool_*` 是模型/工具的产物,属于 `AIMessage` 与 `ToolMessage`,不要手工塞进 `HumanMessage`。

### 示例

```python
from langchain.messages import HumanMessage

# 形态 1: 字符串
HumanMessage("What is machine learning?")

# 形态 2: provider-native (OpenAI Chat Completions 风格)
HumanMessage(content=[
    {"type": "text", "text": "What's in this image?"},
    {"type": "image_url",
     "image_url": {"url": "https://example.com/cat.jpg", "detail": "high"}},
])

# 形态 3: LangChain 标准 content blocks
HumanMessage(content_blocks=[
    {"type": "text",  "text": "What's in this image?"},
    {"type": "image", "url":  "https://example.com/cat.jpg"},
])
```

> 若希望 content 字段在序列化时也用标准 block(便于跨进程传递),设置环境变量 `LC_OUTPUT_VERSION=v1`,或 `init_chat_model(..., output_version="v1")`。

---

## 2. OpenAI 官方文档里 `image_url` 的两个位置

LangChain 的 provider-native 写法,实际上就是抄 OpenAI 的格式。但 OpenAI 自己有**两套 API**,`image_url` 在两套里**形态不同**,极易混淆。

### 2.1 Chat Completions API(老接口)

文档位置:`API Reference → Chat → Create chat completion` 的 **Chat Completion Content Parts** 章节(`ChatCompletionContentPartImage`)。

- `type` 固定为 `"image_url"`
- `image_url` 是一个 **object**

```json
{
  "type": "image_url",
  "image_url": {
    "url":    "https://example.com/cat.jpg",
    "detail": "high"
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `type` | `"image_url"` | 固定 |
| `image_url.url` | string | 图片 URL **或** base64 data URL(`data:image/png;base64,...`) |
| `image_url.detail` | `"auto" \| "low" \| "high"` | 可选,控制视觉编码精度 |

LangChain `HumanMessage` 用 provider-native 格式时跟随的就是这套写法。

### 2.2 Responses API(新接口,推荐)

文档位置:`API Reference → Responses → Create model response` + `Guides → Images`(`ResponseInputImage`)。

- `type` 改为 `"input_image"`
- `image_url` 退化成 **string**

```json
{
  "type": "input_image",
  "image_url": "https://example.com/cat.jpg",
  "detail": "auto"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `type` | `"input_image"` | 固定 |
| `image_url` | string(可选) | URL 或 base64 data URL |
| `file_id` | string(可选) | 已上传文件的 ID,与 `image_url` 二选一 |
| `detail` | `"high" \| "low" \| "auto" \| "original"` | 默认 `auto` |

### 一句话区分

- Chat Completions:`type: "image_url"`,`image_url` 是 **object**。
- Responses:`type: "input_image"`,`image_url` 是 **string**。

---

## 3. 推论:text + image 同一条 message 发送 = 一次请求

```python
response = client.responses.create(
    model="gpt-4.1-mini",
    input=[{
        "role": "user",
        "content": [
            {"type": "input_text",  "text": "what's in this image?"},
            {"type": "input_image", "image_url": "https://.../cat.jpg"},
        ],
    }],
)
```

要点:

1. **单次 API call**。`content` 数组里两个 part 同属一条 `user` message,模型一次性看到全部内容。
2. **数组顺序 = 模型看到的顺序**。习惯上把指令/问题放前面,素材放后面。
3. **共享同一个 `role`**。不是两轮对话,是同一轮的多个组成部分。
4. **计费**:文本按 token,图片按 `detail` 等级折算后计入 `usage.input_tokens`。
5. **可放多张图**:`content` 里塞多个 `input_image` 即可,用于「比较 A 和 B」等场景。
6. **对照「多轮」**:先发文本、收到回复、再发图片,才会形成两条 message,即两轮交互。

---

## 4. `Runnable.with_config` 的作用

### 4.1 原理

`with_config` 是所有 `Runnable`(chat model、prompt、chain、retriever 等)都有的方法。它**返回一个新的 Runnable**,把传入的 `RunnableConfig` 默认绑定到新对象上;之后每次调用,这些配置会自动合并进去。**不修改原对象**,类似 `functools.partial`。

签名:

```python
Runnable.with_config(
    config: RunnableConfig | None = None,
    **kwargs,
) -> Runnable
```

`RunnableConfig` 里常用字段:

| 字段 | 用途 | 是否继承到 sub-call |
|---|---|---|
| `run_name` | LangSmith / 日志里这次调用显示的名字 | ❌ 不继承 |
| `tags` | 标签列表,LangSmith 里用来筛选 | ✅ 继承 |
| `metadata` | 任意 key-value,附加到 trace | ✅ 继承 |
| `callbacks` | 事件回调 | ✅ 继承 |
| `configurable` | 运行时覆盖 `configurable_fields` 声明的值 | ✅ 继承 |
| `max_concurrency` | `batch()` 并发上限 | — |
| `recursion_limit` | chain 最大递归深度 | — |

### 4.2 推论:为什么 pipeline 里要给同一个 llm 套多次 `with_config`

```python
second_llm_chain = LLMChain(
    llm=llm.with_config(run_name="ai_meeting_second_llm"),
    prompt=SECOND_PROMPT,
)
```

含义:克隆 llm 得到一个新对象,在 LangSmith trace 树里显示为 `ai_meeting_second_llm`。当同一条 pipeline 里有多次 LLM 调用(first / second / refine ...)时,trace 上能一眼区分,而不是全都叫 `ChatOpenAI`。

### 4.3 最小示例

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# 同一个 llm,克隆出两个带不同 run_name 的版本
first_llm  = llm.with_config(run_name="first_call",  tags=["stage:draft"])
second_llm = llm.with_config(run_name="second_call", tags=["stage:refine"])

print(first_llm.invoke("写一句关于春天的诗").content)
print(second_llm.invoke("再润色一下:春风又绿江南岸").content)
```

LangSmith 上看 trace:
- 第一次调用 span 名 = `first_call`,带 tag `stage:draft`
- 第二次调用 span 名 = `second_call`,带 tag `stage:refine`
- `llm` 本身保持不变,可继续在别处复用

### 4.4 `with_config` vs `invoke(..., config=...)`

#### 4.4.1 原理:两者**不互斥、不替换,而是合并**

两边接收的都是同一种 `RunnableConfig`(字段集合完全一致),差别只在**绑定时机**与**生命周期**:

| 维度 | `with_config(...)` | `invoke(..., config=...)` |
|---|---|---|
| 返回值 | **新的 Runnable**(不可变,原对象不变) | 直接返回模型输出 |
| 生效范围 | 该 Runnable 之后**每一次**调用 | 仅这**一次**调用 |
| 适合放什么 | 不随请求变的、Runnable 的「身份」 | 随请求变的、单次调用的「数据」 |
| 重复使用 | 一次 `with_config`,多次 invoke 都带 | 每次 invoke 都要重新传 |

#### 4.4.2 合并语义(关键)

两边同时设置时,LangChain 在 `langchain_core.runnables.config.merge_configs` 里按**字段类型**做不同合并:

| 字段 | 合并规则 |
|---|---|
| `tags` | **拼接 + 去重**(两边的 tag 都保留) |
| `metadata` | **dict 合并**;key 冲突时 invoke 端覆盖 with_config 端 |
| `callbacks` | **合并到同一个 callback manager**,两边的 handler **都会触发** |
| `configurable` | dict 合并,invoke 端覆盖 |
| `run_name` / `run_id` / `max_concurrency` / `recursion_limit` | **标量值,invoke 端覆盖** |

推论:`callbacks` 放在哪边都不会丢——「全局监控 handler」可以绑在 with_config,「请求级 trace handler」放在 invoke,两者并存。

#### 4.4.3 实例分析:为什么 invoke 端是对的

```python
summary_chain.invoke(
    invoke_input,
    config={
        "callbacks": [monitor_handler, langfuse_handler],
        "metadata": {summary_id, scenario, user_id, file_id, note_id},
    },
)
```

放进 `config=` 的两类数据,都**强依赖当前请求**:

- `summary_id` / `user_id` / `file_id` / `note_id` — 每次请求都不同,不可能预先 `with_config` 绑死;
- `monitor_handler` / `langfuse_handler` — 通常为这次请求构造(尤其 Langfuse `CallbackHandler` 常带本次 trace 的 `session_id` / `user_id`),也是请求级状态。

反例(会产生**跨请求状态污染**):

```python
# ❌ 不要这样做
summary_chain = summary_chain.with_config({
    "callbacks": [monitor_handler],     # handler 在所有请求间共享,事件累积
    "metadata":  {"user_id": user_id},  # 第二个请求的 trace 会带上第一个请求的 user_id
})
```

#### 4.4.4 分工口诀

- **写代码时已经确定 → `with_config`**(角色名、阶段标签、模型层默认 callback)。
- **运行时才能确定,且每次都不一样 → `invoke(config=...)`**(user_id、trace_id、请求级 handler)。
- **两边都设也安全**:`tags` 拼接、`callbacks` 都触发、`metadata` 合并(冲突时 invoke 赢)。

#### 4.4.5 配套写法

```python
# 一次性构建期 —— Runnable 的「身份」放 with_config
summary_chain = (PROMPT | llm | parser).with_config(
    run_name="summary_chain",       # trace 上的稳定名字
    tags=["stage:summary", "v2"],   # 该阶段标签
    # 全局通用、无请求状态的 handler 也可放这里
)

# 每次请求 —— 请求级数据放 invoke(config=...)
summary_chain.invoke(
    invoke_input,
    config={
        "callbacks": [monitor_handler, langfuse_handler],
        "metadata": {
            "summary_id": summary_id,
            "user_id":    user_id,
            "file_id":    file_id,
            "note_id":    note_id,
        },
        # 也可在这里覆盖 run_name 做更细的 span 名:
        # "run_name": f"summary:{summary_id}",
    },
)
```

---

## 参考

- [LangChain Messages](https://docs.langchain.com/oss/python/langchain/messages)
- [LangChain Models 与 Invocation config](https://docs.langchain.com/oss/python/langchain/models)
- [LangSmith · Add metadata and tags(`with_config` 与 invoke 端并用)](https://docs.langchain.com/langsmith/trace-with-langchain#add-metadata-and-tags-to-traces)
- [LangSmith · Customize run name / run id](https://docs.langchain.com/langsmith/trace-with-langchain)
- [OpenAI Chat Completions content parts](https://developers.openai.com/api/docs/api-reference/chat/object)
- [OpenAI Responses 多模态输入](https://developers.openai.com/api/docs/api-reference/responses/create)
- [OpenAI Vision / Images Guide](https://developers.openai.com/api/docs/guides/images)
