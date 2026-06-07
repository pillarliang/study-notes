# 12 - TUI Components：构建终端 UI

> 来源：https://pi.dev/docs/latest/tui

Pi extensions 和 custom tool 可以通过 `@earendil-works/pi-tui` 渲染交互式终端 UI。

## 1. Component 接口

每个 component 实现：

```ts
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

| 方法 | 说明 |
|------|------|
| `render(width)` | 每行一个 string；**每行长度不得超过 `width`** |
| `handleInput?(data)` | focus 时接收键盘输入 |
| `wantsKeyRelease?` | 是否启用 Kitty 协议的 key release 事件 |
| `invalidate()` | 清理缓存渲染（theme 切换时被调） |

> TUI **在每行末尾自动追加完整 SGR reset 和 OSC 8 reset**——样式不跨行。需要跨行保留样式时用 `wrapTextWithAnsi()`。

### 1.1 Focusable 接口（IME 支持）

带文本光标的 component 实现 `Focusable` 才能让 CJK IME 候选窗口出现在正确位置：

```ts
import { CURSOR_MARKER, type Component, type Focusable } from "@earendil-works/pi-tui";

class MyInput implements Component, Focusable {
  focused: boolean = false;

  render(width: number): string[] {
    const marker = this.focused ? CURSOR_MARKER : "";
    return [`> ${beforeCursor}${marker}\x1b[7m${atCursor}\x1b[27m${afterCursor}`];
  }
}
```

TUI 扫描输出里的 `CURSOR_MARKER`（零宽 APC escape）来定位硬件光标。`Editor` 和 `Input` 内置已实现。

**Container 包 `Input`/`Editor` 时也要实现 `Focusable` 并把 `focused` 传给 child**，否则 IME 候选窗口位置会偏。

## 2. 使用方式

### 2.1 在 extension 里

```ts
pi.on("session_start", async (_event, ctx) => {
  const handle = ctx.ui.custom(myComponent);
  // handle.requestRender();
  // handle.close();
});
```

### 2.2 在 custom tool 里

```ts
async execute(toolCallId, params, onUpdate, ctx, signal) {
  const handle = pi.ui.custom(myComponent);
  handle.close();
}
```

## 3. Overlays（覆盖层）

在已有内容上面渲染、不清屏。传 `{ overlay: true }`：

```ts
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new SidePanel({ onClose: done }),
  {
    overlay: true,
    overlayOptions: {
      width: "50%",           // 数字或百分比
      minWidth: 40,
      maxHeight: "80%",
      anchor: "right-center", // 9 种位置
      offsetX: -2,
      offsetY: 0,
      row: "25%",
      col: 10,
      margin: 2,              // 或 { top, right, bottom, left }
      visible: (termWidth, termHeight) => termWidth >= 80,
    },
    onHandle: (handle) => {
      // handle.setHidden(true/false); handle.hide();
    },
  }
);
```

**生命周期注意**：overlay component 关闭时被 dispose——**不要持有过期引用**，要重新显示就再调一次 show 函数。

## 4. 内置 Component

从 `@earendil-works/pi-tui` import。

### Text

带 word wrap 的多行文本。

```ts
new Text("Hello", 1 /*paddingX*/, 1 /*paddingY*/, (s) => bgGray(s));
text.setText("Updated");
```

### Box

带 padding 和背景色的容器。

```ts
const box = new Box(1, 1, (s) => bgGray(s));
box.addChild(new Text("Content", 0, 0));
box.setBgFn((s) => bgBlue(s));
```

### Container

垂直堆叠 children。

```ts
const c = new Container();
c.addChild(component1);
c.removeChild(component1);
```

### Spacer

```ts
new Spacer(2); // 2 行空白
```

### Markdown

带语法高亮的 markdown。

```ts
new Markdown("# Title\n**bold**", 1, 1, theme);
md.setText("Updated");
```

### Image

在 Kitty / iTerm2 / Ghostty / WezTerm 上能显示。

```ts
new Image(base64Data, "image/png", theme, {
  maxWidthCells: 80,
  maxHeightCells: 24,
});
```

## 5. 键盘输入

```ts
import { matchesKey, Key } from "@earendil-works/pi-tui";

