# OpenClaw 安装、使用、配置与架构笔记

> 版本基线：`OpenClaw 2026.3.2`（本机当前）
> 适用环境：Linux + Node.js 22 + 本地 Gateway 模式

## 1. OpenClaw 是什么

OpenClaw 可以理解为一个“本地可控的 Agent 网关系统”，它把以下能力组合在一起：

1. 多模型调度（不同 provider、主模型/回退模型、图片模型）。
2. 多渠道消息接入（Telegram、WhatsApp 等）。
3. Agent 会话管理（session、history、memory、tools）。
4. 本地网关（WebSocket + Dashboard）和服务化运行。
5. 安全控制（token/password、pairing、权限审计）。

## 2. 核心架构概念

```text
[User/Channel]
   |  (Telegram/Slack/...)
   v
[OpenClaw Gateway]
   |-- Session Router
   |-- Channel Connectors
   |-- Agent Runtime
   |-- Tools / Skills / Memory
   v
[Model Providers]
   |-- volcengine
   |-- google-vertex
   |-- (others)
```

### 2.1 关键组件

1. CLI（`openclaw`）
- 负责配置、启动、排障、发消息、发 agent 指令。

2. Gateway
- 运行时核心。
- 默认监听 `ws://127.0.0.1:18789`。
- Dashboard 默认 `http://127.0.0.1:18789/`。

3. Agent
- 真正执行模型调用、工具调用、上下文组装。
- 可以通过 `openclaw agent` 直接触发。

4. Channels
- 负责接入 Telegram 等外部渠道。
- 支持 pairing（授权发信人）和 allowlist 策略。

5. Models/Auth
- 模型配置决定“用谁推理”。
- Auth profile 决定“如何鉴权（API key/OAuth/token）”。

6. Memory/Workspace
- Memory 负责检索与索引。
- Workspace 放置 agent 运行上下文文件（AGENTS.md、TOOLS.md 等）。

## 3. 本机目录结构（重点）

OpenClaw 默认状态目录：`~/.openclaw`

常见子目录：

1. `~/.openclaw/openclaw.json`
- 主配置文件（最重要）。

2. `~/.openclaw/agents/main/agent/auth-profiles.json`
- provider 鉴权信息（API key 等）。

3. `~/.openclaw/agents/main/sessions/`
- 会话与上下文历史。

4. `~/.openclaw/memory/`
- memory 存储（例如 `main.sqlite`）。

5. `~/.openclaw/telegram/`
- Telegram 相关状态（offset、command hash）。

6. `~/.openclaw/logs/` 与 `/tmp/openclaw/*.log`
- 配置审计与运行日志。

## 4. 安装与初始化

## 4.1 安装（npm 全局）

```bash
npm i -g openclaw
openclaw --version
```

## 4.2 初始化配置

```bash
openclaw setup
# 或
openclaw configure
# 或
openclaw onboard
```

三者都能初始化，区别在于向导风格和覆盖范围，详见下文。

### 4.2.1 `openclaw setup` — 基础环境初始化

定位："Initialize config + workspace"，只负责打地基。

做的事情：
1. 创建 `~/.openclaw/` 目录结构。
2. 生成默认的 `openclaw.json` 主配置文件。
3. 初始化 workspace 目录（agent 运行上下文）。

常用参数：
- `--workspace <path>`：指定 agent workspace 路径（默认 `~/.openclaw/agents/main`）。
- `--wizard`：触发交互式向导（效果接近 `onboard`）。
- `--non-interactive`：非交互模式，适合脚本/CI 自动化部署。

```bash
# 最小化初始化（适合老手/自动化）
openclaw setup --non-interactive

# 带交互向导的初始化
openclaw setup --wizard
```

覆盖范围最窄：只管"文件和目录创建好了没"，不涉及模型选择、渠道接入、API 密钥配置。setup 结束后仍需手动编辑 `openclaw.json` 和 `auth-profiles.json`。

### 4.2.2 `openclaw configure` — 交互式配置调整

定位："Interactive configuration wizard (models, channels, skills, gateway)"，按模块逐项配置的向导。

做的事情：
1. 引导配置/修改模型 provider（选择哪个模型、设置 API key）。
2. 引导配置/修改渠道账户（Telegram bot token、WhatsApp 等）。
3. 引导配置/修改 Skills（启用/禁用哪些技能）。
4. 引导配置/修改 Gateway 参数（端口、bind、auth 模式）。

```bash
# 全量交互式配置
openclaw configure

# 也可以用 config 子命令做单项精确修改
openclaw config set agents.defaults.model.primary "volcengine/doubao-seed-2-0-pro-260215"
openclaw config get gateway.port
```

覆盖范围中等：假设环境已经存在（`setup` 已执行过），专注于对各模块参数进行精细调整。适合已有运行环境要换模型/换渠道、添加新 provider 或 channel、日常配置变更等场景。

