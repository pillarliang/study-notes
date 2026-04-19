# K8s 扩展机制：CRD、Operator 与 Admission Webhook

> 前置知识：了解 [[05-k8s-architecture#五、声明式 API 与控制器模式|声明式 API 与控制器模式]]。本文介绍如何扩展 K8s 的能力——你日常使用的 ArgoCD、Prometheus Operator、External Secrets Operator 都是通过这些机制实现的。
>
> **学习建议**：首次阅读重点关注第一节和第五节（基础：为什么需要扩展 K8s、Helm vs Kustomize）。第二至三节（CRD、Operator）为进阶，第四节和第六节为深入。

---

## 一、为什么需要扩展 K8s

K8s 内置的资源类型（Deployment、Service、ConfigMap 等）覆盖了通用场景。但实际生产中有很多领域特定的需求：

| 需求 | K8s 内置能做吗 | 扩展方案 |
|------|------|------|
| 声明式 GitOps 部署 | 不能（需要 watch Git 仓库） | ArgoCD（CRD: Application） |
| 自动同步外部密钥 | 不能（Secret 是静态的） | External Secrets Operator（CRD: ExternalSecret） |
| 监控告警规则管理 | 不能 | Prometheus Operator（CRD: PrometheusRule） |
| 数据库集群运维 | StatefulSet 太底层 | CloudNativePG（CRD: Cluster） |

**K8s 的扩展机制让你可以"教会 K8s 新的概念"**——定义新的资源类型，编写对应的控制器来管理它们。

---

## 二、CRD（Custom Resource Definition）（进阶）

### 2.1 CRD 是什么

CRD 让你在 K8s 中定义全新的资源类型。定义之后，就可以像使用 Deployment、Service 一样使用 `kubectl` 操作它。

```yaml
# 定义一个新资源类型：Application（ArgoCD 的核心 CRD）
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.argoproj.io
spec:
  group: argoproj.io                # API 组
  names:
    kind: Application               # 资源类型名
    plural: applications             # 复数形式（kubectl get applications）
    shortNames: ["app", "apps"]      # 简称（kubectl get apps）
  scope: Namespaced                  # namespace 级别
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                source:
                  type: object
                  properties:
                    repoURL:
                      type: string
                    path:
                      type: string
                destination:
                  type: object
                  properties:
                    server:
                      type: string
                    namespace:
                      type: string
```

### 2.2 使用 CRD 定义的资源

CRD 注册后，就可以创建自定义资源（CR）：

```yaml
# 这就是你在 Helm 笔记中看到的 ArgoCD Application 资源
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: plaud-project-summary
  namespace: argocd
spec:
  source:
    repoURL: git@github.com:Plaud-AI/deploy.git
    path: plaud-project-summary/applicationsets
  destination:
    server: https://kubernetes.default.svc
    namespace: plaud-project-summary
```

> 这和你写 Deployment YAML 的体验完全一样——`kubectl apply` 提交到 API Server → 存储到 etcd → 被对应的控制器 watch 并处理。

### 2.3 查看集群中的 CRD

```bash
# 列出所有 CRD
kubectl get crd

# 常见输出（EKS + ArgoCD 集群）：
# NAME                                        CREATED AT
# applications.argoproj.io                     2024-01-15    ← ArgoCD
# applicationsets.argoproj.io                  2024-01-15    ← ArgoCD
# externalsecrets.external-secrets.io          2024-02-01    ← External Secrets
# prometheusrules.monitoring.coreos.com        2024-01-20    ← Prometheus Operator
# servicemonitors.monitoring.coreos.com        2024-01-20    ← Prometheus Operator
# eniconfigs.crd.k8s.amazonaws.com             2024-01-10    ← VPC CNI
```

### 🔗 实战链接：项目中的 CRD 使用全景

在我们的 EKS 集群中，以下 CRD 每天都在运行：

**1) ExternalSecret — 自动同步 AWS Secrets Manager 密钥**

`generic-deployer/templates/externalsecret.yaml` 模板为每个微服务生成 ExternalSecret CR：

```yaml
# 来自 plaud-admin 的 values 配置（脱敏）
externalSecrets:
  - name: plaud-admin-es
    secretStoreRef:
      name: aws-secrets-manager-store
      kind: ClusterSecretStore          # 集群级别的 SecretStore，所有 namespace 可用
    target:
      name: plaud-admin-es
      template:
        type: Opaque
        data:
          DB_USER: "{{ .username }}"     # 从 AWS Secrets Manager 映射字段
          DB_PASSWORD: "{{ .password }}"
          DB_HOST: "{{ .host }}"
    dataFrom:
      - extract:
          key: db/global-plaud-mysql     # AWS Secrets Manager 中的密钥路径
```

> 这就是 CRD 的价值：开发者只需声明"我要哪个密钥的哪些字段"，External Secrets Operator 自动从 AWS Secrets Manager 拉取并创建 K8s Secret。

**2) ServiceMonitor — 声明式 Prometheus 监控**

