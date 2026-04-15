---
title: "Where teams and agents work together"
source: "https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef"
author:
published:
created: 2026-04-13
description: "A collaborative AI workspace, built on your company context. Build and orchestrate agents right alongside your team's projects, meetings, and connected apps."
tags:
  - "clippings"
---
## 安装与启动[一、前置检查](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab08081b25ce9e78f3d1006)[二、安装 Go 的两种主流方式](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab08039a2eeee9ee3cf891e)[方式 A：使用官方安装包（推荐小白/稳定）](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab08061b03fd175984ca5a2)[方式 B：使用 Homebrew（便于升级/多版本管理）](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab080aaa5a5e09847a94081)[三、环境变量与路径设置](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab080408f19e9b36d98ea30)[1) PATH](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab080bc9af1d83114710994)[2) GOPATH（可选）](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab0801681e7fa78ff62119b)[3) 代理（可选）](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab080c58d80ee0098c3cd92)[Go Modules & GOPATH](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab0809db445fe074ce80d7d)[四、第一个 Go 项目（Go Modules 正确姿势）](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab0804d8605c0154d18709d)[五、编辑器与调试](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab0805b954aec6f7b04384d)[六、常见命令速查](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab0806a99d1f0dd4da9cfe3)[1\. 当你 clone（拉取）一个 Go 项目 时：](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab080d88f37e0c5932801f1)[2\. 要自己加依赖，怎么做？](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab0808994b3c9e32a405a33)[✅ 推荐方式：直接 import + go mod tidy](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab080e790f5e8dbdc3b5491)[🧠 三、总结成一个简单的心智模型](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab0806e93c9d51ff97b7c7a)[八、升级/卸载](https://www.notion.so/pillarl/2880718c9ab0801eb001d9fab461f3ef?pvs=25#2880718c9ab080858f56d23d38903429)

## 一、前置检查

确认芯片架构

uname -m # 显示 arm64（Apple Silicon）或 x86\_64（Intel）

更新命令行工具（可选但推荐）

xcode-select --install

## 二、安装 Go 的两种主流方式

### 方式 A：使用官方安装包（推荐小白/稳定）

打开官网下载安装包（自动匹配架构）：

