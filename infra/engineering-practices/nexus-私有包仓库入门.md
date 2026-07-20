# Nexus 私有包仓库入门：从换个源到发私有包（做中带学）

> [!abstract] Nexus 只是「包仓库管理器」这一类工具的一个实例，同样的模型也适用于 Artifactory、Verdaccio，乃至任何语言生态。

![[nexus-repo-model.png|760]]

---

## 第 1 步 · 换一个源，装个包

**做**：随便找个项目，把装包命令的源地址临时换成 Nexus，装一个包：

```bash
npm install --registry https://nexus.plaud.cn/repository/npm-group/ lodash
```

第一次装可能慢一两秒；删掉 `node_modules` 再装一次，**几乎瞬间完成**。同一个包，第二次明显更快——这就是 Nexus 在中间做了缓存。

**现学**：

- **registry（源地址）**：每个包管理器都有一个「去哪里下包」的旋钮。npm 默认指向 `registry.npmjs.org`，`--registry` 把它临时改指向 Nexus。**Nexus 不是新的包源，而是坐在客户端和公网源中间的一层**。
- **包仓库管理器（repository manager）** 存在的理由，就是这一层带来的四个好处：
  - **快**：第一次从公网拉取后缓存到内网，之后所有人都从内网拿。
  - **稳**：公网源抽风或被墙，只要缓存里有，照样能装。
  - **私有**：能存放公网没有、只属于团队的包。
  - **可控**：所有依赖走一个入口，便于审计和统一管控。
- **泛化**：这套「中间缓存层」的思路与语言无关。pip 有 `--index-url`、Maven 有 `<repository>`、Cargo 有 `registry`、Docker 有 registry mirror——旋钮名字不同，换的都是同一个「源地址」。

> [!tip] pnpm 与 npm 共用一套配置
> pnpm 读取的也是 `.npmrc` 里的 `registry`，下面所有 npm 的配置对 pnpm 同样生效，不必单独设置。

---



## 第 2 步 · 看懂那串 URL

第 1 步的地址是 `https://nexus.plaud.cn/repository/npm-group/`。这个结构不是随便写的。

**做**：把地址拆成三段看：

```
https://nexus.plaud.cn  /repository/  npm-group
└─── Nexus 服务地址 ──┘ └─固定前缀─┘ └─仓库名─┘
```

**现学**：

- 一个 Nexus 里可以有**很多个仓库（repository）**，每个仓库有自己的名字，访问地址就是 `服务地址/repository/<仓库名>/`。
- 仓库名遵循 `<格式>-<角色>` 的命名习惯：
  - **格式（format）**：`npm`、`pypi`、`docker`、`maven`…… 同一个 Nexus 能**同时管理多种格式**的包。所以会看到 `npm-group`、`pypi-group` 并存。
  - **角色**：`group` / `proxy` / `hosted`——这是下一步的主角。
- **泛化**：换成 pip 就是把地址填进 `--index-url`，pypi 的地址还要多一段 `/simple/`（见第 4 步）。仓库名换成 `pypi-group`，其余规律完全一致。

---



## 第 3 步 · 三种仓库：proxy / hosted / group（核心）

第 2 步末尾的三个「角色」，是理解 Nexus 的**唯一关键**。搞懂这三者的分工，整套东西就通了。

**做**：对照上方架构图，记住这三种仓库各干一件事：


| 角色         | 干什么              | 里面装的包                                    |
| ---------- | ---------------- | ---------------------------------------- |
| **proxy**  | 公网源的缓存镜像         | 从 `npmjs.org` / `pypi.org` 拉下来缓存的**公网包** |
| **hosted** | 自家私有库            | 团队自己发布、公网**没有**的私有包                      |
| **group**  | 把上面两者聚合成**一个入口** | 不装包，只做转发                                 |


**现学**：

