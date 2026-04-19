# Kubernetes Pod 优雅终止 (Graceful Shutdown) 完全指南

> 前置知识：了解 [[02-k8s-core-concepts|Pod 和 Deployment]] 的基本概念。本文深入讲解 Pod 被终止时的完整流程。
>
> **学习建议**：首次阅读重点关注背景、问题现象和前三层（滚动更新策略、Pod 终止流程、preStop hook）。第四至五层（supervisord 调优、应用层代码）和 proxy-next-upstream 讨论为进阶。

## 背景

在 Kubernetes 中运行长耗时任务（如 AI 摘要生成、音视频转码）时，服务重启/滚动更新会导致正在执行的任务被强制终止。本文以一次真实的排障过程为例，从第一性原理出发，系统梳理优雅终止的每一层机制。

**核心问题：** 滚动更新时如何做到——

1. 正在执行的存量任务不被中断
2. 新发起的任务不会丢失或失败
3. 整个过程对客户端完全透明

---

## 问题现象

任务一般需要 **10 分钟** 才能执行完成。当发起任务后立刻重启 EKS 服务（滚动更新），Pod 没有等待任务完成就直接被杀掉，导致运行中的任务失败。

---

## 第一层：滚动更新策略 — 新 Pod Ready 前不杀旧 Pod

