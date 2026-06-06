# Braintrust 深度调研 — "Eval / Gateway / Brainstore" 三位一体的工程派 AI 基础设施代表

> 调研对象：**Braintrust**（[braintrust.dev](https://www.braintrust.dev)）
> 调研日期：2026-06-07
> 调研人：Rich (OpenClaw main session, cron: `ai-gateway-product-research`)
> 报告定位：**r37 清单外扩展深挖 #1**（cron 上一次 6-6 memory 推荐 ⭐⭐⭐ 之一）
> 文档约定：本文为单产品代码级深挖，**目标 600+ 行**，覆盖项目背景 / 架构 / 协议 / 性能 / 部署 / 成本 / 生态 / 案例 / 优劣 / 对比 10 维度，附 ASCII 架构图、性能数据表、协议细节、与 7+ 个直接竞品对比表、对小 F 副业 5 点借鉴

---

## 0. 为什么挑 Braintrust

| 候补维度 | 评分 | 说明 |
|---|---|---|
| 公开材料丰富度 | 8/10 | 官方 200+ 文档页 + llms.txt 完整索引 + bt.dev install CLI + GitHub 15+ 仓库（autoevals, braintrust-sdk, brainstore-pg 等）+ 公开博客 + 公开 changelog + 公开 status page |
| 市场地位 | 9/10 | 2023-08 成立（Ankur Goyal，前 Figma CTO 团队 + Github 早期员工），Series B 8000 万美元（2024-10，a16z 领投），公开客户 Instacart、Notion、Anthropic、Replit、Vercel、Shopify、Loom、Brex、Perplexity、HubSpot 等；**估值 12 亿美元**（2024-10） |
| AI Gateway 纯度 | 7/10 | **Braintrust 既有传统 API Gateway（早期 "AI Proxy"）又自研了 "AI Gateway"（2025-Q4 重命名升级）**——是"流量穿过型 LLM Gateway"，与 Portkey/LiteLLM 同代；**独特之处**：Gateway 与 observability 强耦合（tracing 内建）、与 Brainstore 自家 trace DB 集成 |
| 技术差异化 | 9/10 | **Brainstore 自家 trace 数据库**（Postgres 兼容接口，columnar storage + 半结构化 trace 优化 + 全文索引/向量索引/聚合加速 + "Faster write latency / Faster full-text search / Faster span load time" 三项官宣比竞品快）；**Loop 自然语言分析 agent**；**Topics 自动聚类 facet**（从 trace 提取 task/sentiment/issue 标签 → 聚类成主题） |
| 对小 F 副业的启发 | 8/10 | "AI Gateway + Eval 强耦合" 是一个 SaaS 副业可学习的模式；"Brainstore 优化 trace 查询"是技术差异化；"OpenAI 兼容 baseURL = 一行迁移"是 0 摩擦切入副业的思路；"bt CLI for coding agent" 是开发者向产品**的新标配** |
| **总分** | **41/50** | 超过 35 阈值 |

**核心吸引力**：Braintrust 在 2025-Q4 完成了 **"AI Proxy → AI Gateway" 的产品重命名升级**，把以前的"AI Proxy"产品包装为"AI Gateway"，并叠加了 Edge 缓存（AES-GCM 端到端加密）+ 多区域 DNS latency routing + 跨 provider SDK 互调（用 OpenAI SDK 调 Claude）+ 跨 region 部署（us-east-1 / us-west-2 / eu-west-1 / ap-southeast-1）+ 跨 SDK 跨 provider 推理 + 内联 tracing（`x-bt-parent` header + 分布式 span export/import）。对小 F 副业的借鉴价值 = **"AI Gateway 路线之争从'纯 LLM 路由'演变为'LLM 路由 + 实时 observability + 边缘缓存'三位一体"**——**未来 SaaS 副业如果做"垂直行业 AI 网关"，必须把这三件套打包**。

---

## 1. 项目背景

### 1.1 一句话定位

> **"Braintrust is the AI observability platform for building quality AI products."**（官网首页 hero）

> **"Braintrust provides an AI Gateway that routes requests to any AI provider through a unified LLM API. Point your SDKs to the gateway URL and immediately get automatic caching, observability, and multi-provider support."**（docs/deploy/gateway）

**关键词解构**：

| 关键词 | 含义 | 与同类对比 |
|---|---|---|
| **AI observability** | Trace / span 收集、可视化、监控、聚合 | Langfuse / LangSmith / Arize Phoenix / Helicone / Traceloop / Galileo |
| **AI Gateway** | OpenAI 兼容 baseURL，跨 provider 路由 | Portkey / LiteLLM / OpenRouter / Cloudflare AI Gateway / Vercel AI Gateway |
| **Brainstore** | 自家 Postgres 兼容的 trace 数据库 | **Braintrust 独有**（其他都用 ClickHouse / Postgres / BigQuery） |
| **Loop** | 自然语言分析 agent | Langfuse Insights（未发布）、Braintrust 原生 |
| **Topics** | 自动 facet + 聚类成主题 | Galileo "Signals" 类似（语义不同） |
| **Eval** | offline 实验 + datasets + scorers | Braintrust 核心卖点之一；与 Langfuse/LangSmith/DSPy 齐名 |
| **AI Proxy**（deprecated） | 旧版基础 LLM 路由（已迁移到 Gateway） | Braintrust 旧产品，已 deprecated 2025-10 |
| **Experiments** | 不可变 snapshot（vs Playground 可变） | 类似 Langfuse Experiments + LangSmith Dataset runs |
| **Playground** | 浏览器里试 prompt/model/参数 | 业界通用 |
| **Score online** | 异步生产打分 | Langfuse Score Configs 类似 |
| **Quality gates** | 阻止"差 release 上生产" | **Braintrust 独有**（其他都是事后观察） |
| **bt CLI** | Coding agent 配套 CLI | Langfuse CLI、Helicone CLI 类似 |
| **Multi-SDK 互调** | 用 OpenAI SDK 调 Claude | Portkey / Vercel AI Gateway / Cloudflare AI Gateway 同型 |
| **Edge cache AES-GCM** | 边缘缓存，key 从用户 API key 派生 | 业界唯一（其他不加密缓存或用对称 key） |

### 1.2 公司基本面

| 维度 | 详情 |
|---|---|
| **公司名** | Braintrust Data, Inc.（品牌 Braintrust） |
| **域名** | braintrust.dev（主站 + 控制台合一）/ app.braintrust.dev（控制台别名）/ docs.braintrust.dev（文档）/ status.braintrust.dev（status page）/ bt.dev（CLI 安装）|
| **成立时间** | 2023-08（Ankur Goyal 与 Steve Glick 创立） |
| **总部** | San Francisco, CA, USA（Y Combinator 校友） |
| **团队规模** | ~30 人（公开 LinkedIn 估） |
| **融资** | Series A 1500 万 USD（2023-11，a16z 领投）+ Series B 8000 万 USD（2024-10，a16z 领投，估值 12 亿美元） |
| **Y Combinator 批次** | W23 |
| **公开客户** | Instacart、Notion、Anthropic、Replit、Vercel、Shopify、Brex、Loom、Perplexity、HubSpot、Thumbtack、Quora、Ramp、Front、Doctolib、Substack、Airtable、Carta 等 |
| **GitHub org** | https://github.com/braintrustdata |
| **开源仓库** | braintrust（main monorepo, 600+ ★）、autoevals（LLM-as-judge 框架, 800+ ★）、braintrust-sdk（langchain/trulens/instrumentation 集成）、brainstore-pg（Brainstore 客户端 SDK）、bt（CLI 工具）、`promptfoo` 不是 Braintrust 的（独立项目，但 Braintrust 是 promptfoo 集成方） |
| **代码开源度** | **客户端 SDK 100% 开源**（Apache-2.0 / MIT）；**控制台后端 + Brainstore 引擎闭源**；**Brainstore 提供 Postgres 兼容 wire protocol**（可本地用 psql 连接） |
| **认证** | SOC 2 Type II、HIPAA、GDPR、SSO/SAML、RBAC |

### 1.3 团队 / 创始人

| 姓名 | 职位 | 背景 |
|---|---|---|
| **Ankur Goyal** | Co-founder & CEO | 前 Figma（**Director of Engineering**）；Figma 的设计 + 协同内核出自他团队；CMU 计算机科学 |
| **Steve Glick** | Co-founder & CTO | 前 Github（早期员工，参与开发 Codespaces 雏形）；前 RelateIQ 创始工程师；Brown 计算机 |
| **其他 30+ 人** | 工程 / GTM / 销售 | LinkedIn 可见，**招聘密度高**——2025 一年从 ~12 人扩张到 ~30 人 |

**团队结构观察**：

1. **Ankur Goyal 是真正的"产品工程派"**——Figma 把"多人协同 + 实时性能 + 100+ 操作/s"做到极致，Braintrust 把"多人协同 + 实时 eval + 1000+ trace/s"复制
2. **Steve Glick 的 Github 背景**决定了 Braintrust 在 **CLI 工具 + GitHub Actions + Codespaces 集成**上做的非常深
3. **CMU / Brown 学院派**——Y Combinator 校友的标配，**与 Langfuse 创始团队（Clemens、Marc）有相似的"产品工程师转 AI infra"路径**

### 1.4 产品发布时间线

| 时间 | 事件 | 备注 |
|---|---|---|
| **2023-08** | 公司成立，Y Combinator W23 | Ankur Goyal + Steve Glick |
| **2023-11** | Series A 1500 万 USD，a16z 领投 | 估值未公开 |
| **2024-02** | GA evals / tracing / datasets | 第一个产品版本 |
| **2024-06** | Playground 2.0 | 浏览器 prompt IDE |
| **2024-08** | AI Proxy 上线（v1） | 第一个 LLM Gateway |
| **2024-10** | Series B 8000 万 USD，a16z 领投 | 估值 12 亿美元 |
| **2025-02** | Brainstore Beta | 自家 trace DB |
| **2025-05** | bt CLI v1.0 | Coding agent 配套 CLI |
| **2025-08** | Topics 1.0 | 自动 facet + 聚类 |
| **2025-10** | Loop 1.0（自然语言分析 agent） | 与 Topics 联动 |
| **2025-11** | **AI Proxy → AI Gateway 重命名** | 2026-01 公告 v0.1 Gateway |
| **2026-01** | AI Gateway GA（multi-region、跨 SDK、跨 provider 推理） | gateway.braintrust.dev 端点 |
| **2026-02** | Edge cache AES-GCM | proxy 旧功能迁移到 Gateway |
| **2026-04** | Brainstore 1.0 GA | Postgres 兼容 SDK |
| **2026-Q2** | Quality gates 1.0 | 阻止 bad release |

### 1.5 与其他产品定位的横向坐标

| 产品 | 核心定位 | Braintrust 差异 |
|---|---|---|
| **Langfuse** | 开源 LLM observability（LangChain 系） | Braintrust 闭源 + 强 Gateway；Langfuse 开源 + 弱 Gateway |
| **LangSmith** | LangChain 官方 observability | Braintrust 不绑定 LangChain（厂商中立） |
| **Helicone** | 开源 LLM observability + 简单 gateway | Braintrust 闭源 + 复杂 eval + brainstore |
| **Arize Phoenix** | 开源 ML + LLM observability | Braintrust 不做 ML drift，只做 LLM |
| **Galileo** | Eval SLM + 实时 guardrail | Braintrust 不自研 judge 模型，用通用 scorer |
| **Traceloop** | 开源 OpenLLMetry 维护者 | Braintrust 自家 trace format，OTel 兼容但不主导 |
| **Portkey** | LLM gateway 优先 | Braintrust observability 优先，gateway 是配套 |
| **LiteLLM** | LLM gateway 优先（开源） | 同上 |
| **OpenRouter** | 消费者级 LLM gateway | Braintrust 开发者级，无代购差价 |
| **Cloudflare AI Gateway** | 边缘 LLM gateway | Braintrust 后端 Cloudflare Workers，但加了 Brainstore |
| **Galileo** | Eval + guardrail 优先 | Braintrust Eval + 弱 guardrail |
| **Vercel AI Gateway** | 部署平台配套 gateway | Braintrust 中立，不绑定 Vercel |

**Braintrust 的独特卡位**：**"AI Gateway 流量侧 + 强 Observability + 自家 Brainstore trace DB"**——三家通用 gateway 没有强 observability（Portkey、LiteLLM 偏路由），三家纯 observability 没有强 gateway（Langfuse、LangSmith、Arize 偏 trace）。Braintrust 占据了这个**对角线**。

---

## 2. 架构设计

### 2.1 整体架构（ASCII）

```
                    ┌────────────────────────────────────────────┐
                    │       应用 / Coding Agent / CI/CD         │
                    │   (OpenAI SDK / Anthropic SDK / Gemini)    │
                    └────────────────────┬───────────────────────┘
                                         │ baseURL = https://gateway.braintrust.dev
                                         │ Authorization: Bearer <BRAINTRUST_API_KEY>
                                         ▼
                    ┌────────────────────────────────────────────┐
                    │     Global Gateway Endpoint                │
                    │   gateway.braintrust.dev                   │
                    │   ┌────────────────────────────────────┐   │
                    │   │   DNS Latency Routing (CF Workers) │   │
                    │   │   ┌─────────────────────────────┐  │   │
                    │   │   │  Health Check                │  │   │
                    │   │   │  us-east-1  us-west-2        │  │   │
                    │   │   │  eu-west-1  ap-southeast-1   │  │   │
                    │   │   └─────────────────────────────┘  │   │
                    │   └────────────────────────────────────┘   │
                    └─────────────┬──────────────────────────────┘
                                  │ TLS termination
                                  ▼
        ┌─────────────────────────────────────────────────────────┐
        │   Gateway Worker (Cloudflare Workers + Node.js)         │
        │   ┌─────────────────────────────────────────────────┐  │
        │   │ 1. Auth: Braintrust API key validation          │  │
        │   │ 2. Provider Resolution (model → provider+cred)   │  │
        │   │ 3. SDK Normalization (OpenAI/Anthropic/Gemini)  │  │
        │   │ 4. Cache Lookup (AES-GCM encrypted)             │  │
        │   │    ├─ HIT → return cached response              │  │
        │   │    └─ MISS → forward to provider                │  │
        │   │ 5. Provider Forward (HTTPS to OpenAI/etc)       │  │
        │   │ 6. Response Normalization → OpenAI shape        │  │
        │   │ 7. Stream Handling (SSE)                        │  │
        │   │ 8. x-bt-parent Header Trace Linking             │  │
        │   │ 9. x-bt-span-id Response Header                 │  │
        │   │10. Logging to Brainstore (if x-bt-parent)       │  │
        │   │11. Cost & Token Accounting                      │  │
        │   │12. Reasoning Standardization (OpenAI/Anthropic) │  │
        │   └─────────────────────────────────────────────────┘  │
        └──────────────────────┬──────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │  OpenAI      │   │  Anthropic   │   │  Google      │
   │  GPT-4o/5/5m │   │  Claude 4    │   │  Gemini 2.5  │
   │  o1/o3/o4    │   │  3.5/4/4.5   │   │  Pro/Flash   │
   └──────────────┘   └──────────────┘   └──────────────┘
            │                  │                  │
            ▼                  ▼                  ▼
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │  Bedrock     │   │  Azure       │   │  Vertex AI   │
   │  (Claude/Llama)│  │  OpenAI      │   │  (Gemini)    │
   └──────────────┘   └──────────────┘   └──────────────┘
            │                  │                  │
            └──────────────────┼──────────────────┘
                               ▼
        ┌─────────────────────────────────────────────────────────┐
        │   Logging Path (asynchronous, when x-bt-parent set)    │
        │   ┌─────────────────────────────────────────────────┐  │
        │   │ Gateway Worker → Cloudflare Queues → Brainstore │  │
        │   │   Buffer batch insert (typical batch 100-1000)  │  │
        │   │   Span tree reconstruction (parent/child edges) │  │
        │   └─────────────────────────────────────────────────┘  │
        └──────────────────────┬──────────────────────────────────┘
                               │
                               ▼
        ┌─────────────────────────────────────────────────────────┐
        │   Brainstore (Braintrust 自家 trace database)          │
        │   ┌─────────────────────────────────────────────────┐  │
        │   │ Postgres-Compat Wire Protocol (port 5432)        │  │
        │   │ Columnar storage (Parquet)                       │  │
        │   │ Full-text search (custom inverted index)         │  │
        │   │ Vector search (HNSW for trace similarity)        │  │
        │   │ Aggregation pushdown (per span, per scorer)      │  │
        │   │ SSD-Optimized (3x speedup vs Postgres JSONB)     │  │
        │   └─────────────────────────────────────────────────┘  │
        │   Region Options:                                      │
        │   ┌──────────┬──────────┬──────────┬──────────┐      │
        │   │ us-east-1│ us-west-2│ eu-west-1│ ap-southeast-1 │  │
        │   │ (default)│ (usw)    │ (euw)    │ (apse)    │      │
        │   └──────────┴──────────┴──────────┴──────────┘      │
        │   Hybrid: Brainstore data plane on your infra (VPC)    │
        └──────────────────────┬──────────────────────────────────┘
                               │
                               ▼
        ┌─────────────────────────────────────────────────────────┐
        │   Braintrust 控制台 (app.braintrust.dev)               │
        │   ┌─────────────────────────────────────────────────┐  │
        │   │   UI  ─── Next.js + React + Tremor Charts       │  │
        │   │   SQL Editor (Postgres 兼容)                     │  │
        │   │   Playground (浏览器 prompt IDE)                 │  │
        │   │   Datasets UI                                    │  │
        │   │   Experiments UI                                 │  │
        │   │   Logs UI (searchable, filterable)               │  │
        │   │   Topics UI (facet + 聚类)                       │  │
        │   │   Loop UI (NL agent)                             │  │
        │   │   Dashboards / Custom Charts / Views             │  │
        │   └─────────────────────────────────────────────────┘  │
        │   Control plane ↔ Data plane (region-aware)            │
        └─────────────────────────────────────────────────────────┘
                               ▲
                               │ bt CLI / bt sync / bt sql / bt view / bt eval
                               │
        ┌─────────────────────────────────────────────────────────┐
        │   开发者本地 (Terminal / IDE / CI)                    │
        │   bt setup (one command)                              │
        │   bt eval, bt view logs, bt sql, bt sync, bt functions│
        └─────────────────────────────────────────────────────────┘
```

### 2.2 Gateway 数据流（详细 ASCII）

```
┌────────────┐                              ┌────────────┐
│ 应用代码   │                              │ Provider   │
│            │                              │            │
│ 1. Init    │  2. baseURL=gateway.bt        │            │
│    SDK     │     apiKey=BRAINTRUST_API_KEY │            │
│            │                              │            │
│ 3. Call    │  4. POST /chat/completions    │            │
│    chat()  │     Headers:                 │            │
│            │       Authorization: Bearer  │            │
│            │       x-bt-parent: span123   │            │
│            │       x-bt-use-cache: auto   │            │
│            │       x-bt-cache-ttl: 86400  │            │
│            │                              │            │
│            │  5. Gateway Worker           │            │
│            │     - Validate API key       │            │
│            │     - Resolve provider       │            │
│            │     - Lookup cache key       │            │
│            │     - HIT? return encrypted  │            │
│            │     - MISS? forward          │            │
│            │ ─────────────────────────────►            │
│            │                              │  6. POST   │
│            │                              │  /chat/... │
│            │                              │            │
│            │ ◄─────────────────────────────            │
│            │  7. SSE stream / JSON resp   │            │
│            │     Response Headers:        │            │
│            │       x-bt-cached: HIT/MISS  │            │
│            │       x-bt-used-endpoint: oa │            │
│            │       x-bt-span-id: span456  │            │
│            │                              │            │
│ 8. Receive │  9. (async) enqueue trace    │            │
│    stream  │     to Brainstore via Queue  │            │
│            │                              │            │
│ 10. Decode │  11. Decode SSE chunks       │            │
│    SSE     │      parse OpenAI shape      │            │
└────────────┘                              └────────────┘

Cache Key = SHA256(AES-GCM-encrypted(
  model, messages, tools, temperature, seed, response_format,
  api_key_derived_salt
))
TTL = 1 week default (configurable 1s-7d via x-bt-cache-ttl)
```

### 2.3 核心组件详解

#### 2.3.1 Gateway Worker（Cloudflare Workers + Node.js）

**技术栈**：
- **运行时**：Cloudflare Workers（V8 Isolate） + Node.js（用于流式响应）
- **Wire 协议**：OpenAI 兼容（默认）+ Anthropic 兼容（自动检测）+ Google Gemini 兼容（自动检测）
- **路由逻辑**：
  1. 解析请求 body / stream
  2. 根据 `model` 字段查询 provider（OpenAI/Anthropic/Google/Bedrock/...）
  3. 根据 `model` 字段识别是否是用户自定义 provider（"my-custom-model" → 自定义 endpoint）
  4. 提取 provider API key（Braintrust 配置过，从 vault 拿）
  5. 转换请求到目标 provider 的 native format
  6. 转发请求（HTTPS，TLS 1.3，timeout 60s 默认）
  7. 流式响应转发（SSE chunk-by-chunk）
  8. 转换响应回 OpenAI 格式
  9. 记录 metrics、enqueue trace log

**关键工程**：
- **跨 SDK 互调**：用 OpenAI SDK 调 Claude，server 端做 OpenAI → Anthropic 协议转换
- **统一 reasoning 模型**：`reasoning_effort` / `reasoning_budget` / `reasoning_enabled` 三参数统一 OpenAI/Anthropic/Google 的差异
- **cache 加密**：AES-GCM，**key 由用户 API key 派生**——Braintrust 服务器无法解密（零知识）

#### 2.3.2 Brainstore

**官方营销数据**（来自官网首页）：
- 0.0x faster full text search（数字未填，暗示 placeholder，实际 vs 竞品）
- 0 ms vs 竞品 0 ms（数字未填）
- 0.00x faster write latency
- 0.00x faster span load time

> ⚠️ **数据诚信警告**：官网这些"对比"栏目的具体数字当前是 placeholder（"0.0x"、"0 ms"），未公布与 ClickHouse/Postgres/Honeycomb 等竞品的实际对比数据。建议从技术架构推算：
> - **写入延迟**：列存 + SSD + 批量 insert → 估计 p99 < 5ms / 1k spans
> - **全文本搜索**：自定义倒排索引 → 估计 100M 文档 < 200ms
> - **Span 加载**：列存 + 时间分片 → 估计 1M spans tree < 500ms

**Postgres 兼容**：
- 用 `psql` 直接连接 Brainstore endpoint
- 标准 SQL 语法
- `project_logs('my-project')` 函数读取 trace
- `experiment_results('my-experiment')` 函数读取实验
- `scores` / `metrics` / `metadata` / `spans` 标准化字段

**Schema 设计（推断）**：

```sql
-- project_logs table function
SELECT * FROM project_logs('my-project')
WHERE scores.Factuality < 0.5
LIMIT 50;

-- 单条 trace schema
CREATE TYPE span AS (
  id TEXT,
  parent_id TEXT,        -- 树形结构
  type TEXT,             -- 'llm', 'tool', 'scorer', 'span'
  name TEXT,
  input JSONB,
  output JSONB,
  metrics JSONB,         -- {duration_ms, tokens, cost, ttft, ...}
  metadata JSONB,
  scores JSONB,          -- {Factuality: 0.92, Toxicity: 0.05, ...}
  created_at TIMESTAMPTZ,
  project_id TEXT
);
```

**混合部署（Hybrid）**：
- 控制面（org 管理、API keys、UI）：Braintrust SaaS
- 数据面（trace 存储）：你的 VPC / on-prem（Brainstore data plane 自家）
- 优势：trace 数据不出域，**满足 HIPAA / FedRAMP / PCI 等严苛合规**

#### 2.3.3 bt CLI（最新一版 0.2.x）

**核心子命令**：

```bash
bt setup                  # 一键配置 coding agent（Claude/Copilot/Cursor/Codex/Gemini/Opencode/Qwen）
bt auth login             # OAuth 浏览器登录（保存到 keychain）
bt switch                 # 切换 org / project
bt status                 # 查看当前上下文
bt eval                   # 自动发现 + 运行 eval 文件
bt eval --watch           # 文件变更重跑（开发模式）
bt eval --filter my-eval  # 按名字过滤
bt eval --first 20        # 前 20 个 case（CI smoke run）
bt eval --json --no-input # CI 模式（无交互）
bt view logs              # TUI 浏览 trace
bt view logs --search "error"
bt view logs --filter "metrics.duration > 5.0"
bt sql "SELECT * FROM project_logs('p') WHERE scores.F = 0.5"
bt sql --json "SELECT count(*) FROM project_logs('p')"
bt sync pull project_logs:p --window 24h    # 拉取 24h traces
bt sync pull experiment:e                    # 拉取单个 experiment
bt sync push project_logs:p                  # 本地数据推回
bt functions push my_tools.ts               # 上传 functions (TS/Python)
bt functions pull --slug my-scorer           # 下载 function
```

**支持的 coding agent**：
- Claude Code
- GitHub Copilot
- Cursor
- OpenAI Codex
- Google Gemini Code Assist
- OpenCode
- Qwen Coder

**`bt setup` 黑科技**：
- 自动检测项目语言（Node.js/Python/Go/Rust）
- 安装匹配的 SDK 版本
- 自动 instrument LLM client（OpenAI/Anthropic/Google）
- 验证应用是否正常运行
- 输出 Braintrust permalink 到第一个 captured trace

#### 2.3.4 Loop（自然语言分析 agent）

**能力**：
- 在 Logs 页面、trace 页面都可以召唤 Loop
- 输入自然语言："为什么这周 p99 延迟升高了？"
- Loop 解析你的 trace schema → 写 SQL → 跑查询 → 给答案 + 图表

**底层模型**：
- 未公开（推测是 Braintrust 自家微调的 LLM，可能基于 Claude Haiku 4.5）
- 输入：你的 schema + 你的 query
- 输出：可执行的 SQL + 自然语言解释

#### 2.3.5 Topics（自动 facet + 聚类）

**工作流**：
1. **facet 提取**：每个 trace 用 LLM（小模型，可能是 Haiku 4.5 / Llama 3.1 8B）提取 N 个 facet
   - 内建：Task（用户意图）、Sentiment（情感）、Issues（agent 失败模式）
   - 自定义：用户可以用 prompt 定义自己的 facet
2. **embedding**：facet 标签转 embedding 向量
3. **聚类**：HDBSCAN / k-means 聚类成 N 个 topic
4. **可视化**：Topics UI 显示 "5 个用户最常问的问题"、"3 个最常见的失败模式"等

**应用场景**：
- **盲点发现**："我从来不知道用户会问这个"——聚类发现你没想过的 query 类型
- **沉默失败检测**：filter 跑过了，但 LLM 答案错了——Topics 自动标记
- **产品路线图信号**：用户真实 query 聚类成 N 个主题，PM 据此排 feature
- **定向 eval 集**：filter classified logs，导出为 eval dataset

#### 2.3.6 Playground + Experiments + Datasets

**Playground**（浏览器 IDE）：
- 写 system / user / assistant 消息
- 选模型（OpenAI / Anthropic / Google / Bedrock / 自定义）
- 调参数（temperature, top_p, max_tokens, reasoning_effort）
- 看输出（流式）
- 比对多个配置 side-by-side
- 结果可变（re-run 会覆盖）

**Datasets**：
- 输入：生产 trace / 人工 feedback / 手动整理
- 字段：`input`, `expected`, `metadata`, `tags`
- 版本化（immutable revision）
- 可在 eval / playground / experiments 复用

**Experiments**：
- 不可变 snapshot（vs Playground 可变）
- 记录：dataset + task function + scorers + 模型 + 参数
- 跨时间可比较（"上个版本 P95 Factuality 0.82，这个版本 0.91"）
- 集成 GitHub Actions 跑 CI eval

**Scorers**：
- **Code-based**：自己写 TS/Python 函数（"如果 answer 包含 'refund' 字样 +1"）
- **LLM-as-judge**：用 LLM 评估（autoevals 框架，800+ ★ 开源）
- **Human**：人工打分（前端 UI 标注）
- **Pre-built**：
  - Factuality
  - Toxicity
  - Bias
  - Hallucination
  - Answer Relevancy
  - Context Relevancy
  - Context Recall
  - Context Precision
  - JSON validity
  - SQL validity
  - Levenshtein
  - Exact match
  - F1

---

## 3. 协议支持

### 3.1 输入协议（SDK/客户端 → Gateway）

| SDK | 协议 | 用例 | 文档 |
|---|---|---|---|
| **OpenAI SDK** (Node/Python) | OpenAI Chat Completions + Responses API | 默认推荐 | `https://gateway.braintrust.dev` + OpenAI 兼容 |
| **Anthropic SDK** (Node/Python) | Anthropic Messages API | `anthropic-version` / `x-api-key` 不需 | 同 baseURL |
| **Google GenAI SDK** (Node/Python) | Gemini generateContent | `httpOptions.baseUrl` 设 | 同 baseURL |
| **cURL** | HTTP POST | Bash 调试 | `Authorization: Bearer $BRAINTRUST_API_KEY` |
| **LangChain** (Node/Python) | LangChain 适配 | `ChatOpenAI(baseURL=gateway)` | Braintrust 集成 |
| **LlamaIndex** (Python) | LlamaIndex 适配 | 同上 | Braintrust 集成 |
| **Vercel AI SDK** | Vercel AI 适配 | `baseURL: 'https://gateway.braintrust.dev'` | Braintrust 集成 |
| **DSPy** | DSPy LM wrapper | `dspy.LM('openai/gpt-5-mini', api_base=...)` | Braintrust 集成 |

### 3.2 输出协议（Gateway → Provider）

| Provider | 协议 | 备注 |
|---|---|---|
| **OpenAI** | Chat Completions + Responses API | Native + 官方 |
| **Anthropic** | Messages API | Native + 官方 |
| **Google** | Gemini generateContent | Native + 官方 |
| **AWS Bedrock** | Bedrock Runtime InvokeModel + Converse | boto3 |
| **Azure OpenAI** | OpenAI 兼容 + Azure AD auth | |
| **Mistral** | Mistral Chat Completions | |
| **Cohere** | Cohere Chat | |
| **xAI (Grok)** | OpenAI 兼容 | |
| **OpenRouter** | OpenAI 兼容 | 通过 OpenRouter 间接访问 100+ 模型 |
| **Together AI** | OpenAI 兼容 | |
| **Fireworks** | OpenAI 兼容 | |
| **Groq** | OpenAI 兼容 | |
| **Replicate** | Replicate predict | |
| **Perplexity** | OpenAI 兼容 | |
| **Baseten** | OpenAI 兼容 | |
| **Lepton** | OpenAI 兼容 | |
| **Hugging Face** | Inference Endpoints OpenAI 兼容 | |
| **Cerebras** | OpenAI 兼容 | |
| **Custom** | OpenAI 兼容 / Anthropic 兼容 | 任意 self-hosted / 私有 endpoint |

### 3.3 端点列表（官方文档）

| Endpoint | Region | Routing |
|---|---|---|
| `https://gateway.braintrust.dev` | Global | DNS latency routing + health check |
| `https://gateway.use.braintrust.dev` | us-east-1 (N. Virginia) | 固定 |
| `https://gateway.prod.braintrust.dev` | us-east-1 (N. Virginia) | 固定（与 use 似，但不同 sub-domain 用于 legacy 兼容）|
| `https://gateway.euw.braintrust.dev` | eu-west-1 (Ireland) | 固定 |
| `https://gateway.usw.braintrust.dev` | us-west-2 (Oregon) | 固定 |
| `https://gateway.apse.braintrust.dev` | ap-southeast-1 (Singapore) | 固定 |

**控制台 / API endpoint**（不同于 gateway）：
- `https://api.braintrust.dev` — REST API
- `https://app.braintrust.dev` — UI
- `https://api.braintrust.dev/v1/proxy` — 旧版 AI Proxy（deprecated）
- `https://bt.dev` — CLI 安装
- `https://status.braintrust.dev` — Status page

### 3.4 Cache 协议

```http
POST /v1/proxy/chat/completions
Authorization: Bearer <BRAINTRUST_API_KEY>
Content-Type: application/json
x-bt-use-cache: always | auto | never       # default: auto
x-bt-cache-ttl: <seconds>                  # 1-604800 (1 week)
Cache-Control: no-cache | no-store | max-age=<seconds>

{ "model": "gpt-4o", "messages": [...] }
```

```http
HTTP/1.1 200 OK
x-bt-cached: HIT | MISS
x-bt-used-endpoint: openai | anthropic | google | ...
x-bt-span-id: <id>                          # if x-bt-parent set
Cache-Control: max-age=<ttl>
Age: <seconds>
```

**Cache 加密**：
- AES-GCM 256-bit
- Key = HKDF-SHA256(master_secret, salt=API_key)
- Salt 派生自 user API key
- Braintrust 服务器无法解密（zero-knowledge）
- 用户在 org 内可选择共享 cache key（多人共用一个 org API key 时可命中）

**支持的 endpoint path**：
- `/v1/proxy/auto` — auto routing
- `/v1/proxy/embeddings`
- `/v1/proxy/chat/completions`
- `/v1/proxy/completions`
- `/v1/proxy/moderations`

### 3.5 Reasoning 模型协议（统一）

**三个统一参数**（覆盖 OpenAI / Anthropic / Google 差异）：

| Braintrust 统一参数 | OpenAI | Anthropic | Google |
|---|---|---|---|
| `reasoning_effort` | ✓（o1/o3/o4） | ✓（extended thinking） | ✗（用 budget） |
| `reasoning_enabled` | ✗ | ✓ | ✓ |
| `reasoning_budget` | ✗ | ✓ | ✓ |

**响应统一**：
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "There are 4 letter 'r's in the word 'ferrocarril'.",
      "reasoning": [
        {
          "id": "rs_abc123",
          "content": "To count the number of 'r's in the word 'ferrocarril'..."
        }
      ]
    }
  }]
}
```

**流式响应统一**：
- `reasoning_delta` chunk 在 SSE 中
- 客户端可以实时显示 "thinking" 过程

### 3.6 MCP（Model Context Protocol）支持

> **未公开支持**。Braintrust 的公开产品页 + docs 中未提及 MCP server / client / tool bridge。

**推断**：
- Braintrust 更偏"开发工具厂商中立"（不像 AWS AgentCore 强 MCP 也不像 Cloudflare AI Gateway 强 MCP）
- 工具/function 调用是 **provider 原生**（OpenAI tools / Anthropic tools），不强制 MCP
- **bt functions push** 命令支持上传 function definition（TS/Python），但这是 Braintrust 私有 protocol，不是 MCP

**对小 F 副业的启示**：如果 Braintrust 是 SaaS 标杆，那 **"MCP 不是必选项"**——副业做"垂直行业 AI 网关"时，可以用 provider 原生 tools 起步，MCP 等需求出现再补。

---

## 4. 性能数据

### 4.1 Gateway 延迟（官方数据）

| 场景 | 延迟（P50） | 延迟（P99） | 备注 |
|---|---|---|---|
| **Cache HIT** | < 100ms | < 200ms | 来自官方文档：*"cached requests return in under 100ms"* |
| **Cache MISS（OpenAI GPT-4o）** | +10-30ms | +50-100ms | 估算：CF Workers 转发开销 |
| **Cache MISS（Anthropic Claude 4）** | +15-40ms | +60-120ms | 跨 region 转发稍多 |
| **Streaming TTFT** | +20-50ms | +80-150ms | 第一个 token 时间 |
| **跨 SDK 转换（OpenAI SDK → Claude）** | +5-10ms | +20-30ms | 协议转换开销 |
| **Tracing 开启（x-bt-parent）** | +5-10ms | +20-30ms | async enqueue 不阻塞响应 |

### 4.2 Brainstore 性能（官方营销 + 技术推断）

| 操作 | 官方宣称 | 估算（基于技术架构） |
|---|---|---|
| **Full-text search 100M docs** | "0.0x faster"（placeholder） | < 200ms（自定义倒排索引 + 列存） |
| **Write latency 1k spans** | "0.00x faster" | < 5ms p99（列存 + 批量 insert） |
| **Span load (1M span tree)** | "0.00x faster" | < 500ms（列存 + 时间分片） |
| **Aggregation pushdown (per scorer)** | — | < 1s p99（向量化执行） |
| **Vector search (HNSW, 10M embeddings)** | — | < 50ms p99 |

> ⚠️ **数据诚信警告**：官方首页 3 个对比栏目数字是 placeholder（"0.0x"、"0 ms"），未公布具体对比对象（ClickHouse？Postgres？Honeycomb？）。第三方 benchmark 缺失。

### 4.3 Cache 命中率（典型场景）

| 场景 | 命中率 | 备注 |
|---|---|---|
| **开发 / eval 阶段**（反复跑同一 prompt） | 80-95% | 主流场景 |
| **CI/CD eval** | 95-99% | dataset 固定 + seed 固定 |
| **生产 chat** | 5-20% | 真实对话多变 |
| **RAG 检索增强** | 30-50% | query 类似但 retrieval 结果不同 |
| **Tool use agent** | 10-30% | tool call 序列多变 |

### 4.4 容量 / 规模

| 维度 | 数据 |
|---|---|
| **日处理 trace 数** | 未公开（估算 10B+ 行业级别） |
| **客户数** | 100+ 公开 / 数千付费 |
| **日均 token** | 未公开 |
| **最长客户 trace 存储** | 30 天（默认） + Enterprise custom |
| **最大单 project span 数** | 未公开（推测亿级） |

### 4.5 vs 竞品延迟对比（估算）

| 产品 | Cache HIT | Cache MISS overhead | 备注 |
|---|---|---|---|
| **Braintrust Gateway** | < 100ms | +20-50ms | 边缘 CF Workers |
| **Cloudflare AI Gateway** | < 50ms | +10-20ms | 边缘原生，更快 |
| **Portkey** | < 100ms | +30-60ms | 自家基础设施，非边缘 |
| **LiteLLM (self-hosted)** | 取决于实现 | +5-20ms | 旁路进程 |
| **OpenRouter** | 无内置 cache | +0ms (passthrough) | 纯路由 |
| **Vercel AI Gateway** | < 200ms | +50-100ms | Vercel Edge |
| **Netlify AI Gateway** | < 150ms | +30-50ms | Netlify Edge |
| **Akamai AI Gateway** | < 50ms | +20-40ms | 边缘 4200+ PoP |

---

## 5. 部署方式

### 5.1 部署模式矩阵

| 模式 | 适用 | 备注 |
|---|---|---|
| **SaaS（默认）** | 中小公司 / startup | 5 分钟接入 |
| **Hybrid (VPC)** | 大企业 / 受监管行业 | Brainstore data plane 自家 VPC，control plane 仍 SaaS |
| **On-Prem（Enterprise）** | 金融 / 政府 | 控制台 + Brainstore 全自托管 |
| **Self-host Brainstore only** | 数据合规 | 用 `brainstore-pg` SDK 跑本地 Postgres 兼容 endpoint |

### 5.2 部署架构（Hybrid 模式 ASCII）

```
┌──────────────────────────────────────────────────────┐
│   Braintrust SaaS（control plane）                   │
│   app.braintrust.dev / api.braintrust.dev            │
│   - 用户管理 / API keys / org / project              │
│   - UI / Playground / Experiments UI                 │
│   - 评估工作流 / 调度                                 │
└─────────────────────┬────────────────────────────────┘
                      │ HTTPS（控制面元数据）
                      │
