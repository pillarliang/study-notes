# K8s 核心概念入门

---

## 一、要解决什么问题

Docker 镜像已经打好。在一台机器上 `docker run` 可以跑起来，但生产环境要解决的问题远不止"跑起来"：

| 问题 | Docker 手动管 | K8s 自动化 |
|------|-------------|-----------|
| 一个容器挂了 | 人工发现、手动重启 | 自动检测、自动重建 |
| 要跑多个副本 | 手动 ssh 到各机器启动 | 声明 `replicas: 2`，K8s 自己搞定 |
| 其他服务怎么调用它 | 硬编码 IP（Pod 重建就断了） | Service 提供稳定 DNS 名称 |
| 外部用户通过域名访问 | 手动配 Nginx 反代 | Ingress 按域名/路径路由 |
| 不同环境用不同配置 | 写死在镜像或手动注入 | ConfigMap / Secret 集中管理 |
| 更新版本不停机 | 手动逐个替换，容易翻车 | 滚动更新，先启新的再关旧的 |

手动管几台机器勉强行得通，但公司有 80+ 微服务、5 个区域、3 种环境——不可能手动管。

**Kubernetes（K8s）就是解决这些问题的系统。** 它引入了一组"资源对象"，每种对象回答上面一个问题。接下来以 plaud-project-summary 为例，逐个认识它们。

> [!tip] 声明式思维
> K8s 最大的思维转变：不是告诉它"创建一个容器""删除那个容器"（命令式），而是声明"需要 2 个副本跑这个镜像"（声明式）。K8s 自己算出该做什么，并且**持续**保证现实与声明一致——Pod 挂了自动补、节点宕机自动迁移。这个"不断对比期望与现实并自动修正"的机制叫 Reconcile Loop，详见 [[05-k8s-architecture|K8s 架构原理]]。

---

## 二、Pod — 容器的运行单元

**问题**：已有 Docker 镜像 `plaud-project-summary:7e39d6c`，K8s 怎么跑它？

K8s 不直接管理容器，而是管理 **Pod**。Pod 是一个或多个容器的组合，共享网络和存储：

```
┌────────── Pod ─────────────┐
│  ┌──────────┐ ┌──────────┐ │  同一个 Pod 内的容器共享：
│  │ 容器 A    │ │ 容器 B   │ │  - 同一个 IP 地址
│  │ (主应用)  │ │ (sidecar)│ │  - 同一个 localhost
│  └──────────┘ └──────────┘ │  - 可以共享 Volume
└────────────────────────────┘
```

实际使用中，90% 的 Pod 只有一个容器。一个最小的 Pod 长这样：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: plaud-project-summary
  labels:
    app_name: plaud-project-summary    # ← 标签，下一节会解释
spec:
  containers:
    - name: app
      image: 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary:7e39d6c # 私有镜像必须写完整路径（ECR 域名/仓库名:tag），否则 kubelet 默认去 Docker Hub 找，报 ImagePullBackOff # 实际项目中这个地址由 Helm 从 values.image.repository + values.image.tag 拼接生成
      ports:
        - containerPort: 8001
```

但实际上**不会直接创建 Pod**——Pod 是"易碎品"，挂了没人重建。需要一个"管理者"来维持副本数，这就是 Deployment。不过在讲 Deployment 之前，先认识一个贯穿 K8s 的核心机制。

---

## 三、Label、metadata.name 与 Selector — Helm 如何给资源"取名"和"配对"

**问题**：集群里可能跑着上百个 Pod，Deployment 怎么知道"哪些 Pod 归自己管"？Service 怎么知道"流量该转发给谁"？

靠名字不行——Pod 名字是随机的（如 `plaud-project-summary-deployer-7d8f9b6c4-xk2pq`）。K8s 用的是 **Label**——贴在资源上的键值对标签，配合 **Selector** 来批量选中一群资源：

```
Pod-1  labels: { app.kubernetes.io/instance: plaud-project-summary }    ✓ 被选中
Pod-2  labels: { app.kubernetes.io/instance: plaud-project-summary }    ✓ 被选中
Pod-3  labels: { app.kubernetes.io/instance: other-service }            ✗ 没选中

