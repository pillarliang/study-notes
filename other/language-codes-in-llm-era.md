# LLM 时代的语言标识指南：从 ISO 639 到 BCP 47

## 一、为什么语言代码很重要？

在 LLM 时代，语言标识的应用场景远不止传统的 i18n（国际化）：

- **语种检测**：识别用户输入的语言，路由到对应的处理逻辑
- **Prompt 切换**：根据语种选择不同的 System Prompt 或回复语言
- **语音转录（ASR）**：Whisper 等模型需要 language hint 来提升准确率
- **内容索引与检索（RAG）**：不同语言需要不同的分词、Embedding 策略
- **多语言摘要**：输入语言 A，输出语言 B，需要明确标识两端

这些场景都需要一套**统一、无歧义、可扩展**的语言标识体系。答案就是 **BCP 47**。

---

## 二、标准体系全景

语言标识并非只有一个标准，而是一个层层嵌套的体系：

```
┌─────────────────────────────────────────────────┐
│                    BCP 47                        │  ← 最终使用的标准
│  ┌───────────┐ ┌───────────┐ ┌───────────┐      │
│  │ ISO 639   │ │ ISO 15924 │ │ ISO 3166  │      │
│  │ 语言代码   │ │ 脚本代码   │ │ 地区代码   │      │
│  │ en, zh, ja│ │ Hans, Hant│ │ CN, TW, US│      │
│  └───────────┘ └───────────┘ └───────────┘      │
│                                                  │
│  组合方式: 语言[-脚本][-地区][-变体]               │
│  示例:     zh-Hans-CN / en-US / pt-BR            │
└─────────────────────────────────────────────────┘
```

### 2.1 ISO 639 —— 语言代码

ISO 639 定义了语言本身的标识符，有多个部分：

| 部分 | 长度 | 覆盖范围 | 示例 |
|------|------|---------|------|
| ISO 639-1 | 2 字母 | 184 种主要语言 | `en`, `zh`, `ja`, `ko`, `fr` |
| ISO 639-2 | 3 字母 | ~487 种语言 | `eng`, `zho`, `jpn` |
| ISO 639-3 | 3 字母 | 7000+ 所有已知语言 | `cmn`(普通话), `yue`(粤语), `wuu`(吴语) |

**关键认知**：ISO 639-1 的 `zh` 代表的是"中文"这个宏语言（macrolanguage），它不区分简繁体，也不区分普通话/粤语/吴语。这就是为什么单独用 `zh` 是不够的。

### 2.2 ISO 15924 —— 脚本（书写系统）代码

定义文字的书写系统，4 字母，首字母大写：

| 代码 | 脚本 | 说明 |
|------|------|------|
| `Latn` | 拉丁字母 | 英、法、德、西、葡... |
| `Hans` | 简体中文 | 中国大陆、新加坡、马来西亚 |
| `Hant` | 繁体中文 | 台湾、香港、澳门 |
| `Cyrl` | 西里尔字母 | 俄语、乌克兰语 |
| `Arab` | 阿拉伯字母 | 阿拉伯语、波斯语、乌尔都语 |
| `Jpan` | 日文混合 | 汉字 + 平假名 + 片假名 |
| `Kore` | 韩文 | 谚文 + 汉字 |
| `Deva` | 天城文 | 印地语、梵语 |

大多数语言**只有一种通用脚本**（英语 → 拉丁、日语 → 日文），所以脚本代码通常可以省略。中文是需要脚本代码的典型例外。

### 2.3 ISO 3166 —— 地区代码

定义国家/地区，2 字母大写：

| 代码 | 地区 |
|------|------|
| `CN` | 中国大陆 |
| `TW` | 台湾 |
| `HK` | 香港 |
| `US` | 美国 |
| `GB` | 英国 |
| `BR` | 巴西 |
| `PT` | 葡萄牙 |

### 2.4 BCP 47 —— 将它们组合起来

BCP 47（Best Current Practice 47）是 IETF 制定的语言标签规范，定义于 RFC 5646。它不是一个独立的编码表，而是一套**组合规则**：

```
语言代码 [-脚本代码] [-地区代码] [-变体]
   │          │          │        │
 必选       可选       可选     可选
```