[https://go.dev/dl/](https://go.dev/dl/)

下载类似

go1.xx.x.darwin-arm64.pkg

（Apple Silicon）或

...-amd64.pkg

（Intel）。双击

.pkg

一路 “Continue”，默认会将 Go 安装到：二进制：

/usr/local/go/bin/go

建议：此方式通常无需手动配置

GOROOT

。

验证：

go version # 输出例：go version go1.23.1 darwin/arm64

### 方式 B：使用 Homebrew（便于升级/多版本管理）

> 若未装 Homebrew：

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

安装 Go：

brew install go

验证：

go version

升级：

brew upgrade go

> 需要多版本管理时，可考虑 asdf：

brew install asdf asdf plugin add golang https://github.com/asdf-community/asdf-golang.git asdf install golang 1.22.6 asdf global golang 1.22.6 go versio

## 三、环境变量与路径设置

> 现代 Go（1.13+）默认使用 Go Modules，不再强制依赖 GOPATH。一般只需要把 Go 的 bin 目录加入 PATH。

#### 1) PATH

官方

.pkg

安装通常已把

/usr/local/go/bin

放入 PATH；若无，或用 Homebrew：zsh（macOS 默认 Shell）：编辑

~/.zshrc

\# 官方 pkg 安装： export PATH="/usr/local/go/bin:$PATH" # Homebrew 安装（Apple Silicon 默认前缀）： export PATH="/opt/homebrew/bin:$PATH"

应用生效：

source ~/.zshrc

#### 2) GOPATH（可选）

不设也可以。若你希望有一个固定的工作区（常用于放置可执行文件、缓存等）：

mkdir -p $HOME/go echo 'export GOPATH="$HOME/go"' >> ~/.zshrc echo 'export PATH="$GOPATH/bin:$PATH"' >> ~/.zshrc source ~/.zshrc

查看当前配置：

go env GOPATH go env GOROOT # 通常无需手动设置

#### 3) 代理（可选）

新加坡网络一般直连 OK。若你需要加速依赖下载，可设置：

go env -w GOPROXY=https://proxy.golang.org,direct # 若在国内网络，可用 goproxy.cn： # go env -w GOPROXY=https://goproxy.cn,direct

### Go Modules & GOPATH

## 四、第一个 Go 项目（Go Modules 正确姿势）

新建项目文件夹：

mkdir -p ~/Dev/hello-go && cd ~/Dev/hello-go

初始化模块（

<module\_path>

建议用你的仓库地址或占位名）：

go mod init hello-go

写一个最小程序：

\# main.go package main import "fmt" func main() { fmt.Println("Hello, Go on macOS!") }

运行与构建：

go run. # 直接运行 go build -o hello # 生成可执行文件./hello./hello

## 五、编辑器与调试

VS Code + 官方 Go 扩展（由 Go 团队维护）：

安装 VS Code

安装扩展：搜索 “Go”（作者 golang.Go）

首次打开 Go 项目，会提示安装调试/分析工具，按提示安装即可。

调试：在

.vscode/launch.json

里使用

dlv

（扩展会自动管理）。

## 六、常见命令速查

go version # 查看版本 go env # 查看所有 env go env -w KEY=VALUE # 永久写入某个 env go mod init <module> # 初始化模块 go mod tidy # 整理依赖 go get <pkg> # 获取依赖（Go 1.17+ 语义等价于添加到 go.mod） go build # 编译 go run. # 运行当前模块 go test./... # 运行所有测试

文件说明：

go.mod # 模块清单（模块名、直接/必要间接依赖的版本约束） go.sum # 所有已解析依赖版本的校验哈希（保证可重复构建）

| 操作 | 行为 | 建议 |
| --- | --- | --- |
| 只 go get | 下载该包、记录依赖版本到  go.mod  / 校验到  go.sum  ; 但不使用不会保留 | 不常用 |
| 只 import | 代码声明依赖，  go mod tidy  自动添加 | ✅ 推荐方式 |
| go get + import | 手动控制版本号 + 实际使用 | ✅ 控制版本时用 |

#### 1\. 当你 clone（拉取）一个 Go 项目 时：

第一步：不需要立刻

go mod tidy

。通常项目里已经有：这两个文件描述了项目所需的所有依赖和版本。

go.mod

go.sum

直接执行：

go build./... 或 go test./...

Go 会自动下载 go.mod 中列出的所有依赖到本地缓存（

$GOPATH/pkg/mod

）。

go mod tidy

什么时候用？

当你怀疑依赖清单不整洁（比如别人手动改过 go.mod），或者你自己添加 / 删除了 import。

它会清理和同步：

把 用到但没写入 go.mod 的依赖加上；

把 没用的依赖删掉；

更新 go.sum。

#### 2\. 要自己加依赖，怎么做？

#### ✅ 推荐方式：直接 import + go mod tidy

👉 这就是最自然、最自动的方式（几乎所有团队都这么做）。

import "github.com/fatih/color" go mod tidy

Go 会自动：

识别这个包；

下载对应依赖；

更新

go.mod

与

go.sum

。

⚙️ 可选方式：go get 显式指定版本

如果你想精确控制版本（比如锁定某个 tag）：

go get github.com/fatih/color@v1.16.0 go mod tidy （可选）

区别在于：

go get

→ 你手动指定版本；

import + tidy

→ Go 自动选择兼容版本（通常是最新）。

#### 🧠 三、总结成一个简单的心智模型

| 场景 | 该做什么 | 说明 |
| --- | --- | --- |
| ✅ 刚 clone 下来 | go build  或  go test | 自动下载依赖，不必手动 tidy |
| 🧹 想清理或同步依赖 | go mod tidy | 保持 go.mod/go.sum 干净 |
| ➕ 想添加依赖 | 直接  import  然后  go mod tidy | 自动下载并更新清单 |
| 🎯 想固定版本 | go get 包@版本  然后  go mod tidy | 显式版本控制 |
| 🗑 想删除依赖 | 删除 import →  go mod tidy 或者 go get 包@none | 自动移除无用依赖 |

## 八、升级/卸载

官方 pkg 升级：直接安装新版本

.pkg

即可覆盖。

卸载（官方 pkg）：

sudo rm -rf /usr/local/go sudo rm -f /etc/paths.d/go # 如果存在 sed -i '' '/\\/usr\\/local\\/go\\/bin/d' ~/.zshrc

Homebrew 卸载：

brew uninstall go