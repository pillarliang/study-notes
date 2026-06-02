# 05 - Keybindings 快捷键

> 来源：https://pi.dev/docs/latest/keybindings

## 1. 自定义配置

文件位置：`~/.pi/agent/keybindings.json`。

每个 action ID 接受**单个 key 字符串**或**字符串数组**。用户配置覆盖默认值；旧的非命名空间 ID（如 `cursorUp`）会自动迁移成新的（`tui.editor.cursorUp`）。

修改后跑 `/reload` 应用，不必重启 Pi。

## 2. Key 字符串格式

格式：`modifier+key`。modifier 有 `ctrl` / `shift` / `alt`，可组合（如 `ctrl+shift+x`）。

| 类别 | 可用键 |
|------|--------|
| 字母 | `a-z` |
| 数字 | `0-9` |
| 特殊键 | `escape`(`esc`)、`enter`(`return`)、`tab`、`space`、`backspace`、`delete`、`insert`、`clear`、`home`、`end`、`pageUp`、`pageDown`、`up`、`down`、`left`、`right` |
| 功能键 | `f1`–`f12` |
| 符号 | `` ` `` `-` `=` `[` `]` `\` `;` `'` `,` `.` `/` `!` `@` `#` `$` `%` `^` `&` `*` `(` `)` `_` `+` `\|` `~` `{` `}` `:` `<` `>` `?` |

## 3. 默认快捷键全集

### 3.1 TUI Editor — 光标移动

| Action ID | 默认 | 说明 |
|---|---|---|
| `tui.editor.cursorUp` | `up` | 上移 |
| `tui.editor.cursorDown` | `down` | 下移 |
| `tui.editor.cursorLeft` | `left`, `ctrl+b` | 左移 |
| `tui.editor.cursorRight` | `right`, `ctrl+f` | 右移 |
| `tui.editor.cursorWordLeft` | `alt+left`, `ctrl+left`, `alt+b` | 按词左移 |
| `tui.editor.cursorWordRight` | `alt+right`, `ctrl+right`, `alt+f` | 按词右移 |
| `tui.editor.cursorLineStart` | `home`, `ctrl+a` | 行首 |
| `tui.editor.cursorLineEnd` | `end`, `ctrl+e` | 行尾 |
| `tui.editor.jumpForward` | `ctrl+]` | 向前跳到字符 |
| `tui.editor.jumpBackward` | `ctrl+alt+]` | 向后跳到字符 |
| `tui.editor.pageUp` | `pageUp` | 上翻一页 |
| `tui.editor.pageDown` | `pageDown` | 下翻一页 |

### 3.2 TUI Editor — 删除

| Action ID | 默认 | 说明 |
|---|---|---|
| `tui.editor.deleteCharBackward` | `backspace` | 删除前一字符 |
| `tui.editor.deleteCharForward` | `delete`, `ctrl+d` | 删除后一字符 |
| `tui.editor.deleteWordBackward` | `ctrl+w`, `alt+backspace` | 删除前一词 |
| `tui.editor.deleteWordForward` | `alt+d`, `alt+delete` | 删除后一词 |
| `tui.editor.deleteToLineStart` | `ctrl+u` | 删到行首 |
| `tui.editor.deleteToLineEnd` | `ctrl+k` | 删到行尾 |

### 3.3 TUI Input

| Action ID | 默认 | 说明 |
|---|---|---|
| `tui.input.newLine` | `shift+enter` | 换行 |
| `tui.input.submit` | `enter` | 提交 |
| `tui.input.tab` | `tab` | Tab / 补全 |

### 3.4 TUI Kill Ring（删除文本暂存环）

| Action ID | 默认 | 说明 |
|---|---|---|
| `tui.editor.yank` | `ctrl+y` | 粘贴最近删除的文本 |
| `tui.editor.yankPop` | `alt+y` | yank 之后循环切换历史删除项 |
| `tui.editor.undo` | `ctrl+-` | 撤销 |

> kill ring 是 Emacs 的概念：删除操作不丢弃而入栈；yank 出来，alt+y 翻历史。

### 3.5 TUI Clipboard 与选择

| Action ID | 默认 | 说明 |
|---|---|---|
| `tui.input.copy` | `ctrl+c` | 复制选区 |
| `tui.select.up` | `up` | 选区上移 |
| `tui.select.down` | `down` | 选区下移 |
| `tui.select.pageUp` | `pageUp` | 列表上翻页 |
| `tui.select.pageDown` | `pageDown` | 列表下翻页 |
| `tui.select.confirm` | `enter` | 确认选择 |
| `tui.select.cancel` | `escape`, `ctrl+c` | 取消 |

### 3.6 Application

| Action ID | 默认 | 说明 |
|---|---|---|
| `app.interrupt` | `escape` | 取消 / 中断 |
| `app.clear` | `ctrl+c` | 清空 editor |
| `app.exit` | `ctrl+d` | editor 空时退出 |
| `app.suspend` | `ctrl+z`（Windows 无） | 挂起到后台 |
| `app.editor.external` | `ctrl+g` | 调用 `$VISUAL` / `$EDITOR` |
| `app.clipboard.pasteImage` | `ctrl+v`（Windows 用 `alt+v`） | 粘贴剪贴板图片 |

