# 04 - Settings 配置全集

> 来源：https://pi.dev/docs/latest/settings

## 1. 配置文件位置与合并规则

| 位置 | 作用域 |
|------|--------|
| `~/.pi/agent/settings.json` | 全局（所有项目） |
| `.pi/settings.json` | 项目级（当前目录） |

**合并规则**：项目级覆盖全局；嵌套对象做 deep merge。可以直接编辑文件，也可以在交互里跑 `/settings` 改常用项。

合并示例：

```json
// 全局
{ "theme": "dark", "compaction": { "enabled": true, "reserveTokens": 16384 } }

// 项目
{ "compaction": { "reserveTokens": 8192 } }

// 合并后
{ "theme": "dark", "compaction": { "enabled": true, "reserveTokens": 8192 } }
```

## 2. Model 与 Thinking

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `defaultProvider` | string | — | 例如 `"anthropic"` / `"openai"` |
| `defaultModel` | string | — | model ID |
| `defaultThinkingLevel` | string | — | `off` / `minimal` / `low` / `medium` / `high` / `xhigh` |
| `hideThinkingBlock` | boolean | `false` | 隐藏 thinking 块 |
| `thinkingBudgets` | object | — | 每个 thinking level 的 token 预算 |

```json
{
  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 10240,
    "high": 32768
  }
}
```

## 3. UI 与显示

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `theme` | string | `"dark"` | `dark` / `light` / 自定义主题名 |
| `quietStartup` | boolean | `false` | 隐藏 startup header |
| `collapseChangelog` | boolean | `false` | 更新后压缩 changelog 显示 |
| `enableInstallTelemetry` | boolean | `true` | 匿名 install/update 版本 ping |
| `doubleEscapeAction` | string | `"tree"` | `tree` / `fork` / `none`，双击 esc 的行为 |
| `treeFilterMode` | string | `"default"` | `default` / `no-tools` / `user-only` / `labeled-only` / `all` |
| `editorPaddingX` | number | `0` | 输入框水平内边距（0–3） |
| `autocompleteMaxVisible` | number | `5` | 自动补全可见条数（3–20） |
| `showHardwareCursor` | boolean | `false` | 显示终端硬件光标 |

### Telemetry 与版本检查的关系

- `enableInstallTelemetry` 只控制向 `https://pi.dev/api/report-install` 的匿名 ping
- **版本检查** 是另一回事，仍会查 `https://pi.dev/api/latest-version`
- `PI_SKIP_VERSION_CHECK=1` 禁版本检查
- `--offline` 或 `PI_OFFLINE=1` 一刀切，禁所有启动时网络操作

## 4. Warnings

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `warnings.anthropicExtraUsage` | boolean | `true` | 用 Anthropic 订阅认证可能产生 extra usage 计费时是否警告 |

```json
{ "warnings": { "anthropicExtraUsage": false } }
```

