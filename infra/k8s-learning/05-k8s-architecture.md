# Kubernetes 架构原理

> 前置知识：[[02-k8s-core-concepts|K8s 核心概念]]（Pod、Deployment、Service、Namespace 等资源对象）。
> 02 中提到"K8s 的核心是声明式 + Reconcile Loop"，本文展开讲：K8s 内部**如何实现**这套机制——请求从 git push 到 Pod 运行，完整穿越了哪些组件，Reconcile Loop 和 Controller 链具体怎么工作。
>
> 延伸阅读：[[10-helm-argocd-deployment|Helm 与 EKS 部署体系]]、[[07-k8s-scheduling-resources|调度与资源管理]]、[[11-k8s-extension-mechanisms|扩展机制（CRD 与 Operator）]]。

---

## 一、K8s 解决什么问题——架构视角

[[02-k8s-core-concepts|02]] 介绍了 K8s 的核心资源对象（Pod、Deployment、Service…）以及声明式思维。但仅知道"声明 `replicas: 2`，K8s 就能维持 2 个副本"还不够——架构层面需要回答的问题是：

- 声明写好了，**谁收到这个声明**？（API Server）
- 声明存在哪里，重启后还在吗？（etcd）
- 新 Pod 该跑在哪台机器上？（Scheduler）
- 谁负责"发现现实偏离声明→自动修正"？（Controller Manager / Reconcile Loop）
- 最终谁在节点上拉镜像、启动容器？（kubelet + containerd）

本文以 plaud-project-summary 的一次真实部署为线索，追踪请求从 `git push` 到 Pod 接收流量的完整路径，逐个拆解这些组件。

---

## 二、整体架构——控制平面 vs 数据平面

K8s 集群由两部分组成：**控制平面**（Control Plane，决策层）和**数据平面**（Data Plane，执行层）。

### 2.1 plaud-project-summary 部署链路穿越图

先看全貌——一次 `git push` 如何穿越整个架构：

