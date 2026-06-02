# 12 - Pi Packages：分发与安装

> 来源：https://pi.dev/docs/latest/packages

## 1. 用途

Pi package 把 extension / skill / prompt template / theme 打成一个包，通过 npm 或 git 分发。资源声明有两种方式：

1. `package.json` 里的 `pi` key（显式 manifest）
2. 约定的目录（无 manifest 时自动发现）

## 2. 创建一个 Pi Package

加 `pi` manifest 到 `package.json`，并带 `pi-package` 关键字让 gallery 能发现：

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

路径相对于 package 根。数组支持 glob 和 `!exclusions`。

### Gallery 元数据

想让 package gallery 显示预览，加 `video` 或 `image`：

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "video": "https://example.com/demo.mp4",
    "image": "https://example.com/screenshot.png"
  }
}
```

- **video**：只支持 MP4；桌面端 hover 自动播放，点击全屏
- **image**：PNG / JPEG / GIF / WebP，静态预览
- 两者都有时 **video 优先**

### 约定目录（无 manifest）

没 `pi` manifest 时，自动从这些目录发现：

| 目录 | 加载 |
|------|------|
| `extensions/` | `.ts` 和 `.js` 文件 |
| `skills/` | 递归找含 `SKILL.md` 的目录，外加顶层 `.md` 文件 |
| `prompts/` | `.md` 文件 |
| `themes/` | `.json` 文件 |

## 3. 依赖管理

运行时依赖放 `dependencies`——Pi fetch 时跑 `npm install`。

**核心包要放 `peerDependencies` 用 `"*"`，不要打包进去**：

- `@earendil-works/pi-ai`
- `@earendil-works/pi-agent-core`
- `@earendil-works/pi-coding-agent`
- `@earendil-works/pi-tui`
- `typebox`

依赖其它 pi package 时用 `bundledDependencies` 打包，通过 `node_modules/` 引用：

```json
{
  "dependencies": {
    "shitty-extensions": "^1.0.1"
  },
  "bundledDependencies": ["shitty-extensions"],
  "pi": {
    "extensions": ["extensions", "node_modules/shitty-extensions/extensions"],
    "skills": ["skills", "node_modules/shitty-extensions/skills"]
  }
}
```

## 4. 安装与管理

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo       # 裸 URL 也行
pi install /absolute/path/to/package
pi install ./relative/path/to/package

pi remove npm:@foo/bar
pi list                     # 列出 settings 里的 package
pi update                   # 更新 pi、更新 package、reconcile pinned git ref
pi update --extensions      # 只更 package + reconcile git ref
pi update --self            # 只更 pi
pi update --self --force    # 强制重装 pi
pi update npm:@foo/bar      # 更新指定 package
pi update --extension npm:@foo/bar
```

默认写到全局 `~/.pi/agent/settings.json`。加 `-l` 写到项目 `.pi/settings.json`——**项目 settings 里缺失的 package 会在启动时自动装**。

### 不存盘的临时试用

```bash
pi -e npm:@foo/bar
pi -e git:github.com/user/repo
```

> ⚠️ **安全警告**："Pi packages run with full system access." Extension 跑任意代码、skill 能引导 model 跑任意东西。**装第三方前看一眼源码**。

## 5. Package 来源

### 5.1 npm

```text
npm:@scope/pkg@1.2.3
npm:pkg
```

带版本的 pin 不会被 `pi update` 升级。

| 安装类型 | 路径 |
|---------|------|
| user-level | `~/.pi/agent/npm/` |
| project-level | `.pi/npm/` |

要用 mise 这类工具包 npm：

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

### 5.2 git

```text
git:github.com/user/repo@v1
git:git@github.com:user/repo@v1
https://github.com/user/repo@v1
ssh://git@github.com/user/repo@v1
```

- 无 `git:` 前缀时**只接受协议 URL**
- 带 `git:` 前缀时支持 shorthand
- SSH 遵循 `~/.ssh/config`
- CI 用 `GIT_TERMINAL_PROMPT=0` 和 `GIT_SSH_COMMAND='ssh -o BatchMode=yes -o ConnectTimeout=5'`

**Ref 是 pinned 的**：`pi update` 不会升级 ref，但会 reconcile clone 到配置的 ref。换 ref 就重新 install。

Clone 位置：`~/.pi/agent/git/<host>/<path>` 或 `.pi/git/<host>/<path>`。reconciliation 改变 checkout 时，Pi 会 reset/clean，并在有 `package.json` 时跑 `npm install`。

SSH 示例：

```bash
# git@host:path shorthand（要 git: 前缀）
pi install git:git@github.com:user/repo

# ssh:// 协议
pi install ssh://git@github.com/user/repo

# 带 ref
pi install git:git@github.com:user/repo@v1.0.0
```

### 5.3 本地路径

```text
/absolute/path/to/package
./relative/path/to/package
```

本地路径**只在 settings 里引用、不复制文件**。相对路径相对于 settings 文件解析。文件加载为单 extension；目录按 package 规则加载。

## 6. 过滤 package 加载内容

settings 用对象形式精确控制：

```json
{
  "packages": [
    "npm:simple-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"],
      "themes": ["+themes/legacy.json"]
    }
  ]
}
```

规则：

| 写法 | 含义 |
|------|------|
| 省略 key | 加载该类型全部 |
| `[]` | 该类型一个都不加载 |
| `!pattern` | 排除匹配项 |
| `+path` | 强制加入某路径 |
| `-path` | 强制排除某路径 |

过滤只能**收窄**，不能扩展。

## 7. 启用 / 禁用资源

```bash
pi config
```

交互式开关 package 提供的 extension / skill / prompt template / theme——全局和项目级都支持。

## 8. 作用域与去重

同一个 package 在全局和项目 settings 里都可以出现——**项目优先**。Identity 判定：

| 来源 | 标识 |
|------|------|
| npm | package 名 |
| git | repo URL（不含 ref） |
| local | 解析后的绝对路径 |
