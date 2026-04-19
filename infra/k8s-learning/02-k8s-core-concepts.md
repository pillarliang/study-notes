# K8s 核心概念入门

---

## 一、从 Docker 到 K8s —— 你需要哪些新概念

Docker 解决了"把应用打包成容器"的问题。但当你有几十个微服务、成百上千个容器分布在多台机器上时，你需要一个系统来自动管理它们——这就是 Kubernetes（简称 K8s）。

在学习 K8s 之前，先建立一个整体认知：

```
Docker 世界（一台机器）              K8s 世界（多台机器组成的集群）
─────────────────────              ──────────────────────────────
docker run → 启动一个容器            kubectl apply → 声明"我要什么状态"
手动管理容器生命周期                   K8s 自动管理、自愈、扩缩容
容器用 IP + 端口访问                  Service 提供稳定的 DNS 名称
配置写在环境变量 / 文件里              ConfigMap / Secret 集中管理配置
```

K8s 引入了一组**资源对象（Resource）**来描述你的应用应该怎么运行。接下来逐一介绍最核心的几个。

### 什么是"资源对象"？

传统方式（Docker / 手动运维）中，你直接执行命令来运行应用——`docker run`、`kill`、手动配置端口、手动重启崩溃的进程。你告诉机器**"做什么"**（命令式）。

K8s 的方式不同：你不直接操作容器，而是写一份描述文件（YAML），告诉 K8s **"我想要什么状态"**（声明式）。K8s 自己去实现和维持这个状态。

这些描述文件里用到的就是**资源对象**——K8s 提供的一套标准化"词汇"，每种资源描述应用的一个方面：

| 你要描述的事情 | 用哪个资源对象 |
|------------|------------|
| "我的应用跑几个副本、用什么镜像" | Deployment |
| "别人怎么访问我的应用" | Service |
| "外部 HTTP 请求怎么路由进来" | Ingress |
| "我的应用需要什么配置" | ConfigMap / Secret |
| "我的应用需要多少 CPU 和内存" | Pod 的 resources 字段 |

> [!tip] 类比理解
> 如果 K8s 是一个餐厅，资源对象就是菜单上的各种"表单"。你不需要进厨房自己炒菜（手动管理容器），只需要填好表单——"我要 3 份红烧肉（replicas: 3）、用新鲜五花肉（image: v2）、放在 2 号桌（namespace: team-b）"——厨房（K8s 控制平面）会自动帮你做好。如果有一份掉地上了（Pod 崩溃），厨房会自动重做一份，始终维持你要求的 3 份。

### 声明式 API + 控制器模式

在 K8s 内部，每种资源（Deployment、Service、Pod...）都是 API Server 中的一个"对象"，有统一的结构（`apiVersion`、`kind`、`metadata`、`spec`），存储在 etcd 数据库中。K8s 的各种控制器（Controller）会不断监听它们，确保集群的**实际状态**与你声明的**期望状态**一致——这就是"声明式 API + 控制器模式"的核心思想。

**一次部署背后发生了什么？**

以 plaud-project-summary 为例，当你修改 values 文件中的 `image.tag` 并推送到 Git：

```
你（开发者）
  │
  │  git push（修改 values 中的 image.tag）
  ▼
┌─────────────────┐
│    ArgoCD       │  ← 监听 Git 仓库，发现 values 变了，渲染 Helm 模板，生成新的 Deployment YAML，调用 kubectl apply 提交给 API Server  
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   API Server    │  ← K8s 的"前台"，所有操作都经过它，收到 YAML，验证格式，然后存起来 
└────────┬────────┘
         │  写入
         ▼
┌─────────────────┐
│     etcd        │  ← 分布式数据库，K8s 的"账本"，记录着：my-app 应该用新镜像、跑 3 个副本  
└────────┬────────┘
         │  Deployment Controller 在监听 etcd 的变化
         │  发现："镜像版本变了，需要滚动更新"
         ▼
┌─────────────────┐
│  Controller     │  ← "声明式"的核心执行者
│  （控制器）      │     对比期望状态和实际状态，执行滚动更新
└─────────────────┘
```

> [!info] 开发者 vs 运维的区别
> 运维可能直接使用 `kubectl apply` 提交 YAML。但作为开发者，你不直接操作 K8s——你只需要修改 Git 中的配置（如 Helm values），ArgoCD 会自动完成从 Git 到集群的同步。这就是 **GitOps** 模式：Git 仓库是唯一的真相来源。详见 [[10-helm-argocd-deployment|Helm 与 ArgoCD 部署体系]]。

**"声明式"到底是什么意思？**

对比两种思维方式：

```
命令式（Docker / 脚本）—— 你告诉系统每一步该做什么：
  docker run myapp        # 第 1 步：启动容器
  docker run myapp        # 第 2 步：再启动一个
  docker run myapp        # 第 3 步：再启动一个
  # 如果其中一个挂了？你得自己写脚本检测并重启

声明式（K8s）—— 你只说"我要什么"，不管"怎么做"：
  replicas: 3             # 我要 3 个副本。怎么做到的？我不关心。
```