```text
git push（修改 image.tag: "7e39d6c"）
  │
  ▼
ArgoCD 检测 Git 变更 → helm template 渲染 → diff 对比 → apply
  │
  ▼
┌──────────────────── 控制平面 ─────────────────────────────────────┐
│                                                                   │
│  ① API Server 收到请求                                             │
│     → 认证、授权、准入控制                                           │
│     → 写入 etcd ②                                                 │
│                                                                   │
│  ③ Deployment Controller（watch 到变更）                            │
│     → 创建新 ReplicaSet                                            │
│                                                                   │
│  ④ ReplicaSet Controller（watch 到新 ReplicaSet）                   │
│     → 创建 2 个 Pod（spec.nodeName 为空）                            │
│                                                                   │
│  ⑤ Scheduler（watch 到未调度的 Pod）                                 │
│     → 过滤 + 打分 → 为每个 Pod 选定节点                               │
│                                                                   │
└───────────────────────┬───────────────────────────────────────────┘
                        │ HTTPS
┌───────────────────────┼───── 数据平面 ────────────────────────────┐
│                       │                                           │
│  ⑥ kubelet（watch 到分配给本节点的 Pod）                              │
│     → 调用 containerd 拉取 ECR 镜像                                 │
│     → 创建容器、设置 cgroups                                        │
│     → 启动健康检查（liveness / readiness）                           │
│     → Pod Ready → 报告状态给 API Server                             │
│                                                                   │
│  ⑦ Endpoint Controller（watch 到 Pod Ready）                       │
│     → 将 Pod IP 加入 Service 的 Endpoints                          │
│                                                                   │
│  ⑧ kube-proxy（watch 到 Endpoints 变更）                            │
│     → 更新 iptables 规则 → Pod 开始接收流量                           │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 2.2 架构全景图

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                         Control Plane（控制平面）                         │
│                                                                         │
│  ┌──────────────┐  ┌─────────────┐  ┌───────────┐  ┌────────────────┐  │
│  │  API Server   │  │    etcd     │  │ Scheduler │  │   Controller   │  │
│  │  (唯一入口)    │  │  (状态存储)  │  │  (调度器)  │  │   Manager      │  │
│  │              │  │             │  │           │  │  (控制器管理器)  │  │
│  └──────┬───────┘  └──────┬──────┘  └─────┬─────┘  └──────┬─────────┘  │
│         │                 │               │                │            │
│         └────────── 所有组件通过 API Server 通信 ─────────────┘            │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │ HTTPS（API 调用）
┌─────────────────────────────────┼───────────────────────────────────────┐
│                         Data Plane（数据平面）                            │
│                                 │                                       │
│  ┌─── Node 1 ──────────────┐   │   ┌─── Node 2 ──────────────┐        │
│  │  ┌─────────┐ ┌────────┐ │   │   │  ┌─────────┐ ┌────────┐ │        │
│  │  │ kubelet │ │kube-   │ │   │   │  │ kubelet │ │kube-   │ │        │
│  │  │         │ │proxy   │ │   │   │  │         │ │proxy   │ │        │
│  │  └────┬────┘ └────────┘ │   │   │  └────┬────┘ └────────┘ │        │
│  │       │                  │   │   │       │                  │        │
│  │  ┌────┴──────────────┐  │   │   │  ┌────┴──────────────┐  │        │
│  │  │   containerd      │  │   │   │  │   containerd      │  │        │
│  │  │  ┌─────┐ ┌─────┐  │  │   │   │  │  ┌─────┐ ┌─────┐  │  │        │
│  │  │  │Pod A│ │Pod B│  │  │   │   │  │  │Pod C│ │Pod D│  │  │        │
│  │  │  └─────┘ └─────┘  │  │   │   │  │  └─────┘ └─────┘  │  │        │
│  │  └───────────────────┘  │   │   │  └───────────────────┘  │        │
│  └─────────────────────────┘   │   └─────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

核心设计原则：**所有组件只与 API Server 通信**。Scheduler 不直接跟 kubelet 说话，Controller 不直接写 etcd。这种星形拓扑让每个组件可以独立升级、独立扩展。

---

## 三、控制平面组件

在 EKS 中，控制平面由 AWS 托管（不需要自行维护），但理解每个组件对排障和架构决策至关重要。

### 3.1 API Server — 唯一入口

API Server 是控制平面的前端，**所有操作都必须经过它**——无论来自 kubectl、ArgoCD 还是集群内部的 Controller。

**plaud-project-summary 的部署请求是怎么到达 API Server 的？**

每个 EKS 集群有独立的 API Server 地址。ArgoCD 的 ApplicationSet 通过 `server` 字段指定目标集群：

```yaml
# deploy/plaud-project-summary/applicationsets/applicationsets.yaml（简化）
generators:
  - list:
      elements:
        - cluster: jp-staging
          server: https://0AE3C052...gr7.ap-northeast-1.eks.amazonaws.com
        - cluster: us-staging
          server: https://C85149DE...gr7.us-west-2.eks.amazonaws.com
        - cluster: cn-staging
          server: https://4F8B0222...yl4.cn-northwest-1.eks.amazonaws.com.cn
```

当 `git push` 修改了 image tag 后，ArgoCD 执行四个步骤：

```text
1. git pull      — 拉取最新代码
2. helm template — 渲染 Chart + values 为完整 K8s YAML
3. diff          — 对比渲染结果与集群当前状态
4. apply         — 有差异则通过 K8s Go SDK 发 HTTPS 请求给对应集群的 API Server
```

> [!info] ArgoCD 与 kubectl 殊途同归
> ArgoCD 不是在后台执行 `kubectl apply`，而是用 Go 语言的 K8s 客户端库直接发 HTTP 请求。但 API Server 端收到的东西完全一样——都是一段 YAML/JSON 格式的资源定义。
>
> ```text
> 手动部署：kubectl（命令行工具）→ HTTP 请求 → API Server
> 自动部署：ArgoCD（K8s Go SDK） → HTTP 请求 → API Server
>                                   ↑ 到这里完全一样
> ```

**API Server 收到请求后的处理流程**：

```text
HTTPS 请求到达 API Server
  ├── 1. 认证（Authentication）：请求方是谁？（kubeconfig 证书 / ServiceAccount token）
  ├── 2. 授权（Authorization）：有权限操作这个资源吗？（RBAC 检查）
  ├── 3. 准入控制（Admission Control）：
  │       ├── Mutating Webhook — 修改请求（如注入 sidecar）
  │       └── Validating Webhook — 校验合法性（如拒绝没设 resource limits 的 Pod）
  ├── 4. 写入 etcd — 持久化新的期望状态
  └── 5. 通知 watch 者 — Controller、Scheduler 感知变更
