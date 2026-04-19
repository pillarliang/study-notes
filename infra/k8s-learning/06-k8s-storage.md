# K8s 存储

> 前置知识：了解 [[02-k8s-core-concepts|Pod 的基本概念]]和 [[01-docker-basics#7.6 Volume 详解|Docker Volume]]。本文解释 K8s 中如何为 Pod 提供持久化存储——你已经用过 `emptyDir`，这里介绍完整的存储体系。
>
> StatefulSet 的独立持久存储（`volumeClaimTemplates`）依赖本文介绍的 PV/PVC 机制，详见 [[03-k8s-workload-types#三、StatefulSet — 有状态应用|StatefulSet]]。
>
> **学习建议**：首次阅读重点关注第一至二节（存储全景、PV/PVC）、第四节（访问模式）和第八至九节（项目关联、排障）。StorageClass 定义（3.2–3.3）、卷扩容（六）、回收策略（七）为进阶，CSI（五）为深入。

---

## 一、存储类型全景

| 存储类型 | 生命周期 | 数据持久性 | 适用场景 |
|------|------|------|------|
| **emptyDir** | 与 Pod 相同 | Pod 删除即丢失 | 临时缓存、同 Pod 容器间共享 |
| **hostPath** | 与节点相同 | Pod 调度到其他节点则不可见 | DaemonSet 读节点日志 |
| **PersistentVolume (PV)** | 独立于 Pod | Pod 删除后数据保留 | 数据库、文件存储 |
| **ConfigMap / Secret** | 与资源相同 | 集群存在即在 | 配置文件注入 |

在本项目中，大多数微服务是无状态的，使用 `emptyDir` 存放临时日志（见 [[10-helm-argocd-deployment#3.5 ⑤⑥⑦ Values 文件详解 — 参数逐项说明|values 文件的 volumes 配置]]）。但当你需要在 K8s 上运行数据库或其他有状态服务时，就需要 PV/PVC。

> [!tip] 🔗 实战链接：plaud-project-summary 的 emptyDir 存储
> `plaud-project-summary` 是无状态服务，使用 `emptyDir` 存放临时日志：
>
> ```yaml
> # plaud-project-summary/values/ap-northeast-1/prod/main.yaml
> volumeMounts:
>   - name: logs
>     mountPath: /data/plaud-sync/logs    # 应用将日志写入此路径
> volumes:
>   - name: logs
>     emptyDir: {}                        # Pod 级临时卷，Pod 删除后日志消失
> ```
>
> | 存储类型 | 项目中的例子 | 使用场景 |
> | ------ | ------ | ------ |
> | **emptyDir** | `plaud-project-summary` 的日志目录 | 临时日志，由 Fluent Bit 采集后即可丢弃 |
> | **PVC** | OpenSearch data 节点的 500Gi 持久存储 | 有状态服务需要数据持久化 |
> | **ConfigMap** | `plaud-project-summary` 的 config.json 注入 | 应用配置文件 |
>
> 这体现了"按需选择最合适的存储类型"原则——`plaud-project-summary` 作为无状态服务只需 emptyDir，而数据库类服务才需要 PVC。

---

## 二、PV 与 PVC — 存储的供给与消费

### 2.1 为什么要分 PV 和 PVC

K8s 用 PV（PersistentVolume）和 PVC（PersistentVolumeClaim）将存储的**供给方**和**消费方**解耦：

```
┌─── 存储管理员（或 StorageClass 自动） ───┐    ┌─── 开发者 ──────────────┐
│                                        │    │                        │
│  PersistentVolume（PV）                 │    │  PersistentVolumeClaim │
│  "我有一块 100Gi 的 EBS 卷"              │←──→│  "我需要 50Gi 存储"     │
│  实际的存储资源                           │ 绑定 │  存储请求                │
│                                        │    │                        │
│  具体参数：                              │    │  Pod 引用 PVC：          │
│  - 存储后端：EBS gp3                     │    │  volumes:              │
│  - 容量：100Gi                          │    │    - name: data         │
│  - 访问模式：ReadWriteOnce              │    │      persistentVolume-  │
│  - 回收策略：Retain                     │    │      Claim:             │
│                                        │    │        claimName: mydata│
└────────────────────────────────────────┘    └────────────────────────┘
```

**为什么不直接在 Pod 里写"用哪块磁盘"**：
1. **可移植性**：开发者不关心底层是 EBS、EFS 还是 NFS，只需要声明"我要多大的空间"
2. **生命周期解耦**：Pod 被删除后，PV/PVC 仍然存在，数据不丢
3. **权限分离**：集群管理员管理存储资源（PV），开发者只提需求（PVC）

### 2.2 PV 定义示例

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-ebs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce          # 只能被一个节点挂载
  persistentVolumeReclaimPolicy: Retain   # PVC 删除后保留数据
  storageClassName: gp3
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc123def456789     # AWS EBS 卷 ID
```

### 2.3 PVC 定义示例

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi          # 需要 50Gi（PV 容量 ≥ 50Gi 的才能匹配）
```

### 2.4 Pod 使用 PVC

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: postgres
      image: postgres:15
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data    # 引用上面的 PVC
```

> [!tip] 🔗 实战链接：PVC 在 audio-gateway 服务中的使用
> `my-gateway` 的静态文件服务（nginx）通过 PVC 挂载 FSx OpenZFS 共享存储，让多个 Pod 可以访问同一份音频文件：
> ```yaml
> # deploy 项目：my-gateway 的 values 文件
> volumeMounts:
>   - name: fsx-ado1
>     mountPath: /data/my-gateway/ado1
>   - name: fsx-ado2
>     mountPath: /data/my-gateway/ado2
>   - name: fsx-ado3
>     mountPath: /data/my-gateway/ado3
> volumes:
>   - name: fsx-ado1
>     persistentVolumeClaim:
>       claimName: fsx-openzfs-gateway-fsx-openzfs-pvc-ado1
>   - name: fsx-ado2
>     persistentVolumeClaim:
>       claimName: fsx-openzfs-gateway-fsx-openzfs-pvc-ado2
>   - name: fsx-ado3
>     persistentVolumeClaim:
>       claimName: fsx-openzfs-gateway-fsx-openzfs-pvc-ado3
> ```
> 这是一个典型的 **ReadWriteMany (RWX)** 场景——多个 nginx Pod 需要同时读取共享文件系统上的音频数据。FSx OpenZFS 作为 NFS 兼容存储支持多 Pod 并发挂载，而普通 EBS 卷做不到这一点。

### 2.5 PV 的绑定流程

```
PVC 创建
    ↓
K8s 查找匹配的 PV（StorageClass 相同、容量 ≥ 请求、访问模式兼容）
    ↓
  找到 → PV 和 PVC 绑定（一对一），PVC 状态变为 Bound
  没找到 → PVC 状态为 Pending（如果有 StorageClass，则动态创建 PV）
    ↓
Pod 挂载 PVC → kubelet 调用 CSI 驱动挂载实际存储
```

---

## 三、StorageClass — 动态供给

### 3.1 手动 vs 动态供给

手动供给（Static Provisioning）：管理员预先创建 PV，开发者创建 PVC 去"认领"。
动态供给（Dynamic Provisioning）：**开发者只创建 PVC，StorageClass 自动创建对应的 PV 和底层存储资源**。

```
手动供给：管理员创建 EBS 卷 → 创建 PV → 开发者创建 PVC → 绑定
动态供给：开发者创建 PVC（指定 StorageClass）→ StorageClass 自动创建 EBS 卷 + PV → 绑定
```

> 生产环境几乎都用动态供给——不需要管理员为每个 PVC 手动创建 EBS 卷。

### 3.2 StorageClass 定义（进阶）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com     # 使用 EBS CSI 驱动
parameters:
  type: gp3                       # EBS 卷类型
  fsType: ext4                    # 文件系统类型
  encrypted: "true"               # 加密
reclaimPolicy: Delete             # PVC 删除时自动删除 PV 和 EBS 卷
volumeBindingMode: WaitForFirstConsumer   # 等 Pod 调度后再创建卷（保证同 AZ）
allowVolumeExpansion: true        # 允许扩容
```

**关键参数**：

| 参数 | 含义 |
|------|------|
| `provisioner` | 使用哪个 CSI 驱动创建存储 |
| `reclaimPolicy` | PVC 删除后如何处理 PV：`Delete`（删除）、`Retain`（保留） |
| `volumeBindingMode` | `Immediate`（立即创建）、`WaitForFirstConsumer`（等 Pod 调度后再创建，推荐） |
| `allowVolumeExpansion` | 是否允许后续扩大 PVC 容量 |

> [!tip] 🔗 实战链接：OpenSearch 的持久化存储配置
> 在 deploy 项目的 `infra/values/opensearch-data/` 中，OpenSearch 数据节点使用了 500Gi 的持久存储：
> ```yaml
> # deploy 项目：infra/values/opensearch-data/default.yaml
> persistence:
>   enabled: true
>   accessModes:
>     - ReadWriteOnce         # EBS 块存储，单节点挂载
>   size: 500Gi               # 每个数据节点 500Gi
> ```
> OpenSearch 是有状态服务，每个数据节点都需要独立的持久存储。这里使用 **ReadWriteOnce** 模式（EBS 卷），StorageClass 动态创建 PV——开发者只需声明需要多大空间，不需要手动管理 EBS 卷。这正是第三节"动态供给"的典型应用。

### 3.3 EKS 中常用的 StorageClass（进阶）

| StorageClass | 底层存储 | 特点 | 适用场景 |
|------|------|------|------|
| **gp3**（推荐） | EBS gp3 | 基线 3000 IOPS、125 MB/s，可独立配置 | 通用工作负载 |
| **gp2** | EBS gp2 | IOPS 与容量绑定（3 IOPS/GiB） | 旧集群默认 |
| **io2** | EBS io2 | 最高 64000 IOPS，99.999% 可用性 | 高性能数据库 |
| **efs-sc** | EFS | NFS 协议，多 Pod 共享读写 | 共享文件、ML 数据集 |

---

## 四、访问模式（Access Modes）

| 访问模式 | 缩写 | 含义 | 支持的存储 |
|------|------|------|------|
| **ReadWriteOnce** | RWO | 只能被一个**节点**挂载为读写 | EBS、大多数块存储 |
| **ReadOnlyMany** | ROX | 可以被多个节点挂载为只读 | EFS、NFS |
| **ReadWriteMany** | RWX | 可以被多个节点挂载为读写 | EFS、NFS、CephFS |
| **ReadWriteOncePod** | RWOP | 只能被一个 **Pod** 挂载为读写（K8s 1.27+） | CSI 驱动支持时 |

> **EBS 的限制**：EBS 卷只能挂载到一个 EC2 实例（RWO），不能被多个节点同时读写。如果需要多 Pod 共享存储，用 EFS（RWX）。

---

## 五、CSI（Container Storage Interface）（深入）

### 5.1 CSI 是什么

CSI 是 K8s 的存储插件接口（类似网络的 CNI）。存储厂商实现 CSI 驱动，K8s 通过统一接口对接各种存储后端。

```
K8s 存储流程：
PVC 创建 → StorageClass 指定 provisioner → CSI Controller 创建卷
    ↓
Pod 调度到节点 → kubelet 调用 CSI Node 插件 → 挂载卷到 Pod
```

### 5.2 EKS 常用 CSI 驱动

| CSI 驱动 | 安装方式 | 提供的存储 |
|------|------|------|
| **EBS CSI Driver** | EKS 托管插件 | EBS 块存储（gp3、io2 等） |
| **EFS CSI Driver** | EKS 托管插件 | EFS 共享文件存储 |
| **FSx CSI Driver** | 手动安装 | FSx for Lustre（高性能计算） |

```bash
# 查看集群安装的 CSI 驱动
kubectl get csidrivers
# NAME                  ATTACHREQUIRED   PODINFOONMOUNT
# ebs.csi.aws.com       true             false
# efs.csi.aws.com       false            false
```

---

## 六、卷扩容（进阶）

PVC 创建后发现空间不够，可以在线扩容（前提：StorageClass 开启了 `allowVolumeExpansion`）：

```bash
# 修改 PVC 的容量
kubectl patch pvc postgres-data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# 查看扩容状态
kubectl get pvc postgres-data
# NAME            STATUS   VOLUME          CAPACITY   ACCESS MODES
# postgres-data   Bound    pv-xxx          200Gi      RWO
```

> **注意**：EBS 卷扩容后，文件系统也需要扩展。EBS CSI 驱动会自动处理（在线扩展 ext4/xfs）。但 **PVC 只能扩容，不能缩容**。

---

## 七、回收策略（Reclaim Policy）（进阶）

当 PVC 被删除时，PV 和底层存储的处理方式：

| 策略 | 行为 | 适用场景 |
|------|------|------|
| **Delete** | PV 和底层存储（EBS 卷）一起删除 | 临时数据、可重建的内容 |
| **Retain** | PV 变为 Released 状态，底层存储保留 | 重要数据，需要手动确认后清理 |

```
PVC 删除后的生命周期：

Delete 策略：PVC 删除 → PV 删除 → EBS 卷删除 → 数据永久丢失
Retain 策略：PVC 删除 → PV 变为 Released → EBS 卷保留 → 管理员手动处理
```

> **生产建议**：数据库等重要数据用 `Retain`，避免误删 PVC 导致数据丢失。配合定期备份（EBS Snapshot）。

---

## 八、与本项目的关联

### 8.1 当前用法：emptyDir

```yaml
# values 文件中的配置
volumeMounts:
  - name: logs
    mountPath: /data/plaud-sync/logs
volumes:
  - name: logs
    emptyDir: {}              # 临时存储，Pod 销毁即丢失
```

这对于无状态的微服务足够——日志最终会被 Fluent Bit 采集到集中存储。

> [!tip] 🔗 实战链接：Kafka 的 JBOD 持久存储
> 在 deploy 项目的 `infra/values/strimzi-kafka/` 中，Kafka 使用 JBOD（Just a Bunch of Disks）模式配置持久存储，不同环境使用不同容量：
> ```yaml
> # deploy 项目：infra/values/strimzi-kafka/base/KafkaNodePool.yaml
> storage:
>   type: jbod
>   volumes:
>     - id: 0
>       type: persistent-claim
>       size: 100Gi             # 基础配置 100Gi
>       deleteClaim: false      # 删除 Kafka 节点时保留 PVC（类似 Retain 策略）
>       kraftMetadata: shared
> ```
> ```yaml
> # deploy 项目：生产环境覆盖 (overlays/global/prod)
> storage:
>   type: jbod
>   volumes:
>     - id: 0
>       type: persistent-claim
>       size: 1Ti               # 生产环境扩大到 1Ti
> ```
> 注意 `deleteClaim: false` 这个配置——它确保即使 Kafka Pod 被删除，底层的 PVC 和 EBS 卷也会保留，防止消息数据丢失。这对应了第七节中 **Retain 回收策略**的思想。

> [!tip] 🔗 实战链接：OpenSearch 的 emptyDir 用于共享内存
> OpenSearch 还使用了一个有趣的 emptyDir 用法——挂载为 `/dev/shm` 以扩展共享内存：
> ```yaml
> # deploy 项目：infra/values/opensearch-data/default.yaml
> extraVolumes:
>   - name: shm-volume
>     emptyDir:
>       sizeLimit: 1Gi          # 限制大小为 1Gi
> extraVolumeMounts:
>   - name: shm-volume
>     mountPath: /dev/shm       # 挂载为共享内存
> ```
> emptyDir 不仅可以存临时文件，还可以用于挂载 `/dev/shm`（共享内存）。容器默认的 `/dev/shm` 只有 64MB，对于需要大量内存映射的应用（如 OpenSearch、数据库）远远不够，通过 emptyDir 可以扩展到指定大小。

### 8.2 何时需要 PV/PVC

| 场景 | 当前方案 | 需要 PV/PVC 吗 |
|------|------|------|
| 微服务日志 | emptyDir + 日志采集 | 不需要 |
| 模型缓存 | 每次启动重新下载 | 可以考虑（节省启动时间） |
| 数据库 | 使用 RDS（托管服务） | 不需要（K8s 外部管理） |
| ML 训练数据 | S3 | 可以用 EFS（多 Pod 共享） |

> **最佳实践**：无状态的微服务尽量不依赖持久存储。数据库等有状态服务优先使用云托管服务（RDS、ElastiCache），避免在 K8s 上自己管理有状态工作负载的复杂性。

---

## 九、存储排障

```bash
# 1. 查看 PVC 状态（Pending 说明没找到匹配的 PV）
kubectl get pvc -n <namespace>

# 2. 查看 PVC 详情（Events 里有失败原因）
kubectl describe pvc <pvc-name> -n <namespace>

# 3. 查看 PV
kubectl get pv

# 4. 查看 StorageClass
kubectl get storageclass

# 5. 查看 CSI 驱动是否正常
kubectl get pods -n kube-system | grep ebs-csi

# 常见问题：
# - PVC Pending："no persistent volumes available" → StorageClass 名称不匹配
# - PVC Pending："waiting for first consumer" → volumeBindingMode=WaitForFirstConsumer，正常
# - Pod 无法挂载卷："Multi-Attach error" → EBS 卷被另一个节点占用（RWO 限制）
```

---

## 延伸阅读

- [[10-helm-argocd-deployment|Helm 与 EKS 部署体系]] — emptyDir 和 volumeMounts 的实际配置
- [[03-k8s-workload-types|K8s 工作负载类型]] — StatefulSet 的 volumeClaimTemplates
- [[05-k8s-architecture|K8s 架构原理]] — kubelet 如何调用 CSI 驱动挂载卷
- [[01-docker-basics|Docker 学习笔记]] — Docker Volume 与 K8s PV/PVC 的关系
