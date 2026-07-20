# 凭证扫描：gitleaks + detect-secrets 速查与详解

> [!abstract] 场景背景：给一个已经泄露过存量凭证的仓库接入「防止再次泄露」的扫描，核心难点是 **既要拦住新密钥、又不能被存量密钥卡死**。

---

## 一、全景：两个工具、两道关卡

接入凭证扫描，本质是把两个**独立工具**挂到两道**关卡**上。先记住这张表，后面全是它的展开：


| 关卡                 | 时机                | 跑的工具                      | 读的配置                                   | 能否绕过                       |
| ------------------ | ----------------- | ------------------------- | -------------------------------------- | -------------------------- |
| **pre-commit**     | 本地 `git commit` 时 | gitleaks + detect-secrets | `.gitleaks.toml` + `.secrets.baseline` | 能（`--no-verify` / 不装 hook） |
| **CI secret-scan** | push 到远端后         | 只有 gitleaks               | 只有 `.gitleaks.toml`                    | 不能（服务端强制）                  |


由这张表直接推出三条结论，是理解全篇的钥匙：

1. `.gitleaks.toml` **是双关卡共用配置**（gitleaks 本地和 CI 都跑）——最重要的文件。
2. `.secrets.baseline` **只有本地 pre-commit 用**，CI 不读——它是「本地补充」，不是「CI 底线」。
3. **真正不可绕过的强制底线是 CI 里的 gitleaks**。detect-secrets 只在本地生效，一旦 `--no-verify` 或没装 hook 就完全不参与。

> [!tip] 两个工具是互补，不是重复
> gitleaks 靠**内置正则规则库**认主流厂商格式（AWS 的 `AKIA...`、GitHub 的 `ghp_...`）；detect-secrets 靠**熵检测 + plugin** 兜住不符合常见格式的自定义密钥。两个角度都扫，覆盖面更全。



### 核心心智模型：只拦增量、存量走豁免

给一个**已经有存量泄露**的仓库接扫描，最容易踩的坑是「全量扫描即失败」——第一天 CI 就因为历史遗留的几十处明文凭证全红，把所有人卡住。

正确策略是**只拦增量、存量登记豁免**：

- 存量凭证（清理前）**先登记在案**，扫描时放行——不阻塞日常开发。
- 只有**新出现、不在登记册上**的密钥才拦截。
- 每清理完一批存量（凭证轮换 + 移入 Secret Manager），就**从登记册删掉对应条目**，逐步收紧。终态是登记册为空。

两个配置文件就是这个策略的两种「登记册」，只是粒度不同：

- `.gitleaks.toml` —— **粗粒度**，按**文件路径**整体豁免。
- `.secrets.baseline` —— **细粒度**，按**单个密钥的哈希**逐条登记。

---



## 二、最小可用配置



### pre-commit（`.pre-commit-config.yaml`）

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.1
    hooks:
      - id: gitleaks                       # 只扫 staged 增量,读仓库根的 .gitleaks.toml

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ["--baseline", ".secrets.baseline"]   # 与基线比对,只报账外新密钥
        exclude: ^(\.secrets\.baseline|.*\.ipynb)$   # 排除基线自身和 notebook
```

装好后每次 `git commit` 自动跑。**新 clone 的仓库必须先** `pre-commit install`，否则 hook 不生效（这是本地机制，装没装 CI 管不到）。

### CI（`.github/workflows/ci.yaml` 里加一个 job）

```yaml
  secret-scan:
    runs-on: iops-runner-general
    steps:
      - uses: actions/checkout@v4
      - name: gitleaks
        run: |
          docker run --rm -v "$GITHUB_WORKSPACE:/repo" \
            ghcr.io/gitleaks/gitleaks:v8.30.1 \
            dir /repo --no-banner --redact --exit-code 1   # 命中即 exit 1 阻断

  ci:                          # 原有的构建 job
    needs: [cfg, secret-scan]  # 关键:构建依赖扫描,扫描红了构建就不跑
