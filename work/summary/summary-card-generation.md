# 总结头图（Summary Card）生成完整流程

> 本文档详细描述了 Summary Card（总结头图）从条件判断到最终插入 Markdown 的完整生成流程，涵盖每一个模块、方法、配置和数据流。

## 目录

- [1. 整体架构](#1-整体架构)
- [2. 核心文件清单](#2-核心文件清单)
- [3. 第一阶段：条件判断](#3-第一阶段条件判断)
- [4. 第二阶段：LLM 生成 HTML Body](#4-第二阶段llm-生成-html-body)
- [5. 第三阶段：HTML 组装与验证](#5-第三阶段html-组装与验证)
- [6. 第四阶段：HTML 转 PNG](#6-第四阶段html-转-png)
- [7. 第五阶段：S3 上传与存储](#7-第五阶段s3-上传与存储)
- [8. 第六阶段：插入 Markdown](#8-第六阶段插入-markdown)
- [9. 第七阶段：内容审核与清理](#9-第七阶段内容审核与清理)
- [10. 设计系统](#10-设计系统)
- [11. Prompt 工程（LLM 指令体系）](#11-prompt-工程llm-指令体系)
- [12. 监控与埋点](#12-监控与埋点)
- [13. 错误处理策略](#13-错误处理策略)
- [14. 配置参数大全](#14-配置参数大全)
- [15. 数据流总览](#15-数据流总览)

---

## 1. 整体架构

### 流水线概览

```
总结执行完成
    │
    ▼
┌──────────────────────┐
│ 条件判断              │ ← 场景白名单 + 全局开关 + 灰度 + token 门槛
│ _should_generate_     │
│ summary_card()        │
└──────┬───────────────┘
       │ True
       ▼
┌──────────────────────┐
│ Step 1: LLM 生成      │ ← Langfuse Prompt / Fallback Prompt
│ _generate_html_body() │    Gemini / OpenAI / 其他 LLM
│                        │    输出: HTML body 片段 (Tailwind CSS)
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ Step 2: HTML 组装     │ ← 清洗 + 校验 + 修复 + 拼接骨架
│ _build_html()         │    添加 Google Fonts / Tailwind / Lucide Icons
│   → assemble_html()   │    CN 区域添加水印
│                        │    输出: 完整可渲染 HTML
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ Step 3: HTML → PNG    │ ← 外部无头浏览器渲染服务
│ _convert_html_to_png()│    /api/render endpoint
│                        │    重试策略: 3 次, 指数退避
│                        │    输出: PNG 字节流
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ Step 4: S3 上传       │ ← boto3 上传至 AWS S3
│ _upload_to_s3()       │    CN 区域嵌入 AIGC 元数据
│                        │    输出: S3 storage key
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ 插入 Markdown         │ ← ![PLAUD NOTE](storage_key)
│ insert_card_to_       │    插入在 H1 标题之后
│ markdown()            │
└──────┬───────────────┘
       ▼
┌──────────────────────┐
│ 内容审核              │ ← 提取 HTML 纯文本 → 审核
│ 如审核失败:           │    失败则删除 S3 + 清除 Markdown
│   delete_from_s3()    │
│   remove_card_from_   │
│   markdown()          │
└──────────────────────┘
```

### 设计原则

- **优雅降级**：头图生成任何阶段失败都不阻塞总结主流程
- **全链路可观测**：每个阶段独立计时，通过 Prometheus + 事件埋点 + Langfuse 追踪
- **灰度发布**：支持按比例灰度放量，配合场景白名单
- **合规嵌入**：CN 区域自动添加 AIGC 元数据和水印

---

## 2. 核心文件清单


| 文件路径                                                      | 职责                                 | 关键行号                                   |
| --------------------------------------------------------- | ---------------------------------- | -------------------------------------- |
| `plaud_summary/services/summary_card/poster_generator.py` | 主生成器 `SummaryCardGenerator`，串联整个流程 | 全文 ~2412 行                             |
| `plaud_summary/services/summary_card/html_assembler.py`   | HTML 校验、修复、骨架组装                    | 全文 ~770 行                              |
| `plaud_summary/services/summary_card/design_system.py`    | 设计系统（调色板、字体、Tailwind 配置）           | 全文 ~195 行                              |
| `plaud_summary/services/summary_card/metrics.py`          | Prometheus 指标定义                    | 全文 ~292 行                              |
| `plaud_summary/services/summary_card/__init__.py`         | 模块导出                               | -                                      |
| `plaud_summary/plaud/basic_runnable_summary.py`           | 入口：调用生成器 + 插入 Markdown             | L1062-1093, L1391-1425                 |
| `plaud_summary/summary.py`                                | 内容审核清理逻辑                           | L440-480                               |
| `plaud_summary/config.py`                                 | 所有配置项定义                            | L520-618                               |
| `plaud_summary/services/tracking/events.py`               | 埋点事件定义                             | `track_card`, `track_pii_card_content` |


---

## 3. 第一阶段：条件判断

### 3.1 入口方法

**文件**：`plaud_summary/plaud/basic_runnable_summary.py`
**方法**：`_should_generate_summary_card(self, summary_markdown: str) -> bool`

### 3.2 判断流程（按顺序，短路求值）

```
检查 1: 场景白名单 (allowed_scenarios)
    ├── 空列表 → 全部禁止 → return False
    ├── 包含 "*" → 允许所有场景
    └── 白名单匹配 → current_scenario 是否在列表中

检查 2: 全局开关 (summary_card.enabled)
    └── "false" → return False

检查 3: 灰度发布 (summary_card.rollout_percentage)
    └── random.random() > rollout_percentage → return False

检查 4: 原始内容 token 阈值 (summary_card.min_tokens)
    └── tokens_lens < min_tokens → return False

检查 5: 总结内容 token 阈值 (summary_card.min_summary_tokens)
    └── 使用 tiktoken (gpt-4o encoder) 计算 summary_markdown 的 token 数
    └── summary_tokens < min_summary_tokens → return False

所有检查通过 → return True, reason="eligible"
```

### 3.3 两层判断的职责分离


| 层级  | 方法                                              | 职责                  |
| --- | ----------------------------------------------- | ------------------- |
| 外层  | `_should_generate_summary_card()`               | 场景白名单检查（业务层面）       |
| 内层  | `SummaryCardGenerator.should_generate()` (静态方法) | 技术条件检查（开关/灰度/token） |


### 3.4 返回值语义

`should_generate()` 返回 `(bool, reason_str)`，reason 用于日志追踪：


| reason                                               | 含义         |
| ---------------------------------------------------- | ---------- |
| `"eligible"`                                         | 满足所有条件，应生成 |
| `"feature_disabled"`                                 | 全局开关关闭     |
| `"rollout_skipped(percentage=X)"`                    | 被灰度放量过滤    |
| `"transcript_too_short(transcript_tokens=X, min=Y)"` | 原始内容太短     |
| `"summary_too_short(summary_tokens=X, min=Y)"`       | 总结太短       |


---

## 4. 第二阶段：LLM 生成 HTML Body

### 4.1 方法调用链

```
generate()
  └── _generate_html_body(markdown_content)
        ├── _get_final_prompt(markdown_content)     # 获取并填充 Prompt
        │     ├── Langfuse.get_prompt(name, label)  # 优先从 Langfuse 获取
        │     └── Template(FALLBACK_PROMPT).render() # 失败则用代码内置 Prompt
        ├── _create_llm()                           # 通过 EndpointDispatcher 创建 LLM
        └── _invoke_llm_with_timeout(llm, prompt, callbacks, timeout)
              └── ThreadPoolExecutor + future.result(timeout)
```

### 4.2 Prompt 获取策略

**Langfuse 远程 Prompt（优先）**：

- 名称：`summary/card-image/global/module-generator`
- 标签：`production`
- 支持版本管理和 A/B 测试

**代码内置 Fallback Prompt**：

- 位于 `poster_generator.py` L94-L1333
- 约 1240 行的详细设计指令

### 4.3 Prompt 模板变量


| 变量                 | 来源                                         | 说明              |
| ------------------ | ------------------------------------------ | --------------- |
| `{{canvas_width}}` | `config.get_summary_card_max_dimension()`  | 画布宽度，默认 1920px  |
| `{{palettes}}`     | `PALETTES` 列表 join                         | 可用调色板名称         |
| `{{language}}`     | `get_language_display_name(self.language)` | 语言显示名（如 "简体中文"） |
| `{{content}}`      | `markdown_content` 参数                      | 总结 Markdown 文本  |


### 4.4 LLM 创建流程

通过 `EndpointDispatcher` 统一路由，支持负载均衡和限流：

```python
def _create_llm(self):
    # 1. 将配置的模型名（如 "gemini-2.5-pro"）映射为 ModelName 枚举
    # 2. 通过 EndpointDispatcher.dispatch() 获取可用 endpoint
    # 3. 针对不同模型设置特殊参数:
    #    - Gemini 3 Pro/Flash → 设置 thinking_level
    #    - OpenAI o3/o4 Mini   → 设置 reasoning_effort
    # 4. 返回 endpoint.llm 实例
```

模型特殊参数：


| 模型系列               | 参数                 | 配置项                             | 默认值        |
| ------------------ | ------------------ | ------------------------------- | ---------- |
| Gemini 3 Pro/Flash | `thinking_level`   | `summary_card.thinking_level`   | `"low"`    |
| OpenAI o3/o4 Mini  | `reasoning_effort` | `summary_card.reasoning_effort` | `"medium"` |


### 4.5 超时控制

使用 `ThreadPoolExecutor` 实现线程级超时：

```python
def _invoke_llm_with_timeout(self, llm, prompt, callbacks, timeout):
    with ThreadPoolExecutor(max_workers=1) as executor:
        future = executor.submit(lambda: llm.invoke(prompt, config={...}))
        return future.result(timeout=timeout)  # 默认 120s
```

### 4.6 结果提取

`_extract_text_from_result()` 处理不同 LLM 的返回格式：


| 格式                     | 处理方式                             |
| ---------------------- | -------------------------------- |
| `str`                  | 直接返回                             |
| `obj.content` 为 `str`  | 返回 content                       |
| `obj.content` 为 `list` | 遍历提取 text 类型 block，`"\n".join()` |


### 4.7 输出校验

LLM 输出长度 < 50 字符视为无效，返回 None。

---

## 5. 第三阶段：HTML 组装与验证

### 5.1 方法调用链

```
_build_html(html_body)
  └── assemble_html(body_content, language, canvas_width, scenario, summary_id, file_id)
        ├── clean_llm_output(body_content)          # 清洗 LLM 输出
        ├── validate_html(content, scenario)         # 语法 + 语义校验
        │     ├── validate_html_syntax()             # HTMLParser 语法检查
        │     └── check_common_html_errors()         # 正则检查 LLM 常见错误
        ├── fix_html(content)                        # 自动修复（如校验不通过）
        ├── get_fonts_for_language(language)          # 根据语言选字体
        ├── generate_tailwind_config()               # 生成 Tailwind 配置 JS
        └── HTML_SKELETON.format(...)                # 拼接完整 HTML 页面
```

### 5.2 LLM 输出清洗 (`clean_llm_output`)

按顺序执行：

1. **去除 Markdown 代码块标记**：移除 ````html` / ````` 包裹
2. **去除 PLAN 注释块**：移除 `<!-- PLAN: ... -->` (LLM CoT 思考过程)
3. **过滤思考前缀**：找到第一个以 `<` 开头的行，丢弃之前的文本

### 5.3 HTML 校验体系 (`validate_html`)

返回 `ValidationResult` 对象：

```python
@dataclass
class ValidationResult:
    is_valid: bool          # 是否通过所有校验
    errors: list[str]       # 错误列表
    warnings: list[str]     # 警告列表
    has_syntax_error: bool   # 是否有语法错误（严重，直接拒绝）
```

#### 5.3.1 语法校验 (`validate_html_syntax`)

使用 `HTMLSyntaxValidator`（继承 `html.parser.HTMLParser`）检测：


| 检测项   | 说明              |
| ----- | --------------- |
| 未闭合标签 | 有开标签无对应闭标签      |
| 标签不匹配 | 闭标签与栈顶开标签不一致    |
| 意外闭标签 | 闭标签无对应开标签       |
| 解析异常  | HTMLParser 内部错误 |


#### 5.3.2 LLM 常见错误检测 (`check_common_html_errors`)

使用正则 + BeautifulSoup 检测 6 类 LLM 高频错误：


| #   | 错误类型                          | 说明                  | 典型场景                                         |
| --- | ----------------------------- | ------------------- | -------------------------------------------- |
| 1   | `invalid_css_attr`            | `css=` 替代 `class=`  | DeepSeek 常见                                  |
| 2   | `unclosed_attribute`          | 属性引号未闭合             | 如 `class="text-lg text-secondary mt-2policy` |
| 3   | `broken_close_tag`            | 破损闭合标签              | `div>` 替代 `</div>`                           |
| 4   | `incomplete_open_tag`         | 不完整开标签              | `<div` 缺少 `>`                                |
| 5   | `malformed_tailwind_class`    | Tailwind 类名格式错误     | `col-span+8` 应为 `col-span-8`                 |
| 6   | `nested_grid_without_colspan` | 嵌套 grid 缺少 col-span | DeepSeek 常见，导致布局 1/12 宽度                     |


#### 5.3.3 语义校验


| 检测项                      | 级别      | 说明        |
| ------------------------ | ------- | --------- |
| Grid 行 col-span 总和 ≠ 12  | Warning | 布局可能错位    |
| 使用未知颜色                   | Warning | 不在调色板内的颜色 |
| `<i>` 无 `data-lucide` 属性 | Error   | 图标不会渲染    |
| HTML 内容为空                | Error   | 无有效内容     |


#### 5.3.4 校验结果处理

```
has_syntax_error = True  → 直接拒绝，返回空 HTML（严重错误不可修复）
is_valid = False         → 自动修复后继续（非严重错误）
is_valid = True          → 直接使用
```

### 5.4 自动修复 (`fix_html`)


| 修复项        | 说明                                                |
| ---------- | ------------------------------------------------- |
| 空 icon 修复  | 给无 `data-lucide` 的 `<i>` 添加默认 `circle`            |
| 缺失 flex 修复 | 给有 `rounded-card` 但无 `flex` 的卡片添加 `flex flex-col` |


### 5.5 字体配置 (`get_fonts_for_language`)

根据语言代码选择对应的 Google Fonts：


| 语言           | 字体栈                                   | 加载字体                      |
| ------------ | ------------------------------------- | ------------------------- |
| zh-CN (简体中文) | `'Noto Sans SC', 'Inter', sans-serif` | Noto Sans SC + Inter      |
| zh-TW (繁体中文) | `'Noto Sans TC', 'Inter', sans-serif` | Noto Sans TC + Inter      |
| ja (日语)      | `'Noto Sans JP', 'Inter', sans-serif` | Noto Sans JP + Inter      |
| ko (韩语)      | `'Noto Sans KR', 'Inter', sans-serif` | Noto Sans KR + Inter      |
| en 等西方语言     | `'Inter', sans-serif`                 | Inter                     |
| auto / 未知    | 全部加载                                  | Inter + SC + TC + JP + KR |


语言代码标准化逻辑 (`normalize_html_lang`)：


| 输入                         | 标准化输出   | 说明     |
| -------------------------- | ------- | ------ |
| `zh-0`, `zh-cn`, `zh-hans` | `zh-CN` | 简体中文   |
| `zh-1`, `zh-tw`, `zh-hant` | `zh-TW` | 繁体中文   |
| `ja`, `jp`, `ja-jp`        | `ja`    | 日语     |
| `ko`, `kr`, `ko-kr`        | `ko`    | 韩语     |
| `简体中文`, `Chinese`          | `zh-CN` | 显示名称匹配 |


### 5.6 HTML 骨架模板 (`HTML_SKELETON`)

最终组装的完整 HTML 结构：

```html
<!DOCTYPE html>
<html lang="{lang}">
<head>
    <meta charset="UTF-8">
    <script src="https://cdn.tailwindcss.com"></script>
    <script>{tailwind_config}</script>                <!-- 设计系统配置 -->
    <script src="{icon_cdn}"></script>                 <!-- Lucide Icons -->
    <link href="{google_fonts_url}" rel="stylesheet">  <!-- Google Fonts -->
    <style>
        body { font-family: {font_stack}; ... }
        /* flex 防压缩规则 */
        /* 文字自然换行规则 */
    </style>
</head>
<body class="bg-gray-50">
    <div id="canvas" class="inline-block p-4{canvas_extra_class}">
        {watermark}                                    <!-- CN 区域水印 -->
        <div class="w-[{canvas_width}px] flex flex-col gap-4">
            {content}                                  <!-- LLM 生成的 HTML body -->
        </div>
    </div>
    <script>lucide.createIcons();</script>
</body>
</html>
```

### 5.7 CN 区域水印

当 `config.is_cn_region()` 为 True 时，在画布右上角添加水印：

```html
<div class="absolute" style="top: 24px; right: 24px; font-size: 16px; line-height: 26px; color: #7A7A7A;">
    内容由 Plaud AI 生成
</div>
```

水印文案多语言映射（`WATERMARK_TEXT_MAP`）：


| 语言      | 水印文案                   |
| ------- | ---------------------- |
| zh-cn   | 内容由 Plaud AI 生成        |
| zh-tw   | 內容由 Plaud AI 生成        |
| ja      | Plaud AI により生成         |
| ko      | Plaud AI에 의해 생성됨       |
| de      | Von Plaud AI generiert |
| fr      | Généré par Plaud AI    |
| en (默认) | Generated by Plaud AI  |


---

## 6. 第四阶段：HTML 转 PNG

### 6.1 方法

`_convert_html_to_png(html_content: str) -> Optional[bytes]`

### 6.2 外部服务调用

调用无头浏览器渲染服务的 `/api/render` 端点：

```python
POST {html_to_png_url}/api/render
Content-Type: application/json

{
    "html": "<完整 HTML 页面>",
    "javaScriptEnabled": true,
    "waitUntil": "networkidle",        # 等待所有网络请求完成
    "contextKey": "summary-card",       # 浏览器上下文复用 key
    "maxDimension": 1920,               # 最大尺寸
    "deviceScaleFactor": 2.0,           # DPI 缩放因子
    "format": "png",                    # 输出格式
    "pngCompressionLevel": 9,           # 压缩级别（最高）
    "pngEffort": 10,                    # 压缩努力度（最高）
    "pngPalette": true,                 # 使用调色板模式（减小文件大小）
    "quality": 80,                      # 质量
    "pngDither": 0.5                    # 抖动系数
}
```

### 6.3 重试策略

```python
Retry(
    total=3,                                    # 最多 3 次重试
    backoff_factor=0.5,                         # 退避系数
    status_forcelist=[500, 502, 503, 504],      # 仅对 5xx 重试
    allowed_methods=["POST"],                   # 仅对 POST 重试
)
```

退避间隔：第 1 次 0.5s → 第 2 次 1s → 第 3 次 2s

### 6.4 超时控制


| 参数     | 配置项                                      | 默认值 | 说明                        |
| ------ | ---------------------------------------- | --- | ------------------------- |
| 单次请求超时 | `summary_card.html_to_png_timeout`       | 30s | `requests.post(timeout=)` |
| 总超时    | `summary_card.html_to_png_total_timeout` | 60s | 含重试的总耗时上限                 |


### 6.5 图片尺寸提取

从 PNG 字节流直接读取 IHDR chunk（无需解码完整图片）：

```python
def _get_png_dimensions(self, png_data: bytes) -> tuple[int, int]:
    # PNG: [8字节签名][4字节IHDR长度][4字节IHDR类型][4字节width][4字节height]
    width = int.from_bytes(png_data[16:20], byteorder='big')
    height = int.from_bytes(png_data[20:24], byteorder='big')
```

---

## 7. 第五阶段：S3 上传与存储

### 7.1 方法

`_upload_to_s3(png_data: bytes) -> Optional[str]`

### 7.2 存储路径

路径格式：`{storage_type}/{user_id}/{content_type}/{filename}`


| 组成部分           | 值                                           | 说明         |
| -------------- | ------------------------------------------- | ---------- |
| `storage_type` | `permanent` 或 `temporary_30_days`           | 由 PPC 状态决定 |
| `user_id`      | 用户 ID 或 `"global"`                          | 用户标识       |
| `content_type` | `summary_poster`                            | 固定值        |
| `filename`     | `card_{summary_id}_{timestamp}_{uuid8}.png` | 唯一文件名      |


**PPC 状态 → 存储类型映射**：


| PPC 状态                   | 存储类型                | 过期策略    |
| ------------------------ | ------------------- | ------- |
| `PpcStatus.ENABLED` (1)  | `permanent`         | 永久保存    |
| `PpcStatus.DISABLED` (2) | `temporary_30_days` | 30 天后过期 |


完整路径示例：

```
permanent/4bc87de7ffae43dfaeddd3deda74b5e9/summary_poster/card_abc123_20260413120000_a1b2c3d4.png
```

### 7.3 CN 区域 AIGC 元数据嵌入

当 `config.is_cn_region()` 为 True 时，在 S3 上传前将 AIGC 合规元数据写入 PNG 文件的 tEXt chunk：

```python
def _add_aigc_metadata_to_png(self, png_data: bytes) -> bytes:
    img = Image.open(io.BytesIO(png_data))
    png_info = PngImagePlugin.PngInfo()
    png_info.add_text("AIGC", json.dumps({
        "Label": "1",                                    # AIGC 内容标记
        "ContentProducer": "plaud-summary",               # 内容生产者
        "ProduceID": self.file_id,                        # 生产 ID
        "ReservedCode1": MD5(self.file_id),               # 保留码 1
        "ContentPropagator": "Plaud",                     # 内容传播者
        "PropagateID": self.summary_id,                   # 传播 ID
        "ReservedCode2": MD5(self.summary_id),            # 保留码 2
        "DisclaimerText": "内容由 Plaud AI 生成",          # 免责声明（多语言）
    }))
    img.save(output, format="PNG", pnginfo=png_info)
```

使用 Pillow (`PIL`) 库操作 PNG 元数据，失败时返回原始 PNG（不阻断流程）。

### 7.4 S3 上传

```python
s3_client = boto3.client('s3', region_name=settings.AWS_REGION)
s3_client.put_object(
    Bucket=bucket_name,           # config.get_s3_bucket_name_content_storage()
    Key=storage_key,              # 完整存储路径
    Body=png_data,                # PNG 字节（含 AIGC 元数据）
    ContentType='image/png'
)
```

返回值是 `storage_key`（非预签名 URL），客户端自行转换为可访问的 URL。

---

## 8. 第六阶段：插入 Markdown

### 8.1 方法

`insert_card_to_markdown(markdown_content: str, card_storage_key: str) -> str`（静态方法）

### 8.2 Markdown 格式

```markdown
![PLAUD NOTE](storage_key)
```

### 8.3 插入位置逻辑

```python
# 遍历所有行，找到第一个 H1 标题（# 开头但非 ## 开头）
for i, line in enumerate(lines):
    if line.strip().startswith('# ') and not line.strip().startswith('## '):
        insert_index = i + 1
        break

# 有 H1: 插入在 H1 之后，前后各加空行
# 无 H1: 插入在文档开头
```

插入后效果：

```markdown
# 会议总结

![PLAUD NOTE](permanent/user123/summary_poster/card_xxx.png)

## 关键要点
...
```

### 8.4 移除逻辑

`remove_card_from_markdown()` 用于审核失败时清理：

```python
# 1. 找到包含 "![PLAUD NOTE](storage_key)" 的行
# 2. 删除该行
# 3. 智能删除前后的空行（避免留下多余空行）
```

---

## 9. 第七阶段：内容审核与清理

### 9.1 审核数据准备

在 `basic_runnable_summary.py` 中，生成头图后立即提取文本：

```python
# 从 HTML body 中提取纯文本用于审核
card_html_body = card_generator.last_html_body
card_text = extract_text_from_html(card_html_body)
extra_moderation = {
    "card_text": card_text,          # 纯文本，供审核使用
    "card_storage_key": card_storage_key,  # S3 key，审核失败时用于删除
}
```

`extract_text_from_html()` 使用 BeautifulSoup 提取：

- 用空格分隔所有文本节点
- 合并多个连续空白字符为单个空格
- 解析失败时返回原始 HTML（确保审核覆盖）

### 9.2 审核触发

在 `summary.py` 的内容审核流程中，card_text 与总结 Markdown 一起提交审核：

```python
# 组合审核文本 = summary markdown + card_text
audit_text = markdown_content
if extra_moderation and extra_moderation.get("card_text"):
    audit_text += "\n" + extra_moderation["card_text"]

result = _moderation_service.audit_text(audit_text)
```

### 9.3 审核失败清理

当审核失败时，执行三步清理：

```python
# 1. 从 S3 删除图片
SummaryCardGenerator.delete_from_s3(card_storage_key, summary_id)

# 2. 从 Markdown 中移除头图引用
result["result"]["markdown"] = SummaryCardGenerator.remove_card_from_markdown(
    result["result"]["markdown"], card_storage_key
)

# 3. 清空整个 Markdown（审核失败不返回任何内容）
result["result"]["markdown"] = ""
```

### 9.4 元数据清理

无论审核是否通过，都会从 result 中删除 `_extra_moderation` 字段，避免泄露到调用方：

```python
if "_extra_moderation" in result:
    del result["_extra_moderation"]
```

---

## 10. 设计系统

### 10.1 文件

`plaud_summary/services/summary_card/design_system.py`

### 10.2 调色板 (9 套)

每套调色板包含 100-900 共 9 个色阶：


| 调色板             | 风格        | 适用场景      |
| --------------- | --------- | --------- |
| `honey-tea`     | 温暖、传统、舒适  | 日常工作、历史内容 |
| `almond-cream`  | 中性、经典、优雅  | 基础文档、通用内容 |
| `willow-bud`    | 成长、生态、积极  | 成长主题、环保内容 |
| `sea-mist`      | 创新、平静、空间感 | 创新讨论、开放话题 |
| `glacier-blue`  | 技术、数据、结构化 | 技术内容、数据分析 |
| `orchid-grey`   | 正式、权威、深度  | 正式场合、决策内容 |
| `lavender-mist` | 创意、艺术、放松  | 创意讨论、头脑风暴 |
| `sakura-pink`   | 活力、社交、情感  | 社交互动、庆祝   |
| `soft-rose`     | 关怀、温暖、包容  | 人文关怀、团队互动 |


色值示例（glacier-blue）：


| 色阶  | 色值        | 用途             |
| --- | --------- | -------------- |
| 100 | `#F7FBFC` | 最浅背景 / 嵌套子卡片背景 |
| 200 | `#ECF5F9` | 标准卡片背景         |
| 300 | `#DDECF3` | Tag 背景         |
| 400 | `#CEE4EE` | 强调卡片背景 / 卡片边框  |
| 500 | `#B3D5E5` | 装饰色            |
| 600 | `#ACCCDC` | 进度条填充          |
| 700 | `#96B3C0` | 连接线、装饰线        |
| 800 | `#738893` | 暗色文本           |
| 900 | `#5A6B73` | 图标色（彩色背景上）     |


每套调色板还有专属语义色（`palette_semantic_colors`），用于在彩色背景上表示成功/错误/警告。

### 10.3 文字色彩体系


| 名称          | 色值        | 用途          |
| ----------- | --------- | ----------- |
| `primary`   | `#000000` | 标题、正文主色     |
| `secondary` | `#3D3D3D` | 次要文本        |
| `tertiary`  | `#7A7A7A` | 仅用于水印（正文禁用） |
| `error`     | `#FF503F` | 错误/负面指标     |
| `success`   | `#36D96C` | 成功/正面指标     |
| `warning`   | `#FABE3E` | 警告/进行中      |


### 10.4 字体排版体系


| 类名            | 大小   | 行高   | 字重  | 用途                         |
| ------------- | ---- | ---- | --- | -------------------------- |
| `display-2xl` | 72px | 90px | 700 | 英雄数字（单个 KPI）               |
| `display-xl`  | 60px | 72px | 700 | 大数字（2 个指标）                 |
| `display-lg`  | 48px | 60px | 700 | 数据网格（3-4 个指标）              |
| `display-md`  | 36px | 48px | 700 | 页面主标题                      |
| `display-sm`  | 32px | 44px | 700 | 章节标题                       |
| `display-xs`  | 28px | 36px | 700 | 大卡片标题                      |
| `text-xl`     | 24px | 36px | 700 | 卡片标题                       |
| `text-lg`     | 20px | 30px | 600 | 副标题                        |
| `text-md`     | 18px | 28px | 400 | **默认正文**（90%+ 正文用这个）       |
| `text-sm`     | 16px | 26px | 500 | **最小字号**，仅用于 badge/tag/时间戳 |


### 10.5 图标系统

使用 Lucide Icons（[https://unpkg.com/lucide@latest），SVG](https://unpkg.com/lucide@latest），SVG) 图标库：

```html
<i data-lucide="icon-name" class="w-5 h-5"></i>
```

通过 `lucide.createIcons()` 在页面加载后将 `<i>` 替换为 SVG。

### 10.6 Tailwind 配置生成

`generate_tailwind_config()` 动态生成 Tailwind 配置 JS：

```javascript
tailwind.config = {
    "theme": {
        "fontSize": { /* 10 个自定义字号 */ },
        "extend": {
            "colors": { /* 调色板所有色值 + 文字色 + 语义色 */ },
            "borderRadius": { "card": "5px" },
            "borderWidth": { "card": "1px" }
        }
    }
}
```

---

## 11. Prompt 工程（LLM 指令体系）

内置 Fallback Prompt 结构（约 1240 行），分为以下核心模块：

### 11.1 Bento 设计理念

要求 LLM 将信息像日式便当盒一样排列在"格间"中：

- **模块化**：每个格间是独立的信息单元
- **层级**：用大小和颜色区分重要性
- **留白**：不填满所有空间
- **平衡**：稳定构图，清晰视觉焦点

### 11.2 通用布局约束（5 条强制规则）


| 规则                | 说明                                      |
| ----------------- | --------------------------------------- |
| Rule 1: 垂直对齐      | 图标/文字对齐方式取决于文本行数和元素角色。禁止用 `mt-`* 调整图标位置 |
| Rule 2: 固定容器内容约束  | 圆形容器只能放 1-2 个字符，≥3 位数字必须用药丸形状           |
| Rule 3: Flex 容器宽度 | 禁止在 `flex-wrap` 容器上设置 `max-w-*`         |
| Rule 4: 禁止元信息     | 不输出 "Generated by..." 等生成过程相关文字         |
| Rule 5: 禁止推断日期    | 严格保留原文中的日期表述，不做补全                       |


### 11.3 信息编排策略（4 层结构）


| 层级  | 名称                | 定义                          |
| --- | ----------------- | --------------------------- |
| L1  | Context Anchor    | 页面主标题，< 8 个词                |
| L2  | Narrative Pillars | 2-5 个核心维度（逻辑骨架）             |
| L3  | Adaptive Nodes    | 具体证据（Rule A 直引 / Rule B 精简） |
| L4  | Action Trigger    | 待办事项（仅有明确任务时激活）             |


### 11.4 L2 叙事架构策略

根据内容类型选择首个卡片的聚焦策略：


| 策略            | 触发条件    | 首卡焦点  | 展开逻辑    |
| ------------- | ------- | ----- | ------- |
| Problem First | 有风险/危机  | 关键阻碍  | 根因 → 方案 |
| Outcome First | 有决策/结果  | 最终结论  | 理由 → 影响 |
| Status First  | 多线程状态   | 总体仪表盘 | 详情 → 下步 |
| Anchor First  | 静态知识/指南 | 核心定义  | 机制 → 影响 |


### 11.5 生成流程（严格 4 阶段）


| 阶段      | 名称                | 输出                                     |
| ------- | ----------------- | -------------------------------------- |
| Phase 1 | Content Inventory | 内容清单（CRITICAL/IMPORTANT/SUPPORTING 分类） |
| Phase 2 | Layout Blueprint  | 卡片布局蓝图（每个卡片的 col-span + 内容映射）          |
| Phase 3 | Pre-flight Check  | 7 项预检（空卡片/节点数/高度/30秒测试等）               |
| Phase 4 | HTML Output       | 严格按蓝图生成 HTML                           |


LLM 输出格式：先输出 `<!-- PLAN: ... -->` 注释块（Phase 1-3），再输出 HTML。

### 11.6 组件库（12 类）


| #   | 组件                     | 适用场景              |
| --- | ---------------------- | ----------------- |
| 1   | Badge/Tag              | 标签、分类、状态标记        |
| 2   | Progress Bar           | 进度展示（单色/多段）       |
| 3   | Timeline               | 时间线、流程回顾          |
| 4   | Quote Block            | 引用、金句             |
| 5   | Data Comparison        | 数据对比（前后/优劣）       |
| 6   | Steps/Process          | 步骤流程（水平/垂直）       |
| 7   | Person Card            | 人物信息卡             |
| 8   | Stats Grid             | 数据面板              |
| 9   | List Variants          | 列表（图标/编号/分组）      |
| 10  | Status Indicators      | 状态徽章              |
| 11  | Dividers & Decorations | 分割线、装饰            |
| 12  | Embeddable Elements    | Tag 组/头像组/图标网格/评分 |


---

## 12. 监控与埋点

### 12.1 Prometheus 指标

定义在 `plaud_summary/services/summary_card/metrics.py`：


| 指标名                                          | 类型        | 单位    | 说明                 |
| -------------------------------------------- | --------- | ----- | ------------------ |
| `summary_card_monitor_counter`               | Counter   | -     | 任务执行计数（成功/失败）      |
| `summary_card_monitor_html_validation_error` | Counter   | -     | HTML 校验错误计数（按错误类型） |
| `summary_card_monitor_llm_latency`           | Histogram | ms    | LLM 生成耗时           |
| `summary_card_monitor_html_latency`          | Histogram | ms    | HTML 组装耗时          |
| `summary_card_monitor_png_latency`           | Histogram | ms    | PNG 转换耗时           |
| `summary_card_monitor_s3_latency`            | Histogram | ms    | S3 上传耗时            |
| `summary_card_monitor_total_latency`         | Histogram | ms    | 总耗时                |
| `summary_card_monitor_png_size`              | Histogram | bytes | PNG 文件大小           |
| `summary_card_monitor_png_width`             | Histogram | px    | 图片宽度               |
| `summary_card_monitor_png_height`            | Histogram | px    | 图片高度               |
| `summary_card_monitor_markdown_length`       | Histogram | chars | 输入 Markdown 长度     |


Histogram 桶边界配置：


| 指标          | 桶边界                                                  |
| ----------- | ---------------------------------------------------- |
| 耗时          | 0, 100ms, 500ms, 1s, 2s, 5s, 10s, 30s, 60s           |
| 文件大小        | 0, 10KB, 50KB, 100KB, 500KB, 1MB, 2MB, 5MB           |
| 分辨率         | 0, 400, 600, 800, 1000, 1200, 1400, 1600, 2000, 2800 |
| Markdown 长度 | 0, 500, 1000, 2000, 5000, 10000, 20000, 50000        |


### 12.2 事件埋点

定义在 `plaud_summary/services/tracking/events.py`：

#### `track_card` — 非 PII 指标事件

- **事件类型**：`summary.metrics.card`
- **触发时机**：生成成功 或 失败
- **主要字段**：


| 字段                                 | 类型   | 说明                       |
| ---------------------------------- | ---- | ------------------------ |
| `status`                           | str  | `"success"` 或 `"failed"` |
| `scenario`                         | str  | 场景类型                     |
| `language`                         | str  | 语言                       |
| `bs_class`                         | str  | 业务类名                     |
| `ppc_status`                       | int  | PPC 状态                   |
| `llm_elapsed_ms`                   | int  | LLM 耗时                   |
| `html_elapsed_ms`                  | int  | HTML 组装耗时                |
| `png_elapsed_ms`                   | int  | PNG 转换耗时                 |
| `s3_elapsed_ms`                    | int  | S3 上传耗时                  |
| `total_elapsed_ms`                 | int  | 总耗时                      |
| `png_size_bytes`                   | int  | PNG 大小                   |
| `png_width` / `png_height`         | int  | 图片尺寸                     |
| `markdown_length`                  | int  | 输入 Markdown 长度           |
| `html_body_length` / `html_length` | int  | HTML body / 完整 HTML 长度   |
| `validation_error_count`           | int  | 校验错误数                    |
| `has_syntax_error`                 | bool | 是否有语法错误                  |
| `failed_stage`                     | str  | 失败阶段                     |
| `error_message`                    | str  | 错误消息                     |


#### `track_pii_card_content` — PII 内容事件

- **事件类型**：`summary.pii.card_content`
- **触发时机**：LLM 输出 html_body 后（无论后续是否成功）
- **包含 PII**：`html_body`（LLM 生成的原始 HTML）, `html_content`（组装后的完整 HTML）
- **用途**：调试 LLM 输出质量

### 12.3 Langfuse 追踪

通过 `LLMMonitorCallbackHandler` 和 `langfuse_callback_handler` 集成：

- 记录 LLM 调用的输入/输出/耗时/token 使用
- 关联 `summary_id` 和 `scenario` 元数据

---

## 13. 错误处理策略

### 13.1 总体原则

**头图生成失败不阻塞总结主流程。** 所有异常均被捕获并记录，最终 `generate()` 返回 `None`。

### 13.2 各阶段失败处理


| 阶段            | `failed_stage`   | 处理方式                       |
| ------------- | ---------------- | -------------------------- |
| LLM 生成超时/失败   | `llm_generation` | 记录日志 + 埋点，返回 None          |
| HTML 语法错误（严重） | `html_assembly`  | 拒绝生成图片，不做自动修复              |
| HTML 非严重错误    | -                | 自动修复后继续                    |
| PNG 渲染服务不可用   | `html_to_png`    | 3 次重试后放弃                   |
| S3 上传失败       | `s3_upload`      | 记录日志 + 埋点，返回 None          |
| 任何未预期异常       | `exception`      | 外层 try-except 捕获，返回 None   |
| 内容审核失败        | -                | 删除 S3 + 清除 Markdown + 清空结果 |


### 13.3 调用方容错

`basic_runnable_summary.py` 中的调用：

```python
try:
    card_storage_key = card_generator.generate(summary_markdown)
    if card_storage_key:
        result["markdown"] = SummaryCardGenerator.insert_card_to_markdown(...)
except Exception as e:
    self._log_warning(f"Failed to generate summary card, continuing without it: {e}")
```

即使 `generate()` 内部异常逃逸，外层也会捕获并继续。

---

## 14. 配置参数大全

所有配置通过 AWS AppConfig 管理，定义在 `plaud_summary/config.py`：


| 配置项                                      | 类型    | 默认值                | 说明                                          |
| ---------------------------------------- | ----- | ------------------ | ------------------------------------------- |
| `summary_card.enabled`                   | bool  | `false`            | 全局开关                                        |
| `summary_card.rollout_percentage`        | float | `0.0`              | 灰度放量比例 (0.0-1.0)                            |
| `summary_card.allowed_scenarios`         | str   | `""`               | 场景白名单（逗号分隔）。空=全禁，`"*"`=全允                   |
| `summary_card.min_tokens`                | int   | `500`              | 原始内容最低 token 数（约 2 分钟语音）                    |
| `summary_card.min_summary_tokens`        | int   | `200`              | 总结最低 token 数                                |
| `summary_card.llm_model`                 | str   | `"gemini-2.5-pro"` | LLM 模型名称                                    |
| `summary_card.llm_timeout`               | int   | `120`              | LLM 调用超时（秒）                                 |
| `summary_card.thinking_level`            | str   | `"low"`            | Gemini thinking 级别 (low/high)               |
| `summary_card.reasoning_effort`          | str   | `"medium"`         | OpenAI o3/o4 reasoning 级别 (low/medium/high) |
| `summary_card.html_to_png_url`           | str   | `""`               | 渲染服务 URL                                    |
| `summary_card.html_to_png_timeout`       | int   | `30`               | 渲染单次请求超时（秒）                                 |
| `summary_card.html_to_png_total_timeout` | int   | `60`               | 渲染总超时（含重试，秒）                                |
| `summary_card.max_dimension`             | int   | `1920`             | 画布最大尺寸（像素）                                  |
| `summary_card.device_scale_factor`       | float | `2.0`              | 设备缩放因子（DPI）                                 |
| `summary_card.image_format`              | str   | `"png"`            | 输出图片格式                                      |
| `summary_card.png_palette`               | bool  | `true`             | PNG 调色板模式（减小文件大小）                           |
| `summary_card.image_quality`             | int   | `80`               | 图片质量 (0-100)                                |
| `summary_card.png_dither`                | float | `0.5`              | PNG 抖动系数                                    |


---

## 15. 数据流总览

### 15.1 正常流程数据流

```
输入: summary_markdown (str)
  │
  ▼ [_should_generate_summary_card]
  判断: tokens_lens, summary_markdown → (bool, reason)
  │
  ▼ [SummaryCardGenerator.__init__]
  初始化: summary_id, language, user_id, ppc_status, scenario, file_id, bs_class_name
  │
  ▼ [_generate_html_body]
  Langfuse/Fallback Prompt + {canvas_width, palettes, language, content}
  → LLM invoke → raw_text (HTML body fragment)
  │
  ▼ [_build_html → assemble_html]
  raw_text → clean_llm_output → validate_html → fix_html
  → HTML_SKELETON.format(lang, tailwind_config, icon_cdn, google_fonts_url,
                          font_stack, canvas_width, watermark, content)
  → full_html (str)
  │
  ▼ [_convert_html_to_png]
  full_html → POST /api/render → png_data (bytes)
  │
  ▼ [_upload_to_s3]
  png_data → _add_aigc_metadata_to_png (CN) → s3_client.put_object
  → storage_key (str)
  │
  ▼ [insert_card_to_markdown]
  markdown + storage_key → "![PLAUD NOTE](storage_key)" 插入 H1 后
  → updated_markdown (str)
  │
  ▼ [extract_text_from_html]
  html_body → BeautifulSoup.get_text → card_text (str)
  → extra_moderation = {card_text, card_storage_key}
  │
  ▼ [内容审核]
  markdown + card_text → audit_text → moderation_service
  │ 通过
  ▼
  返回 updated_markdown（含头图引用）
```

### 15.2 审核失败数据流

```
audit_text → moderation_service → 审核不通过
  │
  ├── delete_from_s3(card_storage_key)        # 删除 S3 图片
  ├── remove_card_from_markdown(markdown, key) # 移除 Markdown 引用
  ├── result["markdown"] = ""                  # 清空 Markdown
  └── del result["_extra_moderation"]          # 清理内部字段
```

### 15.3 关键对象生命周期


| 对象                        | 创建时机                        | 销毁/清理时机                                  |
| ------------------------- | --------------------------- | ---------------------------------------- |
| `SummaryCardGenerator` 实例 | 条件判断通过后                     | 方法调用结束后 GC                               |
| `last_html_body`          | `_generate_html_body()` 返回后 | 随 Generator 实例销毁                         |
| `extra_moderation`        | 头图生成成功后                     | 审核完成后 `del result["_extra_moderation"]`  |
| S3 存储的 PNG                | `_upload_to_s3()` 成功后       | 审核失败时 `delete_from_s3()`；PPC 关闭时 30 天后过期 |
| Markdown 中的头图引用           | `insert_card_to_markdown()` | 审核失败时 `remove_card_from_markdown()`      |


