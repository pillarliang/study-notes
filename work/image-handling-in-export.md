# 文件内容中图片处理机制说明

本文档梳理 plaud-api、plaud-document、plaud-tools、plaud-ask 四个项目在"文件内容包含图片"场景下的处理方式及差异。

---

## 1. 图片在内容中的存储形式

文件内容（summary、ask_note、mark_memo 等）中的图片以 Markdown 语法存储：

```python
![PLAUD NOTE](permanent/cb9023f1a54c8eb516e5ece6e5c2afea/mem_bckQfRCzdf/ask_note/note:ca3fa132:xxx_yyy)
```

括号内是 **S3 对象的 key**（相对路径），不是可直接访问的 URL。

---

## 2. summary 服务 → plaud-api → 前端：图片 URL 的生命周期

这是理解整个图片处理机制的核心链路。

### 完整流程

```
summary 服务                       plaud-api (detail)                          前端
     │                                  │                                       │
     │  S3 key 写入 markdown             │                                       │
     │  (永久存储，不过期)                │                                       │
     ├──────────────────────────────────→│                                       │
     │                                  │  1. 读取 markdown（不修改原文）          │
     │                                  │  2. 扫描 ![PLAUD NOTE](s3_key)         │
     │                                  │  3. generate_presigned_url(key, 900)   │
     │                                  │  4. 返回：                              │
     │                                  │     content = 原始 md（含 S3 key）      │
     │                                  │     download_path_mapping =            │
     │                                  │       {s3_key: presigned_url}          │
     │                                  ├──────────────────────────────────────→│
     │                                  │                                       │  用 mapping 替换
     │                                  │                                       │  S3 key → presigned URL
     │                                  │                                       │  渲染 <img src="url">
```

### 关键设计：内容与链接分离

plaud-api **不修改 markdown 原文**，而是返回两个独立字段：

```json
{
  "content_list": [
    { "data_content": "...![PLAUD NOTE](permanent/xxx/yyy)..." }
  ],
  "download_path_mapping": {
    "permanent/xxx/yyy": "https://...s3...?X-Amz-Expires=900&X-Amz-Signature=..."
  }
}
```

- **markdown 中始终保留 S3 key**：不可直接访问，但永久有效
- **presigned URL 在 mapping 中单独返回**：可直接访问，但有过期时间

### presigned URL 的过期特性

`download_path_mapping` 中的每个 URL 都是 S3 presigned URL，有效期 **15 分钟**（`X-Amz-Expires=900`）。过期后：

- S3 返回 **403 Forbidden**，图片无法加载
- 前端需要**重新调用 detail 接口**获取新的 mapping（新的 presigned URL）
- markdown 原文中的 S3 key 不受影响，任何时候都可以重新签名

这也是为什么"内容与链接分离"是合理的设计：S3 key 是稳定的（永久有效），presigned URL 是临时的（15 分钟过期），两者各归其位。

---

## 3. plaud-api：detail 接口的 download_path_mapping 实现细节

**入口**：`GET /api/file/detail/{file_id}`

**核心函数**：`make_download_link_map_from_sources()`（`service/download_link.py`）

**流程**：

1. **收集**：从 `auto_sum_note`、`mark_memo`、`high_light`、`ask_note` 等字段中提取图片路径
   - Markdown 格式：正则 `![PLAUD NOTE](path)` 提取 path
   - highlight JSON：提取 `picture_link` 字段
2. **分类**：
   - `http://` / `https://` 开头 → HTTP 直链，透传（不签名）
   - 其余 → S3 相对路径，需要签名
3. **S3 路径处理**：
   - 查找临时路径 → 永久路径的映射（`ExtraData` 表）
   - 校验路径中的 user_id 是否匹配（`_has_uid_permission`）
   - 调用 S3 SDK 生成 presigned URL（TTL = 900 秒 / 15 分钟）
4. **返回**：`download_path_mapping = {原始S3 key: presigned URL}`

前端拿到 mapping 后，在页面渲染时将内容中的 S3 key 替换为 presigned URL 来展示图片。

---

## 3. plaud-api → plaud-document：导出 PDF（summary 服务产生的图片）

**入口**：`POST /api/file/document/export`（`api/file/document.py`）

### plaud-api 侧

