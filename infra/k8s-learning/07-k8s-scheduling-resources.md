# K8s 调度与资源管理

> 前置知识：了解 [[05-k8s-architecture#3.3 Scheduler — "Pod 应该跑在哪个节点"|Scheduler 的基本调度流程]]和 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|resources 和 HPA 的配置]]。本文深入讲解资源管理、调度策略和自动扩缩容的完整体系。
>
> **学习建议**：首次阅读重点关注第一至二节（requests/limits、QoS）、4.1（nodeSelector）、5.1（HPA）和第七节（最佳实践）。标记为（进阶）的小节和第三节掌握基础后再看，第六节（Karpenter）为深入。

---

## 一、资源模型：requests 与 limits

### 1.1 两者的含义

```yaml
resources:
  requests:           # "我至少需要这么多"—— Scheduler 据此选节点
    cpu: 2
    memory: 4Gi
  limits:             # "我最多用这么多"—— kubelet 据此限制使用
    cpu: 4
    memory: 8Gi
```

| 字段 | 谁使用 | 作用 | 超出会怎样 |
|------|------|------|------|
| **requests** | Scheduler | 选择有足够剩余资源的节点 | — |
| **limits** | kubelet (cgroups) | 限制容器实际可用资源 | CPU 被限流（throttle），内存超出则 OOMKilled |

### 1.2 CPU 与内存的行为差异

| 资源 | 可压缩 | 超过 limits 的后果 | 单位 |
|------|------|------|------|
| **CPU** | 是 | 不会被杀，但会被限流（进程变慢） | `1` = 1 核，`500m` = 0.5 核 |
| **内存** | 否 | OOMKilled（容器被杀重启） | `Mi`（MiB）、`Gi`（GiB） |

> **CPU 限流（Throttling）** 是生产环境常见的性能问题。当容器在一个 CPU 调度周期（默认 100ms）内使用量达到 limits，剩余时间会被暂停。表现为：延迟突然增大但 CPU 利用率看起来"不高"。如果 Pod 有延迟毛刺但 CPU 使用率不到 limits，可能就是 throttling。

### 1.3 本项目中的资源配置模式

在 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|values 文件]]中，生产环境 `requests = limits`：

```yaml
resources:
  limits:
    cpu: 8
    memory: 16Gi
  requests:
    cpu: 8           # requests = limits → Guaranteed QoS
    memory: 16Gi
```

这确保了 **Guaranteed QoS**（下文详述），Pod 独占所申请的资源，不会被驱逐。

> [!tip] 🔗 实战链接：不同服务的资源配置差异
> 
> 在 deploy 项目中，不同服务根据业务特性有截然不同的资源配置：
> 
> **plaud-project-summary（AI 总结服务，CPU 密集型）** — `plaud-project-summary/values/ap-northeast-1/prod/main.yaml`：
> ```yaml
> resources:
>   limits:
>     cpu: 8
>     memory: 16Gi
>   requests:
>     cpu: 8            # requests = limits → Guaranteed QoS
>     memory: 16Gi
> ```
> 
> **plaud-transcribe（语音转录服务，同样是计算密集型）** — `plaud-transcribe/values/us-west-2/prod/values-main.yaml`：
> ```yaml
> resources:
>   limits:
>     cpu: 8
>     memory: 8Gi
>   requests:
>     cpu: 8            # 同样 Guaranteed QoS，但内存需求只有一半
>     memory: 8Gi
> ```
> 
> **plaud-global-gms-agent（AI Agent 服务，超大规格）** — `plaud-global-gms-agent/values/ap-northeast-1/prod/values-main.yaml`：
> ```yaml
> resources:
>   limits:
>     cpu: 12
>     memory: 24Gi
>   requests:
>     cpu: 12           # 这是项目中资源配置最大的服务之一
>     memory: 24Gi
> ```
> 
> **同一服务在 staging 环境资源缩减** — `plaud-project-summary/values/ap-northeast-1/staging/main.yaml`：
> ```yaml
> resources:
>   limits:
>     cpu: 1            # 生产 8 核 → staging 1 核，节省成本
>     memory: 4Gi       # 生产 16Gi → staging 4Gi
>   requests:
>     cpu: 1
>     memory: 4Gi
> ```
> 
> 这体现了"7.1 资源配置策略"表格中生产核心 vs staging 的差异——生产用 Guaranteed QoS 保障稳定性，staging 给更小资源降低成本。

---

## 二、QoS（服务质量等级）

K8s 根据 requests 和 limits 的配置，自动为 Pod 分配 QoS 等级，决定资源紧张时的驱逐优先级。