```

**关键特性**：

- **唯一的 etcd 访问者**：其他组件（Scheduler、kubelet）都不直接读写 etcd，全部通过 API Server
- **Watch 机制**：组件可以 watch 资源变更（"有新 Pod 待调度""Deployment 副本数变了"），这是 K8s 事件驱动架构的核心
- **无状态**：API Server 本身不保存数据，可以水平扩展（EKS 自动做了这件事）

### 3.2 etcd — 集群的"数据库"

etcd 是分布式键值存储，保存集群的**全部状态**——所有资源对象的期望状态和当前状态都在这里：

```text
etcd 中存储的内容示例：
├── /registry/deployments/plaud-project-summary/...    ← Deployment 定义
├── /registry/replicasets/plaud-project-summary/...    ← ReplicaSet
├── /registry/pods/plaud-project-summary/...           ← Pod 状态
├── /registry/services/plaud-project-summary/...       ← Service 定义
├── /registry/secrets/plaud-project-summary/...        ← Secret 数据
└── ...
```

**核心特性**：

- **强一致性**（Raft 共识算法）：多副本数据一致，不会出现"节点 A 看到 3 个 Pod、节点 B 看到 2 个"
- **Watch 支持**：API Server watch etcd 变更，再转发给其他组件
- **单点故障风险**：etcd 不可用 = 整个集群无法工作（EKS 自动做了跨 AZ 高可用）

> 性能提示：etcd 对磁盘 IO 敏感，大量 CRD 或频繁 watch 会带来压力。这是 K8s 建议单集群不超过 5000 节点的原因之一。

### 3.3 Scheduler — "Pod 应该跑在哪个节点"

当新 Pod 被创建但尚未分配节点时（`spec.nodeName` 为空），Scheduler 负责选择目标节点。

**决策的核心输入是 `resources.requests`**。plaud-project-summary 在不同环境配了不同的资源需求：

```yaml
# deploy/plaud-project-summary/values/us-west-2/staging/main.yaml（简化）
resources:
  requests: { cpu: 1, memory: 4Gi }
  limits:   { cpu: 1, memory: 4Gi }

# deploy/plaud-project-summary/values/us-west-2/prod/main.yaml（简化）
resources:
  requests: { cpu: 8, memory: 16Gi }
  limits:   { cpu: 8, memory: 16Gi }
```

Scheduler 的三阶段决策：

```text
1. 过滤（Filtering）— 排除不满足条件的节点
   ├── 节点剩余资源 < Pod 的 requests（16Gi 的 Pod 不会调度到只剩 8Gi 的节点）
   ├── 节点有 taint，Pod 没有对应 toleration
   ├── Pod 指定了 nodeSelector / nodeAffinity，节点不匹配
   └── 节点处于磁盘/内存/PID 压力

2. 打分（Scoring）— 在候选节点中选最优
   ├── 资源均衡度（尽量让各节点负载均匀）
   ├── Pod 亲和/反亲和偏好
   ├── 拓扑分散约束（TopologySpreadConstraints）
   └── 镜像是否已缓存在该节点