Selector: app.kubernetes.io/instance = plaud-project-summary → 选中 Pod-1 和 Pod-2
```

在项目中，这些 Label 和资源名称都**不需要手动写**——由 Helm 模板自动生成。

### 3.1 所有名称和标签的共同来源

Helm 生成 `metadata.name`（资源名称）和 `labels`（配对标签）时，用的是**同一组原始值**，来自两个文件：

```
deploy/plaud-project-summary/application.yaml        deploy/plaud-project-summary/Chart.yaml
┌─────────────────────────────────┐                  ┌──────────────────────────────────┐
│ metadata:                       │                  │ dependencies:                    │
│   name: plaud-project-summary ──┼── .Release.Name  │   - name: deployer ──────────────┼── .Chart.Name
└─────────────────────────────────┘                  └──────────────────────────────────┘
        │                                                       │
        └──────────────────┬────────────────────────────────────┘
                           ↓
              deploy/generic-deployer/templates/_helpers.tpl
              （公共函数库，所有模板共用）
                           │
            ┌──────────────┼──────────────────────┐
            ↓              ↓                      ↓
        fullname       selectorLabels           labels
      （资源名称）     （配对标签）           （完整标签）
```

这两个值在项目创建时写好后基本不会变。`_helpers.tpl` 中定义了三个函数，各模板通过 `{{ include ... }}` 引用，保证所有资源的名称和标签自动一致。

### 3.2 metadata.name — 资源的唯一标识

`metadata.name` 由 `_helpers.tpl` 中的 `fullname` 函数生成，拼接规则是 `.Release.Name` + `-` + `.Chart.Name`：

```
"plaud-project-summary" + "-" + "deployer" = "plaud-project-summary-deployer"
```

各模板都引用同一个函数，因此 Deployment、Service 等资源的 `metadata.name` 都是 `plaud-project-summary-deployer`：

```yaml
# deployment.yaml / service.yaml 中都是：
metadata:
  name: {{ include "generic-deployer.fullname" . }}   → plaud-project-summary-deployer
```

```
kubectl get deploy,svc -n plaud-project-summary
→ deployment/plaud-project-summary-deployer     ← Deployment 的 metadata.name
→ service/plaud-project-summary-deployer        ← Service 的 metadata.name

kubectl get pods -n plaud-project-summary
→ plaud-project-summary-deployer-7d8f9b6c4-xk2pq   ← Pod 名 = Deployment 名 + 随机后缀
```

`metadata.name` 是 K8s 资源的"身份证号"，同一 Namespace 内不能重名。用于 kubectl 操作（`kubectl describe deploy plaud-project-summary-deployer`）、Service DNS 生成（`plaud-project-summary-deployer.plaud-project-summary.svc.cluster.local`）等场景。

> 如果需要覆盖自动生成的名称，可以在 values 中设置 `fullnameOverride`（当前为空，不覆盖）。

### 3.3 selectorLabels — Deployment/Service 找 Pod 的配对标签

`selectorLabels` 函数把同样的两个值分别放入两个标签：

```yaml
# 来源：deploy/generic-deployer/templates/_helpers.tpl（渲染后的效果）