### 2.1 三种 QoS 等级

| QoS 等级 | 条件 | 驱逐优先级 | 适用场景 |
|------|------|------|------|
| **Guaranteed** | 所有容器的 requests = limits（CPU 和内存都设） | 最后被驱逐 | 生产核心服务 |
| **Burstable** | 至少一个容器设了 requests 或 limits，但不满足 Guaranteed | 中间 | 一般工作负载 |
| **BestEffort** | 没有设任何 requests 和 limits | 最先被驱逐 | 测试、不重要的任务 |

### 2.2 判定规则

```
Pod 中的每个容器都满足 requests == limits（CPU + Memory）？
    ├── 是 → Guaranteed（最高优先级）
    └── 否 → 至少一个容器设了 requests 或 limits？
                ├── 是 → Burstable
                └── 否 → BestEffort（最先被驱逐）
```

### 2.3 驱逐顺序

当节点内存不足时，kubelet 按以下顺序驱逐 Pod：

```
节点内存压力！
    ↓
1. 驱逐 BestEffort Pod（先杀实际内存使用最多的）
    ↓ 还不够
2. 驱逐 Burstable Pod（先杀超出 requests 比例最多的）
    ↓ 还不够
3. 驱逐 Guaranteed Pod（极端情况，几乎不会发生）
```

> 这就是为什么生产环境的核心服务要设 `requests = limits`——确保 Guaranteed QoS，在资源竞争时最后被影响。

---

## 三、LimitRange 与 ResourceQuota（进阶）

### 3.1 LimitRange — namespace 级默认值

如果开发者忘记设 resources，LimitRange 自动补上默认值：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:               # 没设 limits 时的默认值
        cpu: "1"
        memory: 1Gi
      defaultRequest:        # 没设 requests 时的默认值
        cpu: 500m
        memory: 512Mi
      max:                   # 单个容器的上限
        cpu: "8"
        memory: 32Gi
      min:                   # 单个容器的下限
        cpu: 100m
        memory: 128Mi
```

### 3.2 ResourceQuota — namespace 级总量限制

防止某个团队/项目占用过多集群资源：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"          # namespace 内所有 Pod 的 CPU requests 总和上限
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    pods: "50"                   # Pod 总数上限
    persistentvolumeclaims: "20" # PVC 数量上限
```

```bash
# 查看配额使用情况
kubectl describe resourcequota team-quota -n production
# Name:             team-quota
# Resource          Used    Hard
# --------          ----    ----
# limits.cpu        48      200
# limits.memory     128Gi   400Gi
# pods              12      50
```

---

## 四、调度策略

### 4.1 nodeSelector — 最简单的节点选择

```yaml
spec:
  nodeSelector:
    node-type: gpu           # Pod 只能调度到有此 label 的节点
```

### 4.2 Node Affinity — 更灵活的节点选择（进阶）

```yaml
spec:
  affinity:
    nodeAffinity:
      # 硬性要求：必须满足（等同于 nodeSelector 但更灵活）
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["ap-northeast-1a", "ap-northeast-1c"]
      
      # 软性偏好：尽量满足，但不强制
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80           # 权重（0-100），越高越优先
          preference:
            matchExpressions:
              - key: node-type
                operator: In
                values: ["compute-optimized"]
```

### 4.3 Pod Anti-Affinity — 让 Pod 分散（进阶）

避免同一个 Deployment 的多个 Pod 挤在同一个节点/可用区：

```yaml
spec:
  affinity:
    podAntiAffinity:
      # 硬性：同一个 Deployment 的 Pod 不能在同一个节点
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: plaud-api
          topologyKey: kubernetes.io/hostname   # 按节点分散
      
      # 软性：尽量分散到不同可用区
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: plaud-api
            topologyKey: topology.kubernetes.io/zone   # 按 AZ 分散
```

### 4.4 Topology Spread Constraints — 均匀分布（进阶）

比 Anti-Affinity 更精确地控制 Pod 在拓扑域中的分布：

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                           # 最多允许各域之间差 1 个 Pod
      topologyKey: topology.kubernetes.io/zone   # 按可用区分布
      whenUnsatisfiable: DoNotSchedule     # 不满足就不调度
      labelSelector:
        matchLabels:
          app: plaud-api