`generic-deployer/templates/servicemonitor.yaml` 模板自动为微服务配置指标采集：

```yaml
# 模板渲染后的效果（以某微服务为例）
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: plaud-sync-jp-prod-main-deployer
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - plaud-sync                       # 监控目标 namespace
  endpoints:
    - port: web
      path: /metrics
      interval: 15s                      # 每 15 秒采集一次
```

> 服务只需在 values 中设置 `serviceMonitor.enabled: true`，Prometheus Operator 就会自动发现并采集该服务的指标。

**3) Kafka / KafkaNodePool — Strimzi Operator 管理的 Kafka 集群**

```yaml
# infra/values/strimzi-kafka/base/Kafka.yaml
apiVersion: kafka.strimzi.io/v1
kind: Kafka
metadata:
  name: kafka-log
spec:
  kafka:
    version: 4.1.1
    listeners:
      - name: plain
        port: 9092
        type: internal
    config:
      default.replication.factor: 3
      min.insync.replicas: 2
  entityOperator:
    topicOperator: {}          # 自动管理 Kafka Topic
    userOperator: {}           # 自动管理 Kafka User
  cruiseControl:
    autoRebalance:             # 自动均衡分区
      - mode: add-brokers
      - mode: remove-brokers
```

> `Kafka` 和 `KafkaNodePool` 都是 Strimzi Operator 定义的 CRD。你用一个 YAML 声明"我要一个 3 节点 Kafka 集群"，Operator 自动创建 StatefulSet、Service、PVC 等底层资源，并处理滚动升级、分区再均衡等复杂运维。

---

## 三、Operator 模式（进阶）

### 3.1 Operator = CRD + 自定义 Controller

CRD 只是定义了"新的资源长什么样"，还需要一个 Controller 来"看到这个资源后做什么"。两者组合就是 Operator。

```
┌──── CRD ────────────┐     ┌──── Controller ──────────────────────┐
│ "定义新概念"          │     │ "看到新概念后执行逻辑"                  │
│                      │     │                                      │
│ Application CRD      │     │ ArgoCD Application Controller        │
│ (定义了 source、     │ ──→ │ watch Application 资源                │
│  destination 等字段)  │     │ 发现新的 → clone Git repo → Helm 渲染  │
│                      │     │ → 部署到目标集群                       │
│                      │     │ 发现变更 → 自动同步/告警                 │
└──────────────────────┘     └──────────────────────────────────────┘
```

### 3.2 Operator 的核心价值

Operator 将**运维知识编码为软件**。以数据库 Operator 为例：

```
手动运维 PostgreSQL：
1. 创建 StatefulSet + PVC                    ← 部署
2. 配置主从复制                               ← 需要 DBA 知识
3. 监控复制延迟                               ← 需要告警配置
4. 主库挂了 → 手动 failover → 提升从库为主库   ← 需要深夜 oncall
5. 定期备份 → 验证备份可恢复                   ← 需要定时任务

CloudNativePG Operator 自动完成：
1. kubectl apply -f cluster.yaml             ← 声明式定义
2. Operator 自动：
   ├── 创建主库 + 从库 Pod
   ├── 配置流复制
   ├── 监控健康状态
   ├── 主库挂了 → 自动 failover（秒级）
   ├── 定期备份到 S3
   └── 支持时间点恢复（PITR）
```

### 3.3 本项目使用的 Operator