3. 绑定（Binding）— 写入 Pod 的 spec.nodeName
```

> 上面的配置中 requests = limits，意味着该 Pod 获得 **Guaranteed QoS**——资源紧张时最后被驱逐。详见 [[07-k8s-scheduling-resources|调度与资源管理]]。

### 3.4 Controller Manager — "让现实追上期望"

Controller Manager 运行着一组**控制器（Controller）**，每个控制器负责一种资源类型。这里需要展开讲两个关键机制：**Reconcile Loop** 和 **Controller 链式传递**。

#### Reconcile Loop（调谐循环）

这是 K8s 最核心的运行机制。每个 Controller 都在不断执行同一个循环：

```text
┌────────────────────────────────────────────────────────────┐
│                   Reconcile Loop                           │
│                                                            │
│   ┌──────────────┐     对比      ┌──────────────┐          │
│   │  期望状态      │ ──────────→  │  当前状态      │          │
│   │ (etcd 中的    │              │ (集群中实际    │          │
│   │  spec 字段)   │              │  运行的资源)   │          │
│   └──────────────┘              └──────────────┘          │
│          │                            │                    │
│          └──────── 不一致？ ────────────┘                    │
│                     │                                      │
│              ┌──────┴──────┐                               │
│              │ 执行修正动作  │                               │
│              │ (创建/删除/  │                               │
│              │  更新资源)   │                               │
│              └─────────────┘                               │
│                                                            │
│   一致？→ 什么都不做，等待下一次事件触发                        │
└────────────────────────────────────────────────────────────┘
```

以 Deployment Controller 为例，伪代码逻辑：

```python
# 伪代码 — Deployment Controller 的 Reconcile 逻辑
def reconcile(deployment):
    desired_replicas = deployment.spec.replicas          # 期望状态：2
    current_pods = list_pods(selector=deployment.selector)
    actual_replicas = len(current_pods)                  # 当前状态：1（有一个刚挂了）

    if actual_replicas < desired_replicas:
        create_pods(desired_replicas - actual_replicas)  # 创建 1 个新 Pod
    elif actual_replicas > desired_replicas:
        delete_pods(actual_replicas - desired_replicas)  # 删掉多余的
    # 相等则什么都不做

    # 不是 sleep 轮询，而是 watch 事件驱动：
    # Pod 被删除 → API Server 通知 → 再次触发 reconcile
```

**为什么这种模式如此强大**：

| 特性 | 说明 |
| --- | --- |
| **自愈** | Pod 被意外删除 → Controller 发现实际数 < 期望数 → 自动创建 |
| **幂等** | 多次 `kubectl apply` 同一份 YAML，结果一样（不会创建重复资源） |
| **最终一致** | 即使中间出现短暂偏离，Controller 会持续修正直到达到期望状态 |
| **事件驱动** | 不是定时轮询，而是 watch 机制触发——高效且实时 |

#### Controller 链式传递：Deployment → ReplicaSet → Pod（进阶）

一个 Deployment 的创建并不直接产生 Pod。多个 Controller 通过 **链式传递** 协作，每个 Controller 只负责自己的一层：

```text
Deployment Controller                ReplicaSet Controller              Scheduler + kubelet
       │                                    │                                │
  watch 到新 Deployment               watch 到新 ReplicaSet              watch 到未调度 Pod
       │                                    │                                │
       ▼                                    ▼                                ▼
  创建 ReplicaSet ──────────────→   创建 N 个 Pod ───────────────→   调度 + 启动容器
  (replicas=2,                     (spec.nodeName 为空)             (选节点、拉镜像、
   template 含镜像/资源)                                              创建容器)
```

具体到 plaud-project-summary 的一次部署（image tag 从 `abc1234` 更新为 `7e39d6c`）：

```text
1. ArgoCD apply → API Server 更新 Deployment 的 image tag

2. Deployment Controller（watch 到 Deployment 变更）：
   → 发现 image tag 变了 → 创建新的 ReplicaSet-v2（replicas=2, image=7e39d6c）
   → 逐步缩小旧的 ReplicaSet-v1

3. ReplicaSet Controller（watch 到 ReplicaSet-v2）：
   → 期望 2 个 Pod，当前 0 个 → 创建 Pod-A 和 Pod-B

4. Scheduler（watch 到 Pod-A 和 Pod-B 的 spec.nodeName 为空）：
   → Pod-A: requests cpu:8, memory:16Gi → 过滤 → Node-3 剩余资源足够 → 绑定
   → Pod-B: 同理 → Node-5

5. kubelet on Node-3（watch 到 Pod-A 被分配过来）：
   → 调用 containerd 拉取 ECR 镜像 → 创建容器 → 启动 readinessProbe
   → Pod Ready → 报告状态给 API Server

6. Endpoint Controller（watch 到 Pod-A Ready）：
   → 将 Pod-A 的 IP 加入 Service 的 Endpoints

7. kube-proxy（watch 到 Endpoints 变更）：
   → 更新 iptables → Pod-A 开始接收流量

8. Deployment Controller 继续滚动：
   → 缩小 ReplicaSet-v1（旧版本 Pod 被优雅终止）
   → 直到所有 Pod 都运行新版本
