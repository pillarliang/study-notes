# Docker 学习笔记

> 面向有 Python 经验、Docker 不熟的工程师。每个概念从"为什么需要它"出发。

---

## 目录

1. [为什么需要 Docker？](#1-为什么需要-docker)
2. [核心概念：镜像与容器](#2-核心概念镜像与容器)
3. [Dockerfile：如何构建镜像](#3-dockerfile如何构建镜像)
4. [supervisord：容器内的进程管理器](#4-supervisord容器内的进程管理器)
5. [本地开发 vs Docker 启动](#5-本地开发-vs-docker-启动)
6. [Docker 常用命令速查](#6-docker-常用命令速查)
7. [Docker Compose](#7-docker-compose)
8. [常用操作速查](#8-常用操作速查)

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

其中：
- **Dockerfile** = 描述「环境怎么装」的脚本（构建时用）
- **镜像** = 代码 + 环境，打包好的静态文件
- **supervisord** = 容器启动后负责管理进程的工具（运行时用）

完整流程：

```
Dockerfile（描述环境）
    ↓ docker build
镜像（代码 + 环境，静态）
    ↓ docker run
容器启动，执行 CMD
    ↓
supervisord 开始工作，管理容器内的 4 个子进程
```

**Dockerfile 和 supervisord 的关联只有两处**：

① **安装阶段**（构建时，`RUN` 指令）——把 supervisord 装进镜像：
```dockerfile
RUN apt-get install -y supervisor
```

② **启动阶段**（运行时，`CMD` 指令）——容器启动时第一个跑的就是 supervisord：
```dockerfile
COPY supervisord.conf ...
CMD ["/usr/bin/supervisord", ...]
```

两者是不同阶段的事：Dockerfile 在**构建时**把 supervisord 安装好，supervisord 在**运行时**管理子进程。

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

Docker 把每一条指令都构建成一个**层（Layer）**。如果某一层的输入没变，下次构建直接用缓存，跳过这一层。

这就是为什么依赖安装要放在代码复制之前：

```dockerfile
COPY pyproject.toml uv.lock ./   # 依赖声明文件（很少变）
RUN uv pip install ...            # 安装依赖（命中缓存，几秒完成）

COPY agent ./agent               # 业务代码（经常变，但只重跑这层之后的）
```

如果反过来先 `COPY . .`，每次改一行代码都会重新安装所有依赖，构建要几分钟。

### 3.3 真实项目 Dockerfile 逐行解析

```dockerfile
# ① 选基础镜像：python:3.12-slim
# 为什么 slim：完整版 ~900MB，slim ~150MB，去掉了编译工具但保留运行时
# 为什么不用 alpine：alpine 用 musl libc，某些 Python 包（如 numpy）有兼容问题
FROM python:3.12-slim

# ② 设置工作目录
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

### 4.2 真实项目 supervisord.conf 逐段解析

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

### 4.3 容器启动后的进程树

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

---

## 6. Docker 常用命令速查

### 构建镜像

```bash
docker build -t plaud-project-summary .
#             ↑ 镜像名称（-t = tag）
#                                    ↑ 构建上下文（当前目录，Dockerfile 在这里）

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
# -p 主机端口:容器端口：端口映射
# --env-file：从文件加载环境变量
```

### 查看 & 调试

```bash
docker ps                        # 查看运行中的容器
docker ps -a                     # 包括已停止的
docker logs -f plaud-project-summary   # 实时跟踪日志（-f = follow）
docker logs --tail 100 plaud-project-summary  # 最后 100 行

docker exec -it plaud-project-summary bash  # 进入容器 shell
# -i：保持 stdin 打开，-t：分配终端；合起来才能交互
# 如果没有 bash，用 sh
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

---

## 7. Docker Compose

### 7.1 为什么需要 Docker Compose？

当你的服务依赖多个组件（如 web + db + redis），每个都要 `docker run` 并手动配置网络、端口、环境变量，很繁琐。Docker Compose 用一个 `docker-compose.yml` 文件声明所有服务，一条命令启停。

> 本项目目前**不用** docker-compose（单容器 + supervisord），但了解 Compose 很有用。

### 7.2 docker-compose.yml 基础模板

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

### 7.3 常用命令

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

### 7.4 Volume 详解

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

## 8. 常用操作速查

### 项目日常命令

```bash
make build    # 构建镜像
make run      # 启动容器
make up       # 构建 + 启动（自动清理旧容器）
make stop     # 停止并删除容器
make logs     # 实时查看日志
make dev      # 本地开发模式（无 Docker，有热更新）
make api-dev  # 只启动 API 服务
make agent-dev # 只启动 Agent
```

### 调试技巧

```bash
# 进入容器查看文件/环境
docker exec -it plaud-project-summary bash

# 查看容器内环境变量
docker exec plaud-project-summary env

# 查看容器资源使用（CPU/内存）
docker stats plaud-project-summary

# 查看容器详情（网络、挂载、配置等）
docker inspect plaud-project-summary

# 从容器复制文件到本地（如日志文件）
docker cp plaud-project-summary:/app/logs ./logs
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
