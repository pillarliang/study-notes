---
title: Agent 平台 MCP 与 Skill 目录速查
created: 2026-08-13
updated: 2026-08-13
tags: [AI/Agent, MCP, Agent-Skills]
---

# Agent 平台 MCP 与 Skill 目录速查

> macOS；`~` 表示 `/Users/liangzhu`。路径可能随产品版本变化。

## 一、本质

Agent 相关文件只有三类：

1. **注册配置**：声明启用了哪些 MCP、Skill 或插件。
2. **真实文件**：MCP Server 程序或包含 `SKILL.md` 的 Skill 目录。
3. **运行数据**：缓存、索引、日志、会话和 OAuth 状态；通常不是能力来源。

清理时按这个顺序处理：

```text
注册配置 → 软链接 → 真实文件 → 运行数据
```

不要仅因为路径中含有 `mcp` 就删除它。

---

## 二、平台路径总表

| 平台 | MCP 注册配置 | Skill | 其他关键位置 |
|---|---|---|---|
| Claude Code | `~/.claude.json` 的 `mcpServers`；项目 `.mcp.json` | `~/.claude/skills/`；项目 `.claude/skills/` | `~/.claude/settings.json`、`commands/`、`agents/`、`plugins/`、`CLAUDE.md` |
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` | — | 与 Claude Code 不一定共用 MCP 配置 |
| Codex | `~/.codex/config.toml` 的 `[mcp_servers.*]` | `~/.codex/skills/`；项目还应检查 `.agents/skills/` | `~/.codex/AGENTS.md`、`rules/`、`plugins/` |
| Cursor | `~/.cursor/mcp.json`；项目 `.cursor/mcp.json` | `~/.cursor/skills/`、项目 `.cursor/skills/`，也可能读取 `.agents/skills/` | `~/.cursor/agents/`、`plugins/`、`extensions/`、项目 `.cursor/rules/` |
| Pi | 由设置、扩展或自定义工具决定 | `~/.pi/agent/skills/`、`~/.agents/skills/`、项目 `.pi/skills/` 与 `.agents/skills/` | `~/.pi/agent/settings.json`；还可由 `skills` 设置、包或 `--skill` 引入 |
| Gemini CLI | `~/.gemini/settings.json`；项目 `.gemini/settings.json` | `~/.gemini/skills/`；项目 `.gemini/skills/` | `~/.gemini/GEMINI.md` |
| Antigravity | `~/.gemini/antigravity/mcp_config.json` | — | `~/.gemini/antigravity/` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` | `~/.codeium/windsurf/skills/`；项目 `.windsurf/skills/` | 项目 `.windsurf/rules/`、`.windsurf/workflows/` |
| Continue | 在 `~/.continue/` 或项目 `.continue/` 内搜索 `mcp` | `~/.continue/skills/` | 配置文件名随版本变化 |
| VS Code / Copilot | `~/Library/Application Support/Code/User/mcp.json`；项目 `.vscode/mcp.json` | 项目 `.github/skills/` | `.github/agents/`、`.github/copilot-instructions.md` |
| Cline / Roo Code | VS Code 或 Cursor 的 `User/globalStorage/<扩展ID>/` | 由扩展版本决定 | 应按扩展 ID 搜索，不能只依赖固定文件名 |
| OpenCode | `~/.config/opencode/opencode.json`；项目 `opencode.json` | `~/.config/opencode/skills/`、项目 `.opencode/skills/` 或 `.agents/skills/` | `~/.config/opencode/` |

### IDE 扩展的 globalStorage 根目录

```text
~/Library/Application Support/Code/User/globalStorage/
~/Library/Application Support/Cursor/User/globalStorage/
```

---

## 三、本机的真实共享关系

本机大量 Skill 的**真实源目录**是：

```text
~/.agents/skills/
```

各平台通过软链接共享它们：

```text
~/.claude/skills/<name>          ─┐
~/.pi/agent/skills/<name>        ├──> ~/.agents/skills/<name>/
~/.codeium/windsurf/skills/<name>├──> ~/.agents/skills/<name>/
~/.continue/skills/<name>        ─┘
```

部分 Codex Skill 先链接到 Claude，因此可能形成两级链：

