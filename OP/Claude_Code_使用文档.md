# Claude Code 使用文档

更新时间：2026-03-09  
本机校验版本：`claude 2.1.71 (Claude Code)`

## 1. 安装

### 1.1 前置条件

- Node.js ≥ 18

macOS 安装 Node.js：

```bash
# 方式一：通过 Homebrew（推荐）
brew install node

# 方式二：通过 nvm（需要多版本管理时）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install --lts
```

### 1.2 通过 npm 全局安装

```bash
npm install -g @anthropic-ai/claude-code
```

### 1.3 验证安装

```bash
claude --version
```

### 1.4 首次登录

```bash
claude auth login
```

按提示完成 Anthropic 账号认证即可。

---

## 2. 快速开始

### 2.1 启动交互会话

```bash
claude
```

在当前目录启动一个交互式会话。

### 2.2 单次输出（非交互）

```bash
claude -p "请总结 README.md 的核心内容"
```

说明：

- `-p` / `--print`：输出后立即退出
- 适合脚本、流水线、批处理场景

### 2.3 带初始提示启动

```bash
claude "帮我检查这个仓库中可能的空指针风险"
```

---

## 3. 认证与状态

```bash
claude auth login
claude auth status
claude auth logout
```

---

## 4. 常用命令速查

```bash
# 查看版本和帮助
claude --version
claude --help

# 继续当前目录最近会话
claude -c

# 按会话 ID 恢复
claude -r <session_id>

# 指定模型
claude --model sonnet
claude --model opus

# 调整推理力度
claude --effort low
claude --effort medium
claude --effort high

# 输出 JSON（配合 -p）
claude -p --output-format json "给我一个发布说明草稿"

# 查看可用 agent
claude agents
```

---

## 5. 配置文件层级

Claude Code 常见配置分三层（后者通常覆盖前者）：

1. 用户全局配置：`~/.claude.json`
2. 项目共享配置：`<repo>/.claude/settings.json`
3. 项目本地配置：`<repo>/.claude/settings.local.json`

---

## 6. 添加 MCP 服务（步骤）

### 6.1 添加 HTTP MCP（示例：Feishu）

```bash
claude mcp add --transport http -s user feishu-mcp https://mcp.feishu.cn/mcp/<your_id>
```

### 6.2 添加本地 stdio MCP

```bash
claude mcp add -s user my-mcp -- npx -y my-mcp-server
```

### 6.3 查看与校验

```bash
claude mcp list
claude mcp get feishu-mcp
```

### 6.4 删除 MCP

```bash
claude mcp remove feishu-mcp -s user
```

### 6.5 作用域说明

1. `local`：当前目录（默认）
2. `project`：当前项目
3. `user`：全局可用

---

## 7. 添加 Skills（Plugin 方式）

说明：在 Claude Code 里，Skills 通常通过 Plugin/Marketplace 安装。

/Users/liangzhu/.claude/skills/xxx

### 7.1 查看可安装列表

```bash
claude plugin list --available --json
```

### 7.2 安装 Skills/Plugin

```bash
claude plugin install <plugin-id> -s user
# 或指定市场
claude plugin install <plugin-id>@<marketplace> -s project
```

### 7.3 管理已安装项

```bash
claude plugin list --json
claude plugin enable <plugin-id>@<marketplace>
claude plugin disable <plugin-id>@<marketplace>
claude plugin uninstall <plugin-id>@<marketplace>
```

### 7.4 添加第三方 Marketplace（可选）

```bash
claude plugin marketplace add <owner>/<repo> --scope user
claude plugin marketplace list --json
```

### 7.5 安装作用域说明

1. `user`：用户全局
2. `project`：项目共享
3. `local`：项目本地（通常不提交）

---

## 8. 权限与安全

常见权限模式（`--permission-mode`）：

- `default`
- `acceptEdits`
- `auto`
- `dontAsk`
- `plan`
- `bypassPermissions`

高风险选项（谨慎使用）：

- `--dangerously-skip-permissions`
- `--allow-dangerously-skip-permissions`

建议：

- 日常开发使用 `default` 或 `auto`
- 在可信沙箱中再考虑放宽权限
- 对外网访问、文件删除、批量改写保持人工复核

---

## 9. 一分钟排障

`claude: command not found`

- 检查安装与 PATH
- 重开终端再试

认证失败或会话异常

- `claude auth status`
- 必要时重新 `claude auth login`

MCP 连不上

- 检查配置文件路径和 JSON
- 检查网络连通性和鉴权
- 用 `--debug` 查看详细报错

---

## 10. 推荐起步命令（可直接复制）

```bash
# 1) 进入项目
cd /Users/liangzhu/Documents/work/resources/project-summary-evalation

# 2) 启动交互
claude

# 3) 常用：继续上次会话
claude -c

# 4) 常用：一次性输出
claude -p "请总结 project_52/evaluation_review_multi_scene.md 的关键改进点"
```

---

## 11. 备注

1. 安装或启用新的 MCP/Plugin 后，建议重开 `claude` 会话。
2. 你当前环境中的全局配置位于 `~/.claude.json`（含 `mcpServers`）。
3. Plugin 启用与 marketplace 信息常见于 `~/.claude/settings.json`。