if (matchesKey(data, Key.up)) { /* ... */ }
if (matchesKey(data, Key.ctrl("c"))) { /* ... */ }
```

**Key 标识**：

- 基础：`Key.enter`、`escape`、`tab`、`space`、`backspace`、`delete`、`home`、`end`
- 方向：`Key.up/down/left/right`
- 带 modifier：`Key.ctrl("c")`、`Key.shift("tab")`、`Key.alt("left")`、`Key.ctrlShift("p")`
- 字符串形式：`"enter"`、`"ctrl+c"`、`"shift+tab"`、`"ctrl+shift+p"`

## 6. 行宽工具

每行渲染**必须**不超过 `width` 参数。

| 函数 | 用途 |
|------|------|
| `visibleWidth(str)` | 计算显示宽度（忽略 ANSI） |
| `truncateToWidth(str, width, ellipsis?)` | 截断到指定宽度 |
| `wrapTextWithAnsi(str, width)` | 按词换行同时保留 ANSI |

## 7. 自定义 Component 模板

把缓存、输入处理、生命周期回调结合起来：

```ts
import { matchesKey, Key, truncateToWidth } from "@earendil-works/pi-tui";

class MySelector {
  private items: string[];
  private selected = 0;
  private cachedWidth?: number;
  private cachedLines?: string[];
  public onSelect?: (item: string) => void;
  public onCancel?: () => void;

  constructor(items: string[]) {
    this.items = items;
  }

  handleInput(data: string): void {
    if (matchesKey(data, Key.up) && this.selected > 0) {
      this.selected--;
      this.invalidate();
    } else if (matchesKey(data, Key.down) && this.selected < this.items.length - 1) {
      this.selected++;
      this.invalidate();
    } else if (matchesKey(data, Key.enter)) {
      this.onSelect?.(this.items[this.selected]);
    } else if (matchesKey(data, Key.escape)) {
      this.onCancel?.();
    }
  }

  render(width: number): string[] {
    if (this.cachedLines && this.cachedWidth === width) return this.cachedLines;
    this.cachedLines = this.items.map((item, i) => {
      const prefix = i === this.selected ? "> " : "  ";
      return truncateToWidth(prefix + item, width);
    });
    this.cachedWidth = width;
    return this.cachedLines;
  }

  invalidate(): void {
    this.cachedWidth = undefined;
    this.cachedLines = undefined;
  }
}
```

## 8. 主题（Theming）

**始终用回调里传入的 `theme`**，不要 import 全局 theme。

### 8.1 前景色（`theme.fg(color, text)`）分类

| 分类 | 颜色 key |
|------|---------|
| 通用 | `text`, `accent`, `muted`, `dim` |
| 状态 | `success`, `error`, `warning` |
| 边框 | `border`, `borderAccent`, `borderMuted` |
| 消息 | `userMessageText`, `customMessageText`, `customMessageLabel` |
| Tool | `toolTitle`, `toolOutput` |
| Diff | `toolDiffAdded`, `toolDiffRemoved`, `toolDiffContext` |
| Markdown | `mdHeading`, `mdLink`, `mdLinkUrl`, `mdCode`, `mdCodeBlock`, `mdCodeBlockBorder`, `mdQuote`, `mdQuoteBorder`, `mdHr`, `mdListBullet` |
| 语法 | `syntaxComment`, `syntaxKeyword`, `syntaxFunction`, `syntaxVariable`, `syntaxString`, `syntaxNumber`, `syntaxType`, `syntaxOperator`, `syntaxPunctuation` |
| Thinking | `thinkingOff/Minimal/Low/Medium/High/Xhigh` |
| Mode | `bashMode` |

### 8.2 背景色（`theme.bg(color, text)`）

`selectedBg`、`userMessageBg`、`customMessageBg`、`toolPendingBg`、`toolSuccessBg`、`toolErrorBg`。

### 8.3 Markdown 主题

用 `getMarkdownTheme()`（从 `@earendil-works/pi-coding-agent`）。

## 9. Invalidation 与主题切换

如果 component 把 theme 颜色**预先烤进**字符串再存到 child component 里，主题切换时必须在 `invalidate()` 时重建：

```ts
class GoodComponent extends Container {
  private message: string;
  private content: Text;