| Operator | CRD | 用途 |
|------|------|------|
| **ArgoCD** | Application, ApplicationSet | GitOps 持续部署（[[10-helm-argocd-deployment#2.3 ArgoCD ApplicationSet + Helm 的协作|详见 Helm 笔记]]） |
| **External Secrets** | ExternalSecret, SecretStore, ClusterSecretStore | 从 AWS Secrets Manager 同步密钥（[[08-k8s-security-rbac#5.2 ExternalSecret — 外部密钥管理（推荐）（进阶）|详见安全笔记]]） |
| **Prometheus Operator** | ServiceMonitor, PrometheusRule | 监控指标采集和告警规则 |
| **Strimzi Kafka Operator** | Kafka, KafkaNodePool, KafkaTopic | Kafka 集群全生命周期管理 |
| **ClickHouse Operator** | ClickHouseInstallation | ClickHouse 集群部署与运维 |
| **Nginx Ingress Controller** | IngressClass | 反向代理和流量路由 |
| **EBS CSI Driver** | — (CSI 标准接口) | EBS 卷的生命周期管理 |

### 🔗 实战链接：Strimzi Kafka Operator 的 Operator 模式实践

Strimzi 是一个典型的 Level 4-5 Operator，我们的项目通过它管理多区域 Kafka 集群。

**Operator 安装**（通过 ArgoCD ApplicationSet 分发到所有集群）：

```yaml
# infra/applicationsets/strimzi-kafka-operator.yaml（简化）
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: strimzi-kafka-operator
spec:
  generators:
    - list:
        elements:
          - name: jp-prod
            targetRevision: 0.50.0       # Operator 版本
          - name: cn-prod
            targetRevision: 0.50.0
          # ... 11 个集群
  template:
    spec:
      sources:
        - repoURL: quay.io/strimzi-helm  # Strimzi 官方 Helm Chart
          targetRevision: "{{ targetRevision }}"
          chart: strimzi-kafka-operator
          helm:
            valueFiles:
              - $values/infra/values/strimzi-kafka-operator/default.yaml
        - repoURL: git@github.com:Plaud-AI/deploy.git
          targetRevision: main
          ref: values                    # 多源引用：Helm Chart + 自定义 values
```

**通过 CRD 声明 Kafka 集群**（上面 2.3 节已展示 Kafka CR），Operator 自动完成：
- 创建 KRaft 模式的 Kafka 节点（broker + controller 双角色）
- 配置跨可用区拓扑分散（`topologySpreadConstraints`）
- 管理持久化存储（JBOD 模式，每节点 100Gi ~ 1Ti）
- 内置 CruiseControl 自动分区再均衡

**不同环境通过 Kustomize overlay 差异化资源**：

```yaml
# base: 8Gi 内存 + 100Gi 存储（默认配置）
# overlays/ap-northeast-1/prod/kafka-nodepool-patch.yaml:
apiVersion: kafka.strimzi.io/v1
kind: KafkaNodePool
metadata:
  name: dual-role
spec:
  resources:
    requests:
      memory: 32Gi                       # 生产环境：32Gi（4 倍于 base）
    limits:
      memory: 32Gi
  storage:
    type: jbod
    volumes:
      - size: 1Ti                        # 生产环境：1Ti（10 倍于 base）
```

> 这就是 Operator 的核心价值：运维知识编码为软件。你不需要手动配置 Kafka 的副本同步、分区分配、滚动升级——Strimzi Operator 都帮你处理了。

### 3.4 Operator 的成熟度模型

社区定义了 Operator 的五个成熟度级别：

```
Level 1: 基础安装
         └── 自动化部署和配置

Level 2: 无缝升级
         └── 支持版本升级和补丁

Level 3: 完整生命周期
         └── 备份、恢复、故障转移

Level 4: 深度洞察
         └── 指标、告警、日志集成

Level 5: 自动驾驶
         └── 自动调优、自动扩缩容、异常自愈
```

> ArgoCD 和 Prometheus Operator 处于 Level 4-5，External Secrets Operator 处于 Level 3。

---

## 四、Admission Webhook — 请求拦截与修改（深入）

### 4.1 两种 Webhook

Admission Webhook 在 API Server 处理请求的链路中拦截请求（认证和授权之后，写入 etcd 之前）：

```
API 请求 → 认证 → 授权 → Mutating Webhook → Validating Webhook → etcd
                           │                    │
                           │ 可以修改请求         │ 只能通过/拒绝
                           │ （注入 sidecar 等）  │ （校验策略合规性）
```

| Webhook 类型 | 能做什么 | 典型场景 |
|------|------|------|
| **Mutating** | 修改请求内容 | 注入 sidecar（Istio）、添加默认标签、注入环境变量 |
| **Validating** | 通过或拒绝请求 | 镜像来源检查、资源配额校验、命名规范检查 |

### 4.2 实际例子：Istio Sidecar 注入

当你的 namespace 开启了 Istio 注入，每次创建 Pod 时：

```yaml
# 你提交的 Pod spec
spec:
  containers:
    - name: api
      image: myapp:latest

# Istio 的 Mutating Webhook 自动修改为：
spec:
  containers:
    - name: api
      image: myapp:latest
    - name: istio-proxy              # ← 自动注入的 sidecar
      image: istio/proxyv2:latest
  initContainers:
    - name: istio-init               # ← 自动注入的初始化容器
      image: istio/proxyv2:latest
```

> 开发者的 YAML 里只有一个容器，但实际运行的 Pod 有两个——Mutating Webhook 在中间"偷偷"加了一个。

### 4.3 策略引擎

策略引擎通过 Validating Webhook 实现集群级别的安全和合规策略：

| 工具 | 特点 |
|------|------|
| **OPA/Gatekeeper** | 通用策略引擎，用 Rego 语言编写策略 |
| **Kyverno** | K8s 原生策略引擎，用 YAML 编写策略（更易用） |

```yaml
# Kyverno 示例：禁止使用 latest 标签
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-image-tag
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Using 'latest' tag is not allowed"
        pattern:
          spec:
            containers:
              - image: "!*:latest"    # 不允许 latest 标签
```

### 🔗 实战链接：PrometheusRule — 用 CRD 声明告警规则

我们的集群通过 `PrometheusRule` CRD 管理所有告警规则，而不是手动编辑 Prometheus 配置文件：

```yaml
# prometheusrule/rule.yaml（节选，脱敏）
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-pod-alert-rules
  namespace: monitoring
  labels:
    prometheus: k8s              # Prometheus Operator 通过此标签发现规则
    role: alert-rules
spec:
  groups:
    - name: node.rules
      rules:
      - alert: NodeMemoryUsage
        expr: |
          100 - (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes)
          / node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Instance {{ $labels.instance }} 内存使用率过高"

    - name: pod.rules
      rules:
      - alert: PodCrashLoopBackOff
        expr: |
          sum by(namespace,pod) (kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}) == 1
        for: 1m
        labels:
          severity: warning
        annotations:
          description: "命名空间: {{ $labels.namespace }} | Pod: {{ $labels.pod }} CrashLoopBackOff"

      - alert: KubernetesContainerOomKiller
        expr: |
          (kube_pod_container_status_restarts_total - kube_pod_container_status_restarts_total offset 10m >= 1)
          and ignoring (reason) min_over_time(kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}[10m]) == 1
        for: 0m
        labels:
          severity: warning
        annotations:
          description: "{{ $labels.namespace }}/{{ $labels.pod }} 在过去 10 分钟内被 OOMKilled {{ $value }} 次"
```

> 这体现了 CRD 的声明式优势：告警规则和 K8s 资源一样通过 YAML 管理，可以 Git 版本控制、Code Review、ArgoCD 自动同步。Prometheus Operator watch 到 PrometheusRule 变更后，自动更新 Prometheus 的告警配置。

---

## 五、Helm vs Kustomize

两种主流的 K8s 配置管理方式，本项目使用 Helm（见 [[10-helm-argocd-deployment|Helm 笔记]]），这里对比 Kustomize。

### 5.1 核心理念对比

| 方面 | Helm | Kustomize |
|------|------|------|
| **核心思路** | 模板化：用 Go 模板生成 YAML | 叠加（Overlay）：在基础 YAML 上打补丁 |
| **配置方式** | `values.yaml` 传参数 → 模板渲染 | `kustomization.yaml` 声明补丁 → 覆盖合并 |
| **学习曲线** | 需要学 Go 模板语法 | 纯 YAML，无模板语法 |
| **包管理** | 有（Chart 仓库、版本、依赖） | 无（只是配置管理） |
| **回滚** | `helm rollback`（内置版本管理） | 需要靠 Git 回滚 |
| **社区生态** | 丰富的公共 Chart（Bitnami 等） | kubectl 内置支持 |

### 5.2 Kustomize 示例

```
# 目录结构
├── base/                       # 基础配置
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── staging/                # Staging 覆盖
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    └── production/             # Production 覆盖
        ├── kustomization.yaml
        └── patch-resources.yaml
```

```yaml
# base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml

# overlays/production/kustomization.yaml
resources:
  - ../../base                  # 引用基础配置
patchesStrategicMerge:
  - patch-resources.yaml        # 覆盖 resources

# overlays/production/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 4                   # 覆盖副本数
  template:
    spec:
      containers:
        - name: api
          resources:             # 覆盖资源
            limits:
              cpu: 8
              memory: 16Gi
```

```bash
# 渲染预览
kubectl kustomize overlays/production/

# 直接部署
kubectl apply -k overlays/production/
```

### 5.3 什么时候用什么

| 场景 | 推荐 | 原因 |
|------|------|------|
| 多环境部署（dev/staging/prod） | 两者都行 | Helm 用不同 values 文件，Kustomize 用不同 overlay |
| 多集群/多区域部署 | **Helm** + ArgoCD | Chart 的参数化能力 + ApplicationSet 的编排能力 |
| 需要版本管理和回滚 | **Helm** | 内置 release 版本管理 |
| 简单的配置差异（只改几个字段） | **Kustomize** | 不需要模板，直接打补丁 |
| 公共组件分发 | **Helm** | Chart 仓库生态 |
| 团队不熟悉 Go 模板 | **Kustomize** | 纯 YAML 无学习成本 |

> **本项目选择 Helm** 的原因：需要一个通用的 `generic-deployer` Chart 作为所有微服务的基座，通过 values 文件参数化多环境、多区域配置，配合 ArgoCD ApplicationSet 实现多集群分发。这是 Helm 的典型优势场景。

### 🔗 实战链接：项目中 Helm 与 Kustomize 的混合使用

有趣的是，我们的项目并非只用 Helm——基础设施层（infra）部分使用了 Kustomize：

**微服务层 → Helm**（80+ 服务统一使用 `generic-deployer` Chart）：

```text
plaud-sync/
├── Chart.yaml                         # 依赖 generic-deployer
└── values/
    ├── ap-northeast-1/prod/values-main.yaml   # Helm values 参数化
    └── us-west-2/staging/values-main.yaml
```

**基础设施层 → Kustomize**（Kafka、External Secrets 等 Operator 管理的资源）：

```text
infra/values/strimzi-kafka/
├── base/                              # 基础 Kafka 集群定义
│   ├── kustomization.yaml
│   ├── Kafka.yaml                     # Kafka CRD（8Gi 内存、100Gi 存储）
│   └── KafkaNodePool.yaml
└── overlays/
    ├── ap-northeast-1/prod/           # JP 生产环境覆盖
    │   ├── kustomization.yaml         # resources: [../../../base] + patches
    │   └── kafka-nodepool-patch.yaml  # 32Gi 内存、1Ti 存储
    ├── cn-northwest-1/prod/           # CN 生产环境覆盖
    └── us-west-2/staging/             # US 测试环境（使用 base 默认值）
```

> **选择依据**：微服务有复杂的模板化需求（Deployment、Service、Ingress、HPA 等多种资源），适合 Helm；基础设施是简单的 CRD 资源差异化（只改内存、存储大小等几个字段），适合 Kustomize overlay 直接打补丁。两者在同一项目中各取所长。

---

## 六、自定义控制器开发（概览）（深入）

如果你需要写自己的 Operator，有几种主流框架：

| 框架 | 语言 | 特点 |
|------|------|------|
| **kubebuilder** | Go | 官方推荐，脚手架工具 |
| **Operator SDK** | Go / Ansible / Helm | Red Hat 维护，支持多种方式 |
| **kopf** | Python | Python 友好，适合快速原型 |
| **metacontroller** | 任何语言 | 通过 Webhook 实现，无需学 K8s 客户端库 |

一个最简单的 Python 控制器（使用 kopf）：

```python
import kopf
import kubernetes

@kopf.on.create('mygroup.io', 'v1', 'myresources')
def create_fn(spec, name, namespace, **kwargs):
    """当创建 MyResource 时，自动创建对应的 Deployment"""
    api = kubernetes.client.AppsV1Api()
    deployment = {
        'apiVersion': 'apps/v1',
        'kind': 'Deployment',
        'metadata': {'name': name},
        'spec': {
            'replicas': spec.get('replicas', 1),
            'selector': {'matchLabels': {'app': name}},
            'template': {
                'metadata': {'labels': {'app': name}},
                'spec': {
                    'containers': [{
                        'name': 'main',
                        'image': spec['image'],
                    }]
                }
            }
        }
    }
    api.create_namespaced_deployment(namespace, deployment)
    return {'message': f'Deployment {name} created'}
```

> 大多数场景不需要自己写 Operator——社区已有成熟的 Operator。只有当你有领域特定的运维逻辑需要自动化时，才考虑自己开发。

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — ArgoCD Application/ApplicationSet 的实际使用
- [[05-k8s-architecture|K8s 架构原理]] — 声明式 API 和控制器模式的基础
- [[08-k8s-security-rbac|K8s 安全与权限]] — Admission Webhook 在安全中的应用
- [[03-k8s-workload-types|K8s 工作负载类型]] — Operator 管理 StatefulSet 的高级用法
