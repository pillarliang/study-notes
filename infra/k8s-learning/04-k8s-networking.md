# K8s 网络深入：一个请求的完整旅程

> [[02-k8s-core-concepts|核心概念篇]]介绍了 Service 和 Ingress **是什么**——Service 提供稳定的访问入口，Ingress 按域名/路径路由外部流量。本文接着问：**底层怎么实现的？** 一个 HTTP 请求从浏览器到达 Pod，中间到底经过了哪些环节？
>
> 以 plaud-project-summary 为例，一个用户访问 `plaud-project-summary-usw2.plaud.ai/api/strategy` 时，请求会经历：
>
> ```text
> 浏览器 → DNS → AWS NLB → Ingress Controller (nginx-public)
>       → ClusterIP Service → iptables DNAT → Pod (10.0.1.x:8001)
> ```
>
> 本文沿着这条路径，逐层剥开每个环节的原理。涉及的组件在 [[05-k8s-architecture|K8s 架构]]中有整体定位。

---

## 一、K8s 网络模型——三条必须满足的规则

**问题**：传统 Docker 容器依赖端口映射（`-p 8080:80`）才能被外部访问，容器之间的通信依赖 bridge 网络或 link。当集群有上百个容器分布在多台机器上时，端口映射和 bridge 网络根本管不过来。

K8s 的回答是定义一个**扁平网络模型**，所有网络插件（CNI）都必须满足三条规则：

1. **每个 Pod 拥有独立 IP**——不共享节点 IP，不需要端口映射
2. **所有 Pod 之间可直接通信**——跨节点也不需要 NAT
3. **Pod 看到的自身 IP = 其他 Pod 看到的 IP**——没有隐藏的地址转换

```
Node-1 (10.0.1.10)              Node-2 (10.0.2.10)
┌─────────────────┐             ┌─────────────────┐
│  Pod-A: 10.0.1.5│── 直接通信 ──│  Pod-C: 10.0.2.5│
│  Pod-B: 10.0.1.6│             │  Pod-D: 10.0.2.6│
└─────────────────┘             └─────────────────┘

Pod-A 可以直接访问 10.0.2.5:8001 到达 Pod-C，即使在不同节点上。
```

> [!tip] 与 Docker 的本质区别
> Docker 默认的 bridge 网络中，容器需要 `-p` 端口映射才能被外部访问。K8s Pod 天生拥有可路由的 IP，整个集群就像一个大的扁平局域网。

这三条规则只是"规范"，具体怎么实现取决于 CNI 插件。

---

## 二、CNI——Pod 的 IP 从哪来

**问题**：规则说"每个 Pod 有独立 IP"，但 IP 谁来分配？Pod 创建时网络怎么接通？

### 2.1 CNI 的工作流程

CNI（Container Network Interface）是 K8s 定义的网络插件标准接口。kubelet 创建 Pod 时，调用 CNI 插件完成网络配置：

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

不同的 CNI 插件在"分配 IP"和"配置路由"这两步的实现方式不同，导致了性能和功能上的差异。

### 2.2 AWS VPC CNI——EKS 的默认方案（进阶）

EKS 使用 AWS VPC CNI，其独特之处是：**Pod 直接获得 VPC 子网中的真实 IP**，不需要 overlay 网络。

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

这意味着 Pod IP 是 VPC 原生 IP，可以直接被 SecurityGroup、ALB、RDS 等 AWS 服务识别，无需 overlay 封装/解封装，性能最好。

**代价是单节点 Pod 数量有上限**（受 ENI 数和每 ENI IP 数限制）：

| 实例类型 | 最大 ENI 数 | 每 ENI 最大 IP 数 | 最大 Pod 数 |
|------|------|------|------|
| t3.medium | 3 | 6 | 17 |
| m5.large | 3 | 10 | 29 |
| m5.xlarge | 4 | 15 | 58 |
| m5.4xlarge | 8 | 30 | 234 |

> [!tip] 排障提示
> 如果 Pod 一直 Pending 且事件显示 `Too many pods` 或 IP 分配失败，往往是实例类型的 ENI/IP 上限已用尽。换更大的实例类型或减少节点上的 Pod 数。