  constructor(message: string) {
    super();
    this.message = message;
    this.content = new Text("", 1, 0);
    this.addChild(this.content);
    this.updateDisplay();
  }

  private updateDisplay(): void {
    this.content.setText(theme.fg("accent", this.message));
  }

  override invalidate(): void {
    super.invalidate();
    this.updateDisplay();
  }
}
```

**何时需要重建**：

- 预先烤进 theme 颜色的字符串
- 用 `highlightCode()` 做语法高亮
- 复杂布局

**何时不需要**：

- 用 callback 形式 `(text) => theme.fg("accent", text)`
- 简单 container
- 完全无状态的 render

## 10. 常见 Pattern

### 10.1 选择对话框（`SelectList` + `DynamicBorder`）

```ts
const items: SelectItem[] = [
  { value: "opt1", label: "Option 1", description: "First" },
];
const result = await ctx.ui.custom<string | null>(
  (tui, theme, _kb, done) => {
    const container = new Container();
    container.addChild(new DynamicBorder((s: string) => theme.fg("accent", s)));

    const selectList = new SelectList(items, Math.min(items.length, 10), {
      selectedPrefix: (t) => theme.fg("accent", t),
      selectedText: (t) => theme.fg("accent", t),
      description: (t) => theme.fg("muted", t),
      scrollInfo: (t) => theme.fg("dim", t),
      noMatch: (t) => theme.fg("warning", t),
    });
    selectList.onSelect = (item) => done(item.value);
    selectList.onCancel = () => done(null);

    container.addChild(selectList);
    container.addChild(new DynamicBorder((s: string) => theme.fg("accent", s)));

    return {
      render: (w) => container.render(w),
      invalidate: () => container.invalidate(),
      handleInput: (data) => {
        selectList.handleInput(data);
        tui.requestRender();
      },
    };
  },
);
```

### 10.2 异步带取消（`BorderedLoader`）

```ts
const result = await ctx.ui.custom<string | null>(
  (tui, theme, _kb, done) => {
    const loader = new BorderedLoader(tui, theme, "Fetching data...");
    loader.onAbort = () => done(null);
    fetchData(loader.signal)
      .then((d) => done(d))
      .catch(() => done(null));
    return loader;
  },
);
```

### 10.3 设置/开关（`SettingsList`）

```ts
const items: SettingItem[] = [
  { id: "verbose", label: "Verbose mode", currentValue: "off", values: ["on", "off"] },
];
await ctx.ui.custom((_tui, theme, _kb, done) => {
  const container = new Container();
  const settingsList = new SettingsList(
    items,
    15,
    getSettingsListTheme(),
    (id, newValue) => ctx.ui.notify(`${id} = ${newValue}`, "info"),
    () => done(undefined),
    { enableSearch: true },
  );
  container.addChild(settingsList);
  return {
    render: (w) => container.render(w),
    invalidate: () => container.invalidate(),
    handleInput: (data) => settingsList.handleInput?.(data),
  };
});
```

### 10.4 持久状态指示

```ts
ctx.ui.setStatus("my-ext", ctx.ui.theme.fg("accent", "● active"));
ctx.ui.setStatus("my-ext", undefined); // 清除
```

### 10.5 Working Indicator

```ts
ctx.ui.setWorkingIndicator({
  frames: [
    theme.fg("dim", "·"),
    theme.fg("muted", "•"),
    theme.fg("accent", "●"),
  ],
  intervalMs: 120,
});
ctx.ui.setWorkingIndicator({ frames: [] }); // 隐藏
ctx.ui.setWorkingIndicator();                // 还原默认
```

只影响普通 streaming，compaction/retry 的 loader 保持内置样式。

### 10.6 Widgets（editor 上下方）

```ts
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"]);

