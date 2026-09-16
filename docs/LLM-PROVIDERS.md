# LLM 供应商配置指南

网关的 LLM 层是**多供应商 + 故障转移**(`poc/backend/llm_router.py`)。所有供应商
按 `llm_providers.yml` 声明顺序尝试,第一个成功的返回;全失败才报错。

支持三类接入方式:

| 方式 | `kind` | 后端代码 | 适用 |
|------|--------|---------|------|
| **OpenAI 兼容端点** | `openai` | 原生 | 火山方舟、DeepSeek、通义、Moonshot、OpenAI、任意兼容网关 |
| **Azure OpenAI** | `azure` | 原生(阶段2) | 企业 Azure 订阅 |
| **AWS Bedrock / Google Vertex** | `openai`(经网关) | 无(阶段3,本文档) | Claude on Bedrock、Gemini on Vertex |

> 配置有两种入口:①设置页 UI(License 页 → Providers 卡);②直接编辑
> `llm_providers.yml`。UI 存盘即写该文件。字段一一对应。

---

## 1. OpenAI 兼容端点(默认路径)

绝大多数中国厂商与国际厂商都提供 OpenAI 兼容的 `/v1/chat/completions`。直接配
`kind: openai`(可省略,默认值):

```yaml
providers:
  - id: ark-primary
    kind: openai              # 默认,可省
    base_url: https://ark.cn-beijing.volces.com/api/v3
    api_key_env: ARK_API_KEY  # 或 api_key: <明文>
    model: <endpoint-id>
    timeout_s: 180
    tags: [primary]
  - id: deepseek-backup
    base_url: https://api.deepseek.com
    api_key_env: DEEPSEEK_API_KEY
    model: deepseek-chat
    tags: [backup]

routing:
  strategy: failover
```

常见 `base_url`:

| 厂商 | base_url | model |
|------|----------|-------|
| 火山方舟 | `https://ark.cn-beijing.volces.com/api/v3` | endpoint id |
| DeepSeek | `https://api.deepseek.com` | `deepseek-chat` |
| 通义千问 | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen-plus` |
| Moonshot | `https://api.moonshot.cn/v1` | `moonshot-v1-32k` |
| OpenAI | `https://api.openai.com/v1` | `gpt-4o` |

---

## 2. Azure OpenAI(原生,阶段2)

Azure 按 **deployment** 路由(不是模型名),且 `api_version` 必填(缺失 Azure
每次请求 404)。后端用 `AsyncAzureOpenAI` 处理。

```yaml
providers:
  - id: azure-gpt4o
    kind: azure
    base_url: https://<资源名>.openai.azure.com   # 资源终结点,不带 /openai
    api_version: "2024-06-01"                      # 必填
    api_key_env: AZURE_OPENAI_KEY
    model: <deployment 名称>                        # 注意:deployment,非模型名
    tags: [primary]
```

UI 侧:KIND 选 `azure` 后会出现 **API VERSION** 字段;MODEL 填 deployment 名。
存盘时若 `kind=azure` 且 `api_version` 为空,后端直接拒绝(在设置页报错,而不是
第一次 `/generate` 才炸)。

---

## 3. AWS Bedrock / Google Vertex(经兼容网关,阶段3)

Bedrock 与 Vertex **不是** OpenAI 协议(Bedrock 用 AWS SigV4 签名;Vertex 用
Google OAuth + 自有 schema)。原生接入需各自的 SDK 与凭据链,不在网关内实现。

**推荐方案:前置一个 OpenAI 兼容网关**做协议翻译,网关本产品仍配 `kind: openai`,
**零后端改动**。成熟选择:

- **LiteLLM Proxy** —— 单进程,支持 100+ 后端,含 Bedrock / Vertex;暴露
  `/v1/chat/completions`。
- **One API / New API** —— 中文社区常用,多渠道聚合 + 计费。

### 3a. LiteLLM 翻译 Bedrock(Claude)

`litellm_config.yaml`:

```yaml
model_list:
  - model_name: claude-on-bedrock
    litellm_params:
      model: bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0
      aws_region_name: us-east-1
      # 凭据走标准 AWS 链:环境变量 / IAM role / ~/.aws/credentials
general_settings:
  master_key: sk-rst-gateway-local   # 本产品拿这个当 api_key
```

启动:`litellm --config litellm_config.yaml --port 4000`

本产品 `llm_providers.yml`:

