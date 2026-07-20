# GitHub Actions 入门：从零配一条 CI（做中带学）

> [!abstract] 这份笔记怎么用
> 不讲空泛概念。跟着 7 步动手：**每步先改一段配置、push 看结果，再理解这步用到的知识点**。一条 workflow 从 5 行起步，一步步长成 plaud-summary 的真实 CI。
> 边看边在自己仓库跟着敲，效果最好。

![[ghactions-learn-path.png|738]]

---

## 第 1 步 · 让它自动跑起来

**做**：在仓库里新建文件 `.github/workflows/ci.yaml`，写下面这段，然后 `git push`：

```yaml
name: CI                    # 这条流水线的显示名
on: push                    # 触发条件：任何 push 都触发
jobs:
  hello:                    # 一个作业，名字随便起
    runs-on: ubuntu-latest  # 找一台 Ubuntu 机器来跑
    steps:
      - run: echo "跑起来了！"   # 执行一条命令
```

push 完，打开仓库网页的 **Actions 标签页**，你会看到这条流水线**自动**开始跑，几秒后一个绿勾。**没有任何人点按钮**——这就是「自动触发」。

**现学**：

- `on:` 是触发开关。`on: push` 的意思是「只要有人往这个仓库 push，就跑我」。GitHub 一直在监听仓库事件，事件命中 `on:` 就自动唤醒这条流水线。这是理解「为什么会自动跑」的关键：不是有人主动跑脚本，是**你声明了触发条件，GitHub 帮你盯着**。
- `jobs:` 下面是一个个作业（job）。上面只有一个叫 `hello`。
- `runs-on:` 给这个 job 指定一台运行的机器（叫 runner），`ubuntu-latest` 是 GitHub 免费提供的。
- `steps:` 是 job 里一条条按顺序执行的步骤；`run:` 就是在那台机器的终端里敲一条命令。
- `name:` 是显示在 Actions 页上的名字。

> [!note] 文件放哪、叫什么（第一步就要记住）
>
> - **位置是死规定**：必须放在仓库根目录的 `.github/workflows/` 下，GitHub 只扫描这一层，放进子文件夹（如 `.github/workflows/ci/x.yaml`）**不会被识别**。扩展名 `.yml` / `.yaml` 都行。
> - **文件名随意**：触发只看文件里的 `on:`，跟文件名毫无关系。`ci.yaml` 改名成 `banana.yaml` 行为不变。名字纯粹给人看，习惯按职责命名：`ci.yaml`（构建测试）、`release.yaml`（发布）、`lint.yaml`（检查）。
> - **一个仓库可放多个文件**，每个都是独立流水线、各有各的 `on:`。plaud 就有 4 个（`ci.yaml`、`sync-to-envs-on-label.yml` 等）。

---



## 第 2 步 · 让 runner 拿到你的代码

第 1 步那台机器是**全新空白的**，上面根本没有你的代码。想让它 `ls` 出你的文件会是空的。

**做**：在 `steps:` 最前面加一步，把代码拉下来：

```yaml
    steps:
      - uses: actions/checkout@v4   # 把本仓库代码拉到这台机器上
      - run: ls                     # 现在能看到你的项目文件了
```

**现学**：

- runner 每次都是**干净的临时机器**，跑完就销毁。所以每条流水线开头几乎都要先「拉代码」。
- `uses:` 是和 `run:` 并列的另一种步骤写法：它不执行你的命令，而是**复用一个别人写好的现成模块**，这个模块叫 **Action**。`actions/checkout` 就是官方的「拉代码」Action。
- `@v4` 是版本号。引用别人的 Action 一定要锁版本，否则上游更新可能哪天把你的流水线搞崩。

> [!tip] uses vs run，一句话区分
> `run:` = 自己敲命令；`uses:` = 拿别人封装好的能力（拉代码、登录云、装环境……）。能 `uses` 的就别自己 `run`，省事又稳。

---



## 第 3 步 · 控制「什么时候跑」

第 1 步的 `on: push` 太宽了——改个 README 也触发、任何分支都触发。收窄它。

**做**：把 `on:` 改成带过滤的写法：

```yaml
on:
  push:
    branches: [main, release]        # 只有 main、release 分支的 push 才触发
    paths-ignore: ["**.md"]          # 只改 .md 文档时不触发
```

**现学**：

- `branches` / `paths` 是过滤器，让流水线只在**该跑的时候**跑，省时间省额度。
- `on:` 支持的事件远不止 push。现在你懂了 `on:` 的机制，再认识几个常用的就不抽象了：