```

> [!warning] 用 gitleaks CLI，别用 gitleaks-action
> 官方 GitHub Action（gitleaks-action）自 v2.0.0 起改为**专有许可**，且**组织账号仓库需注册申请 license key**（免费但要填表 + 配 secret）。直接跑 gitleaks CLI / docker 镜像是 MIT 许可、无门槛。功能上我们只要「扫描 + 命中即阻断」，CLI 一行就够，不需要 action 的 GitHub 事件封装。

> [!note] 为什么 secret-scan 不拆成独立 workflow
> GitHub Actions **不支持跨 workflow 的** `needs` **依赖**。若把 secret-scan 拆到单独的 workflow 文件，它会和构建 workflow **并行独立跑**，构建无法 `needs` 它——扫描红了构建照样推镜像，「阻断」就失效了。所以**构建门禁型**的检查必须和被 gate 的 job 待在同一个 workflow 里。参见 [[github-actions-配置速查与详解]] §2.2 `needs`。

---



## 三、gitleaks 详解



### 3.1 是什么

Go 写的独立命令行扫描器（`brew install gitleaks`）。内置上百条正则规则匹配各厂商密钥格式。常用三种扫描模式：

```bash
gitleaks dir .                      # 扫工作区当前文件(CI 用这个)
gitleaks git .                      # 扫 git 全历史(历史有已知泄露时别开,必红)
gitleaks git --pre-commit --staged  # 扫暂存区(pre-commit hook 内部用这个)
```

旧版的 `detect` / `protect` 子命令自 v8.19 起已废弃，分别由 `dir`/`git` 和 `git --pre-commit --staged` 接替——网上旧教程常见 `protect --staged`，别照抄。



### 3.2 `.gitleaks.toml` 结构

放在**仓库根目录**，gitleaks 启动自动读取。结构就三部分：

```toml
[extend]
useDefault = true          # 继承 gitleaks 内置的上百条规则,不写就没有规则

