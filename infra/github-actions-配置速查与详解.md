# GitHub Actions 配置速查与详解

> [!abstract] 这份笔记怎么用
> 这份是参考手册：先用一张**全字段带注释的配置**建立全貌，再给**最小可执行单元**，最后**自顶向下逐层拆解每个字段**。查某个字段直接跳「字段详解」对应小节。

---

## 一、常用配置全貌（带字段注释）

```yaml
name: CI                                  # ① workflow 显示名，出现在 Actions 页
run-name: CI by ${{ github.actor }}       # ② 本次运行的动态标题（可省）

on:                                       # ③ 触发条件（事件 + 过滤器）
  push:
    branches: [main, release]             #    只有这些分支的 push 才触发
    paths-ignore: ["**.md"]               #    只改 .md 文档时不触发
  pull_request:
    branches: [main]                      #    针对 main 的 PR 触发
  workflow_dispatch:                      #    允许在网页上手动触发

permissions:                              # ④ 本次运行 token 的权限（默认收紧）
  contents: read

concurrency:                              # ⑤ 并发控制：同组新运行取消旧运行
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:                                      # ⑥ 全局环境变量（所有 job/step 可见）
  REGISTRY: ghcr.io

jobs:                                     # ⑦ 作业集合（一个 workflow 含 ≥1 个 job）
  build:                                  #    job id，自定义
    runs-on: ubuntu-latest                # ⑧ 指定运行机器（runner）
    outputs:                              # ⑨ 把 step 输出提升为 job 输出，供下游 job 读
      tag: ${{ steps.meta.outputs.tag }}
    steps:                               # ⑩ job 内按顺序执行的步骤
      - uses: actions/checkout@v4         #    复用现成 Action：拉代码
      - id: meta                          #    给 step 起 id，便于引用其输出
        run: echo "tag=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
      - name: Build image                 #    step 显示名
        run: docker build -t $REGISTRY/app:${{ steps.meta.outputs.tag }} .
        env:                              #    仅本 step 可见的环境变量
          TOKEN: ${{ secrets.REGISTRY_TOKEN }}

  deploy:
    needs: build                          # ⑪ 依赖：等 build 成功后才跑
    if: github.ref == 'refs/heads/main'   # ⑫ 条件：仅 main 分支执行
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploy ${{ needs.build.outputs.tag }}"   # 读上游 job 的 output
```

---



## 二、最小可执行单元

能被 GitHub 识别并跑绿的最短 workflow，只需三层结构：**触发（**`on`**）→ 作业（**`jobs`**）→ 步骤（**`steps`**）**。

```yaml
on: push
jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - run: echo "hi"
```

> [!note] 文件位置是死规定
> 必须放在仓库根目录的 `.github/workflows/` 下，扩展名 `.yml` / `.yaml` 均可。GitHub 只扫描这一层，放进子目录不会被识别。文件名随意，触发只看文件里的 `on:`。

---



## 三、字段详解



### 0. 心智模型：事件驱动 + 临时机器

**GitHub Actions 是事件驱动的**。

- 做的事只有两件：**声明什么事件触发（**`on`**）**、**声明触发后做什么（**`jobs`**）**。
- GitHub 持续监听仓库事件（push、开 PR、定时……）。事件命中某个 workflow 的 `on:`，就自动唤醒它，临时开一台干净机器（runner）执行，跑完销毁。

由此推出两条贯穿全篇的结论：

1. **runner 每次都是全新空白机器**——所以几乎每条流水线开头都要先 `checkout` 拉代码。
2. **不同 job 各占独立 runner，互不共享文件与内存**——所以跨 job 传值必须走 `outputs`（见 §2.3）。

配置文件的字段严格分三层，越往里作用域越小：

```
workflow（整个文件）
├── name / run-name          显示名
├── on                       触发条件         ← §1.2
├── permissions              token 权限       ← §1.3
├── concurrency              并发控制         ← §1.4
├── env                      全局环境变量     ← §1.5
└── jobs
    └── <job_id>             一个作业
        ├── runs-on          指定机器         ← §2.1
        ├── needs            job 间依赖       ← §2.2
        ├── outputs          job 间传值       ← §2.3
        ├── if               条件执行         ← §2.4
        ├── strategy         矩阵并行         ← §2.5
        └── steps
            └── - uses/run   一条步骤         ← §3
```

> [!tip] 作用域记忆法
> 字段写在哪一层，就在哪一层的范围内生效。`env` 写在 workflow 顶层 → 全局可见；写在 job 下 → 仅该 job；写在 step 下 → 仅该 step。`if`、`env` 三层都能写，就近覆盖。