Controller 会**持续**保证现实和你的声明一致。这不是一次性的，而是一个**永不停歇的循环**（Reconcile Loop）：

```
┌──────────────────────────────────────┐
│       控制器的无限循环（Reconcile Loop） │
│                                        │
│   ① 读取期望状态（etcd 中存储的 YAML）    │
│      → replicas: 3                     │
│                                        │
│   ② 观察实际状态（当前集群里有几个 Pod）    │
│      → 当前有 2 个 Pod 在运行             │
│                                        │
│   ③ 计算差异                             │
│      → 少了 1 个                         │
│                                        │
│   ④ 执行动作                             │
│      → 创建 1 个新 Pod                    │
│                                        │
│   ⑤ 回到 ①，继续监听...                   │
└──────────────────────────────────────┘
```

这就是 K8s 能**自愈**的原因：
- Pod 崩溃了 → Controller 发现实际 2 个、期望 3 个 → 自动创建 1 个
- 节点宕机了 → 上面的 Pod 丢失 → Controller 在其他节点上重建
- 你改了镜像版本 → Controller 发现期望和实际不一致 → 滚动更新

**资源、控制器、etcd 的关系全景**

K8s 不是一个"大程序"来管理所有东西，而是**多个专职控制器各管一类资源**，它们都通过 API Server 读写 etcd：

```
┌─────────────────────────────────────────────────────────────────┐
│                          etcd（数据库）                           │
│                                                                   │
│  存储所有资源对象的期望状态：                                        │
│  ┌────────────┐ ┌────────────┐ ┌─────────┐ ┌───────────────┐    │
│  │ Deployment │ │ ReplicaSet │ │   Pod   │ │    Service    │    │
│  │ replicas:3 │ │ replicas:3 │ │ pod-1   │ │ selector:     │    │
│  │ image:v2   │ │ image:v2   │ │ pod-2   │ │   app=my-app  │    │
│  └────────────┘ └────────────┘ │ pod-3   │ └───────────────┘    │
│                                 └─────────┘                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                     ┌──────┴──────┐
                     │  API Server  │  ← 所有读写 etcd 的操作都必须经过它
                     └──────┬──────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
│  Deployment   │  │  ReplicaSet  │  │   Endpoint        │
│  Controller   │  │  Controller  │  │   Controller      │
│               │  │              │  │                    │
│ 职责：         │  │ 职责：        │  │ 职责：              │
│ 监听Deployment│  │ 监听ReplicaSet│  │ 监听 Service+Pod   │
│ 创建/更新     │  │ 创建/删除 Pod │  │ 维护 Endpoints     │
│ ReplicaSet    │  │ 维持副本数    │  │ (Pod IP 列表)      │
└──────┬───────┘  └──────┬───────┘  └──────────────────┘
       │                 │
       ▼                 ▼
  ReplicaSet           Pod-1, Pod-2, Pod-3
```

**核心要点：每个控制器只关心"自己那一层"**

这是一个**链式传递**的过程，不是一步到位的：

```
你写的 YAML          Deployment Controller      ReplicaSet Controller
     │                      │                          │
     ▼                      ▼                          ▼
 Deployment  ──创建──→  ReplicaSet  ──创建──→   Pod-1, Pod-2, Pod-3
 "我要3副本"           "维持3个Pod"             实际运行的容器
 "用镜像v2"            "用镜像v2"
```

- **Deployment Controller**：只负责"Deployment → ReplicaSet"这一层。你改了镜像版本，它创建一个新 ReplicaSet（新版本），缩小旧 ReplicaSet——这就是滚动更新。
- **ReplicaSet Controller**：只负责"ReplicaSet → Pod"这一层。ReplicaSet 说要 3 个 Pod，它就确保始终有 3 个在跑。
- **Endpoint Controller**：监听 Service 和 Pod，维护一张"Service 背后有哪些 Pod IP"的映射表（Endpoints），让流量知道该去哪。

> [!tip] 为什么要分这么多层？
> 想象一下如果只有一个"超级控制器"管所有事情——代码会极其复杂，一个 bug 影响所有功能。拆分成多个专职控制器，每个逻辑简单、独立运行、互不干扰。这也是 K8s 的设计哲学：**通过组合简单组件来实现复杂行为**。

**统一的四段式结构**

K8s 中所有资源对象都遵循相同的结构，区别只在于 `spec` 里写什么：

```yaml
apiVersion: ___    # 这个对象属于哪个 API 组
kind: ___          # 这是什么类型的对象（Deployment? Service? Pod?）
metadata: ___      # 名称、命名空间、标签等元信息
spec: ___          # 你期望的状态（核心内容，每种资源不同）
```

**各组件的角色**

| 组件 | 角色 | 类比 |
|------|------|------|
| **API Server** | 唯一的入口，接收和验证所有请求 | 餐厅前台，只有它能接单 |
| **etcd** | 存储所有资源对象的数据库 | 账本，记录"客户要了什么" |
| **Controller** | 持续监听，让实际状态匹配期望状态 | 厨房经理，不断检查"还差几道菜" |
| **ArgoCD** | 监听 Git 变化，自动同步到集群 | 自动化的点餐系统，Git 一改就下单 |

