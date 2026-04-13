# Plaud Summary - 所有 Model 调用汇总

> 生成日期：2026-04-13
> 项目路径：`/Users/liangzhu/Documents/work/plaud-summary`
> 目的：为后续 model 调用改造提供全景索引

---

## 一、架构概览

```
请求入口 (API/Kafka/Script)
    ↓
summary.py (模型选择逻辑)
    ↓
EndpointDispatcher.dispatch() (核心调度器)
    ↓
endpoint.py (实例化各厂商 LLM 客户端)
    ↓
LangChain 统一接口 (.invoke / .ainvoke / .batch)
    ↓
各业务模块通过 endpoint.llm 调用模型
```

### 核心类型体系 (`plaud_summary/plaud/endpoint_manager/llm_config.py`)

| 枚举 | 说明 |
|------|------|
| `ModelName` | 内部模型标识（35+），如 `GPT_5`, `CLAUDE_SONNET_4`, `GEMINI_2_5_PRO` |
| `LLMName` | 前端/API 层面向用户的模型名（与 APP/WEB 约定） |
| `EndpointPrefix` | AWS 配置中的 endpoint name 前缀，用于路由匹配 |
| `EndpointType` | 厂商类型：AZURE, OPENAI, CLAUDE, PTU, GEMINI, QWEN, DOUBAO 等 |
| `ReasoningEffort` | 推理力度：LOW / MEDIUM / HIGH |
| `ModelDef` | 模型元数据定义，包含 dispatch 策略（weighted/gemini/dedicated/general） |

### 模型注册表 (`MODEL_REGISTRY`)
- 位置：`llm_config.py:182-221`
- 支持从 AWS AppConfig (`llm.yaml`) 动态刷新 (`refresh_model_index()`)
- 硬编码 fallback + 远程配置覆盖

---

## 二、LLM 客户端实例化（endpoint.py）

所有 LLM 客户端都在 `plaud_summary/plaud/endpoint_manager/endpoint.py` 的 `EndpointDispatcher` 中创建。

### 2.1 OpenAI 系列

| 行号 | 客户端类 | 模型 | 备注 |
|------|----------|------|------|
| 909 | `AzureChatOpenAI` | GPT-4O (Azure) | temperature 可配 |
| 956 | `ChatOpenAI` | GPT-4O (OpenAI 直连) | |
| 996 | `ChatOpenAI` | O3-MINI | reasoning_effort=MEDIUM, max_tokens=8192 |
| 1037 | `AzureChatOpenAI` | O3-MINI (Azure) | reasoning, max_tokens=8192 |
| 1708 | `ChatOpenAI` | GPT-5 | |
| 1749 | `ChatOpenAI` | GPT-5 变体 | |
| 1989 | `ChatOpenAI` | O3 | |
| 2032 | `ChatOpenAI` | O4-MINI | |
| 2593 | `AzureChatOpenAI` | Azure PTU | |
| 2620 | `AzureChatOpenAI` | Azure GPT-4.1 | |
| 2653 | `AzureChatOpenAI` | Azure GPT-5 | |
| 2707 | `AzureChatOpenAI` | Azure GPT-5.1 | |
| 2761 | `AzureChatOpenAI` | Azure GPT-5.2 | |
| 2808 | `ChatOpenAI` | O3 endpoint 变体 | |

### 2.2 Claude 系列

| 行号 | 客户端类 | 模型 | 备注 |
|------|----------|------|------|
| 1090 | `ChatBedrock` | Claude 3.5 (AWS Bedrock) | max_tokens=8192 |
| 1671 | `ChatBedrock` | Claude 3.7 (AWS Bedrock) | |
| 1802 | `ChatBedrock` | Claude Sonnet 4 (Bedrock) | |
| 1853 | `ChatBedrock` | Claude Sonnet 4 Thinking | |
| 1901 | `ChatBedrock` | Claude Sonnet 4.5 | |
| 1952 | `ChatBedrock` | Claude Opus 4 | |
| 2394 | `ChatAnthropicVertex` | Claude (GCP Vertex AI) | |
| 2451 | `ChatAnthropicVertex` | Claude Vertex 变体 | |
| 2506 | `ChatAnthropicVertex` | Claude Vertex 变体 | |
| 2561 | `ChatAnthropicVertex` | Claude Vertex 变体 | |

### 2.3 Gemini 系列

| 行号 | 客户端类 | 模型 | 备注 |
|------|----------|------|------|
| 2081 | `ChatVertexGemini` | Gemini 2.5 Flash Lite | timeout=10s |
| 2141 | `ChatVertexGemini` | Gemini 2.5 Pro | 支持 thinking_budget |
| 2210 | `ChatVertexGemini` | Gemini 3 Pro / Flash | |
| 2270 | `ChatVertexGemini` | Gemini endpoint 变体 | |
| 2329 | `ChatVertexGemini` | Gemini endpoint 变体 | |

### 2.4 DeepSeek 系列

| 行号 | 客户端类 | 模型 | 备注 |
|------|----------|------|------|
| 1162 | `ChatBedrockConverse` | DeepSeek R1 (Bedrock) | max_tokens=8000 |
| 1577 | `ChatPlaudAI` | DeepSeek R1 (CN 直连) | |
| 1615 | `ChatPlaudAI` | DeepSeek V3.2 (CN 直连) | |
| 3076 | `ChatBedrockConverse` | DeepSeek (global fallback) | |

### 2.5 CN 国产模型系列

