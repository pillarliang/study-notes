# Helm 与 EKS 集群地址学习笔记

## 一、Helm 是什么

### 1.1 核心概念

Helm 是 Kubernetes 的**包管理工具**，类似于 Linux 中的 `apt/yum`、Node.js 中的 `npm`。它解决的核心问题是：**将一组 Kubernetes YAML 资源打包、模板化、版本化管理**。

核心术语：

| 术语 | 说明 |
|------|------|
| **Chart** | 一个 Helm 包，包含一组模板化的 K8s 资源定义 |
| **Release** | Chart 的一次安装实例（同一个 Chart 可在不同集群/命名空间安装多次） |
| **Values** | 传入 Chart 模板的配置参数，用于定制化部署 |
| **Template** | Go 模板语法编写的 K8s YAML，通过 Values 渲染为最终清单 |
| **Repository** | 存储和分发 Chart 的仓库 |

### 1.2 Helm 解决了什么问题

1. **消除重复 YAML**：同一个微服务部署到多个环境/区域，只需维护一套模板 + 不同 values 文件
2. **版本管理**：Chart 有版本号，可以升级、回滚
3. **依赖管理**：一个 Chart 可以依赖其他 Chart（子 Chart）
4. **参数化配置**：通过 `values.yaml` 注入不同环境的配置（镜像版本、副本数、资源限制等）

### 1.3 Chart 目录结构

```
my-chart/
├── Chart.yaml          # Chart 元数据（名称、版本、依赖）
├── values.yaml         # 默认参数值
├── templates/          # K8s 资源模板
│   ├── _helpers.tpl    # 可复用的模板片段
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── serviceaccount.yaml
└── charts/             # 子 Chart（依赖）
```

### 1.4 关键模板机制

**模板渲染流程**：`Chart 模板` + `Values 参数` → 渲染 → 最终 K8s YAML → 部署到集群

```yaml
# templates/service.yaml 示例
apiVersion: v1
kind: Service
metadata:
  name: {{ include "generic-deployer.fullname" . }}  # 通过 _helpers.tpl 中的函数生成
spec:
  type: {{ .Values.service.type }}
  ports:
    {{- range .Values.service.ports }}
    - port: {{ .port }}
      targetPort: {{ .targetPort | default .port }}
    {{- end }}
```

**fullname 生成逻辑**（`_helpers.tpl`）：

```
1. 如果设置了 fullnameOverride → 直接使用
2. 否则 name = nameOverride || Chart.Name
3. 如果 Release.Name 包含 name → 使用 Release.Name
4. 否则 → Release.Name + "-" + name（截取前 63 字符）
```

### 1.5 子 Chart（依赖）机制

在 `Chart.yaml` 中声明依赖：

```yaml
# plaud-project-summary/Chart.yaml
dependencies:
- name: deployer            # 子 Chart 名称
  version: 0.1.0
  repository: "file://../generic-deployer"  # 引用本地路径
```

父 Chart 的 values 通过子 Chart 名称作为 key 传递：

```yaml
# main.yaml
deployer:                   # 对应子 Chart 名称
  replicaCount: 2
  image:
    repository: 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
    tag: "93347b0"
  service:
    type: ClusterIP
    ports:
      - port: 8001
        name: api
        targetPort: 8001
```

---

## 二、项目中 Helm 的实际用法

### 2.1 generic-deployer：通用部署基座

项目中使用一个共享的 `generic-deployer` Chart 作为所有微服务的基座，提供标准化的：

- Deployment（含滚动更新策略、拓扑分散约束、优雅关停）
- Service
- Ingress（单/多 ingress 支持）
- ServiceAccount（支持 EKS IRSA）
- HPA（水平自动扩缩容）
- ExternalSecret（外部密钥管理）
- ServiceMonitor（Prometheus 监控）
- ConfigMap

### 2.2 项目目录结构

每个微服务项目采用统一的目录结构：

