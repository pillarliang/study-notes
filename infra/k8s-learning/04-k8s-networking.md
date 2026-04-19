# K8s 网络深入

> 前置知识：了解 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|Service 和 Ingress 的基本用法]]。本文深入解释 K8s 网络的实现原理——为什么 Pod 之间能通信、Service DNS 怎么工作、如何做网络隔离。
>
> 本文涉及的组件在 [[05-k8s-architecture|K8s 架构]]中有整体定位。
>
> **学习建议**：首次阅读重点关注各节概念部分（网络三条规则、CNI 概念、Service 四种类型、CoreDNS 基础、Ingress 本质、NetworkPolicy 基本示例、排障速查）。标记为（进阶）的小节掌握基础后再看，第七节（Service Mesh）为深入。

---

## 一、K8s 网络模型的三条规则

K8s 定义了一个扁平的网络模型，所有实现（CNI 插件）都必须满足：

1. **每个 Pod 有自己的 IP 地址**（不是共享节点 IP）
2. **所有 Pod 之间可以直接通信**（不需要 NAT）
3. **Pod 看到的自己的 IP = 其他 Pod 看到的 IP**（没有隐藏的地址转换）

```
Node-1 (10.0.1.10)              Node-2 (10.0.2.10)
┌─────────────────┐             ┌─────────────────┐
│  Pod-A: 10.0.1.5│── 直接通信 ──│  Pod-C: 10.0.2.5│
│  Pod-B: 10.0.1.6│             │  Pod-D: 10.0.2.6│
└─────────────────┘             └─────────────────┘

Pod-A 可以直接访问 10.0.2.5:8001 到达 Pod-C，即使在不同节点上。
```

> 这和 Docker 默认的 bridge 网络不同——Docker 容器需要端口映射（`-p`）才能被外部访问，而 K8s Pod 天生就有可路由的 IP。

---

## 二、CNI（Container Network Interface）

### 2.1 CNI 是什么

CNI 是 K8s 定义的网络插件接口。kubelet 创建 Pod 时，调用 CNI 插件为 Pod 分配 IP 和配置网络：

```
kubelet 创建 Pod
    ↓ 调用 CNI 插件
CNI 插件执行：
    ├── 创建 veth pair（虚拟网络接口对）
    ├── 一端放入 Pod 的 network namespace
    ├── 另一端连接到节点的网络
    ├── 分配 IP 地址
    └── 配置路由规则
```

### 2.2 AWS VPC CNI（EKS 默认）（进阶）

AWS VPC CNI 是 EKS 的默认网络插件，它的独特之处是：**每个 Pod 直接获得 VPC 子网中的真实 IP**。

```
实现原理：
1. 每个 EC2 节点有多个 ENI（Elastic Network Interface）
2. 每个 ENI 可以有多个 Secondary IP
3. Pod 的 IP = ENI 的某个 Secondary IP

EC2 实例 (m5.large)
├── ENI-1 (主网卡): 10.0.1.10 (节点 IP)
│   ├── Secondary IP: 10.0.1.11 → Pod-A
│   ├── Secondary IP: 10.0.1.12 → Pod-B
│   └── Secondary IP: 10.0.1.13 → Pod-C
└── ENI-2 (附加网卡)
    ├── Secondary IP: 10.0.1.20 → Pod-D
    └── Secondary IP: 10.0.1.21 → Pod-E
```

**单节点 Pod 数量限制**：

| 实例类型 | 最大 ENI 数 | 每 ENI 最大 IP 数 | 最大 Pod 数 |
|------|------|------|------|
| t3.medium | 3 | 6 | 17 |
| m5.large | 3 | 10 | 29 |
| m5.xlarge | 4 | 15 | 58 |
| m5.4xlarge | 8 | 30 | 234 |

> 这是 EKS 的一个重要限制：小实例类型能跑的 Pod 数有限。如果 Pod 数量超出，Scheduler 会因为"IP 不够"而无法调度，节点状态显示 `Too many pods`。

