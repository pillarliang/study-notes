# 03 - Providers：认证与配置

> 来源：https://pi.dev/docs/latest/providers

## 1. 认证范式：两条路

Pi 支持两类认证：

| 路径 | 触发方式 | 凭据存放 |
|------|---------|---------|
| **Subscription（OAuth）** | `/login` 选 provider | `~/.pi/agent/auth.json`，过期自动刷新 |
| **API Key** | 环境变量 或 `/login` 走 API-key provider | 环境变量 或 `~/.pi/agent/auth.json` |

`/logout` 清掉凭据。`auth.json` 文件权限是 `0600`（只有 owner 可读写）。

## 2. Subscription Providers（OAuth）

### OpenAI Codex

- 需要 ChatGPT Plus 或 Pro 订阅
- "Codex for OSS"——OpenAI 官方背书使用

### Claude Pro/Max

- Anthropic 订阅账号 OAuth 登录
- **关键**：第三方 harness 使用 Claude Pro/Max 时，**计入 "extra usage"，按 token 计费**，不计入 plan 配额
- 默认弹警告，可在 settings 里 `warnings.anthropicExtraUsage = false` 关掉（见 [[04-Settings-配置全集#warnings]]）

### GitHub Copilot

- 回车默认连 github.com；用 GitHub Enterprise Server 的输入企业域名
- 遇到 "model not supported"：先在 VS Code 里通过 Copilot Chat → 模型选择器 → "Enable" 启用该 model

## 3. API Key Providers 一览

环境变量名和 `auth.json` 的 key 名对照表：

| Provider | 环境变量 | `auth.json` Key |
|---|---|---|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| Azure OpenAI Responses | `AZURE_OPENAI_API_KEY` | `azure-openai-responses` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| Groq | `GROQ_API_KEY` | `groq` |
| Cerebras | `CEREBRAS_API_KEY` | `cerebras` |
| Cloudflare AI Gateway | `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID` + `CLOUDFLARE_GATEWAY_ID` | `cloudflare-ai-gateway` |
| Cloudflare Workers AI | `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID` | `cloudflare-workers-ai` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` | `vercel-ai-gateway` |
| ZAI | `ZAI_API_KEY` | `zai` |
| OpenCode Zen | `OPENCODE_API_KEY` | `opencode` |
| OpenCode Go | `OPENCODE_API_KEY` | `opencode-go` |
| Hugging Face | `HF_TOKEN` | `huggingface` |
| Fireworks | `FIREWORKS_API_KEY` | `fireworks` |
| Together AI | `TOGETHER_API_KEY` | `together` |
| Kimi For Coding | `KIMI_API_KEY` | `kimi-coding` |
| MiniMax | `MINIMAX_API_KEY` | `minimax` |
| MiniMax (CN) | `MINIMAX_CN_API_KEY` | `minimax-cn` |
| Xiaomi MiMo | `XIAOMI_API_KEY` | `xiaomi` |
| Xiaomi MiMo Token Plan (CN) | `XIAOMI_TOKEN_PLAN_CN_API_KEY` | `xiaomi-token-plan-cn` |
| Xiaomi MiMo Token Plan (AMS) | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` | `xiaomi-token-plan-ams` |
| Xiaomi MiMo Token Plan (SGP) | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` | `xiaomi-token-plan-sgp` |

## 4. `auth.json` 的 `key` 字段三种写法

`auth.json` 里每个 provider 条目的 `key` 字段支持三种表达：

| 形式 | 写法 | 行为 |
|------|------|------|
| **Shell 命令** | `"!command"`（开头是 `!`） | 执行命令，用 stdout 作 key；进程内缓存 |
| **环境变量名** | `"MY_ANTHROPIC_KEY"`（纯名字） | 解析为该环境变量的当前值 |
| **字面值** | `"sk-ant-..."` | 直接作为 key |

**Shell 命令形式特别有用**——例如把 key 存进 macOS Keychain：

```json
{
  "anthropic": { "key": "!security find-generic-password -ws 'anthropic'" }
}
```

每次 Pi 启动时执行一次 `security` 命令拿明文 key，避免把 key 明文写进配置文件。

## 5. 云厂商详细设置

### Azure OpenAI

```bash
export AZURE_OPENAI_API_KEY=...
export AZURE_OPENAI_BASE_URL=https://your-resource.openai.azure.com
# 或者：
export AZURE_OPENAI_RESOURCE_NAME=your-resource

# 可选：
export AZURE_OPENAI_API_VERSION=2024-02-01
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP=gpt-4=my-gpt4,gpt-4o=my-gpt4o
```

- 支持 `cognitiveservices.azure.com` 端点
- 根端点会被自动 normalize 成 `/openai/v1`
- `DEPLOYMENT_NAME_MAP` 把标准 model id 映射到 Azure 上的自定义 deployment 名

### Amazon Bedrock

三种凭据：

1. **AWS Profile**：`AWS_PROFILE=your-profile`
2. **IAM keys**：`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`
3. **Bearer token**：`AWS_BEARER_TOKEN_BEDROCK`

可选 `AWS_REGION`（默认 `us-east-1`）。也支持 ECS task role（`AWS_CONTAINER_CREDENTIALS_*`）和 IRSA（`AWS_WEB_IDENTITY_TOKEN_FILE`）。

**Bedrock 专属注意点**：

- 对可识别 model ID 的 Claude，自动启用 prompt caching
- 用 application inference profile ARN 时，需 `AWS_BEDROCK_FORCE_CACHE=1` 才会下发 cache point
- 代理与调试：`AWS_ENDPOINT_URL_BEDROCK_RUNTIME`（自定义端点）、`AWS_BEDROCK_SKIP_AUTH=1`（跳过签名）、`AWS_BEDROCK_FORCE_HTTP1=1`（强 HTTP/1.1）

使用例：

```bash
pi --provider amazon-bedrock --model us.anthropic.claude-sonnet-4-20250514-v1:0
```

### Cloudflare AI Gateway

需要：`CLOUDFLARE_API_KEY`（通过 `/login` 或环境变量）+ `CLOUDFLARE_ACCOUNT_ID` + `CLOUDFLARE_GATEWAY_ID`。

可路由到 OpenAI、Anthropic、Workers AI。四种上游认证模式：

| 模式 | 请求侧 auth | 上游 auth | 适合 |
|------|------------|----------|------|
| **Workers AI** | 只用 CF token | Cloudflare 原生 | 跑 Workers AI 模型 |
| **Unified billing** | 只用 CF token | Cloudflare 自付上游、扣信用 | 统一在 CF 出账 |
| **Stored BYOK** | 只用 CF token | CF 从 dashboard 注入用户的 provider key | 上游 key 集中存在 CF |
| **Inline BYOK** | CF token + 上游 `Authorization` header | 请求侧自带上游 key | 临时切换上游 key |

文档推荐日常用 **unified billing** 或 **stored BYOK**。

### Cloudflare Workers AI

只需 `CLOUDFLARE_API_KEY` + `CLOUDFLARE_ACCOUNT_ID`。Pi 自动加 `x-session-affinity` header 触发 prefix cache 折扣。

```bash
pi --provider cloudflare-workers-ai --model "@cf/moonshotai/kimi-k2.6"
```

### Google Vertex AI

走 ADC（Application Default Credentials）：

```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT=your-project
export GOOGLE_CLOUD_LOCATION=us-central1
```

或者用 service account：`GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa.json`。

## 6. Custom Providers

- **走 `models.json`**：支持 Ollama、LM Studio、vLLM，以及任何能讲 OpenAI Completions / OpenAI Responses / Anthropic Messages / Google Generative AI API 的服务
- **走 extension**：需要自定义 API 逻辑或 OAuth flow 时，用 TypeScript extension 实现

## 7. 凭据解析优先级

同一个 provider 多处都配了 key 时，按以下顺序选：

1. CLI 上的 `--api-key` flag
2. `auth.json` 里的条目（API key 或 OAuth token）
3. 环境变量
4. `models.json` 里 custom provider 的 key

> **`auth.json` 优先级高于环境变量**。临时切 key 用 CLI flag 最稳。