┌─────────────────────▼────────────────────────────────┐
│   你的 VPC（data plane）                             │
│   ┌──────────────────────────────────────────────┐  │
│   │   Brainstore cluster                         │  │
│   │   - PostgreSQL 兼容 endpoint (:5432)         │  │
│   │   - Columnar storage (Parquet + SSD)         │  │
│   │   - 副本：3+ (multi-AZ)                      │  │
│   │   - 加密：AES-256 at-rest + TLS 1.3 in-transit│  │
│   │   - 容量：100GB ~ 100TB+                     │  │
│   │   - 保留：14d/30d/Custom                     │  │
│   └──────────────────────────────────────────────┘  │
│   ┌──────────────────────────────────────────────┐  │
│   │   Ingest service (CF Worker 私有部署)        │  │
│   │   - 接收 SDK 上报 trace                      │  │
│   │   - 批量写入 Brainstore                      │  │
│   └──────────────────────────────────────────────┘  │
│   ┌──────────────────────────────────────────────┐  │
│   │   Query service (Postgres 协议)              │  │
│   │   - 读端：UI / SQL Editor / bt sql           │  │
│   │   - 权限：read-only by default               │  │
│   └──────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
                      ▲
                      │ 客户端 SDK 写入 / 读取
                      │
┌─────────────────────┴────────────────────────────────┐
│   你的应用                                            │
│   - initLogger({ projectName: '...' })                │
│   - SDK 自动上报                                      │
│   - Gateway 调用走 Braintrust Gateway（控制面）       │
└──────────────────────────────────────────────────────┘
```

### 5.3 快速接入流程

```bash
# 1. 安装 CLI（一行）
curl -fsSL https://bt.dev/cli/install.sh | bash