### 4.2.3 `openclaw onboard` — 一站式端到端初始化

定位：最完整的引导向导，从零到可运行一步到位。官方推荐的首次安装路径。

做的事情（按顺序）：
1. 创建配置文件和 workspace（= `setup` 的工作）。
2. 选择运行模式（local / remote）。
3. 配置模型 provider + 输入 API key。
4. 配置渠道（Telegram、WhatsApp 等）并完成认证。
5. 安装 Skills。
6. 可选：安装 Gateway 守护进程（systemd/launchd）。

常用参数：
- `--install-daemon`：同时安装系统守护进程（推荐），让 Gateway 开机自启。
- `--mode <local|remote>`：运行模式，local=本地网关，remote=远程服务器。
- `--flow <quickstart|advanced|manual>`：向导详细程度，quickstart 最快上手，advanced 可调更多参数。
- `--reset`：清除已有配置后重新开始。
- `--anthropic-api-key <key>`：直接传入 API key（类似参数支持 OpenAI、Mistral 等）。

```bash
# 新机器推荐（官方推荐路径）
openclaw onboard --install-daemon

# 快速模式
openclaw onboard --flow quickstart --install-daemon

# 高级模式（更多可调参数）
openclaw onboard --flow advanced --mode local

# Docker 环境（不装 daemon，由 docker-compose 管理）
openclaw onboard --mode local --no-install-daemon

# 重置后重新配置
openclaw onboard --reset
```

覆盖范围最广：等于 `setup` + `configure` + daemon 安装，一次走完。

### 4.2.4 三者对比速查

| 能力 | `setup` | `configure` | `onboard` |
|---|---|---|---|
| 创建目录/配置文件 | Yes | No（需已存在） | Yes |
| 模型 provider 配置 | No | Yes | Yes |
| API key 输入 | No | Yes | Yes |
| 渠道（Telegram 等） | No | Yes | Yes |
| Skills 安装 | No | Yes | Yes |
| Gateway 参数 | No | Yes | Yes |
| 安装守护进程 | No | No | Yes（`--install-daemon`） |
| 适用场景 | 老手/CI/最小初始化 | 已有环境做局部调整 | 新机器从零开始 |

推荐实践：
1. 全新部署 → `openclaw onboard --install-daemon`，一步到位。
2. 已有环境改配置 → `openclaw configure` 或 `openclaw config set/get`。
3. CI/自动化脚本 → `openclaw setup --non-interactive` + `config set` 逐项写入。

## 4.3 启动与状态

```bash
openclaw gateway run          # 前台运行
openclaw status               # 全局状态
openclaw gateway status       # 网关状态
openclaw health               # 网关健康
```

## 4.4 更新

```bash
openclaw update status
openclaw update --yes
```

## 5. 配置文件 openclaw.json（字段速查）

以下是实战最常改的字段组。

### 5.1 `env`

用于 provider 的环境变量注入（例如 Google Vertex）：

```json
"env": {
  "GOOGLE_APPLICATION_CREDENTIALS": "/path/to/service-account.json",
  "GOOGLE_CLOUD_PROJECT": "your-project",
  "GOOGLE_CLOUD_LOCATION": "global"
}
```

### 5.2 `auth.profiles`

声明本地有哪些 provider profile（只声明 profile，不含真正密钥内容）：

```json
"auth": {
  "profiles": {
    "volcengine:default": { "provider": "volcengine", "mode": "api_key" }
  }
}
```

### 5.3 `models`

用于自定义 provider 接入（例如 OpenAI-compatible base URL）。

你当前主要使用：
- `volcengine` 自定义 provider（baseUrl + api + model metadata）。

### 5.4 `agents.defaults.model` 与 `agents.defaults.imageModel`

这两个是“实际默认路由”的核心：

```json
"model": {
  "primary": "volcengine/doubao-seed-2-0-pro-260215",
  "fallbacks": ["volcengine/doubao-seed-1-8-251228"]
}
```

```json
"imageModel": {
  "primary": "volcengine/doubao-seed-2-0-pro-260215",
  "fallbacks": ["volcengine/doubao-seed-1-8-251228"]
}
```

### 5.5 `channels.telegram`

Telegram Bot 入口配置：

```json
"channels": {
  "telegram": {
    "enabled": true,
    "dmPolicy": "pairing",
    "botToken": "<REDACTED>",
    "groupPolicy": "allowlist",
    "streaming": "partial"
  }
}
```

说明：
1. `dmPolicy=pairing`：未授权用户先收到 pairing code。
2. `groupPolicy=allowlist`：未在 allowlist 的群消息会被丢弃。

### 5.6 `gateway`

本地网关监听、鉴权与暴露方式：

```json
"gateway": {
  "port": 18789,
  "mode": "local",
  "bind": "loopback",
  "auth": { "mode": "token", "token": "<REDACTED>" }
}
```