> 整句话翻译成大白话：**你在 Git 里写一个配置说"我要什么"，ArgoCD 把它提交给 K8s，K8s 存到数据库里，然后一群后台程序不停地检查"现实和要求是否一致"，不一致就自动修正——永远如此。**

---

## 二、Pod — 最小部署单元

**Pod 是 K8s 中最小的可部署单元**，它包裹一个或多个容器，共享网络和存储。

```
┌─── Pod ─────────────────────┐
│                              │
│  ┌──────────┐  ┌──────────┐ │
│  │ 容器 A    │  │ 容器 B    │ │  ← 同一个 Pod 内的容器共享：
│  │ (主应用)   │  │ (日志收集) │ │     - 同一个 IP 地址
│  └──────────┘  └──────────┘ │     - 同一个 localhost
│         共享网络 + 存储        │     - 可以共享挂载的 Volume
└──────────────────────────────┘
```

**为什么不直接用"容器"？** 因为有些场景需要多个容器紧密协作（如主应用 + 日志收集 sidecar），它们必须跑在同一台机器上、共享网络。Pod 就是这个"共享上下文"的抽象。

**实际使用中**：90% 的 Pod 只有一个容器。你几乎不会直接创建 Pod——而是通过 Deployment 来管理它们。

```yaml
# 一个最简单的 Pod 定义（了解结构即可，实际不会直接写）
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app              # ← Label，后面会详细解释
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
```

> [!example] 🔗 实战：plaud-project-summary 的 Pod 长什么样
> 你不会手写 Pod YAML——在项目中，Pod 由 Deployment 自动创建，最终跑起来的 Pod spec 大致如下：
> ```yaml
> # 由 generic-deployer/templates/deployment.yaml 渲染生成
> spec:
>   terminationGracePeriodSeconds: 600    # Pod 被终止时，最多等 10 分钟让任务完成
>   containers:
>     - name: deployer
>       image: "236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:7e39d6c"
>       ports:
>         - containerPort: 8001           # API 端口
>         - containerPort: 8889           # 前端 UI 端口
>         - containerPort: 8000           # 健康检查端口
>       resources:
>         requests: { cpu: 8, memory: 16Gi }
>         limits:   { cpu: 8, memory: 16Gi }
>       livenessProbe:                    # K8s 定期检查容器是否存活
>         httpGet: { path: /health, port: 8000 }
>       readinessProbe:                   # K8s 判断容器是否可接收流量
>         httpGet: { path: /health, port: 8000 }
>       env:
>         - name: AWS_ENV
>           value: prod
>         - name: AWS_REGION
>           value: us-west-2
> ```
> 这些配置不是直接手写的——而是通过 Helm values 文件（如 `values/us-west-2/prod/main.yaml`）注入到模板中生成的。详见 [[10-helm-argocd-deployment|Helm 部署体系]]。

---

## 三、Deployment — 声明式管理 Pod

**Deployment 是管理 Pod 的控制器**——你告诉它"我要几个副本、用什么镜像"，它负责创建、维持、更新这些 Pod。

```
你声明：                              K8s 自动做：
─────────                            ─────────
replicas: 3                          创建 3 个 Pod
image: myapp:v2                      滚动更新（逐个替换旧 Pod）
Pod 挂了                             自动重建，维持 3 个副本
```

Deployment 和 Pod 之间还有一层 **ReplicaSet**——你不需要直接管它，但知道它存在有助于理解：

```
Deployment（你定义的）
  └── ReplicaSet（自动创建，管理副本数）
        ├── Pod-1
        ├── Pod-2
        └── Pod-3
```

一个典型的 Deployment YAML（以 plaud-project-summary 为例）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plaud-project-summary-deployer
  namespace: plaud-project-summary
spec:
  replicas: 2                          # 我要 2 个副本
  selector:
    matchLabels:                       # ← "配对暗号"：我管所有带这个标签的 Pod
      app_name: plaud-project-summary
  template:                            # Pod 模板（每个副本长这样）
    metadata:
      labels:                          # ← Pod 贴上的标签，必须和上面的暗号一致
        app_name: plaud-project-summary
    spec:
      containers:
        - name: deployer
          image: "236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:7e39d6c"
          ports:
            - containerPort: 8001      # API
            - containerPort: 8889      # 前端 UI
            - containerPort: 8000      # 健康检查
          resources:
            requests:                  # 最少需要多少资源
              cpu: 8
              memory: 16Gi
            limits:                    # 最多用多少资源
              cpu: 8
              memory: 16Gi
