# 07 - Compaction 上下文压缩

> 来源：https://pi.dev/docs/latest/compaction

## 1. 原理：为什么必须压缩

LLM 的 context window 有限。长会话里旧消息无限累积，最终撞到上限——再继续就要截断或报错。

Pi 提供两个互补的压缩机制：

| 机制                       | 触发时机                               | 用途                  |
| ------------------------ | ---------------------------------- | ------------------- |
| **Compaction**           | context 用量超过阈值（自动）或 `/compact`（手动） | 沿时间轴压缩旧消息，保留近端原文    |
| **Branch Summarization** | `/tree` 跨分支跳转时                     | 把即将抛弃的分支总结成一段，注入新位置 |

两者都生成**结构化 markdown 摘要**，并**累计追踪文件读写历史**。

## 2. Compaction

### 2.1 触发条件

**自动**：当前 context token 数超过 `contextWindow - reserveTokens`（默认 reserve = 16384）触发。
**手动**：`/compact [instructions]`，可选 instructions 用来引导摘要重点（如 `/compact 聚焦在 auth 重构的决策上`）。

### 2.2 工作流程（5 步）

1. **找切分点**：从最新消息倒序往回累计 token，直到达到 `keepRecentTokens`（默认 20000），把切分点定在这里
2. **摘取消息**：从上一次的"保留边界"（或 session 起点）到这个切分点之间的所有消息
3. **生成摘要**：调 LLM 生成摘要；如果之前有摘要，把它作为迭代上下文一起传入
4. **追加 `CompactionEntry`**：把摘要和 `firstKeptEntryId`（保留消息的起点 ID）写进 session
5. **重载**：session 重新加载，view 变成"摘要 + `firstKeptEntryId` 之后的原文消息"

**重复 compaction 时**：被压缩区间起点是上次 compaction 的保留边界——也就是说，**上次幸存下来的消息这次不会再被压缩**。重建后 token 数会重算，再写文件。

### 2.3 切分点规则

**有效切分点**：user 消息、assistant 消息、`BashExecution` 消息、custom 消息。
**绝不切**在 tool result 处——它必须和发起它的 tool call 留在一起。

### 2.4 Split turn：单 turn 过大怎么办

一个 turn 从 user 消息开始，包括后续所有 assistant 和 tool 消息，到下一条 user 消息前结束。

正常情况下切分点落在 turn 边界。但**如果单个 turn 自身超过 `keepRecentTokens`**，切分点会落在 turn 内部某条 assistant 消息上。Pi 此时生成两段摘要（history 段 + turn prefix 段）然后合并。

### 2.5 `CompactionEntry` 关键字段

| 字段 | 含义 |
|------|------|
| `summary` | markdown 文本 |
| `firstKeptEntryId` | 保留消息的起点 ID |
| `tokensBefore` | 压缩前的 token 数（统计用） |
| `details` | 默认追踪 `readFiles` 和 `modifiedFiles` |
| `fromHook` | 是否来自 extension hook |

完整结构见 [[15-Session-文件格式#CompactionEntry]]。

## 3. Branch Summarization

### 3.1 触发

`/tree` 把 active leaf 从旧分支移到新分支时，Pi 询问是否给旧分支生成摘要。

### 3.2 流程

1. 找旧位置和新位置的**最深公共祖先**（deepest common ancestor）
2. 从旧 leaf 反向走到公共祖先，收集这段路径上的条目
3. 按 token 预算包含消息（**从最新开始**）
4. 生成摘要，在新位置追加一个 `BranchSummaryEntry`

> 最深公共祖先是树论的概念：找到旧路径和新路径"分岔之前"的那个节点。从这里到旧 leaf 的整条路径就是即将"被抛弃"的内容——摘要的就是这段。

### 3.3 累计文件追踪

两个机制都从被摘要消息的 tool call 里抽取文件操作，**同时合并上一段摘要 `details` 里的记录**。这意味着即使经过多层嵌套摘要，完整的 read / modified 历史不会丢。

## 4. 摘要格式

两种机制产出同一种结构化 markdown，固定段落：

- **Goal**
- **Constraints & Preferences**
- **Progress**（细分 Done / In Progress / Blocked）
- **Key Decisions**
- **Next Steps**
- **Critical Context**
- 配套 `<read-files>` 和 `<modified-files>` 块

这套段落是为了让"下次回顾"时一眼能找到目标、约束、进展和未尽事项。

## 5. 消息序列化

送给摘要 LLM 的消息走 `serializeConversation()`，标签化成行：

```text
[User]: ...
[Assistant thinking]: ...
[Assistant]: ...
[Assistant tool calls]: read(path="foo.ts"); edit(path="bar.ts", ...)
[Tool result]: ...
```

**Tool result 会被截断到 2000 字符**，并标注被截掉了多少——避免摘要请求自己又把 token 撑爆。

## 6. Extension Hooks（进阶）

两个 event 让 extension 干预默认行为：

| Event | 时机 | 能做什么 |
|-------|------|---------|
| `session_before_compact` | 自动或手动 compaction 之前 | 取消，或提供自定义摘要 |
| `session_before_tree` | `/tree` 导航之前（**总会触发**，与是否要摘要无关） | 取消导航，或在 `userWantsSummary` 时提供自定义摘要 |

### 自定义压缩示例

```ts
import { convertToLlm, serializeConversation } from "@earendil-works/pi-coding-agent";

pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;
  const conversationText = serializeConversation(
    convertToLlm(preparation.messagesToSummarize)
  );
  const summary = await myModel.summarize(conversationText);
  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
    }
  };
});
```

取消的话：`return { cancel: true };`。

**典型用途**：用更便宜的本地 model 跑摘要、按自定义规则保留特定消息、给摘要插入项目特有的提示。

## 7. 配置

放在 `~/.pi/agent/settings.json` 或 `<project>/.pi/settings.json`：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| Setting | 默认 | 说明 |
|---------|------|------|
| `enabled` | `true` | 关闭自动压缩 |
| `reserveTokens` | `16384` | 给 LLM 响应预留的 token |
| `keepRecentTokens` | `20000` | 近端不被压缩的 token 量 |

> 把 `enabled: false` 关掉**只**禁用自动压缩；`/compact` 仍可手动调用。

分支摘要相关配置（`branchSummary.reserveTokens` / `branchSummary.skipPrompt`）见 [[04-Settings-配置全集#branch-summary]]。

## 8. 相关命令

- `/compact [instructions]`：手动触发压缩；instructions 定向摘要焦点
- `/tree`：触发分支摘要流程，机制见 [[06-Sessions-会话树]]

## 9. 心智模型小结

- **Compaction** 是**时间轴上的压缩**：旧消息变摘要，近端原样保留
- **Branch summarization** 是**树形结构上的压缩**：被放弃的整条分支变成一段
- 二者都**累计追踪文件读写**，多次嵌套也不丢历史
- 摘要本身有**固定段落结构**，方便后续 review 和 LLM 消费
- 关键参数：`reserveTokens`（响应预留）和 `keepRecentTokens`（不动近端）
