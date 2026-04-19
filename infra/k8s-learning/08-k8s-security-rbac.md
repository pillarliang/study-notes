# K8s 安全与权限

> 前置知识：了解 [[05-k8s-architecture#3.1 API Server — 唯一的入口|API Server 的认证/授权流程]]和 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|ServiceAccount（IRSA）的基本用法]]。本文系统介绍 K8s 的安全体系——谁能做什么、Pod 有什么权限、Secret 怎么管理。
>
> **学习建议**：首次阅读重点关注第一节（安全概览）、RBAC 核心（2.1–2.3）、Pod Security 概念（3.1）、Secret 基础（5.1、5.3）和第七节检查清单。标记为（进阶）的小节和第四、六节掌握基础后再看。

---

## 一、K8s 安全的四个层面

```
请求到达 API Server 的处理链路：

┌────────────────────────────────────────────────────────────────┐
│  ① Authentication（认证）：你是谁？                               │
│     证书 / Token / OIDC / ServiceAccount Token                  │
├────────────────────────────────────────────────────────────────┤
│  ② Authorization（授权/RBAC）：你有权限吗？                       │
│     Role + RoleBinding → 允许/拒绝                               │
├────────────────────────────────────────────────────────────────┤
│  ③ Admission Control（准入控制）：请求合法吗？                     │
│     Mutating Webhook → Validating Webhook                       │
├────────────────────────────────────────────────────────────────┤
│  ④ 运行时安全：Pod 运行时有什么权限？                               │
│     Pod Security Standards / SecurityContext                     │
└────────────────────────────────────────────────────────────────┘
```

---

## 二、RBAC（基于角色的访问控制）

### 2.1 核心概念

RBAC 回答的问题是：**谁（Subject）** 可以对 **什么资源（Resource）** 做 **什么操作（Verb）**。

```
┌──── Subject（谁） ────┐     ┌──── RoleBinding ────┐     ┌──── Role ────────────┐
│                       │     │                      │     │                      │
│  User（用户）          │     │  将 Subject 绑定到    │     │  定义权限：             │
│  Group（组）           │←───→│  Role                │←───→│  resources: pods      │
│  ServiceAccount（SA） │     │                      │     │  verbs: get, list     │
│                       │     │                      │     │                      │
└───────────────────────┘     └──────────────────────┘     └──────────────────────┘
```

**四种 RBAC 资源**：

| 资源 | 作用范围 | 说明 |
|------|------|------|
| **Role** | 单个 Namespace | 定义"可以对哪些资源做哪些操作" |
| **ClusterRole** | 整个集群 | 同上，但跨 namespace 或针对集群级资源 |
| **RoleBinding** | 单个 Namespace | 将 Role/ClusterRole 绑定到 Subject |
| **ClusterRoleBinding** | 整个集群 | 将 ClusterRole 绑定到 Subject（集群范围） |

### 2.2 Role 定义示例

```yaml
# 允许在 production namespace 中读取 Pod 和日志
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]           # 核心 API 组（Pod、Service 等）
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]   # Pod 的子资源
    verbs: ["get"]
```

**常见 verbs**：

| Verb | 对应操作 | HTTP 方法 |
|------|------|------|
| `get` | 获取单个资源 | GET |
| `list` | 列出资源 | GET（集合） |
| `watch` | 监听变更 | GET（watch） |
| `create` | 创建资源 | POST |
| `update` | 更新资源 | PUT |
| `patch` | 部分更新 | PATCH |
| `delete` | 删除资源 | DELETE |

### 2.3 RoleBinding 示例

```yaml
# 将 pod-reader 角色绑定给开发团队
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-reader
  namespace: production
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: ci-deployer           # CI/CD 使用的 SA
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 2.4 ClusterRole 常用模式（进阶）

```yaml
# 1. 只读查看器（适合 oncall 人员）
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-viewer
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]

# 2. Namespace 管理员（适合团队 lead）
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["namespaces"]    # 但不能创建/删除 namespace
    verbs: ["get", "list"]
```

> [!tip] 🔗 实战链接：generic-deployer 模板中的 ServiceAccount 创建
> deploy 项目的 `generic-deployer/templates/serviceaccount.yaml` 展示了如何通过 Helm 模板化 ServiceAccount 的创建：
> ```yaml
> # deploy 项目：generic-deployer/templates/serviceaccount.yaml
> {{- if .Values.serviceAccount.create -}}
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: {{ .Values.serviceAccount.name }}
>   labels:
>     {{ .Values.serviceAccount.labels }}
>   {{- with .Values.serviceAccount.annotations }}
>   annotations:
>     {{- toYaml . | nindent 4 }}
>   {{- end }}
> automountServiceAccountToken: {{ .Values.serviceAccount.automount }}
> {{- end }}
> ```
> 每个微服务在 values 文件中通过 `serviceAccount.create: true` 和 `serviceAccount.annotations` 来控制是否创建 SA 以及附加哪些注解（如 IRSA 角色）。`automountServiceAccountToken` 控制是否自动挂载 SA Token 到 Pod——RBAC 的 Subject 之一就是 ServiceAccount。

### 2.5 EKS 中的 RBAC（进阶）

EKS 将 AWS IAM 身份映射到 K8s RBAC：

```
AWS IAM User/Role
    ↓ aws-auth ConfigMap 映射
K8s User/Group
    ↓ RoleBinding / ClusterRoleBinding
K8s Role / ClusterRole
    ↓
具体权限
```

```yaml
# aws-auth ConfigMap（kube-system namespace）
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456:role/EKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:nodes
    - rolearn: arn:aws:iam::123456:role/DevTeamRole
      username: dev-user
      groups:
        - dev-team              # 对应 RoleBinding 中的 Group
```

### 2.6 RBAC 排障

```bash
# 检查当前用户的权限
kubectl auth can-i create deployments -n production
# yes / no

# 列出所有权限
kubectl auth can-i --list -n production

# 检查特定 ServiceAccount 的权限
kubectl auth can-i get pods -n production --as=system:serviceaccount:production:ci-deployer

# 查看 RoleBinding
kubectl get rolebindings -n production
kubectl describe rolebinding dev-pod-reader -n production
```

---

## 三、Pod Security — 限制 Pod 的运行时权限

> [!tip] 🔗 实战链接：plaud-project-summary 的 IRSA 配置
> `plaud-project-summary` 通过 IRSA（IAM Roles for Service Accounts）获取 AWS 权限。以日本区生产环境为例：
> ```yaml
> # plaud-project-summary/values/ap-northeast-1/prod/main.yaml
> serviceAccount:
>   create: true
>   automount: true
>   annotations:
>     eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/plaud-project-summary-role
>   name: plaud-project-summary
> ```
> 这个注解 `eks.amazonaws.com/role-arn` 是 IRSA 的核心——它告诉 EKS 为这个 ServiceAccount 关联一个 IAM Role。Pod 启动时，AWS 会自动通过 OIDC 注入临时凭证，无需在环境变量中传递 AK/SK。这是 RBAC + 云原生安全的最佳实践：**K8s RBAC 控制集群内权限，IRSA 控制 AWS 云资源权限**。
>
> 注意中国区的 ARN 格式不同（`aws-cn` 分区）：
> ```yaml
> # plaud-project-summary/values/cn-northwest-1/prod/main.yaml
> annotations:
>   eks.amazonaws.com/role-arn: arn:aws-cn:iam::<account-id>:role/plaud-project-summary-role
> ```
>
> 对比 staging 环境中注释掉的反面做法——说明团队已经从静态密钥迁移到 IRSA：
>
> ```yaml
> # plaud-project-summary/values/ap-northeast-1/staging/main.yaml（已注释）
> # - name: AWS_ACCESS_KEY_ID
> #   valueFrom:
> #     secretKeyRef:
> #       name: plaud-project-summaryc-aws
> #       key: AWS_ACCESS_KEY_ID
> ```
> IRSA 方式不需要在任何地方存储密钥，是更安全的选择。

### 3.1 为什么需要限制 Pod 权限

默认情况下，Pod 中的容器可以做很多危险的事：

- 以 root 用户运行
- 访问节点的文件系统（hostPath）
- 使用特权模式（相当于在节点上有完全权限）
- 访问节点网络

**Pod Security Standards** 定义了三个安全级别：

| 级别 | 限制程度 | 适用场景 |
|------|------|------|
| **Privileged** | 无限制 | 系统组件（CNI、CSI 驱动） |
| **Baseline** | 禁止已知的危险配置 | 大多数应用（推荐最低标准） |
| **Restricted** | 最严格，遵循安全最佳实践 | 安全敏感的生产负载 |

### 3.2 Baseline 级别禁止的配置（进阶）

```yaml
# 以下配置在 Baseline 级别下被禁止：
spec:
  hostNetwork: true          # ❌ 使用节点网络
  hostPID: true              # ❌ 共享节点 PID namespace
  hostIPC: true              # ❌ 共享节点 IPC namespace
  containers:
    - securityContext:
        privileged: true     # ❌ 特权模式
        capabilities:
          add: ["SYS_ADMIN"] # ❌ 危险的 Linux capabilities
```

### 3.3 SecurityContext（Pod 级安全配置）（进阶）

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:              # Pod 级别
    runAsNonRoot: true          # 禁止以 root 运行
    runAsUser: 1000             # 以 UID 1000 运行
    fsGroup: 2000               # 卷文件的 GID
    seccompProfile:
      type: RuntimeDefault      # 启用 seccomp 限制系统调用
  containers:
    - name: app
      securityContext:          # 容器级别（覆盖 Pod 级别）
        allowPrivilegeEscalation: false   # 禁止提权
        readOnlyRootFilesystem: true      # 根文件系统只读
        capabilities:
          drop: ["ALL"]         # 丢弃所有 Linux capabilities
```

> [!tip] 🔗 实战链接：SecurityContext 在 deploy 项目中的两种模式
> **模式一：generic-deployer 默认值（安全模板，留白让服务自定义）**
> ```yaml
> # deploy 项目：generic-deployer/values.yaml
> podSecurityContext: {}
>   # fsGroup: 2000
> securityContext: {}
>   # capabilities:
>   #   drop:
>   #   - ALL
>   # readOnlyRootFilesystem: true
>   # runAsNonRoot: true
>   # runAsUser: 1000
> ```
> 默认值为空，由各服务按需覆盖。模板中 `{{- with .Values.securityContext }}` 的写法意味着不配置就不渲染。
>
> **模式二：需要特权的 sidecar 容器**
> ```yaml
> # deploy 项目：my-gateway 的 GoReplay sidecar
> extraContainers:
>   - name: goreplay-sidecar
>     image: buger/goreplay:latest
>     securityContext:
>       runAsUser: 0              # 必须 root
>       capabilities:
>         add: ["NET_RAW", "NET_ADMIN"]  # 需要网络抓包权限
> ```
> GoReplay 需要原始网络包捕获能力，所以显式添加了 `NET_RAW` 和 `NET_ADMIN` capabilities 并以 root 运行。这正是笔记 3.2 节提到的 **Baseline 级别禁止的配置**——在实践中，只有确实需要特权的系统级工具才应该这样配置。
>
> **模式三：静态文件服务（需要 root 操作文件系统）**
> ```yaml
> # deploy 项目：my-gateway 的 nginx 静态文件服务
> podSecurityContext:
>   runAsUser: 0                  # nginx 需要 root 绑定 80 端口
>   runAsGroup: 0
>   allowPrivilegeEscalation: false  # 但禁止提权
> ```
> 即使需要 root 运行，也应该禁止 `allowPrivilegeEscalation`，限制潜在的攻击面。

### 3.4 Pod Security Admission（PSA）（进阶）

PSA 是 K8s 内置的准入控制器，在 namespace 级别强制执行 Pod Security Standards：

```yaml
# 在 namespace 上设置安全级别
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: baseline    # 强制执行 baseline
    pod-security.kubernetes.io/warn: restricted     # 不满足 restricted 时警告
    pod-security.kubernetes.io/audit: restricted    # 审计日志记录
```

| 模式 | 行为 |
|------|------|
| `enforce` | 拒绝不符合的 Pod |
| `warn` | 允许但在 kubectl 输出中显示警告 |
| `audit` | 允许但记录到审计日志 |

---

## 四、NetworkPolicy 网络隔离（进阶）

详见 [[04-k8s-networking#六、NetworkPolicy — 网络隔离|K8s 网络]]中的完整介绍。这里强调安全最佳实践：

### 4.1 零信任网络模型

```yaml
# Step 1: 默认拒绝所有流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

# Step 2: 按需开放（最小权限）
# 允许 API Pod 接收来自 Ingress Controller 的流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8001

# 允许 API Pod 访问数据库
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    - to:                       # 允许 DNS 查询（否则无法解析 Service 名）
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
```

---

## 五、Secret 管理

### 5.1 K8s 原生 Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=        # base64 编码（不是加密！）
  password: bXlwYXNzd29yZA==
```

**K8s Secret 的问题**：
- **只是 base64 编码**，不是加密（`echo cG9zdGdyZXM= | base64 -d` → `postgres`）
- 存在 etcd 中，需要启用 etcd 加密（EKS 默认开启）
- 存在 Git 仓库中就是明文泄露

### 5.2 ExternalSecret — 外部密钥管理（推荐）（进阶）

本项目的 `generic-deployer` 支持 ExternalSecret（见 [[10-helm-argocd-deployment#2.1 generic-deployer：通用部署基座|通用部署基座]]），从 AWS Secrets Manager 同步密钥到 K8s Secret：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h           # 每小时从 AWS 同步一次
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials        # 生成的 K8s Secret 名称
  data:
    - secretKey: username
      remoteRef:
        key: production/db       # AWS Secrets Manager 中的 secret
        property: username
    - secretKey: password
      remoteRef:
        key: production/db
        property: password
```

**流程**：

```
AWS Secrets Manager（加密存储，IAM 权限控制）
    ↓ External Secrets Operator 定期同步
K8s Secret（自动创建/更新）
    ↓ Pod 挂载
容器环境变量 或 文件
```

> [!tip] 🔗 实战链接：ClusterSecretStore + ExternalSecret 完整链路
> deploy 项目中的密钥管理分为两层：基础设施层定义 ClusterSecretStore，应用层通过 ExternalSecret 消费。
>
> **第一层：ClusterSecretStore（集群级，所有 namespace 共享）**
> ```yaml
> # deploy 项目：infra/values/external-secrets/.../aws-secrets-manager-store.yaml
> apiVersion: external-secrets.io/v1
> kind: ClusterSecretStore
> metadata:
>   name: aws-secrets-manager-store
> spec:
>   provider:
>     aws:
>       service: SecretsManager
>       region: us-west-2
>       auth:
>         jwt:
>           serviceAccountRef:
>             name: external-secrets          # ESO 自身的 SA（也通过 IRSA 获取 AWS 权限）
>             namespace: external-secrets
> ```
>
> **第二层：应用侧通过环境变量引用 AWS Secrets Manager**
> `plaud-project-summary` 使用 AWS Secrets Manager 存储 Gemini API 密钥，通过环境变量告诉应用去哪里取：
> ```yaml
> # plaud-project-summary/values/ap-northeast-1/prod/main.yaml
> env:
>   - name: SECRETS_MANAGER_ENABLED
>     value: "true"
>   - name: SECRETS_MANAGER_GEMINI_KEY
>     value: prod/project-summary/gemini-key   # AWS Secrets Manager 中的 secret 路径
> ```
> staging 环境使用不同的 secret 路径（`dev/project-summary/gemini-key`），实现了**环境隔离**。密钥本身存储在 AWS Secrets Manager 中加密管理，**从未出现在 Git 仓库中**——应用通过 IRSA 获取的 IAM 权限来读取 secret。

> [!tip] 🔗 实战链接：generic-deployer 的 ExternalSecret 模板
> `generic-deployer/templates/externalsecret.yaml` 支持在一个服务中定义多个 ExternalSecret：
> ```yaml
> # deploy 项目：generic-deployer/templates/externalsecret.yaml（简化）
> {{- range .Values.externalSecrets }}
> apiVersion: external-secrets.io/v1
> kind: ExternalSecret
> metadata:
>   name: {{ .name | default (include "generic-deployer.fullname" $) }}
> spec:
>   refreshInterval: {{ .refreshInterval | default "1h" }}
>   secretStoreRef:
>     name: {{ .secretStoreRef.name }}
>     kind: {{ .secretStoreRef.kind }}
>   target:
>     name: {{ .target.name | default (include "generic-deployer.fullname" $) }}
>     creationPolicy: {{ .target.creationPolicy | default "Owner" }}
> {{- end }}
> ```
> 通过 `range .Values.externalSecrets` 循环，一个微服务可以声明多个外部密钥源。`creationPolicy: Owner` 意味着 ExternalSecret 被删除时，它创建的 K8s Secret 也会被自动清理。

### 5.3 密钥管理最佳实践

| 做法 | 推荐度 | 说明 |
|------|------|------|
| 硬编码在代码中 | 绝对禁止 | 代码泄露 = 密钥泄露 |
| 环境变量 `.env` + K8s Secret | 不推荐 | base64 非加密，Git 中容易泄露 |
| **ExternalSecret + AWS Secrets Manager** | 推荐 | 集中管理、加密存储、审计日志 |
| **IRSA（IAM Roles for Service Accounts）** | 最佳 | 无密钥！Pod 通过 SA 自动获取 AWS 权限 |

> 本项目使用 IRSA 获取 AWS 权限（见 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|ServiceAccount 配置]]），不在环境变量中传 AK/SK，是最安全的方式。

---

## 六、镜像安全策略（进阶）

### 6.1 准入控制

通过 Admission Webhook 限制只能使用可信镜像仓库的镜像：

```yaml
# 使用 Kyverno 策略引擎示例
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-registries
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Images must be from our private ECR"
        pattern:
          spec:
            containers:
              - image: "236604669925.dkr.ecr.*.amazonaws.com/*"
```

> [!tip] 🔗 实战链接：deploy 项目中的镜像来源控制
> deploy 项目中所有微服务都使用私有 ECR 仓库，镜像地址在 values 文件中统一管理：
> ```yaml
> # deploy 项目：每个微服务的 values 文件
> image:
>   repository: <account-id>.dkr.ecr.us-west-2.amazonaws.com/my-service
>   pullPolicy: IfNotPresent
>   tag: "b0f6af75"       # 使用 commit hash 作为 tag，确保可追溯
> ```
> 所有业务镜像都来自同一个 AWS 账号的 ECR（`<account-id>.dkr.ecr.*.amazonaws.com/*`），基础设施组件则使用 `public.ecr.aws` 的公共镜像。使用 commit hash 而非 `latest` 作为 tag，确保每次部署的镜像可追溯、可回滚。这与 6.1 节中 Kyverno 策略限制镜像来源的思路一致。

### 6.2 镜像安全检查清单

| 检查项 | 工具/方法 | 说明 |
|------|------|------|
| 漏洞扫描 | [[01-docker-basics#9.3 镜像安全扫描|Trivy]] | CI/CD 中扫描，高危漏洞阻断部署 |
| 镜像签名 | cosign (Sigstore) | 确保镜像未被篡改 |
| 基础镜像更新 | Dependabot / Renovate | 自动更新基础镜像版本 |
| 最小权限 | [[01-docker-basics#9.1 基础镜像选择|slim/distroless 镜像]] | 减少攻击面 |
| 非 root 运行 | Dockerfile `USER` + SecurityContext | 即使容器被攻破也限制权限 |

---

## 七、安全检查清单

```
集群级别：
☐ etcd 启用静态加密（EKS 默认开启）
☐ API Server 审计日志开启
☐ RBAC 遵循最小权限原则
☐ aws-auth ConfigMap 限制 IAM 映射

Namespace 级别：
☐ Pod Security Standards 至少 Baseline
☐ NetworkPolicy 默认拒绝，按需开放
☐ ResourceQuota 防止资源滥用
☐ LimitRange 设置默认资源限制

Pod 级别：
☐ 使用 IRSA 而非 AK/SK
☐ Secret 使用 ExternalSecret 管理
☐ 容器以非 root 运行
☐ 只读根文件系统
☐ 禁止特权模式和提权

镜像级别：
☐ CI/CD 中扫描漏洞
☐ 限制镜像来源（准入控制）
☐ 使用最小基础镜像
☐ 定期更新基础镜像
```

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — IRSA、ExternalSecret 的实际配置
- [[05-k8s-architecture|K8s 架构原理]] — API Server 的认证/授权/准入控制链路
- [[04-k8s-networking|K8s 网络深入]] — NetworkPolicy 的实现原理
- [[01-docker-basics|Docker 学习笔记]] — 镜像安全扫描和优化
- [[11-k8s-extension-mechanisms|K8s 扩展机制]] — Admission Webhook 的工作原理
