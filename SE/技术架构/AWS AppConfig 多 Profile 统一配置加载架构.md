---
tags:
  - 技术架构
  - AWS-AppConfig
  - 配置管理
aliases:
  - AppConfig 统一配置架构
---

# AWS AppConfig 多 Profile 统一配置加载架构

> [!abstract] 定位
> 这是一套面向生产环境的通用配置控制架构。部署环境只提供配置入口；Main Profile 声明依赖；每个进程内的配置控制器加载、校验并发布完整运行时快照；业务只读取已接受的快照。

![[aws-appconfig-unified-configuration-architecture.png|1200]]

> [!tip] 全局心智模型
> AWS AppConfig 保存非敏感的期望状态，AWS Secrets Manager 保存凭证原文。配置控制器把多个外部版本收敛为一个进程内已接受状态。业务看不到“正在加载”或“校验失败”的中间状态，只能读取完整旧快照或完整新快照。

## 1. 适用范围与目标

这套架构适用于同时满足以下条件的服务：

- 使用多个 AWS AppConfig Profile；
- Profile 具有独立发布节奏或权限边界；
- 配置更新前需要执行 schema、跨 Profile 引用或派生对象校验；
- 需要在运行期间轮询配置，并在错误更新到达时保留 last-known-good；
- Secret 保存在 AWS Secrets Manager，AppConfig 只保存引用；
- API、Worker 或脚本需要共享同一套启动顺序。

核心目标只有一个：

```text
外部期望状态
  → 构建完整候选
  → 校验候选
  → 发布已接受状态
  → 业务只读使用
```

这套架构不负责自动切换数据库连接池、消息消费者、第三方 SDK Client 等有状态资源。配置快照更新只表示新配置已经可见，不表示所有外部资源已经完成重建。

## 2. 核心模型

### 2.1 四层职责

| 层次 | 回答的问题 | 核心内容 |
| --- | --- | --- |
| 启动定位层 | 去哪里加载 | Region、Application、Environment、Main Profile |
| 外部来源层 | 期望状态存在哪里 | AWS AppConfig、本地 Profile、AWS Secrets Manager |
| 进程控制层 | 哪些状态可以生效 | 依赖拓扑、Profile Session、候选构建、校验、原子发布 |
| 业务消费层 | 业务可以读取什么 | 已接受的只读快照、必需资源和应用就绪状态 |

主链路如下：

```text
启动定位参数
  → 加载 Main Profile
  → 固定依赖拓扑
  → 读取 Secret 引用
  → 加载所有依赖 Profile
  → 构建完整候选与派生对象
  → 完整校验
  → 原子发布已接受快照
  → 构建必需资源
  → 应用进入 Ready
```

### 2.2 组件边界

| 组件 | 负责 | 不负责 |
| --- | --- | --- |
| 部署环境 | 提供最小启动定位参数 | 保存业务配置或 Secret 原文 |
| Main Profile | 声明 Profile 与 Secret 引用 | 保存 Secret 原文、控制运行时线程 |
| AWS AppConfig | 保存、验证和分阶段部署非敏感配置 | 保证多个 Profile 同时生效 |
| AWS Secrets Manager | 保存、授权访问并版本化 Secret | 组织普通业务配置 |
| 配置控制器 | 加载 Profile、构建候选、校验、发布、轮询和关闭 | 向业务暴露 AWS Client 或未校验内容 |
| Secret Runtime | 读取、注入和跟踪 Secret 版本 | 组合普通配置快照 |
| 应用配置快照 | 保存完整、有效、逻辑只读的运行时状态 | 访问 AWS、持有轮询 Session |
| 业务组件 | 定义领域约束并读取已接受快照 | 自行创建 AppConfig 加载器或第二份配置缓存 |

### 2.3 七条不变量

1. 每个 OS 进程只有一个配置控制器。
2. 每个 Profile 拥有独立 AppConfig Data Session。
3. 依赖拓扑在启动时确定，运行期间不局部改写。
4. 候选包含所有必需 Profile、来源版本和派生对象。
5. 候选通过格式、schema、跨 Profile 引用和派生对象校验后才能发布。
6. 热更新失败时保留 last-known-good，不能用未校验内容覆盖它。
7. Secret 原文不进入 AppConfig、普通配置 payload、日志和诊断接口。

> [!important] “只读”的准确含义
> 快照提供逻辑只读契约。`frozen` 模型无法自动冻结内部 `dict` 或第三方 SDK 对象，因此构造时必须隔离源数据，消费者不得修改快照内容。安全要求高的字段应使用不可变容器。