| 行号 | 客户端类 | 模型 | 备注 |
|------|----------|------|------|
| 1196 | `ChatPlaudAI` | Qwen Plus 250714 | 禁用内容安全检测 |
| 1230 | `ChatPlaudAI` | Qwen Plus Stable / Qwen3 Max | |
| 1266 | `ChatPlaudAI` | Doubao Seed 1.6 | 禁用内容审核 |
| 1310 | `ChatPlaudAI` | Doubao Seed 1.6 Thinking/Flash | |
| 1354 | `ChatPlaudAI` | Doubao Seed 1.8 | |
| 1410 | `ChatPlaudAI` | Doubao Seed 2.0 Pro | 支持 reasoning_effort |
| 1461 | `ChatPlaudAI` | Doubao Seed 2.0 Lite | |
| 1503 | `ChatPlaudAI` | Kimi K2 | |
| 1541 | `ChatPlaudAI` | Kimi K2.5 | |

### 2.6 Embedding 模型

| 行号 | 客户端类 | 说明 |
|------|----------|------|
| 2857 | `AzureOpenAIEmbeddings` | Azure Embedding |
| 2878 | `OpenAIEmbeddings` | OpenAI Embedding |
| 2899 | `OpenAIEmbeddings` | Qwen Embedding (兼容 OpenAI 接口) |
| 3027-3037 | `AzureOpenAIEmbeddings` | 工厂方法 `create_azure_embedding()` |

### 2.7 通用工厂方法（endpoint.py 底部）

| 行号 | 方法 | 类 |
|------|------|-----|
| 3166 | Generic Azure | `AzureChatOpenAI` |
| 3178 | Generic OpenAI | `ChatOpenAI` |
| 3186 | Generic Qwen/Doubao | `ChatPlaudAI` |
| 3208 | Generic Bedrock | `ChatBedrock` |
| 3223 | Generic Vertex Claude | `ChatAnthropicVertex` |
| 3236 | Generic Vertex Gemini | `ChatVertexGemini` |

---

## 三、自定义 LLM 封装

### 3.1 ChatPlaudAI (`plaud_summary/llm/langchain_plaud_llm.py`)
- 继承自 `ChatOpenAI`
- 作用：为 Volcengine（豆包）和阿里云（通义）注入内容安全豁免 Header
  - 豆包：`x-ark-moderation-scene: disable`
  - 通义：`X-DashScope-DataInspection: disable`
- 处理内容审核过滤和长度限制错误

### 3.2 ChatVertexGemini (`plaud_summary/plaud/endpoint_manager/vertex_gemini.py`)
- 继承自 `ChatGoogleGenerativeAI`
- 新增 `thinking_budget` 和 `thinking_level` 参数
- 默认 timeout=10s

### 3.3 AWSChatDeepSeek (`plaud_summary/plaud/endpoint_manager/aws_deepseek.py`)
- 继承自 `BaseChatOpenAI`
- 通过 AWS Bedrock runtime `client.converse()` 调用
- 默认参数：`maxTokens=2000, temperature=0.6, topP=0.9`

---

## 四、模型调度入口（按业务模块）

### 4.1 主摘要流程 (`plaud_summary/summary.py`)

| 行号 | 调度方式 | 使用模型 | 场景 |
|------|----------|----------|------|
| 2142 | `dispatcher.dispatch()` | GPT_5 | 默认摘要 |
| 2151 | `dispatcher.dispatch_auto_model()` | AUTO | 自动模型选择（概率分配） |
| 2166 | `dispatcher.dispatch()` | 用户指定 | 用户选择特定模型 |
| 2177 | `dispatcher.dispatch_by_endpoint_prefixes()` | 按前缀匹配 | 按 endpoint 前缀路由 |
| 2202 | `dispatcher.dispatch()` | 按模型名 | 遍历 fallback 列表 |
| 2206 | `dispatcher.dispatch()` | 默认 | 最终 fallback |
| 604 | mark_note dispatch | GEMINI_2_5_PRO / DOUBAO_SEED_1_8(CN) | Mark Note 标注 |

### 4.2 基础摘要执行 (`plaud_summary/plaud/basic_runnable_summary.py`)

| 行号 | 调用方式 | 场景 |
|------|----------|------|
| 640-728 | `EndpointDispatcher.dispatch()` | 模型选择 + 429 限流降级 + 超时重试 |
| 958 | `summary_chain.invoke()` | 主摘要生成（LangChain chain） |
| 966 | `summary_chain.invoke()` | 摘要生成（备用路径） |
| 1074 | `card_generator.generate()` | Summary Card 生成 |
| 1175 | `.invoke(input_dict, config)` | Runnable chain 执行 |

**降级逻辑**（basic_runnable_summary.py:640-730）：
- 默认模型列表：`GPT_5 → GEMINI_2_5_PRO → CLAUDE_SONNET_4 → GEMINI_2_5_FLASH`
- CN 模型列表：`DOUBAO_SEED_1_8 → QWEN3_MAX → DOUBAO_SEED_1_6_FLASH`
- Gemini 429 → 切 global endpoint
- Claude 超时 → 切换到其他 Claude endpoint
- 最终 fallback 链：`GPT_5 → GPT_5 → GEMINI_2_5_PRO → GEMINI_2_5_FLASH`

### 4.3 Knowledge Notes (`plaud_summary/plaud/knowledge_notes/`)

#### knowledge_base_note.py
| 行号 | 调用方式 | 场景 |
|------|----------|------|
| 354 | `self._language_chain.invoke()` | 语言检测 |
| 378 | `self._header_llm_chain.invoke()` | 标题生成 |
| 703 | `processor.map().invoke()` | 批量处理 |
| 970-976 | 阶段模型列表 | framework/expand/polish 各阶段 |
| 1000 | `dispatcher.dispatch()` | 按模型名获取 endpoint |
| 1080 | `.invoke(prompt)` | 正文生成 |
| 1117 | `dispatcher.dispatch()` | global fallback |