```

**selector 的本质就是"配对暗号"**——名字和值都可以自定义。`app_name: plaud-project-summary` 也行，`team: backend` 也行，只要 selector 和 Pod template 上贴的 Label **完全一致**就能配上。

> [!info] replicas、selector、template 的关系
> - **replicas**：期望的 Pod 数量
> - **selector**：告诉 Deployment "哪些 Pod 是我管的"（通过 Label 匹配）
> - **template**：创建 Pod 的模板（包含 Label、容器定义、资源配置等）
>
> `selector.matchLabels` 和 `template.metadata.labels` 必须一致，否则 Deployment 找不到自己创建的 Pod。

> [!example] 🔗 实战：plaud-project-summary 的 Deployment 配置
> 在项目中，你不直接写 Deployment YAML。而是在 values 文件中声明参数，Helm 渲染出完整的 Deployment：
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
> deployer:
>   replicaCount: 2                     # 初始 2 个副本
>   image:
>     repository: 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
>     tag: "7e39d6c"                    # Git commit hash，精确追溯代码版本
>
>   autoscaling:
>     enabled: true                     # 开启 HPA 自动扩缩容
>     minReplicas: 2
>     maxReplicas: 10
>     targetCPUUtilizationPercentage: 75
> ```
> 对应到 Deployment 概念：`replicaCount: 2` 就是 `spec.replicas`，`image.tag: "7e39d6c"` 就是 `spec.template.spec.containers[0].image` 的标签部分。
>
> 同时，generic-deployer 模板中预设了滚动更新策略：
> ```yaml
> # deploy/generic-deployer/templates/deployment.yaml
> strategy:
>   type: RollingUpdate
>   rollingUpdate:
>     maxUnavailable: 0       # 更新时不减少可用 Pod（先启新的，再关旧的）
>     maxSurge: "25%"         # 最多多创建 25% 的 Pod
> ```
> 这意味着发版时 K8s 会先启动新版 Pod，确认健康后再逐个关闭旧版——用户无感知。

---

## 四、Service — 稳定的网络入口

Pod 有 IP 地址，但 Pod 随时可能被重建（IP 会变）。**Service 提供一个稳定的访问入口**——不管后面的 Pod 怎么变，Service 的地址始终不变。

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  Service: plaud-project-summary-deployer                                     │
│  ClusterIP: 10.100.50.23（集群内虚拟 IP）                                     │
│  DNS: plaud-project-summary-deployer.plaud-project-summary.svc.cluster.local │
│                                                                              │
│         ┌────────┬────────┐                                                  │
│         ▼        ▼                                                           │
│      Pod-1    Pod-2          ← 都带有 app_name=plaud-project-summary 标签     │
│   172.16.0.5  172.16.0.8                                                     │
│                                                                              │
│  ← Service 通过 Label Selector 找到这些 Pod                                   │
│  ← 请求被自动负载均衡到其中一个 Pod                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Service 怎么知道该转发给哪些 Pod？** 答案还是 **Label Selector**——和 Deployment 一样的配对机制：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: plaud-project-summary-deployer
  namespace: plaud-project-summary
spec:
  selector:
    app_name: plaud-project-summary   # ← 同样的"暗号"！找到所有带这个 Label 的 Pod
  ports:
    - port: 8001              # Service 对外暴露的端口
      name: api
      targetPort: 8001        # 转发到 Pod 的端口
    - port: 8889
      name: frontend
      targetPort: 8889
  type: ClusterIP             # 默认类型，只在集群内部可访问
```

**Deployment 和 Service 通过同一个 Label 关联到同一批 Pod**——Deployment 用它管理 Pod 的生命周期，Service 用它把流量路由到这些 Pod：

```
Deployment                     Pod（贴着标签）                Service
selector:                      labels:                       selector:
  app_name: plaud-...  ──管理──→  app_name: plaud-...  ←──路由──  app_name: plaud-...
```

**三种常见的 Service 类型**：

| 类型 | 作用 | 谁能访问 |
|------|------|---------|
| **ClusterIP**（默认） | 集群内部的虚拟 IP | 集群内的 Pod |
| **NodePort** | 在每个节点上开一个端口 | 集群外（通过 节点IP:端口） |
| **LoadBalancer** | 创建云厂商的负载均衡器 | 公网（AWS ALB/NLB 等） |

> 详细的网络机制（CNI、CoreDNS、kube-proxy 如何实现转发）见 [[04-k8s-networking|K8s 网络深入]]。

> [!example] 🔗 实战：plaud-project-summary 的 Service — 一个服务暴露多个端口
> 项目中 Service 暴露了 3 个端口，分别对应 API、前端 UI、健康检查：
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
> deployer:
>   service:
>     type: ClusterIP
>     ports:
>       - port: 8001          # API 接口
>         name: api
>         targetPort: 8001
>       - port: 8889          # NiceGUI 前端
>         name: frontend
>         targetPort: 8889
>       - port: 8000          # 健康检查
>         name: health
>         targetPort: 8000
> ```
> 集群内其他服务可以通过 DNS 访问它：
> ```
> plaud-project-summary-us-prod-main-deployer.plaud-project-summary:8001
> └── Service 名称                              └── Namespace       └── 端口
> ```

---

## 五、Ingress — 集群外部的 HTTP 入口

Service 的 LoadBalancer 类型可以暴露服务，但**每个 Service 都创建一个 LB 既贵又不灵活**。Ingress 提供了一种统一的 HTTP 路由层：