```

每个 Controller 只关注自己的资源层级，通过 API Server + Watch 机制串联——这就是 K8s "松耦合、可扩展"的根基。

#### 常见控制器一览（深入）

| 控制器 | 职责 |
| --- | --- |
| **Deployment Controller** | 管理 ReplicaSet，执行滚动更新 |
| **ReplicaSet Controller** | 确保指定数量的 Pod 副本在运行 |
| **Node Controller** | 监控节点健康状态，节点失联后驱逐 Pod |
| **Job Controller** | 管理一次性任务的 Pod |
| **Endpoint Controller** | 维护 Service 与 Pod 的映射关系 |
| **ServiceAccount Controller** | 为新 Namespace 创建默认 ServiceAccount |

> ArgoCD 也是一种 Controller——watch Git 仓库的变更，调谐集群状态与 Git 声明的一致。任何人都可以写自定义 Controller 来扩展 K8s。详见 [[11-k8s-extension-mechanisms|CRD 与 Operator 模式]]。

---

## 四、数据平面组件

数据平面由一组 **Worker Node** 组成，每个节点运行三个关键组件。

### 4.1 kubelet — 节点上的代理

kubelet 是运行在每个节点上的进程，负责将 Pod spec 变为实际运行的容器。

**kubelet 的工作指令来自 Pod spec**。以 plaud-project-summary 为例（Helm 渲染后的效果）：

```yaml
# deploy/generic-deployer/templates/deployment.yaml（渲染后，简化展示）
spec:
  terminationGracePeriodSeconds: 600
  containers:
    - name: deployer
      image: "<account>.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:7e39d6c"
      resources:
        requests: { cpu: 8, memory: 16Gi }
        limits:   { cpu: 8, memory: 16Gi }
      livenessProbe:
        httpGet: { path: /health, port: 8000 }
      readinessProbe:
        httpGet: { path: /health, port: 8000 }
```

kubelet 的完整职责：

```text
1. Pod 管理
   ├── watch API Server 获取分配到本节点的 Pod
   ├── 调用 containerd 创建/销毁容器
   └── 持续报告 Pod 状态给 API Server

2. 健康检查
   ├── livenessProbe → 失败则重启容器
   ├── readinessProbe → 失败则从 Service 的 Endpoints 摘除
   └── startupProbe → 启动期间暂缓其他探针

3. 资源管理
   ├── 向 API Server 报告节点可用资源（Scheduler 据此决策）
   ├── 通过 cgroups 限制容器资源使用
   └── 资源紧张时驱逐低优先级 Pod（Eviction）

4. 卷挂载
   ├── 调用 CSI 驱动挂载 PV
   └── 管理 emptyDir、hostPath 等本地卷
```

### 4.2 kube-proxy — 节点上的网络代理

kube-proxy 负责实现 K8s 的 **Service 网络**——让 Pod 可以通过 Service 名称互相访问：

```text
请求流程：Pod-X 访问 plaud-project-summary-deployer:8001
    │
    ├── CoreDNS 解析：plaud-project-summary-deployer → ClusterIP 10.100.50.23
    │
    ├── kube-proxy 维护的 iptables/IPVS 规则：
    │   10.100.50.23:8001 → {
    │       Pod-A (172.16.0.5:8001)  ← 负载均衡
    │       Pod-B (172.16.0.8:8001)
    │   }
    │
    └── 请求被转发到 Pod-A 的实际 IP
```

两种代理模式：

| 模式 | 实现 | 性能 | 适用 |
| --- | --- | --- | --- |
| **iptables**（默认） | iptables 规则做 DNAT | Service 多时规则链长，O(n) | 中小集群 |
| **IPVS** | 内核级负载均衡 | O(1) 查找 | 大集群（>1000 Service） |

> 在 [[12-k8s-pod-graceful-shutdown#第二层：Pod 终止流程 — 两条并行路径|Pod 优雅终止]] 中，kube-proxy watch 到 Endpoints 变更后更新 iptables 需要 1~2 秒——这就是需要 preStop hook 争取时间的原因。

### 4.3 Container Runtime — 容器运行时（深入）

kubelet 通过 **CRI（Container Runtime Interface）** 调用容器运行时来管理容器：

```text
kubelet
  ↓ CRI gRPC 调用
