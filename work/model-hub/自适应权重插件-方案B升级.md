# 自适应权重插件 - 方案 B 升级说明

> 本文档记录 [自适应权重插件设计方案.md](自适应权重插件设计方案.md) 的关键修订：自适应权重插件采用**重分配公式**（按原权重比例把降权部分分给健康 endpoint），并新增 **`recovery_rate` 速率限制**。两者是配套方案，必须同时启用。

---

## 一、变更动机

升级前的实现"各 endpoint 独立降权，不动其他 endpoint 的 base weight"。评审中发现两个不能接受的问题。

### 问题 1：多端点同时故障时小权重 endpoint 无法承接流量

配置示例（取自 dev-endpoints-model-hub.yaml 的 summary:auto 思路）：

```yaml
auto-gemini-2-5-pro: weight: 66
auto-gpt-5:          weight: 34
auto-gpt-4-1:        weight: 1
auto-o3:             weight: 1
```

如果只让降权 endpoint 自降权、不重分配，gemini + gpt-5 同时 100% 429（multiplier 都=0.1）时：


|           | gemini | gpt-5 | gpt-4-1 | o3   |
| --------- | ------ | ----- | ------- | ---- |
| effective | 6.6    | 3.4   | 1       | 1    |
| 占比        | 55.0%  | 28.3% | 8.3%    | 8.3% |


两个主力都打到下限了，gpt-4-1 / o3 占比也才到 8.3%——撑不住业务流量。

weight=1 的语义本应是"日常少用、应急可顶上"。仅靠"自降权"会让 base weight 同时扮演"日常分配"和"容量上限"两个角色，与配置者意图不符。

### 问题 2：恢复时刻的瞬间跳变会引发振荡

multiplier 完全跟随滑动窗口算出的 target。窗口里最后一个 429 过期那一瞬间，error_rate 从 X 跳到 0，target 从低位跳到 1.0，multiplier 也瞬间跳。

后果在双向同时发生：

- **主力 endpoint**：流量瞬间 ×N 涌回上游。"过去 120 秒没 429" ≠ "上游配额完全恢复"——很大概率再次 429，又触发降权，又把流量倒给备份，**振荡**
- **备份 endpoint**：连接池 / 缓存刚预热好承担应急流量，瞬间被砍，资源浪费 + 上游侧速率突变

---

## 二、变更总览


| 章节   | 变更内容                                                     |
| ---- | -------------------------------------------------------- |
| §3   | 明确采用方案 B 重分配公式，并保留 multiplier dict 接口，用 `multiplier > 1.0` 表达补偿 |
| §3.8 | 新增多 `WeightAdjustmentProvider` 的降权 / 加成分量合并语义                 |
| §4   | `recovery_rate` 改为基于降权起点限速，并要求查询接口无副作用                    |
| §5   | 新增 `base_weights` 协议兼容性要求，兼容旧版无参插件                         |
| §6   | yaml 配置增加 `recovery_rate: 0.05`                            |
| §8   | 实现影响范围补充 engine merge、Protocol 兼容和端到端路由测试                  |


---

## 三、核心机制 1：方案 B 重分配公式

**原理**：降权 endpoint 减少的权重，按健康 endpoint 的原权重比例分配回去。

### 3.1 为什么需要重分配：仅靠分母效应不够

加权随机路由的本质：`endpoint 被选中概率 = weight_i / Σ weight`。改变占比只有两条路径：

1. 把分子（自己的 weight）变小 → 自己占比下降
2. 把分母（总和）变小 → 别人占比被动上升

如果只靠路径 1（自降权），剩下端点占比通过"分母变小"被动上升，**被动上升受 base weight 限制**。

gemini 和 gpt-5 都打到 multiplier=0.1（base=66:34:1:1）：

```
分母 = 6.6 + 3.4 + 1 + 1 = 12
gpt-4-1 占比 = 1 / 12 = 8.3%
```

base=1 的端点天花板就是 8.3%——因为它的"分子"始终是 1，不动分子永远到不了更高占比。要让小端点真正能承接流量，必须显式把降权部分加到健康端点的"分子"上，这就是重分配。

### 3.2 为什么按"原权重比例"分

配置者写下 `gpt-5: 34, gpt-4-1: 1` 时已经表达了"gpt-5 比 gpt-4-1 重要 34 倍"的偏好。应急时仍应尊重——gpt-5 该比 gpt-4-1 多接 34 倍流量。