# selectorLabels — Deployment selector、Service selector、Pod labels 都包含这组标签
app.kubernetes.io/name: deployer                  # ← .Chart.Name
app.kubernetes.io/instance: plaud-project-summary # ← .Release.Name
```

Deployment、Service、Pod 模板都通过 `{{ include "generic-deployer.selectorLabels" . }}` 引用同一个函数，保证三者的配对标签一定一致——不需要手动维护。

### 3.4 metadata.name vs labels 的区别

两者由同一组输入生成，但用途完全不同：

| | `metadata.name` | `labels` |
|---|---|---|
| 作用 | 资源的唯一标识（"身份证号"） | 批量筛选（"分组标签"） |
| 生成函数 | `fullname`（拼接成一个字符串） | `selectorLabels`（拆成两个键值对） |
| plaud-project-summary 的值 | `plaud-project-summary-deployer` | `app.kubernetes.io/name: deployer` + `app.kubernetes.io/instance: plaud-project-summary` |
| 典型用法 | `kubectl describe deploy plaud-project-summary-deployer` | Deployment/Service 的 selector 配对 |

### 3.5 两类标签的区别

Pod 上实际会有两组标签，用途不同：

| 来源 | 标签示例 | 用途 |
|------|---------|------|
| `_helpers.tpl` 自动生成 | `app.kubernetes.io/name: deployer` | **Deployment 和 Service 的 selector 配对**（核心） |
| values 中 `podLabels` 手动声明 | `app_name: plaud-project-summary` | 监控、日志筛选等业务用途（不参与配对） |

### 3.6 容易混淆的概念

| 概念 | 例子 | 作用 |
|------|------|------|
| Docker image tag | `plaud-project-summary:7e39d6c` | 镜像版本号，"跑什么代码" |
| `metadata.name` | `plaud-project-summary-deployer` | 资源的唯一名称，"身份证号" |
| `labels`（自动） | `app.kubernetes.io/instance: plaud-project-summary` | 分组标签，"批量筛选" |
| `podLabels`（手动） | `app_name: plaud-project-summary` | 业务标签，用于监控 |

> **Label vs Annotation**：Label 用于筛选（Selector 能查到），Annotation 用于附加元信息（如 `metrics.labels/x-pld-lane: "workspace"` 让 Prometheus 按泳道维度监控），不参与筛选。

---

## 四、Deployment — "需要 2 个副本，永远保持"

**问题**：Pod 挂了没人重建，手动管副本不现实。怎么告诉 K8s "需要 2 个 plaud-project-summary，挂了自动补"？

Deployment 是"管理 Pod 的控制器"——声明期望状态，由它负责达到并维持：

```yaml
# 来源：deploy/plaud-project-summary/values/us-west-2/staging/main.yaml（简化展示）
deployer:
  replicaCount: 2          # "需要 2 个副本"
  image:
    tag: "7e39d6c"         # "跑这个版本的代码"
```

Deployment 的行为：

| 声明的内容 | K8s 自动做的 |
|---------|-------------|
| `replicas: 2` | 创建 2 个 Pod，始终维持 |
| `image.tag: "7e39d6c"` → 改成 `"a1b2c3d"` | 滚动更新：先启新版 Pod，健康后再关旧版 |
| 一个 Pod 挂了 | 自动重建，补回 2 个 |

**Deployment 怎么知道哪些 Pod 是它管的？** 就是上一节的 Label：

```
Deployment
  selector.matchLabels:                              ← Deployment 用这组标签筛选 Pod
    app.kubernetes.io/name: deployer
    app.kubernetes.io/instance: plaud-project-summary

Pod template
  labels:                                            ← Pod 创建时自动贴上同一组标签
    app.kubernetes.io/name: deployer
    app.kubernetes.io/instance: plaud-project-summary
```

selector 和 template 里的标签**必须一致**，否则 Deployment 找不到自己创建的 Pod。这组标签由 `_helpers.tpl` 中的 `selectorLabels` 统一生成，不需要手动维护一致性。

Deployment 和 Pod 之间还有一层 **ReplicaSet**——不需要直接管它，但知道它存在有助于理解：

```
Deployment（开发者定义的）
  └── ReplicaSet（Deployment 自动创建）
        ├── Pod-1
        └── Pod-2