**VPC CNI 的优势**：
- Pod IP 是真实的 VPC IP，可以直接被 SecurityGroup、ALB、RDS 等 AWS 服务识别
- 不需要 overlay 网络，性能高（没有封装/解封装开销）
- Pod 可以直接关联 SecurityGroup（Pod 级别的网络隔离）

### 2.3 其他常见 CNI 插件

| CNI 插件 | 特点 | 适用场景 |
|------|------|------|
| **AWS VPC CNI** | Pod 用 VPC 真实 IP，性能最好 | EKS（默认） |
| **Calico** | 支持 NetworkPolicy、BGP 路由 | 自建集群、需要网络策略 |
| **Cilium** | 基于 eBPF，高性能，强大的网络策略和可观测性 | 大规模集群、安全要求高 |
| **Flannel** | 最简单的 overlay 网络（VXLAN） | 学习、小集群 |
| **Weave** | 自动组网，易用 | 小规模多云 |

---

## 三、Service 类型详解

在 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|Helm 笔记]]中你已经使用了 ClusterIP 类型的 Service。这里完整介绍所有类型。

### 3.1 四种 Service 类型

```
外部流量入口
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ LoadBalancer（在 ClusterIP + NodePort 基础上，创建云负载均衡器）│
│ ┌──────────────────────────────────────────────────────┐ │
│ │ NodePort（在 ClusterIP 基础上，在每个节点开放固定端口）   │ │
│ │ ┌──────────────────────────────────────────────────┐ │ │
│ │ │ ClusterIP（基础：在集群内分配虚拟 IP）               │ │ │
│ │ │                                                  │ │ │
│ │ │   ClusterIP → iptables/IPVS → Pod IP             │ │ │
│ │ └──────────────────────────────────────────────────┘ │ │
│ └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

| Service 类型 | IP/端口 | 访问范围 | 适用场景 |
|------|------|------|------|
| **ClusterIP** | 集群内虚拟 IP（如 `10.100.50.23`） | 仅集群内 | 微服务间通信（本项目主要使用） |
| **NodePort** | 节点 IP + 固定端口（30000-32767） | 集群外可通过节点 IP 访问 | 开发测试、简单暴露 |
| **LoadBalancer** | 云厂商负载均衡器的外部 IP | 公网 | 直接暴露服务（不经过 Ingress） |
| **ExternalName** | CNAME 记录（DNS 别名） | DNS 级别 | 引用集群外部服务 |

### 3.2 ClusterIP 的实现（进阶）

```yaml
# Service 定义
apiVersion: v1
kind: Service
metadata:
  name: plaud-api
spec:
  type: ClusterIP
  selector:
    app: plaud-api          # 匹配 Pod 的 label
  ports:
    - port: 8001            # Service 暴露的端口
      targetPort: 8001      # Pod 上的端口
```

当 Pod 访问 `plaud-api:8001` 时的完整流程：

```
1. Pod 发起请求：plaud-api:8001
         ↓
2. CoreDNS 解析：plaud-api → 10.100.50.23（ClusterIP）
         ↓
3. 请求到达节点网络栈
         ↓
4. kube-proxy 配置的 iptables 规则：
   DNAT 10.100.50.23:8001 → 172.16.0.5:8001（某个 Pod IP）
   （如果有多个 Pod，随机选一个）
         ↓
5. 请求到达 Pod
```

#### 🔗 实战链接：generic-deployer 的 Service 模板

项目中所有微服务的 Service 通过 Helm 模板统一管理：

```yaml
# generic-deployer/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "generic-deployer.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    {{- range .Values.service.ports }}
    - port: {{ .port }}
      targetPort: {{ .targetPort | default .port }}
      protocol: {{ .protocol | default "TCP" }}
      name: {{ .name | default "http" }}
    {{- end }}
  selector:
    {{- include "generic-deployer.selectorLabels" . | nindent 4 }}
```

以 `plaud-api` 为例，它使用 ClusterIP 类型，将 Service 的 80 端口映射到容器的 8000 端口：

```yaml
# plaud-api/values/ap-northeast-1/prod/values-main.yaml（摘录）
service:
  type: ClusterIP
  ports:
    - port: 80
      name: web
      targetPort: 8000