## 3. 依赖拓扑

### 3.1 Main Profile 是依赖清单

Main Profile 先被加载。配置控制器随后从中提取当前进程需要的 Profile 和 Secret 引用。

```yaml
profiles:
  routing: routing.yaml
  feature_flags: feature-flags.yaml

secrets:
  - id: prod/service/api-key
    kind: env
  - id: prod/service/account
    kind: file
    env_var: SERVICE_ACCOUNT_FILE
```

依赖分为两类：

| 依赖 | 生命周期 | 变更方式 |
| --- | --- | --- |
| Region、Profile 名、Secret 引用 | 启动拓扑 | 修改后滚动重启 |
| Profile 内容、Secret 版本 | 运行时状态 | 受控热更新或轮换 |

固定拓扑的原因是 AppConfig Session、校验关系和后台资源都依赖 Profile 集合。运行期间若 Main Profile 改变依赖集合，当前进程拒绝该候选并继续使用 last-known-good。新拓扑由滚动重启后的新进程加载。

### 3.2 Profile 拆分原则

判断标准是配置能否独立发布、验证和回滚。

| 配置关系 | 归属方式 |
| --- | --- |
| 必须一起修改、验证和回滚 | 放入同一个 Profile |
| 发布节奏或权限边界独立 | 拆成不同 Profile |
| 只服务于可选能力 | 单独建 Profile，由 Main Profile 显式引用 |
| 密码、Token、服务账号凭证 | 存入 Secrets Manager，AppConfig 只保存引用 |

跨 Profile 变更必须保持分阶段兼容。假设 `routing v11` 只有配合 `feature v21` 才合法：

```text
routing v11 + feature v20 → 拒绝
routing v10 + feature v21 → 拒绝
```

两个新版本会互相等待，系统始终停留在旧组合。正确做法是使用“扩展 → 迁移 → 收缩”的兼容发布流程；无法提供兼容窗口的字段应放入同一个 Profile。

## 4. 首次启动

### 4.1 启动顺序

```text
1. 校验启动定位参数
2. 创建当前进程唯一的配置控制器
3. 加载 Main Profile 并解析固定依赖拓扑
4. 根据 Secret 引用读取并准备凭证
5. 为每个依赖 Profile 创建独立 Session 并加载内容
6. 解析配置，构建索引、映射和 SDK typed config 等派生对象
7. 校验完整候选
8. 原子发布应用配置快照
9. 启动 Profile 轮询和 Secret 轮换
10. 构建必需资源，通过应用就绪门后进入 Ready
```

首次启动没有 last-known-good。任意必需 Profile 无法获取，或完整候选无法通过校验，进程都必须在接收流量或任务前退出。

### 4.2 配置就绪与应用就绪

配置控制器只能决定配置是否就绪：

```text
config_ready =
  首个完整快照已发布
  AND 必需 Secret 已准备
  AND 配置生命周期已经建立
```

Pod 是否进入 Ready 由应用就绪门决定：

```text
application_ready =
  config_ready
  AND 必需数据库 / Redis / 队列等资源已就绪
  AND 进程未进入关闭状态
```

Readiness 探针必须查询应用就绪门，不能只探测进程存活或 TCP 端口。

> [!warning] Liveness 与 Readiness 不可混用
> AppConfig 暂时不可用但进程仍持有 last-known-good 时，服务通常应继续 Ready，同时告警。若把外部配置源的瞬时故障当作 liveness 失败，Kubernetes 会重启仍可正常服务的进程，并放大故障。

## 5. Profile 更新状态机

### 5.1 远端会话与业务状态必须分离

每个 Profile 至少维护三类状态：

| 状态 | 含义 |
| --- | --- |
| 轮询游标 | `NextPollConfigurationToken` 与下一次允许轮询时间 |
| 最近观察版本 | AppConfig 最近返回并完成解析尝试的版本 |
| 最近接受版本 | 最近一次进入完整有效快照的版本，即 last-known-good |

AppConfig 返回的 Token 通常是一次性的。即使新内容校验失败，也必须推进轮询游标，否则下一轮可能重复使用失效 Token。推进远端会话不等于接受新配置。

```text
拉取远端响应
  → 立即保存下一枚 Token 和轮询间隔
  → 内容为空：结束本轮
  → 内容存在：记录观察版本并构建候选
      ├─ 校验成功：更新接受版本并发布
      └─ 校验失败：保留旧接受版本，记录拒绝原因
```