```

> 本项目的 `generic-deployer` 模板中使用了 Topology Spread Constraints（见 [[10-helm-argocd-deployment#2.1 generic-deployer：通用部署基座|通用部署基座]]），确保 Pod 在可用区间均匀分布。

> [!tip] 🔗 实战链接：generic-deployer 的默认 Topology Spread Constraints
>
> `generic-deployer/templates/deployment.yaml` 中内置了默认的拓扑分布约束——如果服务的 values 中没有显式配置，模板会自动注入：
>
> ```yaml
> # 默认 topology spread constraints（values 中未配置时自动生效）
> topologySpreadConstraints:
>   - maxSkew: 1
>     topologyKey: topology.kubernetes.io/zone       # 按可用区均匀分布
>     whenUnsatisfiable: ScheduleAnyway              # 软性约束：尽量均匀但不阻止调度
>     labelSelector:
>       matchLabels: ...                              # 自动注入 Deployment 的 selectorLabels
>   - maxSkew: 1
>     topologyKey: kubernetes.io/hostname             # 按节点均匀分布
>     whenUnsatisfiable: ScheduleAnyway
>     labelSelector:
>       matchLabels: ...
> ```
>
> 这里用了 **两层拓扑分布**：先按 AZ（可用区）打散，再按节点打散，确保 Pod 不会集中在某个 AZ 或某台机器上。注意用的是 `ScheduleAnyway`（软性）而非 `DoNotSchedule`（硬性），这样在节点不足时不会阻塞部署。

**Topology Spread vs Anti-Affinity**：

| 特性 | Anti-Affinity | Topology Spread |
|------|------|------|
| 精确度 | "不在同一个 X 上" | "各 X 之间最多差 N 个" |
| 效果 | 3 AZ 3 Pod → 每 AZ 1个（理想情况） | 3 AZ 5 Pod → 2/2/1（精确控制 maxSkew=1） |
| 可扩展性 | Pod 数 > 节点数时无法调度 | maxSkew 允许一定程度不均衡 |

### 4.5 Taints & Tolerations — 节点"驱赶"（进阶）

Taint（污点）标记在节点上，表示"除非你能容忍我，否则不要过来"。Toleration（容忍）标记在 Pod 上。

```bash
# 给节点打 taint
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
```

```yaml
# Pod 声明容忍
spec:
  tolerations:
    - key: nvidia.com/gpu
      operator: Equal
      value: "true"
      effect: NoSchedule
```

**三种 Taint 效果**：

| Effect | 行为 |
|------|------|
| `NoSchedule` | 新 Pod 不会被调度到有此 taint 的节点 |
| `PreferNoSchedule` | 尽量不调度，但实在没地方了还是会 |
| `NoExecute` | 已在运行的 Pod 如果没有容忍也会被驱逐 |

**常见用法**：
- GPU 节点打 taint → 只有需要 GPU 的 Pod 才调度过去
- Master 节点默认有 `NoSchedule` taint → 工作负载不会跑在 master 上
- 节点维护时打 `NoExecute` → 所有 Pod 被驱逐到其他节点

> [!tip] 🔗 实战链接：WebSocket 服务的 Taint + NodeAffinity 组合
>
> `plaud-ws-notify`（WebSocket 长连接通知服务）需要运行在专用节点上，避免与普通服务争抢资源。`plaud-ws-notify/values/us-west-2/prod/values-main.yaml` 中同时配置了 tolerations 和 nodeAffinity：
>
> ```yaml
> tolerations:
>   - key: "workload"
>     operator: "Equal"
>     value: "websockit"
>     effect: "NoSchedule"
>
> affinity:
>   nodeAffinity:
>     requiredDuringSchedulingIgnoredDuringExecution:
>       nodeSelectorTerms:
>         - matchExpressions:
>             - key: workload
>               operator: In
>               values: ["websockit"]
> ```
>
> 这里用了 **Taint + Affinity 双保险**：Taint 确保普通 Pod 不会被调度到 WebSocket 专用节点；NodeAffinity 确保 WebSocket Pod 只会被调度到这些专用节点。两者配合实现了"专用节点池"的效果。

> [!tip] 🔗 实战链接：plaud-file 的 Pod Anti-Affinity
>
> `plaud-file`（文件处理服务）使用了 hostPath 挂载本地磁盘做临时存储，因此同一节点不能跑多个副本。`plaud-file/values/ap-northeast-1/prod/values-main.yaml`：
>
> ```yaml
> nodeSelector:
>   workload: general              # 只调度到 general 工作负载节点
>
> affinity:
>   podAntiAffinity:
>     requiredDuringSchedulingIgnoredDuringExecution:
>       - labelSelector:
>           matchLabels:
>             app_name: plaud-file
>         topologyKey: kubernetes.io/hostname   # 同一节点不能有两个 plaud-file Pod
> ```
>
> 用 `requiredDuring...`（硬性约束）+ `kubernetes.io/hostname` 确保每个节点最多一个 plaud-file Pod，避免 hostPath 冲突。

---

## 五、自动扩缩容

### 5.1 HPA（水平自动扩缩容）

HPA 根据指标自动调整 Deployment 的副本数。在 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|values 文件]]中已经配置了基于 CPU 的 HPA：

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 75
```

