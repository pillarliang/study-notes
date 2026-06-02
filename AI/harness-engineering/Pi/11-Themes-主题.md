# 11 - Themes：主题

> 来源：https://pi.dev/docs/latest/themes

## 1. 内置主题

Pi 内置两个主题：`dark` 和 `light`。**首次启动时**根据终端背景色自动选一个作默认。

## 2. 主题文件格式

主题是 JSON。最简结构：

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "blue": "#0066cc",
    "gray": 242
  },
  "colors": {
    "accent": "blue",
    "muted": "gray",
    "text": ""
  }
}
```

规则：

- `name` — 必需，唯一
- `vars` — 可选，可复用的颜色定义（在 `colors` 里按名引用）
- `colors` — 必须包含全部 **51 个** 必需 token
- `$schema` — 启用编辑器自动补全/校验

## 3. 文件位置

| 类型 | 位置 |
|------|------|
| 内置 | `dark` / `light` |
| 全局 | `~/.pi/agent/themes/*.json` |
| 项目 | `.pi/themes/*.json` |
| Package | `themes/` 目录 或 `package.json` 的 `pi.themes` 入口 |
| Settings | `themes` 数组 |
| CLI | `--theme <path>`，可重复 |

用 `--no-themes` 禁用发现。

## 4. 选择 / 切换主题

通过 `/settings` 菜单选，或在 `settings.json`：

```json
{ "theme": "my-theme" }
```

**热重载**：编辑当前活动主题文件，Pi 自动重载预览。

## 5. 创建自定义主题

```bash
mkdir -p ~/.pi/agent/themes
vim ~/.pi/agent/themes/my-theme.json
```

定义全部 51 个 color token（见下），用 `/settings` 激活。

## 6. 全部 51 个 color token

### 6.1 核心 UI（11）

`accent` / `border` / `borderAccent` / `borderMuted` / `success` / `error` / `warning` / `muted` / `dim` / `text` / `thinkingText`

### 6.2 背景与内容（11）

`selectedBg` / `userMessageBg` / `userMessageText` / `customMessageBg` / `customMessageText` / `customMessageLabel` / `toolPendingBg` / `toolSuccessBg` / `toolErrorBg` / `toolTitle` / `toolOutput`

### 6.3 Markdown（10）

`mdHeading` / `mdLink` / `mdLinkUrl` / `mdCode` / `mdCodeBlock` / `mdCodeBlockBorder` / `mdQuote` / `mdQuoteBorder` / `mdHr` / `mdListBullet`

### 6.4 Tool Diff（3）

`toolDiffAdded` / `toolDiffRemoved` / `toolDiffContext`

### 6.5 语法高亮（9）

`syntaxComment` / `syntaxKeyword` / `syntaxFunction` / `syntaxVariable` / `syntaxString` / `syntaxNumber` / `syntaxType` / `syntaxOperator` / `syntaxPunctuation`

### 6.6 Thinking Level 边框（6）

`thinkingOff` / `thinkingMinimal` / `thinkingLow` / `thinkingMedium` / `thinkingHigh` / `thinkingXhigh`

### 6.7 Bash 模式（1）

`bashMode` — `!` 前缀时 editor 边框颜色

### 6.8 可选 HTML Export 段

控制 `/export` HTML 输出。省略时从 `userMessageBg` 推导：

```json
{
  "export": {
    "pageBg": "#18181e",
    "cardBg": "#1e1e24",
    "infoBg": "#3c3728"
  }
}
```

## 7. Color 值格式

| 格式 | 例子 | 说明 |
|------|------|------|
| Hex | `"#ff0000"` | 6 位 hex RGB |
| 256-color | `39` | xterm palette 索引 0–255 |
| Variable | `"primary"` | 引用 `vars` 里的条目 |
| Default | `""` | 终端默认色 |

### 256-color palette

- `0–15`：基础 ANSI（依赖终端）
- `16–231`：6×6×6 RGB 立方体——`16 + 36×R + 6×G + B`，R/G/B 取 0–5
- `232–255`：灰阶斜坡

### 终端兼容性

Pi 用 24-bit RGB；现代终端（iTerm2、Kitty、WezTerm、Windows Terminal、VS Code）都支持。老的 256-color 终端会做最近色回落。

检查 truecolor 支持：

```bash
echo $COLORTERM  # 应输出 "truecolor" 或 "24bit"
```

## 8. 完整示例主题

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "primary": "#00aaff",
    "secondary": 242
  },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "borderAccent": "#00ffff",
    "borderMuted": "secondary",
    "success": "#00ff00",
    "error": "#ff0000",
    "warning": "#ffff00",
    "muted": "secondary",
    "dim": 240,
    "text": "",
    "thinkingText": "secondary",
    "selectedBg": "#2d2d30",
    "userMessageBg": "#2d2d30",
    "userMessageText": "",
    "customMessageBg": "#2d2d30",
    "customMessageText": "",
    "customMessageLabel": "primary",
    "toolPendingBg": "#1e1e2e",
    "toolSuccessBg": "#1e2e1e",
    "toolErrorBg": "#2e1e1e",
    "toolTitle": "primary",
    "toolOutput": "",
    "mdHeading": "#ffaa00",
    "mdLink": "primary",
    "mdLinkUrl": "secondary",
    "mdCode": "#00ffff",
    "mdCodeBlock": "",
    "mdCodeBlockBorder": "secondary",
    "mdQuote": "secondary",
    "mdQuoteBorder": "secondary",
    "mdHr": "secondary",
    "mdListBullet": "#00ffff",
    "toolDiffAdded": "#00ff00",
    "toolDiffRemoved": "#ff0000",
    "toolDiffContext": "secondary",
    "syntaxComment": "secondary",
    "syntaxKeyword": "primary",
    "syntaxFunction": "#00aaff",
    "syntaxVariable": "#ffaa00",
    "syntaxString": "#00ff00",
    "syntaxNumber": "#ff00ff",
    "syntaxType": "#00aaff",
    "syntaxOperator": "primary",
    "syntaxPunctuation": "secondary",
    "thinkingOff": "secondary",
    "thinkingMinimal": "primary",
    "thinkingLow": "#00aaff",
    "thinkingMedium": "#00ffff",
    "thinkingHigh": "#ff00ff",
    "thinkingXhigh": "#ff0000",
    "bashMode": "#ffaa00"
  }
}
```

## 9. 实用 tips

- **深色终端**：用亮、饱和、高对比颜色
- **浅色终端**：用暗、柔和、低对比颜色
- **配色和谐**：从现成 palette 起步（Nord、Gruvbox、Tokyo Night），塞进 `vars` 复用
- **测试**：覆盖各种消息类型、tool 状态、markdown 渲染、换行场景
- **VS Code**：把 `terminal.integrated.minimumContrastRatio` 设为 `1` 才能看到准确颜色

## 10. 参考样例

源码里的内置主题：`packages/coding-agent/src/modes/interactive/theme/` 下的 `dark.json` 和 `light.json`。

> 文档原话："pi can create themes. Ask it to build one for your setup."