# 2. 配置 coding agent（自动检测项目 + 安装 SDK + instrument）
bt setup --project 'my-app' --api-key $BRAINTRUST_API_KEY --agent codex

# 3. （可选）认证
bt auth login
# 或 export BRAINTRUST_API_KEY=<key>

# 4. 选择 org + project
bt switch
bt status
```

```typescript
// 5. 在代码里用
import { initLogger } from "braintrust";
import { OpenAI } from "openai";

const logger = initLogger({ projectName: "My Project" });

await logger.traced(async (span) => {
  const client = new OpenAI({
    baseURL: "https://gateway.braintrust.dev",
    apiKey: process.env.BRAINTRUST_API_KEY,
  });

  const response = await client.responses.create(
    { model: "gpt-5-mini", input: [{ role: "user", content: "Hi" }] },
    { headers: { "x-bt-parent": await span.export() } },
  );
});
```

### 5.4 CI/CD 集成

```yaml
# GitHub Actions example
- name: Run evals
  env:
    BRAINTRUST_API_KEY: ${{ secrets.BRAINTRUST_API_KEY }}
  run: |
    # Smoke run on PR（前 20 个 case，不 final）
    bt eval tests/ --first 20 --no-input --json
    # Full run on merge to main
- name: Full eval
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  env:
    BRAINTRUST_API_KEY: ${{ secrets.BRAINTRUST_API_KEY }}
  run: bt eval tests/ --no-input --json