```
plaud-project-summary/
├── Chart.yaml                      # 声明对 generic-deployer 的依赖
├── application.yaml                # ArgoCD 顶层 Application（指向 applicationsets/）
├── applicationsets/
│   ├── applicationsets.yaml        # Staging 环境的 ApplicationSet
│   └── applicationsets-prod.yaml   # Production 环境的 ApplicationSet
└── values/
    ├── ap-northeast-1/
    │   ├── staging/main.yaml
    │   └── prod/
    │       ├── main.yaml
    │       └── preview.yaml
    ├── ap-southeast-1/prod/main.yaml
    ├── eu-central-1/prod/main.yaml
    ├── us-west-2/
    │   ├── staging/main.yaml
    │   └── prod/
    │       ├── main.yaml
    │       └── preview.yaml
    └── cn-northwest-1/
        ├── staging/main.yaml
        └── prod/
            ├── main.yaml
            └── preview.yaml
```

### 2.3 ArgoCD ApplicationSet + Helm 的协作

ArgoCD ApplicationSet 充当"编排层"，将 Helm Chart 分发到多个集群：

```yaml
# plaud-project-summary/applicationsets/applicationsets-prod.yaml
spec:
  generators:
    - list:
        elements:
          - cluster: jp-prod
            server: https://686061FA24...eks.amazonaws.com   # EKS 集群地址
            env: prod
            region: ap-northeast-1
            lane: main
          - cluster: cn-prod
            server: https://D2572EA5B8...eks.amazonaws.com.cn
            env: prod
            region: cn-northwest-1
            lane: main
  template:
    metadata:
      name: "plaud-project-summary-{{ cluster }}-{{ lane }}"   # Helm Release 名
      namespace: plaud-project-summary
    spec:
      source:
        path: plaud-project-summary
        helm:
          valueFiles:
            - values/{{ region }}/{{ env }}/{{ lane }}.yaml    # 注意：不带 values- 前缀
      destination:
        server: "{{ server }}"                 # 部署目标集群
        namespace: plaud-project-summary
```

**部署流程**：

```
ApplicationSet Generator（集群列表）
    ↓ 为每个 element 生成一个 ArgoCD Application
Application（per cluster + lane）
    ↓ 使用 Helm 渲染
Helm Chart (generic-deployer) + Values 文件
    ↓ 渲染出 K8s 资源
部署到对应 EKS 集群的目标 namespace
```

### 2.4 集群内服务地址（K8s Service DNS）

最终在集群内生成的服务名格式为：

```
<release-name>-deployer.<namespace>:<port>
```

即：

```
plaud-project-summary-<cluster>-<lane>-deployer.plaud-project-summary:<port>
```

示例：

- `plaud-project-summary-jp-prod-main-deployer.plaud-project-summary:8001`
- `plaud-project-summary-jp-prod-preview-deployer.plaud-project-summary:8001`
- `plaud-project-summary-cn-prod-main-deployer.plaud-project-summary:8001`

---

## 三、实战示例：plaud-project-summary 项目全解析

以 `plaud-project-summary`（摘要服务）为例，完整拆解一个微服务从 Chart 定义到多集群部署的全过程。

### 3.1 项目目录结构

```
plaud-project-summary/
├── Chart.yaml                                    # ① Chart 元信息 + 依赖声明
├── application.yaml                              # ② ArgoCD 顶层 Application
├── applicationsets/
│   ├── applicationsets.yaml                      # ③ Staging ApplicationSet
│   └── applicationsets-prod.yaml                 # ④ Production ApplicationSet
└── values/
    ├── ap-northeast-1/
    │   ├── staging/main.yaml                     # ⑤ JP Staging 配置
    │   └── prod/
    │       ├── main.yaml                         # ⑥ JP Prod 配置
    │       └── preview.yaml                      # ⑦ JP Prod Preview 配置
    ├── us-west-2/
    │   ├── staging/main.yaml                     # US Staging
    │   └── prod/
    │       ├── main.yaml                         # US Prod
    │       └── preview.yaml                      # US Prod Preview
    ├── ap-southeast-1/prod/main.yaml             # SG Prod
    ├── eu-central-1/prod/main.yaml               # EU Prod
    └── cn-northwest-1/
        ├── staging/main.yaml                     # CN Staging
        └── prod/
            ├── main.yaml                         # CN Prod
            └── preview.yaml                      # CN Prod Preview
```

### 3.2 ① Chart.yaml — 声明依赖

```yaml
apiVersion: v2
name: plaud-project-summary      # Chart 名称，也是默认的 K8s 资源名前缀
type: application
version: 0.1.0

dependencies:
- name: deployer                          # 子 Chart 名称（values 中以此为 key）
  version: 0.1.0
  repository: "file://../generic-deployer" # 引用本地的 generic-deployer
```