**支持的模型映射**（knowledge_base_note.py:82-101）：
- Gemini: 2.5-pro, 2.5-flash, 2.5-flash-lite, 3-pro
- GPT: 4o, 4.1, 5, 5.1, 5.2
- O系列: o3, o3-mini, o4-mini
- Claude: 3.5, 3.7, sonnet-4, sonnet-4.5, opus-4

#### knowledge_base_note_short.py
| 行号 | 调用方式 | 场景 |
|------|----------|------|
| 294 | `self._language_chain.invoke()` | 语言检测 |
| 317 | `self._header_llm_chain.invoke()` | 标题生成 |
| 428 | `.invoke(prompt)` | 正文生成 |
| 462 | `dispatcher.dispatch()` | global fallback |

### 4.4 Dimension Notes (`plaud_summary/plaud/dimension_notes/`)

所有 dimension note 都通过 chain.invoke() 调用模型（使用上层传入的 endpoint）：

| 文件 | 行号 | 场景 |
|------|------|------|
| `intent_analysis_note.py` | 81 | 意图分析 |
| `speaker_note.py` | 86 | 说话人分析 |
| `todo_note.py` | 85, 152 | TODO 提取 |
| `key_data_note.py` | 184, 214, 245 | 数据分类/去重/格式化 |
| `concept_hunter_note.py` | 118 | 概念提取 |
| `conversation_gems_note.py` | 98 | 金句提取 |
| `essence_note.py` | 77 | 精华提取 |
| `meeting_details_note.py` | 84 | 会议细节 |
| `meeting_effectiveness_note.py` | 78 | 会议效能评估 |
| `speaker_communication_efficiency_note.py` | 83 | 沟通效率分析 |

### 4.5 Official Notes (`plaud_summary/plaud/official_notes/`)

| 文件 | 行号 | 场景 |
|------|------|------|
| `base/sequential_processing_note.py` | 404 | 顺序处理 chain.invoke |
| `base/sequential_processing_note.py` | 506 | 顺序处理 chain.invoke |

### 4.6 Frontend Agent Notes (`frontend/generate_note/agents/`)

| 文件 | 行号 | 默认模型 | 场景 |
|------|------|----------|------|
| `doc_review_agent.py` | 37, 83 | GPT_4_1 | 文档审核 |
| `review_agent.py` | 81, 202, 388 | GPT_4_1 | 评审 Agent |
| `strategy_planning_agent.py` | 148, 326 | GPT_4_1 | 策略规划 |
| `summary_generator_agent.py` | 59, 203 | GPT_4_1 | 摘要生成 |
| `doc_relationship_agent.py` | 110, 217, 225 | GPT_4_1 | 文档关系分析 |
| `prompt_assembler_agent.py` | 498 | GPT_4_1 | Prompt 组装 |

> 注：Frontend Agent 全部硬编码使用 `ModelName.GPT_4_1`

### 4.7 Chains（通用 LLM 链）

| 文件 | 行号 | 使用模型 | 场景 |
|------|------|----------|------|
| `form_llm_chain.py` | 58-81 | 上层传入 | 表单翻译 (.with_structured_output) |
| `batch_llm.py` | 37-62 | gpt-4o（示例） | 批量推理 |
| `speaker_disable_chain.py` | 57, 124 | 上层传入 | 说话人/语言链 |
| `enable_result_chain.py` | 43-60 | GEMINI_2_5_FLASH / DOUBAO_SEED_1_6_FLASH(CN) | 结果验证 |
| `self_check_chain.py` | 29-40 | GPT_5 / DOUBAO_SEED_1_8(CN) | 自检链 |
| `re_act_chain.py` | 141-145 | GEMINI_2_5_FLASH / DOUBAO_SEED_1_6_FLASH(CN) | ReAct 链 |
| `compare_results.py` | 38-43 | O3_MINI | 多模型结果比较 |
| `batch_llm_chain.py` | 181 | 上层传入 | 批量 LLM 链 |

### 4.8 Server 层

| 文件 | 行号 | 使用模型 | 场景 |
|------|------|----------|------|
| `generate_prompt.py` | 127-158 | GPT_5（默认）/ GEMINI_2_5_PRO（图片<7MB）/ DOUBAO_SEED_1_8(CN) | Prompt 生成 + 图片处理 |
| `generate_prompt.py` | 154-156 | GEMINI_2_5_FLASH / DOUBAO_SEED_2_0_LITE(CN) | 优化模型 |
| `generate_prompt.py` | 212, 221 | `.ainvoke()` + `.with_fallbacks()` | 异步调用 + fallback |
| `utils.py` | 693 | `chain.astream()` | 流式输出 |
| `zilliz_region_sync.py` | 810 | Embedding | 向量同步 |

### 4.9 Community 模块 (`community/`)