```
外部请求
  │
  ▼
┌──────────────────────────────────┐
│  Ingress Controller（如 nginx）    │
│                                    │
│  规则 1: api.example.com → svc-a   │
│  规则 2: web.example.com → svc-b   │
│  规则 3: api.example.com/v2 → svc-c│
└──────┬──────────┬──────────┬──────┘
       ▼          ▼          ▼
    svc-a      svc-b      svc-c
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-svc     # 转发到哪个 Service
                port:
                  number: 80
```

> 多个域名、多个服务共用一个 Ingress Controller（一个 LB），通过 host 和 path 路由到不同的 Service。详见 [[04-k8s-networking|K8s 网络深入]]。

> [!example] 🔗 实战：plaud-project-summary 的三层 Ingress
> 项目中同一个服务配了 3 个 Ingress，对应不同的网络层级和访问权限：
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
> deployer:
>   ingresses:
>     internal:                                    # ① 内网 — 给内部 UI
>       ingressClassName: "nginx-internal"
>       hosts:
>         - host: plaud-project-summary-usw2.nicebuild.click
>           paths:
>             - path: /                             # 前端 UI → 8889
>               servicePort: 8889
>             - path: /api                          # 内部 API → 8001
>               servicePort: 8001
>
>     private:                                     # ② 私网 — 给内部微服务调用
>       ingressClassName: "nginx-pvt"
>       hosts:
>         - host: plaud-project-summary-usw2-lan.plaud.ai
>           paths:
>             - path: /api/temporal
>               servicePort: 8001
>
>     public:                                      # ③ 公网 — 给外部客户端
>       ingressClassName: "nginx-public"
>       hosts:
>         - host: plaud-project-summary-usw2.plaud.ai
>           paths:
>             - path: /api/strategy
>               servicePort: 8001
> ```
> 三个 Ingress 分别绑定不同的 Ingress Controller（`nginx-internal`、`nginx-pvt`、`nginx-public`），对应不同的 AWS 负载均衡器。这样同一个服务可以精细控制"谁能访问什么接口"。

---

## 六、Namespace — 逻辑隔离

Namespace 是 K8s 中的"文件夹"，用来将资源分组和隔离。

```
K8s 集群
├── namespace: default              ← 默认命名空间
├── namespace: kube-system          ← K8s 系统组件（CoreDNS、kube-proxy 等）
├── namespace: plaud-project-summary ← 你的项目
├── namespace: monitoring           ← Prometheus / Grafana
└── namespace: argocd               ← ArgoCD
```

**为什么需要 Namespace？**
- **资源隔离**：不同团队 / 项目的资源不会互相干扰
- **权限控制**：可以限制"用户 A 只能操作 namespace X"（见 [[08-k8s-security-rbac|安全与权限]]）
- **资源配额**：限制某个 namespace 最多用多少 CPU / 内存

同一个 Namespace 内资源可以用短名称互访（如 `my-app-svc`），跨 Namespace 需要完整 DNS（如 `my-app-svc.other-ns.svc.cluster.local`）。

---

## 七、ConfigMap 与 Secret — 配置管理

在 Docker 中，配置通常通过环境变量或挂载文件传入容器。K8s 将这个能力标准化为两种资源：

| 资源 | 用途 | 数据是否加密 |
|------|------|------------|
| **ConfigMap** | 非敏感配置（端口号、功能开关、配置文件） | 否（明文存储） |
| **Secret** | 敏感配置（数据库密码、API Key、证书） | Base64 编码（非加密，需配合加密方案） |

```yaml
# ConfigMap 示例
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  MAX_WORKERS: "4"

---
# 在 Pod 中使用 ConfigMap
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef:
            name: app-config     # 所有 key 自动变成环境变量