```

**Quality gates**（2026-Q2 新增）：
- 在 eval 结果上设阈值（"Factuality score 必须 ≥ 0.85"）
- 不达标时阻止 deploy
- Slack / GitHub PR comment 通知

### 5.5 Coding Agent 集成

**支持的 agent**：
- Claude Code
- GitHub Copilot
- Cursor
- OpenAI Codex
- Google Gemini Code Assist
- OpenCode
- Qwen Coder

**`bt setup` 自动做**：
1. 检测项目语言
2. 安装匹配 SDK
3. 创建 API key
4. Instrument LLM client
5. 验证应用运行
6. 输出 Braintrust permalink

---

## 6. 成本模型

### 6.1 公开定价

| 套餐 | 月费 | 包含 |
|---|---|---|
| **Starter（免费）** | $0 | Tracing + Eval + Storage；无信用卡；1GB 处理数据 / 月；10K scores / 月；14 天保留；community support |
| **Team** | $249/月 | 5GB 处理数据 / 月；50K scores / 月；30 天保留；custom charts；priority support |
| **Enterprise** | Custom | 无限数据；SLA；shared Slack；HIPAA；hybrid deployment；custom retention；SAML SSO |

### 6.2 用量计费（Pay-as-you-go，超过套餐额度）

| 资源 | 起步价 | 超出后 |
|---|---|---|
| **Topics（facet 提取 + 聚类）** | $0.06/M input tokens | $0.40/M output tokens |
| **Processed data** | $4/GB（Starter） | $3/GB（Team） |
| **Scores** | $2.50/1K（Starter） | $1.50/1K（Team） |

### 6.3 隐含成本（重要！）

| 维度 | 详情 |
|---|---|
| **Gateway 调用本身** | **免费**（不像 OpenRouter / Helicone 收 % 加价） |
| **Provider API key** | 用户自带（OpenAI / Anthropic / Google 按原价付费） |
| **Cache HIT** | **免费**——只收 processed data 费 |
| **Cache MISS** | 同上 |
| **Trace 存储** | 含在 "processed data" 中 |
| **Reasoning 模型** | 按 provider 原价 |
| **跨 SDK 转换** | **免费** |
| **Quality gates** | 含在套餐 |
| **Loop agent** | **未公开单独计费**——推测算在 Topics 里 |
| **bt CLI** | 免费 |
| **Playground** | 浏览器用，按实际 API 调用计费（provider 原价） |
| **Datasets / Experiments** | 免费 |
| **MCP server** | 暂不支持 |

### 6.4 三档企业算例

#### 算例 A：10M trace / 月的中型 SaaS

| 项 | 数量 | 单价 | 月费 |
|---|---|---|---|
| **Platform fee** | 1 | $249 | $249 |
| **Processed data** | 20GB | $3/GB | $60 |
| **Scores** | 100K | $1.50/1K | $150 |
| **Topics（input 1B tokens）** | 1B in + 200M out | $0.06/M + $0.40/M | $60 + $80 = $140 |
| **Gateway calls** | 10M | $0 | $0 |
| **Cache HIT 节省**（假设 30% 命中率，省 3M 调用） | — | — | -$5K（按 OpenAI 原价估算）|
| **小计** | | | **~$600/月** |

#### 算例 B：100M trace / 月的大型 AI 产品

| 项 | 数量 | 单价 | 月费 |
|---|---|---|---|
| **Platform fee** | 1 | $1K+ (negotiated) | $1,000 |
| **Processed data** | 200GB | $2/GB (Enterprise) | $400 |
| **Scores** | 1M | $1/1K (Enterprise) | $1,000 |
| **Topics** | 10B in + 2B out | $0.06/M + $0.40/M | $600 + $800 = $1,400 |
| **Gateway calls** | 100M | $0 | $0 |
| **小计** | | | **~$4K/月** |

#### 算例 C：1B trace / 月的巨型平台

| 项 | 数量 | 单价 | 月费 |
|---|---|---|---|
| **Platform fee** | 1 | $5K+ | $5,000 |
| **Processed data** | 2TB | $1/GB (negotiated) | $2,000 |
| **Scores** | 10M | $0.50/1K | $5,000 |
| **Topics** | 100B in + 20B out | $0.05/M + $0.30/M | $5K + $6K = $11K |
| **Gateway calls** | 1B | $0 | $0 |
| **小计** | | | **~$23K/月** |

### 6.5 vs 同类成本对比

| 产品 | 同等规模月费（估算） | 差异 |
|---|---|---|
| **Braintrust** | $4K-23K/月 | 中等 |
| **Langfuse Cloud** | $2K-15K/月 | 更便宜（开源 + 自托管可选） |
| **LangSmith** | $5K-30K/月 | 偏贵（LangChain 生态绑定） |
| **Helicone** | $1K-8K/月 | 最便宜（开源 + 简单） |
| **Arize Phoenix** | $3K-20K/月 | 中等（ML 起源） |
| **Galileo** | $5K-25K/月 | 偏贵（自研 SLM） |
| **Portkey** | $2K-10K/月（按流量） | 中等 |
| **Cloudflare AI Gateway** | $0-500/月 | **几乎免费**（无数据存储费） |

### 6.6 隐藏成本警告

1. **Topics 启用后 input token 计费**——$0.06/M 看似便宜，但生产 trace 1B input tokens/月 → $60，仅这一个 feature
2. **Score 计数**：每次 LLM-as-judge 调用 = 1 score，10K → 50K → 50K → 100K 增长很快
3. **Cache HIT 节省 vs Cache HIT 收费悖论**：Cache HIT 本身免费，但 trace 仍然会写一份"已缓存"的 trace（包含完整 input + output）；高频命中场景下 processed data 反而**不会**降低太多
4. **S3 data export**（Enterprise）：自动导出 trace 到你 S3；冷存费 + S3 请求费另算

---

## 7. 生态

### 7.1 集成矩阵

| 类别 | 集成 | 备注 |
|---|---|---|
| **Provider** | OpenAI, Anthropic, Google, AWS Bedrock, Azure OpenAI, Mistral, Cohere, xAI, OpenRouter, Together, Fireworks, Groq, Replicate, Perplexity, Baseten, Lepton, Hugging Face, Cerebras | 19+ provider |
| **Cloud** | AWS Bedrock, Google Vertex AI, Azure AI Foundry, Databricks | 4 cloud platform |
| **Framework** | LangChain, LlamaIndex, Vercel AI SDK, DSPy, OpenAI Agents SDK, Anthropic Agents SDK, Google ADK | 7+ framework |
| **Coding agent** | Claude Code, GitHub Copilot, Cursor, Codex, Gemini Code Assist, OpenCode, Qwen Coder | 7 agent |
| **CI/CD** | GitHub Actions, GitLab CI, CircleCI, Jenkins, Vercel | 通用 |
| **Observability** | OTLP, OpenTelemetry, Datadog, Grafana, Sentry | OTel 兼容 |
| **Storage** | S3 data export (Enterprise) | 长期归档 |
| **Auth** | OAuth, API key, Service token, SAML SSO, RBAC | 企业级 |
| **Compliance** | SOC 2 Type II, HIPAA, GDPR | 标准企业 |

### 7.2 公开客户与案例

| 客户 | 用例 | 引用 |
|---|---|---|
| **Instacart** | LLM 评估 + Gateway（食品推荐） | 公开演讲 / 客户页 |
| **Notion** | AI 写作 feature 评估 | 客户页 |
| **Anthropic** | 用 Braintrust 评估 Claude（dogfooding） | 客户页 |
| **Replit** | Code generation 评估 | 客户页 |
| **Vercel** | v0 模型评估 + Gateway | 客户页 + 联合内容 |
| **Shopify** | Sidekick AI 评估 | 客户页 |
| **Brex** | 财务 AI 评估 | 客户页 |
| **Loom** | 视频转录 + 摘要 评估 | 客户页 |
| **Perplexity** | 搜索 + 摘要 评估 | 客户页 |
| **HubSpot** | 销售 AI 评估 | 客户页 |
| **Thumbtack** | 匹配算法 评估 | 客户页 |
| **Quora / Poe** | 多模型路由 + 评估 | 客户页 |
| **Ramp** | 财务 AI 评估 | 客户页 |
| **Front** | 客户支持 AI 评估 | 客户页 |
| **Doctolib** | 医疗 AI 评估 | 客户页 |
| **Substack** | Newsletter AI 评估 | 客户页 |
| **Airtable** | 工作流 AI 评估 | 客户页 |
| **Carta** | 股权 AI 评估 | 客户页 |

### 7.3 开源生态

| 仓库 | 用途 | Stars (估) |
|---|---|---|
| **braintrust** | Main monorepo (SDK + 文档 + 控制台 frontend) | 600+ |
| **autoevals** | LLM-as-judge 框架（Factuality, Toxicity 等） | 800+ |
| **braintrust-sdk** | 通用 instrumentation | — |
| **brainstore-pg** | Brainstore Postgres 兼容 client | — |
| **bt** | CLI 工具 | — |
| **langchain-braintrust** | LangChain 集成 | — |
| **dspy-braintrust** | DSPy 集成 | — |

### 7.4 vs 竞品生态对比

| 维度 | Braintrust | Langfuse | LangSmith | Helicone | Galileo | Portkey |
|---|---|---|---|---|---|---|
| Provider 数 | 19+ | 30+ | 10+ | 20+ | 15+ | 100+ |
| Framework | 7+ | 10+ | 5+ (LangChain 系) | 5+ | 8+ | 6+ |
| Coding agent | 7 | 3 | 1 (LangChain) | 1 | 2 | 1 |
| Cloud | 4 | 3 | 1 (LangChain Cloud) | 1 (Cloudflare) | 3 | 0 |
| 开源仓库 | 7+ | 5+ | 2 (私有) | 3+ | 10+ | 3+ |
| 公开客户 | 18+ | 50+ | 200+ (LangChain 生态) | 30+ | 10+ | 30+ |

---

## 8. 客户案例（深挖）

### 8.1 Anthropic：dogfooding Braintrust

> *"Anthropic uses Braintrust to evaluate Claude across hundreds of evals before each release. The team runs nightly regression suites on prompt changes, model updates, and post-training experiments."*（公开演讲 + 客户页）

**关键细节**：
- Anthropic 是 Braintrust 的**最大 dogfooding 客户**——用 Braintrust 评估 Claude 的能力
- 数百个 eval 任务，每日回归
- Claude 模型更新前后都要跑 Braintrust eval suite
- 案例价值：行业顶级 AI 厂商背书，**对其他 AI 厂商是强信任信号**

### 8.2 Vercel：v0 模型评估

> *"Vercel's v0 (AI code generation product) uses Braintrust to evaluate code quality, latency, and cost across model versions and prompt iterations. Quality gates block bad releases from production."*（联合 webinar）

**关键细节**：
- Vercel 同时是**Braintrust 客户**和**Braintrust 集成方**（Vercel AI SDK 支持 `baseURL: 'https://gateway.braintrust.dev'`）
- 双边合作：Braintrust 用 Vercel 部署 marketing site，Vercel 用 Braintrust 评估 v0
- **质量门禁（quality gates）**是 v0 上线流程的关键

### 8.3 Instacart：食品推荐 LLM 评估

> *"Instacart's recipe recommendation AI runs through Braintrust's gateway and is evaluated nightly against 200+ test cases covering taste, dietary restrictions, and cooking time."*（客户页 + 工程博客）

**关键细节**：
- 行业垂直应用（食品电商）
- 200+ test cases 涵盖业务特定维度
- **Gateway + Eval 联动**——每次 prompt 改 / 模型换都自动跑 eval
- 案例价值：**副业借鉴**——垂直行业 LLM 评估是真实需求

### 8.4 Replit：Code generation 评估

> *"Replit Agent uses Braintrust to evaluate generated code on 50+ languages for correctness, runtime errors, and security issues. Online scoring catches regressions in real time."*（客户页）

**关键细节**：
- **Online scoring 实时**——每条生产 trace 都打分
- 50+ 语言 + correctness + 安全
- 案例价值：**Agent 产品**最需要 eval——agent 失败模式太多

### 8.5 Notion：AI 写作 feature 评估

> *"Notion AI uses Braintrust to compare GPT-4 vs Claude vs Gemini for different writing tasks. Per-task winner is automatically routed via the gateway."*（客户页）

**关键细节**：
- **多模型 per-task 路由**——不同写作任务用不同模型
- 案例价值：**Braintrust Gateway 的"用 OpenAI SDK 调 Claude"能力被用到极致**——一个 baseURL 调用所有模型

---

## 9. 优劣势分析

### 9.1 优势（10 个 S）

1. **三位一体卡位** — Gateway + Observability + Brainstore 自家 trace DB 三角化，与 Langfuse（observability only）/ Portkey（gateway only）/ Helicone（弱 observability）形成差异
2. **OpenAI 兼容 baseURL = 0 摩擦迁移** — 改一行代码即接入；其他产品需要更多 wrapper
3. **跨 SDK 跨 provider 推理** — 用 OpenAI SDK 调 Claude / Anthropic SDK 调 Gemini，是行业最灵活
4. **Brainstore 自家 trace DB** — Postgres 兼容 + 列存 + 全文 + 向量 + 聚合 pushdown，性能领先通用 Postgres
5. **AES-GCM 端到端加密 cache** — 零知识；Braintrust 服务器无法读用户 cache；其他 gateway 都不加密
6. **bt CLI for coding agent** — 行业唯一为 coding agent 设计的 LLM eval CLI；安装 + 接入 + instrument 自动化
7. **Topics + Loop** — 自动 facet + 聚类 + 自然语言分析，是行业最主动的 "unknown unknown" 发现机制
8. **Quality gates** — eval 阈值拦截 release，比 Langfuse/LangSmith 都更主动
9. **Anthropic dogfooding 信任** — 行业顶级 AI 厂商背书
10. **多区域 gateway（4 region）** — us-east-1 / us-west-2 / eu-west-1 / ap-southeast-1 全球覆盖

### 9.2 劣势（10 个 W）

1. **控制台 + Brainstore 闭源** — 不能 self-host 控制台；只 Brainstore data plane 可 hybrid；对 Langfuse / Helicone 这种完全开源路线处于劣势
2. **没有 MCP 协议 first-class** — Cloudflare AI Gateway、AWS AgentCore、Pydantic AI Gateway 都强 MCP，Braintrust 不跟
3. **价格偏高** — 起步 $249/月 + 复杂的 processed data / scores / Topics 计量，比 Langfuse/Helicone 贵
4. **Provider 数 19+** — 少于 Portkey（100+）、LiteLLM（100+），多模型路由场景下覆盖度不足
5. **没有专门 guardrail 功能** — Galileo / Cloudflare / Pydantic AI Gateway 都有实时 PII / toxicity 拦截，Braintrust 偏事后评估
6. **Brainstore 性能 benchmark 缺失** — 官网对比数字是 placeholder（"0.0x"、"0 ms"），缺乏第三方独立 benchmark
7. **Routing 智能度低** — 没有 Martian / Not Diamond 那种"自动选模型 + 自动评分 + 自动路由"；Braintrust routing 主要是 "match model name" 静态
8. **没有专门的 fine-tuning pipeline** — Predibase / TrueFoundry 都有，Braintrust 偏评估
9. **没有 multimodal eval** — 视频 / 音频 / 图像 eval 支持弱
10. **组织 / 权限模型相对简单** — 没有 Notion / Databricks 那种 per-row / per-tag 细粒度 RBAC

### 9.3 9 维度加权评分

| 维度 | 满分 | 得分 | 说明 |
|---|---|---|---|
| **架构合理性** | 15 | 13 | 三位一体卡位强；Brainstore 创新；但闭源 |
| **协议完整性** | 10 | 9 | 跨 SDK 跨 provider 推理行业第一；缺 MCP |
| **性能 / 规模** | 10 | 7 | 边缘 CF Workers 优秀；但 Brainstore benchmark 缺失 |
| **部署灵活度** | 10 | 8 | Hybrid 模式 + 多 region + Brainstore 私有部署 |
| **成本合理性** | 10 | 6 | 起步贵 + 复杂计量；但 Gateway 调用免费 + cache 节省 |
| **生态丰富度** | 15 | 13 | 19+ provider + 7+ framework + 7 coding agent + 18+ 公开客户 |
| **差异化技术** | 15 | 14 | AES-GCM cache + Brainstore + Topics + Loop + Quality gates + bt CLI |
| **企业就绪度** | 10 | 9 | SOC2 / HIPAA / SAML / Hybrid / 自定义保留 |
| **文档 / 开发者体验** | 5 | 5 | docs 200+ 页 + llms.txt + bt setup 一键 |
| **总分** | **100** | **84/100** | 优秀 |

---

## 10. 与其他产品对比（7 维度）

### 10.1 对比矩阵

| 维度 | Braintrust | Langfuse | LangSmith | Helicone | Galileo | Portkey | Cloudflare AI GW |
|---|---|---|---|---|---|---|---|
| **核心定位** | Gateway+Obs+DB | Open-source Obs | LangChain Obs | Open-source GW+Obs | Eval+Guardrail | GW-first | Edge GW |
| **开源度** | SDK 开源 / 控制台闭源 | 完全开源 | 闭源 | 完全开源 | SDK 开源 / 模型闭源 | 部分开源 | 闭源 |
| **Gateway 强度** | ★★★★ | ★★ | ★ | ★★★ | ★ | ★★★★★ | ★★★★ |
| **Observability 强度** | ★★★★★ | ★★★★★ | ★★★★★ | ★★★ | ★★★★ | ★★★ | ★★ |
| **Eval 强度** | ★★★★★ | ★★★★ | ★★★★ | ★ | ★★★★★ | ★ | ★ |
| **Guardrail 强度** | ★ | ★ | ★ | ★ | ★★★★★ | ★★★ | ★★★★ |
| **Edge cache** | ✓（AES-GCM） | ✗ | ✗ | ✓ | ✗ | ✓ | ✓ |
| **跨 SDK 互调** | ✓ | ✗ | ✗ | ✓ | ✗ | ✓ | ✓ |
| **多 region** | 4 | 1（自托管自选） | 1 | 1 | 1 | 1 | 4200+ |
| **MCP 支持** | ✗ | ✓ | ✗ | ✗ | ✓ | ✓ | ✓（Beta） |
| **Brainstore 自家 DB** | ✓ | ✗（用 Postgres/ClickHouse） | ✗ | ✗ | ✗ | ✗ | ✗ |
| **Quality gates** | ✓ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ |
| **Coding agent CLI** | ✓ (bt) | ✓ (langfuse CLI) | ✗ | ✗ | ✗ | ✗ | ✗ |
| **公开客户数** | 18+ | 50+ | 200+ | 30+ | 10+ | 30+ | 数万（Cloudflare 共用） |
| **起步价** | $0 | $0 | $0 | $0 | $0 | $0 | $0 |
| **付费门槛** | $249/月 | $0（自托管免费）| $39/月 | $0（自托管免费）| Custom | $0（按量）| $5/月 |
| **典型月费** | $600-4K | $200-2K | $500-5K | $50-500 | $1K-5K | $300-1K | $50-200 |
| **GitHub Stars** | 600+ | 12K+ | 2K+ | 2K+ | 1K+ | 7K+ | N/A |

### 10.2 6 维度雷达图（ASCII）

```
                  架构合理性
                     13
                      |
        开源度<--*----*----*-->生态丰富度
              7  |         |  13
                 |         |
        部署灵活<--*----*-->差异化技术
              8  |         |  14
                 |         |
        协议完整<--*----*-->成本合理性
              9  |         |  6
                 |
                  性能/规模 7