轮询器应遵守服务端返回的 `NextPollIntervalInSeconds`，并对网络失败、Token 失效和 Session 重建采用带退避的重试策略。

### 5.2 候选构建与原子发布

当前已接受组合为：

```text
main v5 + routing v10 + feature v20
```

当 `routing v11` 到达时，配置控制器构建：

```text
main v5 + routing v11 + feature v20
```

随后执行：

```text
格式解析
  → Profile schema
  → 跨 Profile 引用
  → Secret 解析或引用绑定
  → 派生对象构建
  → 完整根模型校验
      ├─ 全部通过：一次替换当前快照引用
      └─ 任一失败：丢弃候选，保留旧快照
```

原子发布只保证同一进程内的读取者看到完整旧快照或完整新快照。它不表示 AWS AppConfig 能同时发布多个 Profile，也不表示多个进程会在同一时刻切换。

### 5.3 快照版本是版本向量

多个 Profile 没有共同的 AWS 版本号。一个快照应记录：

```text
generation = 42                 # 当前进程内发布代次
profile_versions = {            # 组成快照的接受版本向量
  main: v5,
  routing: v11,
  feature: v20
}
```

`generation` 只用于当前进程内比较先后；诊断跨进程差异时必须使用 Profile 版本向量。

## 6. Secret 运行时与轮换

### 6.1 Secret 的安全边界

Secret 原文必须满足以下约束：

- 不写入 AppConfig；
- 不写入普通配置 payload；
- 不进入结构化日志、异常消息和诊断接口；
- 不参与可序列化的快照输出；
- 临时文件使用受限权限，并在进程退出时清理。

第三方 SDK 有时要求在构造 typed config 时提供明文凭证。此时解析后的凭证会存在于进程内运行时对象中。该对象必须被视为敏感对象：禁止序列化、打印、复制到普通配置或暴露给诊断接口。

因此，准确的不变量是：

> Secret 原文不进入普通配置和可观察面；运行时对象只有在 SDK 契约确实要求时才持有解析后的凭证。

### 6.2 首选凭证模型

通用架构优先向业务发布凭证引用或 `CredentialProvider`，而不是发布明文字符串：

```text
业务 Client
  → CredentialProvider
  → 当前 Secret 版本
```

这使凭证轮换与普通配置快照解耦。SDK 只支持字符串或文件路径时，Secret Runtime 负责适配，但必须明确实际生效条件：

| 消费模式 | Watchdog 更新动作 | 新凭证实际生效条件 |
| --- | --- | --- |
| Credential Provider | 替换 Provider 内部版本 | 后续请求重新取值 |
| 稳定文件路径 | 原路径原子重写内容 | Client 在后续请求重新读取文件 |
| 环境变量或 typed config 字符串 | 更新环境变量 | 重建 typed config 和 Client，或滚动重启 |

Watchdog 能发现并注入新版本，不等于下游请求已经使用新凭证。凭证消费模式和 Client 生命周期共同决定轮换完成时间。

### 6.3 Secret 失败策略

| 场景 | 行为 |
| --- | --- |
| 必需 Secret 首次获取失败，且没有有效受控回退 | 配置候选失败，应用不 Ready |
| 可选能力的 Secret 缺失 | 对应能力保持关闭，不能伪造凭证 |
| 运行时轮换失败 | 继续使用仍有效的旧版本，告警并重试 |
| 旧凭证已经失效且新版本不可用 | 对依赖该凭证的能力降级或置为不可用 |

## 7. 多进程与一致性边界

“唯一配置控制器”的边界是 OS 进程，不是 Pod：

```text
一个 Pod
  ├─ 进程 A → 配置控制器 A → generation 42
  ├─ 进程 B → 配置控制器 B → generation 41
  └─ 进程 C → 配置控制器 C → generation 42
```

各进程独立持有 AppConfig Session，并在各自轮询周期内最终收敛。系统不保证跨进程、跨 Pod 同时切换。

业务操作开始时只读取一次快照，并在本次操作内复用：

```python
settings = get_settings()
handle_request(settings, request)
```

不要在一个请求的不同阶段反复读取当前快照，否则同一操作可能混用两个配置代次。

若业务要求全局同时切换，就不能依赖进程内轮询实现。此类需求应使用兼容发布、流量分组或显式协调协议。

## 8. AWS 服务端发布保护