```

改镜像版本时，Deployment 创建新的 ReplicaSet（新版本），逐渐缩小旧 ReplicaSet——这就是滚动更新。plaud-project-summary 的更新策略是 `maxUnavailable: 0`（更新过程中不减少可用 Pod），确保用户无感知。

---

## 五、Service — "一个稳定的地址，不管后面是哪个 Pod"

**问题**：plaud-project-summary 有 2 个 Pod（IP 分别是 172.16.0.5 和 172.16.0.8）。其他服务怎么调用它？直接用 Pod IP？但 Pod 重建后 IP 会变，硬编码就断了。

Service 提供一个**稳定的 DNS 名称 + 虚拟 IP**，不管后面的 Pod 怎么变，地址永远不变：

```
┌──────────────────────────────────────────────────────────────┐
│  Service: plaud-project-summary-deployer                     │
│  DNS: plaud-project-summary-deployer.plaud-project-summary   │
│                                                              │
│         ┌─────────┬─────────┐                                │
│         ▼                   ▼                                │
│      Pod-1               Pod-2       ← 请求自动负载均衡        │
│   172.16.0.5          172.16.0.8                             │
│                                                              │
│  Service 通过 Label Selector 找到这些 Pod                      │
└──────────────────────────────────────────────────────────────┘
```

**Service 怎么找到 Pod？** 和 Deployment 一样——`_helpers.tpl` 生成的同一组 `selectorLabels`：

```
Deployment                          Pod（贴着标签）                         Service
selector:                           labels:                                selector:
  app.kubernetes.io/name:             app.kubernetes.io/name:                app.kubernetes.io/name:
    deployer            ──管理──→       deployer              ←──路由──        deployer
  app.kubernetes.io/instance:         app.kubernetes.io/instance:            app.kubernetes.io/instance:
    plaud-project-summary               plaud-project-summary                  plaud-project-summary
```

plaud-project-summary 的 Service 暴露了 3 个端口，分别服务 API、前端 UI 和健康检查：

```yaml
# 来源：deploy/plaud-project-summary/values/us-west-2/staging/main.yaml（简化展示）
deployer:
  service:
    type: ClusterIP               # 只在集群内部可访问
    ports:
      - port: 8001, name: api
      - port: 8889, name: frontend
      - port: 8000, name: health
```

**Service 类型**决定"谁能访问"：

| 类型 | 谁能访问 |
|------|---------|
| **ClusterIP**（默认） | 集群内的 Pod（plaud-project-summary 用这个） |
| **NodePort** | 集群外，通过 节点IP:端口 |
| **LoadBalancer** | 公网，云厂商创建负载均衡器 |

但 ClusterIP 只能集群内访问。外部用户怎么进来？

---

## 六、Ingress — "外部请求按域名和路径路由进来"

**问题**：plaud-project-summary 需要同时对外暴露 API（`/api/strategy`）和对内暴露前端 UI（`/`），还要区分"公网"和"内网"。给每个 Service 都创建一个云 LB 既贵又不灵活。

Ingress 是一个 **HTTP 路由层**——多个服务共享一个 LB，通过域名和路径区分：

```
外部请求
  │
  ▼
Ingress Controller（nginx）
  │
  ├── plaud-project-summary-usw2.plaud.ai/api/strategy      → Service:8001 (公网)
  ├── plaud-project-summary-usw2.nicebuild.click/            → Service:8889 (内网 UI)
  └── plaud-project-summary-usw2-lan.plaud.ai/api/temporal   → Service:8001 (私网)
```

plaud-project-summary 配了 3 个 Ingress，分别绑定 `nginx-public`、`nginx-internal`、`nginx-pvt` 三种 Ingress Controller，对应不同的 AWS 负载均衡器。**同一个服务可以精细控制"谁能访问什么接口"。**

> 完整的 Ingress 配置和网络实现原理见 [[04-k8s-networking|K8s 网络深入]]。

---

## 七、Namespace — "各服务各管各的"

到这里，plaud-project-summary 的 Deployment、Service、Ingress 都有了。但集群里还有 80+ 其他微服务，它们的资源不能混在一起。

**Namespace** 是 K8s 的"文件夹"，用来分组和隔离：

```
K8s 集群
├── namespace: plaud-project-summary    ← 摘要服务
├── namespace: plaud-transcribe         ← 转录服务
├── namespace: monitoring               ← Prometheus / Grafana
├── namespace: argocd                   ← ArgoCD
└── namespace: kube-system              ← K8s 系统组件
```

同 Namespace 内可用短名称互访（`my-svc:8001`），跨 Namespace 需完整 DNS（`my-svc.other-ns.svc.cluster.local`）。

---

## 八、ConfigMap 与 Secret — "配置不硬编码在镜像里"

**问题**：plaud-project-summary 需要知道自己跑在哪个区域（`AWS_REGION=us-west-2`）、连接哪个密钥。不同区域/环境的值不同，不能写死在镜像里。

K8s 提供两种配置资源：

| 资源 | 用途 | 例子 |
|------|------|------|
| **ConfigMap** | 非敏感配置 | 日志级别、功能开关、配置文件 |
| **Secret** | 敏感配置 | 数据库密码、API Key |

plaud-project-summary 用了两种方式注入配置：

**① 环境变量** — 非敏感配置直接写在 values 里，Helm 渲染进 Deployment：

```yaml
# 来源：deploy/plaud-project-summary/values/us-west-2/staging/main.yaml（简化展示）
deployer:
  env:
    - name: AWS_REGION
      value: us-west-2
    - name: AWS_ENV
      value: prod
