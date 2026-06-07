# 10 - Prompt Templates：提示模板

> 来源：https://pi.dev/docs/latest/prompt-templates

## 1. 定位

Prompt template 是 markdown 文件——用 slash 命令触发后展开成 prompt 发给 agent。

**文件名（去掉 `.md`）就是命令名**：`review.md` → `/review`。

跟 [[09-Skills-按需技能|skill]] 的区别：

| 维度 | Prompt Template | Skill |
|------|-----------------|-------|
| 内容 | 直接展开成 prompt | 提供给 agent 的工作流文档 |
| 是否在 system prompt 出现 | 否（仅 slash 补全可见） | 是（描述被注入） |
| auto-invocation | 无 | 有（model 可决定） |
| 适合 | 重复性 prompt 复用 | 复杂任务工作流 |

## 2. 存放位置

| 类型 | 位置 |
|------|------|
| **全局** | `~/.pi/agent/prompts/*.md` |
| **项目** | `.pi/prompts/*.md` |
| **Package** | `prompts/` 目录 或 `package.json` 的 `pi.prompts` 入口 |
| **Settings** | `prompts` 数组指向文件或目录 |
| **CLI** | `--prompt-template <path>`，可重复 |

`prompts/` 目录**非递归扫描**——子目录的 template 必须通过 settings 或 package manifest 显式注册。

用 `--no-prompt-templates` 整体禁用。

## 3. 文件格式

YAML frontmatter + prompt 正文：

```markdown
---
description: Review staged git changes
---
Review the staged changes (`git diff --cached`). Focus on:
- Bugs and logic errors
- Security issues
- Error handling gaps
```

| 字段 | 说明 |
|------|------|
| `description` | 可选；缺省时 Pi 取**第一行非空内容** |
| `argument-hint` | 可选；autocomplete 里显示在 description 之前 |

## 4. Argument Hints

用尖括号 `<>` 表示必填、方括号 `[]` 表示可选：

```markdown
---
description: Review PRs from URLs with structured issue and code analysis
argument-hint: "<PR-URL>"
---
```

autocomplete 渲染样式：

```text
→ pr   <PR-URL>       — Review PRs from URLs with structured issue and code analysis
  is   <issue>        — Analyze GitHub issues (bugs or feature requests)
  wr   [instructions] — Finish the current task end-to-end
  cl   — Audit changelog entries before release
```

## 5. 调用

输入 `/` + template 名，autocomplete 会列出带 description 的所有 template。

```text
/review                            # 展开 review.md
/component Button                  # 带参数展开
/component Button "click handler"  # 多参数
```

## 6. 参数占位符

| 占位符 | 含义 |
|--------|------|
| `$1`, `$2`, ... | 第 N 个 positional 参数 |
| `$@` 或 `$ARGUMENTS` | 全部参数拼接 |
| `${@:N}` | 从第 N 个开始的所有参数（1-indexed） |
| `${@:N:L}` | 从第 N 个开始的 L 个参数 |

### 例子

template `component.md`：

```markdown
---
description: Create a component
---
Create a React component named $1 with features: $@
```

调用：

```text
/component Button "onClick handler" "disabled support"
```

展开为：

```text
Create a React component named Button with features: Button "onClick handler" "disabled support"
```

## 7. 加载规则

- `prompts/` 目录**非递归**
- 子目录里的 template 要在 settings 或 package manifest 里显式列出来才会被加载
