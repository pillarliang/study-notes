# ADK Agent Skill 五大设计模式 — 学习笔记

> 来源：
>
> - [5 Agent Skill Design Patterns Every ADK Developer Should Know](https://lavinigam.com/posts/adk-skill-design-patterns/) — Lavi Nigam, 2026-03-07
> - [Agent Skills 官方规范](https://agentskills.io/specification) — agentskills.io
> - [Claude Skills 笔记](https://www.notion.so/pillarl/Claude-skills-2bf0718c9ab080018af0f6a6f70a2539) — Notion 早期笔记

---

## 一、为什么需要 Skill？— Tool 与 Skill 的本质区别

在 Google ADK（Agent Development Kit，Google 推出的 Agent 开发框架）中，**Tool** 和 **Skill** 是两个经常混淆但本质不同的概念：


| 维度  | Tool（工具）             | Skill（技能）                         |
| --- | -------------------- | --------------------------------- |
| 本质  | 赋予 Agent **执行动作**的能力 | 教会 Agent **何时、如何**使用工具            |
| 类比  | "调用天气 API"           | "当用户问旅行时，查询每个目的地的天气，比较结果，格式化为行程表" |
| 关注点 | 单一操作                 | 编排与决策                             |


一句话总结：**Tool 是手，Skill 是脑。** Tool 让 Agent 能做事，Skill 让 Agent 知道什么时候做什么事、怎么组合着做。

---

## 二、ADK Skill 的技术基础

### 2.1 SKILL.md 文件

每个 Skill 的核心是一个 `SKILL.md` 文件，它定义了 Skill 的名称、描述、指令以及可选的引用资源。一个 Skill 的目录结构通常包含：

```
my-skill/
├── SKILL.md          # 核心指令文件（必需）
├── references/       # 参考文档（约定、检查清单等）
├── assets/           # 静态资源（模板、图片、数据文件）
└── scripts/          # 可执行脚本（Python/Bash/JS）
```

#### SKILL.md 的 Frontmatter 字段（基于 agentskills.io 规范）

SKILL.md 文件由 **YAML Frontmatter**（配置）+ **Markdown Body**（指令）两部分组成：

```yaml
---
name: pdf-processing           # 必填，1-64 字符，小写+连字符，须与文件夹名一致
description: >                 # 必填，1-1024 字符，描述 Skill 做什么、何时使用
  Extract PDF text, fill forms, merge files.
  Use when handling PDFs.
license: Apache-2.0            # 可选，许可证
compatibility: >               # 可选，环境要求（目标产品、系统依赖等）
  Requires Python 3.14+ and uv
metadata:                      # 可选，自定义键值对（用于管理/审计）
  author: example-org
  version: "1.0"
allowed-tools: Bash(git:*) Read Write  # 可选（实验性），空格分隔的预授权工具列表
---

（Markdown 指令内容...）
```

> `**name` 命名规则**：仅允许小写字母、数字和连字符；不能以连字符开头/结尾；不能有连续连字符（`--`）；**必须与父目录名一致**。

#### 三个可选目录的区别（基于 agentskills.io 官方规范）

这是理解 Skill 结构的关键。三个目录的内容都在 **L3（Resources）** 层级按需加载，但它们的**用途和内容类型**不同：


| 维度         | `references/`                                                      | `assets/`                                                    | `scripts/`                              |
| ---------- | ------------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------- |
| **本质**     | 指导 Agent **怎么做**的知识                                                | 提供 Agent **输出素材**的资源                                         | Agent **执行**的代码                         |
| **与输出的关系** | 影响过程，**不出现在**最终输出中                                                 | 成为输出的**一部分**                                                 | 产生中间数据供 Agent 使用                        |
| **用途**     | 为 Agent 提供决策所需的背景知识、规范、检查清单                                        | 提供输出模板、配置模板、示例图片、数据 schema                                   | 将复杂计算逻辑卸载给可执行脚本                         |
| **加载方式**   | 按需加载（Agent 用 Read 工具读取文件内容到上下文）                                    | 按需加载（文本文件读取内容；图片等非文本资源引用路径）                                  | Agent 用 Bash 工具执行                       |
| **典型文件**   | `REFERENCE.md`、`conventions.md`、`review-checklist.md`、`finance.md` | `report-template.md`、`config-template.yaml`、`schema.json`、图片 | `analyze.py`、`extract.sh`、`validate.js` |
| **设计原则**   | 保持单个文件聚焦，越小越好（节省上下文）                                               | 模板与指令分离，换模板即换输出                                              | 自包含，清晰的错误消息，优雅处理边界情况                    |


> **加载方式说明**：agentskills.io 规范对三个目录的加载方式描述是统一的——"Files in scripts/, references/, or assets/ are loaded only when required"，三者都是按需加载，不存在"references 读内容、assets 只引用路径"的区别。

**具体区别解析：**

一句话概括：**references 是给 Agent "看的参考书"，assets 是 Agent "用来组装产品的零件"。**

1. `**references/`（参考文档）**：Agent 需要**理解其内容**才能完成任务。比如编码规范文档——Agent 必须读懂规范才能按规范写代码。这些知识指导 Agent 的行为，但不会直接出现在最终输出中。规范建议保持单个 reference 文件聚焦且精简，以节省上下文 token。
2. `**assets/`（静态资源）**：Agent 用来**组装输出**的素材。比如报告模板——Agent 读取模板结构，填充内容后输出。图片等非文本资源则通过路径引用嵌入输出（如 `![架构图](assets/architecture.png)`）。资源类型广泛，包括模板、图片、数据文件、lookup tables 等。
3. `**scripts/`（可执行脚本）**：Agent 不需要理解脚本内部逻辑，只需要**运行它并处理输出**。适合复杂计算、数据处理等不适合 LLM 直接做的任务。支持的语言取决于 Agent 实现，常见的有 Python、Bash、JavaScript。

**在 SKILL.md 中引用这些文件时，使用相对路径：**

```markdown
# 引用 reference（Agent 会读取内容）
请阅读 [编码规范](references/conventions.md) 了解最佳实践。

# 引用 asset（Agent 会使用模板）
按照 assets/report-template.md 的结构生成报告。

# 执行 script（Agent 会运行脚本）
运行 scripts/analyze.py 处理数据。
```

> **注意**：保持文件引用在一层深度，避免深度嵌套的引用链。主 SKILL.md 建议控制在 500 行以内，详细材料拆分到这三个目录中。


### 2.2 内部架构：元工具与双消息机制

理解 Skill 的运行原理有助于编写更好的 Skill。

#### 元工具（Meta-Tool）

系统中存在一个名为 "Skill" 的**元工具**，它作为所有单个 Skill 的**容器和调度器**。Agent 并不直接"知道"每个 Skill 的存在，而是通过这个统一的 Skill Tool 来发现和激活具体的 Skill。

#### 双消息机制（Dual-Message）

当 Skill 被激活时，系统注入**两条消息**，分别服务于不同目的：


| 消息类型          | 可见性            | 内容                                 | 目的               |
| ------------- | -------------- | ---------------------------------- | ---------------- |
| **元数据消息**     | 用户可见           | "正在加载 PDF Skill"                   | 保持透明度，让用户知道发生了什么 |
| **Prompt 消息** | 用户不可见（仅发给 API） | 完整的 SKILL.md 内容（标记 `isMeta: true`） | 注入指令，避免刷屏        |


这个设计解决了一个矛盾：**Agent 需要大量指令才能表现好，但用户不想看到数千字的 prompt 刷屏。**

#### 执行生命周期（四步）

以一个 PDF 处理 Skill 为例：

```
1. 发现（Discovery）
   → 系统扫描 skill 目录，加载所有 Skill 的 name + description（L1）

2. 决策（Decision）
   → 用户说"帮我提取这个 PDF 的文本"
   → Agent 匹配 description 关键词，决定激活 pdf-processing skill

3. 注入（Injection）
   → 加载完整 SKILL.md 内容注入对话上下文（L2）
   → 动态授予 allowed-tools 中指定的工具权限（无需用户再次确认）
   → 按需加载 references/、assets/、scripts/ 中的文件（L3）

4. 执行（Execution）
   → Agent 基于注入的指令，使用标准工具（Bash、Read、Write 等）完成任务
```

> **关键洞察**：传统工具是为了**"执行动作"**（如读文件），而 Skill 是为了**"改变大脑"**（赋予 Agent 新的思维方式和流程知识）。Skill 激活后，Agent 相当于临时变身为一个"专用 Agent"。


### 2.3 Skill 解决的四个核心问题

| 问题                                                 | Skill 的解决方案                                  |
| -------------------------------------------------- | -------------------------------------------- |
| **通用模型 vs 特定任务** — LLM 缺乏特定领域的深度上下文                | 通过**上下文注入**将通用 Agent 临时转变为"专用 Agent"         |
| **复杂工作流引导** — 单一工具无法指导多步骤业务逻辑                      | Skill 不直接执行动作，而是为 Agent **注入步骤说明**，指导其调度基础工具 |
| **上下文窗口效率** — 所有指令塞进 System Prompt 会浪费 token 且导致混淆 | **渐进式披露**机制，按需加载                             |
| **用户体验 vs 底层复杂性** — Agent 需要大量 prompt 但用户不想看刷屏     | **双消息机制**，一条可见（状态），一条隐藏（完整指令）                |


### 2.4 渐进式披露（Progressive Disclosure）

Skill 的加载分三个层级，目的是**节省上下文窗口的 token 消耗**：

- **L1（List）**：仅加载 Skill 名称和描述，约 100 tokens/skill — 用于让 Agent 判断是否需要激活该 Skill
- **L2（Load）**：加载完整的 SKILL.md 指令（建议 < 5000 tokens）— Skill 被激活后才加载
- **L3（Resources）**：按需加载 `references/`、`assets/`、`scripts/` 中的文件 — 指令中引用时才加载

这意味着即使注册了 50 个 Skill，L1 层级的总开销也仅约 5,000–7,500 tokens，对 128K+ 上下文窗口的模型来说完全可控。

#### Skill 数量进一步增长时：从"全量列出"到"按需召回"

L1 全量列出策略在 Skill 数量适中时（几十个）够用，但当 Skill 库扩展到上百甚至上千个时，会出现两个问题：

1. **Token 成本累积**：1000 个 Skill × 100 tokens ≈ 100K tokens 常驻上下文，对中等规模模型几乎吃掉一半窗口
2. **注意力稀释**：候选项越多，模型越容易选错或忽略冷门 Skill

下一步演进方向是借鉴 [Tool Search 范式](agent_advanced_tools.md)（Anthropic 在 Claude 3 Opus 上验证的工具调用准确率从 49% 提升到 74%），把 L1 的"全量列出"改为"语义召回"：

- **L1 层**：从"列出全部 Skill 的 name+description"改为暴露 `search_skills(query)` 接口，按需召回 Top-K
- **决策方式**：从"LLM 从全量列表中匹配"改为"LLM 从召回的少量候选中选择"
- **适用规模**：从几十个 Skill 扩展到上百乃至上千个 Skill

保障召回准确率的关键机制（与 Tool Search 同源）：

- **混合检索**：向量（语义）+ BM25（关键词），避免漏掉专有名词或特定 ID
- **描述工程**：description 字段必须写清 *Use when* / *Don't use when*，触发条件越具体召回越准
- **自主试错**：首次召回不命中时，模型可以改写 query 重新召回

> **Claude Code 当前的处理方式不是 RAG，而是截断**：listing 预算约为上下文窗口的 1%，每个 Skill 描述上限 `MAX_LISTING_DESC_CHARS = 250` 字符，超出预算时内置 Skill 保留完整描述、非内置 Skill 被截断（详见 [Claude Code Harness Engineering 8.3 节](../harness-engineering/Claude_Code-Harness_Engineering.md)）。这套策略在百级 Skill 之内仍可用，但本质是工程权衡而非根本解；一旦进入千级生态，召回式架构几乎不可避免。


### 2.5 SkillToolset 加载方式

在 ADK Python 代码中通过 `SkillToolset` 注册 Skill：

```python
skill_toolset = SkillToolset(
    skills=[
        load_skill_from_dir(SKILLS_DIR / "skill-name"),
    ],
)
```


### 2.6 Description 字段的重要性

Description 字段相当于 Skill 的**搜索索引**。Agent 根据用户输入匹配 Description 中的关键词来决定是否激活 Skill。

- 反面示例：`"Helps with APIs"` — 太泛，几乎不会被触发
- 正面示例：`"FastAPI, REST APIs, Pydantic models, async endpoints"` — 具体关键词确保可靠触发



### 2.7 存储位置与跨平台兼容

存储位置因 Agent 实现而异：

| 范围             | agentskills.io 标准路径         | Claude Code 路径              |
| -------------- | --------------------------- | --------------------------- |
| **项目级**（团队共享）  | `<project>/.agents/skills/` | `<project>/.claude/skills/` |
| **用户级**（个人跨项目） | `~/.agents/skills/`         | `~/.claude/skills/`         |


Skills 遵循 [agentskills.io](https://agentskills.io) 开放规范（由 Anthropic 发起），已被 **30+ Agent 产品**采用，包括：Claude Code、Gemini CLI、Cursor、GitHub Copilot、VS Code、OpenAI Codex、Roo Code、Kiro、JetBrains Junie、Databricks 等。



### 2.8 Skill 的安装与管理

笔记前面讲了 Skill 的格式和存储位置，但**怎么把别人写好的 Skill 装到自己的环境里？** 这里介绍三种主流方式。

#### 方式一：`npx skills` CLI（通用，推荐）

[skills.sh](https://skills.sh) 提供了一个跨 Agent 平台的 CLI 工具，通过 `npx` 直接运行，无需全局安装：

**基本用法：**

```bash
# 安装某个 GitHub 仓库中的 Skill（交互式选择具体 Skill 和目标 Agent）
npx skills add <owner/repo>

# 示例：安装 Vercel 的 Agent Skills
npx skills add vercel-labs/agent-skills

# 安装 Google ADK 官方 Skills
npx skills add google/adk-docs -y -g
```

**关键参数：**


| 参数                     | 说明                                                 | 示例                                             |
| ---------------------- | -------------------------------------------------- | ---------------------------------------------- |
| `-g, --global`         | 安装到**用户级**（`~/.agents/skills/`），而非项目级              | `npx skills add foo/bar -g`                    |
| `-y, --yes`            | 跳过确认提示，自动安装                                        | `npx skills add foo/bar -y`                    |
| `-a, --agent <agents>` | 指定安装到哪些 Agent（如 `claude-code`、`cursor`），用 `*` 表示所有 | `npx skills add foo/bar -a claude-code cursor` |
| `-s, --skill <skills>` | 仓库中有多个 Skill 时，指定安装哪些                              | `npx skills add foo/bar -s pr-review commit`   |
| `--all`                | `--skill '*' --agent '*' -y` 的简写，全部安装、跳过确认         | `npx skills add foo/bar --all`                 |
| `--copy`               | 复制文件而非创建符号链接                                       | `npx skills add foo/bar --copy`                |


**其他管理命令：**

```bash
# 列出已安装的 Skills
npx skills list              # 项目级
npx skills ls -g             # 用户级
npx skills ls -a claude-code # 按 Agent 过滤
npx skills ls --json         # JSON 格式输出

# 搜索 Skills
npx skills find              # 交互式搜索
npx skills find typescript   # 按关键词搜索

# 更新 Skills
npx skills update            # 更新所有
npx skills update my-skill   # 更新指定 Skill
npx skills update -g         # 仅更新用户级

# 删除 Skills
npx skills remove            # 交互式选择删除
npx skills remove web-design # 按名称删除
npx skills rm --global foo   # 删除用户级的指定 Skill

# 初始化新 Skill（脚手架）
npx skills init my-skill     # 创建 my-skill/SKILL.md
```

> **安装原理**：`npx skills add` 会从 GitHub 仓库拉取 Skill 文件，按照目标 Agent 的约定路径（如 Claude Code 的 `.claude/skills/`、VS Code 的 `.agents/skills/`）放置文件，默认使用**符号链接**（symlink）方式。


#### 方式二：Claude Code Plugin 系统

Claude Code 自身有一套 Plugin 机制（Plugin 是比 Skill 更大的概念，可以包含 Skill、MCP Server 等），提供了三种安装方式：

**方式 2a：通过 Marketplace（推荐）**

```bash
# 步骤 1：添加 marketplace（GitHub 仓库作为 Skill 源）
/plugin marketplace add <owner/repo>

# 步骤 2：安装具体的插件
/plugin install <plugin-name>
```

以 [python-code-style](https://github.com/pillarliang/python-code-style) 为例：

```bash
/plugin marketplace add pillarliang/python-code-style
/plugin install python-code-style
```

**方式 2b：通过 `settings.json` 配置**

在 `~/.claude/settings.json` 中声明 marketplace 和启用的插件：

```json
{
  "extraKnownMarketplaces": {
    "python-code-style": {
      "source": {
        "source": "github",
        "repo": "pillarliang/python-code-style"
      }
    }
  },
  "enabledPlugins": {
    "python-code-style@python-code-style": true
  }
}
```

**方式 2c：手动安装**

```bash
git clone https://github.com/pillarliang/python-code-style.git
cp -r python-code-style ~/.claude/plugins/python-code-style
```

**插件管理：**

```bash
# 查看已安装的插件
/plugin list

# 更新 marketplace 中的插件
/plugin marketplace update <owner/repo>

# 自动更新：运行 /plugin → Marketplaces 标签页 → 选择 marketplace → Enable auto-update
```

#### 方式三：手动放置文件

最朴素的方式——直接把 Skill 目录复制到对应路径：

```bash
# 项目级（团队共享，提交到 Git）
cp -r my-skill/ <project>/.claude/skills/my-skill/   # Claude Code
cp -r my-skill/ <project>/.agents/skills/my-skill/   # VS Code / Copilot 等

# 用户级（个人跨项目）
cp -r my-skill/ ~/.claude/skills/my-skill/            # Claude Code
cp -r my-skill/ ~/.agents/skills/my-skill/            # 其他 Agent
```

#### 项目级 vs 用户级：如何选择？


| 维度            | 项目级（Project）                | 用户级（Global）         |
| ------------- | --------------------------- | ------------------- |
| **路径**        | `<project>/.claude/skills/` | `~/.claude/skills/` |
| **谁能用**       | 所有克隆此仓库的团队成员                | 仅当前用户               |
| **是否提交到 Git** | 是，团队共享                      | 否，仅本地               |
| **典型场景**      | 团队编码规范、项目特定的工作流             | 个人效率工具、通用写作风格       |
| **CLI 参数**    | 默认（无 `-g`）                  | `-g` / `--global`   |


#### Skill 的删除

删除方式与安装方式一一对应，本质上是"装哪儿就在哪儿删"。

**方式一：`npx skills remove`（对应 `npx skills add` 安装的）**

```bash
# 交互式选择删除
npx skills remove

# 按名称删除项目级 Skill
npx skills remove <skill-name>

# 删除用户级 Skill（两种写法等价）
npx skills rm --global <skill-name>
npx skills rm -g <skill-name>

# 按 Agent 过滤后删除
npx skills remove -a claude-code
```

**方式二：Claude Code Plugin 卸载（对应 `/plugin install` 安装的）**

```bash
# 交互式管理界面
/plugin

# 直接卸载指定插件
/plugin uninstall <plugin-name>

# 移除整个 marketplace（会同时移除其下所有已安装插件）
/plugin marketplace remove <owner/repo>
```

也可以编辑 `~/.claude/settings.json`，把对应插件的 `enabledPlugins` 值改为 `false` 或删除该条目，效果等同于卸载。

**方式三：手动删除文件（对应手动复制安装的）**

```bash
# 项目级
rm -rf <project>/.claude/skills/<skill-name>/
rm -rf <project>/.agents/skills/<skill-name>/

# 用户级
rm -rf ~/.claude/skills/<skill-name>/
rm -rf ~/.agents/skills/<skill-name>/
```

**如何判断 Skill 是哪种方式安装的？**

先用以下命令排查：

```bash
npx skills list -g          # 用户级，npx 装的会显示
npx skills list             # 项目级
ls ~/.claude/skills/        # 直接看目录，手动放的也会在
ls ~/.claude/plugins/       # Plugin 装的在这里
```

> **判断技巧**：`npx skills list` 输出里如果是符号链接（symlink），说明是 `npx skills add` 装的；如果是普通目录，可能是手动复制的。Plugin 装的 Skill 通常会出现在 `~/.claude/plugins/<plugin-name>/skills/` 之下，而非 `~/.claude/skills/`。

#### npx skills 安装机制与运维细节

前面讲了 `npx skills` 的基本命令,但日常使用会遇到几个让人困惑的现象:为什么有的 skill 装在 `~/.agents/skills/`、有的在 `~/.claude/skills/`?为什么 `npx skills rm` 有时报"找不到"?为什么 Claude Code 能用 agentskills.io 标准路径下的 skill?这些都源于 npx skills 的**双路径 + symlink + 注册表**三件套机制。

##### 1. 跨 Agent 双路径:为什么会出现两个 skills/ 目录

agentskills.io 规范定义的标准路径是 `~/.agents/skills/`,30+ Agent 都按这个路径找 skill。但 Claude Code 用的是它私有的历史路径 `~/.claude/skills/`,Anthropic 不愿意改。

| Agent | 用户级 skill 路径 |
|---|---|
| **Claude Code**(私有) | `~/.claude/skills/` |
| **Cursor / VS Code Copilot / Gemini CLI / Codex 等 30+ Agent** | `~/.agents/skills/` |

`npx skills` 是跨 Agent 工具,要同时支持这两套路径。具体写到哪里由 `-a/--agent` 参数决定:

```bash
npx skills add foo/bar -a claude-code -g    # → ~/.claude/skills/
npx skills add foo/bar -a cursor -g         # → ~/.agents/skills/
npx skills add foo/bar -a '*' -g            # → 同时多处(见下文 symlink 机制)
```

> 不指定 `-a` 时,npx skills 会检测当前机器装了哪些 Agent,弹交互式列表让用户勾选。**建议总是显式写 `-a` 让行为可预测。**

##### 2. symlink 双写策略:一份真实文件服务多个 Agent

跑 `npx skills add foo -a '*' -g` 时,npx skills 实际做的事:

```
1. git clone foo → 缓存到本地(如 ~/.skills/cache/foo/)
2. 真实文件落地到 ~/.agents/skills/foo (agentskills.io 标准路径)
3. 为每个目标 Agent 创建 symlink:
   ~/.claude/skills/foo  → ~/.agents/skills/foo  (Claude Code 用)
   (其他原生支持 ~/.agents/skills/ 的 Agent 不需要额外 symlink)
```

效果:**一份真实文件占一份磁盘空间,通过 symlink 服务所有 Agent**。

> 默认行为是 symlink,加 `--copy` 参数才会改用复制。symlink 的好处是 `npx skills update foo` 一次更新所有 Agent 的 foo,因为它们都指向同一份真实文件。

##### 3. Claude Code 与 ~/.agents/skills/ 的关系

> **关键认知:Claude Code 不直接扫 `~/.agents/skills/`**

Claude Code 只在以下路径找 skill:

| 范围 | 路径 |
|---|---|
| 用户级 | `~/.claude/skills/<name>/SKILL.md` |
| 项目级 | `<project>/.claude/skills/<name>/SKILL.md` |
| Plugin 带的 | `~/.claude/plugins/<plugin>/skills/...` |

`~/.agents/skills/` 不在扫描列表。**那 npx 装的 skill 为何能在 Claude Code 用?靠 `~/.claude/skills/` 下的 symlink 指过去**。

`npx skills list` 输出里 "Agents: ..., Claude Code, ..." 这一行的含义:**安装时被选中作为目标的 Agent 列表**(每个 Agent 都已通过约定路径下的 symlink 访问到这个 skill),不是说 Claude Code 直接读了 `~/.agents/skills/`。

验证某个 skill 是真目录还是 symlink:

```bash
ls -la ~/.claude/skills/<skill-name>
# 输出含 -> 的就是 symlink:
# lrwxr-xr-x  ...  ~/.claude/skills/lark-okr -> /Users/.../.agents/skills/lark-okr
```

##### 4. npx skills 的注册表:list/rm 为何"看不见"或"删不掉"

`npx skills list` 和 `npx skills rm` **不是直接扫描文件系统**,而是查 npx skills 自己维护的**安装注册表**(记录"我装过哪些 skill")。

| 安装方式 | 在 npx 注册表里? | `npx skills list` 看得到? | `npx skills rm` 能删? |
|---|---|---|---|
| `npx skills add` | ✅ | ✅ | ✅ |
| 手动 `git clone` 或 `cp` | ❌ | ❌ | ❌ "No matching skills found" |
| `/plugin install` 装的 | ❌ | ❌ | ❌ 同上 |
| Claude Code 内置 skill | ❌ | ❌ | ❌ 同上 |

**`npx skills list` 只是 npx 自己的视角,不是文件系统全景**。想看真相,直接用 `ls`:

```bash
ls ~/.claude/skills/                              # Claude Code 私有路径
ls ~/.agents/skills/                              # agentskills.io 标准路径
ls -d ~/.claude/plugins/*/skills/*/ 2>/dev/null   # Plugin 带的
```

##### 5. pull 模型:无自动更新通知

不像 Claude Code Plugin 在启动时自动检查 marketplace,**npx skills 完全是 pull 模型 — 用户主动跑 update 才会拉新版本**:

| 行为 | npx skills | Claude Code Plugin auto-update |
|---|---|---|
| 作者 push 新版本 → 用户被通知 | ❌ 无任何提示 | ✅ 启动时通知 |
| 装上的 skill 一直停在初始版本 | ✅ 是,直到手动 update | ❌ 自动滚动到最新 |
| 触发更新方式 | `npx skills update`(用户主动) | 启动时自动 |

应对策略:

```bash
# 定期手动一键全更
npx skills update -g -y

# 或用 cron 自动化(macOS 下可用 launchd)
0 9 * * 1 npx skills update -g -y >> ~/.skills-update.log 2>&1   # 每周一早 9 点
```

##### 6. 卸载排查:三种来源,三种删法

当 `npx skills rm` 报 "No matching skills found" 时,说明这个 skill 不是 npx 装的。先查路径类型:

```bash
ls -la ~/.claude/skills/<skill-name>
```

根据输出第一字符选删法:

| 第一字符 | 含义 | 删法 |
|---|---|---|
| `l` | symlink → npx skills 装的 | `npx skills rm -g <注册名>`(注册名可能与目录名不同,先 `npx skills list -g --json` 查) |
| `d` | 普通目录 → 手动 `git clone` / `cp` 装的 | `rm -rf ~/.claude/skills/<name>` |
| 不存在 | 已删除或是 plugin 带的 | 查 `ls ~/.claude/plugins/` 找对应 plugin → `/plugin uninstall <plugin-name>` |

> ⚠️ 注意:手动 `rm -rf ~/.claude/skills/foo` 只删了 symlink(如果是 npx 装的),真实文件还在 `~/.agents/skills/foo`,其他 Agent 仍能用。**完全卸载要用 `npx skills rm` 让它清理所有 symlink + 真实文件 + 注册表条目**。

#### 实例分析：双格式兼容的 Skill 仓库

一个 Skill 仓库可以同时支持 `npx skills`（通用）和 Claude Code Plugin（专用）两种安装方式。以 [pillarliang/python-code-style](https://github.com/pillarliang/python-code-style) 为例，它的仓库结构如下：

```
pillarliang/python-code-style/
├── .claude-plugin/                  # ← Claude Code Plugin 格式
│   ├── marketplace.json
│   └── plugin.json
├── skills/
│   └── python-code-style/           # ← agentskills.io 标准格式
│       ├── SKILL.md
│       └── references/
│           ├── docstrings.md
│           ├── naming-conventions.md
│           ├── language-rules.md
│           └── style-rules.md
└── docs/
    └── README.zh-cn.md
```

**两套目录各司其职：**


| 目录                          | 服务于                   | 安装命令                                                    |
| --------------------------- | --------------------- | ------------------------------------------------------- |
| `.claude-plugin/`           | Claude Code Plugin 系统 | `/plugin marketplace add pillarliang/python-code-style` |
| `skills/python-code-style/` | `npx skills` CLI（跨平台） | `npx skills add pillarliang/python-code-style`          |


**识别原理不同：**

- `npx skills add` 会递归扫描仓库中所有包含 `SKILL.md` 的子目录，找到 `skills/python-code-style/SKILL.md` 后识别为一个可安装的 Skill
- Claude Code `/plugin` 则读取 `.claude-plugin/plugin.json` 中的配置来注册插件

**双格式兼容的好处：**

- 使用 Claude Code 的用户可以通过 `/plugin` 获得完整的插件体验（自动更新、marketplace 管理等）
- 使用 Cursor、VS Code Copilot、Gemini CLI 等其他 Agent 的用户也能通过 `npx skills add` 安装同一个 Skill
- Skill 的核心内容（`SKILL.md` + `references/`）只维护一份，两种安装方式共享

> **给 Skill 作者的建议**：如果你的 Skill 希望跨平台使用，建议采用双格式结构。`skills/<name>/SKILL.md` 是通用入口，`.claude-plugin/` 是 Claude Code 的增强入口。两者可以指向同一份 Skill 内容。

#### 实例分析：Skill 集合仓库（一仓多 Skill）

除了上面的"单 Skill 仓库"，另一种常见打包模式是**一个仓库 = 多个同主题 Skill 的集合**，也叫 "Skill Pack"。以 [markdown-viewer/skills](https://github.com/markdown-viewer/skills) 为例,该仓库结构是:

```
markdown-viewer/skills/
├── README.md
├── archimate/        # SKILL.md + examples/ + stencils/
├── architecture/
├── bpmn/
├── canvas/
├── cloud/
├── data-analytics/
├── graphviz/
├── infocard/
├── infographic/
├── iot/
├── mindmap/
├── network/
├── security/
├── uml/
└── vega/
```

根目录下 15 个并列的 Skill 文件夹，每个都是独立的 Skill，各自包含自己的 `SKILL.md` 和可选资源（`stencils/` 图标库、`examples/` 示例等）。

**识别原理：** `npx skills add` 会递归扫描仓库中所有包含 `SKILL.md` 的子目录，把每个子目录识别为一个可安装的 Skill。所以对集合仓库来说，一条命令 = 发现 N 个 Skill，再由用户选择安装哪些。

**两种仓库布局对比：**


| 维度                | 单 Skill 仓库（如 `pillarliang/python-code-style`） | Skill 集合仓库（如 `markdown-viewer/skills`） |
| ----------------- | --------------------------------------------- | -------------------------------------- |
| 根下子目录             | 1 个 SKILL.md 路径                               | N 个并列的 SKILL.md 目录                     |
| 主题范围              | 聚焦一件事                                         | 围绕同一主题的系列工具（如各种图表）                     |
| 是否有 `skills/` 包装层 | 通常有（为了和 `.claude-plugin/` 并列）                 | 通常没有，直接把 Skill 铺在根目录                   |
| 默认安装行为            | 安装那 1 个                                       | 交互式挑选要装哪几个                             |


**对应的安装粒度：** 2.8 节的 `-s` / `--all` 参数正是为一仓多 Skill 的场景设计的：

```bash
# 交互式：跑命令后勾选安装
npx skills add markdown-viewer/skills

# 指定只装某几个
npx skills add markdown-viewer/skills -s uml mindmap

# 全部装上
npx skills add markdown-viewer/skills --all
```

**这种打包方式的好处：**

- **共享维护**：`stencils/`（图标库）、`examples/` 这类资源可跨 Skill 复用维护节奏
- **主题一致性**：同系列 Skill 遵循同一套风格约定
- **用户发现成本低**：找到一个仓库 = 拿到一整套能力
- **批量升级**：`npx skills update` 一次更新全部

本质上这是把 Skill 当成"npm 包里的命令"来组织——一个包可以导出多个命令，一个仓库也可以提供多个 Skill。


### 2.9 Claude Code Plugin 与 Marketplace 深度解析

前面 2.7 节提到 agentskills.io 规范定义了 Skill 的格式和路径，2.8 节介绍了三种安装方式。但 Claude Code 在 agentskills.io 规范之上额外构建了一套 **Plugin 与 Marketplace 系统**——这套系统**不属于 agentskills.io 规范**，是 Claude Code 私有的扩展机制，仅在 Claude Code 内可用。理解这两层对设计可分发的 Skill 仓库至关重要。

#### 一、整体架构与概念辨析

##### 1.1 三层架构总览

```
agentskills.io spec (规范层)         "一个能力怎么写"
        │
        ▼
Plugin (打包层 — Claude 私有)        "把多个能力捆成一个产品"
        │
        ▼
Marketplace (分发层 — Claude 私有)   "怎么让人找到、信任、自动更新"
```

每加一层,都解决了上一层留下的问题:

| 层级 | 解决的问题 | 适用场景 |
|---|---|---|
| **bare Skill** | 单一能力如何编写、加载、跨 Agent 兼容 | 个人零散 skill、跨平台分发 |
| **Plugin** | 多个相关能力(skill + MCP + hook + command)如何捆绑发布 | 完整工具套件,例如 "Python 开发包" |
| **Marketplace** | 多个 plugin 如何集中发现、版本管理、企业管控 | 团队/公司规模化分发 |

##### 1.2 Skill vs Plugin:最常见的混淆

**Plugin 和 Skill 不在同一层级**:

- **Plugin** 是**分发单元(包)** — `/plugin install` 的对象
- **Skill** 是**一种能力类型** — Plugin 内可包含的多种 content 之一

类比应用商店:

```
Marketplace (应用商店)
  └─ Plugin (APP,安装单元)
       ├─ Skills        ← 模型按 description 匹配后自动加载
       ├─ Commands      ← 用户主动输入的 / 斜杠命令
       ├─ Agents        ← 子 agent 定义
       ├─ Hooks         ← 工具事件钩子
       └─ MCP Servers   ← 外部工具集成
```

核心差异:

| 维度 | Plugin | Skill |
|---|---|---|
| **角色** | 容器/包,`/plugin install` 的对象 | 一种能力类型,plugin 内的 content |
| **安装粒度** | 整体安装/禁用 | 不能单独装,随所在 plugin 一起 |
| **加载时机** | 安装后常驻可用 | metadata 常驻,匹配后才注入 SKILL.md |
| **声明文件** | `.claude-plugin/plugin.json` | 各 skill 目录下的 `SKILL.md` |
| **跨 Agent** | Claude Code 私有 | agentskills.io 规范,30+ Agent 通用 |

> ⚠️ **Plugin 不一定包含 Skill**。一个 plugin 可以只装 skills、只装 commands、只装 hooks,或任意组合(如 [pillarliang/python-code-style](https://github.com/pillarliang/python-code-style) 就只有 skills)。把 Plugin 简单理解成 "Skill 的集合" 是常见误解。

##### 1.3 心智模型:类比 npm 体系

把整套体系类比 npm,概念关系立刻清晰:

| Claude 概念 | npm 类比 |
|---|---|
| `plugin.json` | `package.json`(包自身元数据,作者控制) |
| `marketplace.json` | npm registry index(列出多个包及其下载位置) |
| `source` 字段 | `"dist": { "tarball": "..." }`(去哪下载) |
| `category` / `tags` | npm 搜索分类(展示用,不影响包行为) |
| 自动发现机制 | npm `bin/` `man/` 约定路径(无需在 package.json 列出) |
| `strict` 模式 | 类似 lockfile 优先级:谁是 source of truth |

**护照 vs 时刻表的比喻:** plugin.json 是包的"护照"(写"我是谁、有什么能力"),marketplace.json 是机场的"航班时刻表"(写"这趟航班从哪起飞、属于哪家航司")。两者各管各的,信息少有重叠。

#### 二、Plugin:能力打包层

##### 2.1 为什么需要 Plugin 打包

一个具体例子:完整的 "Python 开发套件" 通常需要:

- lint skill + format skill(2 个 SKILL.md)
- Python LSP 服务(实时诊断)
- import-helper MCP(自动补全 import)
- 保存时校验的 hook

**这些必须捆绑发布、版本一致**,否则用户装一半就坏。bare Skill 的扁平模型(一个仓库一个 Skill)做不到这种打包。**Plugin 的本质就是给 Agent 能力提供 npm/pip 那样的"包"语义**。

##### 2.2 Plugin 能装哪些组件

| 组件 | 默认路径 | 作用 |
|---|---|---|
| **Skills** | `skills/<name>/SKILL.md` | 模型自主调用(等同 agentskills.io 规范) |
| **Subagents** | `agents/<name>.md` | 自定义子 agent(系统提示 + 工具限制) |
| **Hooks** | `hooks/hooks.json` | 事件钩子(PostToolUse、SessionStart 等) |
| **MCP servers** | `.mcp.json` | 外部工具集成(GitHub、Notion 等) |
| **LSP servers** | `.lsp.json` | 语言服务器(实时诊断) |
| **Slash commands** | `commands/` | 用户主动调用的命令 |
| **Default settings** | `settings.json` | 预配置的环境变量等 |
| **Executables** | `bin/` | 加入 PATH 的可执行脚本 |
| **Monitors** | `monitors/monitors.json` | 后台守护进程(日志监听等) |

##### 2.3 自动发现机制:组件不需要在 plugin.json 中声明

**Plugin 系统最容易被忽略的设计**:Claude Code 按**约定路径**自动扫描组件,**不需要**在 `plugin.json` 里显式列出"我有哪些 skill / hook / MCP"。

```
my-plugin/
├── .claude-plugin/plugin.json     # 元数据
├── skills/<name>/SKILL.md         # ← 自动发现
├── agents/<name>.md               # ← 自动发现
├── commands/<name>.md             # ← 自动发现
├── hooks/hooks.json               # ← 自动发现
└── .mcp.json                      # ← 自动发现
```

只有当组件**不在默认路径**时(如想把 skills 放在 `extras/skills/`),才需要在 `plugin.json` 里写 `"skills": "./extras/skills/"`。

这意味着**最小可用的 plugin.json 只需一行**:

```json
{ "name": "my-plugin" }
```

##### 2.4 plugin.json 字段完整参考

放在 `<plugin>/.claude-plugin/plugin.json`。**唯一必填字段是 `name`**,其他全可选。按用途分三类:

**A. 元数据类**(描述 plugin 是什么)

| 字段 | 类型 | 作用 | 示例 |
|---|---|---|---|
| `name` ✅ | string | plugin 唯一 ID,kebab-case。会作为 skill 命名空间 `/name:skill` | `"code-formatter"` |
| `description` | string | UI 中展示的简短说明 | `"Automatic code formatting"` |
| `version` | string | 语义化版本。**写了** → bump 才更新;**省略** → 用 git commit SHA | `"2.1.0"` |
| `$schema` | string | 编辑器校验用的 JSON Schema URL,运行时忽略 | `"https://..."` |
| `author` | object | `{ "name": "...", "email": "...", "url": "..." }` | `{ "name": "Dev Team" }` |
| `homepage` | string | 文档 URL | `"https://docs.example.com"` |
| `repository` | string | 源码仓库 URL | `"https://github.com/..."` |
| `license` | string | SPDX 许可证标识 | `"MIT"`、`"Apache-2.0"` |
| `keywords` | array | 发现/搜索标签 | `["formatting", "lint"]` |

**B. 组件路径覆盖类**(可选,只在不用默认路径时才需要)

| 字段 | 类型 | 默认路径 |
|---|---|---|
| `skills` | string \| array | `skills/` |
| `commands` | string \| array | `commands/` |
| `agents` | string \| array | `agents/` |
| `hooks` | string \| object | `hooks/hooks.json` |
| `mcpServers` | string \| object | `.mcp.json` |
| `lspServers` | string \| object | `.lsp.json` |
| `monitors` | string \| array | `monitors/monitors.json` |
| `outputStyles` | string \| array | `output-styles/` |
| `themes` | string \| array | `themes/` |

**C. 高级类**

| 字段 | 类型 | 作用 |
|---|---|---|
| `userConfig` | object | 启用 plugin 时向用户询问的配置项(API key、偏好等),支持 `sensitive`、`required` 等 |
| `dependencies` | array | 依赖的其他 plugin,支持 semver 范围 |
| `channels` | array | 消息通道声明(每个 channel 绑定一个 MCP server) |

#### 三、Marketplace:分发与管控层

##### 3.1 Marketplace 独有能力

Plugin 解决了"装一个东西包含多个组件",Marketplace 解决了"**一组 plugin 如何集中发布、发现、信任、批量管理**":

| 能力 | 价值 |
|---|---|
| **集中目录** | 一个 marketplace 可列多个 plugin,用户 `/plugin` 一处浏览 |
| **多种 source** | 同一 marketplace 可混用 GitHub / GitLab / npm / git URL / 本地路径 / monorepo 子目录 |
| **版本控制** | `version` 字段控制更新节奏 |
| **Release channels** | 同一份代码可做 `stable-` / `latest-` 两个 marketplace 指向不同 git ref |
| **团队分发** | `.claude/settings.json` 中 `extraKnownMarketplaces`,团队成员自动收到提示 |
| **企业管控** | `strictKnownMarketplaces` 锁定允许的 marketplace 白名单 |

> 这些能力的根本来源是 marketplace.json 作为**中间注册表**的存在。bare Skill 的分发是用户直连 GitHub repo,没有中间层,做不了版本指向、白名单、灰度发布这些事。

##### 3.2 marketplace.json 顶层字段

放在 `<marketplace-repo>/.claude-plugin/marketplace.json`。**必填字段:`name`、`owner`、`plugins`**。

| 字段                                    | 类型       | 必填 | 作用                                                                             |
| ------------------------------------- | -------- | -- | ------------------------------------------------------------------------------ |
| `name`                                | string   | ✅  | marketplace 标识,公开可见(`/plugin install xxx@<name>`)。**保留名**:不能用 `claude-code-plugins`、`anthropic-plugins`、`agent-skills` 等 |
| `owner`                               | object   | ✅  | `{ "name": "...", "email": "..." }`,只 name 必填                                   |
| `plugins`                             | array    | ✅  | plugin 条目数组                                                                    |
| `description`                         | string   |    | marketplace 描述                                                                 |
| `version`                             | string   |    | manifest 版本(信息性,不影响行为)                                                       |
| `$schema`                             | string   |    | JSON Schema URL,运行时忽略                                                          |
| `metadata.pluginRoot`                 | string   |    | 公共前缀,简化 source 路径写法。设了之后可写 `"source": "foo"` 而不是 `"source": "./plugins/foo"`   |
| `allowCrossMarketplaceDependenciesOn` | array    |    | 允许 plugin 依赖其他哪些 marketplace 的 plugin                                          |


##### 3.3 plugins[] 条目字段

**必填只有 `name` 和 `source`**。其他字段全可选;**没填的会从 plugin.json 继承**。

| 字段            | 类型              | 必填 | 作用                                                                       |
| ------------- | --------------- | -- | ------------------------------------------------------------------------ |
| `name`        | string          | ✅  | plugin ID(对应 plugin.json 的 name)                                         |
| `source`      | string\|object  | ✅  | **去哪取 plugin**(本地路径 / GitHub / git URL / monorepo / npm)                  |
| `description` | string          |    | 不写则继承 plugin.json 的                                                      |
| `version`     | string          |    | 不写则继承(冲突时 plugin.json 优先,见下文 strict 模式)                                  |
| `category`    | string          |    | 分类标签,用于 UI 浏览(如 `"productivity"`、`"code-quality"`)                         |
| `tags`        | array           |    | 搜索标签                                                                     |
| `author`      | object          |    | 同 plugin.json 的 author                                                   |
| `homepage` / `repository` / `license` / `keywords` | — |    | 同 plugin.json,不写则继承                                                      |
| `strict`      | boolean         |    | **冲突仲裁模式**(默认 `true`)。详见下文                                                |


##### 3.4 source 字段:5 种 plugin 来源

```json
// 1. 本地相对路径(必须基于 git 仓库)
{ "name": "foo", "source": "./plugins/foo" }

// 2. GitHub 仓库
{
  "name": "foo",
  "source": {
    "source": "github",
    "repo": "owner/repo",
    "ref": "v1.0.0",                          // 可选,分支或 tag
    "sha": "a1b2c3d4..."                      // 可选,40 字符 commit SHA(覆盖 ref)
  }
}

// 3. 任意 git URL(GitLab、自建)
{
  "name": "foo",
  "source": {
    "source": "url",
    "url": "https://gitlab.com/team/foo.git",
    "ref": "main"
  }
}

// 4. monorepo 子目录(稀疏克隆)
{
  "name": "foo",
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/owner/monorepo.git",
    "path": "tools/foo",
    "ref": "main"
  }
}

// 5. npm 包
{
  "name": "foo",
  "source": {
    "source": "npm",
    "package": "@org/foo",
    "version": "^2.0.0",
    "registry": "https://npm.example.com"      // 可选,自建 registry
  }
}
```

##### 3.5 strict 模式:字段冲突时谁赢

当 plugin.json 和 marketplace.json 的 plugins[] 条目里**同一字段填了不同值**时,由 `strict` 字段决定谁是权威:

| `strict` 值 | 行为 |
|---|---|
| `true`(默认) | **plugin.json 为权威**,marketplace 条目只起补充作用。两边都写时取 plugin.json 的值 |
| `false` | **marketplace 条目为权威**,plugin.json 中冲突字段会报错。适用于由 marketplace 维护的远程 plugin |

**实践建议:** 永远使用默认 `strict: true`,保持单一数据源 — 信息只在 plugin.json 写一次,marketplace 条目只填 `name` + `source` + 必要展示字段(`category`、`tags`)。

#### 四、两个文件如何配合(实战)

##### 4.1 完整实例:code-formatter

下面用一个虚构的 `code-formatter` plugin(2 个 skill + 1 个 hook + 1 个 MCP server)展示双文件的实际协作。

**仓库结构:**

```
code-tools/                            # ← marketplace 仓库
├── .claude-plugin/
│   └── marketplace.json               # ← 货架入口
└── plugins/
    └── code-formatter/                # ← 一个 plugin
        ├── .claude-plugin/
        │   └── plugin.json            # ← plugin 身份证
        ├── skills/
        │   ├── format-python/SKILL.md
        │   └── format-js/SKILL.md
        ├── hooks/hooks.json
        └── .mcp.json
```

**marketplace.json**(只关心"货架信息"):

```json
{
  "name": "code-tools",
  "owner": { "name": "DevTools Team" },
  "description": "Internal code quality plugins",
  "plugins": [
    {
      "name": "code-formatter",
      "source": "./plugins/code-formatter",
      "category": "code-quality",
      "tags": ["formatting", "lint"]
    }
  ]
}
```

**plugin.json**(只关心"我是谁"):

```json
{
  "name": "code-formatter",
  "description": "Automatic code formatting",
  "version": "2.1.0",
  "author": { "name": "DevTools Team" },
  "license": "MIT",
  "userConfig": {
    "indent_style": {
      "type": "string",
      "title": "Indent Style",
      "default": "space"
    }
  }
}
```

**用户操作流程:** `/plugin marketplace add owner/code-tools` → `/plugin` 浏览 → `/plugin install code-formatter@code-tools`

##### 4.2 字段归属原则:谁写在哪

| 信息 | 写在哪里 | 原因 |
|---|---|---|
| `name` | 两边都写 | marketplace 用它匹配对应的 plugin |
| `description` | 只在 plugin.json | marketplace 不写则继承 |
| `version` | 只在 plugin.json | plugin 自己最了解版本号 |
| `author` / `license` | 只在 plugin.json | plugin 元数据 |
| `category` / `tags` | 只在 marketplace.json | "货架展示信息",plugin 自身不关心 |
| `source` | 只在 marketplace.json | plugin 不需要知道"自己被怎么取" |
| `userConfig` / `dependencies` | 只在 plugin.json | plugin 内部行为定义 |

##### 4.3 version 字段:权威来源与三种处理方式

`version` 是字段归属的特殊情况,值得单独讲清楚。**plugin.json 是 version 的权威来源**,marketplace 条目中的 version 仅作可选补充。三种处理方式对应三种发版风格:

| 方式 | 配置 | 行为 | 适用场景 |
|---|---|---|---|
| **1. 显式版本(推荐)** | plugin.json 写 `"version": "2.1.0"` | 用户只在 bump 版本号时收到更新 | 正常发版,语义化版本管理 |
| **2. 隐式版本(SHA)** | plugin.json 省略 version | 每个 commit 都视为新版本 | 早期高频迭代(注意:容易把半成品推给用户) |
| **3. 两边都写(strict:true)** | plugin.json 与 marketplace 条目都写 | plugin.json 胜出,marketplace 的被忽略 | 一般避免,保持单一来源 |

**实践建议:**

- 永远只在 plugin.json 写 version,marketplace 条目里**不要重复**
- 选择显式版本而非省略 — 即便个人项目,语义化版本能让用户更可控
- 小改动不 bump → push 默认分支不会推给用户;真要发版时再改 version
- 破坏性变更 bump major(`2.x.x → 3.0.0`),用户从版本号能感知风险

#### 五、Auto-update 机制

> 版本策略已在 4.3 节讲过(显式版本 vs SHA),本节聚焦更新行为本身:何时触发、用户如何感知、有哪些开关。

##### 5.1 更新触发流程

```
作者 push 新版本（bump version 或 commit SHA 变更）
        │
        ▼
用户下次启动 Claude Code
        │
        ▼
启动时自动 refresh marketplace 数据 ──→ 无任何询问，直接下载新版本
        │
        ▼
检测到更新后，显示通知：
"Plugins updated. Run /reload-plugins to apply."
        │
        ▼
用户必须主动执行 /reload-plugins（或重启）──→ 新版本才真正生效
```

##### 5.2 UX 关键特点

| 环节         | 是否需要用户操作                            |
| ---------- | ----------------------------------- |
| **检测更新**   | ❌ 启动时自动                              |
| **下载新版本**  | ❌ 自动，不询问                            |
| **加载生效**   | ✅ 必须 `/reload-plugins` 或重启          |
| **权限变更**   | ❌ 不重新弹 permission 提示，新权限随更新一起生效     |


三个值得注意的细节：

1. **权限变更没有二次确认** — 即便插件改了 `allowed-tools`，用户也不会被重新询问。作者改权限要慎重。
2. **代码改动 / 权限改动 / version bump 三者 UX 完全一致** — 都是"自动下载 → 提示 reload"。
3. **会话中不会被打断** — 更新检查只在启动时跑，正在跑的会话不受影响。

##### 5.3 配置开关

| 操作                                       | 命令/路径                                                                            |
| ---------------------------------------- | -------------------------------------------------------------------------------- |
| 启用某个 marketplace 的 auto-update           | `/plugin` → Marketplaces 标签页 → 选中 → Enable auto-update                            |
| 默认行为                                     | Anthropic 官方 marketplace 默认开启，第三方默认关闭                                            |
| 全局禁用 auto-update                          | 环境变量 `DISABLE_AUTOUPDATER=1`                                                     |
| 仅禁用 Claude Code 自更新但保留 plugin 更新         | `FORCE_AUTOUPDATE_PLUGINS=1`                                                     |


#### 六、高级特性:多通道与企业管控

##### 6.1 Release Channels:同源不同通道

**问题场景：** 一个 plugin 想让 QA 团队先用最新版本，正式用户用稳定版本。

**实现方式：** 建立**两个 marketplace.json**，各自指向同一仓库的不同 git ref：

```json
// stable-marketplace.json — 指向 v1.0.0 标签
{
  "name": "my-tools-stable",
  "plugins": [{
    "name": "my-plugin",
    "source": {
      "source": "github",
      "repo": "me/my-plugin",
      "ref": "v1.0.0"           // 锁定稳定版本
    }
  }]
}

// latest-marketplace.json — 指向 main 分支
{
  "name": "my-tools-latest",
  "plugins": [{
    "name": "my-plugin",
    "source": {
      "source": "github",
      "repo": "me/my-plugin",
      "ref": "main"             // 始终拿最新
    }
  }]
}
```

通过企业管理设置,把不同 marketplace 分配给不同用户组,即实现灰度发布。

##### 6.2 strictKnownMarketplaces:企业来源白名单

**问题场景：** 公司不希望员工从外部随便装未审核的 plugin。

**配置方式：** 在公司统一下发的 `settings.json` 中：

```json
{
  "strictKnownMarketplaces": true,
  "extraKnownMarketplaces": {
    "company-internal": {
      "source": {
        "source": "github",
        "repo": "mycompany/internal-marketplace"
      }
    },
    "anthropic-official": {
      "source": {
        "source": "github",
        "repo": "anthropics/claude-plugins"
      }
    }
  }
}
```

效果：

- ✅ 员工可装白名单内的 plugin
- ❌ 员工尝试 `/plugin marketplace add 任意外部repo` → 被拒绝
- ❌ 已装的非白名单 plugin → 加载时被禁用

类比 Chrome 企业版的扩展白名单,或公司私有 npm registry 的限制。

#### 七、选型与边界

##### 7.1 场景到方案的决策表


| 场景                                         | 推荐方案                                              |
| ------------------------------------------ | ------------------------------------------------- |
| 个人零散 skill，偶尔分享                            | bare Skill + `npx skills`                         |
| 几个 skill 围绕同一主题，需要语义化版本                    | Plugin                                            |
| 要捆绑 skill + MCP + hook + 命令成一套工具           | **必须 Plugin**（bare Skill 无打包能力）                   |
| 多个相关 plugin，需要集中发现                         | Marketplace                                       |
| 公司内部分发，要管控来源/版本                            | **必须 Marketplace**（企业控制是其独有能力）                    |
| 要发布到非 Claude Code 用户（Cursor、VS Code 等）     | **必须 bare Skill 格式**（Plugin/Marketplace 跨 Agent 不通用） |


**双格式兼容仓库的实践:** 如果希望同一份 Skill 既能在 Claude Code 享受 marketplace 体验,又能被其他 Agent 用 `npx skills` 安装,采用双格式结构(详见 2.8 节"实例分析:双格式兼容的 Skill 仓库"):

- `skills/<name>/SKILL.md` 是通用入口(agentskills.io 规范)
- `.claude-plugin/` 是 Claude Code 的增强入口
- 两者指向同一份 Skill 内容,只维护一份

##### 7.2 与 agentskills.io 规范的边界划分

明确哪些属于跨 Agent 通用规范，哪些是 Claude Code 私有扩展：


| 类别                                    | agentskills.io 规范 | Claude Code 私有 |
| ------------------------------------- | ----------------- | -------------- |
| **SKILL.md 格式**                       | ✅                 | —              |
| **Frontmatter 字段**                    | ✅                 | —              |
| **`references/` `assets/` `scripts/`** | ✅                 | —              |
| **渐进式披露（L1/L2/L3）**                  | ✅                 | —              |
| **存储路径约定**（`.agents/skills/` 等）        | ✅                 | —              |
| **`plugin.json`**                     | —                 | ✅              |
| **`marketplace.json`**                | —                 | ✅              |
| **`/plugin` 命令族**                     | —                 | ✅              |
| **Auto-update 机制**                    | —                 | ✅              |
| **Release channels**                  | —                 | ✅              |
| **`strictKnownMarketplaces`**         | —                 | ✅              |
| **MCP / Hook / LSP 集成**               | —                 | ✅              |


> **对作者的启示：** SKILL.md 内容遵守 agentskills.io 规范确保跨平台兼容，但若希望在 Claude Code 中获得自动更新、捆绑分发、企业管控等高级特性，需要额外加 `.claude-plugin/` 目录。两者不冲突，可在同一仓库共存。

---

## 三、五大设计模式详解

文章的核心内容。这五种模式解决的是同一个问题：**SKILL.md 内部的内容该怎么组织？** Agent Skills 规范定义了文件格式，但没有规定内容的结构化方式，这五种模式就是对这个空白的填补。

### 模式一：Tool Wrapper（工具包装器）— 教 Agent 学会一个库

#### 核心思想

将某个库、框架或内部工具的**最佳实践和约定**打包成一个 Skill，Agent 只在需要时加载这些专业知识。

#### 为什么需要它

LLM 的训练数据中包含大量库的用法，但不一定包含**你团队的约定**。比如 LLM 知道 FastAPI 怎么写，但不知道你们团队要求所有端点必须用 Pydantic v2 的 `model_validator`、必须返回标准化的错误格式。Tool Wrapper 就是把这些"团队规范"注入到 Agent 的决策中。

#### 目录结构

```
fastapi-skill/
├── SKILL.md              # 指令：引用 conventions.md
└── references/
    └── conventions.md    # 详细的最佳实践文档
```

**特点：最简单的模式**，不需要 `assets/` 或 `scripts/`，只有指令 + 参考文档。

#### 适用场景

- 团队的 FastAPI / Django / Terraform 编码规范
- 安全策略（如内部认证库的使用方式）
- 内部工具的 API 使用约定

#### 真实案例

- **Vercel** 的 `react-best-practices` Skill
- **Supabase** 的 `postgres optimization guidelines`
- **Google** 的 `gemini-api-dev` Skill

---

### 模式二：Generator（生成器）— 生成结构化输出

#### 核心思想

用**模板**定义输出结构，用**风格指南**定义质量标准，用**指令**编排填充过程。三者分离，互不耦合。

#### 为什么需要它

当你需要 Agent 反复生成格式一致的文档（技术报告、API 文档、commit message）时，如果每次都在 prompt 里写格式要求，既冗余又容易遗漏。Generator 模式把"格式"和"质量标准"分别抽取到独立文件中。

#### 目录结构

```
report-generator/
├── SKILL.md                    # 编排指令：读取模板 → 按风格指南填充
├── assets/
│   └── report-template.md     # 输出模板（定义必需章节）
└── references/
    └── style-guide.md         # 质量标准（语言风格、格式要求）
```

#### 关键设计洞察

> "Swap either file to change the output without touching the instructions."
> （替换任一文件即可改变输出，无需修改指令。）

这意味着：

- 换一个 `report-template.md`，可以从"技术报告"切换到"API 文档"
- 换一个 `style-guide.md`，可以从"正式风格"切换到"对话风格"
- 指令本身保持稳定不变

#### 适用场景

- 包含 Executive Summary、Methodology、Findings 等固定章节的技术报告
- 标准化端点描述的 API 文档
- Conventional Commits 格式化
- ADK Agent 项目脚手架生成

---

### 模式三：Reviewer（评审器）— 按标准评估

#### 核心思想

将"**检查什么**"（checklist）和"**怎么检查**"（review protocol）分离。Checklist 是一个按类别和严重程度组织的具体检查项清单，Agent 逐项检查后输出分级报告。

#### 为什么需要它

让 Agent "review 一下代码" 的结果取决于模型的预训练知识，不可控且不可复现。Reviewer 模式通过**外部化检查清单**，使评审行为**可预测、可定制、可审计**。

#### 目录结构

```
code-reviewer/
├── SKILL.md                       # 评审协议（步骤、输出格式）
└── references/
    └── review-checklist.md       # 按类别组织的检查项
```

#### 严重程度分级


| 级别          | 含义   | 示例                 |
| ----------- | ---- | ------------------ |
| **error**   | 必须修复 | 使用了可变默认参数、裸 except |
| **warning** | 应该修复 | 命名不符合规范            |
| **info**    | 建议考虑 | 可以进一步优化的地方         |


#### Checklist 的编写要求

检查项必须**具体且可检查**，例如：

- "No mutable default arguments"（不使用可变默认参数）
- "Functions under 30 lines"（函数不超过 30 行）
- 而不是"代码质量要好"这种模糊描述

#### 实际效果

Giorgio Crivellari 的实践报告：使用 Reviewer Skill 对照治理标准检查代码，**代码质量从 29% 提升到 99%**。

#### 适用场景

- 团队代码风格规范审查
- OWASP Top 10 安全审计
- 内容编辑审查（对照文风指南）
- ADK Agent 命名与结构验证

---

### 模式四：Inversion（反转）— Skill 来面试你

#### 核心思想

翻转对话方向：不是用户提需求、Agent 直接输出，而是 **Skill 先向用户提出结构化的问题**，收集足够信息后再生成输出。

#### 为什么需要它

Agent 最常见的问题之一是**基于假设行动**。用户说"帮我设计一个 API"，Agent 立刻开始写代码 — 但它不知道你的认证方式、数据库选型、性能要求。Inversion 模式通过**显式门控**（explicit gates）强制 Agent 在收集完信息前不得行动。

#### 阶段结构示例

```
Phase 1: 问题发现（Problem Discovery）
  Q1: 你要解决的核心问题是什么？
  Q2: 目标用户是谁？
  Q3: 成功标准是什么？

Phase 2: 技术约束（Technical Constraints）
  Q4: 使用什么语言和框架？
  Q5: 有哪些外部依赖？
  Q6: 性能和扩展要求？

Phase 3: 综合（Synthesis）
  → 用前两阶段的答案填充输出模板
```

#### 关键设计要素：显式门控

在 SKILL.md 中必须写明：

```
DO NOT start building until all phases complete.
```

没有这个门控，Agent 往往会在收到第一阶段的回答后就急于开始生成，跳过后续问题。门控是**纯指令层面的控制**，不依赖任何框架 feature。

#### 适用场景

- 技术设计前的需求收集
- 故障排查前的诊断问询
- 部署偏好的配置向导
- ADK Agent 设计前的架构摸底

---

### 模式五：Pipeline（流水线）— 强制多步骤工作流

#### 核心思想

定义**顺序执行的多个步骤**，步骤之间用**门控条件**（gates）连接，确保上一步完成并验证后才进入下一步。

#### 为什么需要它

有些任务本质上是多步骤的，且步骤之间有依赖关系。例如生成文档：必须先解析代码、再生成注释、再组装文档、最后质量检查。如果 Agent 跳过中间步骤（比如未经用户确认就直接生成最终文档），结果往往不可用。Pipeline 通过门控强制 Agent **逐步推进**。

#### 目录结构

```
doc-pipeline/
├── SKILL.md                       # 定义步骤和门控
├── references/
│   ├── style-guide.md            # 风格指南
│   └── quality-checklist.md      # 质量检查清单
├── assets/
│   └── doc-template.md           # 输出模板
└── scripts/                       # 未来支持的可执行脚本
```

**特点：最复杂的模式**，使用了所有三个可选目录。

#### 四步流水线示例（文档生成）

```
Step 1: Parse & Inventory
  → 提取所有公共 API 元素（类、函数、参数）

Step 2: Generate Docstrings
  → 为每个元素生成文档字符串
  → GATE: "Do NOT proceed to Step 3 until the user confirms"
  （用户确认后才继续）

Step 3: Assemble Documentation
  → 按模板组装最终文档

Step 4: Quality Check
  → 对照 quality-checklist.md 验证完整性
```

#### 门控机制（Gate）

门控是 Pipeline 模式的灵魂。**门控就是 SKILL.md 中的一段强约束 prompt**，没有任何框架层面的强制执行——纯粹靠指令措辞的强硬程度约束 LLM 行为。

门控的核心含义是**"不满足条件就不许往下走"**，但条件不局限于用户介入：


| 门控类型      | 谁来判断     | 是否需要用户介入 | 示例                                                              |
| --------- | -------- | -------- | --------------------------------------------------------------- |
| **用户确认**  | 用户       | 是        | `DO NOT proceed until the user confirms`                        |
| **质量自检**  | Agent 自己 | 否        | `DO NOT proceed unless all checklist items pass`                |
| **前置条件**  | Agent 自己 | 否        | `DO NOT proceed if the file does not exist`                     |
| **信息完整性** | Agent 自己 | 否        | `DO NOT start building until all phases complete`（Inversion 模式） |


在 Pipeline 模式中**用户确认门控**最常见（多步骤流程中人工审核点特别重要），在 Reviewer 或 Inversion 模式中门控更多是 Agent 的自我约束。

**写法建议：** 使用全大写、祈使句、否定式来提高遵从率：

- `DO NOT proceed until...`（推荐，强约束）
- `Please wait for confirmation`（太弱，容易被忽略）

**局限性：** 既然只是 prompt，就不能 100% 保证 Agent 一定遵守。模型可能偶尔跳过门控，尤其是指令措辞不够强硬、上下文太长导致指令被"遗忘"、或模型倾向于一次性完成任务时。

门控的作用：

- 人工审核点被强制插入流程
- Agent 不会跳过验证阶段
- 每一步的输出质量得到保障

#### 适用场景

- 带审批门控的文档生成
- 带中间验证的数据处理管道
- 带人工确认检查点的部署工作流
- 任何跳过步骤会导致错误输出的多步任务

---

## 四、模式组合 — 实际生产中的用法

五种模式**并非互斥**，实际生产系统通常**组合 2–3 个模式**：


| 组合方式                    | 说明                                  |
| ----------------------- | ----------------------------------- |
| Pipeline + Reviewer     | 流水线的某个步骤是一个 Reviewer 检查             |
| Generator + Inversion   | 先用 Inversion 收集需求，再用 Generator 填充模板 |
| Pipeline + Tool Wrapper | Tool Wrapper 作为 Pipeline 某个步骤的参考知识  |


---

## 五、决策框架 — 如何选择合适的模式

```
你需要什么？                         → 选择
─────────────────────────────────────────────
固定结构的输出？                      → Generator
评估代码/内容质量？                   → Reviewer
教 Agent 使用某个库/工具的规范？       → Tool Wrapper
先收集信息再行动？                    → Inversion
多步骤 + 步骤间有门控？               → Pipeline
```

**入门建议：** 从 **Tool Wrapper** 开始 — 它是最简单、最广泛采用的模式。当需要结构化输出时升级到 Generator，需要评估时升级到 Reviewer。

---

## 六、ADK Skills 生态系统

### 社区市场


| 平台                             | 特点                                |
| ------------------------------ | --------------------------------- |
| [skills.sh](https://skills.sh) | 86,000+ 总安装量                      |
| Vercel Labs                    | React/Next.js skills, 22K stars   |
| Anthropic Skills               | Document generation, 86,500 stars |
| Supabase                       | Postgres 优化指南                     |


### Google 官方 ADK Core Skills

Google 提供 6 个官方 Skill，帮助开发者编写 ADK 代码（全部采用 Tool Wrapper 模式）：

1. `adk-dev-guide` — 开发指南
2. `adk-cheatsheet` — 速查表
3. `adk-eval-guide` — 评估指南
4. `adk-deploy-guide` — 部署指南
5. `adk-observability-guide` — 可观测性指南
6. `adk-scaffold` — 脚手架

安装命令：

```bash
npx skills add google/adk-docs -y -g
```

---

## 七、关键要点总结

1. **Skill ≠ Tool**：Tool 是执行能力，Skill 是编排智慧
2. **五种模式解决结构化问题**：Tool Wrapper（知识注入）、Generator（模板输出）、Reviewer（标准评估）、Inversion（信息收集）、Pipeline（多步编排）
3. **Description 字段决定触发率**：用具体关键词，不用泛泛描述
4. **门控是关键设计元素**：显式的 `DO NOT proceed until...` 指令防止 Agent 跳步
5. **模板与规则分离**：Generator 和 Reviewer 的核心设计原则是将"结构"和"标准"放在独立文件中
6. **渐进式加载节省 token**：L1/L2/L3 三级加载，按需消耗上下文窗口
7. **跨平台可移植**：遵循 agentskills.io 规范的 Skill 可在 30+ Agent 中使用
8. **从简单开始**：Tool Wrapper → Generator/Reviewer → Inversion/Pipeline，逐步升级复杂度

---

## 附录 A：实战案例 — 周报生成 Skill

这个案例来自 Notion 笔记，综合运用了 **Generator 模式**（模板 + 指令分离）。它利用 Bash 工具抓取 Git 提交记录，结合模板生成结构化周报。

### 目录结构

```
weekly-report/
├── SKILL.md
└── assets/
    └── template.md       # 周报输出模板
```

### assets/template.md

```markdown
# 周报 - {{Date}}

## 本周工作摘要
{{Summary}}

## 主要进展 (Key Achievements)
- **功能开发**:
  - ...
- **问题修复**:
  - ...
- **代码重构/优化**:
  - ...

## 详细提交记录 (Git Log Summary)
{{GitDetails}}

## 下周计划
- [ ] ...
```

### SKILL.md

```markdown
---
name: weekly-report
description: 自动生成周报。分析项目 Git 提交历史，当用户需要总结本周工作、生成周报或汇报进度时使用。
allowed-tools: Bash(git) Read Write
---

## 角色定义
你是一个技术项目经理和文档专家。你的任务是分析项目的 Git 提交历史，生成一份专业、结构清晰的周报。

## 执行步骤

### 第一步：收集上下文
1. 获取今天的日期。
2. 使用 Bash 运行：
   `git log --since="1 week ago" --pretty=format:"%ad - %s (%an)" --date=short`
3. 使用 Read 读取模板文件 `{baseDir}/assets/template.md`。

### 第二步：分析与生成
1. 将提交分类为"新功能"、"Bug 修复"、"文档/重构"等。
2. 过滤微小提交（如 "typo" 或 "wip"）。
3. 基于模板结构，用自然语言总结本周工作。

### 第三步：输出与保存
1. 展示给用户确认。
2. 使用 Write 保存为 `weekly-report_YYYY-MM-DD.md`。

## 风格要求
- 语言简洁、专业。
- 重点突出成果（Outcome），而不仅仅是列出动作（Activity）。
```

### 案例要点分析


| 要素                  | 说明                                            |
| ------------------- | --------------------------------------------- |
| **设计模式**            | Generator（模板 + 指令分离）                          |
| `**{baseDir}` 变量**  | 引用当前 Skill 目录路径，避免写死绝对路径                      |
| `**allowed-tools`** | `Bash(git)` 仅授权 git 命令，最小权限原则                 |
| **触发方式**            | 用户说"帮我写周报"→ Agent 匹配 description 中的关键词 → 自动激活 |


---

## 附录 B：Skill 验证与测试

### 格式验证

使用官方 [skills-ref](https://github.com/agentskills/agentskills/tree/main/skills-ref) 库验证 Skill 格式：

```bash
skills-ref validate ./my-skill
```

### 效果测试

在 `evals/evals.json` 中创建测试用例，分别在有/无 Skill 的情况下运行，测量 pass rate 差值来评估 Skill 的效果。