- **proxy** 照着一个上游公网源做缓存。请求一个包时，缓存里有就直接返回，没有就去上游拉一份、缓存下来再返回。它只读、只镜像公网。
- **hosted** 是纯本地存储，装的是团队私有包。公网上根本搜不到这些包，只有 hosted 里有。
- **group** 本身不存包，它把若干个 proxy 和 hosted **串成一条查找链**：一个请求进来，按成员顺序逐个查，命中即返回。习惯上让 **hosted 排在 proxy 前面**——这样私有包优先，也能避免公网上出现同名包时被「抢答」。
- **所以装包永远用 group**：一个地址就同时覆盖了「公网包（走 proxy 缓存）」和「私有包（走 hosted）」，客户端无需关心某个包到底是公网的还是私有的。
- **泛化**：proxy / hosted / group（或叫 remote / local / virtual）是 Nexus、Artifactory 这类工具的**通用三件套**。记住一句话：**proxy 管「进来的」，hosted 管「自产的」，group 管「统一出口」。**

> [!warning] 别把角色用混
> **装包用 group，发布用 hosted**，两者不能互换。往 group 发布会失败（group 不存包）；只配 proxy 装包则装不到私有包（proxy 里没有）。第 5 步发布时会再撞到这条。

---



## 第 4 步 · 把配置固化下来（.npmrc / pip.conf）

第 1 步每次都手敲 `--registry` 太累。把源地址写进配置文件，之后 `npm install` / `pip install` 自动走 Nexus。

**做**：npm 在**项目根目录**建 `.npmrc`：

```ini
registry=https://nexus.plaud.cn/repository/npm-group/
```

pip 在项目根目录建 `pip.conf`（Windows 下叫 `pip.ini`）：

```ini
[global]
index-url = https://nexus.plaud.cn/repository/pypi-group/simple/
trusted-host = nexus.plaud.cn
```

之后直接 `npm ci` / `pip install -r requirements.txt`，无需再带源地址参数。

**现学**：

