# 第一章：搭建 Go 开发环境

## 1. 安装 Go 工具链

### 各平台安装方式

| 平台 | 安装方式 |
|------|--------|
| **macOS** | 下载 `.pkg` 安装包，或使用 Homebrew：`brew install go` |
| **Windows** | 下载 `.msi` 安装包，或使用 Chocolatey：`choco install golang` |
| **Linux/BSD** | 下载 gzipped TAR 文件并手动安装 |

Linux 手动安装示例：

```bash
$ tar -C /usr/local -xzf go1.20.5.linux-amd64.tar.gz
$ echo 'export PATH=$PATH:/usr/local/go/bin' >> $HOME/.bash_profile
$ source $HOME/.bash_profile
```

### 验证安装

```bash
$ go version
go version go1.20.5 darwin/arm64
```

> **Go 的一大优势**：Go 程序编译为单一的原生二进制文件，运行时不需要安装任何额外的软件（不像 Java/Python/JavaScript 需要虚拟机或解释器）。这使得分发 Go 程序非常方便。

---

## 2. Go 工具链概览

所有 Go 开发工具都通过 `go` 命令访问：

| 命令 | 用途 |
|------|------|
| `go version` | 查看 Go 版本 |
| `go build` | 编译代码 |
| `go fmt` | 格式化代码 |
| `go mod` | 依赖管理 |
| `go test` | 运行测试 |
| `go vet` | 静态检查常见错误 |

---

## 3. 第一个 Go 程序

### 3.1 创建 Go Module

Go 项目被称为 **module（模块）**。模块不仅包含源代码，还精确指定了代码的依赖关系。每个模块的根目录都有一个 `go.mod` 文件。

```bash
$ mkdir ch1
$ cd ch1
$ go mod init hello_world
go: creating new go.mod: module hello_world
```

生成的 `go.mod` 文件内容：

```
module hello_world

go 1.20
```

> `go.mod` 类似于 Python 的 `requirements.txt` 或 Ruby 的 `Gemfile`。不要直接编辑它，使用 `go get` 和 `go mod tidy` 来管理依赖。

### 3.2 编写 Hello World

创建 `hello.go`：

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world!")
}
```

**代码解析：**

- `package main` — 包声明。Go 模块中的代码组织为一个或多个包，`main` 包是程序的入口
- `import "fmt"` — 导入标准库的 `fmt` 包。Go 只能导入整个包，不能只导入包内的特定类型或函数
- `func main()` — 所有 Go 程序从 `main` 包的 `main` 函数开始执行
- `fmt.Println(...)` — 调用 `fmt` 包的 `Println` 函数打印输出

### 3.3 编译和运行

```bash
$ go run .              # 直接运行

$ go build              # 编译，生成 hello_world 可执行文件
$ ./hello_world
Hello, world!