进程内候选校验是最后一道防线，不应替代 AWS AppConfig 的服务端保护。

```text
发布前
  → JSON Schema / Lambda Validator
部署中
  → Deployment Strategy / Bake Time
运行监控
  → CloudWatch Alarm / 自动回滚
进程内
  → 跨 Profile、Secret 与派生对象完整校验
```

两层校验职责不同：

| 层次 | 适合验证 |
| --- | --- |
| AppConfig Validator | 单 Profile 格式、字段类型、局部约束 |
| 进程内候选校验 | 跨 Profile 引用、部署参数、Secret 可用性、派生对象和领域约束 |

发布策略应控制变更扩散速度。Bake time 内的错误指标触发自动回滚，减少错误版本到达全部进程的概率。

## 9. 失败语义

| 场景 | 行为 |
| --- | --- |
| 启动定位参数缺失或非法 | 创建 AWS Client 前退出 |
| 必需 Profile 首次加载失败 | 启动失败，应用不 Ready |
| 首次候选解析、校验或派生构建失败 | 不发布快照，启动失败 |
| Profile 热更新获取失败 | 保留 last-known-good，推进重试计划 |
| Profile 热更新候选非法 | 记录观察版本和拒绝原因，保留接受版本 |
| Profile 名或 Secret 引用变化 | 拒绝热更新，通过滚动重启生效 |
| AppConfig 暂时不可用且旧快照仍有效 | 继续服务并告警 |
| 有状态资源尚未完成切换 | 不宣称该资源已经应用新配置 |
| 进程关闭 | 立即退出 Ready，停止轮询并清理 Secret 临时文件 |

## 10. 可观测性

配置系统至少暴露以下指标和诊断状态：

| 信号 | 用途 |
| --- | --- |
| 当前进程 `generation` | 判断本进程发布次数 |
| 各 Profile 观察版本 | 判断远端新版本是否已到达 |
| 各 Profile 接受版本 | 判断当前快照实际使用什么 |
| last-known-good 年龄 | 发现长期无法接受更新的进程 |
| 最近成功拉取与发布时间 | 判断轮询是否停滞 |
| 拉取、解析、校验失败计数 | 定位失败阶段 |
| Session 重建次数 | 发现 Token 或网络异常 |
| Secret 轮换成功与失败计数 | 判断凭证控制面是否健康 |
| `config_ready` / `application_ready` | 区分配置就绪与应用就绪 |

日志只记录 Profile 名、版本、候选阶段、Secret ID 和错误类型。不得记录配置全文、Secret 原文、注入后的值或敏感运行时对象。

## 11. 接入规范

业务模块只承担三项职责：

- 定义需要的字段与领域 schema；
- 定义跨字段、跨 Profile 的有效性约束；
- 在一次业务操作开始时读取并固定当前快照。

业务模块不得：

- 自行创建 AppConfig Data Session；
- 自行轮询 AppConfig 或 Secrets Manager；
- 直接读取未校验的 Profile 字典；
- 永久缓存 Prompt、模型映射、白名单等可派生配置；
- 把配置发布等同于有状态资源切换完成。

可由配置纯计算得到的索引、映射和 typed config，应在候选发布前完成构建。需要建立网络连接或后台线程的资源，由资源所有者实现“创建新实例 → 健康检查 → 切换流量 → 排空旧实例 → 失败回退”。

## 12. 验收清单

- [ ] 每个 OS 进程只有一个配置控制器；
- [ ] Main Profile 能完整描述 Profile 与 Secret 引用；
- [ ] 每个 Profile 拥有独立 AppConfig Data Session；
- [ ] 轮询游标、观察版本和接受版本分开维护；
- [ ] 所有 Profile Session 可共用一条调度线程，并遵守各自轮询间隔；
- [ ] 首个完整候选发布前，应用不会进入 Ready；
- [ ] 合法候选一次替换完整快照；
- [ ] 非法候选保留 last-known-good，并能观察拒绝原因；
- [ ] Profile 名和 Secret 引用只能通过滚动重启变更；
- [ ] Secret 原文不进入普通配置、日志和诊断接口；
- [ ] Secret 轮换文档明确下游 Client 的实际生效条件；
- [ ] 单次请求或任务固定使用一个快照；
- [ ] AppConfig Validator、部署策略、告警和回滚已经配置；
- [ ] 配置就绪与应用就绪分别可观测；
- [ ] 进程退出时先退出 Ready，再停止 Profile 与 Secret 后台任务。
