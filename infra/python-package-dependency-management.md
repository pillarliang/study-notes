# Python 包依赖管理：Git 源 vs PyPI 发布

> 适用场景：内部 Python monorepo，使用 uv 管理依赖，hatchling 构建

---

## 一、前端与后端的分工

Python 打包生态分两个角色：


| 角色                     | 职责                       | 常见工具                             |
| ---------------------- | ------------------------ | -------------------------------- |
| **前端 (installer)**     | 解析依赖、下载包、安装              | pip、uv、poetry                    |
| **后端 (build backend)** | 把源码打成 `.whl` / `.tar.gz` | hatchling、setuptools、poetry-core |


- **前端**就是你在终端敲的那个命令（`uv sync`、`pip install`），它负责：解析依赖 → 找到包 → 下载 → 安装到环境
- **后端**你不会直接调用，它在需要时被前端自动调用，负责把 Python 源码目录变成 `.whl` 文件（可安装的包格式）

### 两种安装路径下前端/后端的参与情况

**从 PyPI 安装（下载现成的包）：**

```
uv（前端）→ 从 PyPI 下载现成的 .whl → 直接安装
```

不需要后端，因为 PyPI 上已经是构建好的包。相当于买成品。

**从 Git 源码安装（下载源码现场构建）：**

```
uv（前端）→ clone 源码
         → 发现是源码，需要构建
         → 读源码的 pyproject.toml 里的 build-backend
         → 调用 hatchling（后端）把源码打成 .whl
         → 安装这个 .whl
```

需要后端，因为拿到的是源码不是成品。相当于买原材料，需要厨房加工。

### 声明 build backend

在包自身的 `pyproject.toml` 中通过 `build-backend` 声明：

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

这行告诉前端（uv/pip）：构建这个包的时候，请用 hatchling。

### 谁读哪个配置

`pyproject.toml` 中不同的配置段是给不同角色看的：


| 配置                       | 谁读                         | 用途           |
| ------------------------ | -------------------------- | ------------ |
| `[project] dependencies` | 前端（uv）和后端（hatchling）**都读** | 知道需要哪些包      |
| `[tool.uv.sources]`      | **只有 uv 读**                | 知道去哪里拿这些包    |
| `[build-system]`         | **只有前端读**                  | 知道该调用哪个后端来构建 |


以 `uv sync` 安装 `model-hub-sdk` 为例，完整流程：

```
uv sync
  ↓
uv 读 [project] dependencies → 发现需要 "model-hub-sdk"
  ↓
uv 读 [tool.uv.sources] → 发现要从 Git 仓库拿
  ↓
uv clone 源码 → 调用 hatchling 构建 → 安装
```

**"去哪里拿包"始终是前端（uv）的职责**。hatchling 只负责"把源码变成 .whl"，它看不到也不需要看 `[tool.uv.sources]`。

### 库 vs 应用：是否需要声明 `[build-system]`

- **库（被别人安装的包）** → 必须声明 `[build-system]`，否则前端 clone 下来源码后不知道该调谁来构建
- **应用（自己跑的项目）** → 不需要，`uv sync` 直接安装依赖到环境，不需要把项目自身构建成 `.whl`

### 不同 backend 解析依赖的语法不同

- hatchling / setuptools → 遵循 PEP 621，读 `[project] dependencies`，要求 PEP 508 格式
- poetry-core → 读 `[tool.poetry.dependencies]`，Poetry 自有格式

**build backend 决定了依赖必须写成什么格式**，混用会导致安装失败。

---

## 二、三种依赖格式对比

以从 Git monorepo 安装子包为例：

### 1. PEP 508 内联格式（通用）

```
model-hub-sdk @ git+https://token@github.com/Org/repo.git@master#subdirectory=packages/sdk-python
```

- 放在 `requirements.txt` 或 `[project] dependencies` 中
- pip / uv / hatch **都认**
- 包名、来源、版本全挤在一行，长了不好读

### 2. `[tool.uv.sources]` 格式（uv 专有）

```toml
# 依赖声明：只写包名
[project]
dependencies = ["model-hub-sdk"]

# 来源定义：结构化配置
[tool.uv.sources]
model-hub-sdk = {
    git = "https://token@github.com/Org/repo.git",
    subdirectory = "packages/sdk-python",
    rev = "master"
}
```

- **依赖声明与来源分离**，切换来源只改 sources，dependencies 不动
- 本地开发时可临时指向路径：`model-hub-sdk = { path = "../local-path" }`
- **uv 专有**，pip 不认

### 3. Poetry 格式

```toml
[tool.poetry.dependencies]
model-hub-sdk = { git = "https://token@github.com/Org/repo.git", subdirectory = "...", rev = "master" }
```