$ go build -o hello     # 用 -o 指定输出文件名
```

---

## 4. go fmt — 代码格式化

Go 强制采用统一的代码格式，这是语言的核心设计目标之一。好处包括：

- 消除代码风格之争（花括号位置、tab vs 空格等）
- 简化编译器和代码生成工具的实现
- 让代码更容易阅读和维护

```bash
$ go fmt ./...    # 格式化当前目录及所有子目录的 Go 文件
```

> `./...` 是 Go 工具的通用模式，表示"当前目录及所有子目录"。

**Go 的格式规则：**
- 使用 **tab** 缩进（不是空格）
- 左花括号 `{` **必须**和声明在同一行（否则是语法错误）

### 分号插入规则

Go 和 C/Java 一样在语句末尾需要分号，但开发者不需要手动写。Go 编译器会自动插入分号——如果一行的最后一个 token 是以下之一，lexer 会在其后插入分号：

- 标识符（如 `int`、`float64`）
- 基本字面量（数字、字符串常量）
- 关键字 `break`、`continue`、`fallthrough`、`return`
- 运算符 `++`、`--`、`)`、`}`

这就是为什么左花括号不能换行写。例如：

```go
// 错误写法！
func main()
{
    fmt.Println("Hello, world!")
}
```

编译器会在 `main()` 后自动插入分号，变成：

```go
func main();  // 语法错误！
{
    fmt.Println("Hello, world!");
};
```

---

## 5. go vet — 静态分析

`go vet` 用于检测语法正确但逻辑可能有问题的代码。例如：

```go
// 有 bug：%s 占位符没有对应的参数
fmt.Printf("Hello, %s!\n")
```

运行 `go vet` 检测：

```bash
$ go vet ./...
# hello_world
./hello.go:6:2: fmt.Printf format %s reads arg #1, but call has 0 args
```

修复方式：

```go
fmt.Printf("Hello, %s!\n", "world")
```

> 建议在每次提交代码前都运行 `go fmt` 和 `go vet`。

---

## 6. 开发工具选择

### Visual Studio Code

- 免费、最流行的代码编辑器
- 安装 Go 扩展即可获得 Go 支持
- 依赖第三方扩展：Go Development tools、Delve 调试器、gopls 语言服务器
- 需要先安装 Go 编译器，扩展会自动安装 Delve 和 gopls

### GoLand

- JetBrains 出品的 Go 专用 IDE
- 开箱即用，无需安装插件
- 功能包括：重构、语法高亮、代码补全和导航、文档弹窗、调试器、代码覆盖率
- 商业软件（学生和开源贡献者可申请免费授权）

### The Go Playground

- 在线运行 Go 代码的工具，访问 `go.dev/play`
- 支持多文件模拟（用 `-- filename.go --` 分隔）
- **限制**：只能连接 `localhost`、运行时间和内存有限、时钟固定为 2009-11-10 23:00:00 UTC
- 适合快速试验和分享代码片段
- **注意**：不要在 Playground 中输入敏感信息，Share 的内容保存在 Google 服务器上

---

## 7. Makefile 自动化构建

使用 `make` 将格式化、检查、编译串联为自动化流程：

```makefile
.DEFAULT_GOAL := build

.PHONY: fmt vet build

fmt:
	go fmt ./...

vet: fmt
	go vet ./...

build: vet
	go build
```

**关键概念：**

- `.DEFAULT_GOAL` — 定义默认目标（不指定 target 时执行 `build`）
- `.PHONY` — 声明伪目标，防止与同名文件冲突
- 目标依赖链：`build` → `vet` → `fmt`，执行 `make` 会依次运行三个步骤
- Makefile 中**必须用 tab 缩进**（不能用空格）

```bash
$ make
go fmt ./...
go vet ./...
go build
```

> 一条 `make` 命令就完成了格式化、静态检查和编译，确保构建流程可重复、可自动化。

---

## 8. Go 兼容性承诺

- Go 从 1.2 起大约每 6 个月发布一个新版本，同时发布补丁和安全修复
- **Go Compatibility Promise**：对于任何以 1 开头的 Go 版本，语言和标准库不会有破坏性变更（除非是修复 bug 或安全问题）
- 注意：该承诺不适用于 `go` 命令本身的 flag 和功能

---

## 9. 保持 Go 版本更新

- Go 编译为独立的原生二进制文件，更新开发工具不会影响已部署的程序
- 同一台机器可以运行不同 Go 版本编译的程序

**更新方式：**

| 平台 | 更新方式 |
|------|--------|
| macOS (Homebrew) | `brew upgrade go` |
| Windows (Chocolatey) | `choco upgrade golang` |
| 安装包用户 | 下载最新安装包覆盖安装 |
| Linux/BSD | 备份旧版本 → 解压新版本 → 删除旧版本 |

Linux 更新示例：

```bash
$ mv /usr/local/go /usr/local/old-go
$ tar -C /usr/local -xzf go1.20.6.linux-amd64.tar.gz
$ rm -rf /usr/local/old-go
```

---

## 小结

本章学习了：

1. **安装 Go** — 各平台的安装和验证方法
2. **核心工具** — `go build`、`go fmt`、`go vet` 的用途
3. **第一个程序** — module 创建、代码结构、编译运行
4. **分号插入规则** — 理解为什么花括号不能换行
5. **IDE 选择** — VS Code、GoLand、Go Playground
6. **Makefile** — 自动化构建流程
7. **兼容性承诺** — Go 的版本稳定性保证