ctx.ui.setWidget("my-widget", ["..."], { placement: "belowEditor" });

ctx.ui.setWidget("my-widget", (_tui, theme) => ({
  render: () => [theme.fg("success", "✓ done")],
  invalidate: () => {},
}));

ctx.ui.setWidget("my-widget", undefined);
```

### 10.7 自定义 Footer

```ts
ctx.ui.setFooter((tui, theme, footerData) => ({
  invalidate() {},
  render(width) {
    return [`${ctx.model?.id} (${footerData.getGitBranch() || "no git"})`];
  },
  dispose: footerData.onBranchChange(() => tui.requestRender()),
}));

ctx.ui.setFooter(undefined);
```

`footerData` 暴露 `getGitBranch()`、`getExtensionStatuses()`、`onBranchChange()`。token 统计走 `ctx.sessionManager.getBranch()` + `ctx.model`。

### 10.8 自定义 Editor（vim 模式）—— 继承 `CustomEditor`

```ts
class VimEditor extends CustomEditor {
  private mode: "normal" | "insert" = "insert";

  handleInput(data: string): void {
    if (matchesKey(data, "escape")) {
      if (this.mode === "insert") {
        this.mode = "normal";
        return;
      }
      super.handleInput(data);
      return;
    }
    if (this.mode === "insert") {
      super.handleInput(data);
      return;
    }
    switch (data) {
      case "i": this.mode = "insert"; return;
      case "h": super.handleInput("\x1b[D"); return;
      case "j": super.handleInput("\x1b[B"); return;
      case "k": super.handleInput("\x1b[A"); return;
      case "l": super.handleInput("\x1b[C"); return;
    }
    if (data.length === 1 && data.charCodeAt(0) >= 32) return;
    super.handleInput(data);
  }
}

ctx.ui.setEditorComponent((tui, theme, keybindings) =>
  new VimEditor(theme, keybindings),
);
ctx.ui.setEditorComponent(undefined); // 还原默认
```

**继承 `CustomEditor`（不要继承 `Editor`）**——这样能保留 app 级别快捷键（escape 中止、ctrl+d 退出、model 切换等）。

## 11. 关键规则

1. 从 callback `(tui, theme, keybindings, done) => ...` 拿 `theme`，**不要 import 全局**
2. `DynamicBorder` 颜色回调参数要**显式声明类型**：`(s: string) => theme.fg("accent", s)`
3. 在 `handleInput` 里改 state 后调 `tui.requestRender()`
4. 自定义 component 返回的对象需要三个方法：`{ render, invalidate, handleInput }`
5. 复用 `SelectList`、`SettingsList`、`BorderedLoader`，不要重新实现

## 12. 调试

抓 stdout 原始 ANSI：

```bash
PI_TUI_WRITE_LOG=/tmp/tui-ansi.log npx tsx packages/tui/test/chat-simple.ts
```

## 13. 性能

按 `width` 缓存 `render()` 输出，`invalidate()` 时清掉缓存；再调 `handle.requestRender()` 触发重绘。

## 14. 官方示例 extension

位于 `packages/coding-agent/examples/extensions/`：

| 文件 | 演示 |
|------|------|
| `preset.ts` | SelectList + DynamicBorder 框选 |
| `qna.ts` | BorderedLoader 跑 LLM 调用 |
| `tools.ts` | SettingsList 切 tool 开关 |
| `plan-mode.ts` | setStatus + setWidget |
| `working-indicator.ts` | setWorkingIndicator |
| `custom-footer.ts` | setFooter + 统计 |
| `modal-editor.ts` | vim 风格 modal 编辑 |
| `snake.ts` | 含输入处理和游戏循环的完整小游戏 |
| `todo.ts` | 自定义 tool 的 `renderCall`/`renderResult` |
| `overlay-qa-tests.ts` | anchor、margin、stacking、响应式可见性 |