- **poetry 专有**，只有 `poetry install` 能解析
- 如果 build backend 是 hatchling，不能用这种格式

### 选择建议


| 场景        | 推荐格式                |
| --------- | ------------------- |
| 内部项目 + uv | `[tool.uv.sources]` |
| 需要 pip 兼容 | PEP 508 内联          |
| poetry 项目 | Poetry 格式           |


---

## 三、PEP 508 中 `@` 符号解析

一行 PEP 508 依赖中可能出现多个 `@`，各有含义：

```
model-hub-sdk @ git+https://token@github.com/Org/repo.git@v0.4.0#subdirectory=packages/sdk
^^^^^^^^^^^^^ ^                  ^                        ^
包名          ① 分隔符(有空格)   ② HTTP auth(无空格)      ③ Git ref(无空格)
```

- **① 包名与 URL 之间的 `@`**：两侧**有空格**（PEP 508 规范要求）
- **② URL 中的 `@`**（token 认证）：**无空格**
- **③ 指定 rev/tag/branch 的 `@`**：**无空格**

---

## 四、版本管理：pyproject.toml version vs Git Tag

### 两个版本号是独立的

```toml
# 包自身的 pyproject.toml
[project]
version = "0.4.0"    # ← 包的元数据版本
```

```bash
git tag v0.4.0       # ← 仓库的版本快照，标记某个 commit
```

**这两者没有自动关联**，不会互相校验。

### 对下游的影响

下游通过 Git rev 安装时：


| 下游写法                                | 拿到的代码            | pyproject.toml 里的 version |
| ----------------------------------- | ---------------- | ------------------------- |
| `rev = "master"`                    | master 最新 commit | 不参与选择，装到什么就是什么            |
| `rev = "v0.4.0"` / `tag = "v0.4.0"` | 该 tag 指向的 commit | 不校验，可能匹配也可能不匹配            |
| `rev = "911ea0ef"`                  | 精确 commit        | 不校验                       |


**结论：通过 Git 源安装时，`version` 字段只是信息标注，不影响安装行为。真正决定版本的是 `rev` 指向的 Git commit。**

### pyproject.toml 中 version 字段的实际用途

1. **发布到 PyPI** — PyPI 用它做版本索引，下游可以写 `model-hub-sdk>=0.4.0`
2. **运行时查询** — `importlib.metadata.version("model-hub-sdk")` 返回此值，用于日志/调试

---

## 五、Git 源 vs PyPI 发布


|          | Git 源安装                  | PyPI 发布                |
| -------- | ------------------------ | ---------------------- |
| **发布流程** | 无，直接指向 commit/branch/tag | 需要 CI 构建并上传到 PyPI      |
| **安装速度** | 慢（clone + 构建）            | 快（下载预构建 wheel）         |
| **访问控制** | 需要 Git 仓库权限 + token      | 公开包任何人能装；私有需要私有 PyPI   |
| **版本约束** | 只能精确指定（不支持范围）            | 支持范围约束（`>=0.4.0,<1.0`） |
| **适用场景** | 内部项目、快速迭代                | 正式发布、开源、多团队共用          |


**关键区别**：Git 源不支持版本范围语法：

```toml
# ✅ PyPI：支持范围约束
dependencies = ["model-hub-sdk>=0.4.0,<1.0"]

# ❌ Git 源：不支持范围，只能精确指定
model-hub-sdk = { git = "...", rev = ">=0.4.0" }  # 报错
```

---

## 六、`rev` 的值：branch vs tag vs commit SHA


| 类型         | 示例                 | 可复现性             |
| ---------- | ------------------ | ---------------- |
| branch     | `rev = "master"`   | ❌ 每次可能不同         |
| tag        | `tag = "v0.4.0"`   | ✅ 稳定（除非 tag 被移动） |
| commit SHA | `rev = "911ea0ef"` | ✅ 完全精确           |


推荐优先级：**commit SHA > tag > branch**

> 即使写 `rev = "master"`，uv 也会在 `uv.lock` 中锁定实际 commit SHA，团队内通过 lock 文件保证可复现。

---

## 七、总结：内部项目的最佳实践

```toml
# pyproject.toml

[project]
dependencies = [
    "model-hub-core",
    "model-hub-sdk",
]

[tool.uv.sources]
# 推荐用 tag，兼顾可读性和可复现性
model-hub-core = { git = "https://token@github.com/Org/repo.git", subdirectory = "packages/core", tag = "v0.4.0" }
model-hub-sdk = { git = "https://token@github.com/Org/repo.git", subdirectory = "packages/sdk-python", tag = "v0.4.0" }
```

```bash
# 安装
uv sync

# 带可选依赖
uv sync --extra all
```

不需要发布到 PyPI，不需要搭建额外基础设施。Git tag + uv.lock 即可满足内部项目的版本管理需求。