containerd
  ↓
runc（创建 Linux namespace + cgroups → 隔离的容器进程）
```

EKS 默认使用 containerd。详见 [[01-docker-basics#10.1 Docker 不是唯一的容器运行时|Docker 笔记的容器运行时章节]]。

---

## 五、声明式 API 与控制器模式（深入）

前几节介绍了各组件的职责，本节从更高层次理解 K8s 的设计哲学。

### 5.1 命令式 vs 声明式

```bash
# 命令式 — 告诉系统"怎么做"
docker run -d --name api -p 8001:8001 myimage      # 启动一个容器
docker run -d --name api2 -p 8002:8001 myimage     # 再启动一个
# 问题：api 挂了没人知道，也没人重启
```

```yaml
# 声明式 — 告诉系统"要什么状态"
# K8s 负责：创建 2 个 Pod，挂了自动重建，更新时滚动替换
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 2
  template:
    spec:
      containers:
        - image: myimage
          ports: [{ containerPort: 8001 }]
```

plaud-project-summary 的声明式体现——不需要手动 `docker run`，只需在 values 中声明期望：

```yaml
# deploy/plaud-project-summary/values/us-west-2/prod/main.yaml（简化）
replicaCount: 2                              # "保持 2 个副本"
autoscaling:
  enabled: true
  minReplicas: 2                             # "最少 2 个"
  maxReplicas: 10                            # "最多 10 个"
  targetCPUUtilizationPercentage: 75         # "CPU 超 75% 就扩容"
```

不需要写脚本监控 CPU 并手动扩容。HPA Controller 持续 reconcile：观察 CPU 指标 → 对比阈值 → 调整副本数。

### 5.2 Reconcile Loop 的四大特性

在 3.4 节已经展示了 Reconcile Loop 的运行机制，这里归纳其设计优势：

1. **自愈**：Pod 被意外删除 → Controller 发现偏离 → 自动创建新 Pod
2. **幂等**：多次 apply 同一份 YAML，结果一样（不会创建重复资源）
3. **可组合**：不同 Controller 管不同层级（Deployment → ReplicaSet → Pod），彼此独立运行
4. **可扩展**：任何人都可以编写自定义 Controller（ArgoCD、Prometheus Operator 都是）

### 5.3 ArgoCD 的 syncPolicy — Reconcile Loop 的真实体现

ArgoCD 本身就是一个 Controller，它的 `syncPolicy` 配置直接对应 Reconcile Loop 的行为：

```yaml
# deploy/plaud-project-summary/applicationsets/applicationsets.yaml（staging）
syncPolicy:
  automated:
    prune: false        # 安全制动：Git 中删掉的资源不自动从集群删除
    selfHeal: true      # 有人手动改了集群状态，自动改回 Git 中的声明

# deploy/plaud-project-summary/applicationsets/applicationsets-prod.yaml（prod）
# syncPolicy:           # 整段注释 — 所有变更需人工触发
```

两个配置项对应两种 reconcile 场景：

- **`selfHeal: true`**：有人用 `kubectl scale` 把副本数改成 5，但 values 中声明的是 2（当前状态偏离期望状态）→ ArgoCD 自动改回 2。经典的 Reconcile Loop。
- **`prune: false`**：从 values 中删掉了 `ingresses.public` 配置，Helm 不再渲染 public Ingress（期望状态变了）→ ArgoCD **不**自动删除集群中已有的资源。这是对 Reconcile 的"安全制动"——删除操作风险大，交给人工确认。

Staging 开启 `selfHeal`（快速迭代），Prod 整个 `automated` 注释掉（人工审核后触发），是安全与效率的 tradeoff。详见 [[10-helm-argocd-deployment#3.3 ② application.yaml — ArgoCD 入口|Helm 笔记 syncPolicy 详解]]。

---

## 六、EKS 特殊之处（进阶）

AWS EKS（Elastic Kubernetes Service）是托管的 K8s。理解 EKS 的责任划分，才能知道出了问题该找 AWS 还是自己排查。

### 6.1 责任划分

```text
┌──────────────────────────────────────────────────────┐
│                AWS 托管（无需自行维护）                  │
│  ┌────────────────────────────────────────────────┐  │
│  │  API Server（跨 3 个 AZ 高可用）                  │  │
│  │  etcd（自动备份、加密）                            │  │
│  │  Controller Manager / Scheduler                 │  │
│  │  CoreDNS / kube-proxy（托管插件）                 │  │
│  └────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────┤
│                用户管理                               │
│  ┌────────────────────────────────────────────────┐  │
│  │  Worker Nodes（EC2 / Fargate）                  │  │
│  │  应用部署（Deployment / Service / Ingress）       │  │
│  │  Ingress Controller（nginx-ingress 等）          │  │
│  │  监控（Prometheus / Grafana）                    │  │
│  │  日志（Fluent Bit / CloudWatch）                 │  │
│  │  网络策略（SecurityGroup / NetworkPolicy）        │  │
│  │  IAM 权限（IRSA）                                │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