### 2.3 主流 CNI 对比

| CNI 插件 | 核心特点 | 典型场景 |
|------|------|------|
| **AWS VPC CNI** | Pod 用 VPC 真实 IP，性能最好 | EKS（默认） |
| **Calico** | 支持 NetworkPolicy、BGP 路由 | 自建集群、需要网络策略 |
| **Cilium** | 基于 eBPF，高性能 + 强网络策略 + 可观测性 | 大规模集群、安全要求高 |
| **Flannel** | 最简单的 overlay 网络（VXLAN） | 学习、小集群 |

---

## 三、Service 的实现——kube-proxy 与 iptables（进阶）

**问题**：Pod IP 有了，但 Pod 会重建、IP 会变。总不能让调用方每次都去查新 IP。Service 提供了稳定的虚拟 IP（ClusterIP），那这个"虚拟 IP"背后是怎么工作的？

### 3.1 ClusterIP 到 Pod 的完整链路

以 plaud-project-summary 的 Service 为例：

```yaml
# 来源：deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
deployer:
  service:
    type: ClusterIP
    ports:
      - port: 8001        # API
        name: api
      - port: 8889        # 前端（NiceGUI）
        name: frontend
      - port: 8000        # 健康检查
        name: health
```

当集群内另一个 Pod 访问 `plaud-project-summary:8001` 时，完整流程：

```
1. Pod 发起请求 → plaud-project-summary:8001
         ↓
2. CoreDNS 解析 → 10.100.50.23（ClusterIP，一个虚拟 IP）
         ↓
3. 请求到达节点的网络栈
         ↓
4. kube-proxy 预先配置的 iptables 规则执行 DNAT：
   10.100.50.23:8001 → 172.16.0.5:8001（某个后端 Pod IP）
   （多副本时随机选择，或按 IPVS 的调度算法选择）
         ↓
5. 请求到达目标 Pod
```

关键点：**ClusterIP 不绑定在任何网卡上**，它是一个仅存在于 iptables/IPVS 规则中的虚拟地址。kube-proxy 运行在每个节点上，负责 watch Service 和 Endpoints 的变更，并实时更新这些规则。

### 3.2 kube-proxy 的两种模式

| 模式               | 原理                               | 优劣                                                         |
|--------------------|------------------------------------|--------------------------------------------------------------|
| **iptables**（默认） | 为每个 Service 生成一组 DNAT 规则     | 简单可靠；规则数随 Service 增长，大规模时性能下降                  |
| **IPVS**           | 使用 Linux 内核的 IPVS 负载均衡模块   | 支持多种调度算法（轮询、最少连接等）；大规模集群更高效               |

### 3.3 四种 Service 类型

每种类型是前一种的叠加：

```
┌──────────────────────────────────────────────────────────┐
│ LoadBalancer（在 NodePort 基础上，创建云负载均衡器）         │
│ ┌──────────────────────────────────────────────────────┐ │
│ │ NodePort（在 ClusterIP 基础上，在每个节点开放固定端口） │ │
│ │ ┌──────────────────────────────────────────────────┐ │ │
│ │ │ ClusterIP（基础：集群内虚拟 IP + iptables 转发）   │ │ │
│ │ └──────────────────────────────────────────────────┘ │ │
│ └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
  ExternalName（特殊：纯 DNS CNAME，不走 iptables）
```

| 类型 | 访问范围 | 典型用法 |
|------|------|------|
| **ClusterIP** | 仅集群内 | 微服务间通信（本项目主要使用） |
| **NodePort** | 集群外可通过节点 IP:端口访问 | 开发测试 |
| **LoadBalancer** | 云厂商 LB 的外部 IP | 直接暴露服务（不经过 Ingress） |
| **ExternalName** | DNS CNAME | 引用集群外部服务或跨 namespace 桥接 |

### 3.4 实战：Headless Service（进阶）

**问题**：普通 Service 的 ClusterIP 会做负载均衡，但数据库客户端需要直连主节点的 IP。怎么办？

设置 `clusterIP: None` 即为 Headless Service，DNS 查询直接返回所有后端 Pod 的真实 IP，不分配虚拟 IP：

