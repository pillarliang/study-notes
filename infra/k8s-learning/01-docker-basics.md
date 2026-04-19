# Docker 学习笔记

> **学习建议**：首次阅读重点关注 **第 1–6 节**（基础内容）。标记为 `（进阶）` 的章节在掌握基础后按需阅读，`（深入）` 的章节作为参考资料即可。

---

## 1. 为什么需要 Docker？

### 问题：「在我电脑上能跑」

你写了一个 Python 服务，本地跑得好好的。但部署到服务器时：

- 服务器 Python 版本是 3.9，你本地是 3.12
- 某个依赖需要系统库 `libssl`，服务器没装
- 同事拉代码后跑不起来，因为他 macOS，你是 Linux

每次部署都要手动对齐环境，既费时又容易出错。

### Docker 的解法：把环境打包进去

Docker 把你的代码 + Python 版本 + 所有依赖 + 系统库，**全部打成一个包（镜像）**。这个包在任何机器上跑起来都是同一个环境——无论是你的 Mac、CI 服务器还是 AWS EKS。

```
没有 Docker：代码 → 手动配环境 → 跑起来（祈祷）
有了 Docker：代码 + 环境 → 打成镜像 → 任何地方直接跑
```

核心概念预览（下一节详细展开）：

- **Dockerfile** = 描述"环境怎么装"的构建脚本
- **镜像（Image）** = 代码 + 环境，打包好的只读文件
- **容器（Container）** = 镜像跑起来的运行实例

> [!example] 🔗 实战链接：deploy 项目中的镜像配置
> 在 deploy 项目中，每个微服务的 `values.yaml` 都配置了镜像信息——这就是"把环境打包好后，告诉 K8s 去哪里拉这个包"：
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/staging/main.yaml
> image:
>   repository: <account-id>.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
>   pullPolicy: IfNotPresent
>   tag: "4548edf"    # git commit hash，精确对应一次代码构建
> ```
> `repository` 就是 ECR 上的镜像地址，`tag` 是用 git commit hash 标记的版本号。同一个服务在不同区域用不同的 ECR：
> ```yaml
> # 海外区 (us-west-2)
> repository: <account-id>.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
> # 中国区 (cn-northwest-1)
> repository: <account-id>.dkr.ecr.cn-northwest-1.amazonaws.com.cn/plaud/plaud-project-summary
> ```
> 这体现了 Docker 的核心价值：**同一份代码构建出的镜像，无论部署到美国、日本还是中国的 EKS 集群，运行环境都完全一致。**

---

## 2. 核心概念：镜像与容器

### 2.1 为什么要区分「镜像」和「容器」？

镜像是静态的（像一个 zip 文件），容器是运行起来的实例（像解压后跑起来的程序）。分开的好处：

- 同一个镜像可以同时跑多个容器（比如水平扩容）
- 镜像只读，保证一致性；容器有自己的可写层，互不影响

### 2.2 镜像（Image）

**是什么**：只读的模板，包含运行程序所需的所有文件、库、依赖项。类比：安装光盘 / Python 的 `.whl` 文件。

**怎么来**：由 `Dockerfile` 通过 `docker build` 构建，或从镜像仓库（Docker Hub、ECR）拉取。

### 2.3 容器（Container）

**是什么**：镜像的运行实例，有自己的可写层（运行时产生的文件写这里）。类比：用安装光盘装好、正在运行的程序。

**关键特性**：
- 容器停止/删除后，可写层数据**丢失**
- 需要持久化数据时，用 **Volume**（挂载目录）

### 2.4 整体流程

```
源代码 + Dockerfile
        ↓ docker build（构建，只做一次）
      镜像 (Image)                ← 存储在本地或 ECR
        ↓ docker run（运行，可反复）
      容器 (Container)            ← 正在跑的进程
```

---

## 3. Dockerfile：如何构建镜像

### 3.1 为什么需要 Dockerfile？

手动在服务器上 `apt install`、`pip install` 不可重复、不可版本化。Dockerfile 是**构建过程的代码**，用声明式的方式描述"如何一步步组装出这个环境"，可以 git 版本管理、CI 自动构建。

### 3.2 Dockerfile 执行原理：分层缓存

Docker 把每一条指令都构建成一个**层（Layer）**。缓存机制是**前缀匹配**：从第一层开始逐层比较，某一层的输入变了，该层及之后所有层全部重跑——即使后面的层本身没有变化。

```
FROM python:3.12-slim          ← 基础镜像没变，命中缓存 ✅
COPY pyproject.toml ./         ← 文件没变，命中缓存 ✅
RUN pip install ...            ← 命中缓存 ✅
COPY agent ./agent             ← 改了一行代码，缓存失效 ❌
COPY api ./api                 ← 即使没改，也必须重跑 ❌（前面断了）
CMD [...]                      ← 必须重跑 ❌
```

原因：每一层的文件系统是在前一层基础上叠加的，前面变了后面的"基底"就变了，缓存不再可信。这和 Transformer 的 KV cache 是同一个模式——某个位置的 token 变了，后续所有位置的 KV 缓存都必须重算，因为 attention 依赖前面所有位置的结果。

**所以 Dockerfile 的核心优化原则是"不常变的放前面，常变的放后面"**，最大化前缀缓存命中率：

```dockerfile
COPY pyproject.toml uv.lock ./   # 依赖声明文件（很少变）→ 放前面
RUN uv pip install ...            # 安装依赖（命中缓存，几秒完成）