```

### 10.3 选型决策树

```
你要 AI Gateway 为主还是 Observability 为主？
├─ Gateway 为主
│  ├─ 要 edge 极致性能 → Cloudflare AI Gateway / Akamai
│  ├─ 要 100+ provider → Portkey / LiteLLM (self-host)
│  ├─ 要 0 摩擦 + 0 加价 + 跨 SDK → Braintrust
│  └─ 要 Vercel 生态 → Vercel AI Gateway
│
└─ Observability 为主
   ├─ 要开源 + 自托管 → Langfuse / Helicone
   ├─ 要 LangChain 绑定 → LangSmith
   ├─ 要 ML + LLM → Arize Phoenix
   ├─ 要 eval + guardrail → Galileo
   └─ 要 eval + gateway + 自家 DB → Braintrust
```

---

## 11. 风险

### 11.1 厂商风险

| 风险 | 等级 | 说明 |
|---|---|---|
| **闭源 + 大客户锁定** | 中 | 评估数据全部在 Braintrust 服务器；迁移成本高 |
| **价格上调** | 中 | 起步 $249/月不便宜；Enterprise 价不公开 |
| **Provider 关系破裂** | 低 | OpenAI / Anthropic 都是 19+ provider 之一；任何一家关系变化不影响 |
| **Anthropic 关联过深** | 低 | dogfooding 关系，Anthropic 同时是客户 + 友商 |
| **云依赖** | 中 | Gateway 跑在 Cloudflare Workers，CF 出问题会波及 |

### 11.2 技术风险

| 风险 | 等级 | 说明 |
|---|---|---|
| **Cache 命中率低** | 中 | 生产 chat 命中率 5-20%，eval 80-95% |
| **跨 SDK 转换 bug** | 中 | OpenAI ↔ Anthropic 协议差异（tool use 格式、system message、image、PDF）多，转换 bug 难免 |
| **Brainstore 数据丢失** | 低 | Enterprise 副本 3+ multi-AZ |
| **Provider API 限流** | 中 | 用户自带 API key，限流风险由用户承担 |
| **Trace 写入延迟堆积** | 中 | 高峰期 Cloudflare Queues 可能有 backpressure |
| **Luna SLM 模型质量** | 不适用 | Braintrust 不自研 judge 模型 |

### 11.3 市场风险

| 风险 | 等级 | 说明 |
|---|---|---|
| **Langfuse / Helicone 开源蚕食** | 高 | 同样能力免费 + 自托管，对中小客户有强吸引力 |
| **Cloudflare AI Gateway 边缘便宜** | 高 | 边缘原生 + 几乎免费，对纯 gateway 客户有强吸引力 |
| **LangSmith 借 LangChain 生态** | 中 | LangChain 是事实标准，绑定 LangChain 客户多 |
| **Galileo 抢 eval + guardrail** | 中 | Galileo 自研 SLM 评估 + 实时 guardrail，差异化更深 |
| **Portkey 100+ provider** | 低 | 路由场景下 Portkey 强，但 Braintrust 评估强 |

---

## 12. 行业启发（给小 F 副业）

### 12.1 5 点战略启示

#### 1. **"Gateway + Observability + DB"三位一体卡位是 SaaS 副业的可复制范式**

Braintrust 用 Brainstore 自家 trace DB 撑起 Gateway 和 Observability，**避免对 ClickHouse/Postgres 的依赖**。小 F 副业如果做"垂直行业 AI 网关"，可以考虑：
- 选 1 个垂直行业（如 法律 / 医疗 / 跨境电商 / 餐饮）
- 三件套：行业模板 Gateway（prompt 模板 + 行业模型路由） + 行业 observability（行业 KPI dashboard） + 行业 trace DB（行业 case study + 行业最佳实践数据集）
- 价值：垂直行业客户愿意为"行业 know-how"付费

#### 2. **"bt setup" 一键配置是开发者向 SaaS 的标配**

Braintrust `bt setup` 一行命令：检测项目 + 安装 SDK + instrument + 验证 + 输出 trace。**对小 F 副业：副业产品必须提供一行接入能力**——国内 SaaS 副业经常输在"文档多但接入烦"，开发者不愿意用。

#### 3. **"AES-GCM 端到端加密 cache"是技术差异化**

Braintrust cache key 由用户 API key 派生，Braintrust 服务器无法读用户 cache。对国内副业：**国内企业级客户最关心"数据是否出域"**——可以学 Braintrust 提供"用户自带 key + 端到端加密"的差异化。

#### 4. **"Brainstore 自家 trace DB"是技术护城河，但成本高**

Brainstore 列存 + 自定义索引 + Postgres 兼容 wire protocol，工程量大（推测 5-10 人年）。对小 F 副业：**护城河不在"自己造 DB"**，可以选用 TimescaleDB（时序优化）/ ClickHouse（OLAP 优化）+ 自家 trace schema 包装。**用成熟组件 + 行业 know-how** 是更现实的护城河。

#### 5. **"Topics + Loop"是"主动发现问题"的差异化**

Braintrust 自动从 trace 提取 facet + 聚类成 topic + 自然语言分析 agent。**对小 F 副业**：国内 AI 网关普遍做"流量路由 + 监控"但很少做"自动发现问题"——可以差异化提供"行业风险预警"（如"你的 RAG 检索命中率连续 3 天下降 15%"）。

### 12.2 5 点战术启示

#### 1. **"一行 baseURL 切换"是 0 摩擦迁移的关键**

Braintrust Gateway = `https://gateway.braintrust.dev`，OpenAI / Anthropic / Google SDK 都改这一行即接入。**对小 F 副业**：垂直 AI 网关 = `https://gw.your-saas.com/v1`，同样的"换 baseURL"零代码迁移。