> 所有项目的 Chart.yaml 几乎一模一样，区别只有 `name` 字段。

### 3.3 ② application.yaml — ArgoCD 入口

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: plaud-project-summary           # 父应用名
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc    # 部署到 ArgoCD 所在集群
    namespace: plaud-project-summary
  project: plaud
  source:
    repoURL: git@github.com:Plaud-AI/deploy.git
    targetRevision: HEAD
    path: plaud-project-summary/applicationsets       # 指向 applicationsets 目录
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
```

> 这个 Application 的作用是让 ArgoCD 发现 `applicationsets/` 目录下的 ApplicationSet 资源。

### 3.4 ③④ ApplicationSet — 多集群编排

**Staging（applicationsets.yaml）**：

```yaml
spec:
  generators:
    - list:
        elements:
          - cluster: jp-staging                          # 用于生成 ArgoCD Application 名
            server: https://0AE3C052...eks.amazonaws.com # JP Staging 集群
            env: staging
            region: ap-northeast-1
            lane: main
          - cluster: us-staging
            server: https://C85149DE...eks.amazonaws.com # US Staging 集群
            env: staging
            region: us-west-2
            lane: main
          - cluster: cn-staging
            server: https://4F8B0222...eks.amazonaws.com.cn  # CN Staging 集群
            env: staging
            region: cn-northwest-1
            lane: main
  template:
    metadata:
      name: "plaud-project-summary-{{ cluster }}-{{ lane }}"  # → plaud-project-summary-jp-staging-main
    spec:
      source:
        targetRevision: dev                           # Staging 用 dev 分支
        helm:
          valueFiles:
            - values/{{ region }}/{{ env }}/{{ lane }}.yaml
      syncPolicy:
        automated:
          selfHeal: true         # Staging 开启自愈（自动同步）
```

**Production（applicationsets-prod.yaml）**：

```yaml
spec:
  generators:
    - list:
        elements:
          - cluster: jp-prod, lane: main       # → plaud-project-summary-jp-prod-main-deployer
          - cluster: jp-prod, lane: preview    # → Preview 环境
          - cluster: eu-prod, lane: main       # → EU 区域
          - cluster: sg-prod, lane: main       # → 新加坡区域
          - cluster: us-prod, lane: main       # → US 区域
          - cluster: us-prod, lane: preview    # → US Preview
          - cluster: cn-prod, lane: main       # → 中国区域
          - cluster: cn-prod, lane: preview    # → 中国 Preview
  template:
    spec:
      source:
        targetRevision: main                          # Prod 用 main 分支
      syncPolicy:
        # automated 被注释掉 → Prod 需要手动同步
```

**Staging vs Production 关键差异**：

| 配置项 | Staging | Production |
|--------|---------|------------|
| targetRevision | `dev` | `main` |
| 自动同步 | `selfHeal: true` | 手动（注释掉了 automated） |
| 集群数量 | 3（JP/US/CN） | 8（JP/EU/SG/US/CN × main+preview） |

### 3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明

以 JP Prod（`values/ap-northeast-1/prod/main.yaml`）为例，逐块解读：

#### 镜像配置（image）

```yaml
deployer:                    # 所有配置都在 deployer: 下（对应子 Chart 名）
  image:
    repository: 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
    pullPolicy: IfNotPresent    # 仅当本地没有时才拉取
    tag: "93347b0"              # 镜像版本（通常是 git commit hash）
```

> **更新部署 = 修改 tag 值**，这是日常最频繁的操作。

#### 资源限制（resources）

```yaml
  resources:
    limits:                     # 容器资源上限
      cpu: 8                    # 8 核 CPU
      memory: 16Gi              # 16GB 内存
    requests:                   # 调度时的最低保障
      cpu: 8
      memory: 16Gi
```

> `requests = limits` 表示 **Guaranteed QoS**，Pod 不会被驱逐。

**不同环境的资源对比**：

| 环境 | CPU | Memory | 说明 |
|------|-----|--------|------|
| JP Staging | 1 | 4Gi | 测试环境，资源节省 |
| JP Prod | 8 | 16Gi | 生产环境，高配 |
| CN Prod | 8 | 16Gi | 中国区生产 |

#### 自动扩缩容（autoscaling / HPA）

```yaml
  autoscaling:
    enabled: true              # 是否开启 HPA
    minReplicas: 2             # 最小副本数
    maxReplicas: 10            # 最大副本数
    targetCPUUtilizationPercentage: 75   # CPU 使用率达到 75% 时扩容