plaud-project-summary 的 ApplicationSet 展示了多集群 EKS 架构——5 个区域、5 个独立的 EKS 集群：

```yaml
# deploy/plaud-project-summary/applicationsets/applicationsets-prod.yaml（简化）
generators:
  - list:
      elements:
        - cluster: jp-prod     # 日本
          server: https://686061FA...yl4.ap-northeast-1.eks.amazonaws.com
        - cluster: eu-prod     # 欧洲
          server: https://055042...gr7.eu-central-1.eks.amazonaws.com
        - cluster: sg-prod     # 新加坡
          server: https://96C04C...yl4.ap-southeast-1.eks.amazonaws.com
        - cluster: us-prod     # 美国
          server: https://3F1BDD...sk1.us-west-2.eks.amazonaws.com
        - cluster: cn-prod     # 中国（注意域名后缀 .amazonaws.com.cn）
          server: https://D2572E...yl4.cn-northwest-1.eks.amazonaws.com.cn
```

每个集群的控制平面由 AWS 独立托管，用户只需管理 Worker Node 上运行的应用。

### 6.2 Node 类型

| 类型 | 特点 | 适用场景 |
| --- | --- | --- |
| **Managed Node Group** | AWS 管理 EC2 生命周期（升级、补丁） | 大多数工作负载 |
| **Self-Managed Nodes** | 完全控制 EC2 实例 | 需要自定义 AMI 或特殊实例类型 |
| **Fargate** | 无服务器，按 Pod 计费 | 批处理、低频服务 |
| **Karpenter** | 根据 Pod 需求自动创建最优节点 | 动态负载（详见 [[07-k8s-scheduling-resources#六、Cluster Autoscaler 与 Karpenter（深入）\|调度笔记]]） |

### 6.3 网络模型

EKS 使用 **AWS VPC CNI**，每个 Pod 直接获得 VPC 子网内的真实 IP：

```text
VPC: 10.0.0.0/16
├── Subnet A: 10.0.1.0/24
│   ├── Node-1 (EC2): 10.0.1.10
│   │   ├── Pod-A: 10.0.1.11    ← 真实 VPC IP
│   │   └── Pod-B: 10.0.1.12
│   └── Node-2 (EC2): 10.0.1.20
│       └── Pod-C: 10.0.1.21
└── Subnet B: 10.0.2.0/24
    └── Node-3 (EC2): 10.0.2.10
        └── Pod-D: 10.0.2.11
```

**优势**：Pod 可直接与 VPC 内的 AWS 资源（RDS、ElastiCache）通信，无需额外网络跳转。

**限制**：每个 EC2 实例的 ENI 数量有限，决定了单节点最多能跑多少 Pod。详见 [[04-k8s-networking#2.2 AWS VPC CNI（EKS 默认）（进阶）|K8s 网络笔记]]。

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — Helm + ArgoCD 如何将应用部署到 EKS
- [[03-k8s-workload-types|工作负载类型]] — Deployment 之外的 StatefulSet、DaemonSet、Job
- [[04-k8s-networking|网络深入]] — Service、CNI、CoreDNS、NetworkPolicy 的实现原理
- [[07-k8s-scheduling-resources|调度与资源管理]] — Scheduler 决策机制、QoS、自动扩缩容
- [[08-k8s-security-rbac|安全与权限]] — RBAC、Pod Security、Secret 管理
- [[11-k8s-extension-mechanisms|扩展机制]] — CRD、Operator、Admission Webhook
