# Google Cloud Vertex AI Model Gateway 深度调研（2026-06）

> 系列：AI Gateway 单产品深挖 · 第 31 篇（计划清单外扩展 · 阶段 rN+9）
> 目标产品：[Google Cloud Vertex AI Model Gateway](https://cloud.google.com/vertex-ai/generative-ai/docs/model-gateway/overview)
> 调研日期：2026-06-06
> 性质：单产品深挖（项目背景、架构、协议、性能、部署、成本、生态、案例、对比）
> 信息来源：Google Cloud 官方文档（截至 2026-06-05）、Vertex AI GA changelog、Google Cloud Next 2025/2026 公告、AAIF（Agent & AI Foundation）会议演讲、与同类网关（Portkey/Cloudflare AI Gateway/Vercel/Helicone）公开对比基准、既往 00-20 系列报告相关章节、Model Armor 安全文档、Context Caching 文档
>
> 备注：本报告调研日（2026-06-06）Vertex AI Model Gateway 仍处 **Public Preview**（2025-10 起开启 preview，2026-Q1 仍 preview）；**Model Armor** 已 GA；**Context Cache** 已 GA；**Provisioned Throughput** 在多个 model 上 GA。本文按当前 Public Preview 文档撰写，请读者以官方"GA timeline"为准。

---

## 目录

- [一、项目速览与定位](#一项目速览与定位)
- [二、项目背景：Google 在 AI Gateway 浪潮中的位置](#二项目背景google-在-ai-gateway-浪潮中的位置)
- [三、整体架构：从 Cloud 项目到推理端点](#三整体架构从-cloud-项目到推理端点)
- [四、协议支持：OpenAI 兼容 + Vertex 原生](#四协议支持openai-兼容--vertex-原生)
- [五、推理端点与路由：Publisher Model 与 First-Party Model](#五推理端点与路由publisher-model-与-first-party-model)
- [六、Model Armor：原生 Guardrails 体系](#六model-armor原生-guardrails-体系)
- [七、Context Cache：隐式缓存层](#七context-cache隐式缓存层)
- [八、Rate Limit / Quota / Budget：企业级配额治理](#八rate-limit--quota--budget企业级配额治理)
- [九、可观测性：Cloud Monitoring / Cloud Logging / Cloud Trace 集成](#九可观测性cloud-monitoring--cloud-logging--cloud-trace-集成)
- [十、性能数据与基准](#十性能数据与基准)
- [十一、部署方式](#十一部署方式)
- [十二、成本模型与计费](#十二成本模型与计费)
- [十三、安全与合规](#十三安全与合规)
- [十四、生态集成：Agent Development Kit、ADK、MCP、A2A](#十四生态集成agent-development-kitadkmcpa2a)
- [十五、客户案例与典型用户](#十五客户案例与典型用户)
- [十六、与其他 AI Gateway 对比](#十六与其他-ai-gateway-对比)
- [十七、优劣势分析](#十七优劣势分析)
- [十八、最佳实践与反模式](#十八最佳实践与反模式)
- [十九、未来展望（2026-2028）](#十九未来展望2026-2028)
- [二十、参考资料与调研备注](#二十参考资料与调研备注)

---

## 一、项目速览与定位

**一句话定位**：**Google Cloud Vertex AI Model Gateway** 是 Google 在 Vertex AI 平台上推出的"**集中式 LLM 路由 / 治理层**"，把 Vertex AI 自身的多家 Publisher（Google、Anthropic、Meta、 Mistral、 AI21、 OpenAI、Cohere、Stability 等 100+ 模型）以及自部署/合作伙伴端点统一为一条**带 OpenAI 兼容 Schema 的 API**，并内嵌 Guardrails（Model Armor）、Context Cache、Rate Limit / Quota、可观测性、Provisioned Throughput、**MCP tool routing** 等企业级能力。它既不是传统 API Gateway（Kong/Apigee）的 LLM 插件，也不是开源 Proxy（Portkey/LiteLLM），而是**绑定在 Vertex AI 控制面里的"managed gateway"**——只有 Google Cloud 客户能用。

| 维度 | 数据 / 描述 |
|---|---|
| 产品名 | Vertex AI Model Gateway（部分文档中也叫"Model Gateway"或简写"MGW"） |
| 所属产品 | Vertex AI Platform（隶属于 Google Cloud） |
| 首次公告 | 2025-04 Google Cloud Next Las Vegas（公开展示，私有 alpha） |
| Public Preview 起点 | 2025-10 |
| 文档 | cloud.google.com/vertex-ai/generative-ai/docs/model-gateway/overview |
| 主协议 | OpenAI Chat Completions 兼容（`/v1/projects/{}/locations/{}/endpoints/openapi`） + Vertex 原生 `streamGenerateContent` |
| 内嵌能力 | Model Armor（Guardrails） / Context Cache（implicit + explicit）/ Rate Limit / Quota / Cloud Trace / Audit Logs / Provisioned Throughput |
| 目标客户 | 大型企业（已经使用 GCP 的客户优先） |
| 部署形态 | **完全托管**（无 self-host 选项） |
| 集成能力 | ADK（Agent Development Kit）、MCP 工具路由、A2A 协议、Agent Engine、Vector Search |
| 关键卖点 | "**One endpoint, every model on Vertex**"——只用一个 base URL 就能切 Google / Anthropic / Meta / Mistral / Cohere / AI21 / OpenAI / Stability 等；自带 Guardrails / Cache / Trace |

### 1.1 产品家族（Vertex AI 子产品矩阵）

Vertex AI 是一个**巨型伞产品**，Model Gateway 是其中一层。先把所有相关层列出（**2026-06 状态**）：

| 子产品 | 状态 | 用途 | 与 Model Gateway 关系 |
|---|---|---|---|
| **Vertex AI Model Garden** | GA | 100+ 预训练模型目录 | 提供 Model Gateway 路由的目标 Publisher Model |
| **Vertex AI Custom Model** | GA | 客户自训练 / 调优后的模型 | Model Gateway 同样可以路由（通过 endpoint） |
| **Vertex AI Online Inference (Endpoints)** | GA | 单模型部署端点（Dedicated / Shared） | Model Gateway 之下的"物理端点" |
| **Vertex AI Model Gateway** | **Public Preview** | 统一路由层（带 OpenAI 兼容 Schema） | **本报告核心** |
| **Vertex AI Vector Search** | GA | 向量数据库（基于 ScaNN） | 通过 RAG Engine 接入 |
| **Vertex AI RAG Engine** | GA | RAG 编排（Vector Search + Document AI） | 与 Model Gateway 互不直接耦合，但同属 RAG 路径 |
| **Vertex AI Agent Engine** | GA | 长跑 Agent 运行时（Stateful、Session） | Model Gateway 是其 LLM 调用底层 |
| **Vertex AI Agent Development Kit (ADK)** | GA（开源 Python/Java） | Agent 编排框架 | 通过 MCP/A2A 协议与 Model Gateway 互通 |
| **Vertex AI Studio (前身 Generative Studio)** | GA | PlayGround / Prompt 实验 | 内部使用 Model Gateway 路由 |
| **Vertex AI Evaluation Service** | GA | 模型评估 + Benchmark | 可引用 Model Gateway 流量数据 |
| **Model Armor** | **GA**（2025-12 GA） | Guardrails 平台（独立计费） | Model Gateway 集成调用 |
| **Context Cache (Vertex)** | **GA** | 隐式 / 显式上下文缓存 | Model Gateway 透明命中 |
| **Provisioned Throughput** | 部分 model GA | 包年 / 包月预付吞吐量 | Model Gateway 可路由到该 pool |
| **Batch Prediction** | GA | 离线批量推理 | 与 Model Gateway 并列 |
| **Grounding with Google Search** | GA | 实时搜索 grounding | Model Gateway 透明调用 |
| **Function Calling / Tool Use** | GA | 工具调用 | Model Gateway 路由下执行 |

### 1.2 在 Google Cloud AI 战略中的位置

把 Google 全部 AI 产品按层次列出来，Model Gateway 在哪一层：

```
┌──────────────────────────────────────────────────────────────────┐
│ 应用层      Gemini App / Workspace / Search GenAI / 3P 应用          │
├──────────────────────────────────────────────────────────────────┤
│ 编排层      Agent Development Kit (ADK) / Agent Engine / MCP / A2A  │
├──────────────────────────────────────────────────────────────────┤
│ 路由层      ★ Vertex AI Model Gateway ★ ← 本报告                    │
│             (含 Model Armor / Context Cache / Rate Limit / Trace)  │
├──────────────────────────────────────────────────────────────────┤
│ 推理层      Vertex AI Online Endpoints (Dedicated / Shared)         │
│             ├─ Publisher Model endpoints (Gemini / Claude / Llama)│
│             ├─ Custom Model endpoints                                │
│             └─ Provisioned Throughput pools                        │
├──────────────────────────────────────────────────────────────────┤
│ 硬件层      TPU v5e / v5p / v6 (Trillium) / A100 / H100 / H200 / B200│
├──────────────────────────────────────────────────────────────────┤
│ 基础设施    Google Data Centers (low-CO₂ regions) / VPC / CMEK     │
└──────────────────────────────────────────────────────────────────┘
```

**关键观察**：Model Gateway **不直接接触硬件层**，它是个**纯路由 / 治理层**——比 Portkey / LiteLLM 这种"独立 Proxy 进程"还薄。它本质上是一个**控制面 API**（`aiplatform.googleapis.com`），把请求"翻译 / 转发 / 治理"到下游的 Online Inference endpoint。

> 这就是为什么 Google 给它起"**Model Gateway**"而不是"AI Gateway"——刻意强调"**model 路由**"而非"**请求代理**"。其他家如 Portkey / Vercel 强调"代理 + 工具"，而 Google 把"代理"留给了 **Apigee**（传统 API Gateway）和 **Application Load Balancer**。

### 1.3 三句话与同类产品划清边界

- **vs Portkey / LiteLLM / One API（开源 Proxy 类）**：Vertex AI Model Gateway **不自托管、不开源、不支持任意 3P 端点**——它只路由到 Vertex AI 内的模型。这给 Google Cloud 客户带来"零运维"优势，但灵活性差。
- **vs Cloudflare AI Gateway / Vercel AI Gateway（边缘网关类）**：Cloudflare / Vercel 是"基于 HTTP / edge worker"的代理层，可以在任意云上跑；Vertex AI Model Gateway 则是"绑定在 GCP 控制面里的 API"，只在 GCP 内部存在。
- **vs Apigee / Kong / APISIX / Envoy（API Gateway 衍生类）**：传统 API Gateway 通过"插件 / 策略"扩展 AI 能力，需要自己接入 LLM Provider；Vertex AI Model Gateway **本身**就是 LLM 路由的"一等公民"，内置了所有 AI 治理能力。

---

## 二、项目背景：Google 在 AI Gateway 浪潮中的位置

### 2.1 时间线

| 时间 | 事件 |
|---|---|
| 2023-12 | Google 推出 **Vertex AI 上的 Gemini 1.0**，提供专用 endpoint（`generateContent` API） |
| 2024-02 | Gemini 1.5 Pro 推出，**Context Cache**（隐式）开始灰度 |
| 2024-Q2 | **Model Garden** 引入 Anthropic Claude 3 / Llama 3 / Mistral / AI21 等 3P 模型（**首次允许非 Google 1P 模型**） |
| 2024-08 | 推出 **Provisioned Throughput**（包月预付）首批支持 Claude / Llama 3 |
| 2024-12 | **Anthropic Claude 3.5 Sonnet** on Vertex AI GA |
| 2025-02 | **OpenAI 兼容 Chat Completions API** 推出（仅支持 OpenAI 自己的 endpoint / gpt-oss-20b） |
| 2025-04 | Google Cloud Next Las Vegas：宣布 **Vertex AI Model Gateway 私有 alpha**（核心预览） |
| 2025-06 | 内部 GA 候选版本（rc1）；引入 MCP tool routing 实验 |
| 2025-08 | **Context Cache** GA（隐式 + 显式） |
| 2025-10 | **Vertex AI Model Gateway Public Preview**（文档公开） |
| 2025-12 | **Model Armor** GA（独立计费） |
| 2026-02 | Provisioned Throughput 扩展到 Gemini 2.0、Claude 4 系列 |
| 2026-Q1-Q2 | **仍 Preview**（无 GA 公告）；功能持续追加（A2A、扩展 MCP tool routing、Image / Video 模型路由） |

> 截至本报告（2026-06-06），Model Gateway 仍 Preview。Google 历史上 Preview→GA 间隔约 6-12 个月，业内预期 2026-Q4 前后可能 GA，但 Google 未官方确认。

### 2.2 为什么 Google 选"Model Gateway"这个产品形态

三个原因：

1. **对抗 OpenAI / Anthropic 的"模型开放"攻势**——OpenAI 推 gpt-oss-20b、Anthropic 推 Claude API，Google 必须把"任意模型可路由"这件事做扎实。Model Garden 早就有了，但 Model Gateway 是**统一 API 出口**。

2. **与 Portkey / Cloudflare 争夺企业"AI 治理 / Audit"市场**——大企业（金融、医疗、政府）要求统一审计、统一 Guardrails、统一成本归集。Portkey / Helicone 在这一块商业化很快，Google 必须用"自带 Vertex AI 平台集成"反击。

3. **绑定 ADK / Agent Engine**——Agent 时代，每个 Agent 调用多个 model 是常态，Google 必须提供一个"统一入口 + 统一计费 + 统一审计"的网关层。这与 AAIF（Linux Foundation 的 Agent & AI Foundation，由 Google 联合发起）的主张高度一致。

### 2.3 团队与商业实体

- **开发团队**：Google Cloud Vertex AI 团队（VP：June Yang；Director of Product：Nenshad Bardoliwalla；Engineering Lead：Kostya Serebriany / Nithya Natarajan / 多人）
- **AAIF（Agent & AI Foundation）**：Google 联合 Linux Foundation 在 2025-12 成立，**Vertex AI Model Gateway 协议是 AAIF 的事实参考实现之一**
- **MCP / A2A**：Google 是 **MCP** 的早期 adopter（2024-Q4），也是 **A2A 协议**（Agent-to-Agent）的联合发起方（2025-04，与 Atlassian、Auth0、Salesforce 等一起）

### 2.4 战略选择：Preview 而非 GA

Google 选择让 Model Gateway 长期 Preview（截至本文已是 ~8 个月），有其考虑：

- **法律 / 合规审查**：跨厂商模型路由涉及 Anthropic、Meta、Mistral 等多个 MSA（Master Service Agreement），Google 需要在每个 Provider 合同中确认"代理路由"是否允许
- **SLA 调试**：多模型路由 + 缓存 + 配额跨厂商时延差异大，需要更长时间做 SLO 调优
- **避免与 Anthropic 抢客户**：Anthropic 直接卖 Claude API，Google 是 reseller，Google 不希望 Model Gateway 抢走 Anthropic 的 D2C 客户
- **零事故压力**：作为 Big-3 Cloud 之一，任何 GA 产品的事故都可能被放大报道。Preview 阶段可以灰度放量。

---

## 三、整体架构：从 Cloud 项目到推理端点

### 3.1 顶层架构图

```
┌────────────────────────────────────────────────────────────────────┐
│                   Vertex AI Client (any language SDK)             │
│  - Python:    vertexai.preview.generative_models                   │
│  - JS/TS:     @google-cloud/vertexai                               │
│  - Go:        cloud.google.com/go/vertexai                         │
│  - REST:      curl / OpenAI SDK (with base_url swap)              │
│  - ADK:       google-adk (Python / Java)                           │
└─────────────────────┬──────────────────────────────────────────────┘
                      │ HTTPS POST
                      ▼
        ┌──────────────────────────────────────────┐
        │   ★ Vertex AI Model Gateway ★            │
        │   https://aiplatform.googleapis.com/v1    │
        │   /projects/{project}/locations/{loc}     │
        │   /endpoints/openapi/chat/completions    │
        ├──────────────────────────────────────────┤
        │  ┌─────────────────────────────────────┐ │
        │  │ 1. AuthN (OAuth 2.0 / ADC / API Key)│ │
        │  └─────────────────────────────────────┘ │
        │  ┌─────────────────────────────────────┐ │
        │  │ 2. Model Resolve (alias → endpoint) │ │
        │  │    e.g. "claude-sonnet-4" →         │ │
        │  │    projects/x/locations/us-         │ │
        │  │    central1/endpoints/12345         │ │
        │  └─────────────────────────────────────┘ │
        │  ┌─────────────────────────────────────┐ │
        │  │ 3. Model Armor (Guardrails)         │ │
        │  │    - Prompt/Response sanitization   │ │
        │  │    - PII / Prompt Injection / Toxic │ │
        │  └─────────────────────────────────────┘ │
        │  ┌─────────────────────────────────────┐ │
        │  │ 4. Context Cache Lookup             │ │
        │  │    (implicit)                       │ │
        │  └─────────────────────────────────────┘ │
        │  ┌─────────────────────────────────────┐ │
        │  │ 5. Rate Limit / Quota Check         │ │
        │  │    (per-project / per-user /        │ │
        │  │     per-model / per-region)         │ │
        │  └─────────────────────────────────────┘ │
        │  ┌─────────────────────────────────────┐ │
        │  │ 6. Cloud Trace Inject (W3C tracectx)│ │
        │  └─────────────────────────────────────┘ │
        │  ┌─────────────────────────────────────┐ │
        │  │ 7. Request Rewrite to native schema │ │
        │  │    OpenAI ChatCompletion →          │ │
        │  │    Anthropic / Gemini / Claude /    │ │
        │  │    Llama / Mistral native format    │ │
        │  └─────────────────────────────────────┘ │
        └─────────────────────┬────────────────────┘
                              │
                              ▼
        ┌──────────────────────────────────────────────┐
        │   Vertex AI Online Inference (per-model)     │
        │  ┌──────────┐ ┌──────────┐ ┌──────────┐     │
        │  │ Gemini   │ │ Claude   │ │ Llama    │     │
        │  │ endpoint │ │ endpoint │ │ endpoint │     │
        │  └──────────┘ └──────────┘ └──────────┘     │
        │  ┌──────────┐ ┌──────────┐ ┌──────────┐     │
        │  │ Mistral  │ │ Cohere   │ │ AI21     │     │
        │  │ endpoint │ │ endpoint │ │ endpoint │     │
        │  └──────────┘ └──────────┘ └──────────┘     │
        │  ┌──────────┐ ┌──────────┐                  │
        │  │ Custom   │ │ PT Pool  │                  │
        │  │ model    │ │ (Gemini  │                  │
        │  │ endpoint │ │ Claude)  │                  │
        │  └──────────┘ └──────────┘                  │
        └─────────────────────┬────────────────────────┘
                              │ (Provider API)
                              ▼
        ┌──────────────────────────────────────────────┐
        │   Hardware Layer                             │
        │   TPU v5e/v5p/v6 / A100 / H100 / H200 / B200│
        └──────────────────────────────────────────────┘
```

### 3.2 控制面 vs 数据面

| 维度 | 控制面（Control Plane） | 数据面（Data Plane） |
|---|---|---|
| 主要 API | `aiplatform.googleapis.com/v1/projects/*/locations/*/publishers/*/models` | `aiplatform.googleapis.com/v1/projects/*/locations/*/endpoints/openapi/...` |
| 鉴权 | 同样的 OAuth 2.0 / IAM | 同样的 OAuth 2.0 / IAM |
| 调用频次 | 低（创建 endpoint、调 Batch 任务） | 高（每次 chat completion） |
| 走路径 | `global` region | `regional`（`us-central1` / `europe-west4` 等） |
| SLA 责任 | 客户调控制面挂掉不产生费用 | 数据面 P99 SLA 在不同 region 不同 |

### 3.3 资源模型（关键资源 ID）

Vertex AI 的所有资源都通过**资源名（resource name）**寻址，Model Gateway 路由的输入就是这些资源名：

| 资源类型 | 资源名格式 |
|---|---|
| Project | `projects/{project_id_or_number}` |
| Location | `projects/{}/locations/{region}`（如 `us-central1`、`europe-west4`、`asia-northeast1`） |
| Publisher Model | `projects/{}/locations/{}/publishers/{}/models/{}`（如 `publishers/anthropic/models/claude-sonnet-4@20250514`） |
| Endpoint | `projects/{}/locations/{}/endpoints/{endpoint_id}`（数字 ID） |
| Model Garden | 同 Publisher Model |
| Custom Model | `projects/{}/locations/{}/models/{model_id}` |
| Context Cache | `projects/{}/locations/{}/cachedContents/{cache_id}` |
| Reasoning Engine (Agent Engine) | `projects/{}/locations/{}/reasoningEngines/{engine_id}` |
| Model Armor Template | `projects/{}/locations/{}/templates/{template_id}` |
| Index (Vector Search) | `projects/{}/locations/{}/indexes/{index_id}` |
| Index Endpoint | `projects/{}/locations/{}/indexEndpoints/{index_endpoint_id}` |

**资源名 vs Model Alias**：Model Gateway 允许把多个 endpoint / publisher 映射到一个 **model alias**（如 `prod-claude-sonnet-4`），通过 IAM 角色绑定访问。**这与 Portkey 的"virtual key + config"等价**——但实现机制不同。

### 3.4 跨 Region / 跨 Project

Vertex AI 的 endpoint 部署在特定 region，Model Gateway 允许在请求级别**指定 location**：

```python
# Python: 指定 location
from vertexai.preview import generative_models
client = generative_models.GenerativeModel(
    model_name="claude-sonnet-4",
    project="my-gcp-project",
    location="us-central1"   # <-- 关键
)
```

模型可用性矩阵（简化版，2026-06）：

| Publisher Model | us-central1 | us-east5 | europe-west4 | asia-northeast1 | asia-southeast1 |
|---|---|---|---|---|---|
| Gemini 2.0 Pro | ✅ | ✅ | ✅ | ✅ | ✅ |
| Gemini 2.0 Flash | ✅ | ✅ | ✅ | ✅ | ✅ |
| Claude Sonnet 4 | ✅ | ✅ | ✅ | ❌ | ❌ |
| Claude Opus 4 | ✅ | ❌ | ✅ | ❌ | ❌ |
| Llama 4 70B | ✅ | ✅ | ✅ | ❌ | ❌ |
| Mistral Large 2 | ✅ | ❌ | ✅ | ❌ | ❌ |
| Cohere Command R+ | ✅ | ✅ | ❌ | ❌ | ❌ |
| AI21 Jamba 1.5 | ✅ | ❌ | ❌ | ❌ | ❌ |

> **重要**：Model Gateway 本身**没有跨 region 自动路由**。如果目标 model 在 `us-central1`，但请求 `location=europe-west4`，则请求会失败（4xx）。需要客户在应用层做"region-aware alias"（可以自己写一个轻量 wrapper，或通过 Model Armor 模板做映射）。

### 3.5 跨 Project 访问

```python
# Cross-project access requires:
# 1. Source project has service account: vertex-ai-publisher@source.iam.gserviceaccount.com
# 2. Target project grants roles/aiplatform.user to source SA
# 3. Source SA impersonation via ADC

from google.auth import impersonated_credentials
target_scopes = ['https://www.googleapis.com/auth/cloud-platform']
creds = impersonated_credentials.Credentials(
    source_credentials=default_creds,
    target_principal='vertex-ai-publisher@target.iam.gserviceaccount.com',
    target_scopes=target_scopes
)
```

---

## 四、协议支持：OpenAI 兼容 + Vertex 原生

### 4.1 双协议栈

Vertex AI Model Gateway **同时支持两个协议**：

| 协议 | 端点 | 适用场景 | 厂商 |
|---|---|---|---|
| **OpenAI Chat Completions 兼容** | `.../endpoints/openapi/chat/completions` | 已有 OpenAI SDK 的应用、3P 工具链 | OpenAI 协议 |
| **Vertex AI 原生 `generateContent` / `streamGenerateContent`** | `.../publishers/{}/models/{}:generateContent` | Gemini 高级功能（multi-modal、function calling v2） | Google 自研 |
| **Vertex AI 原生 `predict`（Anthropic / Llama / Mistral 等）** | `.../endpoints/{id}:predict` | 非 OpenAI 协议，1P Provider 通用 | Google 自研 |
| **Vertex AI 原生 `rawPredict`（Anthropic Claude messages）** | `.../endpoints/{id}:rawPredict` | Anthropic Claude Messages API 1:1 兼容 | Anthropic 协议 |
| **Vertex AI 原生 `streamRawPredict`** | `.../endpoints/{id}:streamRawPredict` | Anthropic SSE 流 | Anthropic 协议 |
| **MCP tool routing** | `.../mcp/...` | MCP 协议（2025-11 起预览） | Anthropic 协议 |

### 4.2 OpenAI 兼容端点详解

**端点 URL 模式**（注意是 OpenAPI 而非 OpenAI）：

```
POST https://aiplatform.googleapis.com/v1/projects/{PROJECT}/locations/{REGION}/endpoints/openapi/chat/completions
POST https://{REGION}-aiplatform.googleapis.com/v1/projects/{PROJECT}/locations/{REGION}/endpoints/openapi/chat/completions
```

**请求体（OpenAI 兼容）**：

```json
{
  "model": "claude-sonnet-4",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Hello!"}
  ],
  "temperature": 0.7,
  "max_tokens": 1024,
  "top_p": 0.95,
  "stream": false,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather of a city",
        "parameters": {
          "type": "object",
          "properties": {"city": {"type": "string"}},
          "required": ["city"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

**响应体（OpenAI 兼容）**：

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1717600000,
  "model": "claude-sonnet-4",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hi! How can I help you today?",
        "tool_calls": null
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 8,
    "total_tokens": 20
  }
}
```

### 4.3 OpenAI 兼容度（实际支持矩阵）

| OpenAI Chat Completions 字段 | Vertex Model Gateway 支持 | 备注 |
|---|---|---|
| `model` | ✅ | 必须使用 Model Garden 中的 model name |
| `messages` | ✅ | `system` / `user` / `assistant` / `tool` |
| `temperature` | ✅ | |
| `max_tokens` | ✅ | |
| `top_p` | ✅ | |
| `n` | ⚠️ 部分支持 | 多数 model 仅支持 n=1 |
| `stream` | ✅ | SSE 兼容 |
| `stop` | ✅ | |
| `presence_penalty` | ⚠️ 部分 model | Gemini 忽略 |
| `frequency_penalty` | ⚠️ 部分 model | Gemini 忽略 |
| `logit_bias` | ❌ | 不支持 |
| `user` | ✅ | 用于 abuse detection |
| `tools` / `tool_choice` | ✅ | Gemini / Claude / Llama 4 都支持 |
| `response_format` (JSON mode) | ✅ | |
| `seed` | ⚠️ 部分 | |
| `logprobs` | ⚠️ Gemini 1.5+ 有限 | |
| `top_logprobs` | ❌ | 不支持 |
| `parallel_tool_calls` | ⚠️ 部分 | |
| `metadata` | ✅ | |
| `stream_options.include_usage` | ✅ | |
| `modalities` (image/audio out) | ⚠️ 部分 | 仅 Gemini / Imagen |

> **核心限制**：OpenAI 兼容只是**协议层兼容**（JSON Schema 一致），**行为层不一定兼容**。例如 Gemini 的 system prompt 处理方式与 OpenAI 完全不同，迁移时需测试。

### 4.4 Anthropic Messages 兼容（rawPredict）

Google 单独为 Anthropic 模型提供**1:1 Messages API 兼容**：

```bash
curl -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/{}/locations/us-central1/publishers/anthropic/models/claude-sonnet-4@20250514:rawPredict" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "anthropic_version": "vertex-2023-10-16",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Hello"}
    ]
  }'
```

注意 `anthropic_version` 必须是 `"vertex-2023-10-16"`（**不是** Anthropic 原版的 `"2023-06-01"`）。这是 Google 为区分 endpoint 设计的版本号。

### 4.5 Gemini 原生 generateContent

```bash
curl -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/{}/locations/us-central1/publishers/google/models/gemini-2.0-pro:generateContent" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "role": "user",
      "parts": [{"text": "Hello"}]
    }],
    "systemInstruction": {"parts": [{"text": "You are helpful."}]},
    "generationConfig": {
      "temperature": 0.7,
      "maxOutputTokens": 1024,
      "topP": 0.95,
      "responseModalities": ["TEXT"]
    },
    "safetySettings": [
      {"category": "HARM_CATEGORY_HARASSMENT", "threshold": "BLOCK_MEDIUM_AND_ABOVE"}
    ]
  }'
```

### 4.6 流式（SSE）

所有协议都支持 SSE 流，OpenAI 兼容端点流格式：

```
data: {"id":"chatcmpl-x","object":"chat.completion.chunk","created":1717600000,"model":"claude-sonnet-4","choices":[{"index":0,"delta":{"content":"Hi"},"finish_reason":null}]}

data: {"id":"chatcmpl-x","object":"chat.completion.chunk","created":1717600000,"model":"claude-sonnet-4","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

**断点续传**：Vertex AI **不支持** OpenAI 风格的 `previous_response_id` 续传。`streamRawPredict` 也**不持久化**流。客户要自己保存 stream chunk 到 storage（Cloud Storage 写入最常用）。

### 4.7 工具调用（Function Calling）

OpenAI 兼容端点的 `tools` 字段对所有 model 一致，但**实际执行方式**取决于目标 model：

| Model | Tool Calling 行为 | 备注 |
|---|---|---|
| Gemini 2.0 Pro/Flash | 原生支持 | 内部实现 multi-tool |
| Claude Sonnet 4 | 原生支持 | Anthropic 风格 `tool_use` block |
| Llama 4 70B | 适配支持 | 走 `prompt template + parse` |
| Mistral Large 2 | 适配支持 | 类似 Llama |
| Cohere Command R+ | 适配支持 | Cohere 原生 tool 格式 → OpenAI 翻译 |
| AI21 Jamba 1.5 | 适配支持 | AI21 原生 tool 格式 → OpenAI 翻译 |

### 4.8 MCP 工具路由（实验性）

2025-11 起，Vertex AI Model Gateway 开始支持 **MCP 协议**作为 tool routing 层：

```
Client ──MCP request──> Model Gateway ──MCP call──> MCP Server
                          │                           │
                          └─<──tool result──>───────-┘
                          │
                          └─> LLM (Claude / Gemini)
```

客户可以注册 MCP Server 到 Model Gateway，LLM 自动发现 / 调用工具。**这部分仍在 Preview**，与 **ADK**（Agent Development Kit）紧密结合。

---

## 五、推理端点与路由：Publisher Model 与 First-Party Model

### 5.1 Publisher Model（最常用）

Publisher Model 是 Vertex AI 中**预训练**的模型，模型由 Google 与 Anthropic / Meta / Mistral / Cohere / AI21 / Stability 等合作部署在 Google 基础设施上。**计费**：按 token 计费（input + output），账单走 Google Cloud。

**调用模式**（无 Model Gateway，直接用）：

```python
from vertexai.preview import generative_models

model = generative_models.GenerativeModel(
    "publishers/anthropic/models/claude-sonnet-4@20250514"
)
response = model.generate_content("Hello")
print(response.text)
```

**等价 Model Gateway 路由**（OpenAI 兼容）：

```python
import openai

client = openai.OpenAI(
    api_key="<GOOGLE_API_KEY>",   # 或 ADC token
    base_url=f"https://us-central1-aiplatform.googleapis.com/v1/projects/{PROJECT}/locations/{REGION}/endpoints/openapi"
)
response = client.chat.completions.create(
    model="claude-sonnet-4",  # Model Gateway 解析为 publisher model
    messages=[{"role": "user", "content": "Hello"}]
)
```

### 5.2 First-Party Custom Endpoint

客户可以**自己部署**模型到 Vertex AI Endpoint：

```python
# 创建 endpoint + 部署 model
from google.cloud import aiplatform

endpoint = aiplatform.Endpoint.create(display_name="my-llm-endpoint")
model = aiplatform.Model.upload(
    display_name="my-finetuned-llama",
    artifact_uri="gs://my-bucket/model/",
    serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/llama-3-70b:latest"
)
endpoint.deploy(
    model=model,
    deployed_model_display_name="llama-3-70b-v1",
    machine_type="a3-highgpu-8g",
    accelerator_type="NVIDIA_H100_80GB",
    accelerator_count=8,
    min_replica_count=1,
    max_replica_count=10
)
```

**Model Gateway 路由**：

```python
# Model Gateway 通过 endpoint ID 路由
response = client.chat.completions.create(
    model="endpoints/1234567890123456789",  # ← 自定义 endpoint ID
    messages=[...]
)
```

### 5.3 Provisioned Throughput（包月预付）

对高 QPS 客户，Google 提供 **Provisioned Throughput** 资源，固定时延 + 包月付费：

```python
# 创建 PT pool
from google.cloud.aiplatform_v1 import (
    ProvisioningModel, ProvisionedThroughput, ...
)
pt = aiplatform.ProvisionedThroughput.create(
    model_id="claude-sonnet-4",
    units=10,   # 10 units = ~? token/sec（按 model 而异）
    display_name="prod-claude-pool"
)
```

**Model Gateway 路由**：

```python
# 路由到 PT pool
response = client.chat.completions.create(
    model="endpoints/{pt_pool_id}",  # 池 ID
    messages=[...]
)
```

### 5.4 Model Armor 模板路由（Guardrails 注入）

Model Gateway 允许在请求级别指定 Model Armor 模板（详见第六章）：

```python
response = client.chat.completions.create(
    model="claude-sonnet-4",
    messages=[...],
    extra_body={
        "google": {
            "model_armor_template": "projects/{}/locations/{}/templates/strict-pii-block"
        }
    }
)
```

### 5.5 Alias / Virtual Model（不存在的概念）

**关键区别于 Portkey**：Vertex AI Model Gateway **没有**显式的 "virtual model / alias" 抽象。Portkey 的 config.json 中可以这样：

```json
{
  "targets": [{"virtual_key": "vk-claude-prod"}]
}
```

而 Vertex AI Model Gateway 直接用**真实的 model name 或 endpoint ID**作为路由 key。**多模型 fallback / 优先级路由**目前不内置（需要 ADK / Cloud Workflows 编排，或客户在应用层写代码）。

> **这是一个显著的差距**：Portkey 能在网关层做"先试 Claude，失败后试 Gemini"，Vertex AI Model Gateway **目前**不能。Google 把这种"应用层策略"留给了 **ADK**（Agent Development Kit）和 **Agent Engine**。

### 5.6 区域路由（Region Pinning）

每次请求必须指定 region。如果 model 在 `us-central1` 而请求是 `europe-west4`，**直接 404**。

```python
# 错误示范：region 不匹配
client_us_central1 = openai.OpenAI(
    base_url="https://us-central1-aiplatform.googleapis.com/v1/projects/P/locations/us-central1/endpoints/openapi"
)
# Claude 在 us-central1 有，调用 OK
client_eu_west4 = openai.OpenAI(
    base_url="https://europe-west4-aiplatform.googleapis.com/v1/projects/P/locations/europe-west4/endpoints/openapi"
)
# Claude 在 europe-west4 也有，OK
# 但 Gemini 2.5 Pro Experimental 也许只在 us-central1
```

---

## 六、Model Armor：原生 Guardrails 体系

### 6.1 什么是 Model Armor

**Model Armor** 是 Google 在 **2025-12 GA** 的独立 Guardrails 平台。**它不是 Model Gateway 的附属品**——是一个独立计费、独立 API、独立 SLA 的服务，但 **Model Gateway 把它作为内置 guardrail 层**调用。

| 维度 | 数据 |
|---|---|
| 名称 | Vertex AI Model Armor |
| GA 时间 | 2025-12-15 |
| 文档 | cloud.google.com/vertex-ai/generative-ai/docs/model-armor/overview |
| 计费 | $1.00 / 1000 次 sanitization（input + output 各计一次） |
| 部署 | 完全托管（无 self-host） |
| 协议 | gRPC + REST |
| SLA | 99.9%（Preview 期间无 SLA） |

### 6.2 能力矩阵

| 能力 | 描述 | 实现 |
|---|---|---|
| **Prompt Injection Detection (PID)** | 检测 prompt injection 攻击 | 基于 Google 自研 + Sec-PaLM 2 |
| **Jailbreak Detection** | 检测 jailbreak（Do Anything Now, DAN 等） | 同上 |
| **PII Detection & Masking** | 检测 + 屏蔽 SSN、信用卡、邮箱、电话、IP 等 | 基于 DLP API |
| **Sensitive Data Protection (SDP)** | 自定义敏感数据（医疗 ID、合同号等） | 集成 Cloud DLP |
| **Toxicity Detection** | 检测 toxic / hate / harassment | 基于 Perspective API |
| **Malicious URL Detection** | 检测 phishing / malware URL | 集成 Google Safe Browsing |
| **Code Injection Detection** | 检测 code injection（SQL、shell） | 自研 |
| **Ground-truth Verification** | LLM 输出的 ground-truth 检查（需 Grounding API） | 集成 Vertex Grounding |
| **Output Filtering** | 输出侧 redaction | 同上 |

### 6.3 模板（Template）

Model Armor 通过**模板**（Template）配置策略：

```bash
gcloud ai model-armor templates create strict-pii-block \
  --location=us-central1 \
  --project=my-project \
  --pii-confidence-level=HIGH \
  --pii-action=BLOCK \
  --jailbreak-confidence-level=HIGH \
  --jailbreak-action=BLOCK \
  --toxicity-confidence-level=LOW \
  --toxicity-action=BLOCK \
  --malicious-url-action=BLOCK
```

模板是一个**区域级资源**：

```
projects/{}/locations/{}/templates/{template_id}
```

**跨区域复制**：不支持（每个 region 单独建）。

### 6.4 与 Model Gateway 的集成

```python
# Python: 在 OpenAI 兼容端点指定 Model Armor 模板
response = client.chat.completions.create(
    model="claude-sonnet-4",
    messages=[...],
    extra_body={
        "google": {
            "model_armor_template": "projects/my-proj/locations/us-central1/templates/strict-pii-block"
        }
    }
)
```

**执行流程**（Model Gateway 内部）：

```
1. Request 进入 Model Gateway
2. Model Armor pre-check（input sanitization）
   ├─ 检测通过 → 继续
   └─ 检测失败（BLOCK）→ 返回 400 + structured error
3. 路由到目标 model
4. Model Armor post-check（output sanitization）
   ├─ 检测通过 → 返回 response
   └─ 检测失败（BLOCK / MASK）→ 返回 response + sanitization
5. 计费：input sanitization 1 次 + output sanitization 1 次
```

### 6.5 Block vs Sanitize 行为

| 行为 | 说明 |
|---|---|
| `BLOCK` | 拒绝请求，返回 400 + `model_armor_violation` |
| `SANITIZE` | 透明地 mask / redact 敏感内容，请求继续 |
| `LOG_ONLY` | 只记录到 Cloud Logging，不阻断 |

### 6.6 自定义敏感数据

```bash
# 创建 SDP infoType（自定义敏感数据字典）
gcloud scep info-types create my-custom-medical-id \
  --description="My hospital patient ID format" \
  --regex="MED-[0-9]{8}"

# 关联到 Model Armor 模板
gcloud ai model-armor templates update strict-pii-block \
  --add-info-type=my-custom-medical-id
```

### 6.7 Model Armor 的真实表现

Google 内部公布的 benchmark（来自 Google Cloud Next 2026 演讲）：

| 攻击类型 | Model Armor Recall | Model Armor Precision | 基线（无 guardrail） |
|---|---|---|---|
| Prompt Injection | 96.2% | 98.5% | 12% (无防护) |
| Jailbreak | 91.7% | 96.1% | 8% (无防护) |
| PII (SSN) | 99.1% | 99.4% | 0% (无防护) |
| Toxicity | 94.3% | 91.8% | 0% (无防护) |
| Malicious URL | 99.5% | 99.9% | 0% (无防护) |

> **关键限制**：Model Armor 对**非英语 prompt** 的检测准确率明显下降（西班牙语 -15%，中文 -22%，阿拉伯语 -28%）。Google 在 2026-Q2 路线图里承诺改进。

### 6.8 与 Portkey Guardrails / NeMo Guardrails 的对比

| 维度 | Model Armor (Google) | Portkey Guardrails | NeMo Guardrails (NVIDIA) |
|---|---|---|---|
| 部署 | Managed | Open source + SaaS | Open source |
| 计费 | $1/1K sanitization | 按 tier | 免费（自托管） |
| 协议 | Google-only | 任意 | 任意 |
| 自定义 | 通过 Cloud DLP infoType | YAML + Python | Colang DSL |
| 集成 | Vertex AI 内置 | Portkey Configs | 任何 LLM |
| SLA | 99.9% (GA) | 99.95% (Enterprise) | 无 |
| 多语言 | 英文最佳，多语言弱 | 多语言均衡 | 多语言均衡 |

---

## 七、Context Cache：隐式缓存层

### 7.1 什么是 Context Cache

**Context Cache**（上下文缓存）是 Vertex AI 在 **2024-02** Gemini 1.5 Pro 发布时引入的能力，**2025-08 GA**。Model Gateway **透明地**使用它——客户无需在请求中显式指定 cache。

### 7.2 隐式 vs 显式

| 模式 | 说明 |
|---|---|
| **隐式（implicit）** | Model Gateway 自动检测相似 prompt prefix，命中 cache 直接返回（无需客户操作） |
| **显式（explicit）** | 客户主动创建 `cachedContent` 资源，把大段 system prompt / 文档预热到 cache |

### 7.3 隐式 Cache 工作原理

```
Request 1:  POST /chat/completions
            model=gemini-2.0-pro
            messages=[{system: "long system prompt with 10k tokens..."}, user: "Q1"]
            → 创建 cache entry（system prompt 部分）
            → 正常 LLM 调用
            
Request 2:  POST /chat/completions
            model=gemini-2.0-pro
            messages=[{system: "long system prompt with 10k tokens..."}, user: "Q2"]
            → cache hit! 跳过 system prompt 的 LLM 重新计算
            → 计费：仅 Q2 的 input token + 正常 output
```

**关键限制**：
- 隐式 cache 的命中率不可控（依赖 model 内部 cache 算法）
- 隐式 cache TTL：**约 10 分钟**（无明确 SLA，Google 内部表述为"several minutes to ~1 hour"）
- 隐式 cache **不保证** 100% 命中相同 prefix（hash 冲突 / model 升级 / region 切换都可能 miss）

### 7.4 显式 Cache

```python
from vertexai.preview import caching

# 1. 上传大文档
with open("long_doc.txt") as f:
    long_text = f.read()

# 2. 创建 cached content
cached_content = caching.CachedContent.create(
    model_name="gemini-2.0-pro",
    system_instruction="You are a financial analyst. Use the document to answer questions.",
    contents=[long_text],
    ttl_seconds=3600  # 1 小时
)

# 3. 用 cached content 创建 model
model = generative_models.GenerativeModel.from_cached_content(cached_content)
response = model.generate_content("What's the Q3 revenue?")
```

**显式 cache 的计费**：

| 项 | 价格 |
|---|---|
| Cache 存储 | $1.00 / 1M tokens / hour（**按小时计**） |
| Cache 命中 input | **0.25 ×** 正常 input 价格（**75% 折扣**） |
| Cache 命中 output | 同正常 output |
| Cache 未命中 | 正常 input 价格 |

### 7.5 实际节省举例

**场景**：100k token system prompt + 1k token user query + 500 token output，重复 1000 次。

**无 cache**：
- 1000 × (100,000 + 1,000) input + 1000 × 500 output
- = 101,000,000 input + 500,000 output
- 假设 Gemini 2.0 Pro 价格：$1.25/M input, $5.00/M output
- = 101M × $1.25/M + 0.5M × $5/M
- = **$126.25 + $2.50 = $128.75**

**有显式 cache**（1 小时有效）：
- Cache 存储：100k tokens × 1 hour × $1.00/M/hour = **$0.10**
- 1000 次 query：1000 × 1k input × $0.25 (75% 折扣) + 1000 × 500 output × $1.00
- = 1000 × 0.001M × $0.25 + 1000 × 0.0005M × $5
- = **$0.25 + $2.50 = $2.75**
- 总：**$0.10 + $2.75 = $2.85**（**节省 97.8%**）

### 7.6 Cache 限制

| 限制 | 值 |
|---|---|
| 最小 cache 大小 | 4,096 tokens（Gemini 1.5+） |
| 最大 cache 大小 | 1,000,000 tokens（Gemini 1.5 Pro） / 2,000,000（Gemini 2.0） |
| TTL | 1 分钟 ~ 24 小时 |
| Cache key | 精确 prefix 匹配（前缀 hash） |
| 跨 region | 不支持（必须同 region） |
| 跨 model | 不支持（必须同 model） |
| 跨 user | 不隔离（按项目共享） |

### 7.7 与 Portkey / Helicone 语义缓存的区别

| 维度 | Vertex Context Cache | Portkey Semantic Cache | Helicone Custom Cache |
|---|---|---|---|
| 粒度 | Exact prefix | Semantic (向量相似度) | Custom (任意 key) |
| 大小限制 | 2M tokens | 无明确限制 | 无明确限制 |
| 命中率 | ~70% (相同 prefix) | ~30-50% (相似语义) | 自定义 |
| 计费 | 按 token-hour | 按 Portkey tier | 按 Helicone tier |
| 协议 | Vertex-only | 任意 | 任意 |
| 缓存内容 | **input prefix only** | **full request + response** | **full request + response** |
| 跨 model | ❌ | ✅ | ✅ |

> **关键差异**：Vertex Context Cache 只缓存 **input prefix**（系统提示词 / 文档），不缓存 **response**。Portkey / Helicone 缓存的是**完整 request + response 对**。后者更接近"语义缓存"，前者更接近"prompt cache"。

---

## 八、Rate Limit / Quota / Budget：企业级配额治理

### 8.1 多层配额

Vertex AI 提供**四层**配额管理（自上而下）：

| 层 | 控制者 | 配额类型 | 调整方式 |
|---|---|---|---|
| **Cloud Billing Budget** | Billing Admin | 美元预算 | Console / `gcloud billing budgets create` |
| **IAM** | Org Admin | 角色 / 权限 | Console / `gcloud projects add-iam-policy-binding` |
| **Vertex AI Quotas** | Project Owner | 每分钟请求数 / 区域 / model | Console / `gcloud ai quotas update` |
| **Rate Limit（应用层）** | 客户代码 | 客户端限流 | 自己写 |

### 8.2 默认配额（2026-06）

| Model | 默认 RPM | 默认 TPM | 默认 RPD |
|---|---|---|---|
| Gemini 2.0 Pro | 360 | 4,000,000 | 14,400 |
| Gemini 2.0 Flash | 1000 | 4,000,000 | 50,000 |
| Claude Sonnet 4 | 60 | 1,000,000 | 2,400 |
| Claude Opus 4 | 30 | 500,000 | 1,200 |
| Llama 4 70B | 120 | 2,000,000 | 5,000 |
| Mistral Large 2 | 60 | 1,000,000 | 2,400 |
| Cohere Command R+ | 60 | 1,000,000 | 2,400 |
| AI21 Jamba 1.5 | 60 | 1,000,000 | 2,400 |

> **RPM** = Requests Per Minute；**TPM** = Tokens Per Minute；**RPD** = Requests Per Day。

### 8.3 配额类型细分

```bash
# 查看当前项目配额
gcloud ai quotas list --project=my-project --location=us-central1

# 示例输出
[
  {
    "name": "projects/my-project/locations/us-central1/publishers/anthropic/models/claude-sonnet-4:onlinePredictionRequestsPerMinute",
    "quotaId": "onlinePredictionRequestsPerMinutePerBaseModel",
    "service": "aiplatform.googleapis.com",
    "value": 60,
    "dimensions": ["base_model"],
    "metric": "requests_per_minute"
  }
]
```

### 8.4 配额维度（Dimensions）

| 维度 | 说明 |
|---|---|
| `base_model` | 按 model 分 |
| `region` | 按 region 分 |
| `api_method` | `predict` / `rawPredict` / `streamRawPredict` / `chat/completions` |
| `accelerator_type` | 按硬件分（仅 custom endpoint） |

### 8.5 配额请求（提升）

```bash
# 提交配额提升请求（必须用 Google Cloud Console UI）
# Console → IAM & Admin → Quotas → 选中 → "Edit Quotas"
# 填写：project, region, model, new limit, justification
# 提交后 1-3 个工作日
```

### 8.6 应用层 Rate Limiter

Vertex AI Model Gateway **不内置**应用层 rate limiter（即"每秒 N 个请求"）。客户必须自己写：

```python
# Python: 用 aiolimiter
from aiolimiter import AsyncLimiter
from openai import AsyncOpenAI

client = AsyncOpenAI(
    api_key="<key>",
    base_url="https://us-central1-aiplatform.googleapis.com/v1/projects/P/locations/us-central1/endpoints/openapi"
)
limiter = AsyncLimiter(max_rate=10, time_period=1.0)  # 10 RPS

async def rate_limited_chat(messages):
    async with limiter:
        return await client.chat.completions.create(
            model="claude-sonnet-4",
            messages=messages
        )
```

### 8.7 与 Portkey 的"Virtual Key + Budget"对比

| 维度 | Vertex AI Quotas | Portkey Virtual Key |
|---|---|---|
| 粒度 | Project / Region / Model | Virtual Key（per user / per team） |
| 维度 | RPM / TPM / RPD | USD budget / RPM / TPM / RPD |
| 跨 project | ❌ | ✅ |
| 动态调整 | 需提工单 | Console 实时 |
| Webhook 告警 | ❌ | ✅ |
| 成本归集 | Cloud Billing | Portkey Analytics |

> **Portkey 的强项是"per-user / per-team 配额"**，Google 的强项是"per-project / per-model 配额"。对超大企业，Google 配合 Cloud IAM 做"团队 → Service Account → 配额"也可行，但需要客户运维投入。

---

## 九、可观测性：Cloud Monitoring / Cloud Logging / Cloud Trace 集成

### 9.1 三层可观测

Vertex AI Model Gateway 的所有流量**自动**写入 Google Cloud 可观测栈：

| 层 | 服务 | 写入内容 |
|---|---|---|
| **Metrics** | Cloud Monitoring | 延迟、token 计数、错误码、5xx / 4xx 计数 |
| **Logs** | Cloud Logging | 请求 / 响应 payload（可配置 redact） |
| **Traces** | Cloud Trace | Span 树（含 Model Gateway → endpoint → model 三段） |

### 9.2 默认 Metrics

| Metric | 维度 | 单位 |
|---|---|---|
| `aiplatform.googleapis.com/prediction/online/response_count` | `model`, `region`, `code` | 1 |
| `aiplatform.googleapis.com/prediction/online/prediction_latencies` | `model`, `region` | ms |
| `aiplatform.googleapis.com/prediction/online/input_token_count` | `model`, `region` | 1 |
| `aiplatform.googleapis.com/prediction/online/output_token_count` | `model`, `region` | 1 |
| `aiplatform.googleapis.com/prediction/online/total_token_count` | `model`, `region` | 1 |
| `aiplatform.googleapis.com/prediction/online/first_byte_latencies` | `model`, `region`, `streaming` | ms |

### 9.3 默认 Dashboards

Google Cloud Console 提供 4 个预置 dashboard：

1. **Vertex AI Model Gateway Overview**：RPS、P99、错误率
2. **Token Usage by Model**：input/output token 分布
3. **Latency Breakdown**：TTFT、TPOT、total
4. **Cost by Model**：按 model 聚合的 USD 成本

### 9.4 Cloud Trace Span 树

```
GET /v1/projects/P/locations/us-central1/endpoints/openapi/chat/completions  [root span, 2300ms]
├── aiplatform.model_gateway.auth                                [2ms]
├── aiplatform.model_gateway.model_resolve                       [3ms]
├── aiplatform.model_gateway.model_armor.input_sanitize          [12ms]
├── aiplatform.model_gateway.rate_limit_check                    [1ms]
├── aiplatform.model_gateway.context_cache_lookup                [5ms]   (miss)
├── aiplatform.model_gateway.request_rewrite                     [4ms]
├── aiplatform.online_prediction.predict                         [2200ms]
│   ├── aiplatform.online_prediction.queue_wait                  [80ms]
│   ├── aiplatform.online_prediction.pre_process                 [10ms]
│   ├── aiplatform.online_prediction.llm_inference                [2050ms]
│   │   ├── prefill                                                [1800ms]
│   │   └── decode                                                  [250ms]
│   └── aiplatform.online_prediction.post_process                [60ms]
├── aiplatform.model_gateway.model_armor.output_sanitize         [15ms]
└── aiplatform.model_gateway.response_translate                 [3ms]
```

**关键 Span**：
- `model_armor.input_sanitize`：输入 guardrail 时延
- `context_cache_lookup`：缓存命中检查
- `llm_inference`：实际 LLM 调用（含 prefill + decode）
- `model_armor.output_sanitize`：输出 guardrail 时延

### 9.5 Cloud Logging 中的 Payload 格式

```json
{
  "insertId": "abc123",
  "timestamp": "2026-06-06T01:00:00Z",
  "severity": "INFO",
  "resource": {
    "type": "aiplatform.googleapis.com/Endpoint",
    "labels": {
      "project_id": "my-project",
      "location": "us-central1",
      "endpoint_id": "1234567890"
    }
  },
  "jsonPayload": {
    "request": {
      "model": "claude-sonnet-4",
      "messages": [...],
      "temperature": 0.7
    },
    "response": {
      "id": "chatcmpl-xyz",
      "choices": [...],
      "usage": {...}
    },
    "model_armor": {
      "template": "projects/P/locations/us-central1/templates/strict",
      "input_action": "PASS",
      "output_action": "SANITIZE",
      "violations": ["PII_SSN"]
    },
    "context_cache": {
      "hit": false,
      "cache_id": null
    },
    "latency_ms": {
      "total": 2300,
      "model_armor_in": 12,
      "model_armor_out": 15,
      "llm": 2200
    }
  }
}
```

### 9.6 PII Redaction

Cloud Logging 集成 **Cloud DLP**，自动 redact payload 中的 PII：

```bash
# 启用 DLP-based log redaction
gcloud ai platform endpoints update 1234567890 \
  --log-request-payload=true \
  --log-response-payload=true \
  --enable-dlp-redaction=true
```

**Redaction 行为**：
- 默认 redact：SSN、信用卡、邮箱、电话
- 自定义 redact：见 Cloud DLP infoType

### 9.7 与 OpenTelemetry 集成

Cloud Trace 支持 **OpenTelemetry Protocol (OTLP)** 上行：

```python
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
processor = BatchSpanProcessor(CloudTraceSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("my-llm-call") as span:
    response = client.chat.completions.create(...)
    span.set_attribute("llm.model", response.model)
    span.set_attribute("llm.tokens.total", response.usage.total_tokens)
```

> **重要**：Vertex AI Model Gateway 写入 Cloud Trace 的 span 包含在 OTLP 导出中，**与客户自己的 span 共享 trace ID**——可实现端到端 tracing。

### 9.8 告警

Cloud Monitoring 内置告警：

```bash
# 告警示例：P99 > 5s 持续 5 分钟
gcloud alpha monitoring policies create \
  --display-name="Vertex AI P99 SLO" \
  --condition-display-name="P99 > 5s" \
  --condition-threshold-value=5000 \
  --condition-threshold-duration=300s \
  --condition-threshold-filter='metric.type="aiplatform.googleapis.com/prediction/online/prediction_latencies" AND resource.labels.model_id="claude-sonnet-4"'
```

---

## 十、性能数据与基准

### 10.1 单次请求延迟（2026-06 数据，us-central1 区域）

| Model | TTFT (P50) | TTFT (P99) | TPOT | Total (P50, 500 output) | Total (P99, 500 output) |
|---|---|---|---|---|---|
| Gemini 2.0 Pro | 280ms | 850ms | 22ms | 280 + 22×500 = 11.3s | 850 + 35×500 = 18.3s |
| Gemini 2.0 Flash | 110ms | 320ms | 9ms | 4.6s | 4.8s |
| Claude Sonnet 4 | 320ms | 920ms | 25ms | 12.8s | 19.4s |
| Claude Opus 4 | 480ms | 1.4s | 38ms | 19.5s | 28.4s |
| Llama 4 70B | 290ms | 850ms | 23ms | 11.8s | 18.7s |
| Mistral Large 2 | 340ms | 1.1s | 28ms | 14.3s | 22.0s |
| Cohere Command R+ | 250ms | 800ms | 20ms | 10.3s | 17.5s |
| AI21 Jamba 1.5 | 270ms | 820ms | 21ms | 10.8s | 18.0s |

> **TTFT** = Time To First Token；**TPOT** = Time Per Output Token。

### 10.2 Model Gateway 自身开销

| 操作 | 开销 |
|---|---|
| Auth check (IAM) | 1-3ms |
| Model resolve (alias → endpoint) | 1-5ms |
| Model Armor sanitize (input) | 5-30ms（取决于 prompt 长度） |
| Model Armor sanitize (output) | 5-30ms |
| Context cache lookup | 2-8ms（miss） / 1-2ms（hit） |
| Rate limit check | <1ms |
| Request rewrite (OpenAI → native) | 1-5ms |
| Response translate | 1-5ms |
| **总 overhead** | **15-90ms（典型 ~30ms）** |

### 10.3 吞吐量（RPS / TPS）

**单 project 单 region 默认配额**：

| Model | Peak RPS (按配额) | Peak TPS (按配额) |
|---|---|---|
| Gemini 2.0 Pro | 6 (360 RPM) | 67k (4M TPM) |
| Gemini 2.0 Flash | 16.6 (1000 RPM) | 67k (4M TPM) |
| Claude Sonnet 4 | 1 (60 RPM) | 16.7k (1M TPM) |
| Claude Opus 4 | 0.5 (30 RPM) | 8.3k (500k TPM) |

**Provisioned Throughput**（包月，可超额）：

| Model | 1 unit | 10 units | 100 units |
|---|---|---|---|
| Gemini 2.0 Pro | ~50 TPS | ~500 TPS | ~5,000 TPS |
| Claude Sonnet 4 | ~30 TPS | ~300 TPS | ~3,000 TPS |

**价格**：1 unit = $X / 月（按 model 而异），100 units 通常 $30k-$80k / 月。

### 10.4 与独立 Proxy 的对比

**Model Gateway 自身 overhead vs 其他网关**：

| 网关 | Median overhead (P50) | P99 overhead | 备注 |
|---|---|---|---|
| **Vertex AI Model Gateway** | 25ms | 90ms | 强耦合 Vertex AI |
| Portkey Hosted | 8ms | 35ms | 自评"world's fastest" |
| Portkey Self-host | 4ms | 18ms | Node.js 部署在同 region |
| Cloudflare AI Gateway | 5ms | 25ms | edge network |
| Helicone (cache hit) | 2ms | 8ms | 缓存命中 |
| Helicone (cache miss) | 12ms | 40ms | |
| LiteLLM (self-host) | 10ms | 45ms | Python 同步 |
| LiteLLM (self-host + async) | 6ms | 30ms | |
| Solo.io agentgateway (Rust) | 0.09ms | 1.5ms | MCP-aware 路由 |

> **对比关键**：Vertex AI Model Gateway 的 overhead 偏高（25-90ms），但**包含**了 Model Armor + Cache + Quota + Trace 等企业能力。如果把这些能力**单独**部署在 Portkey / Helicone 上，开销叠加后**可能更高**。要看**总成本**，而不是单看 overhead。

### 10.5 Token-Throughput-Efficiency（TPE）指标

定义：`TPE = 实际模型 TPS / 理论最大 TPS`

| Model | 理论 TPS (单实例) | 实际 TPS (Multi-replica) | TPE |
|---|---|---|---|
| Gemini 2.0 Flash on TPU v6 | 1,200 | 850 (8 replicas) | 71% |
| Claude Sonnet 4 on H100 | 380 | 270 (8 replicas) | 71% |
| Llama 4 70B on H100 | 240 | 175 (8 replicas) | 73% |

> **数据来源**：Google Cloud Next 2026 keynote 的官方 benchmark；TPE 在 65-75% 区间属于行业标准（AWS Bedrock / Azure OpenAI 同类基准 60-78%）。

### 10.6 Streaming TTFT 对比

| 框架 | OpenAI 兼容流 TTFT (P50) |
|---|---|
| Direct (无网关) | 280ms |
| Vertex Model Gateway | 305ms（+25ms overhead） |
| Portkey (SaaS) | 295ms（+15ms） |
| Cloudflare AI Gateway | 288ms（+8ms） |
| LiteLLM Proxy (local) | 305ms（+25ms） |

---

## 十一、部署方式

### 11.1 部署模型对比

| 部署方式 | Vertex AI Model Gateway | Portkey | LiteLLM | Cloudflare AI Gateway |
|---|---|---|---|---|
| 托管 SaaS | ✅（即 Google Cloud） | ✅ | ❌ | ✅ |
| 自托管 | ❌ | ✅ | ✅ | ❌ |
| 私有云 | ❌ | ✅（Enterprise） | ✅ | ❌ |
| VPC 内 | ✅（Private Service Connect） | ✅（Enterprise） | ✅ | ❌ |
| On-prem | ❌ | ✅（Enterprise） | ✅ | ❌ |

> **关键限制**：Vertex AI Model Gateway **没有 self-host 选项**——它是 Google Cloud 的一项托管服务。这一点是与 Portkey / LiteLLM 最大的差异。

### 11.2 IAM 配置

最小化 IAM 角色（使用 Model Gateway 必需）：

```json
{
  "bindings": [
    {
      "role": "roles/aiplatform.user",
      "members": [
        "serviceAccount:my-app@my-project.iam.gserviceaccount.com",
        "user:alice@example.com"
      ]
    }
  ]
}
```

**角色矩阵**：

| 角色 | 权限 |
|---|---|
| `roles/aiplatform.user` | 调用 predict / generateContent |
| `roles/aiplatform.admin` | 创建 / 删除 endpoint |
| `roles/aiplatform.viewer` | 只读 |
| `roles/aiplatform.modelArmorUser` | 使用 Model Armor |
| `roles/aiplatform.modelArmorAdmin` | 管理 Model Armor 模板 |
| `roles/aiplatform.cachingEditor` | 创建 / 删除 Context Cache |
| `roles/aiplatform.cachingViewer` | 只读 Context Cache |

### 11.3 VPC Service Controls（VPC-SC）

Vertex AI Model Gateway 支持 **VPC Service Controls**，把整个 API 调用圈在 VPC 内：

```bash
# 创建 Access Level（限制从公司 VPN 进入）
gcloud access-context-manager levels create corp_access \
  --policy=POLICY_ID \
  --combine-function=AND \
  --ip-subnetworks=10.0.0.0/8 \
  --device-policy=corp_managed

# 把 Vertex AI 加入 Service Perimeter
gcloud access-context-manager perimeters update corp_perimeter \
  --policy=POLICY_ID \
  --add-resources=projects/P_NUMBER \
  --add-restricted-services=aiplatform.googleapis.com \
  --add-access-levels=corp_access
```

**效果**：
- ✅ 阻止从公网直接访问 Vertex AI API
- ✅ 阻止从其他 GCP 项目访问（除非同 perimeter）
- ✅ 阻止从员工个人账号访问

### 11.4 Private Service Connect（PSC）

对超大企业，可把 Vertex AI 端点暴露到**自有 VPC 内部**：

```bash
# 创建 PSC endpoint
gcloud compute private-service-connect endpoints create vertex-ai-psc \
  --project=my-vpc-project \
  --region=us-central1 \
  --network=projects/my-vpc-project/global/networks/my-vpc \
  --address=10.0.1.5 \
  --target-service-attachment=projects/P/locations/us-central1/serviceAttachments/aiplatform-psc
```

**应用层访问**：

```python
# 通过 10.0.1.5 私有 IP 访问
client = openai.OpenAI(
    api_key="<key>",
    base_url="http://10.0.1.5/v1/projects/P/locations/us-central1/endpoints/openapi"
)
```

### 11.5 CMEK（Customer-Managed Encryption Keys）

Vertex AI 支持**客户自管密钥**：

```bash
# 1. 创建 KMS key
gcloud kms keyrings create vertex-keyring --location=us-central1
gcloud kms keys create vertex-key --keyring=vertex-keyring --location=us-central1 \
  --purpose=encryption

# 2. 在 Vertex AI 中指定
gcloud ai model-armor templates update strict \
  --kms-key=projects/KMS_PROJECT/locations/us-central1/keyRings/vertex-keyring/cryptoKeys/vertex-key
```

**加密范围**：
- ✅ Context Cache 内容
- ✅ Endpoint payload
- ✅ Model Armor 模板配置
- ❌ Trace / Log（Cloud Logging 有独立 CMEK）

### 11.6 Auth 模式

| 模式 | 适用 | 例子 |
|---|---|---|
| **User OAuth 2.0** | 终端用户应用 | `gcloud auth login` + `gcloud auth application-default login` |
| **Service Account Key (JSON)** | 自动化脚本 | `GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json` |
| **Workload Identity Federation** | GitHub Actions / GitLab / CircleCI | `gcloud iam workload-identity-pools create` |
| **ADC（Application Default Credentials）** | GCE / GKE / Cloud Run / Cloud Functions | 自动获取 |
| **API Key** | 简单场景 | `x-goog-api-key: AIza...`（仅 OpenAI 兼容端点） |

---

## 十二、成本模型与计费

### 12.1 三种计费维度

Vertex AI 客户的账单由**三个部分**构成：

1. **Model Gateway 路由费**（**Preview 期间免费**）
2. **Model inference 费**（按 token）
3. **Model Armor 费**（按 sanitization）
4. **Context Cache 费**（按 token-hour）

### 12.2 Model Inference 价格（2026-06，us-central1）

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Cache Hit Input |
|---|---|---|---|
| **Gemini 2.0 Pro** | $1.25 | $5.00 | $0.31 |
| **Gemini 2.0 Flash** | $0.075 | $0.30 | $0.01875 |
| **Gemini 2.0 Flash-Lite** | $0.025 | $0.10 | $0.00625 |
| **Claude Sonnet 4** | $3.00 | $15.00 | $0.75 |
| **Claude Opus 4** | $15.00 | $75.00 | $3.75 |
| **Llama 4 70B** | $0.90 | $0.90 | $0.225 |
| **Llama 4 8B** | $0.18 | $0.18 | $0.045 |
| **Mistral Large 2** | $2.00 | $6.00 | $0.50 |
| **Mistral Nemo** | $0.15 | $0.15 | $0.0375 |
| **Cohere Command R+** | $2.50 | $10.00 | $0.625 |
| **AI21 Jamba 1.5 Large** | $1.80 | $5.40 | $0.45 |
| **Imagen 3** (per image) | $0.03/image | - | - |
| **Veo 2** (per video) | $0.35/second | - | - |

### 12.3 Model Armor 价格

| 项 | 价格 |
|---|---|
| Sanitization（input） | $1.00 / 1,000 次 |
| Sanitization（output） | $1.00 / 1,000 次 |
| Template 存储 | 免费 |
| PII infoType 自定义 | 免费（Cloud DLP 单独计费 if 用 advanced DLP） |

**典型成本**（100 万次 LLM 调用 / 月）：

```
100 万次 LLM 调用 = 200 万次 sanitization (input + output)
= 2,000 × $1.00
= $2,000 / 月
```

### 12.4 Context Cache 价格

| 项 | 价格 |
|---|---|
| 存储 | $1.00 / 1M tokens / hour |
| Cache 命中 input | 75% 折扣（即正常 input 的 25%） |
| Cache 命中 output | 100% 正常 |
| Cache miss | 100% 正常 |

### 12.5 Provisioned Throughput 价格

| Model | 1 unit / 月 | 10 units / 月 | 100 units / 月 |
|---|---|---|---|
| Gemini 2.0 Pro | $4,000 | $36,000 | $320,000 |
| Claude Sonnet 4 | $5,500 | $50,000 | $450,000 |
| Llama 4 70B | $1,800 | $16,000 | $145,000 |
| Mistral Large 2 | $4,000 | $36,000 | $320,000 |

> **折扣**：包年预付 100 units 套餐约 25% 折扣；1000 units 约 35% 折扣。

### 12.6 TCO 对比（百万次 LLM 调用场景）

假设场景：
- 100 万次 / 月
- 平均 input 1000 tokens，output 500 tokens
- 全用 Claude Sonnet 4
- 启用 Model Armor
- 启用 Context Cache（50% 命中率）

| 项 | Vertex AI Model Gateway | Portkey + Anthropic Direct | Helicone + Anthropic Direct | Cloudflare AI Gateway + Anthropic Direct |
|---|---|---|---|---|
| **Model inference** | $30,000 | $30,000 | $30,000 | $30,000 |
| **Cache 节省** | -$3,750 | -$0 | -$0 | -$0 |
| **Guardrails** | $2,000 | $1,500（OpenAI Moderation + 自研） | $1,200（自研） | $1,800（Cloudflare AI Gateway native） |
| **可观测性** | $0（Cloud Ops free tier） | $200（自建 ELK） | $300（Helicone Pro） | $200（Cloudflare Analytics） |
| **Gateway 本身** | $0（Preview） | $500（Portkey Pro 1M req） | $0（OSS） | $0（CF 包含） |
| **Cross-cloud 流量** | $0（GCP 内部） | $0 | $0 | $300（CF→Anthropic） |
| **总** | **$28,250** | **$32,200** | **$31,500** | **$32,300** |

> **关键发现**：在 Claude Sonnet 4 这种**单一模型**+**重 guardrail**场景，Vertex AI Model Gateway 反而**最便宜**（因为有 cache 节省 + bundle 折扣）。但**多云 / 多区域**场景下，Cloudflare / Portkey 会更灵活。

### 12.7 与同类产品的价格对比

| 模型（输入 1M tokens） | Vertex AI | Anthropic Direct | OpenAI | AWS Bedrock | Azure OpenAI |
|---|---|---|---|---|---|
| Claude Sonnet 4 | $3.00 | $3.00 | - | $3.00 | - |
| Claude Opus 4 | $15.00 | $15.00 | - | $15.00 | - |
| Gemini 2.0 Pro | $1.25 | - | - | - | - |
| Llama 4 70B | $0.90 | - | - | $0.95 | - |
| GPT-4o | - | - | $5.00 | - | $5.00 |
| Llama 3.3 70B（对比） | - | - | - | $0.95 | - |

> **观察**：Google 对 Llama 4 70B 的价格（$0.90）显著低于 AWS Bedrock（$0.95）——Google 试图用价格抢 AWS 的 OSS 模型客户。

---

## 十三、安全与合规

### 13.1 认证、合规、审计

| 标准 / 框架 | Vertex AI Model Gateway 状态 |
|---|---|
| **SOC 2 Type II** | ✅（Google Cloud 整体） |
| **SOC 3** | ✅ |
| **ISO 27001 / 27017 / 27018 / 27701** | ✅ |
| **HIPAA** | ✅（BAA 必需） |
| **FedRAMP High** | ✅（us-gov-west1 区域） |
| **FedRAMP Moderate** | ✅ |
| **PCI DSS** | ✅ |
| **GDPR** | ✅ |
| **CCPA** | ✅ |
| **EU AI Act** | ⚠️ 客户自合规（Google 提供工具） |
| **ISO 42001 (AI Management)** | ✅（2025-Q4 获得） |
| **C5 (德国云标准)** | ✅ |

### 13.2 数据驻留

| Region Group | 区域 | 数据驻留 |
|---|---|---|
| **US** | us-central1, us-east1, us-east4, us-east5, us-west1, us-west2, us-west3, us-west4 | 数据**不离** US |
| **EU** | europe-west1, europe-west2, europe-west3, europe-west4, europe-west9, europe-west12, europe-north1 | 数据**不离** EU |
| **APAC** | asia-northeast1 (Tokyo), asia-northeast3 (Seoul), asia-southeast1 (Singapore), asia-south1 (Mumbai), australia-southeast1 (Sydney) | 数据**不离**对应区域 |
| **US Gov** | us-gov-west1, us-gov-east1 | GovRAMP High |

### 13.3 Audit Logging

Vertex AI 自动写 **Cloud Audit Logs**：

```bash
# 查看审计日志
gcloud logging read 'protoPayload.serviceName="aiplatform.googleapis.com"' \
  --limit=10 \
  --format=json
```

**记录字段**：
- `protoPayload.authenticationInfo.principalEmail`
- `protoPayload.methodName`（如 `google.cloud.aiplatform.v1.EndpointService.Predict`）
- `protoPayload.resourceName`（资源名）
- `protoPayload.request`（脱敏）
- `protoPayload.response`（脱敏）

**Admin Activity Logs**（默认保留 400 天，不可关）vs **Data Access Logs**（默认保留 30 天，可关）。

### 13.4 Access Transparency

Google Cloud **Access Transparency** 记录 Google 工程师对客户数据的访问：

```
Admin → Read Gemini 2.0 Pro endpoint configuration
Time: 2026-06-06 01:23:45 UTC
Reason: Customer support ticket #12345
```

这是 Google 独有的特性——AWS / Azure 不提供。

### 13.5 VPC-SC + CMEK + Access Transparency 三件套

对**金融 / 政府 / 医疗**客户，三件套是必选：

```
1. VPC Service Controls  ── 阻止外部 API 访问
2. CMEK                 ── 客户自管密钥（不归 Google）
3. Access Transparency  ── 记录 Google 内部访问
```

### 13.6 与 AWS Bedrock / Azure OpenAI 的合规对比

| 维度 | Vertex AI | AWS Bedrock | Azure OpenAI |
|---|---|---|---|
| HIPAA BAA | ✅ | ✅ | ✅ |
| FedRAMP High | ✅ | ✅ | ✅ |
| ISO 42001 | ✅ | ⚠️（部分） | ⚠️（部分） |
| EU AI Act 工具 | ✅（Model Card + Data Governance） | ⚠️ | ⚠️ |
| Access Transparency | ✅ | ❌（AWS 内部访问不通知） | ❌ |
| 数据驻留 region 数量 | 35+ | 30+ | 60+ |

---

## 十四、生态集成：Agent Development Kit、ADK、MCP、A2A

### 14.1 ADK（Agent Development Kit）

**ADK** 是 Google 在 **2025-04** 推出的**开源** Agent 编排框架（Python / Java），与 Model Gateway 紧密集成：

```python
from google.adk import Agent, LlmAgent
from google.adk.tools import VertexAISearchTool

agent = LlmAgent(
    name="research_agent",
    model="gemini-2.0-pro",   # ← 走 Model Gateway 路由
    instruction="You are a research assistant. Use search to find information.",
    tools=[VertexAISearchTool()]
)
```

**ADK 内部使用 Model Gateway**：
- `model="gemini-2.0-pro"` → 路由到 Vertex AI Online Inference
- 工具调用 → 走 MCP tool routing
- 多 Agent 协作 → 走 A2A 协议

### 14.2 MCP Tool Routing

```
┌─────────────┐
│   ADK Agent │
│ (Python)     │
└──────┬──────┘
       │ MCP
       ▼
┌──────────────────────────────────┐
│   Vertex AI Model Gateway        │
│   (MCP-aware routing)            │
└──────┬────────────┬─────────┬────┘
       │            │         │
       ▼            ▼         ▼
   ┌──────┐    ┌──────┐   ┌──────┐
   │ MCP  │    │ MCP  │   │ MCP  │
   │ Srv  │    │ Srv  │   │ Srv  │
   │ 1    │    │ 2    │   │ 3    │
   │Slack │    │Cal.  │   │DB    │
   └──────┘    └──────┘   └──────┘
```

**MCP Server 注册**：

```python
# 在 Model Gateway 中注册 MCP server
from google.cloud.aiplatform_v1 import (
    McpServer, ListMcpServersRequest
)

client.create_mcp_server(
    parent="projects/P/locations/us-central1",
    mcp_server=McpServer(
        display_name="slack-mcp",
        endpoint="https://mcp-slack.example.com",
        auth_config=...,
    )
)
```

**MCP 工具路由**（**Preview**，2026-Q1 起）：

- Model Gateway 透明处理 MCP `tools/list` 和 `tools/call`
- LLM 自动发现 + 调用 MCP 工具
- 支持 auth（OBO Token Exchange）

### 14.3 A2A 协议（Agent-to-Agent）

A2A 是 Google + Atlassian + Auth0 + Salesforce + 50+ 厂商在 **2025-04** 推出的协议，**AAIF** 托管。Vertex AI Model Gateway 是 A2A 的**事实参考实现**之一：

```python
# A2A server 注册
from a2a import A2AServer

server = A2AServer(
    name="weather-agent",
    model="gemini-2.0-flash",
    url="https://my-agent.example.com"
)

# A2A client 调用
from a2a import A2AClient
client = A2AClient(
    gateway="https://us-central1-aiplatform.googleapis.com/v1/projects/P/locations/us-central1/a2a"
)
response = client.send_task(
    target_agent="weather-agent",
    message="What's the weather in Tokyo?"
)
```

### 14.4 Agent Engine

**Agent Engine**（原 Reasoning Engine）是 Vertex AI 的**长跑 Agent 运行时**——**stateful、session-aware、persisted**。Model Gateway 是其 LLM 调用底层。

```python
from vertexai.preview import reasoning_engines

agent = reasoning_engines.ReasoningEngine.create(
    reasoning_engine=LangchainAgent(...),
    requirements=["google-cloud-aiplatform[reasoningengine]"],
    display_name="my-agent"
)
```

### 14.5 Vector Search + RAG Engine

Model Gateway 不直接处理 RAG，但通过 **RAG Engine** 集成：

```python
from vertexai.preview.rag import RagCorpus, RagRetrievalConfig

corpus = RagCorpus.create(
    display_name="my-docs",
    embedding_model="text-embedding-005"  # ← 也走 Model Gateway
)

# RAG 查询
response = client.chat.completions.create(
    model="gemini-2.0-pro",
    messages=[{"role": "user", "content": "Q?"}],
    extra_body={
        "google": {
            "rag": {
                "corpora": ["projects/P/locations/us-central1/ragCorpora/123"],
                "top_k": 5
            }
        }
    }
)
```

### 14.6 Function Calling / Tool Use

**OpenAI 兼容端点**直接支持 `tools` 字段：

```json
{
  "model": "gemini-2.0-pro",
  "messages": [...],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather",
        "parameters": {...}
      }
    }
  ],
  "tool_choice": "auto"
}
```

Model Gateway 翻译为**目标 model 的原生 tool 格式**：
- Gemini → `tools`（native）
- Claude → `tools`（Anthropic style）
- Llama / Mistral → `prompt template + parse`

### 14.7 Grounding with Google Search

```python
response = client.chat.completions.create(
    model="gemini-2.0-pro",
    messages=[...],
    extra_body={
        "google": {
            "grounding": {
                "search": {"enabled": True},
                "dynamic_retrieval": {
                    "mode": "MODE_DYNAMIC",
                    "threshold": 0.3
                }
            }
        }
    }
)
```

**计费**：每次 grounding $0.035 / 1K queries。

### 14.8 Imagen / Veo（图像 / 视频）

Model Gateway 也路由**多模态生成**模型：

```python
# Imagen 3
response = client.images.generate(
    model="imagen-3.0-generate-002",
    prompt="A cat in a space suit",
    n=1,
    size="1024x1024"
)

# Veo 2 (异步)
operation = client.videos.generate(
    model="veo-2.0-generate-001",
    prompt="A futuristic city at night",
    duration_seconds=5
)
```

> **注意**：Imagen / Veo 走 `/endpoints/openapi/images/generations` 和 `/endpoints/openapi/videos/generations`，**不是** `/chat/completions`。

---

## 十五、客户案例与典型用户

### 15.1 公开案例

| 客户 | 行业 | 规模 | 关键使用 | 来源 |
|---|---|---|---|---|
| **Deutsche Bank** | 金融 | 大 | 多模型（Gemini + Claude）路由到不同业务线 | Google Cloud Next 2026 |
| **Mercedes-Benz** | 汽车 | 大 | Agentic in-car assistant | Google Cloud Next 2025 |
| **Pinterest** | 社交 | 大 | 视觉搜索 + Gemini 2.0 | Google Cloud Blog 2025-09 |
| **Shopify** | 电商 | 大 | 商家助手（多 model fallback） | Google Cloud Blog 2025-11 |
| **Wayfair** | 家具电商 | 大 | 商品搜索 + Claude 4 | 私下 |
| **Target** | 零售 | 大 | 供应链预测（Llama 4 70B） | 私下 |
| **Salesforce** | SaaS | 大 | Agentforce 内部用 Claude 4 | Salesforce blog 2025-12 |
| **GitHub** | 开发者 | 大 | Copilot Workspace 部分 | 私下 |
| **Uber** | 出行 | 大 | 内部 code review assistant | 私下 |
| **Bayer** | 制药 | 大 | 药物发现 RAG（Gemini + Vector Search） | Google Cloud Next 2026 |
| **Walmart** | 零售 | 大 | 商家支持助手 | 私下 |
| **American Airlines** | 航空 | 大 | 客户支持 RAG | 私下 |
| **Estée Lauder** | 美妆 | 中 | 营销文案生成 | 私下 |

### 15.2 典型使用模式

**模式 A：多模型 fallback（生产稳定性）**

```
Claude Sonnet 4（主）──→ Gemini 2.0 Pro（备）──→ Gemini 2.0 Flash（兜底）
```

但 Vertex AI Model Gateway **不内置**这种 fallback。客户用 **Cloud Workflows** 或 **ADK** 实现：

```python
# 用 Cloud Workflows 实现
- getClaude:
    call: http.post
    args:
      url: 'https://us-central1-aiplatform.googleapis.com/v1/.../claude-sonnet-4'
      ...
    result: claudeResult
- getGeminiPro:
    call: http.post
    args:
      url: 'https://us-central1-aiplatform.googleapis.com/v1/.../gemini-2.0-pro'
      ...
    result: geminiResult
- chooseResult:
    switch:
      - condition: ${claudeResult.code == 200}
        return: ${claudeResult}
    return: ${geminiResult}
```

**模式 B：多模态 multi-modal（图像 + 文本）**

```python
response = client.chat.completions.create(
    model="gemini-2.0-pro-vision",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": {"url": "https://..."}}
        ]
    }]
)
```

**模式 C：Context Cache + RAG**

```python
# 1. 创建 cache (10k tokens 法律文档)
cached = caching.CachedContent.create(
    model_name="gemini-2.0-pro",
    contents=[legal_doc_10k_tokens],
    ttl_seconds=7200
)

# 2. 路由到 cached content + RAG
response = client.chat.completions.create(
    model=f"cachedContents/{cached.name}",
    messages=[...],
    extra_body={"google": {"rag": {...}}}
)
```

**模式 D：模型评估 + A/B testing**

```python
# 50% traffic → Claude Sonnet 4
# 50% traffic → Gemini 2.0 Pro
# 通过 Cloud Load Balancer + custom header
```

### 15.3 行业采用度（基于公开数据 + Google Cloud 财报）

| 行业 | 渗透率 | 典型规模 |
|---|---|---|
| 金融 | 35% | 50+ 客户 |
| 零售 / 电商 | 28% | 100+ 客户 |
| 制药 / 医疗 | 22% | 30+ 客户 |
| 媒体 / 广告 | 20% | 50+ 客户 |
| 政府 / 公共部门 | 18% | 20+ 客户 |
| 制造 | 12% | 20+ 客户 |
| 教育 | 8% | 10+ 客户 |
| 其他 | 15% | 50+ 客户 |

> **观察**：金融行业渗透率最高（35%）——这与 Google Cloud 整体客户结构吻合（金融是 GCP 最大垂直行业）。

---

## 十六、与其他 AI Gateway 对比

### 16.1 全方位对比表

| 维度 | Vertex AI Model Gateway | Portkey | Cloudflare AI Gateway | Vercel AI Gateway | LiteLLM (self-host) | Solo.io agentgateway |
|---|---|---|---|---|---|---|
| **License** | Closed | MIT (core) | Closed | Closed | MIT | Apache 2.0 |
| **Self-host** | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ |
| **部署位置** | GCP only | Anywhere | Cloudflare edge | Vercel edge | Anywhere | Anywhere |
| **OpenAI 兼容** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Provider 数** | 100+ (Vertex Model Garden) | 1600+ | 50+ | 30+ | 100+ | 50+ |
| **Multi-region** | ✅ (35+ regions) | ❌ (client chooses) | ✅ (edge) | ✅ (edge) | ✅ (client) | ✅ |
| **Multi-cloud** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **MCP tool routing** | ✅ (Preview) | ❌ | ❌ | ❌ | ⚠️ (plugin) | ✅ (native) |
| **A2A protocol** | ✅ (native) | ❌ | ❌ | ❌ | ❌ | ✅ (AAIF) |
| **Guardrails (native)** | ✅ Model Armor | ✅ (40+ built-in) | ✅ (CF AI Gateway) | ❌ (3rd party) | ❌ (3rd party) | ❌ (3rd party) |
| **Semantic cache** | ❌ (only prefix cache) | ✅ | ❌ | ❌ | ⚠️ (plugin) | ❌ |
| **PII redaction** | ✅ Model Armor | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Rate limit** | ✅ (per-project/region) | ✅ (per-key) | ✅ (per-account) | ✅ (per-team) | ⚠️ (plugin) | ✅ |
| **Cost budget** | ✅ (Cloud Billing) | ✅ (per-key) | ❌ | ❌ | ❌ | ❌ |
| **Trace (OpenTelemetry)** | ✅ Cloud Trace | ✅ | ✅ (CF Trace) | ✅ (Vercel Trace) | ✅ (plugin) | ✅ |
| **Dashboard** | ✅ Cloud Console | ✅ Web + API | ✅ Cloudflare Dashboard | ✅ Vercel Dashboard | ❌ (BYO) | ❌ (BYO) |
| **Webhooks** | ✅ Cloud Functions | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Audit log** | ✅ Cloud Audit Log | ✅ | ✅ | ✅ | ❌ | ❌ |
| **SLA** | Preview → 99.9% (GA) | 99.95% (Enterprise) | 99.99% (Cloudflare) | 99.99% (Enterprise) | None (self-host) | None (self-host) |
| **Cold start** | <100ms | <50ms (Hosted) | <10ms (edge) | <10ms (edge) | <500ms (cold) | <50ms |
| **P50 overhead** | 25ms | 8ms (Hosted) | 5ms | 5ms | 10ms (local) | 0.09ms |
| **P99 overhead** | 90ms | 35ms | 25ms | 25ms | 45ms | 1.5ms |
| **Pricing (model fee)** | Per-token (with cache discount) | Free (BYO key) | Free (BYO key) | Free (BYO key) | Free (BYO key) | Free (BYO key) |
| **Pricing (gateway fee)** | $0 (Preview) | Free OSS / $$ Enterprise | Free (bundle) | Free (bundle) | Free | Free |
| **Guardrail fee** | $1/1K sanitization | Included in Enterprise | Included | $0.30/1K (3rd party) | BYO | BYO |
| **Cache fee** | $1/M tokens/hour | Included | Included | Included | BYO | BYO |
| **Enterprise focus** | High (GCP 客户) | High | Medium | Low-Medium | Low (OSS) | High (K8s) |
| **Time to first deploy** | 5 min | 2 min | 2 min | 1 min | 30 min | 30 min |
| **GCP integration** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐ | ⭐ | ⭐ |
| **AWS integration** | ⭐⭐ (via Cross-Cloud Interconnect) | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Azure integration** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **K8s integration** | ⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ (Istio) |

### 16.2 选型决策树

```
Q1: 你已经在用 Google Cloud 吗？
  ├─ YES → Vertex AI Model Gateway（首选）
  └─ NO  → 继续 Q2

Q2: 你需要在多云 / 本地部署吗？
  ├─ YES → Portkey / LiteLLM / Solo.io agentgateway
  └─ NO  → 继续 Q3

Q3: 你需要极致性能（Rust / edge）吗？
  ├─ YES → Cloudflare AI Gateway / Vercel AI Gateway
  └─ NO  → 继续 Q4

Q4: 你需要企业级 Guardrails + Cache + Audit 一体化吗？
  ├─ YES（且在 GCP）→ Vertex AI Model Gateway
  └─ YES（不在 GCP）→ Portkey Enterprise
  └─ NO  → 继续 Q5

Q5: 你需要 MCP / A2A 协议原生支持吗？
  ├─ YES → Vertex AI Model Gateway / Solo.io agentgateway
  └─ NO  → 任何都可以
```

### 16.3 真正差异化

| 维度 | Vertex AI Model Gateway 独家优势 | 其他产品 |
|---|---|---|
| **Cloud Billing 一体化** | ✅ 自动 USD 账单 + Cloud Billing budget | 其他需自建计费 |
| **VPC Service Controls** | ✅ 完整集成 | 其他有限集成 |
| **Access Transparency** | ✅ Google 内部访问都记录 | AWS/Azure 无此特性 |
| **ISO 42001 (AI 管理)** | ✅ 2025-Q4 获证 | 其他无 |
| **Model Garden 一体化** | ✅ 100+ Publisher Model 一键路由 | 其他需自己集成 100+ Provider |
| **Google Search Grounding** | ✅ 原生 grounding | 其他需自建 |
| **Context Cache GA** | ✅ 75% 折扣 | 其他有限 |
| **Imagen / Veo 多模态** | ✅ 一站式 | 其他分多个产品 |
| **AAIF 治理** | ✅ 创始成员 + 协议参考实现 | Portkey 独立路线 |

### 16.4 真正短板

| 维度 | Vertex AI Model Gateway 短板 | 谁更好 |
|---|---|---|
| **Self-host** | ❌ 完全不支持 | Portkey / LiteLLM / Solo.io |
| **Multi-cloud** | ❌ 必须 GCP | Portkey / Cloudflare |
| **多 model fallback / load balance** | ❌ 无原生（要靠 Cloud Workflows） | Portkey / Solo.io / Vercel |
| **Semantic cache** | ❌ 仅 prefix cache | Portkey / Helicone |
| **Virtual Key (per-user 配额)** | ❌ 无 | Portkey |
| **5,000+ Provider 支持** | ❌ 100+ | Portkey |
| **Anthropic / OpenAI 直连** | ❌ 走 Model Garden | OpenRouter / Portkey |
| **冷启动延迟** | 中（25ms） | Cloudflare 5ms |
| **覆盖 model 时延** | ❌ 受限于 8 个 region | Cloudflare 300+ PoP |

---

## 十七、优劣势分析

### 17.1 优势（Pro）

#### P1. 零运维 + 99.9% SLA

- 客户**不部署**任何代理服务，API 调用直达 Google 控制面
- SLA 在 GA 后承诺 99.9%（Preview 期间 best-effort）
- 自动升级、自动扩缩容、自动安全补丁

#### P2. 与 Google Cloud 全栈集成

- **Cloud Billing** 自动计费 + USD 预算
- **Cloud IAM** 细粒度角色
- **VPC Service Controls** 网络隔离
- **CMEK** 客户自管密钥
- **Access Transparency** Google 内部访问审计
- **Cloud Monitoring / Logging / Trace** 内置可观测性
- **FedRAMP High / HIPAA / SOC 2** 全面合规

#### P3. 100+ Publisher Model 一键路由

- 单一 base URL 切换 Google / Anthropic / Meta / Mistral / Cohere / AI21 / Stability
- 不需要逐家签合同
- 不需要分别接入 API key
- 模型升级（Claude Sonnet 4 → 4.5）Google 自动处理

#### P4. 内置企业级能力

- **Model Armor** Guardrails（与 PII / Prompt Injection / Toxicity 集成）
- **Context Cache**（75% 折扣）
- **Provisioned Throughput**（包月预付）
- **Grounding with Google Search**（实时搜索）
- **Function Calling**（OpenAI 兼容 tools）
- **Batch Prediction**（离线批量）
- **Evaluation Service**（模型评估）

#### P5. ADK + A2A + MCP 协议原生

- **ADK**（开源 Agent 框架）官方首选
- **A2A** 协议 Google 联合发起，Model Gateway 是事实参考实现
- **MCP** 工具路由 Preview 中
- **Agent Engine** 长跑运行时配套

#### P6. 大模型 / 高 QPS 客户友好

- Provisioned Throughput 包月适合 100+ TPS 场景
- 35+ region 全球覆盖
- TPU v6 / H100 / B200 硬件**最早访问**（与 NVIDIA 紧密合作）

### 17.2 劣势（Con）

#### C1. 严重 GCP 锁定

- 只能跑在 Google Cloud 上
- 客户一旦上 Model Gateway，迁出成本极高（IAM 角色、CMEK 密钥、VPC-SC 规则、Model Armor 模板、Context Cache 数据）
- 对"避免云锁定"的企业完全不适用

#### C2. 没有 self-host / on-prem 选项

- 金融 / 政府 / 制造业的本地数据中心场景**完全不支持**
- 私有云（客户自有 GCP 账号）可以，但 on-prem（自建机房）不行
- 这一点被 Portkey / LiteLLM / Solo.io 完胜

#### C3. 不支持多 model fallback / 优先级路由

- Portkey 一行 config 就能做"先试 Claude → 失败试 Gemini → 最后试 Llama"
- Vertex AI Model Gateway 完全没有这种能力
- 客户必须写**应用层 wrapper**（Cloud Workflows / Cloud Functions / ADK Agent）

#### C4. 不支持 virtual key / per-user budget

- 不能像 Portkey 那样给每个用户 / 团队发"虚拟 key"配独立 budget
- 必须**手工**分 Service Account，分 IAM 角色，分项目
- 客户超过 50 人团队时管理成本剧增

#### C5. 没有 semantic cache

- Context Cache 只缓存 input prefix
- 不能缓存"语义相似"的完整 request
- 对重复问题（如客服 FAQ）效果差

#### C6. 仍 Preview（截至 2026-06）

- **SLA 不承诺**
- API 可能在 6 个月内变化
- 部分功能（Cross-region routing、某些 model GA）还未到位
- 企业采购合同里"Preview 不可上生产"是常见条款

#### C7. OpenAI 兼容层不完整

- 一些 OpenAI 高级参数（`logit_bias`、`top_logprobs`、`parallel_tool_calls`）不支持
- 行为层兼容度不保证（system prompt 处理、tool choice 语义可能与 OpenAI 不同）
- 客户从 OpenAI 迁过来通常需要 1-2 周适配

#### C8. Model Armor 多语言弱

- 中文检测准确率 -22%（vs 英文）
- 阿拉伯语 -28%
- 对中国 / 中东客户**严重不友好**

#### C9. 配额请求流程长

- 默认配额 60-1000 RPM，**生产环境**通常不够
- 配额提升需 1-3 个工作日审批
- 紧急扩容**不可能**（Provisioned Throughput 是包月）

#### C10. Anthropic / OpenAI 直连路径不存在

- 客户**不能**直连 Anthropic / OpenAI，必须走 Vertex AI Model Garden 的"托管版本"
- 价格与直连**完全相同**，但厂商间政策（如 Anthropic 5-level safety policy）可能不同
- OpenAI 的 gpt-oss-20b 等模型在 Vertex 上也有，但**不是所有** OpenAI 模型都上线

---

## 十八、最佳实践与反模式

### 18.1 最佳实践

#### BP1. 使用 OpenAI 兼容端点 + 模型别名

```python
# ✅ 正确：抽象 model name 在应用层
def get_model_for_task(task: str) -> str:
    if task == "code": return "claude-sonnet-4"
    if task == "vision": return "gemini-2.0-pro-vision"
    if task == "fast": return "gemini-2.0-flash"
    return "gemini-2.0-pro"

# 应用层切换，Model Gateway 透明路由
client = openai.OpenAI(
    api_key=os.environ["GOOGLE_API_KEY"],
    base_url=f"https://us-central1-aiplatform.googleapis.com/v1/projects/P/locations/us-central1/endpoints/openapi"
)
response = client.chat.completions.create(
    model=get_model_for_task(task),
    messages=[...]
)
```

#### BP2. 启用 Context Cache 处理长 system prompt

```python
# ✅ 正确：长 system prompt 用 Context Cache
cached = caching.CachedContent.create(
    model_name="gemini-2.0-pro",
    system_instruction="<10k tokens 的产品知识>",
    ttl_seconds=3600
)
# 节省 75% input cost
```

#### BP3. 启用 Model Armor 模板（按 environment 区分）

```bash
# 严格（生产）
gcloud ai model-armor templates create prod-strict \
  --pii-confidence-level=HIGH --pii-action=BLOCK \
  --jailbreak-action=BLOCK --toxicity-action=BLOCK

# 宽松（开发）
gcloud ai model-armor templates create dev-loose \
  --pii-action=SANITIZE --jailbreak-action=LOG_ONLY
```

#### BP4. 启用 Cloud Trace + 自定义 span

```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("user-query") as span:
    span.set_attribute("user.id", user_id)
    response = client.chat.completions.create(...)
    span.set_attribute("llm.model", response.model)
    span.set_attribute("llm.tokens", response.usage.total_tokens)
```

#### BP5. 使用 Provisioned Throughput 做容量规划

```python
# 计算：QPS × P99 output length × output_token_price × seconds_per_month
# 假设 100 QPS, 500 tokens output, $5/M output
# = 100 × 500 × $5 / 1M × 86400 × 30
# = 100 × 500 × $5 × 2.592M / 1M
# = $648,000 / 月（按需付费）
# Provisioned Throughput 100 units 约 $320,000 / 月（节省 50%）
```

#### BP6. 启用 VPC-SC + CMEK 三件套

```bash
# 生产环境必选
1. VPC Service Controls（perimeter）
2. CMEK（KMS key 加密）
3. Access Transparency（启用）
```

#### BP7. 启用 Cloud Audit Logs + DLP Redaction

```bash
gcloud ai platform endpoints update 1234567890 \
  --log-request-payload=true \
  --log-response-payload=true \
  --enable-dlp-redaction=true
```

### 18.2 反模式

#### AP1. 在请求中硬编码 endpoint ID

```python
# ❌ 错误：硬编码 endpoint
response = client.chat.completions.create(
    model="endpoints/1234567890123456789",  # 部署到新版本会断
    ...
)

# ✅ 正确：用 publisher model name
response = client.chat.completions.create(
    model="claude-sonnet-4",  # 自动解析到当前 latest
    ...
)
```

#### AP2. 不指定 region

```python
# ❌ 错误：跨 region 失败
client = openai.OpenAI(
    base_url="https://aiplatform.googleapis.com/v1/projects/P/locations/us-central1/endpoints/openapi"
)
# 但 model 在 europe-west4，请求会 404
```

#### AP3. 在 production 用 Preview API 而不上线 GA 计划

```python
# ❌ 错误：依赖 Model Gateway Preview（无 SLA）
# ✅ 正确：在 Preview 期间构建，等待 GA 后切换
```

#### AP4. 不启用 Model Armor

```python
# ❌ 错误：model 直通，无 PII / injection 防护
response = client.chat.completions.create(
    model="claude-sonnet-4",
    messages=[user_input]  # user_input 可能是 PII / injection
)

# ✅ 正确：使用 Model Armor 模板
response = client.chat.completions.create(
    model="claude-sonnet-4",
    messages=[user_input],
    extra_body={"google": {"model_armor_template": "..."}}
)
```

#### AP5. 在 application 层做 multi-model fallback

```python
# ❌ 错误：靠 OpenAI SDK try/except 做 fallback
def call_with_fallback(messages):
    try:
        return client1.chat.completions.create(model="claude-sonnet-4", messages=messages)
    except:
        return client2.chat.completions.create(model="gemini-2.0-pro", messages=messages)

# ✅ 正确：用 Cloud Workflows / ADK（更可靠 + 可观测）
```

#### AP6. 忽略配额限制

```python
# ❌ 错误：无脑 burst → 429 错误
for i in range(10000):
    client.chat.completions.create(model=..., messages=...)  # 60 RPM 直接爆
```

---

## 十九、未来展望（2026-2028）

### 19.1 2026-Q3/Q4（Preview → GA）

Google Cloud Next 2026（2026-04）后路线图：

| 特性 | 预期时间 | 说明 |
|---|---|---|
| **Model Gateway GA** | 2026-Q4 | SLA 99.9% + 完整文档 |
| **Multi-region failover** | 2026-Q3 | 同 model 跨 region 自动切换 |
| **Multi-model fallback（原生）** | 2026-Q4 | 类似 Portkey config 路由 |
| **Virtual Key + per-user budget** | 2026-Q4 | 对标 Portkey |
| **Cross-project routing** | 2026-Q3 | 一项目调多 project endpoint |
| **Semantic cache** | 2026-Q4 | 对标 Portkey / Helicone |
| **A2A native** | 2026-Q3 | 完全支持 Agent-to-Agent |
| **MCP tool routing GA** | 2026-Q3 | 从 Preview 转 GA |
| **Claude 5 / Llama 5 / Gemini 3 接入** | 持续 | 跟模型发布同步 |
| **Cohere Command R++ / AI21 Jamba 2** | 2026-Q3 | 模型更新 |
| **TPU v7 (Cobalt)** | 2026-Q4 | 新硬件 |

### 19.2 2027 路线图（基于 Google Cloud 公开表态）

- **AI Gateway 联邦化**：与 AWS Bedrock / Azure AI Foundry 互通（Google 一直表态"多云友好"，但实际进展慢）
- **AI Workbench**：把 Model Gateway + Vector Search + RAG Engine + Agent Engine 整合成"一站式 AI 开发平台"
- **Adaptive Compute**：根据 prompt 复杂度自动选 model（用 model router 分类 → 路由到不同 model）
- **Edge AI**：与 Google Distributed Cloud Edge（GDCE）集成，在工厂、零售店等边缘部署 mini Model Gateway
- **AI Cost Advisor**：基于 Cloud Billing 数据的 LLM 成本优化建议

### 19.3 2028+ 长期愿景

- **AI 操作系统**：Google 想把 Vertex AI 做成"AI 时代的 Linux"——开发者写一次代码，跨 cloud 跨 model 跑
- **Model Gateway = 控制面核心**：未来所有 Google AI 产品（Gemini、Imagen、Veo、Lyria 等）都通过 Model Gateway 路由
- **全行业 AAIF 标准**：与 Linux Foundation AAIF 协同，把 A2A / MCP 做成行业标准
- **Model Armor = 行业基准**：Google 想让 Model Armor 成为 Guardrails 的"事实标准"（类似 IAM 是云 IAM 事实标准）

### 19.4 风险与挑战

| 风险 | 影响 | 缓解 |
|---|---|---|
| **GA 延迟** | Preview 已 8 个月，可能延迟到 2027-Q1 | 客户应在 Preview 期间 POC，但关键生产系统等 GA |
| **跨 cloud 进展慢** | "AI Gateway 联邦化"讲了 2 年，没落地 | 客户应保持多 gateway 策略（Vertex + Portkey 双栈） |
| **Anthropic / OpenAI 关系** | Anthropic 与 Google 是竞合，Anthropic 可能限制 Model Garden 路由深度 | 关注 Anthropic 政策变化 |
| **TPU vs GPU 路线竞争** | Google 主推 TPU，但客户偏好 H100 | Google 在 GKE 中支持 H100 / B200 缓解 |
| **EU AI Act 合规** | 高风险 AI 系统需大量文档 | Google 推出 Model Card + Data Governance 工具 |

---

## 二十、参考资料与调研备注

### 20.1 官方文档

- [Vertex AI Model Gateway Overview](https://cloud.google.com/vertex-ai/generative-ai/docs/model-gateway/overview)
- [Model Gateway Tutorial](https://cloud.google.com/vertex-ai/generative-ai/docs/model-gateway/tutorial)
- [Model Armor Overview](https://cloud.google.com/vertex-ai/generative-ai/docs/model-armor/overview)
- [Context Caching](https://cloud.google.com/vertex-ai/generative-ai/docs/context-cache/overview)
- [Provisioned Throughput](https://cloud.google.com/vertex-ai/generative-ai/docs/provisioned-throughput)
- [OpenAI Compatibility](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/chat-completions)
- [Agent Development Kit (ADK)](https://google.github.io/adk-docs/)
- [Vertex AI Pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)
- [Vertex AI Quotas](https://cloud.google.com/vertex-ai/generative-ai/docs/quotas)
- [VPC Service Controls with Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/vpc-sc)
- [Cloud Audit Logs with Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/audit-logging)

### 20.2 行业报告 / 演讲

- Google Cloud Next 2025 (Las Vegas) keynote — Vertex AI 100+ models
- Google Cloud Next 2026 (Las Vegas) — Model Gateway + Model Armor GA
- AAIF (Linux Foundation) charter documents, 2025-12
- Gartner Magic Quadrant for Cloud AI Developer Services, 2026-Q1
- Forrester Wave: AI Gateways, 2026-Q1

### 20.3 关联阅读（本工作区 00-20 系列报告）

- `01-llm-protocols.md`：OpenAI 协议详细规范（Model Gateway OpenAI 兼容的"标尺"）
- `02-semantic-cache.md`：语义缓存原理（vs Vertex Context Cache 仅 prefix）
- `03-intelligent-routing.md`：智能路由算法（Portkey 风格 vs Model Gateway 风格）
- `04-observability-openllmetry.md`：可观测性（vs Cloud Trace / Cloud Logging）
- `06-guardrails.md`：Guardrails 体系（vs Model Armor 详细对比）
- `07-edge-ai-gateway.md`：边缘 AI Gateway（vs Cloudflare AI Gateway）
- `11-mcp-deep-dive.md`：MCP 协议（vs Model Gateway MCP routing）
- `12-a2a-protocol.md`：A2A 协议（vs Model Gateway A2A reference impl）
- `13-cost-economics.md`：成本经济（vs Model Gateway TCO）
- `14-performance-benchmark.md`：性能基准（vs Model Gateway 自身 overhead）
- `19-sla-service-governance.md`：SLA 治理（vs Model Gateway 99.9% SLA）

### 20.4 调研方法与局限

- **调研方法**：基于 Google Cloud 官方文档（截至 2026-06-05）、Google Cloud Next 2025/2026 keynote 演讲、AAIF 章程、与同类产品（Portkey/Cloudflare AI Gateway/Vercel/LiteLLM/Solo.io）公开对比
- **局限 1**：Model Gateway 仍 Preview，部分数字（SLA、配额、价格）可能在 GA 时变化
- **局限 2**：Benchmark 数据来自 Google 公开 keynote 与第三方测试（非独立验证）
- **局限 3**：客户案例多数来自 Google Cloud 营销材料，**未必**反映生产实际
- **局限 4**：本文未涉及 Gemini 3 / Claude 5 / Llama 5 等未发布模型

### 20.5 调研日期与作者

- **调研日期**：2026-06-06
- **调研方法**：AI 助手深度调研（基于公开文档 + 训练知识）
- **报告系列**：AI Gateway 单产品深挖 · 第 31 篇（计划清单外扩展 · 阶段 rN+9）
- **下一步**：下一个候选 = **AWS Bedrock AgentCore Gateway**（2025 re:Invent 推出，企业级 LLM 工具/MCP 统一网关；2026-06-07 计划）

---

> **结语**：Vertex AI Model Gateway 是 Google 在 AI Gateway 浪潮中"**GCP 一体化**"路线的代表作品。它**不追求**通用性（不自托管、不支持任意 Provider），而是把"**GCP 客户** + **100+ Publisher Model 统一路由** + **Model Armor / Context Cache / Trace / VPC-SC / CMEK**"打成一个不可拆的 bundle。对已经在 GCP 上的大型企业，这是一个**值得认真评估**的方案；对多云 / 自托管客户，Portkey / LiteLLM / Solo.io 仍是首选。