```

> Staging 环境通常不开启 HPA（`enabled: false`），Prod 环境按需开启。

#### 健康检查（Probes）

```yaml
  livenessProbe:               # 存活探针：失败则重启容器
    httpGet:
      path: /health            # 健康检查路径
      port: 8000               # 独立的健康检查端口
  readinessProbe:              # 就绪探针：失败则从 Service 摘除流量
    httpGet:
      path: /health
      port: 8000
```

> plaud-project-summary 使用独立的 8000 端口提供健康检查，与业务端口（8001）分离。

#### 优雅关停（Graceful Shutdown + preStop Hook）

```yaml
  terminationGracePeriodSeconds: 600   # Pod 终止前最多等待 600 秒（10 分钟）

  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "echo preStop && sleep 10"]
```

> `preStop` 在 SIGTERM 发送前执行，为 Endpoints 摘除传播争取 10 秒窗口，避免流量在传播窗口内发送到即将终止的 Pod。

#### 卷挂载（Volumes & VolumeMounts）

```yaml
  volumeMounts:                            # 容器内的挂载点
    - name: logs
      mountPath: /data/plaud-sync/logs     # 日志目录
  volumes:                                 # 卷定义
    - name: logs
      emptyDir: {}                         # 临时卷，Pod 销毁后丢失
```

> plaud-project-summary 使用 IRSA 获取 AWS 权限，无需挂载额外的 Secret 卷。

#### 环境变量（env）

```yaml
  env:
    # === 基础环境标识 ===
    - name: AWS_ENV
      value: prod
    - name: AWS_REGION
      value: ap-northeast-1
    - name: AWS_DEFAULT_REGION        # Boto3 默认区域
      value: ap-northeast-1

    # === AppConfig 动态配置（多配置文件） ===
    - name: APPCONFIG_SERVICE_NAME
      value: plaud-project-summary
    - name: APPCONFIG_RUN_ENV
      value: prod
    - name: APPCONFIG_CONFIG_FILE_NAME
      value: prod-config.yaml              # 主配置文件
    - name: APPCONFIG_MODEL_CONFIG_FILE
      value: prod-model-config.yaml        # 模型配置文件
    - name: APPCONFIG_GEMINI_KEY_FILE
      value: prod-gemini-key.json          # Gemini API Key
    - name: APPCONFIG_STRATEGY_CONFIG_FILE
      value: prod-config-strategy.json     # 策略配置文件
    - name: APPCONFIG_AWS_REGION
      value: ap-northeast-1               # AppConfig 所在区域
```

> 注意：plaud-project-summary **纯使用 IRSA**，不在 env 中传入 AK/SK，更安全。

#### ServiceAccount（IRSA）

```yaml
  serviceAccount:
    create: true                # 是否创建 ServiceAccount
    automount: true
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::408278014848:role/plaud-project-summary-role
    name: plaud-project-summary
```

> **IRSA**（IAM Roles for Service Accounts）：Pod 通过 ServiceAccount 自动获得 AWS IAM 权限，无需在环境变量中传 AK/SK。

#### Service 配置（多端口）

```yaml
  service:
    type: ClusterIP             # 仅集群内可访问（最常用）
    ports:
      - port: 8001              # API 后端
        name: api
        targetPort: 8001
      - port: 8889              # NiceGUI 前端
        name: frontend
        targetPort: 8889
      - port: 8000              # 健康检查
        name: health
        targetPort: 8000