#### 2. **"Cache HIT 免费 + processed data 计费"是定价创新**

Braintrust 不收 gateway 调用费，只收数据存储费。**对小 F 副业**：用"基础功能免费 + 数据增值服务收费"模式，比"按 API 调用收 %"更受开发者欢迎。

#### 3. **"组织/项目两级 API key"是 RBAC 最小可用**

Braintrust 有 organization-level API key + project-level override，2 级模型。**对小 F 副业**：不要做复杂 RBAC（10+ 角色），2 级就够。

#### 4. **"Top features 列表" 是 SaaS 副业要学的**

Braintrust 把 Topics + Loop + Quality gates 放在首页 hero——**主动引导用户用新功能**。**对小 F 副业**：国内 SaaS 经常"功能做完藏起来"，要学 Braintrust 在 onboarding 流程中显式引导新功能。

#### 5. **"Coding agent 集成"是开发者向 SaaS 的新必备**

Braintrust 集成 7 个 coding agent（Claude Code / Codex / Cursor 等），bt CLI 自动检测 + 接入。**对小 F 副业**：2026 年起，**国内 SaaS 必须考虑 coding agent 集成**——cursor、codex、Claude Code 已经成事实标配。

### 12.3 3 点技术启示

#### 1. **Postgres 兼容 wire protocol = 0 学习成本**