```

> 生产环境通常不直接用 K8s Secret，而是用 **External Secrets Operator** 从 AWS Secrets Manager 等外部系统同步密钥。详见 [[08-k8s-security-rbac|安全与权限]]。

> [!example] 🔗 实战：plaud-project-summary 的三种配置注入方式
> 配置注入也是通过 Helm 渲染的，但不一定创建单独的 ConfigMap/Secret 资源——有三种方式，走不同的路径：
>
> **① 环境变量 — 直接写入 Deployment，不创建 ConfigMap**
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
> deployer:
>   env:                                       # values 中的 env 字段
>     - name: AWS_ENV
>       value: prod
>     - name: AWS_REGION
>       value: us-west-2
>     - name: SECRETS_MANAGER_ENABLED
>       value: "true"
> ```
> Helm 渲染时，这些 env 直接写入 `deployment.yaml` 模板的 `spec.containers[].env` 中，不会创建单独的 ConfigMap 资源。Pod 启动后自动获得这些环境变量。
>
> **② ConfigMap — 作为文件挂载（plaud-project-summary 未使用此方式）**
> ```yaml
> # 如果在 values 中配置了 config 字段，Helm 会渲染 config.yaml 模板创建 ConfigMap：
> deployer:
>   config:
>     mountPath: /app/config              # 挂载到 Pod 内的路径
>     content: |                          # 文件内容
>       {"log_level": "info", "workers": 4}
> ```
> 这会创建一个 ConfigMap 资源，内容以 `config.json` 文件的形式挂载进 Pod。适合需要配置文件（而非环境变量）的场景。
>
> **③ ExternalSecret — 从 AWS Secrets Manager 自动同步密钥**
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
> deployer:
>   externalSecrets:
>     - secretStoreRef:
>         name: plaud-cluster-secret-store
>         kind: ClusterSecretStore
>       refreshInterval: 1h               # 每小时同步一次
>       target:
>         name: plaud-project-summary-secrets
> ```
> 注意：Helm 渲染的**不是 K8s Secret**，而是 **ExternalSecret**（一个 CRD）。然后由集群中运行的 External Secrets Operator 读取 ExternalSecret，从 AWS Secrets Manager 拉取密钥，**自动创建**真正的 K8s Secret。
>
> ```
> 你写 values          Helm 渲染                Operator 自动执行
>    │                    │                         │
>    ▼                    ▼                         ▼
> externalSecrets  →  ExternalSecret 资源  →  K8s Secret 资源
> (values 参数)       (CRD，描述"要同步什么")    (真正的密钥，Pod 可引用)
> ```
>
> 这样敏感信息不写在 Git 仓库中，由 Operator 自动从 AWS Secrets Manager 拉取。
>
> **plaud-project-summary 用的是 ① + ③**：非敏感配置走环境变量，敏感密钥走 ExternalSecret。

---

## 八、Label 与 Selector — K8s 的组织方式

Label 是附加在资源上的键值对标签，Selector 是通过标签筛选资源的查询条件。**这是 K8s 中各资源互相关联的核心机制。**

### 先区分三个容易混淆的概念

| | Docker image tag | K8s `metadata.name` | K8s `labels` |
|--|--|--|--|
| **例子** | `plaud-project-summary:7e39d6c` | `plaud-project-summary-deployer` | `app_name: plaud-project-summary` |
| **作用** | 镜像的版本号，决定"跑什么代码" | 资源的唯一名称，相当于身份证号 | 分类标签，用于批量筛选 |
| **数量** | 一个 | 一个（同 Namespace 内唯一） | **多个**（可以贴很多个） |
| **Docker 类比** | `docker pull nginx:1.25` 中的 `1.25` | `docker run --name xxx` 中的 `xxx` | Docker 没有对应概念 |

**为什么 Docker 没有 Label 这个概念？** 因为 Docker 是单机的——你只管几个容器，用 `--name` 就能逐个操作。但 K8s 管理成百上千个 Pod，不可能逐个操作，所以需要 Label 来**批量选中一群资源**：

```
Docker 世界：只能按名字操作一个容器
  docker stop plaud-project-summary       ← 只停一个

K8s 世界：Label 让你批量操作
  selector: app_name=plaud-project-summary   ← 同时选中所有贴了这个标签的 Pod

  Pod-1  labels: { app_name: plaud-project-summary }    ✓ 被选中
  Pod-2  labels: { app_name: plaud-project-summary }    ✓ 被选中
  Pod-3  labels: { app_name: other-service }            ✗ 没选中
```

### Label 如何关联 Deployment 和 Service

```
Pod（带 Label）                          Service（通过 Selector 查找 Pod）
┌──────────────────────┐               ┌──────────────────────────────┐
│ labels:              │ ◄──匹配─────  │ selector:                    │
│   app_name: plaud-.. │               │   app_name: plaud-..         │
│   env: prod          │               └──────────────────────────────┘
└──────────────────────┘
         ▲
         │ 匹配
┌────────┴─────────────────┐
│ Deployment               │
│ selector.matchLabels:    │
│   app_name: plaud-..     │
└──────────────────────────┘
```

**关键理解**：
- Deployment 通过 selector 知道"哪些 Pod 是我管的"
- Service 通过 selector 知道"流量应该转发给哪些 Pod"
- 同一个 Pod 可以同时被 Deployment 管理和被 Service 路由，因为它们各自通过 Label 匹配

常见的 Label 约定：

| Label | 含义 | 示例 |
|-------|------|------|
| `app` | 应用名称 | `app: plaud-project-summary` |
| `env` | 环境 | `env: prod` / `env: staging` |
| `version` | 版本 | `version: v2` |
| `component` | 组件角色 | `component: api` / `component: worker` |

> [!example] 🔗 实战：plaud-project-summary 中的 Label 和 Annotation
> 项目中 Pod 同时使用了 Label（用于筛选）和 Annotation（用于附加元信息）：
> ```yaml
> # deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
> deployer:
>   podLabels:
>     app_name: plaud-project-summary    # Label — Service selector 用它找到 Pod
>
> # 泳道（workspace）的 values 文件中还会用 Annotation 标记流量标签
> # deploy/plaud-project-summary/values/us-west-2/prod/workspace.yaml
>   podAnnotations:
>     metrics.labels/x-pld-lane: "workspace"   # Annotation — Prometheus 采集时提取
> ```
> Label `app_name` 让 Service 知道"这些 Pod 属于我"；Annotation `metrics.labels/x-pld-lane` 让 Prometheus 在采集指标时带上泳道标记，方便按 lane 维度监控——但它**不参与**资源的筛选和路由。

---

## 九、YAML 基本结构

所有 K8s 资源都用 YAML 定义，遵循统一的四段结构：

```yaml
apiVersion: apps/v1          # ① API 版本 — 这个资源属于哪个 API 组
kind: Deployment             # ② 资源类型 — Deployment / Service / Pod / ...
metadata:                    # ③ 元信息 — 名称、命名空间、标签
  name: my-app
  namespace: default
  labels:
    app: my-app