---



### 1. workflow 顶层字段



#### 1.1 `name` / `run-name`

```yaml
name: CI                              # workflow 的名字，显示在 Actions 列表
run-name: Deploy by ${{ github.actor }}   # 每次运行的标题，可插表达式
```

- `name`：纯给人看，不影响任何行为。省略时 GitHub 用文件路径当名字。
- `run-name`：给**单次运行**起动态标题，常用于把触发人、分支写进标题，一眼区分多次运行。可省。



#### 1.2 `on` —— 触发条件（核心）

`on:` 是触发开关，声明「什么事件发生时唤醒这条 workflow」。它有三种写法，从简到繁：

```yaml
on: push                    # 写法一：单事件

on: [push, pull_request]    # 写法二：多事件并列，任一命中都触发

on:                         # 写法三：事件 + 过滤器（最常用）
  push:
    branches: [main]
```

**常用事件一览：**


| 事件                  | 何时触发                 | 典型用途        |
| ------------------- | -------------------- | ----------- |
| `push`              | 推送代码                 | 构建、测试       |
| `pull_request`      | 开 PR / PR 有新提交 / 打标签 | PR 检查       |
| `schedule`          | cron 定时              | 每晚跑回归       |
| `workflow_dispatch` | 网页手动点按钮              | 手动补跑        |
| `workflow_call`     | 被其他 workflow 调用      | 复用流水线（见 §5） |


**过滤器**让流水线只在该跑的时候跑，省时间省额度。`branches` / `tags` / `paths` 各有 `-ignore` 反向版：

```yaml
on:
  push:
    branches: [main, release]     # 分支白名单
    branches-ignore: [tmp/**]     # 分支黑名单（与 branches 二选一）
    paths: ["src/**"]             # 只有改了这些路径才触发
    paths-ignore: ["**.md"]       # 改了这些路径不触发
    tags: ["v*"]                  # 只有推 v 开头的 tag 才触发
```

`workflow_dispatch` 和 `schedule` 各带专属子字段：

```yaml
on:
  workflow_dispatch:
    inputs:                       # 手动触发时弹出的表单参数
      env:
        description: 部署环境
        required: true
        default: staging
        type: choice
        options: [staging, prod]
  schedule:
    - cron: "0 2 * * *"           # 每天 UTC 02:00（分 时 日 月 周）
```

> [!note] 通配符 `"**"`
> `branches: ["**"]` 匹配所有分支名，等价于「任何分支 push 都触发」。故意开宽的场景：每个环境分支都要出镜像。参见 [[github-actions-触发机制入门]] §第 3 步「控制什么时候跑」。



#### 1.3 `permissions` —— token 权限

```yaml
permissions:
  contents: read        # 只读仓库内容
  packages: write       # 可推镜像到 GitHub Packages
```

每次运行 GitHub 自动注入一个临时 token（`GITHUB_TOKEN`），`permissions` 决定它能干什么。**最佳实践是默认收紧到** `read`**，按需放开**，缩小凭证泄漏的爆炸半径。可写在顶层（全 job 生效）或某个 job 下（仅该 job）。

#### 1.4 `concurrency` —— 并发控制

```yaml
concurrency:
  group: ci-${{ github.ref }}     # 同一分支归为一组
  cancel-in-progress: true        # 组内有新运行时，取消正在跑的旧运行
```

连续 push 时会堆积多次运行。`concurrency` 把同组的旧运行自动取消，只留最新一次——省额度，也避免旧构建覆盖新构建。`group` 通常用 `github.ref`（分支/tag 引用）区分组。

#### 1.5 `env` / `defaults`

```yaml
env:                              # 全局环境变量
  REGISTRY: ghcr.io

defaults:                         # 所有 step 的默认设置
  run:
    shell: bash
    working-directory: ./app
```

- `env`：定义环境变量，`run:` 里用 `$REGISTRY` 或 `${{ env.REGISTRY }}` 取。三层都能写，就近覆盖。
- `defaults.run`：统一设定所有 `run` step 的默认 shell、工作目录，免得每步重复写。

---



### 2. job 级字段

`jobs:` 下每个键是一个 job id（自定义）。**多个 job 默认并行，各占独立 runner。**

#### 2.1 `runs-on`

```yaml
runs-on: ubuntu-latest    # GitHub 免费提供的托管 runner
# runs-on: [self-hosted, linux]   # 或指定自托管 runner 的标签
```

给 job 指定运行机器。常用 `ubuntu-latest` / `windows-latest` / `macos-latest`。企业内网构建则用 `self-hosted` 加标签匹配自建机器。