COPY agent ./agent               # 业务代码（经常变）→ 放后面，只重跑这层之后的
```

如果反过来先 `COPY . .`，每次改一行代码都会重新安装所有依赖，构建要几分钟。

### 3.3 真实项目 Dockerfile 逐行解析

```dockerfile
# ① 选基础镜像：python:3.12-slim
# 为什么 slim：完整版 ~900MB，slim ~150MB，去掉了编译工具但保留运行时
# 为什么不用 alpine：alpine 用 musl libc，某些 Python 包（如 numpy）有兼容问题
FROM python:3.12-slim

# ② 在镜像内部设置当前工作目录，类似于在终端里执行 `cd /app`
# 后续所有 RUN、COPY、CMD 都在 /app 下执行
# 不设的话默认在 /，容易和系统文件混在一起
WORKDIR /app

# ③ 安装系统依赖
# DEBIAN_FRONTEND=noninteractive：避免 apt 弹交互式提示（在 CI/容器里会卡住）
# --no-install-recommends：只装必要包，减小体积
# rm -rf /var/lib/apt/lists/*：清理 apt 缓存，否则会留在这一层增大镜像体积
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    apt-get install -y --no-install-recommends git supervisor && \
    rm -rf /var/lib/apt/lists/*

# ④ 安装 uv（Python 包管理器，比 pip 快很多）
RUN pip install --root-user-action=ignore --no-warn-script-location uv

# ⑤ 先复制依赖文件（利用缓存！）
COPY pyproject.toml uv.lock ./

# ⑥ 处理私有 GitHub 依赖
# 项目依赖了私有 GitHub 包，需要 token 才能拉取
# 做法：从 pyproject.toml 里提取 token，临时配置 git
RUN TOKEN=$(grep -oP 'https://\K[^@]+(?=@github.com)' pyproject.toml | head -1) && \
    git config --global url."https://${TOKEN}@github.com/".insteadOf "https://github.com/"

# ⑦ 安装 Python 依赖
# --frozen：严格按 uv.lock 安装，不允许版本漂移（生产环境必须）
# --no-dev：不装开发依赖（pytest 等不需要进容器）
RUN uv export --frozen --no-dev --no-hashes -o requirements.txt && \
    uv pip install --system --no-cache -r requirements.txt

# ⑧ 清理 git 凭据（安全！token 不能留在镜像里）
RUN git config --global --unset-all url.https://.insteadOf || true && \
    rm -f requirements.txt

# ⑨ 复制业务代码（放最后，代码改动不影响上面的缓存层）
# COPY 格式：COPY <宿主机路径> <镜像内路径>
# 下面这行把宿主机的 agent/ 目录复制到镜像内的 /app/agent/（相对于 WORKDIR）
COPY agent ./agent
COPY api ./api
COPY common ./common
COPY conf ./conf
COPY demo ./demo
COPY evaluator ./evaluator
COPY healthcheck ./healthcheck

# ⑩ 复制 supervisord 配置（下一节详解）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# ⑪ 声明端口（仅文档作用，实际发布要 docker run -p）
EXPOSE 8000 8001 8002 8889

# ⑫ 容器启动命令：用 supervisord 管理多个子进程
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

> [!example] 🔗 实战链接：Helm 模板中如何引用镜像
> Dockerfile 构建出镜像后，K8s 需要知道"去哪拉镜像、用什么版本"。在 deploy 项目中，`generic-deployer` 的 Deployment 模板是这样引用的：
> ```yaml
> # deploy/generic-deployer/templates/deployment.yaml
> containers:
>   - name: {{ .Chart.Name }}
>     image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
>     imagePullPolicy: {{ .Values.image.pullPolicy }}
> ```
> 这里的 `image.repository` 和 `image.tag` 就来自各微服务的 values 文件（如 `plaud-project-summary` 的 `tag: "4548edf"`）。**Dockerfile 负责"怎么打包"，values 文件负责"用哪个版本"**——两者通过镜像地址 + tag 关联起来。

### 3.4 Dockerfile 常用指令汇总

| 指令 | 时机 | 作用 |
|------|------|------|
| `FROM` | 构建时 | 指定基础镜像 |
| `WORKDIR` | 构建时 | 设置工作目录（自动创建） |
| `COPY` | 构建时 | 把本地文件复制进镜像 |
| `RUN` | 构建时 | 执行 shell 命令，结果存入镜像层 |
| `ENV` | 构建+运行时 | 设置环境变量 |
| `EXPOSE` | 文档 | 声明容器监听的端口 |
| `CMD` | 运行时 | 容器启动时执行的命令 |

**RUN vs CMD 的本质区别**：
- `RUN`：构建镜像时执行，结果固化进镜像（如安装软件）
- `CMD`：每次启动容器时执行（如启动服务进程）

### 3.5 ENTRYPOINT vs CMD（进阶）

这两个指令都定义容器启动时执行什么，但语义不同：

| 指令 | 语义 | 能否被 `docker run` 覆盖 |
|------|------|------|
| `ENTRYPOINT` | 容器的**主程序**（"这个容器是干什么的"） | 需要 `--entrypoint` 显式覆盖 |
| `CMD` | 主程序的**默认参数** | `docker run <image> <args>` 直接覆盖 |

**常见组合模式**：

```dockerfile
# 模式 1：只用 CMD（本项目用法，最简单）
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
# docker run 可以直接覆盖：docker run myimage bash

# 模式 2：ENTRYPOINT + CMD（推荐的生产模式）
ENTRYPOINT ["uvicorn"]
CMD ["api.server:app", "--host", "0.0.0.0", "--port", "8001"]
# docker run myimage → 执行 uvicorn api.server:app --host 0.0.0.0 --port 8001
# docker run myimage --port 9000 → 执行 uvicorn --port 9000（CMD 被覆盖）

# 模式 3：ENTRYPOINT 用脚本（需要初始化逻辑时）
COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

> **exec 形式 vs shell 形式**：`CMD ["cmd", "arg"]`（exec 形式）直接启动进程作为 PID 1；`CMD cmd arg`（shell 形式）会先启动 `/bin/sh -c`，你的进程变成子进程，信号传递会出问题。**生产环境始终使用 exec 形式。**

### 3.6 .dockerignore：控制构建上下文（进阶）

`docker build .` 的 `.` 是**构建上下文**——Docker 会把这个目录下的所有文件发送给 Docker daemon。如果目录里有大文件（`node_modules`、`.git`、数据文件），构建会非常慢。

`.dockerignore` 的作用和 `.gitignore` 一样，排除不需要的文件：

```gitignore
# .dockerignore
.git
.github
__pycache__
*.pyc
.env
.venv
node_modules
*.md
tests/
docs/
.mypy_cache
.pytest_cache
```

**不加 `.dockerignore` 的后果**：
1. 构建上下文传输慢（`.git` 目录可能几百 MB）
2. `.env` 文件可能被 `COPY . .` 意外复制进镜像（泄露密钥！）
3. 缓存失效——构建上下文变了，所有 `COPY` 层都会重新执行

### 3.7 多阶段构建（Multi-stage Build）（进阶）

**问题**：构建时需要编译工具（gcc、make、cargo），但运行时不需要。如果用单阶段构建，这些工具会留在最终镜像里，白白增大几百 MB。

**多阶段构建**：用多个 `FROM`，在构建阶段编译，只把最终产物复制到精简的运行阶段：

```dockerfile
# ===== 阶段 1：构建（安装编译依赖，体积大但不要紧） =====
FROM python:3.12-slim AS builder

WORKDIR /app
RUN apt-get update && apt-get install -y gcc libffi-dev
COPY pyproject.toml uv.lock ./
RUN pip install uv && \
    uv export --frozen --no-dev --no-hashes -o requirements.txt && \
    uv pip install --system --no-cache -r requirements.txt

# ===== 阶段 2：运行（只复制需要的文件，体积小） =====
FROM python:3.12-slim AS runtime

WORKDIR /app
# 只从 builder 阶段复制已安装的 Python 包
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# 复制业务代码
COPY agent ./agent
COPY api ./api

CMD ["uvicorn", "api.server:app", "--host", "0.0.0.0", "--port", "8001"]
```

**效果对比**：

| 构建方式 | 镜像大小 | 包含内容 |
|------|------|------|
| 单阶段 | ~800MB | Python + gcc + libffi + 源码 + 编译缓存 |
| 多阶段 | ~250MB | Python + 运行时依赖 + 业务代码 |

**什么时候需要多阶段**：
- 有 C 扩展的 Python 包需要编译（numpy、grpcio 等）
- Go/Rust 项目（编译产物是单个二进制，运行时甚至不需要语言运行时）
- 前端项目（Node.js 构建 → Nginx 提供静态文件）

> 本项目的 Dockerfile 目前用单阶段构建（`python:3.12-slim` 预编译的 wheel 包不需要 gcc），如果后续引入需要编译的依赖，可改为多阶段。

### 3.8 HEALTHCHECK 指令（进阶）

Dockerfile 中可以内置健康检查（独立于 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|K8s 的 livenessProbe/readinessProbe]]）：

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

| 参数 | 含义 |
|------|------|
| `--interval` | 每次检查的间隔 |
| `--timeout` | 单次检查超时 |
| `--start-period` | 容器启动后的宽限期（期间失败不计入重试） |
| `--retries` | 连续失败多少次标记为 unhealthy |

> **在 K8s 环境中**，通常不在 Dockerfile 里写 HEALTHCHECK，而是用 K8s 自己的 `livenessProbe` / `readinessProbe`（更灵活、支持更多协议）。Dockerfile 的 HEALTHCHECK 主要用于 `docker run` 或 Docker Compose 场景。

### 3.9 构建密钥（Build Secrets）（深入）

本项目的 Dockerfile 中用 `RUN` 提取 GitHub token 再清理，但 token 仍然残留在镜像的中间层中（`docker history` 可以看到）。更安全的做法是 **BuildKit Secrets**：

```dockerfile
# syntax=docker/dockerfile:1
# 使用 --mount=type=secret，密钥不会写入任何镜像层
RUN --mount=type=secret,id=github_token \
    TOKEN=$(cat /run/secrets/github_token) && \
    git config --global url."https://${TOKEN}@github.com/".insteadOf "https://github.com/" && \
    pip install -r requirements.txt && \
    git config --global --unset-all url.https://.insteadOf
```

构建时传入密钥：

```bash
DOCKER_BUILDKIT=1 docker build --secret id=github_token,src=./token.txt -t myimage .
```

> `--mount=type=secret` 是 Docker BuildKit 的特性，密钥只在当前 `RUN` 指令中可用，不会留在任何镜像层，比手动清理更安全。

---

## 4. supervisord：容器内的进程管理器

### 4.1 为什么需要 supervisord？

**问题**：Docker 容器的设计哲学是「一个容器跑一个进程」。但这个项目有 4 个服务需要同时运行：

- `api`：业务 API（uvicorn，8001 端口）
- `agent`：后台 Agent 处理任务
- `frontend`：NiceGUI Demo 前端（8889 端口）
- `healthcheck`：健康检查 API（8000 端口，EKS 用）

**方案对比**：

| 方案 | 问题 |
|------|------|
| 跑 4 个容器（docker-compose） | 本项目选择单容器部署，简化 EKS 配置 |
| `CMD` 里写 `&` 启动多个进程 | shell 退出后子进程变孤儿；某个崩了不会重启；日志混乱 |
| **supervisord** | 专门的进程管理器，自动重启、日志归集、信号传递 |

**supervisord 的核心价值**：
1. 所有子进程崩溃后**自动重启**（`autorestart=true`）
2. 作为容器的 PID 1 进程，正确处理 Linux 信号（SIGTERM 等）
3. 把所有子进程的 stdout/stderr 统一转发到容器日志

### 4.2 supervisord 是怎么进入容器的

supervisord 不是 Docker 自带的，需要通过 Dockerfile 显式安装。整个过程只涉及 Dockerfile 里的两处指令：

**① 构建时**——在 Dockerfile 中用 `RUN` 指令将 supervisord 安装到镜像：

```dockerfile
# 这一行写在 Dockerfile 里，docker build 时执行
RUN apt-get install -y supervisor
```

**② 运行时**——在 Dockerfile 中用 `COPY` 放入配置文件，用 `CMD` 指定容器启动命令：

```dockerfile
# 这两行也写在 Dockerfile 里
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

`RUN` 在构建镜像时执行（装好工具），`CMD` 在每次启动容器时执行（用工具管理进程）——两者是不同阶段的事。完整的 Dockerfile 示例见 [3.3 节](#33-真实项目-dockerfile-逐行解析)。

### 4.3 真实项目 supervisord.conf 逐段解析

```ini
# === 全局配置 ===
[supervisord]
nodaemon=true   # 关键！在容器里必须设为 true
                # 默认 supervisord 会后台运行（daemon），容器就立刻退出了
                # nodaemon=true 让它前台运行，作为容器的主进程（PID 1）
user=root
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid

# === 业务 API 服务 ===
[program:api]
command=uvicorn api.server:app --host 0.0.0.0 --port 8001
#        ↑ 注意：容器内必须绑定 0.0.0.0，不能是 127.0.0.1
#          0.0.0.0 表示监听所有网络接口，外部才能访问
directory=/app      # 进程的工作目录（等同于 cd /app 后再跑）
autostart=true      # supervisord 启动时自动启动此程序
autorestart=true    # 进程异常退出后自动重启
stopwaitsecs=590    # 停止时最多等 590 秒（api 可能在处理长任务）

# 把进程的 stdout/stderr 转发到容器的标准输出
# maxbytes=0 表示不限制大小（不截断）
# 这样 docker logs 就能看到所有服务的日志
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

# === Agent 后台服务 ===
[program:agent]
command=python -m agent.main
directory=/app
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

# === Frontend Demo ===
[program:frontend]
command=python demo/frontend/project_summary_demo.py
directory=/app
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
environment=AGENT_API_URL="http://localhost:8001"
#           ↑ 给这个进程单独设环境变量
#             frontend 需要知道 api 的地址

# === 健康检查服务 ===
[program:healthcheck]
command=uvicorn healthcheck.server:app --host 0.0.0.0 --port 8000
directory=/app
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

### 4.4 容器启动后的进程树

```
容器启动
  └── PID 1: supervisord（主进程，nodaemon=true 保持前台）
        ├── api      → uvicorn :8001  （业务 API）
        ├── agent    → python agent.main（后台处理）
        ├── frontend → python demo/...  （NiceGUI :8889）
        └── healthcheck → uvicorn :8000（EKS 健康检查）
```

supervisord 挂了 → 容器退出 → EKS/K8s 重启容器。
子进程挂了 → supervisord 自动重启子进程 → 容器继续运行。

> [!example] 🔗 实战链接：K8s 健康检查如何与 supervisord 配合
> 在 deploy 项目的 values 文件中，K8s 的健康检查探针指向的正是 supervisord 管理的 healthcheck 子进程（监听 8000 端口）：
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/staging/main.yaml
> livenessProbe:
>   httpGet:
>     path: /health
>     port: 8000       # ← 就是 supervisord.conf 中 [program:healthcheck] 的 uvicorn :8000
>
> readinessProbe:
>   httpGet:
>     path: /health
>     port: 8000
> ```
> 整个链路是：**K8s kubelet 发 HTTP 请求到 Pod 的 8000 端口 → supervisord 管理的 healthcheck 子进程响应 → 返回健康状态**。如果 healthcheck 子进程挂了，supervisord 会自动重启它；如果 supervisord 本身挂了，容器退出，K8s 会重建 Pod。

---

## 5. 本地开发 vs Docker 启动

### 5.1 为什么本地开发不用 Docker？

Docker 有构建时间、没有热更新（代码改了要重新 build），不适合频繁修改代码的开发阶段。本地开发直接跑进程，改代码立即生效。

### 5.2 对比

| | 本地开发 (`make dev`) | Docker (`make up`) |
|---|---|---|
| **启动方式** | 直接在本机跑 Python 进程 | 构建镜像 → 启动容器 |
| **进程管理** | shell `&` + `trap` | supervisord |
| **热更新** | ✅ `--reload` 自动重载 | ❌ 改代码要重新 build |
| **环境隔离** | ❌ 依赖本机 Python 环境 | ✅ 完全隔离，和生产一致 |
| **适合场景** | 开发调试 | 测试生产行为、部署 |

### 5.3 make dev 的工作原理

```makefile
dev:
    @trap 'kill 0' INT; \          # Ctrl+C 时，kill 0 杀掉整个进程组（所有 & 启动的子进程）
    uvicorn api.server:app --reload --port 8001 & \       # 后台跑，--reload 热更新
    uvicorn healthcheck.server:app --reload --port 8000 & \
    python -m agent.main & \
    AGENT_API_URL=http://localhost:8001 python demo/frontend/project_summary_demo.py & \
    wait                           # 等待所有后台进程，防止 shell 立刻退出
```

**关键点**：
- `&`：后台运行，立刻返回，继续执行下一行
- `wait`：等待所有后台进程结束（否则 shell 退出，所有子进程也死了）
- `trap 'kill 0' INT`：捕获 Ctrl+C，确保 kill 整个进程组而不只是 shell 本身

### 5.4 make up 的工作原理

```
make up
  ├── docker build -t plaud-project-summary .   # 读 Dockerfile，构建镜像
  ├── docker stop/rm（清理旧容器）
  └── docker run -d \
        -p 8000:8000 -p 8001:8001 ... \         # 端口映射：主机端口 → 容器端口
        --env-file .env \                        # 从 .env 文件注入环境变量
        plaud-project-summary                    # 镜像名（不是 Dockerfile）
          └── 容器内 CMD: supervisord 启动 4 个子进程
```

**端口映射说明**：

```
你的浏览器访问 localhost:8001
      ↓ 主机的 8001 端口
      ↓ -p 8001:8001（主机端口:容器端口）
      ↓ 容器内的 8001 端口
      ↓ uvicorn 监听 0.0.0.0:8001
      ↓ FastAPI 处理请求
```

> [!example] 🔗 实战链接：K8s Service 中的端口映射
> `docker run -p` 的端口映射概念延续到了 K8s 的 Service 配置中。在 deploy 项目中，`plaud-project-summary` 的 Service 定义了三个端口映射：
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/staging/main.yaml
> service:
>   type: ClusterIP
>   ports:
>     - port: 8001          # Service 暴露的端口（其他 Pod 通过这个端口访问）
>       name: api
>       targetPort: 8001    # 容器实际监听的端口（supervisord 管理的 uvicorn）
>     - port: 8889
>       name: frontend
>       targetPort: 8889    # NiceGUI Demo 前端
>     - port: 8000
>       name: health
>       targetPort: 8000    # 健康检查
> ```
> 和 `docker run -p 8001:8001` 的逻辑一样——**冒号左边是"对外暴露的"（Service port），右边是"容器里实际跑的"（targetPort）**。在 K8s 中，其他 Pod 通过 `plaud-project-summary:8001` 访问 API，流量会被转发到容器的 8001 端口。

---

## 6. Docker 常用命令速查

### 构建镜像

```bash
docker build -t plaud-project-summary .
#             ↑ 镜像名称（-t = tag）
#                                    ↑ 构建上下文：Docker 会把这个目录下的【所有文件】
#                                      发送给 Docker daemon（后台服务进程，真正干活的；
#                                      CLI 只是发指令的客户端），不管 Dockerfile 用不用得到
#                                      所以需要 .dockerignore 排除无关文件（见 3.6 节）

docker build --no-cache -t plaud-project-summary .  # 不用缓存，强制全量重建
```

### 运行容器

```bash
docker run -d --name plaud-project-summary \
    -p 8000:8000 -p 8001:8001 -p 8002:8002 -p 8889:8889 \
    --env-file .env \
    plaud-project-summary

# -d：后台运行（detach），不加的话日志直接打到终端
# --name：给容器起名，方便后续 stop/logs/exec
# -p 主机端口:容器端口：真正的端口映射（Dockerfile 里的 EXPOSE 只是文档标注，不写 -p 外部访问不了）
#    Docker 的统一约定：冒号左边是宿主机，右边是容器（-p、-v 都是如此，"从哪来:到哪去"）
# --env-file：从文件加载环境变量
```

### 查看 & 调试

```bash
docker ps                        # 查看运行中的容器（ps = process status，来自 Linux）
docker ps -a                     # 包括已停止的
docker logs -f plaud-project-summary   # 实时跟踪日志（-f = follow）
docker logs --tail 100 plaud-project-summary  # 最后 100 行

docker exec -it plaud-project-summary bash  # 进入容器 shell
# -i：保持 stdin 打开，-t：分配终端；合起来才能交互
# 如果没有 bash，用 sh

docker exec plaud-project-summary env    # 查看容器内环境变量
docker stats plaud-project-summary       # 实时查看 CPU/内存使用
docker inspect plaud-project-summary     # 查看容器详情（网络、挂载、配置等）
docker cp plaud-project-summary:/app/logs ./logs  # 从容器复制文件到本地
```

### 停止 & 清理

```bash
docker stop plaud-project-summary   # 发 SIGTERM，优雅停止
docker rm plaud-project-summary     # 删除容器（要先 stop）
docker rm -f plaud-project-summary  # 强制删除（运行中也行）

docker images                        # 列出所有镜像
docker rmi plaud-project-summary     # 删除镜像

docker system prune -a               # 清理所有未使用的镜像、容器、网络
docker system df                     # 查看 Docker 占用磁盘
```

### 项目日常命令

```bash
make build    # 构建镜像
make run      # 启动容器
make up       # 构建 + 启动（自动清理旧容器）
make stop     # 停止并删除容器
make logs     # 实时查看日志
make dev      # 本地开发模式（无 Docker，有热更新）
```

### EKS 部署流程（完整链路）

```
本地开发
  ↓ make dev（直接跑进程，热更新）

构建 & 推送镜像
  ↓ docker build → docker push → ECR（AWS 镜像仓库）

EKS 部署
  ↓ kubelet 拉取 ECR 镜像（相当于 docker pull）
  ↓ kubelet 启动容器（相当于 docker run）
  ↓ 容器 CMD: supervisord
  ↓ supervisord 启动 4 个子进程

流量路由
  ↓ EKS Service → Pod 8001（业务 API）
  ↓ EKS 健康检查 → Pod 8000（healthcheck API）
```

---

## 7. Docker Compose（进阶）

### 7.1 为什么需要 Docker Compose？

当你的服务依赖多个组件（如 web + db + redis），每个都要 `docker run` 并手动配置网络、端口、环境变量，很繁琐。Docker Compose 用一个 `docker-compose.yml` 文件声明所有服务，一条命令启停。

> 本项目目前**不用** docker-compose（单容器 + supervisord），但了解 Compose 很有用。

### 7.2 常见误解澄清

**误解：docker-compose.yml 是"多个 Dockerfile 的组合"**

不是。docker-compose.yml 编排的是**多个独立容器**，每个容器从自己的镜像启动，拥有独立的文件系统、网络和进程空间：

```
docker-compose.yml 启动的是：
├── 容器 1 (web)     ← 镜像 A（可能来自 Dockerfile 构建）
├── 容器 2 (db)      ← 镜像 B（如 postgres:15，直接从 Docker Hub 拉取）
└── 容器 3 (redis)   ← 镜像 C（如 redis:7，直接拉取）

每个容器是独立的进程，有自己的文件系统、网络、IP。
不是一个容器里塞多个镜像，而是多个容器各跑一个镜像。
```

**Dockerfile、docker-compose、supervisord 三者的关系**：

```
Dockerfile        → 构建一个镜像（"怎么装环境"）
docker run        → 从一个镜像启动一个容器
docker-compose    → 从多个镜像启动多个容器（容器编排）
supervisord       → 在一个容器内管理多个进程（进程管理）
```

docker-compose 解决"多个容器怎么协同"，supervisord 解决"一个容器里多个进程怎么管"。

### 7.3 本项目为什么选 supervisord 而非 docker-compose

本项目有 4 个服务（api、agent、frontend、healthcheck），理论上可以拆成 4 个容器用 docker-compose 编排。但项目选择了单容器 + supervisord：

| 方案 | 容器数 | EKS 部署配置 | 复杂度 |
|------|------|------|------|
| docker-compose（4 个容器） | 4 | 4 个 Deployment + 4 个 Service + 容器间网络配置 | 高 |
| **supervisord（1 个容器）** | 1 | 1 个 Deployment + 1 个 Service | 低 |

选择 supervisord 的核心原因是**简化 EKS 部署**——4 个紧密耦合的服务作为一个整体部署和扩缩容，比拆成 4 个独立的微服务更简单。代价是违反了 Docker "一容器一进程" 的最佳实践，但对于紧密耦合的服务组合，这个 tradeoff 是合理的。

> 注意：supervisord 和 Dockerfile 并不矛盾——Dockerfile 负责构建镜像（安装 supervisord + 业务代码），supervisord 在容器运行时管理多个进程。两者是构建阶段和运行阶段的不同工具。

### 7.4 docker-compose.yml 基础模板

```yaml
services:
  web:
    build: .                      # 从当前目录 Dockerfile 构建
    ports:
      - "8001:8001"               # 主机端口:容器端口
    environment:
      - DATABASE_URL=postgres://...
    env_file:
      - .env                      # 从 .env 文件加载
    depends_on:
      - db                        # 在 db 启动后再启动 web

  db:
    image: postgres:15            # 直接用现有镜像，不需要 Dockerfile
    volumes:
      - db_data:/var/lib/postgresql/data   # 命名卷，数据持久化
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  db_data:                        # 声明命名卷
```

### 7.5 常用命令

```bash
docker compose up -d              # 后台启动所有服务
docker compose up -d --build      # 强制重新构建镜像再启动
docker compose down               # 停止并删除所有容器
docker compose down -v            # 同时删除数据卷（⚠️ 数据会丢失）

docker compose logs -f            # 实时查看所有服务日志
docker compose logs -f web        # 只看 web 服务日志
docker compose ps                 # 查看服务状态
docker compose exec web bash      # 进入 web 容器
```

### 7.6 Volume 详解

**为什么需要 Volume**：容器删除后可写层数据丢失。Volume 是独立于容器生命周期的持久存储。

| 类型 | 写法 | 用途 |
|------|------|------|
| 命名卷 | `db_data:/var/lib/data` | 数据库数据，由 Docker 管理 |
| 绑定挂载 | `./src:/app/src` | 开发时同步代码（热更新） |
| 只读挂载 | `./config.json:/app/config.json:ro` | 配置文件只读注入 |

**COPY vs Volume 的区别**：
- `COPY`（Dockerfile）：构建时把文件**复制**进镜像，成为镜像的一部分，只读
- `volumes`（运行时）：把本地目录**映射**到容器，双向实时同步，容器内改了本地也变

---

## 9. 镜像优化最佳实践（进阶）

### 9.1 基础镜像选择

| 基础镜像 | 大小 | 适用场景 | 注意事项 |
|------|------|------|------|
| `python:3.12` | ~900MB | 需要完整编译工具链时 | 体积大，攻击面大 |
| `python:3.12-slim` | ~150MB | **大多数 Python 项目（推荐）** | 去掉了编译工具但保留 glibc |
| `python:3.12-alpine` | ~50MB | 对体积极度敏感 | 用 musl libc，numpy 等包兼容性差 |
| `gcr.io/distroless/python3` | ~50MB | 安全要求极高的生产环境 | 无 shell、无包管理器，调试困难 |
| `ubuntu:22.04` | ~77MB | 需要系统工具但不需要预装 Python | 需要自己装 Python |

> **本项目选择 `python:3.12-slim`** 是体积和兼容性的最佳平衡点。

### 9.2 减小镜像体积的技巧

```dockerfile
# ① 合并 RUN 指令 — 减少层数，同一层内清理不留痕迹
RUN apt-get update && \
    apt-get install -y --no-install-recommends git && \
    rm -rf /var/lib/apt/lists/*
# 如果 install 和 rm 是两个 RUN，中间层仍包含 apt 缓存

# ② 不安装推荐包
apt-get install -y --no-install-recommends <package>

# ③ 使用 pip --no-cache-dir / uv --no-cache
RUN pip install --no-cache-dir -r requirements.txt

# ④ 多阶段构建（见 3.7 节）

# ⑤ 用 .dockerignore 排除无关文件（见 3.6 节）
```

### 9.3 镜像安全扫描

镜像中的系统包和 Python 包可能包含已知漏洞（CVE），上线前应该扫描：

```bash
# Trivy — 开源镜像扫描工具（推荐）
trivy image plaud-project-summary:latest

# 只报告高危和严重漏洞
trivy image --severity HIGH,CRITICAL plaud-project-summary:latest

# 扫描结果示例：
# plaud-project-summary:latest (debian 12.5)
# Total: 3 (HIGH: 2, CRITICAL: 1)
# ┌──────────────┬───────────────┬──────────┬────────────────┐
# │   Library    │ Vulnerability │ Severity │ Fixed Version  │
# ├──────────────┼───────────────┼──────────┼────────────────┤
# │ libssl3      │ CVE-2024-xxx  │ CRITICAL │ 3.0.13-1       │
# └──────────────┴───────────────┴──────────┴────────────────┘
```

**CI/CD 集成**：在构建流水线中加入扫描步骤，高危漏洞阻断部署。

---

## 10. 容器运行时（深入）

### 10.1 Docker 不是唯一的容器运行时

很多人把 Docker 和容器画等号，但实际上：

```
容器生态的分层：
┌─────────────────────────────────────────────────┐
│  上层工具（用户接触）                               │
│  Docker CLI / Docker Compose / Podman / nerdctl  │
├─────────────────────────────────────────────────┤
│  容器运行时（CRI 接口）                             │
│  containerd / CRI-O                              │
├─────────────────────────────────────────────────┤
│  底层运行时（OCI 规范）                             │
│  runc（创建和运行容器的低级工具）                      │
├─────────────────────────────────────────────────┤
│  Linux 内核（cgroups + namespaces）                │
└─────────────────────────────────────────────────┘
```

- **Docker**：最早的容器平台，包含 CLI + dockerd + containerd + runc 整套工具链
- **containerd**：从 Docker 中剥离出来的容器运行时，K8s 1.24+ 的默认运行时（Kubernetes 已弃用 dockershim）
- **CRI-O**：Red Hat 开发的轻量 CRI 实现，专为 K8s 设计
- **runc**：OCI 标准的参考实现，负责最底层的容器创建（cgroups + namespaces）

### 10.2 OCI 标准

OCI（Open Container Initiative）定义了两个核心规范：

| 规范 | 内容 | 意义 |
|------|------|------|
| **Image Spec** | 镜像格式标准 | Docker 构建的镜像、Podman 构建的镜像都能互用 |
| **Runtime Spec** | 容器运行标准 | runc、crun 等不同实现都能运行同一个镜像 |

> 对你的日常工作的影响：**几乎没有。** 本地开发继续用 Docker CLI，在 EKS 上 kubelet 通过 containerd 运行容器。你 `docker build` 出来的镜像 push 到 ECR 后，[[05-k8s-architecture#4.1 kubelet — 节点上的"管家"|kubelet]] 能直接拉取运行，因为大家都遵循 OCI 标准。

### 10.3 ECR（Elastic Container Registry）

ECR 是 AWS 提供的**私有 Docker 镜像仓库**，相当于公司内部的 Docker Hub。作用就一个：**存放 Docker 镜像**，让 EKS 集群能拉取到构建的镜像来运行。

#### 项目的 ECR 配置

每个微服务在 `ci.json` 中配置 ECR 地址，CI 系统据此构建和推送镜像。以 `plaud-project-summary` 为例，配置了**两个 ECR 仓库**（海外 + 中国）：

| 环境 | ECR 地址 | AWS 账号 | Region |
|------|----------|----------|--------|
| 海外 | `236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary` | 236604669925 | us-west-2 |
| 中国 | `470515048733.dkr.ecr.cn-northwest-1.amazonaws.com.cn/plaud/plaud-project-summary` | 470515048733 | cn-northwest-1 |

> **为什么有两个？** AWS 中国区和海外区完全隔离（不同账号、不同域名 `.amazonaws.com.cn`），部署在中国区 EKS 的服务必须从中国区 ECR 拉镜像，否则跨境拉取速度极慢甚至失败。

#### 镜像地址拆解

```
236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:93347b0
├──────────────────────────────────────────┘ ├──────────────────────────┘ ├─────┘
ECR 域名（AWS 账号 ID + Region）               Repository（仓库名称）        Tag（标签）
```

对应 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|values 文件中的 image.repository]]。标签通常用 **git commit hash 前 7 位**，可精确追溯每个镜像对应的代码版本。

**注意 Repository ≠ 镜像名称。** Repository 是你**提前在 AWS 上创建**的存储空间，用来存放同一个服务的所有版本镜像。一个 Repository 里按 Tag 区分不同版本：

```
plaud/plaud-project-summary:93347b0   ← 第一次构建
plaud/plaud-project-summary:a1b2c3d   ← 第二次构建
plaud/plaud-project-summary:f4e5d6a   ← 第三次构建
```

与 Docker Hub 不同，**ECR 必须先创建 Repository 才能 push**，否则报错 `RepositoryNotFoundException`。

> [!example] 🔗 实战链接：不同环境使用不同的镜像 tag
> 在 deploy 项目中，同一个服务在不同环境（staging / prod）使用不同的 git commit hash 作为 tag，实现版本独立控制：
> ```yaml
> # Staging 环境 — 跑最新的开发版本
> # deploy/plaud-project-summary/values/us-west-2/staging/main.yaml
> image:
>   repository: <account-id>.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
>   tag: "4548edf"    # staging 的 commit hash
>
> # Production 环境 — 跑验证过的稳定版本
> # deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
> image:
>   repository: <account-id>.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
>   tag: "7e39d6c"    # prod 的 commit hash（通常滞后于 staging）
> ```
> 这体现了 ECR + git hash tag 的核心价值：**同一个 Repository 里按 tag 区分版本，staging 先验证新版本，确认没问题后再把 prod 的 tag 指向同一个镜像**。

#### 手动操作流程（了解原理用，日常由 CI 自动完成）

**Step 1 — 登录 ECR**（token 有效期 12 小时）：

```bash
# 海外区登录
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 236604669925.dkr.ecr.us-west-2.amazonaws.com

# 中国区登录
aws ecr get-login-password --region cn-northwest-1 | docker login --username AWS --password-stdin 470515048733.dkr.ecr.cn-northwest-1.amazonaws.com.cn
```

**Step 2 — 构建镜像**：

```bash
docker build -t plaud-project-summary .
```

**Step 3 — 打标签并推送**：

```bash
GIT_HASH=$(git rev-parse --short HEAD)

# 海外区
docker tag plaud-project-summary:latest 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:${GIT_HASH}
docker push 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:${GIT_HASH}

# 中国区
docker tag plaud-project-summary:latest 470515048733.dkr.ecr.cn-northwest-1.amazonaws.com.cn/plaud/plaud-project-summary:${GIT_HASH}
docker push 470515048733.dkr.ecr.cn-northwest-1.amazonaws.com.cn/plaud/plaud-project-summary:${GIT_HASH}
```

#### 实际工作中：CI/CD 自动完成

项目里只需要**三个文件**就能让 CI/CD 跑起来：

| 文件 | 作用 | 需要改动吗 |
|------|------|-----------|
| `Dockerfile` | 描述如何构建镜像 | 按需调整 |
| `ci.json` | 告诉 CI 推到哪些 ECR | 一般不动 |
| `.github/workflows/build_push.yaml` | 触发 CI 的胶水 | 基本不动 |

**为什么只需要这么少？** 因为实际的构建逻辑在**公司共享的 CI workflow** 中，各项目只负责声明，不负责实现：

```yaml
# .github/workflows/build_push.yaml 全部内容：
name: CI

on:
  push:
    branches: ["**"]    # 任何分支的 push 都会触发

jobs:
  cfg:
    runs-on: iops-runner-general
    outputs:
      images: ${{ steps.j.outputs.images }}
    steps:
      - uses: actions/checkout@v4
      - id: j
        shell: bash
        run: |
          set -euo pipefail
          jq -e . ci.json >/dev/null
          echo "images=$(jq -c '.images' ci.json)" >> "$GITHUB_OUTPUT"

  ci:
    needs: cfg
    # 核心：调用公司共享 workflow，构建逻辑全在这里面
    uses: Plaud-AI/plaud-ci-workflows/.github/workflows/docker-build-push.yaml@main
    with:
      images: ${{ needs.cfg.outputs.images }}
    secrets: inherit
```

**完整的自动化流程**：

```
git push（任何分支）
    ↓
GitHub Actions 触发 build_push.yaml
    ↓
cfg job：读取 ci.json，解析出 images 列表
    ↓
ci job：调用共享 workflow（plaud-ci-workflows），传入 images 列表
    ↓
共享 workflow 内部：
  ├─ 登录 ECR
  ├─ docker build -t <ecr>:<git-hash> .
  ├─ docker push 到海外 ECR
  └─ docker push 到中国 ECR
    ↓
ArgoCD 检测到新镜像 tag → 部署到 EKS
```

#### 关于镜像 tag 的 git hash 位数

镜像 tag 使用的 git commit hash 位数**不是项目里配的，而是共享 workflow 里决定的**。通常用：

```bash
git rev-parse --short HEAD      # 默认 7 位，如 93347b0
git rev-parse --short=8 HEAD    # 可以指定位数
```

`--short` 默认输出 **7 位**，但 Git 会智能调整——当 7 位不足以唯一标识时（超大仓库提交数极多），会自动加长到 8、9 位。对一般项目来说 7 位绰绰有余（16^7 ≈ 2.7 亿种组合）。具体位数可查看 `Plaud-AI/plaud-ci-workflows` 仓库。

#### 新建微服务项目的 CI/CD 清单

如果要从零创建一个新的微服务项目，CI/CD 部分只需：

1. **写 `Dockerfile`** — 描述如何构建服务
2. **在 AWS 创建 ECR 仓库** — 海外区和中国区各一个（通常由 Terraform/基础设施团队完成）
3. **写 `ci.json`** — 填上两个 ECR 地址
4. **复制 `.github/workflows/build_push.yaml`** — 从现有项目直接复制，内容不变
5. **push 到 GitHub** — 自动触发构建和推送

#### 排查镜像的常用命令

```bash
# 查看仓库里有哪些镜像 tag
aws ecr list-images --repository-name plaud/plaud-project-summary --region us-west-2

# 查看某个 tag 的详细信息（大小、推送时间等）
aws ecr describe-images --repository-name plaud/plaud-project-summary --image-ids imageTag=93347b0 --region us-west-2

# 手动拉取镜像到本地调试
docker pull 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:93347b0
```

---

## 延伸阅读

- [[02-k8s-core-concepts|K8s 核心概念入门]] — 了解 Docker 之后，K8s 的 Pod、Deployment、Service 等核心对象
- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — Docker 镜像构建完成后，如何通过 Helm + ArgoCD 部署到 EKS 集群
- [[12-k8s-pod-graceful-shutdown|Pod 优雅终止完全指南]] — 容器内 supervisord 如何正确处理 SIGTERM 信号
- [[13-k8s-lane-mechanism|K8s 泳道机制]] — 同一镜像的不同版本如何通过泳道实现流量隔离