| 分配方式                | bonus 比例（gpt-5、gpt-4-1、o3）  | 评价         |
| ------------------- | -------------------------- | ---------- |
| 平均分（每端点等额）          | 1 : 1 : 1                  | ❌ 抹平了配置偏好  |
| 按当前 effective 分     | 引入循环（自己分给自己）               | ❌ 数学不闭合    |
| **按原 base weight 分** | **34 : 1 : 1**             | ✓ 保留配置偏好   |

### 3.3 公式定义

```
D = {i : multiplier_i < 1.0}    降权集合（贡献减少量，不收 bonus）
H = {i : multiplier_i = 1.0}    健康集合（receivers）
```

> H 严格用 `multiplier = 1.0` 判定，不是 `m > 0.5` 这种宽松判定。原因：让本身已有压力的端点再被加 bonus 会立刻被压垮，必须保证 receivers 真健康。

**减少总量**：

```
Δ = Σ_{i ∈ D} base_weight_i × (1 - multiplier_i)
```

**接收方加成**（按原权重比例）：

```
bonus_j = Δ × base_weight_j / Σ_{k ∈ H} base_weight_k    （j ∈ H）
```

**最终 effective weight**：

```
降权方：effective_i = base_weight_i × multiplier_i
健康方：effective_j = base_weight_j + bonus_j
```

**接口表达**：

`get_weight_multipliers()` 仍返回 multiplier dict，不改成 effective weight dict。方案 B 的补偿通过健康端点 `multiplier > 1.0` 表达：

```
降权方：返回 multiplier_i
健康方：返回 effective_j / base_weight_j
```

引擎继续使用 `adjusted_weight = round(base_weight × multiplier)` 计算最终路由权重，并保证最终权重下限为 1。

### 3.4 完整数值演练（base = 66:34:1:1，双故障）

输入：

```
端点          base   error_rate   multiplier
gemini        66     100%         0.10
gpt-5         34     100%         0.10
gpt-4-1       1      0%           1.00
o3            1      0%           1.00
```

**Step 1 - 分组**：

```
D = {gemini, gpt-5}
H = {gpt-4-1, o3}
```

**Step 2 - 减少总量**：

```
gemini 贡献 = 66 × (1 - 0.10) = 59.4
gpt-5  贡献 = 34 × (1 - 0.10) = 30.6
Δ = 59.4 + 30.6 = 90
```

**Step 3 - bonus（按 H 内 base 比例分）**：

```
Σ_{k ∈ H} base = 1 + 1 = 2
bonus_gpt-4-1 = 90 × (1/2) = 45
bonus_o3      = 90 × (1/2) = 45
```

**Step 4 - effective**：

```
gemini   = 66 × 0.10 = 6.6
gpt-5    = 34 × 0.10 = 3.4
gpt-4-1  = 1 + 45    = 46
o3       = 1 + 45    = 46
```

总和 = 6.6 + 3.4 + 46 + 46 = 102（= 原始总和）

### 3.5 两个关键数学性质

**性质 1：总权重守恒（H 非空时）**

```
Σ effective = Σ_D (base × m) + Σ_H (base + bonus)
            = Σ_D (base × m) + Σ_H base + Δ
            = Σ_D (base × m) + Σ_H base + Σ_D base × (1 - m)
            = Σ_H base + Σ_D base
            = Σ base
```

含义：流量在端点间转移，但**总流量不变**。

**性质 2：H 内相对比例不变**

对任意 j, k ∈ H：

```
(base_j + bonus_j) / (base_k + bonus_k)
= base_j × (1 + Δ/Σ_H base) / (base_k × (1 + Δ/Σ_H base))
= base_j / base_k
```

含义：健康端点之间在应急时**仍按原 base 比例分配新增流量**，配置者表达的偏好被严格保留。这是"按原权重比例分"的数学回报；换成平均分这个性质就破坏了。

### 3.6 边界情况（公式自然退化）

| 情况          | 条件         | 公式表现           | 实际效果                                                |
| ----------- | ---------- | -------------- | --------------------------------------------------- |
| D 空集        | 无端点降权      | Δ=0, bonus=0   | effective = base，等同未启用                              |
| **H 空集**    | 全部端点都降权    | 分母为 0（除零）      | 实现需特判：不重分配，effective = base × multiplier |
| D、H 都非空     | 正常情况       | 按公式计算          | 性质 1 + 2 成立                                         |

### 3.7 效果对照与 weight 语义变化