#### 2.2 `needs` —— job 间依赖

```yaml
jobs:
  build: { runs-on: ubuntu-latest, steps: [...] }
  test:
    needs: build            # 等 build 成功后再跑（并行改串行）
  deploy:
    needs: [build, test]    # 等这两个都成功
```

默认并行，`needs` 声明依赖后变串行。**对比记牢：job 之间默认并行（除非** `needs`**），step 在 job 内部永远顺序执行。**

#### 2.3 `outputs` —— job 间传值

因为 job 各占独立机器、不共享文件系统，**跨 job 传值唯一正道是** `outputs`：

```yaml
jobs:
  cfg:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tag }}    # 把 step 输出提升为 job 输出
    steps:
      - id: meta                            # step 需有 id 才能被引用
        run: echo "tag=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
  build:
    needs: cfg
    runs-on: ubuntu-latest
    steps:
      - run: echo "拿到 ${{ needs.cfg.outputs.tag }}"   # 下游用 needs.<job>.outputs.<key> 读
```

链路是两跳：**step 写** `$GITHUB_OUTPUT` **→ 由** `id` **聚合到 job 的** `outputs` **→ 下游 job 经** `needs` **读取**。参见 [[github-actions-触发机制入门]] §第 5 步「让作业之间传数据」。

#### 2.4 `if` —— 条件执行

```yaml
deploy:
  if: github.ref == 'refs/heads/main'   # 仅 main 分支才跑这个 job
```

条件为假则跳过。`if` 也能写在 step 上，控制单步是否执行。表达式里引用 context 不用包 `${{ }}`（`if` 本身就是表达式环境）。

#### 2.5 `strategy` / `matrix` —— 矩阵并行

```yaml
strategy:
  matrix:
    node: [18, 20, 22]        # 自动展开成 3 个并行 job
steps:
  - run: node --version
    # 每个 job 里 ${{ matrix.node }} 分别是 18 / 20 / 22
```

一份 job 定义按矩阵维度展开成多个并行运行，常用于多版本 / 多平台测试。

---



### 3. step 级字段

`steps:` 是一个列表，每个 `-`  是一步，按顺序执行。核心是两种互斥写法：`run` 和 `uses`。

#### 3.1 `run` vs `uses`

```yaml
steps:
  - run: npm test                 # 自己敲命令
  - uses: actions/checkout@v4     # 复用别人封装好的 Action
```

- `run:` = 在 runner 终端里执行命令。多行用 `run: |`。
- `uses:` = 复用一个现成模块（叫 **Action**），如拉代码、登录云、装环境。格式 `所有者/仓库@版本`。
- 口诀：**能** `uses` **的就别自己** `run`，省事又稳。参见 [[github-actions-触发机制入门]] §第 2 步「让 runner 拿到你的代码」。

> [!warning] `uses` 必须锁版本
> `@v4` 是版本号，引用第三方 Action 一定要锁。不锁的话上游哪天更新可能直接把流水线搞崩。更严的做法是锁到 commit SHA。



#### 3.2 step 的其余字段

```yaml
- name: Run tests               # 显示名（可省，省略则显示命令本身）
  id: test                      # 起 id，供后续引用本步输出 / 状态
  uses: actions/setup-node@v4
  with:                         # 给 uses 的 Action 传参
    node-version: 20
  env:                          # 仅本 step 可见的环境变量
    CI: true
  if: success()                 # 条件执行本步
  working-directory: ./app      # 本步的工作目录
  continue-on-error: true       # 本步失败也不让整个 job 失败
```


| 字段                  | 作用                       |
| ------------------- | ------------------------ |
| `name`              | Actions 页上的步骤显示名         |
| `id`                | 给步骤命名以便引用其 `outputs` 或结果 |
| `with`              | 向 `uses` 的 Action 传入参数   |
| `env`               | 本步专属环境变量                 |
| `if`                | 本步是否执行                   |
| `continue-on-error` | 本步失败是否放行                 |


---



### 4. 表达式 `${{ }}` 与上下文

`${{ ... }}` 是表达式语法，用来在 yaml 里取变量、secret、上一步输出、判断条件。

**常用 context（可取的数据来源）：**


| Context     | 取什么                | 例                                        |
| ----------- | ------------------ | ---------------------------------------- |
| `github.*`  | 事件与仓库元信息           | `github.ref`、`github.actor`、`github.sha` |
| `env.*`     | 环境变量               | `env.REGISTRY`                           |
| `secrets.*` | 密钥（见 §4bis）        | `secrets.TOKEN`                          |
| `needs.*`   | 上游 job 的 outputs   | `needs.cfg.outputs.tag`                  |
| `steps.*`   | 同 job 内某步的 outputs | `steps.meta.outputs.tag`                 |
| `matrix.*`  | 矩阵当前维度值            | `matrix.node`                            |
| `inputs.*`  | 手动/被调用时传入的参数       | `inputs.env`                             |