```bash
# 普通 Service → 返回一个虚拟 IP
$ nslookup plaud-api
  10.100.50.23

# Headless Service → 返回所有 Pod IP
$ nslookup postgres-headless
  172.16.0.5    # Pod-0
  172.16.0.8    # Pod-1
  172.16.1.3    # Pod-2
```

> [!tip] Headless Service 主要配合 [[03-k8s-workload-types#三、StatefulSet — 有状态应用|StatefulSet]] 使用，让客户端能精确连接特定 Pod（如数据库主节点）。

### 3.5 实战：ExternalName 跨 Namespace 桥接（进阶）

**问题**：Ingress 的 backend 必须和 Ingress 在同一个 namespace，但目标服务在另一个 namespace。怎么引用？

ExternalName Service 充当 DNS 桥接：

```yaml
# 来源：plaud-api/values/ap-northeast-1/prod/pvt_ing.yaml
apiVersion: v1
kind: Service
metadata:
  name: plaud-ask-external
  namespace: plaud-api
spec:
  type: ExternalName
  externalName: plaud-ask-jp-prod-main-deployer.plaud-ask.svc.cluster.local
```

plaud-api namespace 中的 Ingress 引用 `plaud-ask-external`，DNS 自动解析到 plaud-ask namespace 中的真实 Service。

---

## 四、CoreDNS——Service DNS 怎么工作

**问题**：上面反复提到"DNS 解析 Service 名称为 ClusterIP"，这个 DNS 服务器是谁？记录怎么维护的？

### 4.1 DNS 记录格式

CoreDNS 以 Deployment 形式运行在 `kube-system` namespace，自动为每个 Service 创建 DNS 记录：

```
Service DNS（最常用）：
<service-name>.<namespace>.svc.cluster.local
例：plaud-project-summary-usw2-prod-main-deployer.plaud-project-summary.svc.cluster.local

Pod DNS（Headless Service 下）：
<pod-name>.<service-name>.<namespace>.svc.cluster.local
例：postgres-0.postgres-headless.default.svc.cluster.local
```

**DNS 搜索域简化书写**——Pod 内不需要写全名：

```
同 namespace 内：  plaud-api                  → 自动补全
跨 namespace：     plaud-api.other-ns          → 自动补全 .svc.cluster.local
完整域名：         plaud-api.other-ns.svc.cluster.local
```

### 4.2 ndots:5 的性能陷阱（进阶）

每个 Pod 的 `/etc/resolv.conf` 默认配置 `ndots:5`，含义是：域名中 `.` 少于 5 个时，先在搜索域中查找。

当 Pod 访问外部域名（如 `api.openai.com`，只有 2 个 `.`），DNS 会**依次尝试**：

```
api.openai.com.plaud-project-summary.svc.cluster.local  → 失败
api.openai.com.svc.cluster.local                         → 失败
api.openai.com.cluster.local                              → 失败
api.openai.com.us-west-2.compute.internal                 → 失败
api.openai.com                                            → 成功（第 5 次）
```

对频繁调用外部 API 的服务，这会导致多余的 DNS 查询。解决方式：在域名末尾加 `.` 使用 FQDN（`api.openai.com.`），或在 Pod spec 中降低 `ndots` 值。

### 4.3 CoreDNS 配置概览（深入）

```
.:53 {
    kubernetes cluster.local {   # 处理集群内 DNS
        pods insecure
        fallthrough              # 匹配不到则继续下一个插件
    }
    forward . /etc/resolv.conf   # 集群外域名转发到节点 DNS
    cache 30                     # 缓存 30 秒
}
```

---

## 五、Ingress——HTTP 路由的实现

**问题**：ClusterIP Service 只在集群内可达。外部用户怎么通过域名访问服务？

### 5.1 Ingress 的本质

Ingress 本身**只是一个 API 对象**（一份路由规则的声明）。真正处理流量的是 **Ingress Controller**——一个运行在集群中的反向代理（通常是 nginx），它 watch Ingress 资源的变更，动态生成自己的路由配置：

```
┌─────── K8s API ────────┐      ┌──── Ingress Controller ────┐
│  Ingress 资源（YAML）    │      │  nginx / envoy / traefik   │
│                         │ ───→ │                             │
│  host: api.plaud.ai     │      │  读取 Ingress 规则           │
│  path: /api → svc:8001  │      │  生成 nginx.conf → reload   │
└─────────────────────────┘      └─────────────────────────────┘
```

### 5.2 实战：plaud-project-summary 的三层 Ingress

同一个服务通过三个不同的 `ingressClassName` 暴露到不同网络层级——内网开放全功能，公网只暴露最小 API 表面：

```yaml
# 来源：deploy/plaud-project-summary/values/us-west-2/prod/main.yaml
deployer:
  ingresses:
    # --- 第一层：内部入口（开发调试）---
    internal:
      ingressClassName: "nginx-internal"
      annotations:
        nginx.ingress.kubernetes.io/affinity: "cookie"
        nginx.ingress.kubernetes.io/session-cookie-name: "NICEGUI_AFFINITY"
      hosts:
        - host: plaud-project-summary-usw2.nicebuild.click
          paths:
            - path: /                # 前端 → 8889
              servicePort: 8889
            - path: /api             # API → 8001
              servicePort: 8001

    # --- 第二层：私有入口（公司内网）---
    private:
      ingressClassName: "nginx-pvt"
      hosts:
        - host: plaud-project-summary-usw2-lan.plaud.ai
          paths:
            - path: /api/temporal    # 仅暴露 Temporal API
              servicePort: 8001

    # --- 第三层：公网入口（终端用户）---
    public:
      ingressClassName: "nginx-public"
      hosts:
        - host: plaud-project-summary-usw2.plaud.ai
          paths:
            - path: /api/strategy    # 仅暴露 Strategy API
              servicePort: 8001
```

几个值得注意的设计：

- **Cookie affinity**：internal 层为 NiceGUI 前端配置了 session cookie 亲和性，确保同一用户的 WebSocket 连接始终到达同一个 Pod
- **逐层收窄**：internal 开放 `/` 和 `/api`，private 只开放 `/api/temporal`，public 只开放 `/api/strategy`
- **CN 区域 TLS**：中国区的 Ingress 额外配置了 `tls.secretName: plaudcn`，因为 CN 区域需要自行管理 TLS 证书（非 AWS Certificate Manager）

### 5.3 EKS 中的多 Ingress Controller 架构（进阶）

每个 `ingressClassName` 对应一个独立的 Ingress Controller 实例，各自关联不同的 AWS 负载均衡器：

```
外部流量
    │
    ▼
AWS NLB / ALB
    │
    ├── nginx-public  → 公网 NLB → IngressClass: nginx-public
    │   └── *-usw2.plaud.ai → Service
    │
    ├── nginx-pvt     → 内网 NLB → IngressClass: nginx-pvt
    │   └── *-lan.plaud.ai → Service
    │
    └── nginx-internal → 内部 NLB → IngressClass: nginx-internal
        └── *.nicebuild.click → Service
```

> [!tip] Ingress vs API Gateway vs Service Mesh
> - **Ingress**：L7 路由（域名 + 路径），当前项目方案
> - **API Gateway**（Kong/APISIX）：Ingress + 认证、限流、熔断
> - **Service Mesh**（Istio）：服务间通信的全面管控（mTLS、链路追踪、故障注入）

### 5.4 路径重写——API 网关模式（进阶）

除了三层分级暴露，plaud-project-summary 还通过路径重写实现多服务共享同一域名：

```yaml
# 来源：deploy/plaud-project-summary/values/ap-northeast-1/prod/main.yaml
ingresses:
  api-gateway:
    ingressClassName: "nginx-public"
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /$2
      nginx.ingress.kubernetes.io/use-regex: "true"
    hosts:
      - host: api-apne1.plaud.ai
        paths:
          - path: /project-summary(/|$)(.*)   # /project-summary/xxx → /xxx
            pathType: ImplementationSpecific
            servicePort: 8001
```

`rewrite-target: /$2` 将 `/project-summary/xxx` 重写为 `/xxx`，每个微服务各自定义一个带前缀的 Ingress，统一汇聚到 `api-apne1.plaud.ai` 这个网关域名。

---

## 六、NetworkPolicy——Pod 级别的防火墙（进阶）

**问题**：K8s 默认网络模型是**完全开放**的——任何 Pod 可以访问任何 Pod。生产环境中，前端 Pod 不应该直连数据库，只有 API Pod 可以。怎么实现隔离？

### 6.1 核心思路：先关门，再开窗

NetworkPolicy 就是 Pod 级别的防火墙规则。最佳实践是：先用一条策略拒绝所有入站流量，再逐条放行需要的路径。

```yaml
# 第一步：默认拒绝所有入站（关门）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}          # 空 = 匹配所有 Pod
  policyTypes:
    - Ingress              # 没有 ingress 规则 = 全部拒绝
```

```yaml
# 第二步：只允许 api Pod 访问数据库（开窗）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - port: 5432
```

### 6.2 实战：Prometheus 的网络隔离

项目的监控系统为每个组件配置了精细的 NetworkPolicy：

```yaml
# 来源：infra/values/kube-prometheus/base/manifests/prometheus-networkPolicy.yaml
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: prometheus
  policyTypes: [Egress, Ingress]
  egress:
    - {}                                    # 出站不限（需抓取所有 Pod 指标）
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: grafana   # 只有 Grafana 能查询 9090
      ports:
        - port: 9090
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus  # 集群内自身通信
      ports:
        - port: 9090
```

Alertmanager 的隔离更精细——9093 只允许 Prometheus 推送告警，9094 只允许 Alertmanager 自身 gossip 通信（TCP + UDP）。

### 6.3 常见策略模式速查

| 需求 | 关键配置 |
|------|------|
| 默认拒绝所有入站 | `podSelector: {}` + `policyTypes: [Ingress]` + 不写 ingress 规则 |
| 允许同 namespace 内通信 | `ingress.from.podSelector: {}` |
| 允许 Ingress Controller 流量 | `ingress.from.namespaceSelector.matchLabels: {name: ingress-nginx}` |
| 允许特定 Pod 访问特定端口 | `ingress.from.podSelector + ports` |

> [!tip] NetworkPolicy 需要 CNI 插件支持。Calico 和 Cilium 完整支持；AWS VPC CNI 需配合 Calico 网络策略引擎使用。详见 [[08-k8s-security-rbac#四、NetworkPolicy 网络隔离（进阶）|K8s 安全]]。

---

## 七、排障速查

当网络不通时，沿着请求路径逐层排查：

```bash
# 1. Pod 网络是否正常（Pod 层）
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- cat /etc/resolv.conf

# 2. DNS 能否解析（CoreDNS 层）
kubectl run dnstest --image=busybox --rm -it -- nslookup <service-name>
kubectl logs -n kube-system -l k8s-app=kube-dns

# 3. Service 是否关联到 Pod（Service 层）
kubectl get endpoints <service-name> -n <namespace>
# 如果 ENDPOINTS 为空 → selector 和 Pod label 不匹配，或 Pod 未 Ready

# 4. iptables 规则是否生效（kube-proxy 层，节点上执行）
iptables -t nat -L KUBE-SERVICES | grep <service-name>

# 5. Pod 间能否直接通信（CNI 层）
kubectl run nettest --image=busybox --rm -it -- wget -O- http://<pod-ip>:<port>/health

# 6. NetworkPolicy 是否阻断流量（策略层）
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>
```

> [!tip] 排障顺序遵循请求路径：DNS → Service Endpoints → iptables → Pod 直连 → NetworkPolicy。定位到哪一层不通，就在哪一层修复。

---

## 延伸阅读

- [[02-k8s-core-concepts|K8s 核心概念]] — Service 和 Ingress 的基本用法
- [[05-k8s-architecture|K8s 架构原理]] — kube-proxy 在数据平面中的位置
- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — Service、Ingress 的 Helm 配置实战
- [[13-k8s-lane-mechanism|K8s 泳道机制]] — Ingress canary 实现流量分流
- [[12-k8s-pod-graceful-shutdown|Pod 优雅终止]] — kube-proxy 更新 iptables 的传播延迟
- [[08-k8s-security-rbac|K8s 安全与权限]] — NetworkPolicy 的安全实践