| 文件 | 行号 | 使用模型 | 场景 |
|------|------|----------|------|
| `note_tab_sevice.py` | 200-204 | GPT_5 / DOUBAO_SEED_1_8(CN) | 社区笔记 Tab |
| `note_tab_with_translate_service.py` | 73-77 | GPT_5 / DOUBAO_SEED_1_8(CN) | 带翻译的笔记 Tab |
| `translation_service.py` | 431-433 | GPT_5_2 | 翻译服务主模型 |
| `translation_service.py` | 466-473 | 多模型 fallback 链 | 翻译重试链：GEMINI_2_5_FLASH → GPT_5 → GEMINI_2_5_FLASH → O3_MINI → CLAUDE_3_7 → GEMINI_2_5_PRO → GPT_5 → O4_MINI |
| `translation_service.py` | 512-516 | GEMINI_2_5_PRO / DOUBAO_SEED_1_8(CN) | Outline 翻译 |
| `single_word_translate_service.py` | 91-95 | GPT_5 / DOUBAO_SEED_1_8(CN) | 单词翻译 |
| `search/milvus_clients.py` | 36-37 | Embedding | 向量检索 |

### 4.10 Trans Compress (`plaud_summary/trans_compress/service.py`)

| 行号 | 使用模型 | 场景 |
|------|----------|------|
| 363-367 | DOUBAO_SEED_2_0_LITE / DOUBAO_SEED_1_8(CN), GEMINI_2_5_FLASH / GEMINI_3_FLASH(global) | 转写压缩 |
| 397 | `chain.invoke()` | 语言检测 |
| 510 | `chain.invoke()` | 文本压缩 |
| 585 | `chain.invoke()` | 文本压缩（备用） |

### 4.11 Mark Note (`plaud_summary/mark_note/`)

| 文件 | 行号 | 使用模型 | 场景 |
|------|------|----------|------|
| `mark_note_processor.py` | 46-50 | GEMINI_2_5_PRO / DOUBAO_SEED_1_8(CN) | Mark 处理默认模型 |
| `mark_note_processor.py` | 233-237 | GEMINI_2_5_PRO / DOUBAO_SEED_1_8(CN) | 动态模型切换 |
| `mark_note_post_processor.py` | 49-53 | GEMINI_2_5_PRO / DOUBAO_SEED_1_8(CN) | Mark 后处理 |
| `mark_note_summary.py` | 280, 1794 | 上层传入 | 语言检测 + 标题生成 |

### 4.12 Persona (`plaud_summary/features/persona/persona_service.py`)

| 行号 | 使用模型 | 场景 |
|------|----------|------|
| 215-217 | O3_MINI, GEMINI_2_5_FLASH, GPT_4_1 | Persona 候选模型列表 |
| 239 | `dispatcher.dispatch()` | 主 endpoint |
| 247 | `dispatcher.dispatch()` | fallback endpoint |
| 287 | `chain.invoke()` | Persona 描述生成 |
| 406 | `evaluation_chain.invoke()` | Onboarding 评估 |

### 4.13 Summary Card (`plaud_summary/services/summary_card/poster_generator.py`)

| 行号 | 调度方式 | 场景 |
|------|----------|------|
| 1441 | `dispatcher.dispatch()` | 卡片生成（按 ModelName） |
| 1447 | `dispatcher.dispatch_by_endpoint_prefixes()` | 按 endpoint 前缀匹配 |
| 1462-1469 | 特殊处理 GEMINI_3_PRO/FLASH, O3_MINI/O4_MINI | |
| 1914 | `llm.invoke()` | 卡片内容生成 |

### 4.14 AI Router Note (`plaud_summary/plaud/ai_router_note.py`)

| 行号 | 调度方式 | 场景 |
|------|----------|------|
| 57 | `EndpointDispatcher.dispatch(content)` | 默认 dispatch（无指定模型） |
| 66 | `chain.invoke()` | 路由分析 |

### 4.15 其他调用点

| 文件 | 行号 | 调用 | 场景 |
|------|------|------|------|
| `plaud_summary/plaud/overview_chunk_note.py` | 326 | `chain.invoke(input)` | 概览分块 |
| `plaud_summary/plaud/utils.py` | 192-206 | `dispatch_embedding()` | Embedding 创建 |
| `plaud_summary/parsers/pydantic_fix_retry_output_parser.py` | 84 | `new_chain.invoke()` | 输出修复重试 |
| `plaud_summary/parsers/pydantic_fix_retry_output_parser_new.py` | 81 | `new_chain.invoke()` | 输出修复重试（新版） |
| `tests/openai_image.py` | 348 | `client.chat.completions.create()` | 唯一直接 OpenAI SDK 调用（测试） |
| `tests/evaluation/core/evaluator.py` | 66 | `ChatOpenAI(model="gpt-4o-mini")` | 评估打分 |

---

## 五、Callback 与 Token 追踪

| 文件 | 类名 | 追踪对象 |
|------|------|----------|
| `plaud_summary/callbacks/bedrock_anthropic_callback.py` | `BedrockAnthropicTokenUsageCallbackHandler` | Claude (Bedrock) token + cost |
| `plaud_summary/callbacks/deepseek_callback.py` | `DeepSeekCallbackHandler` | DeepSeek token + cost + reasoning_tokens |
| `plaud_summary/callbacks/gemini_callback.py` | `GeminiCallbackHandler` | Gemini token 用量 |
| `plaud_summary/callbacks/manager.py` | Context managers | 各厂商 callback 的统一管理 |
| `plaud_summary/plaud/total_cost_calculation.py` | - | Token 成本计算 |

---

## 六、配置文件

| 文件 | 说明 |
|------|------|
| `aws_appconfig/llm.yaml` | Global 模型注册表（41 模型定义 + 前端展示配置） |
| `aws_appconfig/llm-cn.yaml` | CN 区模型注册表（13 模型） |
| `aws_appconfig/dev.yaml` / `local.yaml` | 环境级别配置 |
| `conf/config_llm_ctrl.json` | 本地 LLM 控制配置 |
| `plaud_summary/config.py` | 主配置模块（`get_summary_card_llm_model()` 等） |
| `plaud_summary/llm_profile_config.py` | LLM AppConfig profile loader（60s TTL 缓存） |
| `server/models_config.py` | 动态模型配置 API |