```

而多端口服务（如 plaud-project-summary）需要暴露 API、前端和健康检查三个端口：

```yaml
# plaud-project-summary/values/.../workspace.yaml（摘录）
service:
  type: ClusterIP
  ports:
    - port: 8001
      name: api
      targetPort: 8001
    - port: 8889
      name: frontend
      targetPort: 8889
    - port: 8000
      name: health
      targetPort: 8000
```

> 所有微服务都使用 ClusterIP（仅集群内可达），外部访问通过 Ingress 进入——这是 3.1 节中描述的主流模式。

#### 🔗 实战链接：MongoDB LoadBalancer Service（跨 VPC 访问）

当需要让 EKS 集群外的服务（如其他 VPC 中的应用）访问数据库时，使用 LoadBalancer 类型：

```yaml
# infra/values/mongodb-prod/prod-retrieval/base/Cluster.yaml（摘录）
services:
  - annotations:
      service.beta.kubernetes.io/aws-load-balancer-name: "prod-retrieval"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"  # 内网 NLB
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
    componentSelector: mongodb
    name: vpc
    roleSelector: primary            # 只暴露主节点
    spec:
      ports:
        - name: tcp-mongodb
          port: 27017
          protocol: TCP
      type: LoadBalancer
      loadBalancerClass: service.k8s.aws/nlb

  - name: vpc-ro
    roleSelector: secondary          # 只暴露从节点（只读）
    spec:
      type: LoadBalancer
```

> 这里创建了两个 LoadBalancer Service：一个路由到主节点（读写），一个路由到从节点（只读）。`internal` scheme 确保 NLB 只在 VPC 内网可达，不暴露到公网。

#### 🔗 实战链接：ExternalName Service（跨 Namespace 服务引用）

项目中 `plaud-api` 需要将部分路径转发给其他 namespace 的服务，使用 ExternalName 作为 DNS 桥接：

```yaml
# plaud-api/values/ap-northeast-1/prod/pvt_ing.yaml（摘录）
apiVersion: v1
kind: Service
metadata:
  name: plaud-ask-external
  namespace: plaud-api
spec:
  type: ExternalName
  externalName: plaud-ask-jp-prod-main-deployer.plaud-ask.svc.cluster.local
  ports:
    - port: 80
      targetPort: 80
```

> ExternalName 是 3.1 节中的第四种 Service 类型。这里它的作用是：让 plaud-api namespace 中的 Ingress 能引用 plaud-ask namespace 的服务——因为 Ingress 的 backend 必须和 Ingress 在同一个 namespace。

### 3.3 Headless Service（无头服务）（进阶）

设置 `clusterIP: None` 的 Service 不分配虚拟 IP，DNS 直接返回所有 Pod 的 IP：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None           # Headless！
  selector:
    app: postgres
  ports:
    - port: 5432
```

```bash
# 普通 Service DNS 返回 ClusterIP
$ nslookup plaud-api
  10.100.50.23              # 一个虚拟 IP

# Headless Service DNS 返回所有 Pod IP
$ nslookup postgres-headless
  172.16.0.5                # Pod-0 的真实 IP
  172.16.0.8                # Pod-1 的真实 IP
  172.16.1.3                # Pod-2 的真实 IP
```