在讨论终止流程之前，首先要理解 K8s 什么时候决定终止一个旧 Pod。这由 [[03-k8s-workload-types#二、Deployment（回顾）|Deployment]] 的滚动更新策略决定：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0    # 不允许任何 Pod 不可用
    maxSurge: "25%"      # 允许临时多创建 25% 的 Pod
```

`maxUnavailable: 0` 的含义：**可用 Pod 数量任何时刻都不能低于 replicas 数**。

以 `replicas: 2` 为例，`maxSurge: 25%` 向上取整 = 1，所以更新流程为：

```
T0  触发滚动更新
      旧Pod-A: Running ✅    旧Pod-B: Running ✅         → 2 个可用

T1  创建新 Pod（surge）
      旧Pod-A: Running ✅    旧Pod-B: Running ✅
      新Pod-1: Starting ⏳ (拉镜像、启动中，未通过 readinessProbe)  → 仍然 2 个可用

T2  新Pod-1 通过 readinessProbe ✅
      旧Pod-A: Running ✅    旧Pod-B: Running ✅
      新Pod-1: Ready   ✅                                → 3 个可用，有余量了

T3  K8s 开始终止旧Pod-A（详见下一节）
      旧Pod-A: Terminating   旧Pod-B: Running ✅
      新Pod-1: Ready   ✅                                → 仍然 ≥ 2 个可用

    同时创建新Pod-2...

T4  新Pod-2 通过 readinessProbe → 开始终止旧Pod-B
      新Pod-1: Ready ✅      新Pod-2: Ready ✅           → 2 个可用
      旧Pod-A: Terminating   旧Pod-B: Terminating        → 等待存量任务完成

T5  旧 Pod 全部退出，更新完成
      新Pod-1: Running ✅    新Pod-2: Running ✅          → 2 个可用
```

**关键保证：任何时刻至少有 2 个 Ready 的 Pod 在接流量。** 不会出现"新 Pod 还没启动，旧 Pod 就被杀了"的情况。

> [!example] 🔗 实战链接：generic-deployer 的默认滚动更新策略
> 在 `deploy/generic-deployer/templates/deployment.yaml` 中，所有服务共用的 Deployment 模板已经将 `maxUnavailable: 0` 设为默认值：
> ```yaml
> strategy:
>     type: RollingUpdate
>     rollingUpdate:
>       maxUnavailable: {{ .Values.updateStrategy.maxUnavailable | default 0 }}
>       maxSurge: {{ .Values.updateStrategy.maxSurge | default "25%" }}
> ```
> 对应的默认值定义在 `deploy/generic-deployer/values.yaml`：
> ```yaml
> updateStrategy:
>   maxUnavailable: 0
>   maxSurge: "25%"
> ```
> **解读**：这意味着公司 80+ 微服务默认都遵循 `maxUnavailable: 0` 策略——滚动更新时不允许任何 Pod 不可用，新 Pod Ready 后才会终止旧 Pod。这正是上文描述的"第一层保证"在实际项目中的落地。

---

## 第二层：Pod 终止流程 — 两条并行路径

当 K8s 决定终止一个 Pod 时，**两件事同时发生**：

```
K8s 标记 Pod 为 Terminating
    │
    │  ┌─────────────────────────────────────────────────────────────┐
    │  │ 同一瞬间，两条路径并行启动：                                    │
    │  │                                                             │
    │  │ 【路径 A：流量摘除】                                          │
    │  │   API Server 立即从 Endpoints（"该 Service 后面有哪些 Pod"    │
    │  │   的列表）中删除该 Pod                                        │
    │  │       │                                                     │
    │  │       ├→ kube-proxy（每个节点上的网络代理，维护 iptables       │
    │  │       │  转发规则）watch 到变更 → 更新 iptables（~1-2s）       │
    │  │       └→ Ingress Controller watch 到 → reload upstream（~1-3s）│
    │  │       │                                                     │
    │  │       └→ 约 3 秒后，旧 Pod 从所有流量路径中移除 ✅               │
    │  │                                                             │
    │  │ 【路径 B：Pod 终止】                                          │
    │  │   执行 preStop hook → 结束后发送 SIGTERM（终止信号，进程可捕获  │
    │  │   并自定义处理）→ 等待进程退出                                  │
    │  │   若超过 terminationGracePeriodSeconds → 发送 SIGKILL（强杀   │
    │  │   信号，进程无法捕获，立即被内核终止）                           │
    │  │                                                             │
    │  └─────────────────────────────────────────────────────────────┘
```

**核心矛盾：** 路径 A 和路径 B 是同时启动的。路径 A 中 API Server 虽然立即修改了 Endpoints，但 kube-proxy 和 Ingress Controller 需要 1~3 秒才能感知变更并更新各自的路由规则，在此期间新请求仍然会被路由到旧 Pod。而路径 B 中如果没有 preStop hook，SIGTERM 会瞬间到达应用进程，应用立即开始拒绝请求——但此时流量摘除还没传播完，仍有新请求涌入，这些请求就会失败。

### Endpoints 摘除传播 ≠ readinessProbe

这是两个完全不同的机制：


|          | Endpoints 摘除                              | readinessProbe 失败导致的摘除                      |
| -------- | ----------------------------------------- | ------------------------------------------- |
| **触发方式** | Pod 标记为 Terminating 时 API Server **立即**删除 | kubelet 探测失败 N 次后上报                         |
| **速度**   | 事件驱动（watch），传播 1~3 秒                      | `periodSeconds × failureThreshold`（默认 30 秒） |
| **本质**   | 网络传播延迟                                    | 主动探测间隔                                      |


Pod 终止时的 Endpoints 摘除是事件驱动的即时操作，不依赖 readinessProbe 的探测周期。传播延迟来自 kube-proxy 和 Ingress Controller 处理 watch 事件的时间。

---

## 第三层：preStop Hook — 解决并行时序问题的关键

preStop hook 是 K8s 提供的 Pod 生命周期钩子，**在 SIGTERM 发送之前执行**。它的作用是在路径 B 中插入一个延迟，让应用在传播窗口内保持完全健康，等路径 A 先完成再开始真正的终止流程：

```text
没有 preStop:    标记 Terminating → 立即 SIGTERM → 应用开始拒绝
有 preStop:      标记 Terminating → 先执行 preStop (sleep 10) → 然后才发 SIGTERM
```

配置示例：

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "echo preStop && sleep 10"]
```

加入 preStop 后的完整时间线：

```
T+0s    K8s 标记 Pod 为 Terminating
        │
        ├── 路径 A：API Server 删除 Endpoints（即时）
        │     ├── T+1~2s  kube-proxy 更新 iptables
        │     └── T+2~3s  Ingress Controller reload nginx upstream
        │     └── T+3s    旧 Pod 从所有流量路径中移除 ✅
        │
        └── 路径 B：开始执行 preStop hook: sleep 10
              │
              │  T+0s ~ T+10s：
              │  ├── 应用进程完全不知道要关闭
              │  ├── _shutting_down = False
              │  ├── 如果有请求漏进来（传播窗口内），正常处理，返回 200 ✅
              │  └── 到 T+3s 时路径 A 已完成，不再有新请求进来
              │
              T+10s   sleep 结束 → K8s 发送 SIGTERM
              │
              │  此时 Pod 已从 Endpoints 摘除完毕
              │  不会有新请求进来，安心处理存量任务
              │
              │  → supervisord 收到 SIGTERM → 转发给子进程
              │  → 应用开始优雅关闭流程（见下文）
              │
              T+10s ~ T+600s  等待存量任务完成
              │
              T+600s  若仍未退出 → SIGKILL 强杀
```

**preStop sleep 的精妙之处：** 在传播延迟窗口内，Pod 还是完全健康的，漏进来的少量请求照常处理，不会返回 503，不会失败。等 sleep 结束发 SIGTERM 时，已经没有新流量了。

> [!example] 🔗 实战链接：generic-deployer 的默认 preStop hook
> 在 `deploy/generic-deployer/values.yaml` 中，preStop hook 已被设为所有服务的**默认配置**：
> ```yaml
> lifecycle:
>   preStop:
>     exec:
>       command: ["/bin/sh", "-c", "echo preStop && sleep 10"]
> ```
> Deployment 模板通过 `{{- with .Values.lifecycle }}` 将其注入容器：
> ```yaml
> {{- with .Values.lifecycle }}
> lifecycle:
>   {{- toYaml . | nindent 12 }}
> {{- end }}
> ```
> **解读**：所有使用 `generic-deployer` 的服务默认都有 10 秒的 preStop sleep，不需要每个服务单独配置。这意味着公司的标准部署模板已经内置了"第三层"保护——SIGTERM 前先等 10 秒让 Endpoints 摘除传播完成。

> [!example] 🔗 实战链接：plaud-project-summary 各环境的 preStop 配置
> `plaud-project-summary` 是一个 AI 摘要生成服务，在**所有区域和泳道**都显式配置了 preStop hook，包括 main、workspace、preview：
> ```yaml
> # deploy/plaud-project-summary/values/ap-northeast-1/staging/main.yaml
> lifecycle:
>   preStop:
>     exec:
>       command: ["/bin/sh", "-c", "echo preStop && sleep 10"]
> ```
> **解读**：虽然 generic-deployer 已有默认 preStop，plaud-project-summary 仍然在每个 values 文件中显式声明，确保不依赖默认值。这是一种防御性做法——万一某人修改了 Chart 默认值，不会影响关键服务的优雅关闭行为。

### 为什么不用 proxy-next-upstream？（进阶）

Nginx Ingress 支持配置 `proxy-next-upstream: http_503`，让 Nginx 在收到 503 时自动重试下一个后端。但这个方案有缺陷：

1. **健康检查服务在"服务真挂了"时也返回 503**（核心服务不健康 → `overall = "unhealthy"` → 503），重试到所有 Pod 都失败才返回，增加无意义延迟
2. **API 端口关闭时抛的是 RuntimeError → 500 而非 503**，`http_503` 根本拦不住
3. **preStop hook 已经从根本上解决了问题**，不需要在 Ingress 层打补丁

---

## 第四层：supervisord — 不要让它抢先杀进程（进阶）

当容器使用 supervisord 作为 PID 1 管理多个进程时，SIGTERM 信号的传递链为：

```
K8s 发送 SIGTERM
    → supervisord (PID 1) 收到
        → 转发 SIGTERM 给所有子进程
            → 等待 stopwaitsecs 秒
                → 超时则发 SIGKILL 强杀子进程
```

**踩坑：** supervisord 默认 `stopwaitsecs = 10`，即只等 10 秒就会 SIGKILL 子进程。即使 K8s 给了 600 秒的优雅期，supervisord 会在 10 秒后抢先杀掉还在执行任务的进程！

### 解决方案

在 `supervisord.conf` 中为需要长时间等待的进程设置 `stopwaitsecs`：

```ini
[program:api]
command=uvicorn api.server:app --host 0.0.0.0 --port 8001
directory=/app
autostart=true
autorestart=true
stopwaitsecs=590   # 比 K8s 的 600s 略小，留缓冲给 supervisord 自身退出
```

### supervisord 关键配置一览


| 配置项          | 默认值   | 说明                                   |
| ------------ | ----- | ------------------------------------ |
| stopwaitsecs | 10    | 发送 SIGTERM 后等待进程退出的最大秒数，超时后发 SIGKILL |
| stopsignal   | TERM  | 停止进程时发送的信号类型                         |
| stopasgroup  | false | 是否向整个进程组发送停止信号                       |
| killasgroup  | false | 是否向整个进程组发送 SIGKILL                   |


---

## 第五层：应用层优雅关闭（进阶）

SIGTERM 到达应用进程后，需要做三件事：停止接收新任务、等待存量任务完成、退出进程。

### 1. 健康检查返回 503

收到 SIGTERM 后健康检查立即返回 503。虽然此时 Pod 已从 Endpoints 摘除（preStop 保证了这一点），503 主要作为防御性措施：

```python
_shutting_down = False

def _handle_sigterm(*_):
    global _shutting_down
    _shutting_down = True

signal.signal(signal.SIGTERM, _handle_sigterm)

@app.get("/health")
async def health_check():
    if _shutting_down:
        return JSONResponse(status_code=503, content={"status": "shutting_down"})
    return {"status": "healthy"}
```

### 2. 后台任务追踪与排空

核心思路：用一个 `set` 追踪所有运行中的后台任务，关闭时 `await` 等待全部完成：

```python
_running_tasks: set[asyncio.Task] = set()
_shutting_down = False

def schedule_background_task(coro):
    """创建受追踪的后台任务。"""
    if _shutting_down:
        coro.close()
        raise RuntimeError("Cannot schedule tasks during shutdown")
    task = asyncio.create_task(coro)
    _running_tasks.add(task)
    task.add_done_callback(_running_tasks.discard)  # 完成后自动移除
    return task

async def wait_for_running_tasks():
    """阻塞直到所有后台任务完成。在 lifespan shutdown 阶段调用。"""
    global _shutting_down
    _shutting_down = True
    if _running_tasks:
        await asyncio.gather(*_running_tasks, return_exceptions=True)
```

### 3. FastAPI Lifespan 集成

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    yield  # 应用运行中
    # 关闭阶段：等待所有任务完成
    await wait_for_running_tasks()

app = FastAPI(lifespan=lifespan)
```

---

## 完整滚动更新时间线（replicas=2）

将所有层串联起来，一次完整的滚动更新过程：

```
════════════════════════════════════════════════════════════════════
 阶段一：创建新 Pod
════════════════════════════════════════════════════════════════════

  旧Pod-A ───────── Running ✅ ── 接流量
  旧Pod-B ───────── Running ✅ ── 接流量
  新Pod-1 ───────── Starting ⏳   拉镜像、启动中         可用数: 2

  │ 新Pod-1 通过 readinessProbe（可用数变为 3，有余量）
  ▼

════════════════════════════════════════════════════════════════════
 阶段二：终止旧Pod-A（maxUnavailable=0 保证可用数 ≥ 2）
════════════════════════════════════════════════════════════════════

  旧Pod-A:
    T+0s     标记 Terminating
             ├── Endpoints 开始摘除（传播 1~3s）
             └── preStop: sleep 10（Pod 仍健康，正常服务）
    T+3s     Endpoints 摘除传播完成，不再有新流量
    T+10s    sleep 结束 → SIGTERM → supervisord → 子进程
             └── 应用设置 _shutting_down = True
             └── 等待存量任务完成...
    T+10s~600s  存量任务完成 → 进程退出 → Pod 终止

  旧Pod-B ───────── Running ✅ ── 接流量
  新Pod-1 ───────── Ready   ✅ ── 接流量               可用数: ≥ 2

  │ 同时创建新Pod-2，通过 readinessProbe 后开始终止旧Pod-B
  ▼

════════════════════════════════════════════════════════════════════
 阶段三：更新完成
════════════════════════════════════════════════════════════════════

  新Pod-1 ───────── Running ✅ ── 接流量
  新Pod-2 ───────── Running ✅ ── 接流量               可用数: 2
  旧Pod-A/B ─────── 已终止（存量任务均已完成）
```

**此时发起新任务会怎样？** 在任何时刻，新任务都会被路由到 Ready 状态的 Pod（新 Pod 或尚未开始终止的旧 Pod），不会丢失。

---

## 四层超时的正确配置

```
┌─────────────────────────────────────────────────────────────┐
│  K8s terminationGracePeriodSeconds = 600s (最外层)           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  preStop: sleep 10s                                   │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  supervisord stopwaitsecs = 590s                │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │  应用内部任务超时 = 按业务设定 (如 300s)     │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

关系：terminationGracePeriodSeconds ≥ preStop sleep + stopwaitsecs
      stopwaitsecs ≈ terminationGracePeriodSeconds - preStop sleep
```


| 层           | 配置项                             | 值         | 说明                            |
| ----------- | ------------------------------- | --------- | ----------------------------- |
| K8s         | `terminationGracePeriodSeconds` | 600s      | 从 preStop 开始计时的总上限            |
| preStop     | `lifecycle.preStop.exec`        | sleep 10s | 为 Endpoints 摘除传播争取时间          |
| supervisord | `stopwaitsecs`                  | 590s      | SIGTERM 后等子进程退出的时间，略小于 600-10 |
| 应用          | 业务超时                            | 按需设定      | 单个任务的最大执行时间                   |


> [!example] 🔗 实战链接：不同服务的 terminationGracePeriodSeconds 对比
> generic-deployer 模板中，`terminationGracePeriodSeconds` 默认值为 45 秒：
> ```yaml
> # deploy/generic-deployer/templates/deployment.yaml
> terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds | default 45 }}
> ```
> 但各服务会根据业务特性覆盖这个值。以下是几个真实服务的配置对比：
>
> | 服务 | `terminationGracePeriodSeconds` | 原因 |
> |---|---|---|
> | `plaud-project-summary`（AI 摘要） | **600s** | 单个摘要任务可能需要 10 分钟完成 |
> | `plaud-project-recommendation` task-consumer | **1800s** | 推荐任务耗时更长，需要 30 分钟优雅期 |
> | `plaud-project-recommendation` deployer (API) | **30s** | API 服务只处理短请求，不需要长等待 |
> | `plaud-share-service` | **600s** | 文件分享服务有长耗时操作 |
> | `plaud-gateway` | **600s** | 网关需等待所有在途请求完成 |
>
> **解读**：同一个服务（如 `plaud-project-recommendation`）的不同组件可以有不同的优雅期——API 服务只需 30 秒，而后台 task-consumer 需要 1800 秒（30 分钟）。这体现了"按业务需求分层设置超时"的原则。

---

## 完整的优雅关闭检查清单

- `maxUnavailable: 0` — 新 Pod ready 前不杀旧 Pod
- `preStop: sleep 10` — SIGTERM 前等待 Endpoints 摘除传播完成
- `terminationGracePeriodSeconds: 600` — 给存量任务足够的完成时间
- supervisord `stopwaitsecs: 590` — 与 K8s 优雅期匹配，不抢先杀进程
- 应用捕获 SIGTERM，设置 `_shutting_down` 标志
- 健康检查关闭时返回 503（防御性措施）
- 后台任务有追踪和排空机制（`_running_tasks` set）
- 关闭期间拒绝新任务（`schedule_background_task` 抛异常）

> [!example] 🔗 实战链接：plaud-project-summary 的完整优雅关闭配置
> 以 `plaud-project-summary` staging 环境为例，将检查清单中的各项与真实配置对应：
>
> ```yaml
> # deploy/plaud-project-summary/values/ap-northeast-1/staging/main.yaml
> deployer:
>   replicaCount: 2                          # 至少 2 副本，配合 maxUnavailable: 0 保证高可用
>   terminationGracePeriodSeconds: 600       # ✅ 10 分钟优雅期（AI 摘要任务最长约 10 分钟）
>   lifecycle:
>     preStop:
>       exec:
>         command: ["/bin/sh", "-c", "echo preStop && sleep 10"]  # ✅ 10 秒 preStop
>   livenessProbe:
>     httpGet:
>       path: /health                        # ✅ 健康检查端点（应用层配合返回 503）
>       port: 8000
>   readinessProbe:
>     httpGet:
>       path: /health                        # ✅ 就绪探针，与滚动更新策略配合
>       port: 8000
> ```
>
> **解读**：这个 values 文件覆盖了检查清单中 K8s 层面的所有配置项。`replicaCount: 2` + 默认的 `maxUnavailable: 0` 确保滚动更新时始终有 Pod 在服务；`terminationGracePeriodSeconds: 600` + `preStop sleep 10` 组合满足了四层超时中的前两层。剩余两层（supervisord `stopwaitsecs` 和应用层代码）在 Docker 镜像内部配置。

---

## 延伸阅读

- Kubernetes 官方文档：Pod Lifecycle - Termination
- supervisord 配置参考：stopwaitsecs
- SIGTERM vs SIGKILL：SIGTERM 可被进程捕获和处理，SIGKILL 无法被捕获，进程会被强制杀死
- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — Pod、Deployment、Ingress 等基础概念和项目部署架构
- [[13-k8s-lane-mechanism|K8s 泳道机制]] — 同一集群内多版本服务的流量隔离