**组合示例**：

| BCP 47 标签 | 含义 |
|-------------|------|
| `en` | 英语（通用） |
| `en-US` | 美式英语 |
| `en-GB` | 英式英语 |
| `zh-Hans` | 简体中文（不限地区） |
| `zh-Hant` | 繁体中文（不限地区） |
| `zh-Hans-CN` | 中国大陆简体中文 |
| `zh-Hant-TW` | 台湾繁体中文 |
| `zh-Hant-HK` | 香港繁体中文 |
| `pt-BR` | 巴西葡萄牙语 |
| `pt-PT` | 欧洲葡萄牙语 |
| `sr-Cyrl` | 塞尔维亚语（西里尔字母） |
| `sr-Latn` | 塞尔维亚语（拉丁字母） |

**核心原则：能短则短，需长则长。** `en` 够用就不写 `en-Latn-US`。

---

## 三、中文：最需要关注的特例

### 3.1 为什么中文必须区分脚本？

世界上大多数语言只用一种文字，但中文同时存在**两套书写系统**，且差异巨大：

```
简体：语言 / 国际化 / 计算机
繁体：語言 / 國際化 / 計算機
```

这不是"方言差异"，而是**文字系统差异**——同一段语音可以写成两种完全不同的文本。这在世界语言中非常罕见（类似的还有塞尔维亚语的西里尔/拉丁双脚本）。

### 3.2 `zh-CN` vs `zh-Hans`：到底该用哪个？

| 标签 | 语义 | 适用场景 |
|------|------|---------|
| `zh-CN` | 中国大陆的中文 | 绑定了地区，暗示简体 |
| `zh-TW` | 台湾的中文 | 绑定了地区，暗示繁体 |
| `zh-Hans` | 简体中文（纯文字维度） | 不绑定地区 |
| `zh-Hant` | 繁体中文（纯文字维度） | 不绑定地区 |

**推荐使用 `zh-Hans` / `zh-Hant`**，原因：

1. **更精确**：简繁体是文字问题，不是地区问题。马来西亚华人用简体，但他们不是 `zh-CN`
2. **无歧义**：`zh-CN` 在技术上并不严格等于"简体"，只是"中国大陆地区的中文"
3. **Apple 和 Unicode CLDR 的推荐**：iOS/macOS 系统内部使用 `zh-Hans` / `zh-Hant`

### 3.3 绝对不要使用的写法

| 写法 | 问题 |
|------|------|
| `zh` | 无法区分简繁体 |
| `cn` | 不是语言代码，是国家代码 |
| `chinese` | 不是任何标准 |
| `zh-CHS` / `zh-CHT` | 微软历史遗留的非标准写法 |

---

## 四、BCP 47 与各标准的关系

```
BCP 47 ⊃ ISO 639-1
BCP 47 ⊃ ISO 15924
BCP 47 ⊃ ISO 3166

即：所有 ISO 639-1 代码都是合法的 BCP 47 标签
    en = 合法的 BCP 47
    zh-Hans = 合法的 BCP 47
    zh-Hans-CN = 合法的 BCP 47
```

所以你不需要在 ISO 639 和 BCP 47 之间"选一个"——**用 BCP 47 就自动兼容了 ISO 639**。

---

## 五、不同场景的实践建议

### 5.1 国际化 App（i18n）

需要完整的地区信息来处理日期、货币、数字格式：

```
en-US    # 美式英语：MM/DD/YYYY, $
en-GB    # 英式英语：DD/MM/YYYY, £
zh-Hans-CN  # 大陆简体：yyyy年M月d日, ¥
zh-Hant-TW  # 台湾繁体：yyyy年M月d日, NT$
pt-BR    # 巴西葡语：DD/MM/YYYY, R$
pt-PT    # 欧洲葡语：DD/MM/YYYY, €
```

### 5.2 LLM 语种检测

只需要识别"这是什么语言/文字"，不需要地区，使用最短 BCP 47 标签：

```
en, ja, ko, fr, de, es, pt, ru, ar, th, vi, id, tr, pl, nl, sv, da, fi, ...
zh-Hans  # 简体中文
zh-Hant  # 繁体中文
```

### 5.3 语音识别（ASR）

