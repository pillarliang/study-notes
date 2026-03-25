# Python 静态代码检查配置指南

本文档介绍如何在 Python 项目中配置 **Ruff + MyPy** 静态代码检查工具链。

## 目录

- [工具介绍](#工具介绍)
- [快速开始](#快速开始)
- [完整配置文件](#完整配置文件)
- [配置详解](#配置详解)
- [常用命令](#常用命令)
- [IDE 集成](#ide-集成)
- [CI/CD 集成](#cicd-集成)
- [常见问题](#常见问题)

---

## 工具介绍

### 为什么选择 Ruff + MyPy？

| 工具 | 作用 | 特点 |
|------|------|------|
| **Ruff** | Linter + Formatter | 用 Rust 编写，速度极快，替代 Flake8/isort/Black/pyupgrade |
| **MyPy** | 类型检查 | Python 官方推荐的静态类型检查器 |
| **Pre-commit** | Git 钩子管理 | 在 commit 前自动运行检查 |

### 工具对比

```
传统方案（5+ 工具）          现代方案（2 工具）
├── Flake8 (检查)            ├── Ruff (检查 + 格式化)
├── isort (import 排序)  →   └── MyPy (类型检查)
├── Black (格式化)
├── pyupgrade (语法升级)
└── MyPy (类型检查)
```

---

## 快速开始

### 1. 安装依赖

在 `pyproject.toml` 中添加开发依赖：

```toml
[dependency-groups]
dev = [
    "pre-commit>=4.0.1",
    "ruff>=0.8.4",
    "mypy>=1.14.1",
]
```

安装：

```bash
uv sync --group dev
```

### 2. 创建配置文件

需要创建两个文件：

```
项目根目录/
├── .pre-commit-config.yaml   # Pre-commit 钩子配置
└── pyproject.toml            # Ruff/MyPy 规则配置
```

### 3. 安装 Git 钩子

```bash
uv run pre-commit install
```

### 4. 运行检查

```bash
# 运行所有检查
uv run pre-commit run --all-files

# 或使用 Makefile
make lint
```

---

## 完整配置文件

### .pre-commit-config.yaml

```yaml
# .pre-commit-config.yaml
# Pre-commit 钩子配置 - 定义在 git commit 前运行哪些检查

repos:
  # ==================== 基础检查 ====================
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace      # 去除行尾空格
      - id: end-of-file-fixer        # 确保文件以空行结尾
      - id: check-yaml               # 检查 YAML 语法
      - id: check-json               # 检查 JSON 语法
      - id: check-added-large-files  # 防止提交超过 500KB 的大文件
        args: ['--maxkb=500']
      - id: check-ast                # 检查 Python 语法是否正确
      - id: check-merge-conflict     # 检查是否有未解决的合并冲突
      - id: debug-statements         # 检查是否有 debugger/pdb 语句

  # ==================== Ruff (Linter + Formatter) ====================
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.4
    hooks:
      - id: ruff                     # 运行 Linter
        args: [--fix]                # 自动修复可修复的问题
      - id: ruff-format              # 运行 Formatter

  # ==================== MyPy (类型检查) ====================
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.14.1
    hooks:
      - id: mypy
        # 添加项目依赖的类型定义包
        additional_dependencies:
          - types-requests
          - types-redis
        args: [--ignore-missing-imports]
```

### pyproject.toml（完整配置）

```toml
# ============================================================
#                    Ruff 配置
# ============================================================
# 文档: https://docs.astral.sh/ruff/configuration/

[tool.ruff]
# 每行最大长度
line-length = 88

# 目标 Python 版本
target-version = "py312"

# 排除的目录（不进行检查）
exclude = [
    ".git",
    ".venv",
    "venv",
    "__pycache__",
    ".mypy_cache",
    ".pytest_cache",
    "build",
    "dist",
    ".eggs",
    "*.egg-info",
    "migrations",      # 数据库迁移文件
    "node_modules",
]

# ==================== Linter 配置 ====================
[tool.ruff.lint]
# 启用的规则集
# 完整规则列表: https://docs.astral.sh/ruff/rules/
select = [
    "E",      # pycodestyle errors (缩进、空格等)
    "W",      # pycodestyle warnings
    "F",      # Pyflakes (未使用变量、未定义变量等)
    "I",      # isort (import 排序)
    "N",      # pep8-naming (命名规范)
    "UP",     # pyupgrade (自动升级旧语法)
    "B",      # flake8-bugbear (常见 bug 模式)
    "C4",     # flake8-comprehensions (推导式优化)
    "SIM",    # flake8-simplify (简化代码)
    "ARG",    # flake8-unused-arguments (未使用参数)
    "PTH",    # flake8-use-pathlib (推荐使用 pathlib)
    "ERA",    # eradicate (注释掉的代码)
    "PL",     # Pylint
    "RUF",    # Ruff 特有规则
]

# 忽略的规则
ignore = [
    "E501",    # 行长度 (由 formatter 处理)
    "B008",    # 函数调用作为默认参数 (FastAPI Depends 需要)
    "B904",    # raise without from (有时不需要)
    "PLR0913", # 函数参数过多 (有时不可避免)
    "PLR2004", # 魔法数字 (太严格)
]

# 允许自动修复的规则（即使是不安全的修复）
fixable = ["ALL"]
unfixable = []

# ==================== isort 配置 ====================
[tool.ruff.lint.isort]
# 已知的第一方包（你的项目包）
known-first-party = ["demo", "agents", "prompts", "app", "src"]

# 已知的第三方包
known-third-party = ["fastapi", "pydantic", "sqlalchemy", "redis"]

# import 分组之间添加空行
lines-after-imports = 2
lines-between-types = 1

# 强制单行导入
force-single-line = false

# ==================== 按文件类型配置 ====================
[tool.ruff.lint.per-file-ignores]
# 测试文件可以有更宽松的规则
"tests/**/*.py" = [
    "S101",    # 允许使用 assert
    "ARG",     # 允许未使用参数 (fixtures)
    "PLR2004", # 允许魔法数字
]
# __init__.py 允许未使用的导入
"__init__.py" = ["F401"]
# 迁移文件忽略所有检查
"migrations/**/*.py" = ["ALL"]

# ==================== Formatter 配置 ====================
[tool.ruff.format]
# 字符串引号风格: "double" 或 "single"
quote-style = "double"

# 缩进风格: "space" 或 "tab"
indent-style = "space"

# 是否跳过魔法尾随逗号
skip-magic-trailing-comma = false

# 行尾风格: "auto", "lf", "cr-lf", "native"
line-ending = "auto"

# docstring 代码格式化
docstring-code-format = true

# ============================================================
#                    MyPy 配置
# ============================================================
# 文档: https://mypy.readthedocs.io/en/stable/config_file.html

[tool.mypy]
# Python 版本
python_version = "3.12"

# 忽略缺少类型定义的第三方库
ignore_missing_imports = true

# ==================== 严格模式选项 ====================
# 建议逐步启用，不要一开始就全部开启

# 基础警告
warn_return_any = true           # 警告返回 Any 类型
warn_unused_configs = true       # 警告未使用的配置
warn_redundant_casts = true      # 警告多余的类型转换
warn_unused_ignores = true       # 警告未使用的 # type: ignore

# 类型检查严格程度（按需开启）
disallow_untyped_defs = false    # 是否要求所有函数都有类型注解
disallow_incomplete_defs = false # 是否禁止不完整的类型定义
check_untyped_defs = true        # 检查没有类型注解的函数体

# 其他选项
no_implicit_optional = true      # 禁止隐式 Optional
strict_equality = true           # 严格相等性检查

# ==================== 按模块配置 ====================
# 对特定模块使用不同的检查级别

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false    # 测试文件不强制类型注解

[[tool.mypy.overrides]]
module = "migrations.*"
ignore_errors = true             # 迁移文件忽略所有错误
```

### Makefile（便捷命令）

```makefile
.PHONY: install-hooks lint format ruff mypy check

# ========== 代码质量工具 ==========

# 安装 pre-commit hooks
install-hooks:
	uv run pre-commit install

# 运行所有 pre-commit 检查
lint:
	uv run pre-commit run --all-files

# 只运行 ruff formatter
format:
	uv run ruff format .

# 只运行 ruff linter（自动修复）
ruff:
	uv run ruff check . --fix

# 运行 mypy 类型检查
mypy:
	uv run mypy .

# 检查但不修复（CI 用）
check:
	uv run ruff check .
	uv run ruff format --check .
	uv run mypy .

# 显示 ruff 检查结果（不修复）
ruff-check:
	uv run ruff check .

# 更新 pre-commit hooks 到最新版本
update-hooks:
	uv run pre-commit autoupdate
```

---

## 配置详解

### Ruff 规则集说明

| 规则前缀 | 来源 | 说明 |
|----------|------|------|
| `E` | pycodestyle | 代码风格错误（缩进、空格） |
| `W` | pycodestyle | 代码风格警告 |
| `F` | Pyflakes | 逻辑错误（未使用变量、未定义变量） |
| `I` | isort | import 排序 |
| `N` | pep8-naming | 命名规范（类名、函数名、变量名） |
| `UP` | pyupgrade | 自动升级旧语法到新版本 |
| `B` | flake8-bugbear | 常见 bug 和设计问题 |
| `C4` | flake8-comprehensions | 推导式最佳实践 |
| `SIM` | flake8-simplify | 代码简化建议 |
| `ARG` | flake8-unused-arguments | 未使用的函数参数 |
| `PTH` | flake8-use-pathlib | 推荐使用 pathlib 替代 os.path |
| `ERA` | eradicate | 检测注释掉的代码 |
| `PL` | Pylint | Pylint 规则子集 |
| `RUF` | Ruff | Ruff 特有规则 |

### 推荐的规则级别

**入门级（最少干扰）：**

```toml
select = ["E", "F", "I", "W"]
```

**标准级（推荐）：**

```toml
select = ["E", "F", "I", "N", "W", "UP", "B", "C4"]
```

**严格级（追求高质量）：**

```toml
select = ["E", "F", "I", "N", "W", "UP", "B", "C4", "SIM", "ARG", "PTH", "ERA", "PL", "RUF"]
```

---

## 常用命令

### 日常开发

```bash
# 格式化代码
make format

# 检查并自动修复
make ruff

# 运行所有检查（推荐）
make lint
```

### 查看问题（不修复）

```bash
# 只检查，不修复
uv run ruff check .

# 检查特定文件
uv run ruff check path/to/file.py

# 检查特定规则
uv run ruff check . --select E501
```

### 忽略规则

```python
# 忽略整行
x = 1  # noqa: E501

# 忽略多个规则
x = 1  # noqa: E501, F401

# 忽略整个文件（文件顶部）
# ruff: noqa

# 忽略文件中的特定规则
# ruff: noqa: E501
```

### 更新工具版本

```bash
# 更新 pre-commit hooks
make update-hooks

# 或手动更新
uv run pre-commit autoupdate
```

---

## IDE 集成

### VS Code

安装扩展：

- **Ruff** (charliermarsh.ruff)
- **Mypy Type Checker** (ms-python.mypy-type-checker)

配置 `.vscode/settings.json`：

```json
{
    // 使用 Ruff 作为格式化工具
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff",
        "editor.formatOnSave": true,
        "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
        }
    },

    // Ruff 配置
    "ruff.configurationPreference": "filesystemFirst",

    // MyPy 配置
    "mypy-type-checker.args": ["--config-file=pyproject.toml"],

    // 禁用其他格式化工具（避免冲突）
    "python.formatting.provider": "none"
}
```

### PyCharm

1. 安装 Ruff 插件：`Settings → Plugins → Ruff`
2. 启用 Ruff：`Settings → Tools → Ruff → Enable Ruff`
3. 启用保存时格式化：`Settings → Tools → Actions on Save → Reformat code`

---

## CI/CD 集成

### GitHub Actions

创建 `.github/workflows/lint.yml`：

```yaml
name: Lint

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --group dev

      - name: Run Ruff (linter)
        run: uv run ruff check .

      - name: Run Ruff (formatter)
        run: uv run ruff format --check .

      - name: Run MyPy
        run: uv run mypy .
```

### GitLab CI

创建 `.gitlab-ci.yml`：

```yaml
stages:
  - lint

lint:
  stage: lint
  image: python:3.12-slim
  before_script:
    - pip install uv
    - uv sync --group dev
  script:
    - uv run ruff check .
    - uv run ruff format --check .
    - uv run mypy .
```

---

## 常见问题

### Q: Ruff 和 Black 有什么区别？

Ruff Formatter 是 Black 的替代品，两者输出几乎完全一致，但 Ruff 速度快 10-100 倍。如果项目已使用 Black，可以无缝切换到 Ruff。

### Q: 如何忽略特定文件或目录？

在 `pyproject.toml` 的 `exclude` 中添加：

```toml
[tool.ruff]
exclude = [
    "migrations",
    "legacy_code",
    "generated",
]
```

### Q: 如何处理第三方库缺少类型定义？

1. 安装类型定义包：`uv add types-requests --group dev`
2. 或在 `pyproject.toml` 中忽略：

```toml
[tool.mypy]
ignore_missing_imports = true
```

### Q: pre-commit 检查太慢？

1. 使用 Ruff 替代多个工具（已经做了）
2. 只检查暂存的文件（pre-commit 默认行为）
3. 跳过 MyPy（最慢）：

```yaml
# 临时跳过
git commit --no-verify
```

### Q: 如何逐步引入到现有项目？

1. **第一步**：只启用格式化

```toml
select = ["I"]  # 只排序 import
```

2. **第二步**：添加基础检查

```toml
select = ["E", "F", "I", "W"]
```

3. **第三步**：添加更多规则

```toml
select = ["E", "F", "I", "N", "W", "UP", "B", "C4"]
```

4. **第四步**：启用 MyPy

### Q: 团队成员不想安装 pre-commit？

在 CI 中强制检查，确保合并前代码符合规范。本地开发可以不强制，但 PR 必须通过检查。

---

## 参考链接

- [Ruff 文档](https://docs.astral.sh/ruff/)
- [Ruff 规则列表](https://docs.astral.sh/ruff/rules/)
- [MyPy 文档](https://mypy.readthedocs.io/)
- [Pre-commit 文档](https://pre-commit.com/)
- [Python 类型提示 PEP 484](https://peps.python.org/pep-0484/)
