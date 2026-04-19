# K8s 存储

> 前置知识：了解 [[02-k8s-core-concepts|Pod 的基本概念]] 和 [[01-docker-basics#7.6 Volume 详解|Docker Volume]]。
>
> StatefulSet 的独立持久存储（`volumeClaimTemplates`）依赖本文的 PV/PVC 机制，详见 [[03-k8s-workload-types#三、StatefulSet — 有状态应用|StatefulSet]]。

---

## 一、问题：Pod 重启数据就丢了

容器的文件系统是临时的——容器重启后，写入的文件全部消失。对无状态微服务而言这无所谓，但以下场景必须解决数据持久化问题：

| 场景 | 具体需求 | 能否接受数据丢失 |
| ------ | ------ | ------ |
| 临时日志 | 同 Pod 内多容器共享日志目录，由 Fluent Bit 采集 | 可以（采集后丢弃） |
| 节点文件访问 | DaemonSet 读取宿主机 `/var/log` | 不涉及持久化 |
| 消息队列 | Kafka 分区数据需要跨重启保留 | 不能 |
| 数据库 | MongoDB 副本集每节点 3Ti 数据 | 不能 |

K8s 针对这些场景提供了从临时到持久的完整存储方案：

```text
临时 ←————————————————————————————→ 持久

emptyDir     hostPath     PV/PVC + StorageClass
(Pod 级)     (节点级)     (独立于 Pod 生命周期)
```

下面按复杂度递进，逐一介绍每种方案。

---

## 二、emptyDir——Pod 内临时共享

### 2.1 什么是 emptyDir

`emptyDir` 在 Pod 被调度到节点时创建一个空目录，Pod 内所有容器都可以读写该目录。**Pod 删除后目录和数据一起消失**。

适用场景：

- 同 Pod 内多个容器共享临时文件
- 应用运行时的缓存/暂存区
- 扩展容器的 `/dev/shm`（共享内存）

### 2.2 实战：plaud-project-summary cn-northwest-1 的日志卷

`plaud-project-summary` 是无状态服务，使用 `emptyDir` 存放临时日志——日志由 Fluent Bit sidecar 采集到集中存储后即可丢弃：

```yaml
# 来源：deploy/plaud-project-summary/values/cn-northwest-1/prod/main.yaml
deployer:
  volumeMounts:
    - name: logs
      mountPath: /data/plaud-sync/logs   # 应用写日志到此路径
  volumes:
    - name: logs
      emptyDir: {}                       # Pod 删除后日志消失，无影响
```

为什么选 `emptyDir` 而非 PVC？该服务是无状态的，日志是过程数据而非业务数据。使用 PVC 会引入不必要的存储管理开销。

### 2.3 emptyDir 的另一个用法：扩展共享内存（进阶）

容器默认的 `/dev/shm` 只有 64MB，对需要大量内存映射的应用（OpenSearch、数据库）不够用。通过 `emptyDir` 挂载到 `/dev/shm` 可以扩展：

```yaml
# 来源：deploy 项目 infra/values/opensearch-data/default.yaml
extraVolumes:
  - name: shm-volume
    emptyDir:
      sizeLimit: 1Gi             # 限制大小为 1Gi
extraVolumeMounts:
  - name: shm-volume
    mountPath: /dev/shm          # 替换默认的 64MB 共享内存
```

---

## 三、hostPath——访问节点文件系统

`hostPath` 将宿主机的目录或文件挂载到 Pod 中，数据生命周期与节点绑定。

```yaml
# 示例：DaemonSet 读取节点日志
volumes:
  - name: host-logs
    hostPath:
      path: /var/log
      type: Directory
```

典型场景是 DaemonSet（如日志采集器、监控 agent）需要访问节点文件系统。

> [!warning] hostPath 的限制
>
> - Pod 重新调度到其他节点后，无法访问原节点的数据
> - 存在安全风险（容器可读写宿主机文件），生产环境应严格限制使用
> - 不适合任何需要数据持久化的业务场景

---

## 四、PV / PVC / StorageClass——持久存储

### 4.1 为什么需要这层抽象

`emptyDir` 和 `hostPath` 都无法满足"Pod 删除后数据仍然保留"的需求。K8s 引入三个概念来解决持久存储问题：

| 概念 | 角色 | 类比 |
| ------ | ------ | ------ |
| **PersistentVolume (PV)** | 一块实际的存储资源 | 一块物理硬盘 |
| **PersistentVolumeClaim (PVC)** | 对存储的申请 | "需要一块 50Gi 的硬盘" |
| **StorageClass** | 描述存储的"规格"，支持动态创建 PV | 硬盘的品牌和型号 |

这种设计将**存储供给**和**存储消费**解耦——开发者只需声明"需要多大空间、什么类型"，不必关心底层是 EBS、EFS 还是 NFS：

```text
┌── 集群管理员 / StorageClass ──┐    ┌── 开发者 ──────────────┐
│                              │    │                        │
│  PV                          │    │  PVC                   │
│  "有一块 100Gi gp3 EBS 卷"    │←绑定→│  "需要 50Gi 存储"       │
│                              │    │                        │
│  具体参数：                    │    │  Pod 引用 PVC：         │
│  - 存储后端、卷 ID             │    │    volumes:            │
│  - 容量、访问模式              │    │      - persistentVolume│
│  - 回收策略                   │    │        Claim:          │
│                              │    │          claimName: xxx │
└──────────────────────────────┘    └────────────────────────┘
```

### 4.2 静态供给 vs 动态供给

- **静态供给**：管理员预先创建 PV，开发者创建 PVC 去"认领"
- **动态供给**：开发者创建 PVC 时指定 StorageClass，K8s 自动创建 PV 和底层存储

```text
静态：管理员创建 EBS 卷 → 创建 PV → 开发者创建 PVC → 匹配绑定
动态：开发者创建 PVC（指定 StorageClass）→ 自动创建 EBS + PV → 绑定
```

生产环境几乎都用动态供给——管理员只需定义好 StorageClass，开发者自助申请即可。

### 4.3 StorageClass 定义

```yaml
# 示例：EKS 中定义 gp3 加密 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com        # CSI 驱动（见第五节）
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete                # PVC 删除时自动清理 PV 和 EBS
volumeBindingMode: WaitForFirstConsumer  # 等 Pod 调度后再创建卷（保证同 AZ）
allowVolumeExpansion: true           # 允许后续扩容
```

关键参数：

| 参数 | 含义 |
| ------ | ------ |
| `provisioner` | 指定用哪个 CSI 驱动创建存储 |
| `reclaimPolicy` | PVC 删除后处理方式：`Delete`（删除）或 `Retain`（保留） |
| `volumeBindingMode` | `WaitForFirstConsumer`（推荐，等 Pod 调度后创建，避免跨 AZ 问题） |
| `allowVolumeExpansion` | 是否允许扩大 PVC 容量（只能扩不能缩） |

### 4.4 PV → PVC → Pod 的完整绑定流程

```text
PVC 创建（指定 StorageClass + 容量 + 访问模式）
    ↓
K8s 查找匹配的 PV（StorageClass 相同、容量 >= 请求、访问模式兼容）
    ↓
  找到 → PV 与 PVC 一对一绑定，PVC 状态变为 Bound
  没找到 → 若有 StorageClass，动态创建 PV；否则 PVC 保持 Pending
    ↓
Pod 引用 PVC → kubelet 调用 CSI 驱动 → 底层存储挂载到容器
```

### 4.5 访问模式（Access Modes）（进阶）

| 访问模式 | 缩写 | 含义 | 典型存储 |
| ------ | ------ | ------ | ------ |
| **ReadWriteOnce** | RWO | 只能被**一个节点**挂载为读写 | EBS |
| **ReadOnlyMany** | ROX | 可被多节点挂载为只读 | EFS、NFS |
| **ReadWriteMany** | RWX | 可被多节点挂载为读写 | EFS、NFS、FSx |
| **ReadWriteOncePod** | RWOP | 只能被**一个 Pod** 挂载为读写（K8s 1.27+） | 部分 CSI 驱动 |

> [!important] EBS 只支持 RWO
> EBS 块存储只能挂载到一个 EC2 实例。如果需要多 Pod 并发读写同一份数据，应使用 EFS（RWX）或 FSx。

### 4.6 实战：Kafka JBOD 100Gi → 1Ti

Kafka 使用 JBOD（Just a Bunch of Disks）模式，每个 broker 节点独立挂载持久磁盘。基础配置 100Gi，生产环境覆盖为 1Ti：

```yaml
# 来源：deploy 项目 infra/values/strimzi-kafka/base/KafkaNodePool.yaml
storage:
  type: jbod
  volumes:
    - id: 0
      type: persistent-claim
      size: 100Gi              # 基础 100Gi
      deleteClaim: false       # 节点删除时保留 PVC，防止消息丢失
      kraftMetadata: shared
```

```yaml
# 来源：deploy 项目 overlays/global/prod（生产覆盖）
storage:
  type: jbod
  volumes:
    - id: 0
      type: persistent-claim
      size: 1Ti                # 生产扩大到 1Ti
```

`deleteClaim: false` 确保即使 Kafka Pod 被删除，底层 PVC 和 EBS 卷也会保留——这与 StorageClass 的 `Retain` 回收策略思路一致。

### 4.7 实战：MongoDB 3Ti gp3-encrypted

MongoDB 副本集通过 StatefulSet 的 `volumeClaimTemplates` 为每个副本分配独立存储：

```yaml
# 来源：MongoDB StatefulSet volumeClaimTemplates
volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: ["ReadWriteOnce"]     # EBS 块存储，单节点挂载
      storageClassName: gp3-encrypted
      resources:
        requests:
          storage: 3Ti                   # 每副本 3Ti
```

这里的关键设计决策：

- **RWO + EBS**：每个 MongoDB 副本独占一块 EBS 卷，副本间通过 MongoDB 自身的复制机制同步数据
- **gp3-encrypted**：使用加密的 gp3 卷，兼顾性能与安全合规
- **3Ti**：MongoDB 存储的是核心业务数据，预留充足空间避免频繁扩容

### 4.8 回收策略与卷扩容（进阶）

**回收策略（Reclaim Policy）**：PVC 删除时，PV 和底层存储如何处理。

| 策略 | 行为 | 适用场景 |
| ------ | ------ | ------ |
| **Delete** | PV 和底层存储一起删除 | 临时数据、可重建的内容 |
| **Retain** | PV 变为 Released，底层存储保留 | 重要数据（数据库等） |

> [!tip] 生产建议
> 数据库等关键数据使用 `Retain` 策略，防止误删 PVC 导致数据丢失。配合 EBS Snapshot 定期备份。

**卷扩容**：PVC 创建后发现空间不足，可在线扩容（前提：StorageClass 设置了 `allowVolumeExpansion: true`）：

```bash
# 修改 PVC 容量（只能扩，不能缩）
kubectl patch pvc postgres-data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# 查看扩容状态
kubectl get pvc postgres-data
```

EBS CSI 驱动会自动处理文件系统扩展（ext4/xfs 在线扩展）。

---

## 五、CSI 和 EBS（进阶）

### 5.1 CSI 是什么

CSI（Container Storage Interface）是 K8s 的存储插件标准接口，类似网络领域的 CNI。存储厂商实现 CSI 驱动后，K8s 即可对接各种存储后端，无需修改核心代码。

```text
PVC → StorageClass（指定 provisioner）→ CSI Controller 创建卷
                                          ↓
Pod 调度到节点 → kubelet 调用 CSI Node 插件 → 挂载卷到 Pod
```

### 5.2 EKS 常用 CSI 驱动

| CSI 驱动 | 安装方式 | 提供的存储 |
| ------ | ------ | ------ |
| **EBS CSI Driver** | EKS 托管插件 | EBS 块存储（gp3、io2 等），RWO |
| **EFS CSI Driver** | EKS 托管插件 | EFS 共享文件存储，RWX |
| **FSx CSI Driver** | 手动安装 | FSx for Lustre / OpenZFS |

```bash
# 查看集群安装的 CSI 驱动
kubectl get csidrivers
# NAME                  ATTACHREQUIRED   PODINFOONMOUNT
# ebs.csi.aws.com       true             false
# efs.csi.aws.com       false            false
```

### 5.3 EKS 常用 StorageClass

| StorageClass | 底层存储 | 特点 | 适用场景 |
| ------ | ------ | ------ | ------ |
| **gp3**（推荐） | EBS gp3 | 基线 3000 IOPS / 125 MB/s，可独立调整 | 通用工作负载 |
| **gp2** | EBS gp2 | IOPS 与容量绑定（3 IOPS/GiB） | 旧集群默认 |
| **io2** | EBS io2 | 最高 64000 IOPS，99.999% 可用性 | 高性能数据库 |
| **efs-sc** | EFS | NFS 协议，多 Pod 共享读写 | 共享文件、ML 数据集 |

---

## 六、存储选型总结

| 存储类型 | 生命周期 | 数据持久性 | 适用场景 | 项目中的例子 |
| ------ | ------ | ------ | ------ | ------ |
| **emptyDir** | Pod 级 | Pod 删除即丢失 | 临时日志、缓存、扩展 /dev/shm | plaud-project-summary 日志卷 |
| **hostPath** | 节点级 | 调度到其他节点不可见 | DaemonSet 读节点日志 | — |
| **PVC (RWO)** | 独立于 Pod | 持久 | 数据库、消息队列 | Kafka 1Ti、MongoDB 3Ti |
| **PVC (RWX)** | 独立于 Pod | 持久 | 多 Pod 共享读写 | audio-gateway FSx OpenZFS |
| **ConfigMap/Secret** | 与资源相同 | 集群存在即在 | 配置文件注入 | config.json 挂载 |

选型决策路径：

```text
数据是否需要跨 Pod 重启保留？
├── 否 → emptyDir（临时缓存/日志）
└── 是 → 是否需要多 Pod 并发读写？
    ├── 否 → PVC + EBS（RWO）
    └── 是 → PVC + EFS/FSx（RWX）
```

> [!tip] 最佳实践
> 无状态微服务尽量不依赖持久存储。数据库等有状态服务优先考虑云托管方案（RDS、ElastiCache），在 K8s 上自管有状态工作负载会显著增加运维复杂度。

---

## 七、存储排障速查（深入）

```bash
# PVC 状态（Pending = 未绑定到 PV）
kubectl get pvc -n <namespace>

# PVC 详情（Events 包含失败原因）
kubectl describe pvc <name> -n <namespace>

# 查看 PV 和 StorageClass
kubectl get pv
kubectl get storageclass

# 确认 CSI 驱动运行正常
kubectl get pods -n kube-system | grep ebs-csi
```

常见问题：

| 现象 | 可能原因 |
| ------ | ------ |
| PVC Pending: "no persistent volumes available" | StorageClass 名称不匹配 |
| PVC Pending: "waiting for first consumer" | `WaitForFirstConsumer` 模式，等 Pod 调度后自动创建，属正常状态 |
| Pod 挂载失败: "Multi-Attach error" | EBS 被另一节点占用（RWO 限制），需等旧 Pod 完全终止 |

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — emptyDir 和 volumeMounts 的实际 values 配置
- [[03-k8s-workload-types|K8s 工作负载类型]] — StatefulSet 的 volumeClaimTemplates 机制
- [[05-k8s-architecture|K8s 架构原理]] — kubelet 如何调用 CSI 驱动挂载卷
- [[01-docker-basics|Docker 学习笔记]] — Docker Volume 与 K8s PV/PVC 的对比