Anthropic extra usage 的含义见 [[03-Providers-认证与配置#Claude-ProMax]]。

## 5. Compaction

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `compaction.enabled` | boolean | `true` | 启用自动压缩 |
| `compaction.reserveTokens` | number | `16384` | 给 LLM 响应预留的 token |
| `compaction.keepRecentTokens` | number | `20000` | 不压缩的近端 token 量 |

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

压缩触发逻辑见 [[07-Compaction-上下文压缩]]。

## 6. Branch Summary

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `branchSummary.reserveTokens` | number | `16384` | 分支摘要预留 token |
| `branchSummary.skipPrompt` | boolean | `false` | `/tree` 导航时跳过摘要选择提示 |

分支摘要机制见 [[06-Sessions-会话树#分支摘要]]。

## 7. Retry

agent 级 retry 处理瞬态错误；provider 级 retry 走 SDK 自己的策略。

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `retry.enabled` | boolean | `true` | 启用 agent 级 retry |
| `retry.maxRetries` | number | `3` | agent 级最大重试次数 |
| `retry.baseDelayMs` | number | `2000` | 指数退避基准 |
| `retry.provider.timeoutMs` | number | SDK 默认 | provider / SDK 请求超时（毫秒） |
| `retry.provider.maxRetries` | number | SDK 默认 | provider / SDK retry 次数 |
| `retry.provider.maxRetryDelayMs` | number | `60000` | 服务器请求的最长延迟容忍；超过即失败 |

`maxRetryDelayMs = 0` 解除上限。被 provider 要求的过长延迟会立刻失败并返回有信息的错误。

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000,
    "provider": {
      "timeoutMs": 3600000,
      "maxRetries": 0,
      "maxRetryDelayMs": 60000
    }
  }
}
```

## 8. Message Delivery

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `steeringMode` | string | `"one-at-a-time"` | `all` 或 `one-at-a-time` |
| `followUpMode` | string | `"one-at-a-time"` | 同上 |
| `transport` | string | `"sse"` | `sse` / `websocket` / `auto` |

`steeringMode` 控制 `Enter` 排队消息的送达粒度，`followUpMode` 控制 `Alt+Enter` 的。`one-at-a-time` 每次只送一条，`all` 一次性全送。消息队列机制见 [[02-交互模式与-CLI-参考#message-queue边跑边排消息]]。

## 9. Terminal 与图像

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `terminal.showImages` | boolean | `true` | 终端支持时显示图像 |
| `terminal.imageWidthCells` | number | `60` | 内联图像首选宽度（cell 数） |
| `terminal.clearOnShrink` | boolean | `false` | 内容缩小时清空空行 |
| `images.autoResize` | boolean | `true` | 图像最大缩放到 2000x2000 |
| `images.blockImages` | boolean | `false` | 阻止任何图像发给 LLM |

## 10. Shell

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `shellPath` | string | — | 自定义 shell 路径（如 Windows 上的 Cygwin） |
| `shellCommandPrefix` | string | — | 给每条 bash 命令加前缀 |
| `npmCommand` | string[] | — | npm package 操作用的 argv |

```json
{ "npmCommand": ["mise", "exec", "node@20", "--", "npm"] }
```

`shellCommandPrefix` 常见用法是让 shell 别名生效，见 [[20-平台与终端配置#shell-aliases]]。

**npm 安装路径**：

- user 级 package 装在 `~/.pi/agent/npm/`
- 项目级装在 `.pi/npm/`
- 设了 `npmCommand` 之后，git package 的依赖安装走纯 `install`

## 11. Sessions

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `sessionDir` | string | — | session 文件目录；支持绝对/相对路径和 `~` |

```json
{ "sessionDir": ".pi/sessions" }
```

**优先级**：`--session-dir` > `PI_CODING_AGENT_SESSION_DIR` > `sessionDir`。

## 12. Model 循环

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `enabledModels` | string[] | — | `Ctrl+P` 循环的 model pattern（同 `--models`） |

```json
{ "enabledModels": ["claude-*", "gpt-4o", "gemini-2*"] }
```

## 13. Markdown

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `markdown.codeBlockIndent` | string | `" "` | 代码块缩进字符 |

## 14. 资源加载（resources）

全局 settings 里的路径相对于 `~/.pi/agent`；项目级 settings 相对于 `.pi`。绝对路径和 `~` 都支持。

| 设置 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `packages` | array | `[]` | npm 或 git package，加载其中的资源 |
| `extensions` | string[] | `[]` | 本地 extension 路径或目录 |
| `skills` | string[] | `[]` | 本地 skill 路径或目录 |
| `prompts` | string[] | `[]` | 本地 prompt template 路径或目录 |
| `themes` | string[] | `[]` | 本地 theme 路径或目录 |
| `enableSkillCommands` | boolean | `true` | 把 skill 注册成 `/skill:name` 命令 |

**数组里支持 glob 和排除**：

- `!pattern` 排除
- `+path` 强制加入
- `-path` 强制排除

**packages 的两种写法**：

```json
// 字符串形式，加载整个 package 的全部资源
{ "packages": ["pi-skills", "@org/my-extension"] }
```

```json
// 对象形式，按资源类型过滤
{
  "packages": [
    {
      "source": "pi-skills",
      "skills": ["brave-search", "transcribe"],
      "extensions": []
    }
  ]
}
```

对象形式相当于"装了这个 package，但只用其中 `brave-search` 和 `transcribe` 两个 skill，extension 全部不要"。

## 15. 完整示例

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "theme": "dark",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  },
  "enabledModels": ["claude-*", "gpt-4o"],
  "warnings": { "anthropicExtraUsage": true },
  "packages": ["pi-skills"]
}
```