plaud-api **不做任何图片处理**，将前端传来的 `summary_content`（含原始 S3 key）原样透传给 plaud-document 服务：

```python
data = {
    "summary_content": summary_content,  # 包含 ![PLAUD NOTE](s3_key)
    ...
}
download_url = request_document_export(data)  # HTTP POST 到 Document 服务
```

### plaud-document 侧

plaud-document（Go 项目）自行处理图片，流程因导出格式而异：

#### PDF / DOCX 格式

1. **提取**：正则 `![alt](path)` 从内容中提取所有图片路径并去重
2. **生成 presigned URL**：对 S3 key 调用 `GenImageTmpURL()`，用 `config.S3.ContentStorageBucket` 桶生成 presigned URL（有效期 120 分钟）
3. **下载到本地**：用生成的 presigned URL 通过 HTTP GET 下载图片到临时目录 `/tmp/plaud_export_{fileID}_{timestamp}/`
4. **图片预处理**：
   - 读取 EXIF 方向信息，按需旋转（90°/180°/270°）
   - 缩放到 80% 宽度（`picture_list.normal` 中的图片保持原尺寸）
   - JPEG + 需旋转 → 转为 PNG（去除 EXIF）
5. **替换路径**：将 Markdown 中的 S3 key 替换为本地文件路径
   - PDF：使用 HTML `<img>` 标签（normal 图片 `width:100%`，其他 `width:40%`）
   - DOCX：标准 Markdown `![](local_path)`
6. **转换**：调用 mdout/pandoc 生成最终文件
7. **上传**：结果文件上传到 S3，返回 presigned 下载链接（20 分钟有效）
8. **清理**：删除临时目录

#### Markdown 等其他格式

不下载图片，直接将 S3 key 替换为 presigned URL（120 分钟有效），保留在输出文本中。

### 关键配置

| 配置项 | 值 | 说明 |
|--------|-----|------|
| S3 桶 | `config.S3.ContentStorageBucket` | 图片所在的 S3 桶 |
| 图片 presigned URL 有效期 | 120 分钟 | plaud-document 内部生成 |
| 并发下载 | 最多 10 个 goroutine | 图片数 ≤ 2 时串行 |
| 默认缩放比例 | 0.8 | picture_list.normal 中的除外 |

---

## 4. plaud-tools：AI 智能体中的图片处理

plaud-tools 中涉及图片的场景有三个工具，处理方式各不相同。

### 4.1 summary_to_markdown —— 内容清洗

**文件**：`packages/tools/src/plaud_tools/formats/summary_markdown.py`

`ImageTransform` 的处理逻辑：

```python
if url.startswith("http"):
    return m.group(0)       # 已是完整 URL，原样保留
if self._url_resolver is None:
    return ""               # 无 resolver，删除图片引用
resolved = self._url_resolver(url)
return f"![{alt}]({resolved})" if resolved else ""
```

- 入参 `image_path_mapping`：`{S3 key: presigned URL}`（由调用方从 plaud-api detail 接口的 `download_path_mapping` 获取）
- S3 key → 通过 mapping 查找 presigned URL → 替换
- mapping 中找不到 → 删除该图片引用

### 4.2 draft_email —— 起草邮件（调用 LLM）

**文件**：`packages/tools/src/plaud_tools/integrations/drafting.py`

流程：

1. `extract_images(content)`：将 `![alt](url)` 替换为 `[IMAGE_N: desc]` 占位符，URL 存入 `ImageRef` 列表
2. 内容分块后判断：
   - **≤ 2 块**（大多数邮件）：走 `_draft_single`，图片 URL 作为多模态 `image_url` 传给 LLM API
   - **> 2 块**：走 map-reduce，**不传图片给 LLM**
3. LLM 返回包含 `[IMAGE_N: desc]` 占位符的草稿
4. `reattach_images()`：将占位符替换回原始 `![alt](url)`

**注意**：步骤 2 中，LLM 服务端会主动 HTTP GET 图片 URL。如果传入的是 presigned URL 且已过期（如 `X-Amz-Expires=900`，仅 15 分钟），LLM 拉取会失败。

### 4.3 send_email —— 发送邮件

**文件**：`packages/tools/src/plaud_tools/integrations/gmail.py`