```yaml
providers:
  - id: bedrock-via-litellm
    kind: openai
    base_url: http://litellm:4000/v1
    api_key: sk-rst-gateway-local
    model: claude-on-bedrock          # = LiteLLM 的 model_name
    tags: [primary]
```

### 3b. LiteLLM 翻译 Vertex(Gemini)

```yaml
model_list:
  - model_name: gemini-on-vertex
    litellm_params:
      model: vertex_ai/gemini-1.5-pro
      vertex_project: my-gcp-project
      vertex_location: us-central1
      # 凭据:GOOGLE_APPLICATION_CREDENTIALS 指向 service-account json
```

本产品侧同 3a,`model: gemini-on-vertex`,`base_url` 指向同一网关。

### 为什么不原生接?

- **凭据模型不同**:SigV4 / GCP OAuth 需在网关进程内维护刷新逻辑与 SDK 依赖
  (`boto3` / `google-auth`),显著增重且与"OpenAI 单一协议"设计冲突。
- **故障转移语义**:LiteLLM 已做了重试/限流/计费,和本产品的 failover 叠加即可。
- **可换性**:客户日后换云厂商,只动网关配置,本产品配置不变。

若客户明确要求**进程内原生 Bedrock/Vertex**(不接受旁路网关),那是一个独立的
后端工作项(新增 `kind: bedrock` / `kind: vertex` 分支 + SDK 凭据链),需另行评估。

---

## 4. 单供应商 env 回退(v0.x 兼容)

没有 `llm_providers.yml` 时,网关回退到单供应商 env 变量:

```bash
LLM_API_KEY=<key>
LLM_BASE_URL=<base-url>   # 不设则回退火山方舟并告警 llm_base_url_defaulted
LLM_MODEL=<model>
```

此路径只支持 `kind: openai`。要用 Azure / 多供应商 / 故障转移,必须用
`llm_providers.yml`。

---

## 5. Agentic 调查(可选,tool-use 检索循环)

`/api/investigate-alert` **默认走 agentic 模式**:LLM 通过 `es_search` 工具**自主多轮
检索** ES 收集证据再下结论,对复杂事件调查更充分(比单步更慢、更耗 token)。显式关闭
(`RST_AGENTIC_INVESTIGATE=0`)回退单步模式(一次固定上下文查询 + 一次 LLM 调用)。

**开关 + 调优(env):**

| 变量 | 默认 | 说明 |
|------|------|------|
| `RST_AGENTIC_INVESTIGATE` | **开** | 设 `0`/`false` 关闭,回退单步旧路径 |
| `RST_AGENTIC_MAX_STEPS` | `6` | 最大检索轮数(1–20),达上限强制出结论 |
| `RST_AGENTIC_SIZE_CAP` | `20` | 单次 es_search 返回条数上限(1–200) |
| `RST_AGENTIC_TOKEN_BUDGET` | `0`(关) | 累计 token 软上限,超出即停止检索强制出结论 |
| `RST_AGENTIC_DEADLINE_S` | `180` | 整体墙钟时限(秒),超时优雅降级不挂起请求;设 `0` 关闭 |

**前置条件:模型须支持 function-calling / tools。** 若供应商不支持(首轮工具调用
报错),网关**自动回退单步模式**,不会 500。多供应商 failover 场景下,建议把支持
tools 的供应商排在前面。

**安全护栏(对模型每次工具调用都生效,与 `/api/execute` 同源):** 索引白名单、DSL
校验(禁 script/update/delete)、单次条数上限、步数上限、字段脱敏。**聚合结果也会脱敏
后返回**——按 request 里 agg 的 `field` 反查来源字段套用同一分级脱敏策略(桶 key 脱敏、
计数/指标保留、来源不明的 key 防御性遮蔽),模型可用 `size:0` + aggs 做计数/分组统计。

**建议:** 生产开启时**务必**同时设 `RST_INDEX_WHITELIST`(护栏依赖它)、
`RST_AGENTIC_DEADLINE_S`(如 120)与 `RST_AGENTIC_TOKEN_BUDGET`(按成本预算)。

---

## 6. 校验

改配置后:

- UI:设置页点 **Reload**(调 `/api/llm/reload`)。
- API:`GET /api/llm/providers` 看每个 provider 的 `kind` / `last_ok_at` /
  `consec_failures` / `last_error`,确认健康。
- 冒烟:任意 `/api/generate` 调用走通即接入成功。