---

## 七、CN 区域特殊处理模式

几乎所有业务模块都遵循相同的 CN/Global 分支模式：

```python
if is_cn_region():
    model = ModelName.DOUBAO_SEED_1_8  # 或其他 CN 模型
else:
    model = ModelName.GPT_5  # 或其他 Global 模型
```

**CN 默认模型映射**：

| Global 模型 | CN 替代模型 | 使用场景 |
|-------------|------------|----------|
| GPT_5 | DOUBAO_SEED_1_8 | 摘要/社区/翻译/Mark Note |
| GEMINI_2_5_FLASH | DOUBAO_SEED_1_6_FLASH | enable_result / re_act chain |
| GEMINI_2_5_PRO | DOUBAO_SEED_1_8 | Mark Note / 图片处理 |
| GEMINI_2_5_FLASH (压缩) | DOUBAO_SEED_2_0_LITE | Trans Compress |
| GPT_5 (自检) | DOUBAO_SEED_1_8 | Self Check Chain |
| GEMINI_2_5_FLASH (优化) | DOUBAO_SEED_2_0_LITE | Prompt 优化 |

---

## 八、模型调用全链路（从 invoke 到返回结果）

> 以核心摘要流程 `basic_runnable_summary.py` 为主线，覆盖所有分支和异常路径

### 8.1 全链路架构图

```
┌──────────────────────────── Phase 1: 模型选择 ────────────────────────────┐
│                                                                           │
│  summary.py:get_endpoint_for_llm()                                        │
│      │                                                                    │
│      ├─ GPT-4O/4O-old/4.1 → 强制映射到 GPT_5                             │
│      ├─ AUTO → dispatch_auto_model() → 概率选模型                         │
│      ├─ llm_model_map 命中 → dispatch(model_name)                         │
│      ├─ 动态路由 → endpoint_prefixes / compatible_with                    │
│      └─ 全部未命中 → 默认 dispatch(GPT_5)                                │
│                                                                           │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │ Endpoint 或 None
┌───────────────────────────────────┴───────────────────────────────────────┐
│                                                                           │
│  Phase 2: Endpoint 调度 — EndpointDispatcher.dispatch()                   │
│      │                                                                    │
│      ├─ 1. tiktoken 预估 token 数 (×1.7 安全系数)                         │
│      ├─ 2. _pool 为空 → 返回 None                                        │
│      ├─ 3. model_index 查找 ModelDef                                      │
│      │      ├─ 找到 → _dispatch_endpoint() → 4 种策略                    │
│      │      └─ 未找到 → 跳过，走 fallback                                │
│      ├─ 4. 专用池返回 endpoint → 结束                                     │
│      └─ 5. Fallback: 遍历通用 _pool (round-robin) → endpoint 或 None     │
│                                                                           │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │ Endpoint 或 None
┌───────────────────────────────────┴───────────────────────────────────────┐
│                                                                           │
│  Phase 3: 前置校验 — basic_runnable_summary.run()                         │
│      │                                                                    │
│      ├─ tokens < 50 → 抛出 TOO_SHORT_TEXT_ERR（不重试）                    │
│      ├─ check_before_invoke() 失败 → CheckFailureHandler 直接返回         │
│      └─ 通过 → 进入 summary_result()                                      │
│                                                                           │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │
┌───────────────────────────────────┴───────────────────────────────────────┐
│                                                                           │
│  Phase 4: Chain 构建 + invoke                                             │
│      │                                                                    │
│      ├─ generate_summary_chain() 构建 LangChain pipeline                  │
│      ├─ 选择 Callback (Gemini/OpenAI/Bedrock/DeepSeek)                    │
│      ├─ summary_chain.invoke(input, callbacks)                            │
│      └─ 内部子链：language → form → batch_llm → extend → header → parse  │
│                                                                           │
└───────────────────────────────────┬───────────────────────────────────────┘
                                    │ 成功 / 异常
┌───────────────────────────────────┴───────────────────────────────────────┐
│                                                                           │
│  Phase 5: 结果处理 或 异常重试                                             │
│      │                                                                    │
│      ├─ 成功 → token 统计 → check_result() → 填充 metadata → 返回 dict    │
│      └─ 异常 → 异常分类 → 重试判定 → 换 endpoint/模型 → 递归 run()       │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

### 8.2 Phase 1: 模型选择（summary.py:2126-2206）

`get_endpoint_for_llm(content, llm, scenario, language, summary_id)` 按以下优先级决定使用哪个模型：

| 优先级 | 条件 | 行为 | 代码行 |
|--------|------|------|--------|
| 1 | `llm` 是 GPT-4O/4O-old/4.1 | 强制映射到 `ModelName.GPT_5` | 2138-2144 |
| 2 | `llm == "auto"` | `dispatch_auto_model()` 概率分配 | 2147-2153 |
| 3 | `llm` 在 `llm_model_map` 中 | 直接映射到 `ModelName`，如 `"gemini-2.5-pro"` → `GEMINI_2_5_PRO` | 2156-2168 |
| 4 | 动态路由命中 | 先尝试 `endpoint_prefixes` 前缀匹配，再尝试 `compatible_with` 兼容模型 | 2170-2202 |
| 5 | 全部未命中 | 默认 `dispatch(content)` → GPT_5 | 2205-2206 |

**Auto Model 概率分配（`dispatch_auto_model`，endpoint.py:273-346）：**

```
rand = random.random()  // [0, 1)
|--- Gemini 2.5 Pro (0.66) ---|--- GPT-5 (0.34) ---|
0                            0.66                  1.0
```

概率从 `dispatch_config`（AWS AppConfig）加载，dev 环境默认 66% Gemini / 34% GPT-5。还支持 Gemini 3 Pro、GPT-5.1、Qwen、DeepSeek、Doubao 等的概率配置（用于 CN 区域）。

---

### 8.3 Phase 2: Endpoint 调度（endpoint.py:365-427）

`dispatch()` 的核心逻辑：

```python
def dispatch(content, model_name=ModelName.GPT_5, prefer_global=False) -> Endpoint:
    # Step 1: Token 预估
    need_tokens = len(tiktoken.encode(content)) * 1.7  # gpt-4o 编码器 + 安全系数

    # Step 2: 池空检查
    if len(self._pool) == 0:
        return None  # ← 极端情况：所有配置都 using=false 或加载失败

    # Step 3: 目标模型查 MODEL_REGISTRY
    if model_index.get(will_use_model) is not None:
        endpoint = self._dispatch_endpoint(need_tokens, will_use_model, prefer_global)
        if endpoint:
            return endpoint  # ← 正常路径

    # Step 4: Fallback 到通用池 round-robin
    # 遍历 _pool，跳过 Bedrock 和 reasoning 类型
    for ep in _pool (round-robin):
        if _filter_endpoint(ep) and _check_endpoint(ep, need_tokens):
            return ep

    return None  # ← 没有任何可用 endpoint