```

> 集群内其他 Pod 访问 API：`plaud-project-summary-jp-prod-main-deployer.plaud-project-summary:8001`

#### Ingress 配置（多 Ingress + 高级特性）

plaud-project-summary 使用多 Ingress 配置，按流量类型分离：

```yaml
  ingresses:
    # 内网 - Web 前端（NiceGUI 有状态框架，需 cookie 亲和性）
    internal:
      ingressClassName: "nginx-internal"
      annotations:
        nginx.ingress.kubernetes.io/proxy-body-size: "150m"
        nginx.ingress.kubernetes.io/affinity: "cookie"              # 会话亲和性
        nginx.ingress.kubernetes.io/session-cookie-name: "NICEGUI_AFFINITY"
        nginx.ingress.kubernetes.io/session-cookie-expires: "3600"
      hosts:
        - host: plaud-project-summary-apne1.nicebuild.click
          paths:
            - path: /                    # 前端页面
              servicePort: 8889
            - path: /api                 # API 请求
              servicePort: 8001

    # 私有 - API 后端（内网访问）
    private:
      ingressClassName: "nginx-pvt"
      hosts:
        - host: plaud-project-summary-apne1-lan.plaud.ai
          paths:
            - path: /api/temporal
              servicePort: 8001

    # 公网 - API
    public:
      ingressClassName: "nginx-public"
      hosts:
        - host: plaud-project-summary-apne1.plaud.ai
          paths:
            - path: /api/strategy
              servicePort: 8001

    # 公网 - API 网关转发（URL 重写）
    api-gateway:
      ingressClassName: "nginx-public"
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /$2       # URL 重写规则
        nginx.ingress.kubernetes.io/use-regex: "true"
      hosts:
        - host: api-apne1.plaud.ai
          paths:
            - path: /project-summary(/|$)(.*)                 # 前缀路由
              pathType: ImplementationSpecific
              servicePort: 8001
```

> 4 种 Ingress Controller 对应不同网络层级：`nginx-internal`（内网）、`nginx-pvt`（私有）、`nginx-public`（公网）、API 网关转发。

### 3.6 Staging vs Prod Values 对比

| 参数 | Staging (JP) | Prod (JP) | 说明 |
|------|-------------|-----------|------|
| `image.tag` | `60bad01` | `93347b0` | 不同版本 |
| `resources.cpu` | 1 | 8 | Prod 资源更多 |
| `resources.memory` | 4Gi | 16Gi | Prod 内存更大 |
| `autoscaling.enabled` | false | true (2-10) | Prod 开启 HPA |
| `env.AWS_ENV` | test | prod | 环境标识 |
| `env.APPCONFIG_RUN_ENV` | dev | prod | 配置文件环境 |
| `APPCONFIG_CONFIG_FILE_NAME` | dev-config.yaml | prod-config.yaml | 不同配置文件 |
| SA IAM Account | 734110488307 | 408278014848 | 不同 AWS 账号 |
| Ingress 域名 | `*-staging-apne1.*` | `*-apne1.*` | 域名前缀区分 |

### 3.7 中国区 vs 海外区差异

| 参数 | 海外区 (JP/EU/SG/US) | 中国区 (CN) |
|------|---------------------|-------------|
| ECR 地址 | `236604669925.dkr.ecr.us-west-2.amazonaws.com` | `470515048733.dkr.ecr.cn-northwest-1.amazonaws.com.cn` |
| IAM ARN | `arn:aws:iam::408278014848:role/...` | `arn:aws-cn:iam::470515048733:role/...` |
| 域名 | `*.plaud.ai` / `*.nicebuild.click` | `*.plaud.cn` / `*.nicebuild.cn` |
| EKS 域名后缀 | `.eks.amazonaws.com` | `.eks.amazonaws.com.cn` |
| HPA CPU 阈值 | 75% | 80% |

### 3.8 完整部署链路图

```
① Chart.yaml
   声明依赖 generic-deployer
        ↓
② application.yaml
   ArgoCD 读取 → 发现 applicationsets/ 目录
        ↓
③ applicationsets-prod.yaml
   Generator 列表定义 8 个 element:
   [jp-prod main/preview, eu-prod main, sg-prod main,
    us-prod main/preview, cn-prod main/preview]
        ↓ 为每个 element 生成
④ ArgoCD Application × 8
   每个 Application:
   - Release Name = plaud-project-summary-{{ cluster }}-{{ lane }}
   - 使用 values/{{ region }}/{{ env }}/{{ lane }}.yaml
   - 部署到 server: {{ server }}
        ↓ Helm 渲染
⑤ K8s 资源
   - Deployment: plaud-project-summary-jp-prod-main-deployer
   - Service:    plaud-project-summary-jp-prod-main-deployer
                 (ClusterIP: 8001/api, 8889/frontend, 8000/health)
   - HPA:        min=2, max=10, CPU=75%
   - SA:         plaud-project-summary (with IRSA)
   - Ingress:    internal / private / public / api-gateway
        ↓ 部署到