### 3.7 Sessions

| Action ID | 默认 | 说明 |
|---|---|---|
| `app.session.new` | _(无)_ | 新建 session（`/new`） |
| `app.session.tree` | _(无)_ | 打开树导航器（`/tree`） |
| `app.session.fork` | _(无)_ | fork 当前 session |
| `app.session.resume` | _(无)_ | resume 选择器 |
| `app.session.togglePath` | `ctrl+p` | 显示/隐藏路径 |
| `app.session.toggleSort` | `ctrl+s` | 切换排序模式 |
| `app.session.toggleNamedFilter` | `ctrl+n` | 只看命名 session |
| `app.session.rename` | `ctrl+r` | 重命名 session |
| `app.session.delete` | `ctrl+d` | 删 session |
| `app.session.deleteNoninvasive` | `ctrl+backspace` | 输入框为空时的删除 |

### 3.8 Models 与 Thinking

| Action ID | 默认 | 说明 |
|---|---|---|
| `app.model.select` | `ctrl+l` | 打开 model 选择器 |
| `app.model.cycleForward` | `ctrl+p` | 下一个 model |
| `app.model.cycleBackward` | `shift+ctrl+p` | 上一个 model |
| `app.thinking.cycle` | `shift+tab` | 切换 thinking 等级 |
| `app.thinking.toggle` | `ctrl+t` | 折叠/展开 thinking 块 |

### 3.9 显示与消息队列

| Action ID | 默认 | 说明 |
|---|---|---|
| `app.tools.expand` | `ctrl+o` | 折叠/展开 tool 输出 |
| `app.message.followUp` | `alt+enter` | 排 follow-up 消息 |
| `app.message.dequeue` | `alt+up` | 把排队消息取回 editor |

### 3.10 树导航

| Action ID | 默认 | 说明 |
|---|---|---|
| `app.tree.foldOrUp` | `ctrl+left`, `alt+left` | 折叠分支或跳到段首 |
| `app.tree.unfoldOrDown` | `ctrl+right`, `alt+right` | 展开分支或跳到段尾/分支末 |
| `app.tree.editLabel` | `shift+l` | 编辑节点 label |
| `app.tree.toggleLabelTimestamp` | `shift+t` | 切换 label 时间戳显示 |
| `app.tree.filter.default` | `ctrl+d` | 默认过滤视图 |
| `app.tree.filter.noTools` | `ctrl+t` | 隐藏 tool 结果 |
| `app.tree.filter.userOnly` | `ctrl+u` | 只看 user 消息 |
| `app.tree.filter.labeledOnly` | `ctrl+l` | 只看带 label 项 |
| `app.tree.filter.all` | `ctrl+a` | 显示所有 |
| `app.tree.filter.cycleForward` | `ctrl+o` | 过滤模式向下循环 |
| `app.tree.filter.cycleBackward` | `shift+ctrl+o` | 过滤模式向上循环 |

树的形态和操作语义见 [[06-Sessions-会话树]]。

### 3.11 Scoped Models 选择器（`/scoped-models`）

| Action ID | 默认 | 说明 |
|---|---|---|
| `app.models.save` | `ctrl+s` | 保存到 settings |
| `app.models.enableAll` | `ctrl+a` | 全选（或匹配 search 的全选） |
| `app.models.clearAll` | `ctrl+x` | 清空 |
| `app.models.toggleProvider` | `ctrl+p` | 当前 provider 全部 model 切换 |
| `app.models.reorderUp` | `alt+up` | 上移 model 在循环中的位置 |
| `app.models.reorderDown` | `alt+down` | 下移 |

## 4. 平台差异

在原生 Windows 上，`app.suspend` 默认未绑定——Windows 终端不支持 Unix job control。手动绑后只会显示状态消息，不会真挂起。**WSL** 保留标准 `ctrl+z` / `fg` 行为。

## 5. 配置示例

### 基础

```json
{
  "tui.editor.cursorUp": ["up", "ctrl+p"],
  "tui.editor.cursorDown": ["down", "ctrl+n"],
  "tui.editor.deleteWordBackward": ["ctrl+w", "alt+backspace"]
}
```

### Emacs 风

```json
{
  "tui.editor.cursorUp": ["up", "ctrl+p"],
  "tui.editor.cursorDown": ["down", "ctrl+n"],
  "tui.editor.cursorLeft": ["left", "ctrl+b"],
  "tui.editor.cursorRight": ["right", "ctrl+f"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+f"],
  "tui.editor.deleteCharForward": ["delete", "ctrl+d"],
  "tui.editor.deleteCharBackward": ["backspace", "ctrl+h"],
  "tui.input.newLine": ["shift+enter", "ctrl+j"]
}
```

### Vim 风

```json
{
  "tui.editor.cursorUp": ["up", "alt+k"],
  "tui.editor.cursorDown": ["down", "alt+j"],
  "tui.editor.cursorLeft": ["left", "alt+h"],
  "tui.editor.cursorRight": ["right", "alt+l"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+w"]
}
```
