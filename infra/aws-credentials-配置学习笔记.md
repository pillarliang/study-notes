# AWS 凭证配置

> 配套阅读：[[k8s-learning/08-k8s-security-rbac#三、ServiceAccount 与 IRSA|K8s 安全 RBAC — IRSA 实战]]。
> 参考案例：`deploy/plaud-project-summary/values/us-west-2/staging/main.yaml`。

---

## 一、核心问题：调用 AWS API 时凭证从哪来

任何 AWS SDK 调用（Boto3、aws-sdk-go、CLI）都需要回答三件事：

1. **身份**：调用者是谁？对应 `Access Key ID`。
2. **签名密钥**：用什么签 HTTP 请求？对应 `Secret Access Key`。
3. **临时凭证补充**：是否是短期凭证？对应 `Session Token`（仅临时凭证有）。

AWS 把这三项统称 **Credentials**。SDK 不要求显式传入——只要从一条**凭证解析链**中能拿到，就直接用。理解这条链是后续所有配置方式的前提。

---

## 二、关键名词速查

后文频繁出现的 AWS / K8s 缩写，先建立直觉再看具体配置。

| 名词 | 全称 | 一句话 |
| --- | --- | --- |
| **AK / SK** | Access Key / Secret Key | AWS 的长期密码对，类似账号密码 |
| **IAM** | Identity and Access Management | AWS 身份权限系统，定义"谁能对哪些资源做什么" |
| **IAM User** | — | IAM 中的人/程序身份，关联长期 AK/SK |
| **IAM Role** | — | IAM 中的"可扮演身份"，自身无密钥，靠 STS 发临时凭证 |
| **Policy** | — | JSON 文档，挂到 User/Role 上声明允许或拒绝哪些 Action |
| **STS** | Security Token Service | AWS 临时凭证签发服务，把"扮演证明"换成短期凭证 |
| **IMDS** | Instance Metadata Service | EC2 内部元数据 HTTP 服务，固定地址 `169.254.169.254` |
| **ServiceAccount (SA)** | — | K8s 中给 Pod 的身份，对应人类的 User |
| **OIDC** | OpenID Connect | 开放身份认证协议，用公钥验证 JWT token 真伪 |
| **IRSA** | IAM Roles for Service Accounts | K8s SA 通过 OIDC 向 AWS STS 换临时凭证的机制 |
| **EKS** | Elastic Kubernetes Service | AWS 托管的 K8s 服务 |
| **ECR** | Elastic Container Registry | AWS 的容器镜像仓库 |

**抽象层次**：

```txt
身份系统  ──>  IAM (User / Role / Policy)
                  │
凭证签发  ──>     STS (AssumeRole / AssumeRoleWithWebIdentity)
                  │
分发渠道  ──>     IMDS (EC2 节点) / IRSA (EKS Pod) / 配置文件 (本地)
                  │
最终消费  ──>     AWS SDK
```

**记忆口诀**：IAM 定义权限，STS 签发临时凭证，IMDS / IRSA 是凭证到达进程的最后一公里。

---

## 三、凭证解析链（Credential Provider Chain）

SDK 启动时按固定顺序探查每个来源，**第一个命中的就用，不再继续**。Boto3 与 aws-sdk-v2 顺序基本一致：

```txt
SDK 初始化
   │
   ▼
① 显式参数             调用 client(aws_access_key_id=..., aws_secret_access_key=...)
   │ 未命中
   ▼
② 环境变量             AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_SESSION_TOKEN
   │ 未命中
   ▼
③ SSO Token            ~/.aws/sso/cache/*.json（aws sso login 生成）
   │ 未命中
   ▼
④ 共享凭证文件         ~/.aws/credentials 中 [profile-name] 段
   │ 未命中
   ▼
⑤ 共享配置文件         ~/.aws/config 中 source_profile / role_arn 等
   │ 未命中
   ▼
⑥ 容器凭证             ECS / Fargate / IRSA 注入的端点
                       AWS_CONTAINER_CREDENTIALS_RELATIVE_URI
                       AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN
   │ 未命中
   ▼
⑦ EC2 实例元数据       IMDS：http://169.254.169.254/latest/meta-data/iam/security-credentials/
   │ 全部未命中
   ▼
   抛出 NoCredentialsError
```

**关键推论**：

- 环境变量优先级高于配置文件——CI/CD 通过环境变量注入凭证可覆盖本地 profile。
- IRSA 落在 ⑥，比 IMDS 早一步；EKS 节点同时具备 ⑥ 和 ⑦ 时优先用 IRSA。
- 不要在已经有 IRSA 的 Pod 中再注 `AWS_ACCESS_KEY_ID`，否则 ② 会盖掉 ⑥，等同回退到静态密钥。

---

## 四、各种凭证提供方式

按"是否管理长期密钥"分两类：长期凭证（AK/SK）和短期凭证（STS 签发）。生产环境优先短期凭证。

### 4.1 静态 AK/SK（长期凭证）

最原始的方式——在 IAM 中给 User 创建 Access Key 后，把字符串塞进环境变量或配置文件。

```bash
# 方式 A：环境变量
export AWS_ACCESS_KEY_ID=AKIA****
export AWS_SECRET_ACCESS_KEY=****
export AWS_DEFAULT_REGION=us-west-2

# 方式 B：~/.aws/credentials
[default]
aws_access_key_id = AKIA****
aws_secret_access_key = ****
```

**问题**：

- 长期有效，泄露后影响面不可控。
- 轮换困难，多服务共用一把密钥时连锁停服。
- 没有过期机制，无法应对临时授权场景。

**适用场景**：本地开发、CI/CD（配合 GitHub Actions Secrets 等托管存储）。**禁止用于生产 Pod**。

### 4.2 EC2 Instance Profile（实例角色）

EC2 启动时附加一个 IAM Role，运行在该 EC2 上的进程通过 IMDS 拿到 STS 签发的临时凭证。

```txt
EC2 实例（附加 InstanceProfile → IAM Role）
        │
        │ 进程访问 169.254.169.254
        ▼
   IMDS 返回临时凭证（自动滚动续期）
        │
        ▼
   SDK 调用 AWS API
```

**优点**：无长期密钥；凭证自动轮换。
**缺点**：粒度是"整个 EC2"，同一节点上所有进程共享同一组权限。

### 4.3 IRSA（IAM Roles for Service Accounts）

EKS 专属机制，把 K8s ServiceAccount 与 IAM Role 通过 OIDC 协议绑定。**每个 Pod 独立身份**，不受同节点其他 Pod 影响。

核心配置——在 ServiceAccount 加一个 annotation：

```yaml
serviceAccount:
  create: true
  automount: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::329599658616:role/plaud-project-summary-role # K8s SA  ←──── 注解（写 Role ARN）────→  AWS IAM Role
  name: plaud-project-summary
```

应用层无需改任何代码——SDK 自动识别 EKS 注入的环境变量走 STS 路径。

工作原理与"为什么无需 AK/SK"详见 [[#六、IRSA 工作原理深入]]。

> [!note] 区域 ARN 格式
> 标准区：`arn:aws:iam::账号ID:role/角色名`
> 中国区：`arn:aws-cn:iam::账号ID:role/角色名`
> GovCloud：`arn:aws-us-gov:iam::账号ID:role/角色名`

### 4.4 EKS Pod Identity（IRSA 的继任者）

2023 年底推出，目标是简化 IRSA 的运维负担。

| 维度 | IRSA | EKS Pod Identity |
| --- | --- | --- |
| 绑定关系来源 | ServiceAccount annotation | EKS API（PodIdentityAssociation） |
| 依赖 OIDC Provider | 需要每集群配置 | 不需要 |
| Trust Policy | 必须信任集群 OIDC | 统一信任 `pods.eks.amazonaws.com` |
| 跨集群共享角色 | 每个集群一份 trust | 一份 trust 跨多集群 |

**注意**：IRSA 仍是当前最普遍的方案，迁移成本主要在 IAM Trust Policy 的改写。现有项目继续用 IRSA 没有问题。

### 4.5 STS AssumeRole（跨账号 / 临时升权）

不绑定到任何运行环境的通用机制——已有任意一组基础凭证，调用 STS 拿另一个角色的临时凭证。

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::目标账号:role/CrossAccountRole \
  --role-session-name my-session \
  --duration-seconds 3600
```

返回 `AccessKeyId / SecretAccessKey / SessionToken` 三元组，**Session Token 必填**——这是临时凭证的标识。

**适用场景**：跨账号访问、临时升权的运维操作、本地用基础 profile 切换到多个目标角色。

### 4.6 SSO / IAM Identity Center

为开发者本地环境设计，避免给每个人发 AK/SK。

```bash
aws configure sso          # 一次性配置 SSO 起始 URL
aws sso login --profile dev
```

登录后凭证缓存在 `~/.aws/sso/cache/`，SDK 自动读取。Token 通常 8 小时过期。

---

## 五、各方式对比速查

```txt
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│  静态 AK/SK         │  │  EC2 Instance       │  │  IRSA               │
│                     │  │  Profile            │  │                     │
│  · 长期凭证         │  │  · STS 短期凭证     │  │  · STS 短期凭证     │
│  · 易泄露           │  │  · 粒度=节点        │  │  · 粒度=Pod         │
│  · 仅本地/CI 使用   │  │  · EC2 自带         │  │  · 需 OIDC Provider │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘

┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│  EKS Pod Identity   │  │  STS AssumeRole     │  │  SSO / Identity     │
│                     │  │                     │  │  Center             │
│  · IRSA 简化版      │  │  · 跨账号通用       │  │  · 开发者本地登录    │
│  · 无需 OIDC        │  │  · 任意凭证可发起   │  │  · 浏览器 OAuth     │
│  · EKS API 管理     │  │  · 显式声明         │  │  · 缓存到本地       │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
```

| 方式 | 推荐度 | 凭证寿命 | 典型用途 |
| --- | --- | --- | --- |
| IRSA | 高 | ~1 小时 | EKS Pod 访问 AWS 资源 |
| EKS Pod Identity | 高（新集群） | ~1 小时 | 同上，新项目可优先选 |
| EC2 Instance Profile | 中 | ~6 小时 | 非容器化的 EC2 工作负载 |
| SSO | 中 | ~8 小时 | 开发者笔记本 |
| STS AssumeRole | 中 | 自定义 | 跨账号、临时升权 |
| 静态 AK/SK | 低 | 永久 | CI/CD、本地兜底 |

---

## 六、IRSA 工作原理深入

这一节回答一个关键问题：**IRSA 是怎么在不出现任何永久密码的情况下，让 Pod 拿到访问 AWS 的凭证的？**

### 6.1 类比：用学生证换图书馆临时卡

> 图书馆（AWS）想让"清华大学的研究生（Pod）"借书，但不愿意给每个研究生发永久借书证（AK/SK）——卡丢了影响面不可控。

图书馆的方案是：

**事前一次性约定**（建立信任）：

- 馆长跑一趟清华，记住学生证的样式和印章（公钥）
- 在图书馆系统登记："凡是清华校长盖章的学生证都认"

**每次借书时**：

```txt
研究生进图书馆
   │
   ▼
出示学生证（清华校长签名 + 学号 + 有效期）
   │
   ▼
前台核对签名 → 确认是清华发的
   │
   ▼
按规则查询 → 学号属于研究生院，可借专业书
   │
   ▼
发当天有效的临时借书卡
   │
   ▼
研究生用临时卡借书
   │
   ▼
当天结束，临时卡自动失效；次日再来再换一张
```

**关键性质**：

- 研究生从来没有图书馆的永久借书证
- 只有学校发的学生证（只证明身份，不能直接借书）
- 图书馆每天发一张一次性临时卡

### 6.2 类比与真实角色对照

| 类比 | IRSA 对应 |
| --- | --- |
| 清华大学 | EKS 集群 |
| 清华校长 | 集群的 OIDC Provider |
| 学生证 | ServiceAccount Token（JWT，集群私钥签名） |
| 研究生 | Pod |
| 图书馆 | AWS |
| 馆长事前去清华认印章 | 把 EKS OIDC Provider 注册到 IAM |
| 图书馆系统的规则 | IAM Role 的 Trust Policy + Permission Policy |
| 图书馆前台 | AWS STS |
| 临时借书卡 | STS 签发的临时凭证（默认 1 小时） |

### 6.3 SA Token 凭什么被 AWS 认

最反直觉的一步：SA Token 不是 AWS 发的，AWS 凭什么信？**靠数学**，不靠 AWS 自己。

```txt
集群内部有一对密钥：
   · 集群私钥（只有集群知道，绝对保密，从不出集群）
   · 集群公钥（公开给所有人，包括 AWS）

集群签 SA Token 时用私钥签名：
   → 只有持有私钥的人才能伪造
   → 任何持有公钥的人都能验证签名真伪

AWS 在 IRSA 一次性配置中，提前下载并保存了集群公钥：
   → 收到 Pod 给的 SA Token 后，用公钥验签
   → 验签通过 = 这个 token 一定是该集群签的，无法伪造
```

这正是 **OIDC 协议**做的事——用公私钥让两个互不信任的系统也能验证对方发的 token。

### 6.4 完整流程（去掉黑话）

**运维一次性配置**（每个集群每个 Role 各做一次）：

```txt
① 告诉 AWS：有一个 EKS 集群，公钥在某个 URL
   → AWS 下载并保存集群公钥

② 创建 IAM Role，Trust Policy 写明：
   "只有那个集群签的、属于 default/plaud-project-summary 这个 SA 的 token，才允许扮演我"

③ 在 K8s ServiceAccount 加注解：
   eks.amazonaws.com/role-arn = 上面那个 IAM Role 的 ARN
```

**Pod 启动到访问 AWS 的全过程**（自动，每次都发生）：

```txt
Pod 准备启动
   │
   │ EKS 发现 Pod 用的 SA 上有 role-arn 注解
   │ EKS Pod Identity Webhook 改写 Pod spec：
   │   · 把集群刚签好的 SA Token 挂到 Pod 内的某个文件
   │   · 注入环境变量 AWS_ROLE_ARN、AWS_WEB_IDENTITY_TOKEN_FILE
   ▼
Pod 启动，应用调 boto3.client("s3")
   │
   │ SDK 检测到 AWS_WEB_IDENTITY_TOKEN_FILE 环境变量
   │ SDK 自动读那个文件，拿到 SA Token
   ▼
SDK 调 STS:AssumeRoleWithWebIdentity
   │ 携带：RoleArn = 环境变量值，WebIdentityToken = SA Token
   ▼
STS 用之前存的集群公钥验签
   │ 签名对 ✓
   │ token 未过期 ✓
   │ Trust Policy 的 sub/aud 匹配 ✓
   ▼
STS 返回临时凭证（AK + SK + Session Token，1 小时过期）
   │
   ▼
SDK 缓存凭证，调 S3 API
   │
   ▼
50 分钟后，SDK 自动用 SA Token 再换一份新凭证
```

### 6.5 为什么这套机制比 AK/SK 安全得多

| 维度 | 静态 AK/SK | IRSA |
| --- | --- | --- |
| 永久密钥存在哪 | 笔记本、Git、CI 日志、K8s Secret、Pod 环境变量 | **不存在永久密钥** |
| 最值钱的秘密 | AK/SK 字符串 | 集群私钥（从不出集群） |
| 凭证寿命 | 永久 | 1 小时自动滚动 |
| 泄露后影响 | 不可控 | 1 小时内自动失效 |
| 谁能调 AWS | 任何拿到字符串的人 | 只有具体某个 Pod |

**核心抽象**：

> **IRSA 把"AWS 信任一串密码"换成"AWS 信任一个集群"。** Pod 用集群发的身份证去 AWS 换临时通行证——身份证由集群签名保真，通行证一小时一换，全程没有任何永久密码。

---

## 七、实战：plaud-project-summary 配置解析

参考文件：`deploy/plaud-project-summary/values/us-west-2/staging/main.yaml`。

### 7.1 区域与环境标识

```yaml
env:
  - name: AWS_ENV
    value: test
  - name: aws_region
    value: us-west-2
  - name: AWS_REGION
    value: us-west-2
  - name: AWS_DEFAULT_REGION   # Boto3 初始化的默认 region
    value: us-west-2
```

**为什么三个 region 变量都设**：

- `AWS_DEFAULT_REGION`：Boto3 的标准变量，未指定 client region 时使用。
- `AWS_REGION`：aws-sdk-go / Lambda 运行时使用的标准变量。
- `aws_region`（小写）：应用代码自己读的业务变量，与 SDK 无关。

三者并存是兼容多种 SDK 与历史代码的做法。新项目至少要有 `AWS_DEFAULT_REGION` 或 `AWS_REGION`。

### 7.2 IRSA 主路径

ServiceAccount 配置见 [[#4.3 IRSA（IAM Roles for Service Accounts）|4.3 节]]的 yaml 片段。各字段含义：

| 字段 | 作用 |
| --- | --- |
| `create: true` | Helm 创建对应的 ServiceAccount 资源 |
| `automount: true` | 把 SA Token 挂到 Pod 的 `/var/run/secrets/...`，IRSA 必须为 true |
| `annotations.role-arn` | EKS Pod Identity Webhook 据此注入 `AWS_ROLE_ARN` 与 token 路径 |
| `name` | Pod spec 通过 `serviceAccountName` 引用此 SA |

Pod 内 SDK 看到的环境变量（自动注入，无需在 values 写）：

```bash
AWS_ROLE_ARN=arn:aws:iam::329599658616:role/plaud-project-summary-role
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

### 7.3 已废弃的 AK/SK 方式

values 中保留了注释掉的旧配置作为迁移记录：

```yaml
# - name: AWS_ACCESS_KEY_ID
#   valueFrom:
#     secretKeyRef:
#       name: plaud-project-summary-aws
#       key: AWS_ACCESS_KEY_ID
# - name: AWS_SECRET_ACCESS_KEY
#   valueFrom:
#     secretKeyRef:
#       name: plaud-project-summary-aws
#       key: AWS_SECRET_ACCESS_KEY
```

这种写法把 AK/SK 存到 K8s Secret，再通过 `secretKeyRef` 注入到 Pod 环境变量。

**为什么换掉**：

- 静态密钥需要人工轮换，运维成本高。
- K8s Secret 默认仅 base64，etcd 未加密时形同明文。
- 凭证泄露后无法精确追踪到具体 Pod（多 Pod 共用一组密钥）。

迁移到 IRSA 后，这两个 Secret 直接删除——Pod 通过 OIDC 拿临时凭证，从源头消除长期密钥。

### 7.4 Secrets Manager 联动

```yaml
- name: SECRETS_MANAGER_ENABLED
  value: "true"
- name: SECRETS_MANAGER_GEMINI_KEY
  value: dev/project-summary/gemini-key
```

**完整链路**：

```txt
Pod（IRSA 注入临时凭证）
   │
   │ Boto3 调用 secretsmanager:GetSecretValue
   ▼
AWS STS 验证 IAM Role 权限
   │
   ▼
返回密钥明文（仅 Pod 内存可见）
```

IAM Role 必须包含读取该 secret 的策略：

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": "arn:aws:secretsmanager:us-west-2:329599658616:secret:dev/project-summary/*"
}
```

**收益**：第三方 API Key 不进 Git、不进 K8s Secret，集中存于 Secrets Manager，由 IAM 控制访问。

### 7.5 镜像拉取的凭证

```yaml
image:
  repository: 236604669925.dkr.ecr.us-west-2.amazonaws.com/plaud/plaud-project-summary
imagePullSecrets: []
```

`imagePullSecrets: []` 为空意味着 kubelet 用**节点 EC2 的 InstanceProfile** 拉 ECR 镜像——节点角色需含 `AmazonEC2ContainerRegistryReadOnly`。这是 EKS 默认实践，无需为镜像拉取额外配置 Pod 级凭证。

---

## 八、SDK 行为细节（以 Boto3 为例）

### 8.1 凭证刷新

```python
import boto3
client = boto3.client("s3")
```

`boto3.client()` 不立刻拉凭证——首次调用 API 时才执行解析链。**临时凭证默认在过期前 15 分钟自动刷新**，应用无需关心。

### 8.2 显式指定 region 与 profile

```python
session = boto3.Session(profile_name="prod", region_name="us-west-2")
client = session.client("s3")
```

`profile_name` 跳过 ② 环境变量，直接走 ④ ⑤ 的 profile 路径——适合本地需要在多账号间切换的场景。

### 8.3 排查凭证来源

```bash
# CLI 同样走凭证链
aws sts get-caller-identity
# 输出会显示当前用的是哪个 ARN，定位身份问题最快的命令
```

Boto3 应用调试：

```python
import boto3, logging
logging.basicConfig(level=logging.DEBUG)
boto3.client("sts").get_caller_identity()
# 日志会打印走到链上的哪一步
```

---

## 九、常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `NoCredentialsError` | 链上每一步都没命中 | 检查 IRSA annotation 是否生效、Pod spec 的 `serviceAccountName` 是否指向带 annotation 的 SA |
| `AccessDenied` 但身份正确 | IAM Role 缺少对应资源的策略 | `aws sts get-caller-identity` 确认身份后，检查 Role 的 Policy |
| IRSA 注入了但 SDK 仍走 IMDS | SDK 版本太老不识别 `AWS_WEB_IDENTITY_TOKEN_FILE` | 升级 SDK（Boto3 ≥ 1.10、aws-sdk-go ≥ 1.23） |
| 部分调用成功部分失败 | 临时凭证过期，SDK 未在过期前刷新 | 升级 SDK；避免长时间持有同一个 client 不发请求 |
| 中国区配置失败 | ARN partition 写成 `aws` 而非 `aws-cn` | 北京/宁夏区必须用 `arn:aws-cn:...` |
| Pod 看不到 token 文件 | `serviceAccount.automount: false` | 改为 true（IRSA 必需） |
| 环境变量盖掉 IRSA | 同时设了 `AWS_ACCESS_KEY_ID` | 删除环境变量，让链走到 ⑥ |

---

## 十、生产环境的检查清单

- 业务 Pod 走 IRSA 或 EKS Pod Identity，禁止注入静态 AK/SK
- IAM Role 遵循**最小权限**：只授该服务实际用到的 Action + Resource
- 跨账号访问通过 STS AssumeRole，禁止把目标账号的密钥发到源账号
- 第三方 API Key 走 Secrets Manager，禁止硬编码或塞进 K8s Secret 明文
- 本地开发用 SSO 或 named profile，避免长期 AK/SK 落在 `~/.aws/credentials`
- CI/CD 必须用静态密钥时，密钥仅存在托管 Secret Store（GitHub Actions Secrets、AWS CodeBuild），并定期轮换
- 中国区与海外区分别配置 ARN partition

---

## 参考

- AWS 官方：[Configuration and credential file settings](https://docs.aws.amazon.com/sdkref/latest/guide/file-format.html)
- AWS 官方：[IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- AWS 官方：[EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- Boto3 官方：[Credentials resolution order](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html)
- 项目案例：`deploy/plaud-project-summary/values/us-west-2/staging/main.yaml`
- 关联笔记：[[k8s-learning/08-k8s-security-rbac|K8s 安全与 RBAC]]、[[k8s-learning/10-helm-argocd-deployment|Helm 与 EKS 部署体系]]