```

**② ExternalSecret** — 敏感密钥不写在 Git 仓库，由 External Secrets Operator 从 AWS Secrets Manager 自动同步：

```
values 声明 → Helm 渲染出 ExternalSecret → External Secrets Operator 从 AWS 拉取 → 自动创建 K8s Secret → Pod 引用
```

以 sigma-worker 为例，values 中声明需要哪个 AWS 密钥：

```yaml
# 来源：deploy/sigma-agent/worker-service/values/global/staging/values.yaml（简化展示）
externalSecret:
  enabled: true
  name: sigma-worker
  refreshInterval: "5m"             # 每 5 分钟从 AWS 同步一次
  secretStoreRef:
    name: "aws-secrets-manager-store"   # 引用集群里预配置的 ClusterSecretStore
    kind: "ClusterSecretStore"
  awsSecretName: "sigma-worker"     # AWS Secrets Manager 中的密钥名称
  target:
    name: sigma-worker              # 生成的 K8s Secret 名称
    creationPolicy: Owner
```

Helm 渲染后生成 ExternalSecret 资源，External Secrets Operator 据此执行同步：

```yaml
# 来源：deploy/sigma-agent/worker-service/templates/externalsecret.yaml（Helm 渲染后的效果）
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: sigma-worker
spec:
  refreshInterval: "5m"
  secretStoreRef:
    name: "aws-secrets-manager-store"   # 集群级别的密钥存储连接配置
    kind: "ClusterSecretStore"          # 所有 Namespace 共享
  target:
    name: sigma-worker                  # Operator 会自动创建同名的 K8s Secret
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: "sigma-worker"             # 从 AWS Secrets Manager 的这个 key 提取所有键值对
```

整个流程：

```
① values 声明 externalSecret.awsSecretName: "sigma-worker"
                ↓ Helm 渲染
② K8s 集群中出现 ExternalSecret 资源
                ↓ External Secrets Operator 监听到
③ Operator 通过 ClusterSecretStore 配置的 IAM 角色访问 AWS
                ↓
④ 从 AWS Secrets Manager 拉取 "sigma-worker" 的内容
                ↓
⑤ 自动创建 K8s Secret（name: sigma-worker），包含所有键值对
                ↓ 每 5 分钟自动同步更新