[[allowlists]]             # 豁免块,可以有多个
description = "S7/AIS-64: sendx.py 阿里云 SMTP 密码"   # 说明:必须写清对应哪个清理任务
paths = ['''tools/sendx\.py$''']    # 正则匹配文件路径,命中的文件整体跳过扫描
```

字段含义：


| 字段                    | 作用                          |
| --------------------- | --------------------------- |
| `[extend] useDefault` | 是否继承内置规则库，**几乎必须** `true`   |
| `[[allowlists]]`      | 一个豁免块，`[[ ]]` 双括号表示可重复的数组元素 |
| `description`         | 豁免原因，**强制写清对应清理任务号**，方便日后删  |
| `paths`               | 文件路径正则列表，命中即整文件放行           |


`allowlists` 还支持按内容豁免（不常用）：`regexes`（匹配密钥值）、`stopwords`（含某词就放行）、`commits`（豁免指定 commit）。日常主要用 `paths`。

### 3.3 常用操作

**加豁免（少用）**——某存量密钥文件暂时清不掉：

```toml
[[allowlists]]
description = "S8/AIS-65: region_ranking.py Azure key（清理后删本块）"
paths = ['''plaud_summary/region_ranking\.py$''']
```

**删豁免（常做）——收紧**：对应清理任务完成、合入主干后，把整个 `[[allowlists]]` 块删掉。这是防线收紧的动作。

> [!danger] gitignore 文件绝不能加进豁免
> `.env`、`config/*.yaml` 这类本就不该进仓库的文件，**禁止**写进 `paths` 豁免。它们一旦被误提交，扫描必须拦得住——豁免了就等于给误提交开了绿灯。



### 3.4 命令速查

```bash
gitleaks dir . --no-banner --redact --exit-code 1        # 扫工作区,脱敏输出,命中 exit 1
gitleaks dir . --report-format json --report-path x.json --exit-code 0  # 出 JSON 报告不中断
```

- `--redact`：日志里密钥打码，避免二次泄露。
- `--exit-code 1`：命中就返回非 0（CI 靠这个阻断）；调试想「只看报告不中断」用 `--exit-code 0`。
- 注意：`dir` 模式扫的是**磁盘上所有文件**，包括 gitignore 的未跟踪文件——本地全量扫报出 `.env`、`config/*` 是正常噪音；CI 的 checkout 只有 tracked 文件，所以 CI 不会报这些。

---



## 四、detect-secrets 详解



### 4.1 是什么：工具 vs plugin

**detect-secrets 是独立工具**（Yelp 开源，`pip install detect-secrets`），不是谁的插件。容易混的是它 baseline 里的 `plugins_used`——那是**它肚子里的检测器**，两层关系：

```
detect-secrets              ← 工具(一个 pip 包/一条命令)
├── AWSKeyDetector          ← plugin:认 AWS 的 AKIA... key
├── GitHubTokenDetector     ← plugin:认 ghp_... token
├── Base64HighEntropyString ← plugin:认高熵 base64 串(兜底,认不出厂商但看着像密钥)
└── ...(共 27 个)
```

每个 plugin 认一类密钥，扫描时逐个跑，谁命中报谁。

### 4.2 `.secrets.baseline` 怎么生成

不是手写的，是 `scan` 命令的产物：

```bash
detect-secrets scan --exclude-files '.*\.ipynb$' --exclude-files '\.secrets\.baseline$' > .secrets.baseline
```

- `scan`：用全部 plugin 扫描 **git tracked 文件**（默认不含未跟踪 / gitignore 的文件，加 `--all-files` 才扫全部——所以 `.env`、`config/*` 这类未入库文件不会进台账）
- `--exclude-files`：跳过 notebook（输出全是 base64，误报不可控）和 baseline 自身
- `>`：把结果 JSON 写进文件

> [!important] 日常更新用 `scan --baseline`，别每次从零重扫
> 上面的完整命令只在**首次生成**时用。之后更新台账用 `detect-secrets scan --baseline .secrets.baseline`——它按基线里记录的 filters（含排除项）增量刷新：自动删掉已清理的条目、保留 audit 标记，口径不会跑偏。若坚持从零重扫，必须原样带上所有 `--exclude-files`，否则台账口径会变（比如突然把 ipynb 全扫进来）。



### 4.3 三段内容详解

生成的 JSON 分三段：

**①** `plugins_used` **—— 启用了哪些检测器**

锁定扫描口径，供下次重扫对比。部分带参数：

```json
{ "name": "Base64HighEntropyString", "limit": 4.5 }
```

`limit: 4.5` 是熵阈值——base64 串的「随机程度」超 4.5 才算可疑。

**②** `filters_used` **—— 降噪过滤器**

内置规则，排掉一看就不是密钥的东西：`is_potential_uuid`（像 UUID 的不算）、`is_templated_secret`（`{{ password }}` 模板占位不算）、`is_sequential_string`（`abcdef` 顺子不算）。最后一条 `should_exclude_file` 记着你传的 `--exclude-files`。

**③** `results` **—— 核心：存量密钥台账**

key 是文件路径，value 是每处疑似密钥。以 sendx.py 为例：

```json
"tools/sendx.py": [
  { "type": "Secret Keyword", "hashed_secret": "d0a49fab...", "line_number": 157, "is_verified": false }
]
```

每条 4 字段：


| 字段              | 含义                                                                                     |
| --------------- | -------------------------------------------------------------------------------------- |
| `type`          | 哪个 plugin 认出的、什么类型（`AWS Access Key` / `Secret Keyword` / `Base64 High Entropy String`） |
| `hashed_secret` | **密钥明文的 SHA1 哈希**（不是明文！）                                                               |
| `line_number`   | 在文件第几行                                                                                 |
| `is_verified`   | 是否人工核实过——注意：人工 `audit` 的结论写在**另一个字段** `is_secret` 里；`is_verified` 是在线主动验证的标记，默认 false，极少用到                                                              |


> [!note] 同一行常出现多条记录
> 比如某行的 AWS key 会同时被 `AWS Access Key` + `Base64 High Entropy String` + `Secret Keyword` 三个 plugin 命中(哈希相同)——分别从「是 AWS key」「是高熵串」「附近有 key= 字样」三个角度报，所以记 3 条。



### 4.4 为什么存哈希不存明文

`hashed_secret` 是 SHA1 哈希，**看到也反推不出原密钥**。这是这份台账能安全提交进 git 的前提——否则「防泄露的文件」自己就成了泄露源。detect-secrets 比对时对新扫到的密钥同样算哈希再比，哈希一致就认为是同一个已知密钥。（严格说，弱口令仍可能被字典碰撞出来，台账只是「相对安全」——不改变存量要尽快清理的目标。）

### 4.5 常用操作

**更新台账（存量清理后收紧）**：`detect-secrets scan --baseline .secrets.baseline`——已清理的密钥自动从台账消失，audit 标记与排除项保留。

**误报处理（新密钥其实是假的/示例值）**——两种方式：

```python
API_KEY = "example-not-real"  # pragma: allowlist secret  ← 行内注释豁免单行
```

或把它 audit 进台账（见下）。

**审阅台账 / 标记误报**：

```bash
detect-secrets audit .secrets.baseline   # 交互式逐条确认是真密钥还是误报
```

逐条判定的结论写入条目的 `is_secret` 字段（true = 真密钥待清理，false = 误报）。

**提交时提示 `The baseline file was updated`**：不是错误——hook 发现行号漂移自动刷新了台账，`git add .secrets.baseline` 后重新提交即可。

---



## 五、两者对比与分工


|       | gitleaks                   | detect-secrets                      |
| ----- | -------------------------- | ----------------------------------- |
| 语言/安装 | Go，`brew install gitleaks` | Python，`pip install detect-secrets` |
| 检测方式  | 内置正则规则库                    | plugin 检测器 + 熵检测                    |
| 配置文件  | `.gitleaks.toml`（路径豁免）     | `.secrets.baseline`（哈希台账）           |
| 跑在哪   | **本地 + CI**                | **只本地**                             |
| 豁免粒度  | 按文件路径                      | 按单个密钥哈希                             |
| 定位    | 强制底线                       | 本地补充                                |


**同一批存量债务，两个工具各记各的账**。清理完某文件的密钥后，**两边都要更新**：gitleaks 删对应 `[[allowlists]]` 块，detect-secrets 重新 `scan` 生成新台账。

---



## 六、日常工作流


| 场景                     | 你要做什么                                                                                  |
| ---------------------- | -------------------------------------------------------------------------------------- |
| **写代码正常提交**            | 啥都不用管，`pre-commit install` 过一次即可                                                       |
| **清理完一批存量密钥**（某清理任务完成） | ① `.gitleaks.toml` 删对应豁免块 ② `detect-secrets scan --baseline .secrets.baseline` 刷新台账                           |
| **提交被拦，确认是假密钥/示例值**    | gitleaks 侧加 `.gitleaks.toml` 豁免；detect-secrets 侧加 `# pragma: allowlist secret` 或 audit |
| **提交被拦，是真密钥**          | **都不动配置**——把密钥移到 Secret Manager / AppConfig，从代码里删掉                                     |


> [!tip] 核心心智：登记册只减不增
> 这两个文件是「已知存量债务登记册」，方向永远是**越删越少**。如果有人往里加**真实新密钥**的豁免，就是用错了——正确做法是把密钥移出代码，而不是把扫描器的嘴堵上。

---



## 七、验证方法

接入后必须验证真能拦住。造一个**假** AWS key（用假的，别用真的——推到远端也会留痕）：

```bash
cat > /tmp/canary.py <<'EOF'
AWS_ACCESS_KEY_ID = "AKIA2J7QX4M9V3PLW6TR"
AWS_SECRET_ACCESS_KEY = "9x2LmPq8Rt3uVw5yZa7bCd1eFg4hJk6nQs0iOp2X"
EOF
cp /tmp/canary.py ./canary.py && git add canary.py
pre-commit run gitleaks         # 应 Failed
pre-commit run detect-secrets   # 应 Failed
git rm --cached canary.py && rm canary.py   # 清理
```

两个 hook 都应 `Failed`。再推一个含假密钥的测试分支，确认 CI 的 secret-scan 变红、且下游构建未执行。

> [!warning] 国内首装 gitleaks hook 会很慢
> pre-commit 首次装 gitleaks hook 需要拉 Go 依赖编译，无代理会卡很久。先 `export GOPROXY=https://goproxy.cn,direct` 再 `pre-commit install-hooks`。

---



## 八、速查表


| 操作                   | 命令 / 动作                                                                                                       |
| -------------------- | ------------------------------------------------------------------------------------------------------------- |
| 装工具                  | `brew install gitleaks` / `pip install detect-secrets`                                                        |
| 启用 hook              | `pre-commit install`（新 clone 必做）                                                                              |
| 扫工作区（gitleaks）       | `gitleaks dir . --no-banner --redact --exit-code 1`                                                           |
| 首次生成基线              | `detect-secrets scan --exclude-files '.*\.ipynb$' --exclude-files '\.secrets\.baseline$' > .secrets.baseline` |
| 更新基线（清理后收紧） | `detect-secrets scan --baseline .secrets.baseline` |
| 审阅基线                 | `detect-secrets audit .secrets.baseline`                                                                      |
| 加 gitleaks 豁免        | `.gitleaks.toml` 加 `[[allowlists]]` 块（写清任务号）                                                                  |
| 单行豁免（detect-secrets） | 行尾加 `# pragma: allowlist secret`                                                                              |
| 手动跑单个 hook           | `pre-commit run gitleaks` / `pre-commit run detect-secrets`                                                   |
| 跳过本地检查（慎用）           | `git commit --no-verify`（CI 仍会拦）                                                                              |


---



## 关联笔记

- [[github-actions-配置速查与详解]] —— CI secret-scan job 挂在 workflow 里，`needs` 门禁、reusable workflow 见该篇
- [[static-code-analysis-guide]] —— pre-commit、flake8 等静态检查的整体框架
- [[nexus-私有包仓库入门]] —— 凭证治理的一环：把 requirements.txt 的 GitHub PAT 换成 Nexus 私有源

