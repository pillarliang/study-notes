# 06 - Sessions 会话树

> 来源：https://pi.dev/docs/latest/sessions

## 1. 原理：session 是树而非线

大多数 coding agent 把 session 模型成线性时间轴：消息按时间顺序追加，想"换一种走法"就得开新 session。

Pi 的设计不同：**每个 session 是一棵树**。每个条目有 `id` 和 `parentId`，当前位置是 "active leaf"。想试不同方向，不必新建文件——`/tree` 跳回任意节点继续。

举例：

```text
├─ user: "Hello, can you help..."
│  └─ assistant: "Of course! I can..."
│     ├─ user: "Let's try approach A..."
│     │  └─ assistant: "For approach A..."
│     │     └─ user: "That worked..."  ← active
│     └─ user: "Actually, approach B..."
│        └─ assistant: "For approach B..."
```

树结构带来的能力：在**同一个文件**里维护多条尝试路径，能互相对比、互相切回；避免了"开新 session → 上下文丢失"的困境。

## 2. 存储

- 自动存到 `~/.pi/agent/sessions/`，按工作目录分组
- 单文件 JSONL，承载树结构；完整字段定义见 [[15-Session-文件格式]]
- 存储位置可改：`--session-dir` > `PI_CODING_AGENT_SESSION_DIR` > settings 的 `sessionDir`

## 3. 启动 flag

```bash
pi -c                  # 续接最近一次 session
pi -r                  # 浏览并选 session
pi --no-session        # 临时模式，不存盘
pi --session <path|id> # 指定 session 文件或部分 UUID
pi --fork <path|id>    # fork 指定 session 进新文件
```

## 4. Slash 命令

| 命令 | 用途 |
|------|------|
| `/resume` | 历史 session 选择器 |
| `/new` | 开新 session |
| `/name <name>` | 给当前 session 起可读名 |
| `/session` | 显示文件路径、ID、消息数、token、cost |
| `/tree` | 在树里跳转、查看分支 |
| `/fork` | 从某条 user message 起 fork 出新 session |
| `/clone` | 把当前活动分支复制到新 session |
| `/compact [prompt]` | 压缩历史，见 [[07-Compaction-上下文压缩]] |
| `/export [file]` | 导出 HTML |
| `/share` | 上传私有 GitHub gist |

## 5. `/resume` 选择器

`/resume`（同 `pi -r`）支持：

- 输入即搜
- `Ctrl+P` 切换路径显示
- `Ctrl+S` 切换排序模式
- `Ctrl+N` 只看命名 session
- `Ctrl+R` 重命名
- `Ctrl+D` 删除（带确认）

如果系统装了 `trash` CLI，Pi 用它而不是直接 unlink，给一层后悔余地。

## 6. 命名 session

```text
/name Refactor auth module
```

命名后在选择器里更显眼。

## 7. 树导航

### 按键

| Key | 行为 |
|-----|------|
| ↑/↓ | 在可见条目间移动 |
| ←/→ | 翻页 |
| `Ctrl+←` / `Ctrl+→`（或 `Alt+←/→`） | 折叠/展开，或在分支段之间跳 |
| `Shift+L` | 给当前节点加/去 label |
| `Shift+T` | 切换 label 时间戳显示 |
| `Enter` | 确认选择 |
| `Escape` / `Ctrl+C` | 取消 |
| `Ctrl+O` | 循环过滤模式 |

### 过滤模式

`default` / `no-tools` / `user-only` / `labeled-only` / `all`。起始模式在 settings 里用 `treeFilterMode` 控制（见 [[04-Settings-配置全集#UI-与显示]]）。

### 选中节点的行为差异

选 **user 或 custom 消息**：

1. active leaf 移到这条消息的 parent
2. 消息文本载入 editor
3. 可编辑后重新提交——**产生新分支**

选 **assistant / tool / compaction 等非 user 节点**：

1. leaf 移到该节点
2. editor 留空
3. 从这里继续

选 **根 user 消息**：清掉 leaf，把原始 prompt 还原到 editor。

> 直觉：选 user 消息 = "在这一步改提问重来"；选 assistant 节点 = "在这一步继续往下"。

## 8. `/tree` vs `/fork` vs `/clone`

| 维度 | `/tree` | `/fork` | `/clone` |
|------|---------|---------|----------|
| 输出 | **同一个** session 文件 | 新 session 文件 | 新 session 文件 |
| 视图 | 完整树 | user 消息选择器 | 当前活动分支 |
| 用途 | 在原文件里探索多条路 | 从某个早期 prompt 重起一个独立 session | 把当前进度复制成另一个文件再继续 |
| 是否生成分支摘要 | 可选 | 否 | 否 |

**选择原则**：

- 想保留对比，用 `/tree`
- 想从某点彻底另起炉灶，用 `/fork`
- 想保留当前状态再实验，用 `/clone`

## 9. 分支摘要

`/tree` 把 leaf 从一条分支跳到另一条时，Pi 可以**给即将离开的分支生成一个摘要**，把它附加到新位置上——保留关键上下文而不必把整条放弃路径都 replay 进 context。

会被提示选择：

1. 不生成摘要
2. 用默认 prompt 生成摘要
3. 用自定义指令生成摘要

可在 settings 里 `branchSummary.skipPrompt = true` 跳过这个选择，详见 [[04-Settings-配置全集#branch-summary]]。

摘要内容的固定段落结构（Goal / Constraints / Progress / Key Decisions / Next Steps / Critical Context）见 [[07-Compaction-上下文压缩#摘要格式]]。

## 10. 会话文件格式

JSONL，每行一个条目（type / id / parentId / timestamp + 各 type 特有字段），承载消息、model 切换、thinking 切换、compaction、branch_summary、label、session_info 等条目类型。完整 schema、各字段含义、以及 `SessionManager` API 见 [[15-Session-文件格式]]。