⑥ Pod 通过 envFrom 或 secretKeyRef 引用这个 Secret
```

> **Operator 是 K8s 的一种扩展模式**：普通控制器管内置资源（如 Deployment Controller 管 Deployment），Operator 管自定义资源（CRD）。External Secrets Operator 管的就是 `ExternalSecret` 这个 CRD——本质上和"监听 Deployment → 创建 Pod"是同一个 Reconcile Loop 模式，只是管理对象不同。

详见 [[08-k8s-security-rbac|安全与权限]]。

---

## 九、串起来：一个 values 文件如何变成运行中的服务

上面的概念不是孤立的——在项目中，**只需写一个 values 文件**，Helm + ArgoCD + K8s 自动生成并运行所有资源：

```
values/us-west-2/prod/main.yaml
┌──────────────────────────────────────┐
│ deployer:                            │
│   replicaCount: 2             → Deployment（2 个副本）
│   image.tag: "7e39d6c"        → Deployment（跑什么版本）
│   resources: cpu:8, mem:16Gi  → Pod（资源配额）
│   service.ports: [8001,8889]  → Service（端口映射）
│   ingresses: [internal,       → Ingress × 3（HTTP 路由）
│     private, public]          │
│   autoscaling: max:10         → HPA（自动扩缩容）
│   serviceAccount: role-arn    → ServiceAccount（AWS 权限）
│   env: [AWS_ENV=prod, ...]    → Pod 环境变量
└──────────────────────────────────────┘
        │
        ▼  git push
     ArgoCD 检测变更 → 渲染 Helm 模板 → 提交给 K8s
        │
        ▼
     K8s 各控制器协同运行这些资源
```

每个概念都对应 `generic-deployer/templates/` 中的一个模板文件（deployment.yaml、service.yaml、ingresses.yaml…），80+ 微服务共用同一套模板。详见 [[10-helm-argocd-deployment|Helm 与 ArgoCD 部署体系]]。

**三条关键链路**：

1. **部署**：`git push → ArgoCD → Helm → API Server → Deployment → ReplicaSet → Pod`
2. **访问**：`用户 → plaud-project-summary-usw2.plaud.ai → Ingress → Service → Pod`
3. **配置**：`values 中的 env → Pod 环境变量` + `ExternalSecret → AWS Secrets Manager → K8s Secret → Pod`

> 这些资源是怎么被 K8s 内部组件协调运转的？详见 [[05-k8s-architecture|K8s 架构原理]]。

---

## 十、kubectl 常用操作

```bash
# 查看 Pod 状态
kubectl get pods -n plaud-project-summary

# 详细信息（排障必备）
kubectl describe pod <pod-name> -n plaud-project-summary

# 实时日志
kubectl logs -f <pod-name> -n plaud-project-summary

# 进入容器
kubectl exec -it <pod-name> -n plaud-project-summary -- /bin/bash

# 端口转发到本地调试
kubectl port-forward <pod-name> 8080:8000 -n plaud-project-summary
```

| 全称 | 缩写 | 全称 | 缩写 |
|------|------|------|------|
| pods | po | services | svc |
| deployments | deploy | namespaces | ns |
| configmaps | cm | nodes | no |

---

## 速查表

| 概念 | 一句话 | plaud-project-summary 中的体现 |
|------|--------|-------------------------------|
| **Pod** | 容器的运行单元 | 每个 Pod 跑一个 plaud-project-summary 容器 |
| **Label** | 贴在资源上的标签，用于分组筛选 | `app.kubernetes.io/instance: plaud-project-summary`（Helm 自动生成，Deployment/Service 配对用） |
| **Deployment** | 管理 Pod 副本数和更新策略 | 2 个副本，`maxUnavailable: 0` 滚动更新 |
| **Service** | 稳定的内部访问入口 + 负载均衡 | ClusterIP，暴露 8001 / 8889 / 8000 |
| **Ingress** | 外部 HTTP 路由（域名 → Service） | 3 层：internal / private / public |
| **Namespace** | 资源的逻辑隔离 | `plaud-project-summary` |
| **ConfigMap / Secret** | 配置注入 | env 变量 + ExternalSecret |

---

## 延伸阅读

- [[03-k8s-workload-types|工作负载类型]] — Deployment 之外的 StatefulSet、DaemonSet、Job
- [[04-k8s-networking|网络深入]] — Service、CNI、CoreDNS、Ingress 的实现原理
- [[05-k8s-architecture|架构原理]] — 控制平面、数据平面、声明式 API 与 Reconcile Loop
- [[06-k8s-storage|存储]] — PV/PVC、StorageClass
- [[07-k8s-scheduling-resources|调度与资源管理]] — requests/limits、QoS、HPA
- [[08-k8s-security-rbac|安全与权限]] — RBAC、IRSA、ExternalSecret