## 6. Auth Profiles（密钥实际存放处）

文件：`~/.openclaw/agents/main/agent/auth-profiles.json`

示例（已脱敏）：

```json
{
  "profiles": {
    "volcengine:default": {
      "type": "api_key",
      "provider": "volcengine",
      "key": "<REDACTED>"
    }
  }
}
```

建议：
1. 不要把该文件提交到 Git。
2. 使用最小权限 API key。
3. 定期轮换密钥。

## 7. 常用命令手册

## 7.1 模型相关

```bash
openclaw models status
openclaw models list
openclaw models set volcengine/doubao-seed-2-0-pro-260215
openclaw models fallbacks clear
openclaw models fallbacks add volcengine/doubao-seed-1-8-251228
openclaw models set-image volcengine/doubao-seed-2-0-pro-260215
openclaw models image-fallbacks clear
openclaw models image-fallbacks add volcengine/doubao-seed-1-8-251228
openclaw models status --probe --probe-provider volcengine
```

## 7.2 Agent 调用

```bash
openclaw agent --local --agent main --message "请只回复 OK" --json
openclaw agent --agent main --message "总结最近日志" --json
```

## 7.3 渠道相关（Telegram）

```bash
openclaw channels list
openclaw channels status --probe
openclaw channels add --channel telegram --token <BOT_TOKEN>
openclaw message send --channel telegram --target <CHAT_ID> --message "hello" --json
```

Pairing 审批：

```bash
openclaw pairing list --json
openclaw pairing approve telegram <PAIRING_CODE>
```

## 7.4 网关与日志

```bash
openclaw gateway run
openclaw gateway status
openclaw logs --follow
openclaw status --deep
```

## 8. 典型工作流（推荐）

### 8.1 新机器冷启动

1. 安装 OpenClaw。
2. `openclaw setup` 初始化。
3. 配置模型 provider 与 auth profile。
4. 配置 Telegram channel。
5. `openclaw gateway run` 启动。
6. `openclaw models status --probe` + `openclaw agent --local` 双重验证。

### 8.2 变更默认模型

1. `openclaw models set <provider/model>`。
2. 更新 fallback 与 image model。
3. `openclaw models status` 检查。
4. `openclaw agent --local ...` 做真实回复验证。

### 8.3 处理 Telegram “access not configured”

1. 用户收到 pairing code。
2. 管理员执行：`openclaw pairing approve telegram <code>`。
3. 发送测试消息确认权限生效。

## 9. 常见问题与排障

### 9.1 Probe 通过但实际调用失败

现象：
- `models status --probe` 显示 `ok`
- `agent --local` 却报错

建议排查顺序：

1. 先测真实调用（`openclaw agent --local ...`）。
2. 直测 provider auth（例如用官方 SDK 获取 token）。
3. 直测网络连通（`curl oauth2.googleapis.com` 等）。
4. 看 `/tmp/openclaw/*.log` + `openclaw logs --follow`。

### 9.2 Telegram 群消息不触发

如果 `groupPolicy=allowlist` 且没配置 `groupAllowFrom`，群消息会被静默丢弃。

解决：
1. 设置 `groupAllowFrom`。
2. 或将 `groupPolicy` 改为 `open`（谨慎）。

### 9.3 安全审计告警

常见修复：

```bash
chmod 700 ~/.openclaw
chmod 700 ~/.openclaw/credentials
```

## 10. 安全与运维建议

1. 配置文件和凭据目录权限最小化（700）。
2. API key 放在 auth-profiles，不放脚本源码。
3. 按环境分 profile（dev/staging/prod）。
4. 变更前备份 `openclaw.json` 与 `auth-profiles.json`。
5. 每次改模型后都做一次真实消息回归（不是只看 probe）。

## 11. 当前实例（你的机器）摘要

1. Gateway：`loopback:18789` + token auth。
2. Channel：Telegram 已启用。
3. 默认文本模型：豆包 `doubao-seed-2-0-pro-260215`。
4. fallback：`doubao-seed-1-8-251228`。
5. Vertex 配置保留在 `env`，但当前默认生产链路已切豆包。

## 12. 一页速查（最小命令集）

```bash
# 看状态
openclaw status
openclaw models status

# 启动网关
openclaw gateway run

# 发起一次本地 agent 调用
openclaw agent --local --agent main --message "请回复OK" --json

# Telegram 发送测试
openclaw message send --channel telegram --target <CHAT_ID> --message "test" --json

# 审批 pairing
openclaw pairing approve telegram <PAIRING_CODE>
```

---

如果你愿意，我可以在这份笔记基础上再追加两部分：
1. “按你的现网配置生成一份脱敏模板 openclaw.json”。
2. “生产环境发布清单（备份、回滚、监控、告警）”。
