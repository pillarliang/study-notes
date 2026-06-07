# 02 - 交互模式与 CLI 参考

> 来源：[https://pi.dev/docs/latest/usage](https://pi.dev/docs/latest/usage)

## 1. 交互界面的四个区域

启动 `pi` 后的 TUI 自上而下：


| 区域                 | 内容                                                       |
| ------------------ | -------------------------------------------------------- |
| **Startup header** | 快捷键提示、已加载的 context 文件、prompt templates、skills、extensions |
| **Messages**       | 用户消息、assistant 响应、tool 调用与结果、通知、错误、extension 自定义 UI      |
| **Editor**         | 输入区；**边框颜色提示当前 thinking 等级**                             |
| **Footer**         | 工作目录、session 名、token / cache 用量、cost、context 占用、当前 model |


Editor 区域可被临时替换成内置 UI（如 `/settings` 调出的设置面板）或 extension 提供的自定义 UI。

## 2. Editor 功能


| 功能            | 操作                                              |
| ------------- | ----------------------------------------------- |
| 引用文件          | `@` 触发模糊搜索                                      |
| 补全路径          | `Tab`                                           |
| 多行输入          | `Shift+Enter`（Windows Terminal 上是 `Ctrl+Enter`） |
| 粘贴图片          | `Ctrl+V`（Windows 用 `Alt+V`）或拖入终端                |
| 跑 shell 命令    | `!command` 运行并把输出发给模型                           |
| 跑 shell 但隐藏输出 | `!!command` 只执行、不污染 context                     |
| 调用外部编辑器       | `Ctrl+G` 打开 `$VISUAL` 或 `$EDITOR`               |


## 3. Slash 命令

输入 `/` 触发命令补全。Extensions 可以注册自定义命令；skills 以 `/skill:name` 出现；prompt templates 以 `/templatename` 展开。

### 内置命令


| 命令                  | 说明                                                       |
| ------------------- | -------------------------------------------------------- |
| `/login`, `/logout` | 管理 OAuth 或 API key 凭据                                    |
| `/model`            | 切 model                                                  |
| `/scoped-models`    | 配置 `Ctrl+P` 循环里包含哪些 model                                |
| `/settings`         | 调 thinking level、theme、message delivery、transport        |
| `/resume`           | 从历史 session 里挑一个                                         |
| `/new`              | 开新 session                                               |
| `/name <name>`      | 给当前 session 起可读名                                         |
| `/session`          | 显示 session 文件、ID、消息数、token、cost                          |
| `/tree`             | 跳到 session 树里任一节点继续                                      |
| `/fork`             | 从某条 user message 起 fork 出新 session                       |
| `/clone`            | 把当前活动分支复制成新 session                                      |
| `/compact [prompt]` | 手动压缩 context，可附自定义指令                                     |
| `/copy`             | 复制最后一条 assistant 消息到剪贴板                                  |
| `/export [file]`    | session 导出 HTML                                          |
| `/share`            | 上传为私有 GitHub gist，得到可分享的 HTML 链接                         |
| `/reload`           | 重新加载 keybindings、extensions、skills、prompts、context files |
| `/hotkeys`          | 显示所有快捷键                                                  |
| `/changelog`        | 显示版本历史                                                   |
| `/quit`             | 退出                                                       |


会话树相关命令的对比见 [[06-Sessions-会话树#tree-vs-fork-vs-clone]]。

## 4. Message Queue：边跑边排消息

agent 还在工作时也能继续输入：


| 按键          | 行为                                                 |
| ----------- | -------------------------------------------------- |
| `Enter`     | 排一条 **steering message**——当前 turn 的 tool 调用全部完成后送达 |
| `Alt+Enter` | 排一条 **follow-up**——所有工作彻底结束后送达                     |
| `Escape`    | **中断**当前 turn，并把排队消息还原到 editor                     |
| `Alt+Up`    | **不中断**当前 turn，把队尾的排队消息拿回 editor 让你改或删             |


`Alt+Up` 的典型场景：刚 `Enter` 排了条消息又反悔——想改措辞或干脆撤回。按 `Alt+Up` 把那条消息从队列弹回输入框，agent 继续干它的活，编辑完再重新 `Enter` 入队即可。`Escape` 则是"立刻打断重来"。

**Windows Terminal 注意**：`Alt+Enter` 默认触发全屏，需先在终端里改键，详见 [[20-平台与终端配置#Windows-Terminal]]。

配置：`settings.json` 的 `steeringMode` 和 `followUpMode` 控制送达粒度（`all` 一次送全部 / `one-at-a-time` 一次送一条），见 [[04-Settings-配置全集#message-delivery]]。

## 5. Context Files 与 System Prompt

Pi 启动时按以下顺序加载 `AGENTS.md` 或 `CLAUDE.md`：

1. `~/.pi/agent/AGENTS.md`（全局）
2. 当前目录起一路向上的父目录
3. 当前目录

用 `--no-context-files` / `-nc` 整体禁用。

### 替换或追加系统 prompt


| 文件                                                      | 行为                     |
| ------------------------------------------------------- | ---------------------- |
| `.pi/SYSTEM.md` 或 `~/.pi/agent/SYSTEM.md`               | **替换**默认 system prompt |
| `.pi/APPEND_SYSTEM.md` 或 `~/.pi/agent/APPEND_SYSTEM.md` | **追加**到默认 prompt       |


CLI 等价物：`--system-prompt <text>` 和 `--append-system-prompt <text>`。注意：替换默认 prompt 后，context files 和 skills 仍会被附加。

## 6. 导出与分享 session

- `/export [file]` 写成 HTML
- `/share` 上传为私有 GitHub gist，返回可分享的 HTML 链接
- 想发布到 Hugging Face dataset 做研究，参考 `badlogic/pi-share-hf`

## 7. CLI 完整参考

总语法：

```text
pi [options] [@files...] [messages...]
```

### Package 管理命令

> 这些命令管理 **pi packages**（extension/skill 等资源包），不是 pi CLI 本体。


| 命令                                                       | 说明                              |
| -------------------------------------------------------- | ------------------------------- |
| `pi install <source> [-l]`                               | 安装 package，`-l` 是 project-local |
| `pi remove <source> [-l]` / `pi uninstall <source> [-l]` | 卸载 package                      |
| `pi update [source                                       | self                            |
| `pi update --extensions`                                 | 只更新 packages                    |
| `pi update --self`                                       | 只更新 pi                          |
| `pi update --extension <src>`                            | 更新单个 package                    |
| `pi list`                                                | 列出已装 packages                   |
| `pi config`                                              | 启用/禁用 package 提供的资源             |


### Modes


| Flag                  | 说明                                      |
| --------------------- | --------------------------------------- |
| 默认                    | 交互模式                                    |
| `-p`, `--print`       | 跑完打印响应后退出                               |
| `--mode json`         | 所有事件以 JSON line 输出，详见 [[18-JSON-事件流]]   |
| `--mode rpc`          | stdin/stdout 上的 RPC 模式，详见 [[17-RPC-模式]] |
| `--export <in> [out]` | 把指定 session 导成 HTML                     |


print 模式会把 stdin 拼到初始 prompt：

```bash
cat README.md | pi -p "Summarize this text"
```

### Model 选项


| Option                   | 说明                                                      |
| ------------------------ | ------------------------------------------------------- |
| `--provider <name>`      | 例如 `anthropic` / `openai` / `google`                    |
| `--model <pattern>`      | model pattern 或 ID；支持 `provider/id` 和 `:<thinking>`     |
| `--api-key <key>`        | API key，覆盖环境变量                                          |
| `--thinking <level>`     | `off` / `minimal` / `low` / `medium` / `high` / `xhigh` |
| `--models <patterns>`    | 逗号分隔，用于 `Ctrl+P` 循环                                     |
| `--list-models [search]` | 列出可用 model                                              |


### Session 选项


| Option                | 说明               |
| --------------------- | ---------------- |
| `-c`, `--continue`    | 续接最近一次 session   |
| `-r`, `--resume`      | 浏览并选 session     |
| `--session <path      | id>`             |
| `--fork <path         | id>`             |
| `--session-dir <dir>` | 自定义 session 存储目录 |
| `--no-session`        | 临时模式，不存盘         |


### Tool 选项


| Option                        | 说明                                  |
| ----------------------------- | ----------------------------------- |
| `--tools <list>`, `-t <list>` | 白名单 built-in / extension / 自定义 tool |
| `--no-builtin-tools`, `-nbt`  | 关掉内置 tool，保留 extension/自定义 tool     |
| `--no-tools`, `-nt`           | 关掉所有 tool                           |


Built-in tool：`read`、`bash`、`edit`、`write`、`grep`、`find`、`ls`。

只读模式范例：

```bash
pi --tools read,grep,find,ls -p "Review the code"
```

### Resource 选项


| Option                       | 说明                                   |
| ---------------------------- | ------------------------------------ |
| `-e`, `--extension <source>` | 加载 extension，可重复，支持 path / npm / git |
| `--no-extensions`            | 禁用 extension 自动发现                    |
| `--skill <path>`             | 加载 skill，可重复                         |
| `--no-skills`                | 禁用 skill 自动发现                        |
| `--prompt-template <path>`   | 加载 prompt template，可重复               |
| `--no-prompt-templates`      | 禁用 prompt template 自动发现              |
| `--theme <path>`             | 加载 theme，可重复                         |
| `--no-themes`                | 禁用 theme 自动发现                        |
| `--no-context-files`, `-nc`  | 禁用 AGENTS.md / CLAUDE.md 发现          |


组合 `--no-*` + 显式加载，可以做到"只用指定的这个"：

```bash
pi --no-extensions -e ./my-extension.ts
```

### 其它选项


| Option                          | 说明                                       |
| ------------------------------- | ---------------------------------------- |
| `--system-prompt <text>`        | 替换默认 prompt（context files 和 skills 仍会附加） |
| `--append-system-prompt <text>` | 追加到 system prompt                        |
| `--verbose`                     | 强制 verbose 启动                            |
| `-h`, `--help`                  | 帮助                                       |
| `-v`, `--version`               | 版本                                       |


### 文件参数

`@` 前缀的文件附加到 message：

```bash
pi @prompt.md "Answer this"
pi -p @screenshot.png "What's in this image?"
pi @code.ts @test.ts "Review these files"
```

### 常用例子

```bash
# 交互模式带初始 prompt
pi "List all .ts files in src/"

# 非交互
pi -p "Summarize this codebase"

# 用别的 provider
pi --provider openai --model gpt-4o "Help me refactor"

# 用 provider/id 简写
pi --model openai/gpt-4o "Help me refactor"

# thinking level 简写
pi --model sonnet:high "Solve this complex problem"

# 限定 Ctrl+P 循环范围
pi --models "claude-*,gpt-4o"

# 只读模式
pi --tools read,grep,find,ls -p "Review the code"
```

## 8. 环境变量


| 变量                            | 说明                                                              |
| ----------------------------- | --------------------------------------------------------------- |
| `PI_CODING_AGENT_DIR`         | 覆盖配置目录，默认 `~/.pi/agent`                                         |
| `PI_CODING_AGENT_SESSION_DIR` | 覆盖 session 存储目录；被 `--session-dir` 进一步覆盖                         |
| `PI_PACKAGE_DIR`              | 覆盖 package 目录（Nix/Guix store 场景）                                |
| `PI_OFFLINE`                  | 关闭启动时的所有网络操作（含 update 检查和 telemetry）                            |
| `PI_SKIP_VERSION_CHECK`       | 跳过版本检查                                                          |
| `PI_TELEMETRY`                | 覆盖 install/update telemetry：`1`/`true`/`yes` 或 `0`/`false`/`no` |
| `PI_CACHE_RETENTION`          | 设为 `long` 在支持的 provider 上启用长 prompt cache                       |
| `VISUAL`, `EDITOR`            | `Ctrl+G` 调用的外部编辑器                                               |


## 9. 设计原则

Pi 把核心保持最小，工作流相关能力推给 extensions / skills / prompt templates / packages。

**刻意不内置**的能力清单：

- MCP（model context protocol 客户端）
- sub-agents
- permission popups
- plan mode
- to-dos
- background bash

这些要么走 extension/package 自己实现，要么走外部工具（容器、tmux）解决。