```

#### 模型不存在时的行为

- `model_index.get(will_use_model)` 返回 `None` → 跳过 `_dispatch_endpoint()`
- 直接进入 **fallback 遍历通用 `_pool`**
- 从通用池中按 round-robin 选择第一个通过 `_filter_endpoint()` 的 endpoint
- 这意味着：传入一个未注册的 `ModelName` 不会报错，而是静默降级到通用池中的某个 endpoint

#### 四种 Dispatch 策略（_dispatch_endpoint，endpoint.py:693-755）

| 策略 | 适用模型 | 选择算法 | 返回 None 的条件 |
|------|----------|----------|------------------|
| `weighted` | GPT-5/5.1/5.2 | `random.choices(pool, weights=hits)` | 池为空 |
| `gemini` | Gemini 系列 | 加权随机 + regional/global 分离 | 池为空 |
| `dedicated` | Claude/O3 等 | round-robin，跳过上次使用的 | 池为空 或 全部 _check_endpoint 失败 |
| `general` | GPT-4O/O3-mini 等 | 在 `_pool` 中按名称匹配 + round-robin | 无匹配的 endpoint |

#### 模型池为空时的行为

```
_dispatch_endpoint() 返回 None
    → dispatch() 进入 fallback 遍历 _pool
        → 找到可用 endpoint → 返回（可能是完全不同的模型！）
        → 找不到 → 返回 None
```

---

### 8.4 Phase 3: 前置校验（basic_runnable_summary.py:806-850）

`summary_result()` 在构建 chain 之前做两层检查：

```python
# 1. Token 长度检查 (line 817-822)
if self.tokens_lens < 50:
    raise BasicSummaryException(ID.TOO_SHORT_TEXT_ERR)  # 不进入重试，直接抛出

# 2. 模板/场景前置检查 (line 824-850)
enable, failure_reason_key, reason_message = self.check_before_invoke()
if not enable:
    handler = CheckFailureHandlerRegistry.get_handler(failure_reason_key)
    return handler(...)  # 直接返回特殊结果，不调用 LLM
```

---

### 8.5 Phase 4: Chain 构建与 invoke

#### 8.5.1 Chain Pipeline（_default_generate_summary_chain，line 1244-1307）

```python
summary_chain = (
    language_chain                    # ① 语言检测（PromptTemplate | LLM | StrOutputParser）
    | RunnableBranch(form_chain)      # ② 可选：表单翻译 (.with_structured_output)
    | RunnablePassthrough.assign(scenario=...)  # ③ 注入 scenario
    | batch_llm_chain                 # ④ 核心：分块处理 + LLM 调用 + 输出解析
    | RunnableBranch(extend_chain)    # ⑤ 可选：扩展处理
    | header_llm_chain                # ⑥ 标题生成
    | RunnableLambda(parse_summary)   # ⑦ 解析为 Summary Pydantic 对象
    | RunnablePassthrough.assign(markdown=...)  # ⑧ 生成 Markdown
)

# 外层还包裹了两个可选分支（line 853-859）：
final_chain = summary_chain | RunnableBranch(
    re_act_llm_chain(),              # ⑨ ReAct 链（可选）
) | RunnableBranch(
    check_output_length_chain(),     # ⑩ 输出长度检查（可选）
)
```

#### 8.5.2 Callback 选择（line 864-879）

根据 `endpoint.llm` 的类型自动选择对应的 token 追踪 callback：

| LLM 类型 | Callback | 追踪内容 |
|----------|----------|----------|
| `ChatVertexAI` / `ChatVertexGemini` | `get_gemini_callback()` | Gemini token |
| `AzureChatOpenAI` | `get_openai_callback()` | OpenAI token + cost |
| `ChatOpenAI` | `get_openai_callback()` | OpenAI token + cost |
| `ChatBedrock` | `get_bedrock_anthropic_callback()` | Claude token + cost |
| `ChatBedrockConverse` | `get_deepseek_callback()` | DeepSeek token + reasoning |
| 其他 | `get_openai_callback()` (默认) | — |

此外还附加 `LLMMonitorCallbackHandler`（性能监控）和 `LangfuseCallbackHandler`（可观测性追踪）。

#### 8.5.3 invoke 执行（line 943-972）

```python
with callback as cb:  # token 累计上下文
    outputs = summary_chain.invoke(
        invoke_input,  # scenario, content, prompt_speakers, prompt_time 等
        config={
            "callbacks": [monitor_handler, langfuse_handler],
            "metadata": {summary_id, scenario, user_id, file_id, note_id},
        },
    )