按 §3.4 的演算结果，gemini + gpt-5 同时 100% 429 时：


|           | gemini | gpt-5 | gpt-4-1   | o3        |
| --------- | ------ | ----- | --------- | --------- |
| effective | 6.6    | 3.4   | **46.0**  | **46.0**  |
| 占比        | 6.5%   | 3.3%  | **45.1%** | **45.1%** |


gpt-4-1 / o3 真正承接 ~45% 流量，业务在主力全炸时不中断。

**weight 语义的隐含变化**：base weight 从"日常分配 = 容量上限"变为"日常分配 + 应急时按原比例放大"。配置者通过 base weight 表达"应急时分到的相对份额"。

### 3.8 多 WeightAdjustmentProvider 的合并语义

当前内置插件只有自适应权重一个 `WeightAdjustmentProvider`，但 engine 的扩展点允许业务方或后续内置插件同时提供权重调整。例如：

- 自适应权重插件：主力 429 后给健康 backup 返回 `46.0` 的方案 B 加成
- 延迟 / 成本 / 区域容量插件：发现同一个 backup 也有压力，返回 `0.5` 的降权

这两类 multiplier 不能放进同一个标量里用 `min()` 或 `max()` 竞争，否则 `min(46.0, 0.5) = 0.5` 会把方案 B 的补偿完全吞掉。

合并规则：

```
降权分量 penalty = min(所有 < 1.0 的 multiplier)，默认 1.0
加成分量 bonus   = max(所有 > 1.0 的 multiplier)，默认 1.0
最终 multiplier  = penalty × bonus
```

示例：

```
base_weight = 1
方案 B 加成 = 46.0
其他插件降权 = 0.5

最终 multiplier = 46.0 × 0.5 = 23.0
最终 weight = round(1 × 23.0) = 23
```

含义：保留应急补偿，同时尊重另一个插件的降权信号。

---

## 四、核心机制 2：recovery_rate 速率限制

**原理**：降权立即生效（出问题立刻减压），恢复线性爬升（怀疑式信任，慢慢加压）。

**公式**：

```
now = time.monotonic()
target = max(min_weight_ratio, 1.0 - error_rate × penalty_factor)

if target < prev_multiplier:
    multiplier = target                                              # 下行：立即生效
    downgrade_started_at = now
    downgrade_start_multiplier = target
elif recovery_rate <= 0 or downgrade_started_at is None:
    multiplier = target                                              # 不限速，或当前没有恢复时间线
else:
    elapsed = now - downgrade_started_at
    cap = downgrade_start_multiplier + recovery_rate × elapsed
    multiplier = min(target, max(prev_multiplier, cap))              # 上行：按降权起点限速
```

**recovery_rate 的物理意义**：multiplier 每秒最多上升的量。

`elapsed` 必须基于"最近一次下行降权的起点"，不能基于"上次查询时间"。否则恢复速度会被查询频率污染：高频查询和低频查询得到不同恢复曲线，监控抓取甚至可能把恢复状态提前推进。时间源使用 `time.monotonic()`，避免系统时间回拨或 NTP 跳变影响限速。

`recovery_rate = 0.05`：每秒最多上升 0.05。从 multiplier=0.1 爬到 1.0 需上升 0.9，所以最少 0.9 / 0.05 = **18 秒**。"最少"是因为 target 也可能爬得更慢（取决于滑动窗口里 429 过期的节奏），multiplier 取两者较小值。

**不加 recovery_rate 的反例**：


| 时刻         | gemini multiplier | gemini 流量份额 | gpt-4-1 流量份额 |
| ---------- | ----------------- | ----------- | ------------ |
| t=180s 之前  | 0.10              | 6.5%        | 45.1%        |
| t=180s 那一瞬 | **1.00**          | **64.7%**   | **1.0%**     |
| 变化         | -                 | ×11         | ÷45          |


主力 ×11 瞬涌可能再次 429；备份 ÷45 瞬卸资源浪费。

**加 recovery_rate=0.05 后**：

```
t=180s    m=0.10    gemini  6.5%, gpt-4-1 45.1%
t=183s    m=0.25    gemini 16.2%, gpt-4-1 37.7%
t=189s    m=0.55    gemini 35.6%, gpt-4-1 23.0%
t=198s    m=1.00    gemini 64.7%, gpt-4-1  1.0%
```

18 秒线性过渡。给上游配额真正恢复的时间，给备份平滑卸载的时间。