- **配置发现有优先级**：命令行参数 > 项目级配置 > 用户级配置（`~/.npmrc`）> 全局配置。项目级 `.npmrc` 提交进仓库，团队所有人和 CI 就自动统一走 Nexus，这是最常用的做法。
- `**pypi` 地址结尾的 `/simple/**` 不是可选后缀：它是 Python 的 [PEP 503 Simple Index](https://peps.python.org/pep-0503/) 约定，pip 就认这个路径。npm 的 group 地址则不需要额外后缀。
- `**trusted-host` 为什么要单独写**：pip 默认要求 HTTPS 且校验证书。内网 Nexus 若走 HTTP、或用自签证书，pip 会拒连；`trusted-host` 让 pip 对这个域名跳过该校验。**仅内网可信地址才这么配**，公网源不要加。
- **泛化**：每个生态都有「配置文件 + 优先级」这一套。找到它的配置文件（`.npmrc` / `pip.conf` / `settings.xml` / `.cargo/config.toml`），把源地址写进去，就完成了从「临时」到「固化」的升级。

---



## 第 5 步 · 发布一个私有包（hosted + 认证）

前四步都在**装**包（读）。现在反过来：把自己的包**发布**上去（写）。写要登录。

**做**：npm 发布走 `npm-hosted`，先配好认证再 publish：

```bash
# 1. 生成 base64 凭证：把「用户名:密码」编码成一串
echo -n "deploy:<deploy-密码>" | base64
# 2. 把凭证写进 ~/.npmrc（针对 hosted 这个地址）
echo "//nexus.plaud.cn/repository/npm-hosted/:_auth=<上一步的 base64>" >> ~/.npmrc
# 3. 发布
npm publish --registry https://nexus.plaud.cn/repository/npm-hosted/
```

pip 发布用 `twine`，走 `pypi-hosted`：

```bash
pip install twine
twine upload \
  --repository-url https://nexus.plaud.cn/repository/pypi-hosted/ \
  -u deploy -p '<deploy-密码>' \
  dist/*
```

**现学**：

- **读匿名、写认证**：从 proxy / group 装公网缓存包不需要身份——那本就是公开内容。但往 hosted **写入**必须证明身份，否则任何人都能污染团队私有库。这是仓库管理器的基本权限模型。
- `**_auth` 是什么**：npm 用 HTTP Basic Auth，凭证格式是 `base64("用户名:密码")`。`//host/path/:_auth=xxx` 表示「访问这个地址时带上这串凭证」。`echo -n` 的 `-n` 不能省，否则会把换行也编码进去导致认证失败。
- **发布走 hosted，安装走 group**（呼应第 3 步）：发布进 hosted 后，因为 group 把 hosted 串在查找链里，别人用 group 地址就能自动装到这个私有包，不必知道它在 hosted 里。
- **泛化**：几乎所有 artifact 仓库都是这个模式——公开读、认证写、Basic Auth 用 base64 编码凭证。

> [!warning] base64 不是加密，等同明文
> `base64(user:password)` 只是编码，**反解即得原始密码**。所以这串 `_auth` 和密码本身一样敏感，绝不能提交进代码仓库。第 6 步专门解决这个问题。

---



## 第 6 步 · 凭证不落地：用 GitHub Secrets 注入

第 5 步的密码/凭证若写死在文件里并提交，就等于公开了私有库的写权限。CI 里的正确做法：凭证存进 GitHub Secrets，运行时才注入。

**做**：把凭证存到仓库的 **Settings → Secrets and variables → Actions**，CI 里用 `${{ secrets.* }}` 引用：

```yaml
# npm 发布：用预先算好的 base64 凭证
- name: Publish npm package
  env:
    NEXUS_AUTH: ${{ secrets.NEXUS_DEPLOY_AUTH }}   # 值 = base64(deploy:密码)
  run: |
    echo "//nexus.plaud.cn/repository/npm-hosted/:_auth=${NEXUS_AUTH}" >> ~/.npmrc
    npm publish --registry https://nexus.plaud.cn/repository/npm-hosted/

# pip 发布：twine 直接吃用户名/密码
- name: Publish Python package
  run: |
    pip install twine
    twine upload \
      --repository-url https://nexus.plaud.cn/repository/pypi-hosted/ \
      -u deploy -p ${{ secrets.NEXUS_DEPLOY_PASSWORD }} \
      dist/*
```

**现学**：

- **Secrets 注入**：凭证只存在于 CI 运行时的环境变量里，不进代码、不进日志（GitHub 会自动把 secret 值在日志里打码）。这与 GitHub Actions 笔记里「密钥绝不硬编码」是同一条原则，参见 [[github-actions-触发机制入门]] 第 6 步。
- **三个 secret 各有用途**：不同工具吃不同形态的凭证，所以常备三份——
  - `NEXUS_DEPLOY_USERNAME` / `NEXUS_DEPLOY_PASSWORD`：给 twine 这类要「用户名 + 密码」的工具。
  - `NEXUS_DEPLOY_AUTH`：给 npm，值是预先算好的 `base64(user:password)`，直接拼进 `.npmrc`。
- **泛化**：凡是 CI 里要用的密码、token、云 key，一律走 secret 注入，不区分工具、不区分平台。

---



## 第 7 步 · 就近访问：多区域实例（长成 Plaud 的样子）

到这里单区域已经完全能用。但 Plaud 的 Runner 分布在 US 和 CN 两地，跨境拉包慢，于是**每个区域各部署一套 Nexus**，CI 按 Runner 所在区域自动选就近的那套。

**做**：CI 里根据 Runner 名字判断区域，动态设置 Nexus 地址：

```yaml
jobs:
  build:
    steps:
      - name: Set Nexus registry by runner
        run: |
          if [[ "$RUNNER_NAME" == *"cn"* ]]; then
            echo "NEXUS_HOST=nexus.plaud.cn" >> $GITHUB_ENV
          else
            echo "NEXUS_HOST=nexus.nicebuild.click" >> $GITHUB_ENV
          fi
      - name: npm install
        run: npm install --registry https://$NEXUS_HOST/repository/npm-group/
      - name: pip install
        run: |
          pip install \
            --index-url https://$NEXUS_HOST/repository/pypi-proxy/simple/ \
            --trusted-host $NEXUS_HOST \
            -r requirements.txt
```

**到这里，就是 Plaud CI 里 Nexus 配置的真实形态了。**

**现学**：

- **为什么要开第二套实例**：CN 区从境外 `npmjs.org` / `pypi.org` 拉包受出境带宽限制，**首次**拉取慢到约 200 KB/s。就近部署一套 Nexus，让本区域的包缓存在本区域。
- **「同一个包只慢一次」**：慢只发生在 proxy 缓存未命中、需要回源公网的**第一次**。缓存命中后从本地 PVC 直接返回，速度恢复正常——这正是第 1 步观察到的现象，在跨境场景下被放大。
- **内网访问**：两个地址都是内网地址（走 `nginx-internal` / `nginx-pvt`），需在 VPN 或集群内部才能访问，公网打不开。这既是安全边界，也解释了为何只有 Runner 和内网机器能用。
- **泛化**：**就近部署 + 缓存**是跨区域 CI 提速的通用套路。无论 Nexus 还是任何镜像服务，思路都是「把远端内容缓存到离消费者最近的地方，只付一次回源成本」。

---



## Python 实战：发布并装回一个私有包

前面每步都是零散片段。这一节把它们串成一次完整的 Python 操作：**建一个包 → 发布到** `pypi-hosted` **→ 从** `pypi-group` **装回来**。跑完正好走完开头那张图的四条箭头（装包读取 / 私有命中 / 回源拉取 / 发布写入）。

### 1 · 装依赖时先走 Nexus（读路径）

建包要用到 `build`、`twine` 两个工具，装它们本身就先验证一次 proxy 缓存。用环境变量把 pip 的源指向 Nexus，比改配置文件更适合演示和 CI：

```bash
export PIP_INDEX_URL=https://nexus.plaud.cn/repository/pypi-group/simple/
export PIP_TRUSTED_HOST=nexus.plaud.cn

pip install build twine
```

`PIP_INDEX_URL` / `PIP_TRUSTED_HOST` 是第 4 步 `pip.conf` 里 `index-url` / `trusted-host` 的环境变量等价物，优先级更高，用完 `unset` 即可，不污染全局。装 `build`、`twine` 这两个公网包，走的就是图里「group → proxy →（首次）上游」这条链。

### 2 · 建一个最小的包

目录结构（`src` 布局，PEP 621 标准）：

```text
my-nexus-demo/
├── pyproject.toml
└── src/
    └── my_nexus_demo/
        └── __init__.py
```

`pyproject.toml`——声明包元数据和构建后端：

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-nexus-demo"
version = "0.1.0"
description = "一个用于验证 Nexus 私有仓库的最小示例包"
requires-python = ">=3.8"
```

`src/my_nexus_demo/__init__.py`——一个能被 import 的函数，装回来后用它验证：

```python
def greet(name: str) -> str:
    """返回一句问候语，用于验证私有包已成功从 Nexus 安装。"""
    return f"Hello from my-nexus-demo, {name}!"
```

构建产物：

```bash
cd my-nexus-demo
python -m build
# 生成 dist/my_nexus_demo-0.1.0-py3-none-any.whl（wheel）
#      dist/my_nexus_demo-0.1.0.tar.gz（sdist 源码分发）
```

**现学**：`python -m build` 产出两种分发格式——**wheel**（`.whl`，预构建、装得快，pip 优先用）和 **sdist**（`.tar.gz`，源码包，装时现场构建）。发布时两个一起传上去，让不同环境各取所需。

### 3 · 发布到 pypi-hosted（写路径，需认证）

`twine` 原生读取 `TWINE_*` 环境变量，凭证不落文件、天然适配 CI：

```bash
export TWINE_REPOSITORY_URL=https://nexus.plaud.cn/repository/pypi-hosted/
export TWINE_USERNAME=deploy
export TWINE_PASSWORD="$NEXUS_DEPLOY_PASSWORD"   # 从 GitHub Secrets 注入，勿写明文

twine upload dist/*
```

**注意目标是** `pypi-hosted` **不是** `pypi-group`（呼应第 3、5 步：发布只能进 hosted）。这一步走的是图里那条紫色「发布写入」箭头——唯一需要认证的路径。

> [!warning] hosted 默认禁止覆盖同版本
> 私有仓库里的包一经发布即**不可变**：再传一次 `0.1.0` 会被拒（`400 Repository does not allow updating assets`）。改了代码要发新版，必须先把 `pyproject.toml` 的 `version` 抬到 `0.1.1`，重新 `python -m build` 再传。这是所有 artifact 仓库的通用约束——保证「同一个版本号永远是同一份内容」。



### 4 · 从 pypi-group 装回来（私有命中）

换一个干净环境（新 venv），用 **group** 地址安装刚发布的私有包：

```bash
python -m venv /tmp/verify && source /tmp/verify/bin/activate

pip install \
  --index-url https://nexus.plaud.cn/repository/pypi-group/simple/ \
  --trusted-host nexus.plaud.cn \
  my-nexus-demo

python -c "import my_nexus_demo; print(my_nexus_demo.greet('Nexus'))"
# -> Hello from my-nexus-demo, Nexus!
```

**关键点**：安装地址是 **group**，但装到的是 hosted 里的私有包。因为 group 的查找链把 hosted 串在前面（图里「① 私有优先」），客户端**无需知道这个包在 hosted**——公网包和私有包用同一个地址装，这正是 group 存在的意义。

### 5 · 放进 GitHub Actions

把上面的发布搬进 CI，凭证全走 Secrets：

```yaml
- name: Build & publish to Nexus
  env:
    TWINE_REPOSITORY_URL: https://nexus.plaud.cn/repository/pypi-hosted/
    TWINE_USERNAME: ${{ secrets.NEXUS_DEPLOY_USERNAME }}
    TWINE_PASSWORD: ${{ secrets.NEXUS_DEPLOY_PASSWORD }}
    PIP_INDEX_URL: https://nexus.plaud.cn/repository/pypi-group/simple/
    PIP_TRUSTED_HOST: nexus.plaud.cn
  run: |
    pip install build twine
    python -m build
    twine upload dist/*
```

一次跑完，四条箭头全走到：装 `build`/`twine` 走读路径，`twine upload` 走写路径，别的仓库再 `pip install my-nexus-demo` 就能装到——完整闭环。

---



## 回看 Plaud 全链路

把 7 步串起来，一次 CI 装包/发包的真实路径：

1. **选就近实例**：按 `RUNNER_NAME` 判断区域 → US 走 `nexus.nicebuild.click`，CN 走 `nexus.plaud.cn`。
2. **装包走 group**：`.../repository/npm-group/`（或 `pypi-group/simple/`）。group 先查 hosted（私有包），再查 proxy（公网缓存），一个地址通吃。
3. **命中即快**：proxy 缓存命中直接从内网 PVC 返回；未命中才回源公网，缓存后「只慢一次」。
4. **发包走 hosted**：私有包 `npm publish` / `twine upload` 到 `npm-hosted` / `pypi-hosted`，凭证由 GitHub Secrets 注入。
5. **发完即可被装**：hosted 已挂在 group 的查找链上，别人用 group 地址即可装到刚发布的私有包。

---



## 收尾自查清单

在自己项目接入 Nexus 时，按顺序确认：

- [ ] 装包地址用的是 **group**（`npm-group` / `pypi-group/simple/`），而不是 proxy 或 hosted？
- [ ] pip 配了 `trusted-host`，且仅对内网 Nexus 地址配？
- [ ] pypi 地址结尾带 `/simple/`？
- [ ] 项目级 `.npmrc` / `pip.conf` 已提交，团队和 CI 自动统一？
- [ ] 发布走 **hosted**，且凭证走 **Secrets 注入**、没有明文进仓库？
- [ ] `echo -n` 生成 base64 时没把换行编码进去？
- [ ] 多区域场景按 Runner 区域选了就近实例？

> [!note] 运维参考：这套 Nexus 的基础设施信息
> 使用者不必关心，运维接手时用得上。
>
> - **部署**：Helm chart `sonatype/nexus-repository-manager` v61.0.2，App 版本 Nexus 3.61.0，`nexus` namespace。
> - **US 区**：`prod-iops-eks`（us-west-2），地址 `nexus.nicebuild.click`，镜像 `sonatype/nexus3:3.61.0`。
> - **CN 区**：`eks-staging-cnnw1`（cn-northwest-1），地址 `nexus.plaud.cn`，镜像走 ECR 内网 `470515048733.dkr.ecr.cn-northwest-1.amazonaws.com.cn/public/sonatype/nexus3:3.61.0`。
> - **资源**：gp3 50Gi PVC 存储，内存 Request 2Gi / Limit 4Gi。
> - **admin 初始密码**：在 Pod 内 `/nexus-data/admin.password`，`kubectl -n nexus exec deploy/nexus-nexus-repository-manager -- cat /nexus-data/admin.password` 获取，首次登录后需修改。
> - **deploy 账号**：US / CN 共用一套，权限 add / edit / read / browse（无 delete）。**密码不写进本笔记**，见 GitHub Secrets（`NEXUS_DEPLOY_`*）或问运维。

