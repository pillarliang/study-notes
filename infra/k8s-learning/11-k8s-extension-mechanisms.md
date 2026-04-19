# K8s 扩展机制：CRD、Operator 与 Admission Webhook

> 前置知识：了解 [[05-k8s-architecture#五、声明式 API 与控制器模式|声明式 API 与控制器模式]]。
>
> **核心问题**：K8s 内置资源（Deployment、Service、ConfigMap）覆盖了通用场景，但生产环境有大量领域特定需求——GitOps 部署、外部密钥同步、Kafka 集群运维——这些场景如何用声明式的方式管理？

---

## 一、为什么需要扩展 K8s

K8s 提供了扩展机制，允许在集群中定义全新的资源类型并编写控制器来管理它们。以 plaud-project-summary 为例，一个微服务至少依赖 3 种扩展资源：

| 需求 | K8s 内置能做吗 | 扩展方案 |
|------|------|------|
| 声明式 GitOps 部署 | 不能（需要 watch Git 仓库） | ArgoCD（CRD: Application） |
| 自动同步外部密钥 | 不能（Secret 是静态的） | External Secrets Operator（CRD: ExternalSecret） |
| 监控告警规则管理 | 不能 | Prometheus Operator（CRD: PrometheusRule） |
| Kafka 集群运维 | StatefulSet 太底层 | Strimzi（CRD: Kafka, KafkaNodePool） |

这些资源都不是 K8s 内置的——它们由 **CRD 定义**、由 **Operator 管理**。下面从 CRD 开始，逐层展开。

---

## 二、CRD（Custom Resource Definition）

**问题**：如何让 K8s 认识一种全新的资源类型？

### 2.1 CRD 是什么

CRD 向 API Server 注册新的资源类型。注册后，新资源与 Deployment、Service 地位相同——可以用 `kubectl apply` 创建、`kubectl get` 查询、存储在 etcd 中。

```yaml
# 来源：ArgoCD 安装时注册的 CRD（简化）
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.argoproj.io
spec:
  group: argoproj.io                # API 组
  names:
    kind: Application               # 资源类型名
    plural: applications             # kubectl get applications
    shortNames: ["app", "apps"]      # kubectl get apps
  scope: Namespaced
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
                destination:
                  type: object
```

### 2.2 plaud-project-summary 用到的 CRD

CRD 注册后，即可创建自定义资源（CR）。以下三种 CR 在每个微服务的部署流程中都会出现：

```yaml
# 来源：deploy/plaud-project-summary/values（Helm 渲染后的等效资源）

# 1) ExternalSecret — 自动从 AWS Secrets Manager 拉取密钥
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: plaud-project-summary-es
spec:
  secretStoreRef:
    name: aws-secrets-manager-store
    kind: ClusterSecretStore
  target:
    name: plaud-project-summary-es
  dataFrom:
    - extract:
        key: db/global-plaud-mysql

# 2) ServiceMonitor — 声明式配置 Prometheus 指标采集
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: plaud-project-summary-monitor
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames: [plaud-project-summary]
  endpoints:
    - port: web
      path: /metrics
      interval: 15s

# 3) Application — ArgoCD GitOps 部署
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

> 这些资源与 Deployment YAML 的使用体验完全一样——`kubectl apply` 提交到 API Server → 存储到 etcd → 被对应的控制器 watch 并处理。

### 2.3 ExternalSecret 的 Helm 模板化（进阶）

`generic-deployer/templates/externalsecret.yaml` 模板为每个微服务生成 ExternalSecret CR，开发者只需在 values 中声明要哪个密钥的哪些字段：

```yaml
# 来源：deploy/plaud-admin/values（脱敏）
externalSecrets:
  - name: plaud-admin-es
    secretStoreRef:
      name: aws-secrets-manager-store
      kind: ClusterSecretStore          # 集群级别，所有 namespace 可用
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

### 2.4 查看集群中的 CRD（深入）

```bash
kubectl get crd

# 常见输出（EKS + ArgoCD 集群）：
# NAME                                        CREATED AT
# applications.argoproj.io                     2024-01-15    ← ArgoCD
# applicationsets.argoproj.io                  2024-01-15    ← ArgoCD
# externalsecrets.external-secrets.io          2024-02-01    ← External Secrets
# servicemonitors.monitoring.coreos.com        2024-01-20    ← Prometheus Operator
# eniconfigs.crd.k8s.amazonaws.com             2024-01-10    ← VPC CNI
```

**CRD 只定义了"新的资源长什么样"**，还需要一个 Controller 来处理"看到这个资源后做什么"——两者组合就是 Operator。

---

## 三、Operator 模式（进阶）

**问题**：CRD 定义了新概念，谁来执行对应的逻辑？

### 3.1 Operator = CRD + 自定义 Controller

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

### 3.2 核心价值：将运维知识编码为软件

以数据库 Operator 为例：

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

### 3.3 实战：Strimzi Kafka Operator（进阶）

Strimzi 是一个典型的高成熟度 Operator。通过 CRD 声明 Kafka 集群后，Operator 自动完成 KRaft 模式配置、跨可用区拓扑分散、持久化存储管理、CruiseControl 自动分区再均衡。

**Operator 安装**——通过 ArgoCD ApplicationSet 分发到所有集群：

```yaml
# 来源：deploy/infra/applicationsets/strimzi-kafka-operator.yaml（简化）
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: strimzi-kafka-operator
spec:
  generators:
    - list:
        elements:
          - name: jp-prod
            targetRevision: 0.50.0
          - name: cn-prod
            targetRevision: 0.50.0
          # ... 11 个集群
  template:
    spec:
      sources:
        - repoURL: quay.io/strimzi-helm
          targetRevision: "{{ targetRevision }}"
          chart: strimzi-kafka-operator
        - repoURL: git@github.com:Plaud-AI/deploy.git
          targetRevision: main
          ref: values
```

**Kafka 集群声明**——一个 YAML 得到完整集群：

```yaml
# 来源：deploy/infra/values/strimzi-kafka/base/Kafka.yaml
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
    autoRebalance:
      - mode: add-brokers
      - mode: remove-brokers
```

**不同环境通过 Kustomize overlay 差异化资源**：

```yaml
# 来源：deploy/infra/values/strimzi-kafka/overlays/ap-northeast-1/prod/kafka-nodepool-patch.yaml
# base: 8Gi 内存 + 100Gi 存储（默认配置）
# 生产环境 overlay：
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

> 无需手动配置 Kafka 的副本同步、分区分配、滚动升级——Strimzi Operator 将这些运维操作全部自动化。

### 3.4 本项目使用的 Operator 一览

| Operator | CRD | 用途 |
|------|------|------|
| **ArgoCD** | Application, ApplicationSet | GitOps 持续部署（[[10-helm-argocd-deployment#2.3 ArgoCD ApplicationSet + Helm 的协作\|详见 Helm 笔记]]） |
| **External Secrets** | ExternalSecret, SecretStore, ClusterSecretStore | 从 AWS Secrets Manager 同步密钥（[[08-k8s-security-rbac#5.2 ExternalSecret — 外部密钥管理（推荐）（进阶）\|详见安全笔记]]） |
| **Prometheus Operator** | ServiceMonitor, PrometheusRule | 监控指标采集和告警规则 |
| **Strimzi Kafka Operator** | Kafka, KafkaNodePool, KafkaTopic | Kafka 集群全生命周期管理 |
| **ClickHouse Operator** | ClickHouseInstallation | ClickHouse 集群部署与运维 |
| **Nginx Ingress Controller** | IngressClass | 反向代理和流量路由 |
| **EBS CSI Driver** | — (CSI 标准接口) | EBS 卷的生命周期管理 |

### 3.5 Operator 的成熟度模型（深入）

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

## 四、Admission Webhook — 请求拦截与修改（进阶）

**问题**：CRD + Operator 解决了"扩展新资源"，但如何在资源写入 etcd 之前拦截并校验/修改请求？

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

### 4.2 实战：PrometheusRule 的 Validating Webhook

通过 `PrometheusRule` CRD 管理所有告警规则。`kubectl apply` 一条告警规则时，Prometheus Operator 的 Validating Webhook 先校验规则语法是否合法，再由 Controller 同步到 Prometheus 配置：

```yaml
# 来源：deploy/prometheusrule/rule.yaml（节选，脱敏）
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-pod-alert-rules
  namespace: monitoring
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  groups:
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
    - name: node.rules
      rules:
      - alert: NodeMemoryUsage
        expr: |
          100 - (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes)
          / node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
```

> 告警规则和 K8s 资源一样通过 YAML 管理，可以 Git 版本控制、Code Review、ArgoCD 自动同步。Prometheus Operator watch 到 PrometheusRule 变更后，自动更新 Prometheus 的告警配置——不需要手动编辑配置文件或重启服务。

### 4.3 实际例子：Istio Sidecar 注入（Mutating Webhook）（进阶）

当 namespace 开启了 Istio 注入，每次创建 Pod 时，Mutating Webhook 自动修改 Pod spec：

```yaml
# 提交的 Pod spec
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

> 开发者的 YAML 里只有一个容器，但实际运行的 Pod 有两个——Mutating Webhook 在写入 etcd 之前"偷偷"加了一个。

### 4.4 策略引擎（深入）

策略引擎通过 Validating Webhook 实现集群级别的安全和合规策略：

| 工具 | 特点 |
|------|------|
| **OPA/Gatekeeper** | 通用策略引擎，用 Rego 语言编写策略 |
| **Kyverno** | K8s 原生策略引擎，用 YAML 编写策略（更易用） |

```yaml
# 来源：Kyverno 官方示例（禁止使用 latest 标签）
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
              - image: "!*:latest"
```

---

## 五、Helm vs Kustomize — 两种配置管理方式

**问题**：有了 CRD 和 Operator 后，大量 YAML 配置如何管理多环境差异？

本项目同时使用了 Helm 和 Kustomize，各取所长：

```text
# 微服务层 → Helm（80+ 服务统一使用 generic-deployer Chart）
# 来源：deploy/plaud-sync/
plaud-sync/
├── Chart.yaml                         # 依赖 generic-deployer
└── values/
    ├── ap-northeast-1/prod/values-main.yaml
    └── us-west-2/staging/values-main.yaml

# 基础设施层 → Kustomize（Operator 管理的 CRD 资源）
# 来源：deploy/infra/values/strimzi-kafka/
infra/values/strimzi-kafka/
├── base/                              # 基础 Kafka 集群定义（8Gi、100Gi）
│   ├── kustomization.yaml
│   ├── Kafka.yaml
│   └── KafkaNodePool.yaml
└── overlays/
    ├── ap-northeast-1/prod/           # JP 生产（32Gi、1Ti）
    ├── cn-northwest-1/prod/           # CN 生产
    └── us-west-2/staging/             # US 测试（使用 base 默认值）
```

### 5.1 核心理念对比

| 方面 | Helm | Kustomize |
|------|------|------|
| **核心思路** | 模板化：用 Go 模板生成 YAML | 叠加（Overlay）：在基础 YAML 上打补丁 |
| **配置方式** | `values.yaml` 传参数 → 模板渲染 | `kustomization.yaml` 声明补丁 → 覆盖合并 |
| **学习曲线** | 需要学 Go 模板语法 | 纯 YAML，无模板语法 |
| **包管理** | 有（Chart 仓库、版本、依赖） | 无（只是配置管理） |
| **回滚** | `helm rollback`（内置版本管理） | 需要靠 Git 回滚 |

### 5.2 Kustomize 示例

```yaml
# base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml

# overlays/production/kustomization.yaml
resources:
  - ../../base
patchesStrategicMerge:
  - patch-resources.yaml

# overlays/production/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: api
          resources:
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

### 5.3 选型依据

| 场景 | 推荐 | 原因 |
|------|------|------|
| 多集群/多区域部署 | **Helm** + ArgoCD | Chart 的参数化能力 + ApplicationSet 的编排能力 |
| 需要版本管理和回滚 | **Helm** | 内置 release 版本管理 |
| 简单的配置差异（只改几个字段） | **Kustomize** | 不需要模板，直接打补丁 |
| 公共组件分发 | **Helm** | Chart 仓库生态 |

> 本项目选择 Helm 作为主力：需要一个通用的 `generic-deployer` Chart 作为所有微服务的基座，通过 values 文件参数化多环境、多区域配置，配合 ArgoCD ApplicationSet 实现多集群分发。而基础设施层的简单字段差异化则交给 Kustomize。

---

## 六、自定义控制器开发（概览）（深入）

**问题**：如果社区没有现成的 Operator，如何自行开发？

| 框架 | 语言 | 特点 |
|------|------|------|
| **kubebuilder** | Go | 官方推荐，脚手架工具 |
| **Operator SDK** | Go / Ansible / Helm | Red Hat 维护，支持多种方式 |
| **kopf** | Python | Python 友好，适合快速原型 |
| **metacontroller** | 任何语言 | 通过 Webhook 实现，无需学 K8s 客户端库 |

最简 Python 控制器（使用 kopf）：

```python
# 来源：kopf 官方示例（简化）
import kopf
import kubernetes

@kopf.on.create('mygroup.io', 'v1', 'myresources')
def create_fn(spec, name, namespace, **kwargs):
    """当创建 MyResource 时，自动创建对应的 Deployment。"""
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

> 大多数场景不需要自己写 Operator——社区已有成熟的 Operator。只有当存在领域特定的运维逻辑需要自动化时，才考虑自行开发。

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — ArgoCD Application/ApplicationSet 的实际使用
- [[05-k8s-architecture|K8s 架构原理]] — 声明式 API 和控制器模式的基础
- [[08-k8s-security-rbac|K8s 安全与权限]] — Admission Webhook 在安全中的应用
- [[03-k8s-workload-types|K8s 工作负载类型]] — Operator 管理 StatefulSet 的高级用法
