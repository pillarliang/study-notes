# 09 - Skills：按需加载的能力包

> 来源：https://pi.dev/docs/latest/skills

## 1. Skills 是什么

Skill 是按需加载的能力包——每个 skill 把"针对某个任务域的 workflow、setup 步骤、helper 脚本、参考文档"打成一个 bundle。

Pi 遵循 [Agent Skills standard](https://agentskills.io)，但**宽松**——多数违规只产生 warning 不会拒载；Pi 也**不要求 `name` 跟父目录同名**（方便跨多个 agent 工具共享 skill 目录）。

### 跟 Extensions 的区别

| 维度 | Skill | Extension |
|------|-------|-----------|
| 形式 | markdown + 辅助脚本/文档 | TypeScript 模块 |
| 加载时机 | 按需（model 决定或显式调用） | 启动时全部加载 |
| 权限 | 模型间接执行（通过现有 tool） | 直接跑代码 |
| 主要用途 | 把 workflow 文档化、可复用 | 改 Pi 自身行为、加 tool |

详见 [[08-Extensions-扩展编写]]。

## 2. Skill 文件存放位置

Pi 搜索这些位置：

| 类型 | 位置 |
|------|------|
| **全局** | `~/.pi/agent/skills/` 和 `~/.agents/skills/` |
| **项目** | `.pi/skills/`，外加从 cwd 向上到 git root（或文件系统根）的所有 `.agents/skills/` |
| **Package** | package 里的 `skills/` 文件夹 或 `package.json` 的 `pi.skills` 入口 |
| **Settings** | `skills` 数组指向文件或目录 |
| **CLI** | `--skill <path>`，可重复，**对 `--no-skills` 也叠加生效** |

发现规则细节：

- `~/.pi/agent/skills/` 和 `.pi/skills/` 根目录下的散文件 `.md` 算独立 skill
- **任何含 `SKILL.md` 的目录**，在任意上述位置都会被递归发现
- `.agents/skills/` 的根目录散文件 `.md` **不算**

### 复用其它 harness 的 skill

```json
{
  "skills": [
    "~/.claude/skills",
    "~/.codex/skills"
  ]
}
```

项目级 `.pi/settings.json` 复用 Claude Code 的项目 skill：

```json
{
  "skills": ["../.claude/skills"]
}
```

## 3. 调用方式

启动时 Pi 扫描各位置，提取每个 skill 的 `name` 和 `description`，按 spec 的 XML 格式塞进 system prompt——**完整指令留在磁盘上，要用时才载入**。这叫 **progressive disclosure**：descriptions always in context, full instructions load on-demand。

三种触发方式：

### 3.1 自动 / model-invoked

任务跟 description 匹配时，agent 自己去读 SKILL.md。

> 模型在这块可能不稳定，**直接提示** 或用 slash 命令保证一定加载。

### 3.2 Slash 命令

每个 skill 自动注册成 `/skill:name`：

```text
/skill:brave-search           # 加载并执行 skill
/skill:pdf-tools extract      # 加载 skill + 带参数
```

命令名后面的内容会作为 `User: <args>` 拼到 skill 文本后面。

开关：`/settings` 或：

```json
{ "enableSkillCommands": true }
```

### 3.3 显式 application invocation

CLI 的 `--skill <path>`——**对 `--no-skills` 也叠加生效**。

### 隐藏 skill 的 model 调用

frontmatter 加 `disable-model-invocation: true`——skill 从 system prompt 里消失，**只能通过 `/skill:name` 调**。

> **安全警告**：skill 能引导 model 做任何事，也可能带可执行代码。**装之前看一眼**。

## 4. Skill 目录布局

```text
my-skill/
├── SKILL.md              # 必需：frontmatter + 指令
├── scripts/              # 辅助脚本
│   └── process.sh
├── references/           # 按需加载的详细文档
│   └── api-reference.md
└── assets/
    └── template.json
```

## 5. SKILL.md 格式

````markdown
---
name: my-skill
description: What this skill does and when to use it. Be specific.
---

# My Skill

## Setup

Run once before first use:

```bash
cd /path/to/skill && npm install
```

## Usage

```bash
./scripts/process.sh <input>
```
````

引用 bundled 文件用相对路径：

```markdown
See [the reference guide](references/REFERENCE.md) for details.
```

## 6. Frontmatter 字段

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | ✅ | 最长 64 字符；小写字母、数字、连字符。Pi 豁免了 spec "name 必须跟目录同名"的规则 |
| `description` | ✅ | 最长 1024 字符；说明用途和调用时机 |
| `license` | ❌ | license 名或指向 bundled license 文件 |
| `compatibility` | ❌ | 最长 500 字符，描述环境要求 |
| `metadata` | ❌ | 自由格式 key/value |
| `allowed-tools` | ❌ | 空格分隔的预批准 tool 列表（实验性） |
| `disable-model-invocation` | ❌ | 设为 true 后只能通过 `/skill:name` 调 |

### name 规则

- 长度 1–64
- 只允许小写 ASCII 字母、数字、连字符
- **不能**以连字符开头或结尾
- **不能**有连续连字符

| 合法 | 不合法 |
|------|--------|
| `pdf-processing` | `PDF-Processing` |
| `data-analysis` | `-pdf` |
| `code-review` | `pdf--processing` |

### description 写法

description 驱动 auto-loading——**要具体**。

✅ 好：

```text
description: Extracts text and tables from PDF files, fills PDF forms, and merges multiple PDFs. Use when working with PDF documents.
```

❌ 差：

```text
description: Helps with PDFs.
```

## 7. Validation 行为

Pi 对下列问题**只产 warning，仍加载**：

- name 过长
- 非法字符
- 连字符位置问题
- description 超长

未识别的 frontmatter key **静默丢弃**。

**唯一硬失败**：没 description → 不加载。

同名 skill 冲突时 Pi 警告，**保留先发现的那一个**。

## 8. 实战示例

```text
brave-search/
├── SKILL.md
├── search.js
└── content.js
```

`SKILL.md`：

````markdown
---
name: brave-search
description: Web search and content extraction via Brave Search API. Use for searching documentation, facts, or any web content.
---

# Brave Search

## Setup

```bash
cd /path/to/brave-search && npm install
```

## Search

```bash
./search.js "query"              # Basic search
./search.js "query" --content    # Include page content
```

## Extract Page Content

```bash
./content.js https://example.com
```
````

## 9. 现成 skill 仓库

- [Anthropic Skills](https://github.com/anthropics/skills)——docx / pdf / pptx / xlsx 处理器、Web dev helper
- [Pi Skills](https://github.com/badlogic/pi-skills)——web search、browser automation、Google API、转写

> **文档原话**："pi can create skills. Ask it to build one for your use case."