```python
html_body = mistune.html(body)  # Markdown → HTML
```

**不涉及 LLM**，不做图片提取/替换。`![alt](url)` 直接被 mistune 转为 `<img src="url">`，随 HTML 邮件发出。

如果 `url` 是 presigned URL，收件人打开邮件时若 URL 已过期，图片将无法加载（裂图），但代码层面不会报错。

---

## 5. plaud-ask：图片生成 Skill 的处理

plaud-ask 与前述项目不同，它是**自行生成图片**（通过 LLM），而非处理已有的图片 S3 key。

### 5.1 触发方式

`image_generator` 是 ask 服务的一个官方 skill，用户可通过三种方式触发：

- **action** — 底部按钮点击
- **keep_asking** — 追问点击
- **其他** — 用户输入文字描述生图

不同触发方式会从 Langfuse 获取不同的 system prompt。

**入口**：`ask/tools/image_generator.py` → `image_generator()`

### 5.2 图片生成（LLM 调用）

1. **模型选择**：主模型 `Gemini 3 Pro Image`，遇到 429 限流则降级到 `Gemini 3.1 Flash Image`
2. **Prompt 组装**：用户问题 + 笔记摘要（`note_ask_auto_summary`）+ 语言 → `ChatPromptTemplate`
3. **调用 LLM**：返回包含 `image_url`（data URI 格式的 base64 图片）和 `text_chunks`（文字描述）
4. **重试机制**：如果首次没生成图片，用 fallback prompt（要求生成极简风白底信息图）再试一次

### 5.3 图片存储（S3）与短链接设计

生成图片后的处理流程：

```
LLM 返回 data URI                 S3                          Redis                    对话历史
     │                             │                            │                         │
     │  1. base64 解码为图片字节    │                            │                         │
     │  2. 超过 200KB → JPEG 压缩  │                            │                         │
     │                             │                            │                         │
     ├────── save_image() ────────→│                            │                         │
     │   key: permanent/{user_id}/ │                            │                         │
     │        {member_id}/         │                            │                         │
     │        ask_image/           │                            │                         │
     │        {note_id}_{uuid8}    │                            │                         │
     │                             │                            │                         │
     │←── presigned URL (1h) ──────┤                            │                         │
     │                             │                            │                         │
     ├─────────────────────────────┼──── 缓存 presigned URL ──→│ TTL=30min               │
     │                             │                            │                         │
     │  3. 构造短链接占位符         │                            │                         │
     │     https://ask.plaud.ai/   │                            │                         │
     │     image/{uuid8}           │                            │                         │
     │                             │                            │                         │
     ├─────────────────────────────┼────────────────────────────┼── ![image](短链接) ───→│
     │                             │                            │    (持久化存储)          │
```

**关键设计：短链接作为稳定中间层**

ask 服务**不直接在对话历史中存储 S3 key 或 presigned URL**，而是存储一个虚拟短链接：

```
https://ask.plaud.ai/image/{uuid8}
```

这个短链接本身不可直接访问，它的唯一作用是**携带 `uuid8` 标识**。还原时靠 `uuid8` + 调用方上下文（`user_id`、`note_id`）重建出完整的 S3 key：

```python
# 生成时
short_uuid = str(uuid.uuid4())[:8]                        # 如 "a1b2c3d4"
storage_key = f"permanent/{user_id}/{member_id}/ask_image/{note_id}_{short_uuid}"
short_url   = f"https://ask.plaud.ai/image/{short_uuid}"  # 存入对话历史

# 还原时（message_utils.py）
pattern = re.compile(r"https?://ask\.plaud\.ai/image/([A-Za-z0-9_-]+)")
short_id = "a1b2c3d4"                                     # 正则提取
key = f"{note_id}_{short_id}"                              # 拼回 S3 key 的末段
# → build_storage_path_candidates() 重建完整 S3 路径
# → generate_download_presigned_url() 签名
# → 替换短链接为 presigned URL
```

这样对话历史中存的永远是稳定的短链接，presigned URL 每次都是现签发的。

### 5.4 返回给客户端

图片以两种方式同时返回：

**实时流式推送（SSE media 事件）**：

```python
stream_writer(StreamProtocol.create_media_event({
    "type": "image",
    "url": presigned_url,       # 可直接访问的 S3 presigned URL
    "file_key": saved_key,      # S3 存储 key
    "bucket_name": bucket_name  # S3 桶名
}))
```