**HPA 的工作原理**：

```
每 15 秒（默认）：
1. 从 Metrics Server 获取当前 CPU 使用率
2. 计算：期望副本数 = 当前副本数 × (当前使用率 / 目标使用率)
   例：3 个 Pod，CPU = 90%，目标 = 75%
   期望 = ceil(3 × 90/75) = ceil(3.6) = 4
3. 调整 Deployment 的 replicas = 4
```

**扩容与缩容的行为差异**：

| 行为 | 速度 | 原因 |
|------|------|------|
| 扩容 | 快（默认无延迟） | 宁可多花钱也不能影响用户体验 |
| 缩容 | 慢（默认 5 分钟稳定窗口） | 避免流量波动导致反复扩缩 |

> [!tip] 🔗 实战链接：HPA 模板与 plaud-project-summary 的 HPA 配置
>
> `generic-deployer/templates/hpa.yaml` 是共享 HPA 模板，支持 CPU 和内存两种指标：
>
> ```yaml
> apiVersion: autoscaling/v2
> kind: HorizontalPodAutoscaler
> spec:
>   scaleTargetRef:
>     apiVersion: apps/v1
>     kind: Deployment
>     name: {{ include "generic-deployer.fullname" . }}
>   minReplicas: {{ .Values.autoscaling.minReplicas }}
>   maxReplicas: {{ .Values.autoscaling.maxReplicas }}
>   metrics:
>     - type: Resource
>       resource:
>         name: cpu
>         target:
>           type: Utilization
>           averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
> ```
>
> 在 `plaud-project-summary/values/ap-northeast-1/prod/main.yaml` 中实际启用：
>
> ```yaml
> autoscaling:
>   enabled: true
>   minReplicas: 2          # 最少保持 2 个副本（高可用）
>   maxReplicas: 10         # 最多扩到 10 个
>   targetCPUUtilizationPercentage: 75   # CPU 使用率超过 75% 触发扩容
> ```
>
> 而在 staging 环境中（`ap-northeast-1/staging/main.yaml`）HPA 是关闭的，固定 1 个副本节省成本：
>
> ```yaml
> autoscaling:
>   enabled: false
>   minReplicas: 1
>   maxReplicas: 1
> ```
>
> 这体现了生产 vs 非生产的扩缩策略差异——生产需要弹性应对流量波动，staging 只需保证功能可用。

**进阶：基于自定义指标的 HPA**：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second    # 自定义指标
        target:
          type: AverageValue
          averageValue: "1000"              # 每个 Pod 平均 1000 QPS
```

### 5.2 VPA（垂直自动扩缩容）（进阶）

HPA 调整副本数（水平），VPA 调整单个 Pod 的 resources（垂直）：

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: plaud-api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: plaud-api
  updatePolicy:
    updateMode: "Auto"          # 自动调整（需要重启 Pod）
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 500m
          memory: 1Gi
        maxAllowed:
          cpu: 8
          memory: 32Gi
```

**VPA 的三种模式**：

| 模式 | 行为 | 适用场景 |
|------|------|------|
| `Off` | 只推荐，不做任何修改 | 先观察 VPA 建议是否合理 |
| `Initial` | 只在 Pod 创建时设置，不更新已运行的 | 避免运行中的 Pod 被重启 |
| `Auto` | 自动调整，需要重启 Pod | 完全自动化 |

> **HPA 和 VPA 不能同时基于 CPU 指标**：两者都调整同一个资源会互相冲突。常见组合：HPA 基于 CPU + VPA 基于内存，或 HPA 基于自定义指标 + VPA 基于 CPU/内存。

### 5.3 HPA vs VPA 对比

| 特性 | HPA | VPA |
|------|------|------|
| 调整什么 | Pod 数量 | 单 Pod 的 CPU/内存 |
| 需要重启 Pod 吗 | 不需要 | 需要（Auto 模式） |
| 适用工作负载 | 无状态（Deployment） | 无状态和有状态 |
| 前提 | 应用支持水平扩展 | — |
| 典型场景 | Web API（流量波动） | 难以预估资源需求的服务 |

---