```text
~/.codex/skills/<name>
  → ~/.claude/skills/<name>
  → ~/.agents/skills/<name>/
```

但 `~/.claude/skills/` 和 `~/.codex/skills/` 中也有真实目录，所以不能把整个目录都当成链接。

### 删除含义

```bash
rm ~/.claude/skills/archimate
```

只删除 Claude 侧链接，不会删除最终 Skill。

```bash
rm -rf ~/.agents/skills/archimate
```

删除共享源文件，所有引用它的平台都会受影响。

---

## 四、如何确认一个 Skill 到底在哪里

```bash
# 查看是否为软链接
ls -ld ~/.claude/skills/archimate

# 查看链接直接指向哪里
readlink ~/.claude/skills/archimate

# 解析整条链接链，得到最终位置
realpath ~/.claude/skills/archimate
```

列出常见平台中的所有 Skill 软链接：

```bash
find \
  ~/.agents/skills ~/.claude/skills ~/.codex/skills ~/.cursor/skills \
  ~/.pi/agent/skills ~/.codeium/windsurf/skills ~/.continue/skills \
  -type l -exec sh -c '
    for p do printf "%s -> %s\n" "$p" "$(realpath "$p")"; done
  ' sh {} + 2>/dev/null
```

查找失效链接：

```bash
find ~/.claude ~/.codex ~/.cursor ~/.pi ~/.agents ~/.codeium ~/.continue \
  -type l ! -exec test -e {} \; -print 2>/dev/null
```

---

## 五、如何扫描 MCP 与 Skill

### 扫描用户级配置

```bash
rg -l --hidden \
  'mcpServers|mcp_servers|model.?context.?protocol' \
  ~/.claude ~/.claude.json ~/.codex ~/.cursor ~/.gemini \
  ~/.codeium ~/.continue \
  "$HOME/Library/Application Support/Claude" \
  "$HOME/Library/Application Support/Cursor" \
  "$HOME/Library/Application Support/Code/User" \
  2>/dev/null
```

### 扫描项目级配置

在代码根目录执行：

```bash
find . \
  \( -name node_modules -o -name .git -o -name .venv \) -prune -o \
  \( -name '.mcp.json' -o -path '*/.cursor/mcp.json' \
     -o -path '*/.vscode/mcp.json' -o -name 'SKILL.md' \
     -o -name 'AGENTS.md' -o -name 'CLAUDE.md' \) \
  -print
```

配置中可能含 Token、Cookie、API Key 和环境变量，不要公开完整内容。

---

## 六、MCP Server 程序本体

注册配置通常只保存命令或 URL，Server 本体可能在：

| 类型 | 常见位置或查询方式 |
|---|---|
| npx / npm | `~/.npm/_npx/`、项目 `node_modules/`、`npm root -g` |
| Python / uv / pipx | 项目 `.venv/`、`uv tool list`、`pipx list`、uv 缓存 |
| 独立程序 | `command -v <命令>`、`/opt/homebrew/bin/`、`~/.local/bin/` |
| Docker | `docker ps -a`、`docker images`、Docker volumes |
| 远程 MCP | 不一定有本地程序，只在配置中保存 URL 和认证信息 |

删除注册配置不会自动卸载 npm/Python 包、Docker 镜像或远程授权。

---

## 七、缓存不等于配置

例如：

```text
~/.cursor/projects/*/mcps/
~/.cursor/projects/*/mcp-cache.json
~/.codex/mcp-oauth-locks/
~/.claude/mcp-needs-auth-cache.json
```

这些通常表示 MCP 曾被发现、连接或索引过。清理 MCP 时应先处理总表中的注册配置，最后再清缓存。

---

## 八、安全清理原则

1. 退出所有 Agent 和 IDE。
2. 找到注册配置中的 MCP/Skill 条目。
3. 对每个 Skill 执行 `realpath`，确认它是链接还是真实文件。
4. 先取消注册或删除平台侧链接。
5. 确认没有其他平台共用后，再删除真实文件或卸载 Server。
6. 最后清缓存并重启验证。

一句话总结：

> **配置决定启用了什么，软链接决定从哪里引用，真实目录决定文件实际在哪里，缓存只说明它曾经运行过。**