**函数返回值**（用于持久化）：

```python
final_answer = f"{text_content}\n\n![image](https://ask.plaud.ai/image/{short_uuid})"
return final_answer, [{"short_url": short_url, "value": presigned_url}]
```

### 5.5 历史消息中的图片恢复

加载对话历史时，`utility/message_utils.py` → `_replace_short_urls_with_presigned()` 执行还原：

1. 正则扫描所有 `https://ask.plaud.ai/image/{id}` 短链接
2. 用 `user_id` + `note_id` + `short_id` 重建 S3 key（会生成多个候选路径以兼容 workspace 迁移）
3. 先查 **Redis 缓存**（TTL 30 分钟），命中则直接使用
4. 未命中则检查 S3 对象是否存在 → 生成新 presigned URL → 缓存到 Redis
5. 将短链接替换为 presigned URL 返回前端

### 5.6 其他处理

- **旧版 App 兼容**：`_remove_old_version_app_image()` 直接删除所有 `![](...)` markdown 图片语法
- **对话删除时清理**：`_delete_images_for_note()` 从短链接还原 S3 key 并删除 S3 对象

---

## 6. 四个项目对比

| 维度 | plaud-api (detail) | plaud-document (export) | plaud-tools | **plaud-ask (image skill)** |
|------|-------------------|------------------------|-------------|---------------------------|
| **图片来源** | 内容中的 S3 key | 内容中的 S3 key | 已替换为 presigned URL 或完整 URL | **LLM 实时生成（data URI）** |
| **谁上传 S3** | 上游 summary 服务 | 上游 summary 服务 | 不上传 | **ask 服务自身** |
| **谁生成 presigned URL** | plaud-api 自身（S3 SDK） | plaud-document 自身（S3 SDK） | 不生成，依赖上游（plaud-api）提供的 mapping | **ask 服务自身（S3 SDK）** |
| **持久化形式** | markdown 中的 S3 key | markdown 中的 S3 key | N/A | **短链接 `ask.plaud.ai/image/{id}`** |
| **URL 有效期** | 15 分钟（900 秒） | 120 分钟 | 取决于上游，通常 15 分钟 | **presigned URL 默认 1 小时；Redis 缓存 30 分钟** |
| **图片下载** | 不下载，返回 URL 给前端 | 下载到本地临时目录 | send_email 不下载；draft_email 由 LLM 服务端拉取 | **不下载，presigned URL 直接返回前端** |
| **图片预处理** | 无 | EXIF 旋转 + 缩放 | 无 | **JPEG 压缩到 200KB 以内** |
| **S3 桶配置来源** | `plaud_content_storage_service` | `config.S3.ContentStorageBucket`（配置文件） | 不直接访问 S3 | **`Config.get_ask_s3_bucket_name()`（ask 专用桶）** |

---

## 7. 数据流全景

```
                        plaud-api                                     plaud-ask
                            │                                             │
        ┌───────────────────┼────────────────────┐                        │
        ▼                   ▼                    ▼                        ▼
  GET /file/detail    POST /document/export    plaud-tools (MCP)    image_generator skill
        │                   │                    │                        │
        ▼                   ▼                    ▼                        ▼
  download_link.py     原样透传 content      summary_to_markdown()   LLM 生成 data URI
  抽取 S3 key             │                 用 mapping 替换 S3 key       │
  生成 presigned URL      ▼                    │                   base64 解码 + 压缩
  返回 mapping       plaud-document             ▼                   上传 S3（ask 专用桶）
        │            自行生成 presigned URL   draft_email / send_email    │
        ▼            下载图片到本地            图片以 URL 形式保留    生成 presigned URL
      前端            预处理 + 嵌入 PDF         │                   缓存到 Redis
  用 mapping                                    ▼                        │
  替换展示                                   mistune → HTML <img>        ▼
                                             或 LLM 多模态输入      SSE media 事件推送
                                                                   短链接存入对话历史
                                                                        │
                                                                        ▼
                                                                   加载历史时：
                                                                   短链接 → 还原 S3 key
                                                                   → Redis/S3 签名
                                                                   → 替换为 presigned URL
```