> Headless Service 主要配合 [[03-k8s-workload-types#三、StatefulSet — 有状态应用|StatefulSet]] 使用，让客户端能直连特定的 Pod（如连接数据库主节点）。

#### 🔗 实战链接：generic-deployer 的 Headless Service 模板

项目的共享 Helm Chart 中专门提供了 Headless Service 模板，供需要直连 Pod 的场景使用：

```yaml
# generic-deployer/templates/service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "generic-deployer.fullname" . }}-headless
  labels:
    app.kubernetes.io/headless: "true"
spec:
  type: {{ .Values.service.type }}
  clusterIP: None                    # 关键：Headless！
  ports:
    {{- range .Values.service.ports }}
    - port: {{ .port }}
      targetPort: {{ .targetPort | default .port }}
      name: {{ .name | default "http" }}
    {{- end }}
  selector:
    {{- include "generic-deployer.selectorLabels" . | nindent 4 }}
```

> 和普通 Service 模板的唯一区别就是 `clusterIP: None`。DNS 查询这个 Service 时，返回的是所有 Pod 的真实 IP 而非虚拟 IP。

---

## 四、CoreDNS — 集群内的 DNS 服务器

### 4.1 CoreDNS 的作用

在 [[10-helm-argocd-deployment#2.4 集群内服务地址（K8s Service DNS）|Helm 笔记]]中你已经用过 Service DNS（如 `plaud-project-summary-jp-prod-main-deployer.plaud-project-summary:8001`）。这些 DNS 记录由 CoreDNS 自动维护。

### 4.2 DNS 记录格式

```
Service DNS（最常用）：
<service-name>.<namespace>.svc.cluster.local
例：plaud-api.plaud-project-summary.svc.cluster.local

Pod DNS（Headless Service 下）：
<pod-name>.<service-name>.<namespace>.svc.cluster.local
例：postgres-0.postgres-headless.default.svc.cluster.local
```

**DNS 搜索域**：Pod 内可以省略后缀：

```
同 namespace：    plaud-api                     → 自动补全
跨 namespace：    plaud-api.other-ns             → 自动补全 .svc.cluster.local
完整域名：        plaud-api.other-ns.svc.cluster.local
```

### 4.3 CoreDNS 配置（进阶）

CoreDNS 以 DaemonSet（或 Deployment）形式运行在 `kube-system` namespace，配置通过 ConfigMap 管理：

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

```
.:53 {                          # 监听 53 端口
    kubernetes cluster.local {  # 处理集群内 DNS
        pods insecure           # Pod DNS 记录
        fallthrough             # 匹配不到则继续
    }
    forward . /etc/resolv.conf  # 集群外 DNS 转发到节点 DNS
    cache 30                    # 缓存 30 秒
}
```

### 4.4 DNS 排障

```bash
# 在 Pod 内测试 DNS 解析
kubectl run dnstest --image=busybox --rm -it -- nslookup plaud-api

# 查看 Pod 的 DNS 配置
kubectl exec <pod> -- cat /etc/resolv.conf
# nameserver 10.100.0.10          ← CoreDNS 的 ClusterIP
# search plaud-project-summary.svc.cluster.local svc.cluster.local cluster.local
# ndots:5                          ← 域名中 . 少于 5 个时先搜索搜索域

# 检查 CoreDNS 日志
kubectl logs -n kube-system -l k8s-app=kube-dns
```

> **ndots:5 的性能影响**：当 Pod 访问外部域名（如 `api.openai.com`，只有 2 个点），DNS 会先尝试 `api.openai.com.plaud-project-summary.svc.cluster.local` 等 5 个搜索域，全部失败后才查询真实域名。对于频繁访问外部域名的服务，可以在 Pod spec 中降低 `ndots` 或用 FQDN（末尾加 `.`）：`api.openai.com.`

---

## 五、Ingress 与 Ingress Controller

在 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|Helm 笔记]]中已经详细介绍了 Ingress 的用法。这里补充架构层面的理解。

### 5.1 Ingress 的本质

Ingress 本身**只是一个 API 对象**（一份 YAML 配置），定义了路由规则。真正干活的是 **Ingress Controller**——一个运行在集群中的反向代理（通常是 nginx 或 envoy），它 watch Ingress 资源的变更，动态更新自己的路由配置。

```
┌────────── K8s API ──────────┐     ┌──── Ingress Controller ────┐
│  Ingress 资源（YAML 配置）    │     │  nginx / envoy / traefik   │
│                              │ ──→ │                             │
│  host: api.plaud.ai          │     │  读取 Ingress 规则           │
│  path: /api → svc:8001       │     │  生成 nginx.conf            │
│                              │     │  reload 生效                │
└──────────────────────────────┘     └─────────────────────────────┘
```

### 5.2 EKS 中的 Ingress 架构（进阶）

本项目使用多个 Ingress Controller（nginx-internal、nginx-pvt、nginx-public），通过 `ingressClassName` 区分：

```
外部流量
    │
    ▼
AWS NLB/ALB（四层/七层负载均衡）
    │
    ├── nginx-public（公网入口）
    │   └── IngressClass: nginx-public
    │       └── api-apne1.plaud.ai → Service
    │
    ├── nginx-pvt（内网入口）
    │   └── IngressClass: nginx-pvt
    │       └── *-lan.plaud.ai → Service
    │
    └── nginx-internal（纯内部）
        └── IngressClass: nginx-internal
            └── *.nicebuild.click → Service
```

> **Ingress vs API Gateway vs Service Mesh**：
> - **Ingress**：L7 路由（域名 + 路径），当前项目方案
> - **API Gateway**（Kong/APISIX）：Ingress + 认证、限流、熔断等
> - **Service Mesh**（Istio）：服务间通信的全面管控（见第七节）

#### 🔗 实战链接：plaud-project-summary 的四个 Ingress（分层暴露 + 路径重写）

`plaud-project-summary` 同时配置了四个 Ingress，覆盖内网、私有网络和公网三个层级：

```yaml
# plaud-project-summary/values/ap-northeast-1/prod/main.yaml（摘录）
ingresses:
  # 1. 内网 — 前端 + API + 健康检查
  internal:
    ingressClassName: "nginx-internal"
    annotations:
      nginx.ingress.kubernetes.io/affinity: "cookie"           # NiceGUI 有状态框架需要会话亲和
      nginx.ingress.kubernetes.io/session-cookie-name: "NICEGUI_AFFINITY"
    hosts:
      - host: plaud-project-summary-apne1.nicebuild.click
        paths:
          - path: /              # 前端
            servicePort: 8889
          - path: /api           # API
            servicePort: 8001
          - path: /ping          # 健康检查
            servicePort: 8000

  # 2. 私有 — 仅暴露 Temporal API
  private:
    ingressClassName: "nginx-pvt"
    hosts:
      - host: plaud-project-summary-apne1-lan.plaud.ai
        paths:
          - path: /api/temporal
            servicePort: 8001

  # 3. 公网 — 仅暴露策略 API
  public:
    ingressClassName: "nginx-public"
    hosts:
      - host: plaud-project-summary-apne1.plaud.ai
        paths:
          - path: /api/strategy
            servicePort: 8001

  # 4. API 网关 — 路径重写，通过 /project-summary 前缀访问
  api-gateway:
    ingressClassName: "nginx-public"
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /$2
      nginx.ingress.kubernetes.io/use-regex: "true"
    hosts:
      - host: api-apne1.plaud.ai
        paths:
          - path: /project-summary(/|$)(.*)    # 匹配 /project-summary/xxx → 重写为 /xxx
            pathType: ImplementationSpecific
            servicePort: 8001
```

> 这展示了 Ingress 的精细路由能力：同一个服务通过四个 Ingress 在不同网络层级暴露不同的 API。`api-gateway` 的 `rewrite-target: /$2` 将 `/project-summary/xxx` 重写为 `/xxx`，实现了同一域名 `api-apne1.plaud.ai` 下多服务共存。

#### 🔗 实战链接：多 IngressClass 分层暴露（public / pvt / internal）

项目中同一个服务可以通过不同 IngressClass 暴露到不同网络层级。以 `plaud-project-summary` 为例，它配置了四个 Ingress：

```yaml
# plaud-project-summary/values/.../workspace.yaml（摘录）
ingresses:
  # 1. 内部入口 — 只在集群内部 DNS 可达
  internal:
    ingressClassName: "nginx-internal"
    hosts:
      - host: plaud-project-summary-staging-apne1.nicebuild.click
        paths:
          - path: /
            servicePort: 8889           # 前端
          - path: /api
            servicePort: 8001           # API

  # 2. 私有入口 — 公司内网可达
  private:
    ingressClassName: "nginx-pvt"
    hosts:
      - host: plaud-project-summary-staging-apne1-lan.plaud.ai
        paths:
          - path: /api/temporal
            servicePort: 8001

  # 3. 公网入口 — 互联网可达
  public:
    ingressClassName: "nginx-public"
    hosts:
      - host: plaud-project-summary-staging-apne1.plaud.ai
        paths:
          - path: /api/strategy
            servicePort: 8001

  # 4. API 网关转发
  api-gateway:
    ingressClassName: "nginx-public"
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /$2
      nginx.ingress.kubernetes.io/use-regex: "true"
    hosts:
      - host: api-staging-apne1.plaud.ai
        paths:
          - path: /project-summary(/|$)(.*)
            servicePort: 8001
```

> 这是 5.2 节架构图的具体实现。Helm 模板 `ingresses.yaml` 支持在一个 values 文件中定义多个 Ingress，每个 Ingress 使用不同的 `ingressClassName` 对应不同的 Ingress Controller：
> - `nginx-internal`（`*.nicebuild.click`）→ 纯内部，用于开发调试
> - `nginx-pvt`（`*-lan.plaud.ai`）→ 公司内网，用于内部系统调用
> - `nginx-public`（`*.plaud.ai`）→ 公网，用于终端用户访问

---

## 六、NetworkPolicy — 网络隔离

### 6.1 为什么需要 NetworkPolicy

K8s 的默认网络模型是**完全开放**的：任何 Pod 都可以访问任何 Pod。在生产环境中，这是不安全的——前端 Pod 不应该直接访问数据库，只有 API Pod 可以。

NetworkPolicy 定义了 Pod 级别的"防火墙规则"。

### 6.2 基本示例

```yaml
# 只允许来自同 namespace 中带 app=api label 的 Pod 访问数据库
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres          # 应用到数据库 Pod
  policyTypes:
    - Ingress                # 控制入站流量
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api       # 只允许 api Pod
      ports:
        - port: 5432
          protocol: TCP
```

### 6.3 常见策略模式（进阶）

```yaml
# 1. 默认拒绝所有入站流量（最佳实践：先关门，再开窗）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}            # 空 = 匹配所有 Pod
  policyTypes:
    - Ingress                # 没有 ingress 规则 = 全部拒绝

# 2. 允许同 namespace 内部通信
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}    # 同 namespace 的所有 Pod

# 3. 允许来自 Ingress Controller 的流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx   # Ingress Controller 所在 namespace
```

> **注意**：NetworkPolicy 需要 CNI 插件支持。Calico 和 Cilium 完整支持；AWS VPC CNI 需要配合 Calico 网络策略引擎使用。详见 [[08-k8s-security-rbac#四、NetworkPolicy 网络隔离（进阶）|K8s 安全]]。

#### 🔗 实战链接：kube-prometheus 的 NetworkPolicy（最小权限原则）

项目的监控系统 kube-prometheus 为每个组件都配置了 NetworkPolicy，是 6.3 节"先关门再开窗"最佳实践的真实例子：

```yaml
# infra/values/kube-prometheus/base/manifests/prometheus-networkPolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus-k8s
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: prometheus   # 应用到 Prometheus Pod
  policyTypes:
    - Egress
    - Ingress
  egress:
    - {}                                    # 出站不限制（需要抓取所有 Pod 的指标）
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus      # Prometheus 自身集群通信
      ports:
        - port: 9090
          protocol: TCP
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus-adapter  # 允许 metrics adapter
      ports:
        - port: 9090
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: grafana             # 允许 Grafana 查询
      ports:
        - port: 9090
```

Alertmanager 的 NetworkPolicy 更精细——只允许 Prometheus 和 Alertmanager 自身集群的流量：

```yaml
# infra/values/kube-prometheus/base/manifests/alertmanager-networkPolicy.yaml（简化）
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: alertmanager
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus   # 只有 Prometheus 能推送告警
      ports:
        - port: 9093
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: alertmanager  # Alertmanager 集群间通信
      ports:
        - port: 9094
          protocol: TCP
        - port: 9094
          protocol: UDP                             # gossip 协议用 UDP
```

> 这些 NetworkPolicy 精确控制了 monitoring namespace 中每个组件的访问权限：Prometheus 的 9090 端口只有 Grafana 和 prometheus-adapter 能访问；Alertmanager 的 9093 端口只有 Prometheus 能推送。这就是 6.2 节中"只允许特定 Pod 访问"模式的真实应用。

---

## 七、Service Mesh 概述（深入）

### 7.1 Service Mesh 解决什么问题

[[13-k8s-lane-mechanism|泳道机制]]通过 Ingress canary 实现了基础的流量分流，但仅限于外部流量入口。Service Mesh 在**所有服务间通信**中插入一层代理，提供全面的流量管控：

```
没有 Service Mesh：
Pod-A ──── 直接 HTTP ────→ Pod-B
（无加密、无重试、无限流、无链路追踪）

有 Service Mesh（Istio 为例）：
Pod-A ──→ Sidecar Proxy (Envoy) ──── mTLS ────→ Sidecar Proxy (Envoy) ──→ Pod-B
         │                                      │
         └── 重试、超时、熔断、限流、                └── 指标采集、链路追踪
             流量分流（泳道/灰度/蓝绿）
```

### 7.2 核心能力对比

| 能力 | Ingress | Service Mesh |
|------|------|------|
| 流量路由 | 外部流量的域名+路径路由 | 所有服务间流量的精细路由 |
| mTLS 加密 | 不支持（Pod 间通信明文） | 自动双向 TLS，零信任网络 |
| 可观测性 | Ingress Controller 的 access log | 全链路的延迟、成功率、流量指标 |
| 故障注入 | 不支持 | 可以模拟延迟、错误（混沌测试） |
| 流量分流 | canary（按 header） | 按比例（10% 新版本）、按 header、按用户 |
| 重试/超时 | Ingress 注解支持基础重试 | 全链路的重试、超时、熔断 |

### 7.3 主流 Service Mesh

| 方案 | 架构 | 特点 |
|------|------|------|
| **Istio** | Sidecar（每个 Pod 注入 Envoy 代理） | 功能最全、社区最大、复杂度高 |
| **Linkerd** | Sidecar（Rust 写的轻量代理） | 简单轻量、性能好、功能少于 Istio |
| **Cilium Service Mesh** | 无 Sidecar（基于 eBPF 内核级代理） | 性能最好、无 Sidecar 开销 |
| **AWS App Mesh** | Sidecar（Envoy）| AWS 托管，与 EKS 深度集成 |

> **是否需要 Service Mesh**：Service Mesh 增加了显著的复杂度和资源开销（每个 Pod 多一个 Sidecar 容器）。小规模微服务（<20 个）通常不需要。当你面临"服务间通信需要 mTLS"、"需要精细的流量管控"、"需要全链路可观测性"这些需求时再考虑引入。

---

## 八、网络排障速查

```bash
# 1. Pod 间连通性测试
kubectl run nettest --image=busybox --rm -it -- wget -O- http://plaud-api:8001/health

# 2. 查看 Service 的 Endpoints（Pod 映射）
kubectl get endpoints plaud-api -n plaud-project-summary
# NAME        ENDPOINTS                                   AGE
# plaud-api   172.16.0.5:8001,172.16.0.8:8001             5d

# 3. DNS 解析测试
kubectl run dnstest --image=busybox --rm -it -- nslookup plaud-api.plaud-project-summary

# 4. 查看 kube-proxy 的 iptables 规则（节点上执行）
iptables -t nat -L KUBE-SERVICES | grep plaud-api

# 5. 查看 NetworkPolicy 是否阻断流量
kubectl get networkpolicy -n plaud-project-summary

# 6. 查看 Pod 的网络命名空间
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- cat /etc/resolv.conf
```

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — Service、Ingress 的实际配置
- [[13-k8s-lane-mechanism|K8s 泳道机制]] — Ingress canary 实现流量分流
- [[12-k8s-pod-graceful-shutdown|Pod 优雅终止]] — kube-proxy 更新 iptables 的传播延迟
- [[05-k8s-architecture|K8s 架构原理]] — kube-proxy 在数据平面中的位置
- [[08-k8s-security-rbac|K8s 安全与权限]] — NetworkPolicy 的安全实践
