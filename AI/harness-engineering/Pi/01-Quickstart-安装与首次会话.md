# 01 - Quickstart：安装与首次会话

> 来源：[https://pi.dev/docs/latest/quickstart](https://pi.dev/docs/latest/quickstart)

## 1. 安装

Pi 通过 npm 全局安装：

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` 关闭依赖的 lifecycle scripts——Pi 在标准 npm 安装下不需要它们。

Linux / macOS 可选脚本安装：

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

### 卸载

按安装时用的包管理器选对应命令：

```bash
npm uninstall -g @earendil-works/pi-coding-agent      # npm 和 curl 安装都走这条
pnpm remove -g @earendil-works/pi-coding-agent
yarn global remove @earendil-works/pi-coding-agent
bun uninstall -g @earendil-works/pi-coding-agent
```

卸载只删 CLI 本体，**不会动 `~/.pi/agent/` 下的 settings、credentials、sessions 和已安装的 pi packages**。重装后这些都还在。

## 2. 启动

```bash
cd /path/to/project
pi
```

进入项目目录后跑 `pi` 即可。`pi` 把当前工作目录当作项目根。

## 3. 认证：两条路径

### 路径 A：subscription 登录（OAuth）

启动 Pi 后输入：

```text
/login
```

然后选 provider。内置支持的订阅类登录：

- **Claude Pro/Max**（Anthropic）
- **ChatGPT Plus/Pro**（OpenAI Codex 通道）
- **GitHub Copilot**

token 存到 `~/.pi/agent/auth.json`。

### 路径 B：API key

启动前设环境变量：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

或者用 `/login` 选 API-key provider，把 key 存到 `~/.pi/agent/auth.json`。

完整 provider 列表、环境变量名和云厂商配置见 [[03-Providers-认证与配置]]。

## 4. 第一次会话

输入需求回车，例如：

```text
Summarize this repository and tell me how to run its checks.
```

模型默认带 4 个工具：

|Tool|作用|
|---|---|
|`read`|读文件|
|`write`|创建或覆写文件|
|`edit`|给文件打 patch|
|`bash`|执行 shell 命令|

另有 3 个只读辅助：`grep`、`find`、`ls`，通过 tool 选项开启。

### 安全提示

Pi 直接在工作目录里写文件、跑命令——没有 plan mode 也没有 permission popup。官方建议**配合 git 或类似 checkpoint 工作流**，方便回滚。

## 5. 项目级指令文件（AGENTS.md）

在项目根放一个 `AGENTS.md` 引导 Pi 的行为：

```markdown
# Project Instructions

- Run `npm run check` after code changes.
- Do not run production migrations locally.
- Keep responses concise.
```

Pi 启动时按以下顺序加载并拼接上下文文件（来自 `core/resource-loader.js` 的 `loadProjectContextFiles`）：

1. `~/.pi/agent/AGENTS.md`（全局）
2. 从最远祖先到最近父目录的 `AGENTS.md` 或 `CLAUDE.md`
3. 当前目录（cwd）的 `AGENTS.md` 或 `CLAUDE.md`

每个目录只取候选名中的第一个命中：`AGENTS.md` → `AGENTS.MD` → `CLAUDE.md` → `CLAUDE.MD`。同一路径已加载过的不会重复。

加载后按上述顺序顺次拼接到 system prompt 的 `<project_context>` 块内：

```xml
<project_context>
<project_instructions path="~/.pi/agent/AGENTS.md">...</project_instructions>
<project_instructions path="/Users/me/projects/AGENTS.md">...</project_instructions>
<project_instructions path="/Users/me/projects/app/AGENTS.md">...</project_instructions>
</project_context>
```

**优先级**：源码里没有显式的覆盖机制，只是顺序拼接。但 cwd 的文件出现在最后，凭 LLM 对末尾内容的 recency 偏好 + 写作约定上"越具体越靠后"，事实上 cwd 的指令优先级最高。

修改后用 `/reload` 重新加载，或重启 Pi。

## 6. 常用操作速查

> **术语**：
>
> - **TUI**（Terminal User Interface）：在终端里输入 `pi` 回车后进入的**全屏交互界面**，包含输入框、消息历史、状态栏等组件。与之相对的是 `pi -p "..."` 这种一次性跑完就退出、不进 TUI 的 CLI 模式。
> - **editor**：TUI 里那个**输入框**（不是 VS Code/Vim 等外部编辑器）。所有相关快捷键都挂在 `tui.editor.*` 命名空间下（光标移动、删除字符、yank 等）。
> - **session**：当前这次 Pi 对话本身，由 Pi 持续保存。后文 `pi -c` 接续的就是它。

### 引用文件

在 editor 里按 `@` 触发模糊文件搜索，或在 CLI 用 `@` 前缀：

```bash
pi @README.md "Summarize this"
pi @src/app.ts @src/app.test.ts "Review these together"
```

图片可以 `Ctrl+V` 粘贴（Windows 上是 `Alt+V`），也可拖入支持的终端。

### 在 session 里跑 shell

在 editor 里以 `!` 开头输入的内容不会发给 LLM，而是由 Pi 直接交给 shell 执行：

```text
!npm run lint     # 跑命令，输出加入 context（模型能看到）
!!npm run lint    # 跑命令，输出不加入 context（静默执行）
```

用途是让模型基于一段实时命令输出回答，不必切到别的终端跑完再粘贴。`!!` 适合那种只是想顺手跑一下、又不想污染 context 的命令。

### 切换 model

|操作|快捷键|
|---|---|
|选 model|`/model` 或 `Ctrl+L`|
|调 thinking 等级|`Shift+Tab` 循环|
|在 scoped models 间循环|`Ctrl+P` / `Shift+Ctrl+P`|

### 恢复 session

Session 自动保存。恢复方式分两类，**取决于 Pi 此刻是否已经在跑**：

**(1) Pi 还没启动**——在外部 shell（zsh/bash）里用启动 flag，`pi` 是可执行文件名：

```bash
pi -c                  # 接上最近一次 session
pi -r                  # 浏览历史 session
pi --session <path|id> # 打开指定 session
```

**(2) Pi 已经在跑**——在 editor 里用斜杠命令，不带 `pi`，在当前进程内切换：

```text
/resume     # 列出并恢复历史 session
/new        # 开新 session
/tree       # 看 session 树
/fork       # 从当前位置分叉
/clone      # 复制 session
```

机制见 [[06-Sessions-会话树]]。

### 非交互（one-shot）模式

```bash
pi -p "Summarize this codebase"
cat README.md | pi -p "Summarize this text"
pi -p @screenshot.png "What's in this image?"
```

输出格式可选：`--mode json`（JSON event 流）、`--mode rpc`（process 集成）。

完整 CLI flag 见 [[02-交互模式与-CLI-参考#CLI 完整参考]]。