```

如果 Langfuse 可用，还会包裹 `langfuse.start_as_current_observation()` span 用于分布式追踪。

#### 8.5.4 Output 解析的三级 Fallback（pydantic_fix_retry_output_parser_new.py）

`batch_llm_chain` 内部的 output parser 有三层容错：

```
Level 1: GeneralPydanticOutputParser.parse_result()
    → 正常解析 JSON → 返回 Pydantic 对象
    │
    ├─ JSONDecodeError → Level 2: OutputFixingParser
    │      LLM 修复畸形 JSON（再调一次 LLM 来 fix）
    │
    └─ OutputParserException → Level 3: RetryOutputParser
           ├─ 有 llm_output → LLM 拿着原 prompt + 错误输出重新解析
           └─ 无 llm_output → 重新执行整个 prompt | llm | parser 链
```

这意味着 **一次 batch_llm_chain 调用内部可能产生最多 3 次 LLM 请求**。

---

### 8.6 Phase 5: 结果处理（成功路径）

invoke 成功后的处理流程（line 973-1050）：

```
① Token 统计
   TokensCost(total, prompt, completion, reasoning, cached, cost)
   → calculate_llm_cost() 计算费用

② 提取结果
   language = outputs["language"]
   summary = outputs[CHAIN_KEY_PARSE_SUMMARY]  # Pydantic Summary 对象
   fix_summary()  # 修复 headline

③ 合并 header
   header_from_chain（标题、类别等）合并到 result dict

④ 结果校验
   check_result(markdown) → (enable, reason)
   if not enable → 抛出 RESULT_NOT_ENABLED（可重试）

⑤ category 校验
   original_category 存在但 category 缺失 → 抛出 MULTI_SUMMARY_CATEGORY_MISSING（可重试）

⑥ 填充 metadata
   result.update({endpoint, model, text_lens, tokens_lens, retry_count, ...})

⑦ 保存推荐问题
   save_recommend_questions_to_db()

⑧ 返回 result dict
```

---

### 8.7 Phase 5: 异常处理与重试

#### 8.7.1 异常分类（basic_summary_exception.py）

| ID | 名称 | 是否重试 | 说明 |
|----|------|----------|------|
| -1 | `OPENAI_INVALID_REQUEST_ERR` | **是** | OpenAI 请求参数错误 |
| -2 | `TOO_SHORT_TEXT_ERR` | **否** | 文本太短（<50 tokens） |
| -3 | `AZURE_CONTENT_FILTER_ERR` | **否** | Azure 内容过滤 |
| -4 | `OUTPUT_MISSING` | **是** | 输出缺少必要字段 |
| -5 | `OUTPUT_TOO_SHORT` | **是** | 输出过短（有独立计数器） |
| -6 | `MULTI_SUMMARY_CATEGORY_MISSING` | **是** | 类别字段缺失 |
| -7 | `RESULT_NOT_ENABLED` | **是** | 结果质量不达标 |
| -101/-102 | `VOLCANO_*` | **否** | 火山引擎模型错误 |
| -201/-202 | `QWEN_*` | **否** | 千问模型错误 |

**判定逻辑（line 650-658）：** BasicSummaryException 中只有上表标"是"的 5 种进入重试，其余直接 `raise e`。非 BasicSummaryException 的异常（如 `APITimeoutError`、`ResourceExhausted`、`RateLimitError`）全部进入重试。

#### 8.7.2 重试次数控制

```python
MAX_RETRY = 2                  # 常规重试次数
MAX_RETRY_TOO_SHORT = 1        # OUTPUT_TOO_SHORT 额外重试
# 总计最多 retry_num < MAX_RETRY + MAX_RETRY_TOO_SHORT = 3 次
```

#### 8.7.3 重试决策流程图

```
                            异常捕获
                               │
              BasicSummaryException 且不在可重试列表?
                   ├─ 是 → raise e（不重试）
                   └─ 否 ↓
              retry && retry_num < 3?
                   ├─ 否 → 检查是否 APITimeoutError（见 8.7.5）
                   └─ 是 ↓
              retry_num++
                   │
              Gemini regional 429?  ← is_gemini_rate_limit_error()
                   │
          ┌────────┴────────┐
          是                否
          │                 │
    dispatch(prefer_      USE_NEW_RETRY?
    global=True)          ├─ 是 → _select_retry_model()
          │               └─ 否 → 旧逻辑（硬编码切换）
    拿到 global EP?
    ├─ 是 → 用 global
    └─ 否 → _select_retry_model() / 直接切 GPT-5
          │
          ↓
    self.endpoint = 新 endpoint
          │
    endpoint 为 None?
    ├─ 是 → raise e（无可用 endpoint）
    └─ 否 → 递归调用 self.run(endpoint=新endpoint, retry_num=retry_num)