## 六、Cluster Autoscaler 与 Karpenter（深入）

HPA/VPA 调整的是 Pod 级别，但如果整个集群的节点资源都不够了呢？需要节点级别的自动扩缩容。

### 6.1 Cluster Autoscaler

```
Pod Pending（Scheduler 找不到合适的节点）
    ↓ Cluster Autoscaler 检测到
在 Auto Scaling Group 中添加新 EC2 实例
    ↓ 新节点加入集群
Pending Pod 被调度到新节点
```

**局限性**：
- 基于 Auto Scaling Group（ASG），节点类型固定
- 扩容慢（分钟级）：需要创建 EC2 实例、加入集群、拉取镜像
- 一个 ASG 只能用一种实例类型

### 6.2 Karpenter（推荐）

Karpenter 是 AWS 开源的下一代节点自动扩缩容，相比 Cluster Autoscaler 更智能：

```
Pod Pending
    ↓ Karpenter 检测到
分析 Pending Pod 的资源需求
    ↓
选择最优实例类型（从配置的候选列表中）
    ↓
直接调用 EC2 API 创建实例（不依赖 ASG）
    ↓ 更快
新节点加入集群，Pod 被调度
```

**Karpenter vs Cluster Autoscaler**：

| 特性 | Cluster Autoscaler | Karpenter |
|------|------|------|
| 扩容速度 | 分钟级 | 秒级（直接调 EC2 API） |
| 实例选择 | 固定（ASG 配置） | 动态最优（从候选列表选最便宜的能满足需求的） |
| 缩容 | 节点空闲后缩容 | 主动整合（合并分散的 Pod，减少节点数） |
| 配置方式 | ASG + Launch Template | NodePool + EC2NodeClass |
| Spot 支持 | 有限 | 优秀（自动 fallback 到其他实例类型） |

**Karpenter NodePool 示例**：

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]     # 允许 Spot 实例（省钱）
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000                   # NodePool 的 CPU 总上限
    memory: 2000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized   # 利用率低时自动整合节点
    expireAfter: 720h           # 节点最长存活 30 天（强制轮换）
```

---

## 七、资源管理最佳实践

### 7.1 资源配置策略

| 环境 | CPU requests | CPU limits | Memory requests | Memory limits | QoS |
|------|------|------|------|------|------|
| **生产（核心）** | 8 | 8 | 16Gi | 16Gi | Guaranteed |
| **生产（一般）** | 1 | 4 | 2Gi | 4Gi | Burstable |
| **Staging** | 500m | 2 | 1Gi | 2Gi | Burstable |
| **开发/测试** | 100m | 1 | 256Mi | 512Mi | Burstable |

### 7.2 如何确定合适的 resources

```bash
# 1. 用 VPA 的推荐模式观察（最准确）
# 安装 VPA 后设为 Off 模式，观察推荐值

# 2. 查看实际使用量
kubectl top pods -n plaud-project-summary
# NAME                                    CPU(cores)   MEMORY(bytes)
# api-deployment-7d8f9-xk2pq             1200m        3.2Gi

# 3. 用 Prometheus 查询历史数据
# P95 CPU：quantile_over_time(0.95, container_cpu_usage_seconds_total[7d])
# P95 Memory：quantile_over_time(0.95, container_memory_working_set_bytes[7d])
```

**经验法则**：
- `requests` = P95 使用量（覆盖 95% 的场景）
- `limits` = requests × 1.5~2（给突发留空间）
- 内存 `limits` 不要设太高——OOMKilled 比慢慢内存泄漏更好发现

### 7.3 自动扩缩容配置建议

```
                              流量波动大？
                                │
                    ┌───────────┴───────────┐
                    │ 是                     │ 否
                    ▼                       ▼
              启用 HPA                考虑固定副本数
           minReplicas=2             replicaCount=N
           maxReplicas=10~20
           CPU 目标 70-80%
                    │
           节点资源够吗？
                    │
              ┌─────┴─────┐
              │ 是         │ 否
              ▼           ▼
          完成       启用 Karpenter
                   配置 NodePool
                   允许 Spot 实例
```

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — resources、HPA 的实际配置
- [[05-k8s-architecture|K8s 架构原理]] — Scheduler 的过滤和打分机制
- [[12-k8s-pod-graceful-shutdown|Pod 优雅终止]] — 缩容时的 Pod 终止流程
- [[03-k8s-workload-types|K8s 工作负载类型]] — DaemonSet 与 Taints/Tolerations
- [[08-k8s-security-rbac|K8s 安全与权限]] — ResourceQuota 在安全中的角色