⑥ EKS 集群 (ap-northeast-1)
   namespace: plaud-project-summary
```

---

## 四、EKS 集群地址

### 4.1 什么是 EKS 集群地址

EKS（Elastic Kubernetes Service）集群地址是 AWS 托管的 Kubernetes API Server 的 endpoint，格式为：

```
https://<UNIQUE-HASH>.<SUFFIX>.<REGION>.eks.amazonaws.com
```

- `UNIQUE-HASH`：集群唯一标识符
- `SUFFIX`：AWS 内部路由标识（如 `gr7`、`yl4`、`sk1`）
- `REGION`：AWS 区域
- 中国区域后缀为 `.eks.amazonaws.com.cn`

### 4.2 如何查看 EKS 集群地址

#### 方法一：AWS CLI

```bash
# 列出当前 region 的所有 EKS 集群
aws eks list-clusters --region <region>

# 查看指定集群的详细信息（含 endpoint）
aws eks describe-cluster --name <cluster-name> --region <region>

# 仅提取 endpoint 地址
aws eks describe-cluster --name <cluster-name> --region <region> \
  --query "cluster.endpoint" --output text
```

示例输出：

```
https://3F1BDD3214F9F0383D5FD4CC62CDE658.sk1.us-west-2.eks.amazonaws.com
```

#### 方法二：AWS 管理控制台

1. 登录 AWS Console → 进入 **EKS** 服务
2. 选择对应的 Region
3. 点击集群名称 → **概览 (Overview)** 页面
4. 查看 **API server endpoint** 字段

#### 方法三：kubectl 配置

```bash
# 查看当前 kubeconfig 中配置的集群地址
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'

# 查看所有 context 及对应的集群
kubectl config get-contexts
```

#### 方法四：ArgoCD（本项目实际使用的方式）

在本项目中，EKS 地址直接写在 ArgoCD ApplicationSet 的 `server` 字段中：

```bash
# 查看已注册的集群列表
argocd cluster list

# 查看指定集群详情
argocd cluster get <server-url>
```

也可以直接查看代码仓库中的 applicationsets 文件：

```bash
# 查看某个项目的集群配置
cat plaud-project-summary/applicationsets/applicationsets-prod.yaml
# 在 generators.list.elements 中找到 server 字段
```

#### 方法五：Terraform / IaC

如果集群通过 Terraform 创建，可以查看 state：

```bash
terraform output cluster_endpoint
```

### 4.3 集群地址 vs 集群内服务地址

| 对比 | EKS 集群地址 | K8s Service DNS |
|------|-------------|----------------|
| 用途 | 外部访问 K8s API Server | 集群内 Pod 间通信 |
| 格式 | `https://<hash>.<region>.eks.amazonaws.com` | `<svc>.<ns>.svc.cluster.local:<port>` |
| 谁使用 | kubectl / ArgoCD / CI/CD | 集群内的其他 Pod |
| 认证 | 需要 kubeconfig / IAM | 无需额外认证（同集群内） |

---

## 五、常用 Helm 命令速查

```bash
# 查看 Chart 信息
helm show chart <chart-path>
helm show values <chart-path>

# 模板渲染（不实际部署，仅查看生成的 YAML）
helm template <release-name> <chart-path> -f values.yaml

# 安装 / 升级
helm install <release-name> <chart-path> -f values.yaml -n <namespace>
helm upgrade <release-name> <chart-path> -f values.yaml -n <namespace>

# 查看已安装的 release
helm list -n <namespace>

# 查看 release 历史
helm history <release-name> -n <namespace>

# 回滚
helm rollback <release-name> <revision> -n <namespace>

# 卸载
helm uninstall <release-name> -n <namespace>

# 依赖管理
helm dependency update <chart-path>   # 下载/更新子 Chart
helm dependency list <chart-path>     # 查看依赖列表
```

---

## 六、总结

```
Helm 核心作用：模板化 K8s 资源 + 参数化多环境配置 + 版本管理
                  ↓
generic-deployer：统一的部署基座 Chart（Deployment/Service/Ingress/HPA...）
                  ↓
各微服务 Chart：  依赖 generic-deployer，提供项目特定的 values 文件
                  ↓
ArgoCD ApplicationSet：编排层，将 Helm Chart × Values 分发到多个 EKS 集群
                  ↓
EKS 集群：        最终运行 K8s 资源的目标环境
```