spec:                        # ④ 规格 — 你期望的状态（不同资源类型的 spec 不同）
  replicas: 3
  ...
```

| 字段 | 作用 | 备注 |
|------|------|------|
| `apiVersion` | API 版本 | `v1`（核心资源）、`apps/v1`（Deployment）、`networking.k8s.io/v1`（Ingress） |
| `kind` | 资源类型 | Pod、Deployment、Service、ConfigMap、Secret、Ingress... |
| `metadata` | 元信息 | `name`（必填）、`namespace`、`labels`、`annotations` |
| `spec` | 期望状态 | 每种资源不同，这是 YAML 的核心部分 |

> **`metadata.annotations` vs `metadata.labels`**：Labels 用于筛选和组织（Selector 能查），Annotations 用于存附加信息（如构建时间、Git commit、配置参数），不参与筛选。

---

## 十、kubectl 基本操作

kubectl 是 K8s 的命令行工具，相当于 Docker 中的 `docker` 命令。

### 查看资源

```bash
# 查看 Pod（指定命名空间）
kubectl get pods -n plaud-project-summary

# 查看 Deployment
kubectl get deployments -n plaud-project-summary

# 查看 Service
kubectl get services -n plaud-project-summary

# 查看所有类型的资源
kubectl get all -n plaud-project-summary

# 查看详细信息（包含事件、状态等，排障必备）
kubectl describe pod <pod-name> -n plaud-project-summary
```

### 创建 / 更新资源

```bash
# 声明式（推荐）— 让 K8s 自动判断是创建还是更新
kubectl apply -f deployment.yaml

# 命令式（临时用）— 直接创建
kubectl create deployment my-app --image=nginx
```

### 调试

```bash
# 查看 Pod 日志
kubectl logs <pod-name> -n plaud-project-summary

# 实时跟踪日志（类似 tail -f）
kubectl logs -f <pod-name> -n plaud-project-summary

# 进入容器内部（类似 docker exec）
kubectl exec -it <pod-name> -n plaud-project-summary -- /bin/bash

# 端口转发（将 Pod 的端口映射到本地，方便调试）
kubectl port-forward <pod-name> 8080:8000 -n plaud-project-summary
# 然后浏览器访问 http://localhost:8080
```

### 常用缩写

| 全称 | 缩写 | 示例 |
|------|------|------|
| pods | po | `kubectl get po` |
| deployments | deploy | `kubectl get deploy` |
| services | svc | `kubectl get svc` |
| namespaces | ns | `kubectl get ns` |
| configmaps | cm | `kubectl get cm` |
| nodes | no | `kubectl get no` |

---

## 十一、资源关系全景

这张图展示了本文介绍的核心概念如何关联——后续所有章节都在此基础上展开：

```
                        外部流量
                           │
                    ┌──────┴──────┐
                    │   Ingress   │ ← 按域名/路径路由
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │   Service   │ ← 稳定的内部访问入口 + 负载均衡
                    │ (Label 匹配) │    
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
         Pod-1          Pod-2          Pod-3        ← 实际运行的容器
            ▲              ▲              ▲
            └──────────────┼──────────────┘
                           │
                    ┌──────┴──────┐
                    │  ReplicaSet │ ← 维持副本数
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  Deployment │ ← 你定义的"期望状态"
                    └─────────────┘

    Pod 可以引用：
    ├── ConfigMap（非敏感配置）
    ├── Secret（敏感配置）
    └── PersistentVolumeClaim（持久存储，详见存储章节）
    
    所有资源都存在于某个 Namespace 中
