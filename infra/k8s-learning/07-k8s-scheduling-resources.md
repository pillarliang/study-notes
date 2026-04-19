# K8s 调度与资源管理

> 前置知识：[[05-k8s-architecture#3.3 Scheduler — "Pod 应该跑在哪个节点"|Scheduler 基本调度流程]]、[[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|Helm values 中 resources 与 HPA 配置]]。

Pod 创建后，Scheduler 决定它跑在哪个节点。这个决策依赖三层信息：Pod 需要多少资源（资源模型）、哪些节点满足条件（调度策略）、流量变化时如何动态调整（自动扩缩容）。本文沿着这条线展开。

---

## 一、资源模型：requests 与 limits

> [!question] Pod 需要多少 CPU 和内存？超了怎么办？

每个容器可以声明两组资源参数：`requests`（最少需要多少）和 `limits`（最多允许用多少）。两者分别影响调度和运行时行为。

### 1.1 requests 与 limits 的含义

```yaml
# 来源：通用示例
resources:
  requests:          # Scheduler 据此选节点——"至少需要这么多"
    cpu: 2
    memory: 4Gi
  limits:            # kubelet 据此限制使用——"最多用这么多"
    cpu: 4
    memory: 8Gi
```

| 字段 | 消费者 | 作用 | 超出后果 |
|------|--------|------|----------|
| **requests** | Scheduler | 选择有足够剩余资源的节点 | — |
| **limits** | kubelet (cgroups) | 限制容器实际可用资源 | CPU 被限流；内存超出则 OOMKilled |

### 1.2 CPU 与内存的关键差异

| 资源 | 可压缩 | 超过 limits 的后果 | 单位 |
|------|--------|-------------------|------|
| **CPU** | 是 | 不会被杀，但被限流（进程变慢） | `1` = 1 核，`500m` = 0.5 核 |
| **内存** | 否 | OOMKilled（容器被杀重启） | `Mi`（MiB）、`Gi`（GiB） |

> [!tip] CPU Throttling——延迟毛刺的常见元凶
> 当容器在一个 CPU 调度周期（默认 100ms）内使用量达到 limits，剩余时间会被暂停。表现为：延迟突然增大，但 CPU 利用率看起来"不高"。如果 Pod 出现延迟毛刺且 CPU 使用率未达 limits，多半是 throttling。

### 1.3 实战：plaud-project-summary 的资源配置

```yaml
# 来源：plaud-project-summary/values/ap-northeast-1/prod/main.yaml
deployer:
  resources:
    limits:   { cpu: 8, memory: 16Gi }
    requests: { cpu: 8, memory: 16Gi }   # requests = limits → Guaranteed QoS
```

这是一个 CPU 密集型 AI 总结服务的生产配置。`requests = limits` 让 Pod 独占所申请的资源，获得最高优先级的 QoS 等级（下节详述）。

---

## 二、QoS 等级与驱逐顺序

> [!question] 节点资源不足时，K8s 先杀哪些 Pod？

K8s 根据 requests/limits 的配置，自动为 Pod 分配 QoS（Quality of Service）等级，决定资源紧张时的驱逐优先级。

### 2.1 三种 QoS 等级

| QoS 等级 | 条件 | 驱逐优先级 | 适用场景 |
|-----------|------|-----------|----------|
| **Guaranteed** | 所有容器 requests = limits（CPU + 内存都设） | 最后被驱逐 | 生产核心服务 |
| **Burstable** | 至少一个容器设了 requests 或 limits，但不全等 | 中间 | 一般工作负载 |
| **BestEffort** | 未设任何 requests 和 limits | 最先被驱逐 | 测试/临时任务 |

### 2.2 判定流程

```
Pod 中每个容器都满足 requests == limits（CPU + Memory）？
  ├── 是 → Guaranteed
  └── 否 → 至少一个容器设了 requests 或 limits？
              ├── 是 → Burstable
              └── 否 → BestEffort
```

### 2.3 驱逐顺序

节点内存不足时，kubelet 按以下顺序驱逐：

```
节点内存压力
  ↓
1. 驱逐 BestEffort Pod（先杀实际内存使用最多的）
  ↓ 仍不足
2. 驱逐 Burstable Pod（先杀超出 requests 比例最多的）
  ↓ 仍不足
3. 驱逐 Guaranteed Pod（极端情况，几乎不会发生）
```

plaud-project-summary 设 `requests = limits` 的原因正在于此——确保 Guaranteed QoS，在资源竞争时最后被影响。

---

## 三、调度策略

> [!question] 如何控制 Pod 被调度到哪些节点、如何分布？

Scheduler 在分配节点时，除了看资源是否足够，还可以通过标签选择、亲和性规则和拓扑分布约束来精细控制。复杂度递增：nodeSelector → affinity → topology spread constraints。

### 3.1 nodeSelector——最简单的节点选择

通过节点标签做硬性匹配：

```yaml
# 来源：通用示例
spec:
  nodeSelector:
    node-type: gpu          # 只调度到有此 label 的节点
```

实战中，plaud-file 用 `nodeSelector` 限定工作负载节点类型：

```yaml
# 来源：plaud-file/values/ap-northeast-1/prod/values-main.yaml
nodeSelector:
  workload: general         # 只调度到 general 工作负载节点
```

### 3.2 Node Affinity——更灵活的节点选择（进阶）

Node Affinity 支持 `In`、`NotIn`、`Exists` 等操作符，并区分硬性（required）和软性（preferred）：

```yaml
# 来源：通用示例
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:        # 硬性：必须满足
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: ["ap-northeast-1a", "ap-northeast-1c"]
    preferredDuringSchedulingIgnoredDuringExecution:       # 软性：尽量满足
      - weight: 80
        preference:
          matchExpressions:
            - key: node-type
              operator: In
              values: ["compute-optimized"]
```

#### 实战：WebSocket 服务的 Taint + NodeAffinity 双保险（进阶）

plaud-ws-notify（WebSocket 长连接通知服务）需要运行在专用节点上，避免与普通服务争抢资源：

```yaml
# 来源：plaud-ws-notify/values/us-west-2/prod/values-main.yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "websockit"
    effect: "NoSchedule"

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: workload
              operator: In
              values: ["websockit"]
```

Taint 确保普通 Pod 不会被调度到 WebSocket 专用节点；NodeAffinity 确保 WebSocket Pod 只会被调度到这些专用节点。两者配合实现"专用节点池"。

### 3.3 Pod Anti-Affinity——让同类 Pod 分散（进阶）

避免同一 Deployment 的多个 Pod 挤在同一节点或可用区：

```yaml
# 来源：plaud-file/values/ap-northeast-1/prod/values-main.yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:    # 硬性约束
      - labelSelector:
          matchLabels:
            app_name: plaud-file
        topologyKey: kubernetes.io/hostname            # 同一节点不能有两个 plaud-file Pod
```

plaud-file 使用 hostPath 挂载本地磁盘做临时存储，同一节点跑多个副本会导致路径冲突。`requiredDuring...` + `kubernetes.io/hostname` 确保每个节点最多一个 Pod。

### 3.4 Topology Spread Constraints——均匀分布（进阶）

比 Anti-Affinity 更精确地控制 Pod 在拓扑域中的分布——不只是"不在一起"，而是"均匀摊开"。

```yaml
# 来源：generic-deployer/templates/deployment.yaml（默认注入）
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone      # 按可用区均匀分布
    whenUnsatisfiable: ScheduleAnyway             # 软性：尽量均匀但不阻止调度
    labelSelector:
      matchLabels: ...
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname            # 按节点均匀分布
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels: ...
```

generic-deployer 模板为所有服务内置了**两层拓扑分布**：先按 AZ 打散，再按节点打散，确保 Pod 不会集中在某个 AZ 或某台机器上。`ScheduleAnyway`（软性）在节点不足时不会阻塞部署。

**Topology Spread vs Anti-Affinity**：

| 特性 | Anti-Affinity | Topology Spread |
|------|---------------|-----------------|
| 语义 | "不在同一个 X 上" | "各 X 之间最多差 N 个" |
| 效果 | 3 AZ 3 Pod → 每 AZ 1 个（理想） | 3 AZ 5 Pod → 2/2/1（maxSkew=1） |
| 可扩展性 | Pod 数 > 节点数时无法调度 | maxSkew 允许一定不均衡 |

### 3.5 Taints & Tolerations——节点"驱赶"（进阶）

Taint 标记在节点上，表示"除非能容忍，否则不要过来"；Toleration 标记在 Pod 上，声明"可以容忍某种污点"。

```bash
# 给节点打 taint
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
```

```yaml
# 来源：通用示例——Pod 声明容忍
tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
```

三种 Taint 效果：

| Effect | 行为 |
|--------|------|
| `NoSchedule` | 新 Pod 不会被调度到该节点 |
| `PreferNoSchedule` | 尽量不调度，实在没地方了仍会调度 |
| `NoExecute` | 已运行的 Pod 如无容忍也会被驱逐 |

常见场景：GPU 节点打 taint 保证只有 GPU 任务过来；Master 节点默认 `NoSchedule` 防止工作负载上跑；维护时打 `NoExecute` 清空节点。

### 3.6 调度策略选型速查

```
场景                          推荐策略
───────────────────────────   ──────────────────────────
固定到某类节点（如 GPU）       nodeSelector
多条件节点选择 + 软硬性区分    Node Affinity
同类 Pod 不放同一节点/AZ      Pod Anti-Affinity
Pod 在 AZ/节点间均匀分布      Topology Spread Constraints
专用节点池（隔离工作负载）     Taint + Toleration + Affinity
```

---

## 四、HPA——自动水平扩缩容

> [!question] 流量突增时手动加副本来不及，怎么办？

HPA（Horizontal Pod Autoscaler）根据指标自动调整 Deployment 的副本数。

### 4.1 工作原理

```
每 15 秒（默认周期）：
1. 从 Metrics Server 获取当前指标（如 CPU 使用率）
2. 计算期望副本数 = 当前副本数 x (当前使用率 / 目标使用率)
   例：3 Pod，CPU = 90%，目标 = 75%
   期望 = ceil(3 x 90/75) = ceil(3.6) = 4
3. 调整 Deployment replicas = 4
```

### 4.2 实战：plaud-project-summary 的 HPA 配置

```yaml
# 来源：plaud-project-summary/values/ap-northeast-1/prod/main.yaml
autoscaling:
  enabled: true
  minReplicas: 2                          # 最少 2 副本（高可用）
  maxReplicas: 10                         # 最多 10 副本
  targetCPUUtilizationPercentage: 75      # CPU > 75% 触发扩容
  targetMemoryUtilizationPercentage: 70   # Memory > 70% 触发扩容
```

generic-deployer 的 HPA 模板将上述 values 渲染为标准 `autoscaling/v2` 资源：

```yaml
# 来源：generic-deployer/templates/hpa.yaml（渲染后）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: plaud-project-summary
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 75 }
```

staging 环境中 HPA 关闭，固定 1 副本节省成本——生产需要弹性应对流量波动，staging 只需保证功能可用。

### 4.3 扩容与缩容的行为差异

| 行为 | 速度 | 原因 |
|------|------|------|
| 扩容 | 快（默认无延迟） | 宁可多花钱也不能影响用户体验 |
| 缩容 | 慢（默认 5 分钟稳定窗口） | 避免流量波动导致反复扩缩（flapping） |

### 4.4 VPA——垂直自动扩缩容（进阶）

HPA 调整副本数（水平），VPA 调整单个 Pod 的 resources（垂直）：

```yaml
# 来源：通用示例
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: plaud-api
  updatePolicy:
    updateMode: "Off"        # 只推荐，不修改——先观察再决定
```

VPA 三种模式：

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `Off` | 只推荐，不修改 | 先观察建议是否合理 |
| `Initial` | 只在 Pod 创建时设置 | 避免运行中 Pod 被重启 |
| `Auto` | 自动调整（需重启 Pod） | 完全自动化 |

> [!warning] HPA 和 VPA 不能同时基于 CPU 指标——两者都调整同一维度会互相冲突。常见组合：HPA 基于 CPU + VPA 基于内存，或 HPA 基于自定义指标 + VPA 基于 CPU/内存。

---

## 五、Karpenter——节点级自动扩缩（进阶）

> [!question] HPA 扩出新 Pod，但集群节点资源已用尽，Pod 一直 Pending 怎么办？

HPA/VPA 解决 Pod 级别的扩缩，节点级别需要 Cluster Autoscaler 或 Karpenter。

### 5.1 Cluster Autoscaler vs Karpenter（进阶）

| 特性 | Cluster Autoscaler | Karpenter |
|------|-------------------|-----------|
| 扩容速度 | 分钟级（通过 ASG） | 秒级（直接调 EC2 API） |
| 实例选择 | 固定（ASG 配置） | 动态最优（从候选列表选最便宜的能满足需求的） |
| 缩容 | 节点空闲后缩容 | 主动整合（合并分散的 Pod，减少节点） |
| Spot 支持 | 有限 | 优秀（自动 fallback 到其他实例类型） |

### 5.2 Karpenter NodePool 示例（进阶）

```yaml
# 来源：通用示例
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
          values: ["on-demand", "spot"]        # 允许 Spot 实例
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
  limits:
    cpu: 1000
    memory: 2000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized     # 利用率低时自动整合节点
    expireAfter: 720h                          # 节点最长存活 30 天（强制轮换）
```

Karpenter 的核心流程：检测 Pending Pod → 分析资源需求 → 从候选实例中选最优 → 直接调 EC2 API 创建 → 新节点加入集群 → Pod 被调度。

---

## 六、最佳实践

### 6.1 资源配置策略

| 环境 | CPU req/lim | Memory req/lim | QoS | 说明 |
|------|-------------|----------------|-----|------|
| 生产（核心） | 8 / 8 | 16Gi / 16Gi | Guaranteed | 独占资源，不被驱逐 |
| 生产（一般） | 1 / 4 | 2Gi / 4Gi | Burstable | 留突发余量 |
| Staging | 500m / 2 | 1Gi / 2Gi | Burstable | 功能验证即可 |
| 开发/测试 | 100m / 1 | 256Mi / 512Mi | Burstable | 省成本 |

### 6.2 如何确定合适的 resources

```bash
# 查看 Pod 实时资源使用
kubectl top pods -n production

# 用 VPA Off 模式观察推荐值（最准确）

# 用 Prometheus 查询 7 天 P95
# CPU：quantile_over_time(0.95, container_cpu_usage_seconds_total[7d])
# Memory：quantile_over_time(0.95, container_memory_working_set_bytes[7d])
```

经验法则：
- `requests` = P95 使用量（覆盖 95% 场景）
- `limits` = requests x 1.5~2（给突发留空间）
- 内存 `limits` 不要过高——OOMKilled 比慢性内存泄漏更容易发现和定位

### 6.3 自动扩缩容决策路径

```
流量波动大？
  ├── 是 → 启用 HPA（min=2, max=10~20, CPU 目标 70-80%）
  │          └── 节点资源够吗？
  │               ├── 够 → 完成
  │               └── 不够 → 启用 Karpenter（配置 NodePool + 允许 Spot）
  └── 否 → 固定副本数（replicaCount=N）
```

### 6.4 LimitRange 与 ResourceQuota（集群管理）（深入）

防止开发者忘记设 resources 或某个 namespace 占用过多集群资源：

```yaml
# 来源：通用示例——namespace 级默认值
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:        { cpu: "1", memory: 1Gi }       # 没设 limits 时的默认值
      defaultRequest: { cpu: 500m, memory: 512Mi }     # 没设 requests 时的默认值
```

```yaml
# 来源：通用示例——namespace 级总量限制
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    pods: "50"
```

---

## 延伸阅读

- [[05-k8s-architecture|K8s 架构原理]] — Scheduler 的过滤和打分机制
- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — resources、HPA 的实际配置
- [[12-k8s-pod-graceful-shutdown|Pod 优雅终止]] — 缩容时的 Pod 终止流程
- [[03-k8s-workload-types|K8s 工作负载类型]] — DaemonSet 与 Taints/Tolerations
- [[08-k8s-security-rbac|K8s 安全与权限]] — ResourceQuota 在安全中的角色