| 事件                  | 什么时候触发               | 典型用途  |
| ------------------- | -------------------- | ----- |
| `push`              | 推送代码                 | 构建、测试 |
| `pull_request`      | 开 PR / PR 有新提交 / 打标签 | PR 检查 |
| `schedule`          | cron 定时              | 每晚跑回归 |
| `workflow_dispatch` | 网页上手动点按钮             | 手动补跑  |


多个事件可以并列，任一命中都触发：

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:      # 同时允许手动触发，Actions 页会多个 "Run workflow" 按钮
```

> [!note] plaud 的实际写法
> plaud 的 [ci.yaml](该项目根目录) 用的是 `branches: ["**"]`——`"**"` 是通配符，匹配所有分支名，意思是「任何分支 push 都构建」。它故意开得宽，因为每个环境分支的 push 都要出镜像。

---



## 第 4 步 · 拆成多个作业

一条流水线常常有多件事：先读配置、再构建、再部署。把它们拆成多个 job。

**做**：拆成两个 job，用 `needs` 让它们排队：

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "构建中..."
  test:
    needs: build          # 等 build 跑完，test 才开始
    runs-on: ubuntu-latest
    steps:
      - run: echo "测试中..."
```

**现学**：

- 多个 job **默认并行**，而且**各自占一台独立的 runner**（互不影响，也互不看见对方的文件）。
- `needs:` 声明依赖，把并行改成**串行**：`test` 写了 `needs: build`，就会乖乖等 `build` 成功后再跑。
- 对比记牢：**job 之间默认并行**（除非 needs），**step 在 job 内部永远顺序**执行。

---



## 第 5 步 · 让作业之间传数据

`build` job 算出个东西，`test` job 想用——但它俩在**两台不同机器**上，文件传不过去。怎么办？用 `outputs`。

**做**：上游 job 声明 `outputs`，下游 job 用 `needs.<job>.outputs.<key>` 读：

```yaml
jobs:
  cfg:
    runs-on: ubuntu-latest
    outputs:
      images: ${{ steps.j.outputs.images }}   # 把 j 这步的输出，暴露成 job 的输出
    steps:
      - uses: actions/checkout@v4
      - id: j                                  # 给这步起个 id，方便引用
        run: echo "images=$(jq -c '.images' ci.json)" >> "$GITHUB_OUTPUT"
  build:
    needs: cfg
    runs-on: ubuntu-latest
    steps:
      - run: echo "拿到清单：${{ needs.cfg.outputs.images }}"
```

**现学**：

- job 之间**不共享文件系统、不共享内存**。跨 job 传值，唯一正道就是 `outputs`。
- 步骤里往特殊文件 `$GITHUB_OUTPUT` 写 `key=value`，再配合这步的 `id`，值就能被 job 的 `outputs` 引用。
- 这段几乎就是 plaud `cfg` job 的真实代码：它读根目录 `ci.json`，把「要构建哪些镜像」的清单输出给下游。

---



## 第 6 步 · 真构建镜像 + 用密钥

现在干真活：构建 Docker 镜像并推到镜像仓库。推送要登录，登录要密钥——密钥绝不能写进文件。

**做**：

```yaml
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/amazon-ecr-login@v2      # 登录 ECR（用到密钥）
      - run: |
          TAG=${GITHUB_SHA::7}                      # 取本次 commit 短 SHA 当 tag
          docker build -t $ECR:$TAG -f Dockerfile .
          docker push  $ECR:$TAG
        env:
          ECR: ${{ secrets.ECR_URL }}              # 从 secrets 取值，不写明文
```

**现学**：

- `secrets`：凭证（token、密码、云 key）放在仓库网页的 **Settings → Secrets and variables → Actions**，yaml 里用 `${{ secrets.名字 }}` 引用。**任何情况下都不硬编码进文件**。
- `${{ ... }}` 是表达式语法，用来取变量 / secret / 上一步的输出。
- **产物的 tag**：这里用 `${GITHUB_SHA::7}`，即本次 commit 的 SHA 前 7 位（如 `343186e`）。`GITHUB_SHA` 是 GitHub 自动注入的环境变量。最终镜像地址形如 `<仓库地址>:343186e`——**这个短 SHA 就是「镜像 ID」**，后面部署时要用它。

---



## 第 7 步 · 复用组织级流水线（长成 plaud 的样子）