```

### 你实际写的是什么？—— Helm 渲染全景

上面的 Deployment、Service、Ingress、HPA……你不需要逐个手写。在项目中，**所有 K8s 资源都由 Helm 从一个 values 文件渲染出来**：

```
你只写这个 ──────────────────────────────────────────────────────────────────┐
                                                                             │
  values/us-west-2/prod/main.yaml                                            │
  ┌────────────────────────────────────────────┐                             │
  │ deployer:                                  │                             │
  │   replicaCount: 2                          │─→ Deployment 的副本数        │
  │   image:                                   │                             │
  │     repository: ...ecr.../plaud-project-summary │                        │
  │     tag: "7e39d6c"                         │─→ Deployment 的镜像版本      │
  │   resources: { cpu: 8, memory: 16Gi }      │─→ Deployment 的资源配置      │
  │   service:                                 │                             │
  │     ports: [8001, 8889, 8000]              │─→ Service 的端口映射         │
  │   ingresses:                               │                             │
  │     internal: { host: ...nicebuild.click } │─→ Ingress（内网）            │
  │     private:  { host: ...-lan.plaud.ai }   │─→ Ingress（私网）            │
  │     public:   { host: ...plaud.ai }        │─→ Ingress（公网）            │
  │   autoscaling:                             │                             │
  │     enabled: true, maxReplicas: 10         │─→ HPA（自动扩缩容）           │
  │   serviceAccount:                          │                             │
  │     annotations: eks.../role-arn: ...      │─→ ServiceAccount（AWS 权限）  │
  │   env: [AWS_ENV=prod, ...]                 │─→ Pod 的环境变量             │
  │   externalSecret: { enabled: true, ... }   │─→ ExternalSecret（密钥同步）  │
  └────────────────────────────────────────────┘                             │
                    │                                                        │
                    ▼  ArgoCD 执行 helm template                              │
                    │                                                        │
  generic-deployer/templates/ ───── 通用模板（所有微服务共用） ──────────────────┘
  ┌─────────────────────────────────────┐
  │ deployment.yaml   →  Deployment     │
  │ service.yaml      →  Service        │
  │ ingresses.yaml    →  Ingress × 3    │
  │ hpa.yaml          →  HPA            │  ← 模板 + values = 完整的 K8s YAML
  │ serviceaccount.yaml → ServiceAccount│
  │ externalsecret.yaml → ExternalSecret│
  │ servicemonitor.yaml → ServiceMonitor│
  └─────────────────────────────────────┘
                    │
                    ▼  ArgoCD 提交给 API Server
                    │
              K8s 集群开始运行这些资源
```

**核心理解**：
- **你写的**：一个 values 文件（声明参数）
- **Helm 做的**：把参数注入模板，渲染出完整的 K8s YAML
- **ArgoCD 做的**：监听 Git 变更，自动执行 `helm template` 并提交给 K8s
- **K8s 做的**：接收 YAML，通过各控制器让集群达到期望状态

所以本文介绍的每个概念（Deployment、Service、Ingress、ConfigMap……）**都对应 `generic-deployer/templates/` 中的一个模板文件**，而你只需要在 values 文件里填参数。详见 [[10-helm-argocd-deployment|Helm 与 ArgoCD 部署体系]]。

### 关键链路

**以 plaud-project-summary 为例**：
1. **部署链路**：你修改 values 文件的 `image.tag` → ArgoCD 渲染 Helm 模板 → 生成 Deployment YAML → K8s 创建 ReplicaSet → ReplicaSet 创建 Pod
2. **访问链路**：外部请求 `plaud-project-summary-usw2.plaud.ai/api/strategy` → Ingress（nginx-public）→ Service（:8001）→ Pod
3. **配置链路**：Pod 从环境变量（values 中的 `env`）和 ExternalSecret（AWS Secrets Manager）获取配置

> 这些资源是怎么被 K8s 内部组件协调运转的？（API Server、Scheduler、Controller 各自扮演什么角色？）详见 [[05-k8s-architecture|K8s 架构原理]]。

---

## 十二、核心概念速查表

| 概念 | 一句话解释 | 类比 |
|------|----------|------|
| **Pod** | 一个或多个容器的组合，最小部署单元 | 一个进程组 |
| **Deployment** | 管理 Pod 副本数和更新策略 | 进程管理器（如 supervisord） |
| **ReplicaSet** | 维持指定数量的 Pod 副本（Deployment 自动管理） | Deployment 的"执行者" |
| **Service** | 为一组 Pod 提供稳定的网络入口 | 负载均衡器 + DNS |
| **Ingress** | HTTP 层的路由规则（域名 → Service） | Nginx 反向代理 |
| **Namespace** | 资源的逻辑分组和隔离 | 文件夹 |
| **ConfigMap** | 非敏感的键值配置 | 环境变量文件 |
| **Secret** | 敏感的键值配置 | 加密的环境变量文件 |
| **Label** | 附加在资源上的键值标签 | 便利贴标签 |
| **Selector** | 通过 Label 筛选资源的查询条件 | 标签过滤器 |
| **Node** | 集群中的一台机器（物理机或虚拟机） | 服务器 |

---

## 延伸阅读

- [[03-k8s-workload-types|K8s 工作负载类型]] — Deployment 之外的 StatefulSet、DaemonSet、Job
- [[04-k8s-networking|K8s 网络深入]] — Service、CNI、CoreDNS、Ingress 的实现原理
- [[05-k8s-architecture|K8s 架构原理]] — 控制平面、数据平面、声明式 API 与控制器模式
- [[06-k8s-storage|K8s 存储]] — PV/PVC、StorageClass、CSI
- [[07-k8s-scheduling-resources|调度与资源管理]] — resources 的 requests/limits、QoS、HPA 自动扩缩容
- [[08-k8s-security-rbac|K8s 安全与权限]] — RBAC、Pod Security、Secret 管理