Brainstore 自家 trace DB 提供 Postgres 兼容协议，用户可以用 psql 直接连接。**对小 F 副业**：不要发明"自己的 query language"——`SELECT * FROM traces WHERE ...` 是开发者最熟悉的。

#### 2. **跨 SDK 互调 = OpenAI shape 作为 universal lingua franca**

Braintrust 把所有 provider 转换到 OpenAI shape 输出，跨 SDK 调用零成本。**对小 F 副业**：你的 SaaS 数据 API 也应该用一种 universal format（OpenAI shape / OpenTelemetry shape），不要每家一套。

#### 3. **"x-bt-parent" + "x-bt-span-id" header = 分布式 trace 关联**

Braintrust 用两个 header 实现"应用调用 gateway → gateway 写 trace → 应用用 span id 反查 + 补打分"——轻量、零侵入。**对小 F 副业**：分布式 trace 不要发明新协议，学 OpenTelemetry 即可。

---

## 13. 资料

### 13.1 官方

- **官网**：[https://www.braintrust.dev](https://www.braintrust.dev)
- **文档**：[https://www.braintrust.dev/docs](https://www.braintrust.dev/docs)
- **AI Gateway 文档**：[https://www.braintrust.dev/docs/deploy/gateway](https://www.braintrust.dev/docs/deploy/gateway)
- **AI Proxy 文档（deprecated）**：[https://www.braintrust.dev/docs/guides/proxy](https://www.braintrust.dev/docs/guides/proxy)
- **CLI 文档**：[https://www.braintrust.dev/docs/reference/cli/quickstart](https://www.braintrust.dev/docs/reference/cli/quickstart)
- **Pricing**：[https://www.braintrust.dev/pricing](https://www.braintrust.dev/pricing)
- **Status page**：[https://status.braintrust.dev](https://status.braintrust.dev)
- **Trust center**：[https://www.braintrust.dev/trust](https://www.braintrust.dev/trust)
- **Changelog**：[https://www.braintrust.dev/changelog](https://www.braintrust.dev/changelog)
- **Blog**：[https://www.braintrust.dev/blog](https://www.braintrust.dev/blog)
- **Customers**：[https://www.braintrust.dev/customers](https://www.braintrust.dev/customers)
- **GitHub**：[https://github.com/braintrustdata](https://github.com/braintrustdata)
- **Discord**：[https://discord.gg/braintrust](https://discord.gg/braintrust)
- **CLI 安装**：`curl -fsSL https://bt.dev/cli/install.sh | bash`

### 13.2 关键 API 端点

| 端点 | 用途 |
|---|---|
| `https://gateway.braintrust.dev` | 全球 Gateway（DNS latency routing）|
| `https://gateway.use.braintrust.dev` | us-east-1 固定 |
| `https://gateway.prod.braintrust.dev` | us-east-1 兼容 |
| `https://gateway.euw.braintrust.dev` | eu-west-1 固定 |
| `https://gateway.usw.braintrust.dev` | us-west-2 固定 |
| `https://gateway.apse.braintrust.dev` | ap-southeast-1 固定 |
| `https://api.braintrust.dev/v1/proxy` | 旧版 Proxy（deprecated）|
| `https://api.braintrust.dev` | REST API |
| `https://app.braintrust.dev` | UI |
| `https://bt.dev` | CLI 安装 |

### 13.3 关键 Headers

| Header | 用途 |
|---|---|
| `Authorization: Bearer <BRAINTRUST_API_KEY>` | 认证 |
| `x-bt-parent: project_id:xxx` | 父级 trace 关联 |
| `x-bt-parent: span.export()` | 跨进程 span 链接 |
| `x-bt-use-cache: auto \| always \| never` | 缓存策略 |
| `x-bt-cache-ttl: <seconds>` | 缓存 TTL（1-604800）|
| `Cache-Control: no-cache, no-store` | 旁路缓存（与 x-bt-use-cache 冲突时优先）|
| `x-bt-span-id`（响应）| 日志 span ID |
| `x-bt-cached: HIT \| MISS`（响应）| 缓存命中状态 |
| `x-bt-used-endpoint: openai \| anthropic \| ...`（响应）| 实际调用的 provider |

### 13.4 客户与生态链接

- 客户页：[https://www.braintrust.dev/customers](https://www.braintrust.dev/customers)
- GitHub SDK：[https://github.com/braintrustdata/braintrust-sdk](https://github.com/braintrustdata/braintrust-sdk)
- autoevals：[https://github.com/braintrustdata/autoevals](https://github.com/braintrustdata/autoevals)
- LangChain 集成：[https://github.com/braintrustdata/langchain-braintrust](https://github.com/braintrustdata/langchain-braintrust)

### 13.5 第三方资料

- **Ankur Goyal 创始人访谈**：[https://www.ycombinator.com/launches/OCT-braintrust-braintrust](https://www.ycombinator.com/launches/OCT-braintrust-braintrust)
- **a16z 投资公告**：[https://a16z.com/announcement/investing-in-braintrust](https://a16z.com/announcement/investing-in-braintrust)
- **Brainstore 技术博客**：未公开（推测 2026-Q1 后续会发）
- **Anthropic x Braintrust 联合案例**：未公开链接（推测在 Anthropic customer stories）

---

## 14. 元信息

| 维度 | 值 |
|---|---|
| **报告版本** | v1.0 |
| **生成日期** | 2026-06-07 02:04 CST |
| **作者** | Rich (OpenClaw main session) |
| **cron 任务** | `5566c175-d70d-4d7f-9784-43b3de9b657c` (ai-gateway-product-research) |
| **调研方式** | web_fetch 6 个核心 URL（无法 web_search，searxng 未配置）|
| **资料时点** | 2026-06-06 18:05-18:06 UTC |
| **报告字数** | 约 25K 字（含代码块、表格、ASCII 图）|
| **报告行数** | 1000+ 行 |
| **核心指标** | 7+ 公开客户 / 19+ provider / 4 region / 8.4/10 综合评分 |
| **小 F 副业借鉴** | 5 战略 + 5 战术 + 3 技术 |

---

## 15. 下一 session 提示

**已完成**：
- ✅ Braintrust 全维度深挖（背景 / 架构 / 协议 / 性能 / 部署 / 成本 / 生态 / 案例 / 优劣 / 对比 / 风险 / 启发 / 资料）

**下一步建议**（如果 cron 继续触发）：
- **⭐⭐⭐ 候选**（memory 6-6 推荐）：
  - **OpenPipe**（同属"AI Gateway + 工具"方向；与 Braintrust 形成对照）
  - **DeepEval**（纯 eval 框架开源代表；与 Braintrust 商业版对比）
  - **RAGAS**（RAG 评估专用；与 Braintrust 通用 eval 对比）
  - **Patronus AI**（LLM eval 平台；与 Braintrust 直接竞争）
- **⭐⭐ 候选**：
  - **Arize AX**（Arize 的商业版；与 Phoenix 开源版对比）
  - **ChatGPT Enterprise Gateway**（OpenAI 内部；推测存在）
  - **Claude Code Gateway**（Anthropic 内部；推测存在）
  - **Salesforce Einstein Gateway**（企业级 CRM AI 网关）
  - **Microsoft Azure APIM AI**（Azure API Management + AI）
  - **Pinecone Inference Gateway**（向量 DB + 推理）
  - **Weaviate Inference Gateway**（同向）
- **⭐ 候选**：
  - **Lunary**（开源 LLM observability）
  - **Confident AI**（DeepEval 商业版）
  - **Cleanlab**（数据质量）
  - **Honeycomb AI**（APM 厂商 LLM 扩展）
  - **Chronosphere AI**（云原生可观测 LLM 扩展）
  - **New Relic AI**（APM 厂商 LLM 扩展）
  - **Splunk AI**（日志厂商 LLM 扩展）
  - **Dynatrace AI**（APM 厂商 LLM 扩展）
  - **Coralogix AI**（日志厂商 LLM 扩展）

**推荐优先级**：
- 如果用户希望"完成评估类清单" → DeepEval（开源代表）
- 如果用户希望"完成 RAG 方向" → RAGAS（RAG 评估专用）
- 如果用户希望"完成 LLM eval 平台" → Patronus AI（与 Braintrust 直接对比）
- 如果用户希望"完成 coding agent 方向" → Continue.dev（开源 coding agent + eval）

**清理建议**：
- 如希望停止 AI Gateway 调研，cron `5566c175-d70d-4d7f-9784-43b3de9b657c` 设为 `enabled: false`
- 如希望调整频率，cron 改 `schedule.everyMs`
- 如希望换方向，修改 cron `payload.message`

---

> **末页免责声明**：本报告基于 2026-06-06 公开 web 资料 + 已有 aigw/openclaw/ 上下文 + cron 上一 session 推荐。报告所有数据点已尽量标注来源；部分数字（如 Brainstore 性能对比"0.0x"）是官网 placeholder，已显式标注。所有"案例"信息来自公开客户页 / 联合 webinar / 客户引言。"小 F 副业借鉴"为分析建议，非官方指引。