Whisper 等模型使用 ISO 639-1 代码，但中文通常只接受 `zh`（不区分简繁，因为语音层面无差异）：

```python
# Whisper language hint
result = model.transcribe(audio, language="zh")  # 中文
result = model.transcribe(audio, language="en")  # 英语
```

语音转文字后，再用文本层面的语种检测来区分简繁体。

### 5.4 多语言 Embedding / RAG

不同语言的分词和 Embedding 策略可能不同，用 BCP 47 标签来路由：

```
zh-Hans → jieba 分词 → 中文 Embedding 模型
ja      → MeCab 分词 → 日文 Embedding 模型
en      → 直接空格分词 → 英文 Embedding 模型
```

---

## 六、LLM 时代的语种检测实践

传统语种检测依赖统计模型（如 Google 的 CLD3、fastText 的 lid.176），但在 LLM 时代，我们可以直接利用大模型的语言理解能力。以下是一个经过优化的 Prompt 实例：

```text
You are a precise language detection engine. Your task is to analyze the input
text and return the standard IETF BCP 47 language tag.

**Output Standards:**
1. **General Languages:** Use ISO 639-1 two-letter codes (e.g., 'en', 'ja', 'fr', 'ru').
2. **Chinese Exceptions:** You MUST distinguish between Simplified and Traditional script.
   - Return 'zh-Hans' for Simplified Chinese.
   - Return 'zh-Hant' for Traditional Chinese.
   - DO NOT use generic 'zh' or regional codes like 'zh-CN'/'zh-TW'.

**Rules:**
- **Majority Rule:** If the text contains mixed languages, identify the dominant language.
- **Strict Output:** Output ONLY the language code. No markdown, no punctuation, no explanation.

**Examples:**
Input: "Hello world" -> Output: en
Input: "你好世界" -> Output: zh-Hans
Input: "妳好世界" -> Output: zh-Hant
Input: "こんにちは" -> Output: ja
```

### 这个 Prompt 的设计要点

1. **默认走 ISO 639-1 短代码**：对绝大多数语言，2 字母足够且简洁
2. **中文单独规定脚本码**：唯一需要额外标注的语言，使用 `zh-Hans` / `zh-Hant`
3. **明确禁止模糊写法**：不允许 `zh`、`zh-CN`、`zh-TW`，消除歧义
4. **混合语言处理**：采用多数规则，避免返回多个代码的复杂性
5. **严格输出格式**：只返回代码本身，方便下游程序直接解析
6. **Few-shot 示例**：覆盖英/中简/中繁/日四种典型场景，引导模型行为

### 为什么用 LLM 做语种检测？

| 维度 | 传统方案（fastText/CLD3） | LLM 方案 |
|------|--------------------------|----------|
| 短文本准确率 | 较低（<20 字容易误判） | 高（理解语义） |
| 简繁体区分 | 需额外规则 | 原生支持 |
| 混合语言处理 | 通常只返回一个 | 可定制策略 |
| 延迟 | <1ms | 数百ms |
| 成本 | 几乎为零 | API 调用费用 |
| 离线可用 | 是 | 否（需 API） |

**结论**：如果你的系统已经在调用 LLM，语种检测可以作为前置步骤"顺便"完成，几乎零额外成本。如果需要极低延迟或离线场景，传统方案仍有其价值。

---

## 七、速查表

```
场景                     推荐标准          中文写法
─────────────────────────────────────────────────
LLM 语种检测              BCP 47 简化      zh-Hans / zh-Hant
App 国际化 (i18n)         BCP 47 完整      zh-Hans-CN / zh-Hant-TW
语音识别 (ASR)            ISO 639-1        zh
数据库/日志存储            BCP 47 简化      zh-Hans / zh-Hant
HTTP Accept-Language      BCP 47           zh-Hans, en;q=0.9
HTML lang 属性            BCP 47           <html lang="zh-Hans">
```

**一句话总结：BCP 47 是唯一需要记住的标准。大多数语言用 2 字母代码，中文加上 `-Hans` / `-Hant` 脚本后缀。**

---

> 本文基于 IETF RFC 5646 (BCP 47)、Unicode CLDR、ISO 639/15924/3166 等公开标准整理。
