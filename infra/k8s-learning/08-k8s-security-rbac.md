# K8s 安全与 RBAC

> 前置知识：[[05-k8s-architecture#3.1 API Server — 唯一的入口|API Server 的认证/授权流程]]、[[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|ServiceAccount（IRSA）的基本用法]]。

---

## 一、安全的四个层面

**问题**：一个 HTTP 请求到达 API Server 后，K8s 如何决定是否执行？

答案是四道关卡，依次通过才能放行：

```txt
请求 → API Server 处理链路

┌──────────────────────────────────────────────────────────────┐
│  ① Authentication（认证）：请求者是谁？                         │
│     证书 / Token / OIDC / ServiceAccount Token                │
├──────────────────────────────────────────────────────────────┤
│  ② Authorization（授权）：此身份是否有权执行该操作？              │
│     RBAC：Role + RoleBinding → 允许 / 拒绝                    │
├──────────────────────────────────────────────────────────────┤
│  ③ Admission Control（准入控制）：请求内容是否合规？              │
│     Mutating Webhook → Validating Webhook                     │
├──────────────────────────────────────────────────────────────┤
│  ④ Runtime Security（运行时安全）：Pod 运行时拥有哪些权限？       │
│     Pod Security Standards / SecurityContext                   │
└──────────────────────────────────────────────────────────────┘
```

后文按"授权 → 身份 → 运行时 → Secret 管理"的顺序展开，以 `plaud-project-summary` 的 IRSA 和 ExternalSecret 为主线。

---

## 二、RBAC——谁能做什么

### 2.1 核心模型

RBAC 回答一个三元组问题：**谁（Subject）** 可以对 **什么资源（Resource）** 做 **什么操作（Verb）**。

```txt
Subject（谁）             RoleBinding（绑定关系）        Role（权限定义）
┌─────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│ User（用户）     │      │                  │      │ resources: pods  │
│ Group（组）      │←────→│  将 Subject      │←────→│ verbs: get, list │
│ ServiceAccount  │      │  绑定到 Role      │      │                  │
└─────────────────┘      └──────────────────┘      └──────────────────┘
```

**四种 RBAC 资源**：

| 资源 | 作用范围 | 说明 |
| --- | --- | --- |
| **Role** | 单个 Namespace | 定义"可以对哪些资源做哪些操作" |
| **ClusterRole** | 整个集群 | 同上，但跨 Namespace 或针对集群级资源 |
| **RoleBinding** | 单个 Namespace | 将 Role/ClusterRole 绑定到 Subject |
| **ClusterRoleBinding** | 整个集群 | 将 ClusterRole 绑定到 Subject（集群范围） |

### 2.2 Role + RoleBinding 示例

```yaml
# 允许在 production namespace 中读取 Pod 和日志
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]            # 核心 API 组（Pod、Service 等）
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]    # Pod 的子资源
    verbs: ["get"]
---
# 将 pod-reader 角色绑定给开发团队和 CI SA
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
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**常见 Verbs**：

| Verb | 操作 | HTTP 方法 |
| --- | --- | --- |
| `get` | 获取单个资源 | GET |
| `list` | 列出资源集合 | GET |
| `watch` | 监听变更 | GET（watch） |
| `create` | 创建 | POST |
| `update` / `patch` | 更新 | PUT / PATCH |
| `delete` | 删除 | DELETE |

### 2.3 EKS 中的 RBAC 映射（进阶）

EKS 通过 `aws-auth` ConfigMap 将 AWS IAM 身份映射到 K8s RBAC：

```txt
AWS IAM User/Role
    ↓ aws-auth ConfigMap 映射
K8s User/Group
    ↓ RoleBinding / ClusterRoleBinding
K8s Role / ClusterRole
    ↓
具体权限
```

```yaml
# 来源：kube-system/aws-auth ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456:role/EKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups: ["system:nodes"]
    - rolearn: arn:aws:iam::123456:role/DevTeamRole
      username: dev-user
      groups: ["dev-team"]          # 对应 RoleBinding 中的 Group
```

### 2.4 RBAC 排障命令（深入）

```bash
# 检查当前用户权限
kubectl auth can-i create deployments -n production

# 列出所有权限
kubectl auth can-i --list -n production

# 检查特定 ServiceAccount 的权限
kubectl auth can-i get pods -n production \
  --as=system:serviceaccount:production:ci-deployer

# 查看 RoleBinding
kubectl get rolebindings -n production
kubectl describe rolebinding dev-pod-reader -n production
```

---

## 三、ServiceAccount 与 IRSA

### 3.1 问题：Pod 如何访问 AWS 资源？

一个运行在 EKS 上的 Pod 需要读写 S3、Secrets Manager 等 AWS 资源。传统做法是把 AK/SK 塞进环境变量——但静态密钥一旦泄露，影响面不可控。

**IRSA（IAM Roles for Service Accounts）** 解决了这个问题：让 K8s ServiceAccount 与 AWS IAM Role 关联，Pod 启动时通过 OIDC 自动注入临时凭证，无需存储任何密钥。

```txt
K8s ServiceAccount（注解 IAM Role ARN）
    ↓ EKS OIDC Provider 验证
AWS STS 签发临时凭证（自动注入 Pod 环境变量）
    ↓
Pod 使用临时凭证访问 AWS 资源（S3 / Secrets Manager / ...）
```

核心优势：**K8s RBAC 控制集群内权限，IRSA 控制 AWS 云资源权限**，两者互补。

### 3.2 实战：plaud-project-summary 的 IRSA 配置

```yaml
# 来源：plaud-project-summary/values/ap-northeast-1/prod/main.yaml
serviceAccount:
  create: true
  automount: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::041326371978:role/plaud-project-summary-role
  name: plaud-project-summary
```

关键注解 `eks.amazonaws.com/role-arn` 将 ServiceAccount 与 IAM Role 绑定。Pod 启动后，AWS 自动注入临时凭证，应用代码无需任何改动即可调用 AWS SDK。

> [!note] 中国区 ARN 格式不同
> ```yaml
> # 来源：plaud-project-summary/values/cn-northwest-1/prod/main.yaml
> annotations:
>   eks.amazonaws.com/role-arn: arn:aws-cn:iam::<account-id>:role/plaud-project-summary-role
> ```

staging 环境中已注释掉的旧方案体现了团队的演进——从静态密钥迁移到 IRSA：

```yaml
# 来源：plaud-project-summary/values/ap-northeast-1/staging/main.yaml（已废弃）
# - name: AWS_ACCESS_KEY_ID
#   valueFrom:
#     secretKeyRef:
#       name: plaud-project-summaryc-aws
#       key: AWS_ACCESS_KEY_ID
```

### 3.3 generic-deployer 中的 SA 模板

deploy 项目通过 Helm 模板化 ServiceAccount 的创建，各微服务在 values 中按需配置：

```yaml
# 来源：generic-deployer/templates/serviceaccount.yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.serviceAccount.name }}
  labels:
    {{ .Values.serviceAccount.labels }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
automountServiceAccountToken: {{ .Values.serviceAccount.automount }}
{{- end }}
```

`serviceAccount.create: true` 决定是否创建 SA，`annotations` 承载 IRSA Role ARN，`automount` 控制是否将 SA Token 自动挂载到 Pod。

---

## 四、Pod Security——限制运行时权限（进阶）

### 4.1 问题：容器默认能做什么？

未加约束时，容器可以：以 root 运行、挂载宿主机文件系统、使用特权模式、访问宿主机网络。一旦容器被攻破，攻击者等同于拿到了节点权限。

**Pod Security Standards** 定义三个安全级别：

| 级别 | 限制程度 | 适用场景 |
| --- | --- | --- |
| **Privileged** | 无限制 | 系统组件（CNI、CSI 驱动） |
| **Baseline** | 禁止已知危险配置 | 大多数应用（推荐最低标准） |
| **Restricted** | 最严格 | 安全敏感的生产负载 |

Baseline 级别禁止的典型配置：

```yaml
# 以下在 Baseline 级别下均被拒绝
spec:
  hostNetwork: true          # 使用节点网络
  hostPID: true              # 共享节点 PID namespace
  containers:
    - securityContext:
        privileged: true     # 特权模式
        capabilities:
          add: ["SYS_ADMIN"] # 危险的 Linux capability
```

### 4.2 SecurityContext 实战：三种模式

#### 模式一：默认留白（generic-deployer 模板）

```yaml
# 来源：generic-deployer/values.yaml
podSecurityContext: {}
securityContext: {}
```

默认值为空，由各服务按需覆盖。模板中 `{{- with .Values.securityContext }}` 的写法意味着不配置就不渲染。

#### 模式二：需要特权的 sidecar

```yaml
# 来源：my-gateway 的 GoReplay sidecar
extraContainers:
  - name: goreplay-sidecar
    image: buger/goreplay:latest
    securityContext:
      runAsUser: 0
      capabilities:
        add: ["NET_RAW", "NET_ADMIN"]   # 网络抓包所需
```

GoReplay 需要原始网络包捕获能力，属于 Baseline 禁止的配置——只有确实需要特权的系统级工具才应如此配置。

#### 模式三：需要 root 但限制提权

```yaml
# 来源：my-gateway 的 nginx 静态文件服务
podSecurityContext:
  runAsUser: 0
  runAsGroup: 0
  allowPrivilegeEscalation: false   # 即使 root 也禁止提权
```

### 4.3 Pod Security Admission（PSA）（进阶）

PSA 是 K8s 内置的准入控制器，在 Namespace 级别强制执行安全标准：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: baseline     # 不符合则拒绝
    pod-security.kubernetes.io/warn: restricted      # 不符合则警告
    pod-security.kubernetes.io/audit: restricted     # 记录审计日志
```

| 模式 | 行为 |
| --- | --- |
| `enforce` | 拒绝不符合的 Pod |
| `warn` | 允许但 kubectl 输出警告 |
| `audit` | 允许但记录审计日志 |

### 4.4 推荐的 SecurityContext 配置（进阶）

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

---

## 五、Secret 管理——ExternalSecret

### 5.1 问题：K8s 原生 Secret 够安全吗？

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=       # echo cG9zdGdyZXM= | base64 -d → postgres
  password: bXlwYXNzd29yZA==
```

**K8s Secret 的局限**：

- **base64 编码 ≠ 加密**——任何人拿到 YAML 就能解码
- 存储在 etcd 中，需要启用静态加密（EKS 默认开启）
- 提交到 Git 仓库等同于明文泄露

生产环境需要外部密钥管理系统。

### 5.2 ExternalSecret 架构

ExternalSecret Operator（ESO）从外部密钥管理系统（如 AWS Secrets Manager）自动同步密钥到 K8s Secret：

```txt
AWS Secrets Manager（加密存储，IAM 权限控制）
    ↓ ESO 定期同步（refreshInterval）
K8s Secret（自动创建 / 更新）
    ↓ Pod 挂载
容器环境变量 或 文件
```

整个链路分为两层：

#### 第一层：ClusterSecretStore（集群级，所有 Namespace 共享）

```yaml
# 来源：infra/values/external-secrets/.../aws-secrets-manager-store.yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets       # ESO 自身也通过 IRSA 获取 AWS 权限
            namespace: external-secrets
```

#### 第二层：ExternalSecret（应用级，声明需要哪些密钥）

```yaml
# 来源：generic-deployer/templates/externalsecret.yaml（简化）
{{- range .Values.externalSecrets }}
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: {{ .name | default (include "generic-deployer.fullname" $) }}
spec:
  refreshInterval: {{ .refreshInterval | default "1h" }}
  secretStoreRef:
    name: {{ .secretStoreRef.name }}
    kind: {{ .secretStoreRef.kind }}
  target:
    name: {{ .target.name | default (include "generic-deployer.fullname" $) }}
    creationPolicy: {{ .target.creationPolicy | default "Owner" }}
{{- end }}
```

`creationPolicy: Owner` 表示 ExternalSecret 被删除时，自动清理关联的 K8s Secret。

### 5.3 实战：plaud-project-summary 从 AWS Secrets Manager 同步密钥

`plaud-project-summary` 通过环境变量告诉应用去 AWS Secrets Manager 获取 Gemini API 密钥：

```yaml
# 来源：plaud-project-summary/values/ap-northeast-1/prod/main.yaml
env:
  - name: SECRETS_MANAGER_ENABLED
    value: "true"
  - name: SECRETS_MANAGER_GEMINI_KEY
    value: prod/project-summary/gemini-key     # Secrets Manager 中的路径
```

staging 环境使用不同路径（`dev/project-summary/gemini-key`），实现环境隔离。密钥本身存储在 AWS Secrets Manager 中加密管理，**从未出现在 Git 仓库中**——应用通过 IRSA 获取的 IAM 权限来读取。

### 5.4 密钥管理方案对比

| 方案 | 安全等级 | 说明 |
| --- | --- | --- |
| 硬编码在代码中 | 绝对禁止 | 代码泄露 = 密钥泄露 |
| K8s Secret + `.env` | 不推荐 | base64 非加密，Git 中容易泄露 |
| **ExternalSecret + Secrets Manager** | 推荐 | 集中管理、加密存储、审计日志 |
| **IRSA（无密钥）** | 最佳 | Pod 通过 SA 自动获取 AWS 临时凭证 |

---

## 六、安全检查清单

```txt
集群级别：
☐ etcd 启用静态加密（EKS 默认开启）
☐ API Server 审计日志开启
☐ RBAC 遵循最小权限原则
☐ aws-auth ConfigMap 限制 IAM 映射

Namespace 级别：
☐ Pod Security Standards 至少 Baseline
☐ NetworkPolicy 默认拒绝，按需开放
☐ ResourceQuota / LimitRange 防止资源滥用

Pod 级别：
☐ 使用 IRSA 而非 AK/SK
☐ Secret 通过 ExternalSecret 管理
☐ 容器以非 root 运行（runAsNonRoot: true）
☐ 只读根文件系统（readOnlyRootFilesystem: true）
☐ 禁止特权模式和提权

镜像级别：
☐ CI/CD 中扫描漏洞（Trivy）
☐ 准入控制限制镜像来源（Kyverno）
☐ 使用最小基础镜像（distroless / slim）
☐ 使用 commit hash 作为 tag，禁止 latest
```

---

## 延伸阅读

- [[05-k8s-architecture|K8s 架构原理]] — API Server 的认证/授权/准入控制链路
- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — IRSA、ExternalSecret 的实际配置
- [[04-k8s-networking#六、NetworkPolicy — 网络隔离|K8s 网络]] — NetworkPolicy 实现零信任网络
- [[11-k8s-extension-mechanisms|K8s 扩展机制]] — Admission Webhook 的工作原理