```

#### 8.7.4 _select_retry_model()（line 408-446）

从候选池中选择**第一个未失败的模型**：

```python
# Global 候选池（按优先级）
candidate_models = [
    ModelName.GPT_5,            # 优先
    ModelName.GEMINI_2_5_PRO,
    ModelName.CLAUDE_SONNET_4,
    ModelName.GEMINI_2_5_FLASH, # 最后
]

# CN 候选池
candidate_models = [
    ModelName.DOUBAO_SEED_1_8,
    ModelName.QWEN3_MAX,
    ModelName.DOUBAO_SEED_1_6_FLASH,
]

# 跳过已失败的模型
self._failed_models.add(current_model)
for model in candidate_models:
    if model not in self._failed_models:
        return model

# 全部失败 → 返回候选池最后一个（兜底）
return candidate_models[-1]
```

#### 8.7.5 超时额外重试（line 768-804）

即使 `retry_num >= MAX_RETRY`，如果错误是 `APITimeoutError` 且 Claude endpoint 可用，还会**额外尝试一次**：

```python
# retry_num 已超限，但超时可以多试一次 Claude
claude_endpoint = dispatcher.dispatch(text, model_name=ModelName.CLAUDE_SONNET_4)
if isinstance(e, APITimeoutError) and claude_endpoint:
    retry_num += 1
    return self.run(endpoint=claude_endpoint, retry_num=retry_num, ...)
```

#### 8.7.6 旧重试逻辑（deprecated，line 703-729）

通过 `USE_NEW_RETRY` Feature Flag 控制，当 Flag 关闭时使用旧逻辑：

```
retry_num == MAX_RETRY - 1 (即倒数第 2 次) → 强制切 GPT-5
ResourceExhausted 异常 → 切 GPT-5
OUTPUT_TOO_SHORT 计数器已满 → 切 GEMINI_2_5_PRO
BadRequestError → 切 GEMINI_2_5_FLASH
其他 → 保持当前模型（换同模型的其他 endpoint）
```

---

### 8.8 重试层次总结

一个请求从进入到最终返回，最多经历 **四层重试**：

| 层次 | 位置 | 重试什么 | 次数 | 触发条件 |
|------|------|----------|------|----------|
| L1 | LLM SDK | HTTP 连接重试 | `max_retries=1` | 网络错误、5xx |
| L2 | OutputParser | LLM 修复/重新调用 | 最多 2 次 | JSON 解析失败 |
| L3 | basic_runnable_summary | 换 endpoint/模型 | 最多 3 次 | 各种业务异常 |
| L4 | basic_runnable_summary | 超时额外 Claude | 1 次 | APITimeoutError |

**最坏情况下一个请求的 LLM 调用次数：**
- 第 1 次尝试：language(1) + batch_llm(N 个 chunk × 最多 3 次 parse) + header(1)
- 第 2 次重试：同上
- 第 3 次重试：同上
- 第 4 次超时重试：同上

---

### 8.9 Endpoint 配置加载与热更新

**配置源优先级：**
1. **AWS Secrets Manager**（生产主源）：`load_aws_secretmanager()` 拉取完整 endpoint JSON，带 `VersionId` 版本校验——版本未变则跳过重解析
2. **AWS AppConfig**（auto dispatch 概率）：`load_auto_dispatch_config()` 加载模型间流量分配比例
3. **本地 `dev-endpoints.json`**（开发/参考）：结构与 Secrets Manager 一致

**`using` 开关：** 每个 endpoint 配置中的 `"using": false` 会在解析时直接跳过，该 endpoint 不进入任何 pool。

**`hits` 字段：** endpoint 的选择权重（0~1），用于 `random.choices()` 的 `weights` 参数。若池中所有 hits 为 0，则 `_normalize_endpoint_hits()` 均匀分配。

**时间调度（`hits_schedule`）：** Gemini endpoint 按 UTC 时段动态调整权重，由 `_get_scheduled_hits()` 实现。

**TPM/RPM 限流：** `_check_endpoint()` 当前 `return True` 直接放行（限流逻辑存在但禁用）。

### 8.10 异常监控（exception_monitor.py）

独立后台线程，2 分钟窗口滚动监控：

| 异常类型 | 时间窗口 | 阈值 | 触发动作 |
|----------|----------|------|----------|
| PTU-RateLimitError | 2 分钟 | 10 次 | 日志告警 + 回调通知 |
| 其他异常 | 2 分钟 | 10 次（默认） | 日志告警 |

---

## 九、改造注意事项

1. **核心改动点**：`endpoint.py` 是所有模型客户端的创建中心（~3300 行），改造量最大
2. **通用调度器**：所有业务通过 `EndpointDispatcher.dispatch()` 获取 endpoint，统一入口
3. **Chain 调用统一**：LangChain `.invoke()` 是几乎唯一的调用方式
4. **CN/Global 双轨**：每个场景都有 CN 和 Global 两套模型配置
5. **Frontend Agent 硬编码**：`frontend/generate_note/agents/` 全部硬编码 GPT_4_1
6. **Embedding 独立通道**：`dispatch_embedding()` 与 chat 模型分离
7. **Fallback 链复杂**：`basic_runnable_summary.py` 和 `translation_service.py` 有多层 fallback
8. **唯一的直接 SDK 调用**：仅 `tests/openai_image.py` 直接使用 OpenAI SDK，其余全部通过 LangChain
9. **TPM/RPM 限流已禁用**：`_check_endpoint()` 直接 return True，限流代码存在但不生效
10. **`pt_hits_scale_config` 未实现**：配置存在但代码中无对应使用逻辑
11. **新旧重试逻辑并存**：通过 `USE_NEW_RETRY` Feature Flag 切换，旧逻辑标记为 deprecated 但仍在运行