第 6 步的构建逻辑，如果每个仓库都复制一遍，维护会崩。plaud 的做法：**把构建逻辑写在一个公共仓库里，各仓库调用它**。

**做**：把本仓库的构建 job 换成「调用外部 workflow」：

```yaml
jobs:
  cfg:                                   # 还是第 5 步那个：读 ci.json 输出清单
    runs-on: ubuntu-latest
    outputs:
      images: ${{ steps.j.outputs.images }}
    steps:
      - uses: actions/checkout@v4
      - id: j
        run: echo "images=$(jq -c '.images' ci.json)" >> "$GITHUB_OUTPUT"
  ci:
    needs: cfg
    uses: Plaud-AI/plaud-ci-workflows/.github/workflows/docker-build-push.yaml@main
    with:
      images: ${{ needs.cfg.outputs.images }}   # 把清单传进去
    secrets: inherit                            # 把本仓库的密钥继承给它
```

**到这里，就是 plaud 真实** `ci.yaml` **的完整形态了。**

**现学**：

- `workflow_call`：一条 workflow 想「被别人调用」，就在自己顶部声明 `on: workflow_call`。被调用方 `docker-build-push.yaml` 的开头正是：
  ```yaml
  on:
    workflow_call:
      inputs:
        images:
          required: true
          type: string
  ```
- 调用方用 `uses:` 指向它（`组织/仓库/.github/workflows/文件@分支`），用 `with:` 传参数，用 `secrets: inherit` 把自己的密钥一并交给它。
- 好处：构建逻辑写一份，全组织共享。改一次，所有仓库生效。这是流水线工程化的核心手段。

---



## 收尾加固（生产级流水线常配）

上面 7 步已经能跑。真正上生产前，再按需加这几条：

```yaml
concurrency:                          # 连续 push 时，自动取消同分支的旧运行，避免堆积
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:                          # 最小权限：默认收紧，只给必要的
  contents: read
```


| 加固项  | 关键字             | 收益                  |
| ---- | --------------- | ------------------- |
| 防重复跑 | `concurrency`   | 新提交自动取消旧运行          |
| 最小权限 | `permissions`   | 收紧 token 权限，更安全     |
| 锁定版本 | `uses: ...@v4`  | 防第三方 Action 更新炸掉流水线 |
| 缓存依赖 | `actions/cache` | 装依赖秒级完成             |


---



## 回看 plaud 全链路

把 7 步串起来，plaud 从 push 到上线的真实流程：

1. **push 触发**：任意分支 push，命中 `on: push`，CI 唤醒。
2. **cfg job**：读 `ci.json`，把镜像清单 `outputs` 出去。
3. **ci job**：`needs: cfg` 拿清单，`uses:` 复用组织级 workflow，在 runner 上 `docker build -f <Dockerfile>` 并 push。
4. **产物**：镜像 tag = commit 短 SHA（如 `343186e`），地址形如 `<ecr>:343186e`。
5. **到部署**：这个短 SHA 目前需**人工**填进 deploy 仓库对应环境的配置里，才完成上线。

> [!note] 配套：标签驱动的链式反应
> plaud 另有一条 [sync-to-envs-on-label.yml](该项目根目录)：给发往 `main` 的 PR 打标签（`🏗️Testing in DEV` / `🚧Testing in PRE-RELEASE`），自动把 PR 分支 merge 进 `develop` / `release`。这两个分支的 push 又触发上面的 CI——**打标签(事件) → 同步流水线 merge 出 push → push(事件) → CI 构建**，事件一环扣一环。

---



## 收尾自查清单

在自己仓库配完，按顺序确认：

- [ ] 文件在 `.github/workflows/` 下、`.yaml`/`.yml` 结尾？
- [ ] `on:` 的事件和 `branches`/`paths` 过滤符合预期，没开太宽？
- [ ] 开头有 `checkout` 把代码拉下来？
- [ ] 要顺序的 job 加了 `needs`；跨 job 传值用了 `outputs`？
- [ ] 密钥都走 `secrets`，没有明文？
- [ ] 第三方 `uses:` 锁了版本？
- [ ] 配了 `concurrency` 防重复跑？

> [!tip] 调试技巧
> workflow 写错不会提前报错，只在 Actions 页运行时失败。改 `on:` 后先小 push 一次验证是否触发；点开每个 step 看实时日志定位问题；加上 `workflow_dispatch` 就能在网页上手动反复试跑，不用靠一次次 push。