**与重分配的耦合关系**：重分配让恢复时刻同时双向变化（主力涨、备份跌），冲击比纯自降权更强（备份份额从 ~46% 跌到 ~1%）。**recovery_rate 是重分配机制的必备配套，不是可选项**。

**查询接口要求**：外部调试 / 监控方法（如 `get_effective_multiplier()`）必须是无副作用查询，只做纯计算，不更新 `prev_multiplier`，不推进 `downgrade_started_at`，也不清空恢复状态。只有真实参与路由的 `get_weight_multipliers()` 才能提交新的 multiplier 状态。

---

## 五、协议兼容性

方案 B 给 `get_weight_multipliers()` 增加了 `base_weights` 输入，用于计算健康端点的补偿。但这是插件扩展点，必须兼容已经存在的第三方实现。

需要兼容三种签名：

```python
def get_weight_multipliers(self) -> dict[str, float]: ...
def get_weight_multipliers(self, base_weights: dict[str, int] | None = None) -> dict[str, float]: ...
def get_weight_multipliers(self, *, base_weights: dict[str, int] | None = None) -> dict[str, float]: ...
```

engine 负责用签名检查选择调用方式，不能无条件传位置参数，否则旧插件会抛：

```text
TypeError: get_weight_multipliers() takes 1 positional argument but 2 were given
```

---

## 六、配置变更

### 新的 yaml 配置

```yaml
plugins:
  adaptive_weight:
    enabled: true
    window_size_seconds: 120
    penalty_factor: 1.5
    min_weight_ratio: 0.1
    recovery_rate: 0.05            # 新增：multiplier 上行速率（每秒）。0=不限速
    error_codes: [429]
    soft_limit_on_429: true
    alert_threshold: 0.5
```

### recovery_rate 取值参考


| 值          | 含义        | 适用           |
| ---------- | --------- | ------------ |
| `0`        | 不限速（瞬间跳变） | 仅测试，生产不建议    |
| `0.05`（默认） | 18 秒爬满    | 大多数场景        |
| `0.02`     | 45 秒爬满    | 上游配额恢复慢，需更保守 |
| `0.1`      | 9 秒爬满     | 上游容量充足、恢复快   |


---

## 七、本次未采纳的一点

讨论过但本次决定**先不引入**，留作后续按需追加：

1. **`min_calls` 阈值（避免冷启动过激）**：endpoint 调用样本不足时，单次 429 就让 `error_rate=100%`，multiplier 直接打到下限。circuit_breaker 已有 `min_calls=5` 防护，adaptive_weight 暂未跟进

该点是可补强项，但不影响主线设计。**全降退化**已作为 §3.6 的边界行为纳入方案：H 为空时不重分配，effective = base × multiplier。

---

## 八、实现影响范围


| 文件                                             | 改动                                                                                                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `core/plugins/adaptive_weight.py`              | 保留 `get_weight_multipliers()` multiplier dict 接口；提供 `base_weights` 时用健康端点 multiplier > 1.0 表达方案 B 补偿；新增降权起点恢复时间线状态；拆分纯计算与提交状态逻辑                |
| `core/engine.py` `_apply_weight_adjustments()` | 调用 adjuster 时兼容新旧签名；多 adjuster 合并时拆分降权/加成分量；继续按 `round(ep.weight × multiplier)` 计算最终 weight，并保证下限为 1                                                              |
| `core/plugins/base.py`                         | Protocol 文档说明 `base_weights` 可选，并明确 multiplier 可能 > 1.0                                                                                 |
| `core/config/models.py`                        | `AdaptiveWeightPluginConfig` 新增 `recovery_rate: float = 0.05`                                                                               |
| `core/config/parser.py`                        | 解析 `recovery_rate` 字段                                                                                                                       |
| `core/tests/test_adaptive_weight.py`           | 新增重分配 + recovery_rate 限速测试                                                                                                                  |
| `core/tests/test_engine.py`                    | 新增旧签名兼容、多 adjuster 合并、方案 B 穿过 Router 的端到端占比测试                                                                                         |


---

## 九、一句话总结


| 机制            | 做什么                             | 解决什么                          |
| ------------- | ------------------------------- | ----------------------------- |
| 重分配           | 降权 endpoint 减少的权重按健康端点的原权重比例分回去 | 多端点故障时，weight=1 的应急端点也能真正承接流量 |
| recovery_rate | multiplier 上行受速率限制，下行立即生效       | 恢复时不让流量瞬间灌回上游，避免再次 429 引发振荡   |