**两个特殊文件**（往里写，值就能被后续引用）：

```yaml
- run: echo "tag=abc" >> "$GITHUB_OUTPUT"   # 写 step 输出，供 steps.<id>.outputs 读
- run: echo "VER=1.2" >> "$GITHUB_ENV"      # 写环境变量，供后续 step 用 $VER 读
```

**常用内置环境变量**（GitHub 自动注入）：


| 变量                 | 含义                                                 |
| ------------------ | -------------------------------------------------- |
| `GITHUB_SHA`       | 本次 commit 的完整 SHA，`${GITHUB_SHA::7}` 取前 7 位当镜像 tag |
| `GITHUB_REF`       | 触发的分支/tag 引用，如 `refs/heads/main`                   |
| `GITHUB_WORKSPACE` | 代码 checkout 到的目录                                   |
| `GITHUB_OUTPUT`    | 写 step 输出的目标文件（见上）                                 |
| `GITHUB_ENV`       | 写环境变量的目标文件（见上）                                     |


---



### 4bis. `secrets` —— 密钥

```yaml
- run: docker login -u user -p "$TOKEN"
  env:
    TOKEN: ${{ secrets.REGISTRY_TOKEN }}    # 从 secrets 取，不写明文
```

凭证（token、密码、云 key）存在仓库网页的 **Settings → Secrets and variables → Actions**，yaml 里用 `${{ secrets.名字 }}` 引用。**任何情况下都不硬编码进文件**。日志里 secret 值会被自动打码。

---



### 5. 可复用 workflow（`workflow_call`）

把构建逻辑抽到公共仓库，各仓库调用，改一次全组织生效。分两侧：

**被调用侧**——顶部声明 `on: workflow_call`，定义入参：

```yaml
# 文件：Plaud-AI/plaud-ci-workflows/.github/workflows/docker-build-push.yaml
on:
  workflow_call:
    inputs:
      images:
        required: true
        type: string
    secrets:              # 声明需要调用方传哪些密钥
      REGISTRY_TOKEN:
        required: true
```

**调用侧**——用 `uses:` 指向它，`with:` 传参，`secrets:` 交密钥：

```yaml
jobs:
  ci:
    uses: Plaud-AI/plaud-ci-workflows/.github/workflows/docker-build-push.yaml@main
    with:
      images: ${{ needs.cfg.outputs.images }}   # 传入参数
    secrets: inherit                             # 把本仓库全部密钥继承过去
```

- 调用格式：`组织/仓库/.github/workflows/文件@分支`。
- `secrets: inherit` 是偷懒写法，把调用方的所有 secret 一并传入；也可 `secrets: { REGISTRY_TOKEN: ${{ secrets.X }} }` 逐个指定。
- 注意 `uses` 在这里写在 **job 层**（复用整条 workflow），区别于 §3.1 写在 step 层（复用一个 Action）。

完整落地形态见 [[github-actions-触发机制入门]] §第 7 步「复用组织级流水线」。

---



## 四、字段速查表


| 层级       | 字段                | 一句话           | 详解    |
| -------- | ----------------- | ------------- | ----- |
| workflow | `name`            | 显示名           | §1.1  |
| workflow | `on`              | 触发条件          | §1.2  |
| workflow | `permissions`     | token 权限      | §1.3  |
| workflow | `concurrency`     | 并发控制          | §1.4  |
| workflow | `env`             | 全局环境变量        | §1.5  |
| job      | `runs-on`         | 指定机器          | §2.1  |
| job      | `needs`           | job 间依赖（串行）   | §2.2  |
| job      | `outputs`         | job 间传值       | §2.3  |
| job      | `if`              | 条件执行          | §2.4  |
| job      | `strategy.matrix` | 矩阵并行          | §2.5  |
| step     | `run`             | 敲命令           | §3.1  |
| step     | `uses`            | 复用 Action     | §3.1  |
| step     | `with`            | 给 Action 传参   | §3.2  |
| step     | `id`              | 命名以便引用        | §3.2  |
| 通用       | `${{ }}`          | 表达式取值         | §4    |
| 通用       | `secrets`         | 密钥            | §4bis |
| 通用       | `workflow_call`   | 复用整条 workflow | §5    |


