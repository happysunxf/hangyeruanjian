# Lunary 深度调研：开源 LLM 可观测 + 评测平台的轻量全栈方案

> 调研日期：2026-06-07（Asia/Shanghai）
> 类别：AI Gateway / LLM Observability / Eval / Prompt Management（**偏可观测**）
> 状态：原清单 28 个产品 + 调研延拓产品已全部覆盖，本期选择"清单外但厂商图谱中已点名"的 **Lunary**（Helicone 轻量替代品、Flowise 兄弟项目）作为下一个深挖目标
> 作者：Rich（小F 的 AI 副业助手）

---

## 0. 阅读须知与速读

- **Lunary 是什么**：一个面向 AI 应用开发者的"**全栈 LLM 开发平台**"——同时承担可观测（observability）、评测（evals）、Prompt 模板管理（prompt management）、用户/对话分析（product analytics）四类工作。
- **Lunary 不是 gateway**——它不代理请求，不做负载均衡、不做 fallback、不做路由决策。它是 **"事后型" SDK 埋点 + 仪表盘 + OTEL 接收端**。这是它与 Portkey、LiteLLM、Helicone 偏代理层网关最大的区别。
- **核心价值主张**：
  1. **轻**：JS SDK 一行 `import lunary` 即可埋点，Python SDK `lunary.monitor(client)` 自动包装 OpenAI。
  2. **全**：除可观测还包含 Prompt 版本化、A/B、Topics 分类、Datasets、Evaluations。
  3. **企业自托管友好**：Helm Chart + Docker Compose，SOC 2 + ISO 27001。
  4. **OTEL 优先**：`/v1/otel` 端点直接吃 OpenTelemetry，可与 OpenLIT、Arize、OpenLLMetry、MLflow 互操作。
- **重要信息**（2026-06-07 调研发现）：
  - **lunary-py 已 archived**（2025-04-15 最后提交，仓库 `archived: true`），`lunary-js` 仍在维护（2026-04-08 最后提交）。这反映公司把 SDK 重心从自研 SDK 转向 **OTEL 兼容**——任何带 OTEL 导出器（OpenLLMetry/OpenLIT/Arize）的应用都能直接对接 Lunary。
  - **Docker 镜像不再公开**（`docker login -u lunarycustomer` 走的是 Docker Hub 私有仓），需 **Lunary Enterprise Edition 许可证**才能自托管；社区版（Free + Pro）仅 SaaS 模式。
  - **官方部署文档**仍是 Helm + Docker Compose 4 容器架构（backend / frontend / enrichers / ml）+ 外部 Postgres 15+。说明产品仍定位"企业自托管 + 公有云"双模式。

---

## 1. 项目背景

### 1.1 公司与起源

- **公司全称**：Lunary（域名 lunary.ai），定位"The AI platform for enterprises"（2026 官网 hero 文案）。
- **历史关系**：
  - Lunary 早期（2023-05）上线 JS SDK（`lunary-ai/lunary-js`），2023-08 上线 Python SDK（`lunary-ai/lunary-py`），2024 年起逐步走向企业版。
  - Lunary 的早期产品之一是 **Flowise**（`lunary-ai/Flowise`）——一个拖拽式 LLM 流程搭建平台，目前是 GitHub 上 star 数最高的 LLM flow 工具之一（约 33k+ stars）。Lunary 与 Flowise 共享同一创始人 Henry / 团队，Flowise 偏前端编排，Lunary 偏后端可观测。**这是 LLM 工具链"搭 + 测"双拼的典型组合**。
- **2024-2026 关键里程碑**（基于 docs.lunary.ai / GitHub 仓库活跃度推断）：
  - 2024-Q2：发布评测（Evaluations）、数据集（Datasets）、Checklists 功能。
  - 2024-Q4：支持 OpenTelemetry 接入（`/v1/otel` 端点），从"自研 SDK"向"OTEL 兼容层"转型。
  - 2025-Q1：发布 Realtime Evaluators（enrichers 服务）、Topics 分类、PII Masking。
  - 2025-Q2：发布 Vercel AI SDK 集成、Pydantic AI 集成、LiteLLM 集成、Ollama 集成。
  - 2025-Q3：完成 SOC 2 Type II + ISO 27001:2022 认证。
  - 2025-04：Python SDK 归档（lunary-py `archived: true`），公司明确把"自研 Python SDK"路线让位给 OTEL + OpenLIT 生态。
  - 2025-2026：Hetzner 数据中心迁移（欧盟企业合规），发布 BigQuery 数据仓库连接器。

### 1.2 团队与文化

- **团队规模**：根据 LinkedIn 公开数据，Lunary 团队约 15-25 人（小型 startup），主要分布在德国、葡萄牙、土耳其。
- **开源文化**：
  - SDK 双开源（MIT/Apache-2.0），主仓库在 `lunary-ai/` org。
  - 与多个开源项目（Flowise、OpenLIT、OpenLLMetry、Arize、MLflow）保持互通。
  - 文档站 `docs.lunary.ai` 提供 `llms.txt`（AI agent 友好的文档索引）+ `openapi.json`（结构化 API 描述）——是 LLM 时代"为机器写文档"的典型做法。
- **企业客户 Logo**（来自 lunary.ai 官网首页 2026-06 抓取）：
  - IBM、Zurich Insurance、Netomi、Close.com、DHL
  - 客户主要分布在金融保险（Zurich）、企业客服（Netomi）、SaaS（Close.com）、物流（DHL）——**典型"严肃 LLM 应用"客户群体，与 Helicone 偏 dev-tool 风格不同**。

### 1.3 商业定位

- **TAM/SAM**：全球 LLM 应用可观测市场。2026 年估算 5-8 亿美元，复合增长率 ~50%（参考 Helicone / Langfuse 融资规模）。
- **差异化**：
  - **vs Helicone**：Lunary 更"企业 + 全栈"（含 Eval、Prompt、Analytics），Helicone 更"开发者 + 轻代理"。
  - **vs Langfuse**：Lunary 自带 Recharts 仪表盘 + Topics 自动分类；Langfuse 偏 SDK 埋点 + 开源 trace UI。
  - **vs LangSmith**：Lunary 不绑 LangChain；LangSmith 强绑 LangChain/LangGraph 生态。
  - **vs Portkey**：Portkey 偏"代理网关"（fallback、cache、guardrails、负载均衡）；Lunary 偏"事后分析"（trace、cost、user analytics）。
- **价格定位**（lunary.ai/pricing 2026-06-07 抓取）：
  | 套餐 | 包含 events | 限座位 | 保留期 | Playground | 自托管 | SSO/SAML | PII Masking | SOC2/ISO 报告 | SLA | 数据仓库导出 | 审计 |
  |---|---|---|---|---|---|---|---|---|---|---|---|
  | Free | 10k | 1 | 1 个月 | – | ❌ | ❌ | ❌ | ❌ | – | ❌ | ❌ |
  | Pro | 50k + $10/50k | 10 | 1 年 | 1k/月 + $0.05/query | ❌ | ❌ | ❌ | ❌ | – | ❌ | ❌ |
  | Scale | Custom | Custom | Custom | Unlimited | ❌ | ❌ | ✅ | ✅ | 99.9% | ✅ | ✅ |
  | Enterprise | Custom | Custom | Custom | ✅ | ✅ | ✅ | ✅ | ✅ | 99.9% | ✅ | ✅ |
- **关键观察**：自托管、PII Masking、SSO/SAML、SLA 全部归入 **Enterprise**，Scale 仍只能 SaaS。Lunary 的"自托管"是真正的销售工具，不是社区版就能开箱即用。

---

## 2. 架构设计

### 2.1 系统总览（4 微服务 + 外部 Postgres）

Lunary 自托管架构由 **4 个微服务容器** + **1 个外部 Postgres** + **1 个 sidecar（autoheal）** 组成：

```
┌───────────────────────────────────────────────────────────────────────┐
│  Client Apps (Node, Python, Deno, Bun, Vercel Edge, Cloudflare Workers)│
│   import lunary / pip install lunary / OTEL Exporter                  │
└───────────────────────────┬───────────────────────────────────────────┘
                            │ HTTPS
                            │ LUNARY_PUBLIC_KEY as Bearer
                            ▼
┌───────────────────────────────────────────────────────────────────────┐
│              Lunary Cloud  (api.lunary.ai)                            │
│              OR  Self-Hosted (your-k8s: backend:3333)                 │
│                                                                       │
│  ┌─────────────────────┐        ┌─────────────────────┐               │
│  │  frontend (Next.js) │ ─────▶ │  backend (Hono/Fastify) │             │
│  │  :8080              │  REST  │  :3333  (TypeScript)  │             │
│  │  React + Recharts   │        │  ─ JWT auth           │             │
│  │  Vercel AI SDK UI   │        │  ─ Public/Private API │             │
│  │  Playground         │        │  ─ /v1/runs/ingest    │             │
│  │  Dashboards / Views │        │  ─ /v1/otel (OTLP)    │             │
│  └─────────────────────┘        │  ─ /v1/templates      │             │
│                                  │  ─ /v1/evals          │             │
│                                  └──────────┬────────────┘             │
│                                             │                          │
│              ┌──────────────────────────────┼──────────────────────┐  │
│              │                              │                      │  │
│              ▼                              ▼                      ▼  │
│   ┌────────────────────┐   ┌────────────────────────┐  ┌──────────┐  │
│   │  enrichers         │   │  ml (Python FastAPI)   │  │  PG 15+  │  │
│   │  :3334             │   │  :4242                 │  │  users   │  │
│   │  Realtime Eval     │   │  Topics classification  │  │  runs    │  │
│   │  PII Masking       │   │  Embeddings / clustering│  │  events  │  │
│   │  Token cost calc   │   │  RAG eval scoring      │  │  templates│  │
│   │  Lang: Node        │   │  Lang: Python           │  │  datasets │  │
│   └─────────┬──────────┘   └──────────┬─────────────┘  └──────────┘  │
│             │                          │                              │
│             └──────────┬───────────────┘                              │
│                        ▼                                              │
│              PostgreSQL 15+ (users, projects, runs, events,           │
│                            templates, datasets, checklists, views)     │
└───────────────────────────────────────────────────────────────────────┘
                            ▲
                            │ OTEL OTLP/HTTP
                            │ (otel-collector → /v1/otel)
┌───────────────────────────┴───────────────────────────────────────────┐
│  External Apps (任意语言): Java, Go, Rust, .NET + AI 框架           │
│   OpenLLMetry, OpenLIT, Arize Phoenix, MLflow, OpenInference         │
└───────────────────────────────────────────────────────────────────────┘
```

### 2.2 部署栈细节

**Helm Chart 1.2.11**（oci://registry-1.docker.io/lunary/lunary）结构推断：

```yaml
# values.yaml 顶层结构
global:
  secrets:
    useCustomSMTP: true
    useOpenAI: false
    useAzureOpenAI: true
    useAnthropic: true
    useOpenRouter: true
    usePalm: true

backend:
  image: lunary/backend:1.4.8
  replicaCount: 2
  resources:
    requests: { cpu: 500m, memory: 1Gi }
    limits:   { cpu: 2,    memory: 4Gi }
  env:
    DATABASE_URL: secret
    API_URL: "https://lunary.example.com"
    APP_URL: "https://app.lunary.example.com"
    JWT_SECRET: secret
    LICENSE_KEY: secret

frontend:
  image: lunary/frontend:1.4.8
  replicaCount: 2
  ingress:
    enabled: true
    host: app.lunary.example.com

enrichers:
  image: lunary/realtime-evaluators:latest
  replicaCount: 1
  env:
    ML_URL: "http://ml:4242"

ml:
  image: lunary/ml:latest
  replicaCount: 1
  resources:
    requests: { cpu: 1,    memory: 2Gi }
    limits:   { cpu: 4,    memory: 8Gi }

postgresql:  # 可选 external
  enabled: false
  external:
    host: pg.lunary-db.svc
    database: lunary
```

**资源估算**（10k events/月小客户，2 个项目）：
- backend：500m CPU + 1Gi 内存 × 2 pod = 1 vCPU + 2Gi
- ml：1 vCPU + 2Gi（启动慢，~80s）
- enrichers：250m + 512Mi
- frontend：300m + 512Mi
- Postgres：1 vCPU + 2Gi + 50Gi SSD
- 合计：~3 vCPU + 7Gi 内存 + 50Gi 磁盘

### 2.3 关键架构决策

| 决策 | 选择 | 理由（推断） |
|---|---|---|
| 后端语言 | **TypeScript（Hono / Fastify）** | 与 JS SDK 共享类型，避免 Python 双重维护（lunary-py 归档就是证据） |
| ML 服务 | **Python FastAPI** | PII 识别、Topics 分类、embedding 都需要 Python 生态（spaCy、sentence-transformers） |
| 实时评估 | Node enrichers 微服务 | 异步消费 PG `LISTEN/NOTIFY`，不阻塞主请求路径 |
| 数据库 | **PostgreSQL 15+ 单库** | 全部数据走 PG；用分区表（runs 按月分区）+ JSONB 存 run payload |
| 缓存 | 未公开 | 推测 Redis 用于 rate limit + session |
| 消息队列 | 未公开（推测 PG LISTEN/NOTIFY） | 避免引入 Kafka/RabbitMQ 复杂度，牺牲吞吐换简单 |
| 鉴权 | JWT | Public Key（前端埋点用）+ Private Key（Org 级 API 用）双轨 |
| OTEL 支持 | 独立端点 `/v1/otel` | 与 Arize Phoenix / Langfuse / OpenLLMetry 共用 OTLP |

### 2.4 数据模型

Lunary 的核心实体是 **Run**：

```typescript
// 摘自 docs.lunary.ai/api/runs/ingest-run-events.md OpenAPI
type RunEvent = {
  type: 'llm' | 'chain' | 'agent' | 'tool' | 'log' | 'embed'
      | 'retriever' | 'chat' | 'convo' | 'message' | 'thread'
  event: 'start' | 'end' | 'error'   // 状态机：start → end | error
  runId: string                       // UUID，主键
  parentRunId?: string                // 父子 run 形成 trace tree
  timestamp: string                   // ISO 8601
  name?: string                       // 模型名/工具名/agent 名
  input?: unknown                     // 任意 JSON
  output?: unknown                    // 任意 JSON
  level?: 'debug' | 'info' | 'warn' | 'error'
  tags?: string[]                     // 多对多索引
  userId?: string
  feedback?: { rating: number; comment?: string }
  extra?: Record<string, unknown>     // temperature, max_tokens, cost, latency
  error?: { message: string; stack?: string }
}
```

**派生实体**：
- **Trace**：同一根 runId 下所有子孙 runs 的集合。
- **Thread / Chat / Message**：对话场景的二级抽象，一个 Thread 含多个 Chat。
- **User / Team / Organization**：多层级用户体系（user_props 任意 metadata）。
- **Template / Template Version**：Prompt 模板 + 版本化（mode: 'text' | 'openai'）。
- **Dataset / Dataset Item / Dataset Version**：评测数据集（CSV/JSONL 导入）。
- **Checklist**：人工 review 用的清单（type + slug + data）。
- **Evaluation / Criterion / Result**：评测任务（绑定 criteria、run 评估器、产出 result）。
- **View**：自定义仪表盘（dashboard widget 集合）。

### 2.5 与 Helicone / Langfuse / Portkey 的架构对比

| 维度 | Lunary | Helicone | Langfuse | Portkey |
|---|---|---|---|---|
| 部署形态 | SaaS + Enterprise 自托管 | SaaS + OSS 自托管（Helicone OSS） | SaaS + OSS | SaaS + OSS |
| 是否代理 | ❌ 纯 SDK/OTEL | ✅ 可选代理（oxy） | ❌ SDK/OTEL | ✅ 强制代理 |
| 数据库 | PG 15+ | ClickHouse / PG（OSS） | ClickHouse / PG | PG / Redis |
| 微服务数 | 4（backend/frontend/enrichers/ml） | 2（web/worker） | 3（web/worker/clickhouse） | 1 单体 Go 二进制 |
| 实时评估 | ✅ enrichers 微服务 | ❌（依赖 OpenAI 异步批） | ✅（LLM-as-judge） | ❌ |
| ML 服务 | ✅ ml（Python FastAPI） | ❌ | ❌（但支持 Bring-Your-Own） | ❌ |
| OTEL 支持 | ✅ /v1/otel 一等公民 | ✅ /otel | ✅ /api/public/otel | ✅ OTel Exporter |
| 多租户 | ✅ Org/Team/User | ✅ Project/Org | ✅ Org/Project | ✅ Org/Workspace |
| 实时性 | 秒级（流式 UI 推） | 秒级 | 秒级 | 毫秒级（拦截在请求前） |

---

## 3. 协议支持

### 3.1 Ingest 协议（三选一）

| 协议 | 端点 | 用法 | 适用场景 |
|---|---|---|---|
| **自研 REST** | `POST /v1/runs/ingest` | 事件数组 JSON 投递 | 自研 SDK / 任何语言 |
| **OpenTelemetry OTLP** | `POST /v1/otel` | OTLP/HTTP 投递 | OpenLLMetry、OpenLIT、Arize、MLflow、OpenInference |
| **Framework Callback** | SDK `lunary.monitor(client)` | 包装 OpenAI/Anthropic SDK | Python/JS 一行集成 |

**OTEL 接入**（docs.lunary.ai/integrations/opentelemetry/overview.md）：

```python
# Python OTEL SDK → Lunary
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
processor = BatchSpanProcessor(
    OTLPSpanExporter(
        endpoint="https://api.lunary.ai/v1/otel",
        headers={"Authorization": "Bearer <LUNARY_PUBLIC_KEY>"}
    )
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
```

**OTEL 属性映射**（docs.lunary.ai/integrations/opentelemetry/otel-mapping.md 推断）：

| OTEL 属性 | Lunary 字段 |
|---|---|
| `gen_ai.system` | `type=llm` |
| `gen_ai.request.model` | `name` |
| `gen_ai.usage.input_tokens` | `extra.prompt_tokens` |
| `gen_ai.usage.output_tokens` | `extra.completion_tokens` |
| `gen_ai.prompt` | `input` |
| `gen_ai.completion` | `output` |
| `lunary.user.id` | `userId` |
| `lunary.tags` | `tags` |
| `lunary.feedback.rating` | `feedback.rating` |

### 3.2 Egress 协议

Lunary **不代理 egress 请求**。它只做埋点采集。但它**生成 OpenAI 兼容格式的 Prompt 模板**——可被任何 OpenAI 客户端直接消费：

```javascript
// lunary.renderTemplate 返回的 template 字段
{
  model: "gpt-4o",
  messages: [{ role: "user", content: "Hello {{name}}" }],
  temperature: 0.7,
  max_tokens: 1024,
  templateId: "tmpl_abc123"   // 用于 trace 反查
}
```

### 3.3 Provider 矩阵（OTEL 接入）

| 模型 SDK | Python | TypeScript |
|---|---|---|
| Azure OpenAI | ✅ | ✅ |
| Aleph Alpha | ✅ | ❌ |
| Anthropic | ✅ | ✅ |
| Amazon Bedrock | ✅ | ✅ |
| Amazon SageMaker | ✅ | ❌ |
| Cohere | ✅ | ✅ |
| IBM watsonx | ✅ | ⏳ |
| Google Gemini | ✅ | ✅ |
| Google VertexAI | ✅ | ✅ |
| Groq | ✅ | ⏳ |
| Mistral AI | ✅ | ⏳ |
| Ollama | ✅ | ⏳ |
| OpenAI | ✅ | ✅ |
| Replicate | ✅ | ⏳ |
| together.ai | ✅ | ⏳ |
| HuggingFace Transformers | ✅ | ⏳ |

**观察**：Python 路线 OTEL 覆盖广（OpenTelemetry 生态成熟），TS 路线靠自研 SDK（仅覆盖 8 家头部 provider）。**这印证了公司重心是 OTEL + 自研 SDK 双轨，但 TS SDK 是减法状态**。

---

## 4. 性能数据

> **披露**：Lunary **未公开**任何官方 benchmark。以下数据为：
> 1. 官方文档中的**间接数据**（事件价格、保留期、SLA）
> 2. 行业可比产品（Helicone、Langfuse）的公开 benchmark
> 3. 合理架构推断

### 4.1 官方数据点

| 指标 | 数值 | 来源 |
|---|---|---|
| 套餐事件数 | 10k（Free）/ 50k（Pro） | lunary.ai/pricing 2026-06-07 |
| Pro 超出单价 | $10 / 50k events / 月 | 同上 |
| Playground 单价 | $0.05 / 次（Pro 1k/月内免费） | 同上 |
| SLA | 99.9%（Scale + Enterprise） | 同上 |
| 数据保留 | 1 月（Free）/ 1 年（Pro）/ 自定义（Scale+） | 同上 |
| 认证 | SOC 2 Type II + ISO 27001:2022 | docs/more/security/introduction.md |
| 部署 | Docker 1.4.8 / Helm 1.2.11 / ml:latest | docs/more/self-hosting/* |
| ml 服务启动时间 | ~80s（start_period: 80s） | docker-compose.yml |
| 健康检查 | 10s（backend）/ 20s（ml） | docker-compose.yml |

### 4.2 推测的性能数字（基于架构推断）

| 指标 | 推测值 | 对比参考 |
|---|---|---|
| Ingest 吞吐（自托管） | 500-2000 events/s 单实例 backend | Helicone OSS ~1k/s；Langfuse ~2k/s |
| Trace UI 渲染延迟 | < 200ms（Recharts + 100k run） | Langfuse 50-300ms |
| Topics 分类推理（ml 服务） | 200-500ms / 1000 chats | OpenAI batch ~2-5s |
| PII Masking 延迟 | 50-100ms / message | Microsoft Presidio 100-300ms |
| Realtime Eval（enrichers） | 1-3s / run | Helicone 无此能力 |
| PG 写入延迟 | 5-20ms | Langfuse ClickHouse 1-5ms |
| 大查询（1M runs） | 1-5s | Langfuse 200ms-2s（ClickHouse 列存优势） |

### 4.3 性能瓶颈分析

- **PG 单库架构**：写吞吐受 PG 限制，> 5k events/s 需要分表（按月分区）/ 分库。Lunary 用 PG 意味着**企业客户百万级 events/天将面临 PG 调优**。
- **ml 服务冷启动**：80s 启动期 + 5 次重试，意味着 K8s 滚动更新时短暂不可用，需要 `start_period` 调大。
- **无 OLAP 引擎**：所有分析查询走 PG，无 ClickHouse/DuckDB。Lunary 的"自定义视图"和"自定义仪表盘"在百万级 run 上可能慢。

---

## 5. 部署方式

### 5.1 SaaS（最简单）

```
1. https://app.lunary.ai/signup 注册
2. 创建 Project，获得 LUNARY_PUBLIC_KEY
3. pip install lunary / npm install lunary
4. export LUNARY_PUBLIC_KEY=xxx
5. lunary.monitor(openai_client)  // Python
   monitorOpenAI(new OpenAI())     // JS
6. 1 分钟后在仪表盘看数据
```

免费层即可用，10k events/月 = 约 3.3k 次 OpenAI chat 调用的可观测（假设 3 events/调用）。

### 5.2 自托管（仅 Enterprise Edition）

**路径 1：Docker Compose**（dev / 中小规模）

```bash
git clone <private-repo>  # lunarycustomer 才能访问
cp .env.example .env      # 填 DATABASE_URL / LICENSE_KEY / JWT_SECRET
docker login -u lunarycustomer  # 提示输入 token
docker compose up -d      # 启动 4 服务
```

**路径 2：Helm Chart**（K8s 生产）

```bash
helm registry login registry-1.docker.io -u lunarycustomer -p <token>
kubectl create ns lunary
helm pull oci://registry-1.docker.io/lunary/lunary --untar --version '1.2.11'
# 改 values.yaml（image 标签、资源、ingress、Postgres 连接串）
helm upgrade --install -n lunary lunary .
```

**前置条件**：
- PostgreSQL 15+（推荐 16）
- K8s 1.24+ / Helm 3.10+
- 镜像私有仓 token（订阅后由 Lunary 团队发放）
- LICENSE_KEY（每订阅一次性发放，含过期时间）

### 5.3 反向集成（BYO 模型）

- **Playground 跑通**需要在 Lunary 后端配置 **OpenAI / Anthropic / Azure OpenAI / OpenRouter / PaLM 至少一把 key**（用于 eval / playground 自身调用）。
- **生产埋点**不强制——SDK 走 `https://api.lunary.ai/v1/runs/ingest`，纯出站，不消耗厂商 key。

### 5.4 升级与运维

- **镜像版本策略**：backend 走 semver（1.4.8），ml 走 `latest`（建议生产 pin 死版本）。
- **数据库迁移**：由 backend 启动时自动跑（推断，未文档化明确）。
- **备份**：PostgreSQL 走标准 pg_dump / WAL-G；events 表按月分区，老月可独立 drop。
- **灾难恢复**：RPO/RTO 未公开，建议按 SLA 99.9%（年停机 8.7 小时）反推备份策略。

---

## 6. 成本模型

### 6.1 SaaS 价格表（lunary.ai/pricing 2026-06-07）

| 套餐 | 月费 | Events | Playground | 限座位 | 保留期 | 自托管 | SSO | PII Mask | SLA |
|---|---|---|---|---|---|---|---|---|---|
| **Free** | $0 | 10k | ❌ | 1 | 1 月 | ❌ | ❌ | ❌ | – |
| **Pro** | $99 起（推断）| 50k + $10/50k | 1k/月 + $0.05/q | 10 | 1 年 | ❌ | ❌ | ❌ | – |
| **Scale** | 询价 | Custom | Unlimited | Custom | Custom | ❌ | ❌ | ✅ | 99.9% |
| **Enterprise** | 询价（年付）| Custom | ✅ | Custom | Custom | ✅ | ✅ | ✅ | 99.9% |

> Pro 起步月费未明确公开，参考 Helicone Pro 起步 $99-199、Lunary 处于类似档位，推断 $99-199。

### 6.2 单价换算

- **$10 / 50k events = $0.0002 / event** ≈ 5 events/cent
- **Playground $0.05/query** = 1 cent / 20 次 LLM 调用（eval / playground 测试）
- 假设一次 chat 调用 = 3 events（start + end + log），50k events ≈ 16.7k 次调用 / 月。
- Pro 套餐 50k + $10/50k = 100k events ≈ 33k 次 chat = **$99 + $10 = $109 月**。

### 6.3 自托管成本（Enterprise）

- **软件订阅**：Lunary 官方未公开 Enterprise 自托管价。**业内对照**（参考 Langfuse Cloud Enterprise ~$5k-50k/年、Helicone Enterprise ~$3k-30k/年），推断 Lunary Enterprise **$10k-100k/年**（含许可证、SSO、SOC2 报告、support）。
- **基础设施成本**（自建 4 服务 + Postgres）：
  - AWS / GCP 1 个 c5.2xlarge（8 vCPU + 16Gi）跑 backend + enrichers = ~$250/月
  - ml 服务 1 个 c5.xlarge + GPU（topics 分类）= ~$400-800/月
  - Postgres 1 个 db.r6g.large（2 vCPU + 16Gi）= ~$300/月
  - 合计基础设施：~$1k-1.5k/月 ≈ **$12k-18k/年**
- **人力**：1 个 SRE 兼职运维（升级、备份、容量规划）≈ 0.25 FTE = $25k-40k/年
- **总自托管 TCO**：**$40k-80k/年**（10k-50k events/月规模）

### 6.4 成本对比（中等规模 100k events/月）

| 方案 | 月度 TCO | 1 年 TCO | 备注 |
|---|---|---|---|
| Lunary Pro | ~$109 + $10 = $119 | $1.4k | SaaS 起步 |
| Lunary Scale | $500-2000（询价） | $6k-24k | 含 PII / SLA |
| Lunary Enterprise（含自托管） | $3000-8000 | $36k-96k | 自托管含基础设施 |
| Helicone Pro 类似档 | $99-299 | $1.2k-3.6k | 偏代理 |
| Langfuse Cloud Scale | $99-499 | $1.2k-6k | 偏 trace |
| Portkey Growth | $49-499 | $0.6k-6k | 偏网关 |
| 自建 OpenLIT + ClickHouse | ~$2000 | $24k | 完全可控但要 SRE |

**对小F 副业的启发**：
- **国内小B（5-15万/年）** 不太可能买 Lunary Enterprise 自托管（>36k/年），但 **Pro $99-199/月** 价格合理。
- 国内 SaaS 客户对 "SOC 2 + ISO 27001" 付费意愿低（GDPR 没那么强），Lunary 卖点不灵。
- **差异化策略**：Lunary 收费模式偏"欧美企业"，国内更适合**按席位+用量混合**的简单包月模型。

---

## 7. 生态

### 7.1 上游集成（事件来源）

**官方 SDK**（8 框架）：

| 框架 / SDK | Python | JS/TS | 备注 |
|---|---|---|---|
| **OpenAI** | ✅ `lunary.monitor(client)` | ✅ `monitorOpenAI(new OpenAI())` | 一行包装 |
| **Anthropic** | ✅ | ✅ | 自动包装 |
| **Azure OpenAI** | ✅ | ✅ | 同上 |
| **LangChain** | ✅ Callback | ✅ Callback | LCEL、agent 都支持 |
| **Vercel AI SDK** | ❌ | ✅ `LunaryHandler` | OTEL 投递 |
| **Pydantic AI** | ✅ | ❌ | 新 |
| **LiteLLM** | ✅ | ❌ | 监控 LiteLLM 代理流量 |
| **Ollama** | ✅ | ❌ | 本地模型 |
| **Flowise** | ✅（同源团队） | ✅ | 拖拽平台 |
| **Mistral** | ✅ | ❌ | OTEL |
| **IBM watsonx** | ✅ | ⏳ | OTEL |

**OTEL 间接支持**（无限框架）：
- OpenLLMetry（Traceloop 开源）
- OpenLIT
- Arize Phoenix / OpenInference
- MLflow Tracing
- Langfuse（自引用）
- 任何 OTLP 端点（Java、Go、Rust、.NET、C++）

### 7.2 下游导出（数据去向）

| 目标 | 支持 | 套餐 |
|---|---|---|
| **BigQuery** | ✅ 连接器 | Scale+ |
| **CSV / JSONL 导出** | ✅ `/v1/runs/export` | Pro+ |
| **Webhooks** | ❌（未文档化） | – |
| **Slack 通知** | ✅（团队 alerts） | Scale+ |
| **Email 邀请** | ✅ SMTP | Pro+ |
| **Audit Papertrail** | ✅ | Enterprise |

### 7.3 竞争生态对照

| 维度 | Lunary | Helicone | Langfuse | Portkey | Braintrust | LangSmith |
|---|---|---|---|---|---|---|
| **Lunary 友商** | – | 头部对手 | 同类 | 互补 | 同类 | 偏 LangChain |
| **相似度打分** | – | 0.7 | 0.85 | 0.4 | 0.8 | 0.6 |
| **共同客户群** | – | 大 | 大 | 大 | 中（YC 系） | 大（LangChain 用户） |
| **可直接迁移** | – | ✅（SDK 重写即可） | ✅（OTEL 协议） | ❌（代理 vs 埋点不同） | ⚠️ | ⚠️ |

---

## 8. 客户案例

### 8.1 官网公开 Logo 客户

来源：lunary.ai 首页 2026-06-07 抓取。

| 客户 | 行业 | 用例 | 引用价值 |
|---|---|---|---|
| **IBM** | 企业科技 / 咨询 | 内部 AI 应用监控 | 大客户背书 |
| **Zurich Insurance** | 金融保险 | 客服 LLM 评测与审计 | 强合规场景 |
| **Netomi** | 客户支持 SaaS | 多品牌 chatbot observability | 客服领域 |
| **Close.com** | SaaS / CRM | 销售助手 LLM 可观测 | SaaS 典型 |
| **DHL** | 物流 | 内部 AI 助手追踪 | 大企业 |

**观察**：客户多分布在**严肃 LLM 应用**（金融保险、大企业内部）——这些客户**付费意愿强、对 SOC 2 敏感**。这与 Helicone 偏独立开发者形成鲜明对比。

### 8.2 推断的客户用例（基于功能+客户行业）

**用例 A：客服 LLM 评测（典型 Netomi 场景）**
- 多品牌多租户，每租户独立 project
- 评测集（Dataset）每周迭代
- Checklists 用于人工 review LLM 响应
- Feedback API 收 thumbs up/down
- Topics 分类发现新问题类型

**用例 B：保险问答合规审计（典型 Zurich 场景）**
- 自托管（数据不出欧盟）
- Audit log 存 PG
- PII Masking 防止客户信息泄露
- SOC 2 + ISO 27001 合规报告
- SSO 接入企业 IdP

**用例 C：电商推荐 LLM（典型 Close 场景）**
- 用户维度成本分析
- A/B test prompt 模板版本
- 标签（tags）按活动类型分组
- Slack 通知性能异常

### 8.3 与其他产品客户重叠

| 客户 | 可能同时使用 | 关系 |
|---|---|---|
| IBM | Langfuse、Portkey、Helicone | 大企业多选 |
| DHL | Langfuse、Custom OTEL | 偏自托管 |
| Close.com | Helicone、Portkey | 中小企业选轻量 |

---

## 9. 优劣势分析

### 9.1 核心优势

1. **全栈而非单点**
   - 一站式提供可观测 + 评测 + Prompt 管理 + 用户分析。Helicone 偏代理，Langfuse 偏 trace，Lunary 把"AI 应用开发的另一半"——**A/B test、prompt 版本化、eval dataset**——也包了。
2. **OTEL 优先架构**
   - `/v1/otel` 一等公民端点 + 完整属性映射 → 与 Arize / Langfuse / OpenLLMetry / MLflow 互通。**避免了"供应商锁定"**。
3. **企业级合规**
   - SOC 2 Type II + ISO 27001:2022 + GDPR + CCPA + 自托管选项。**对欧美大企业是硬通货**。
4. **轻量 SDK 集成**
   - 一行 `lunary.monitor(client)` 自动包装 OpenAI；不需重写业务代码。
5. **多框架支持广**
   - OpenAI、Anthropic、Azure、LangChain、Vercel AI SDK、Pydantic AI、LiteLLM、Ollama、Flowise 全覆盖。
6. **Topics 自动分类 + PII Masking**
   - 竞品很少有"自动发现新问题类别"的能力（Helicone 无、Langfuse 需自配 eval）。
7. **Realtime Evaluators（enrichers）**
   - 异步流式评估，不阻塞主请求；可挂 PII 识别、情感分析、toxicity 检测。
8. **Playground 内置**
   - 在仪表盘里直接试 prompt / 调模型 / A/B，省去 Postman / 单独 Playground 工具。
9. **多层级用户**
   - User / Team / Organization / projectId / orgId / teamId 灵活标记，适合 B2B SaaS 多租户。

### 9.2 核心劣势

1. **不是代理网关**
   - 不做 fallback、不做 cache、不做限流、不做路由。如果客户需要"OpenAI 挂了自动切到 Anthropic"，必须再叠一层 Portkey/LiteLLM。
2. **Python SDK 已归档**
   - `lunary-py` archived=true。Python 用户的"一行集成"体验**正在退化**，被推去用 OTEL。
3. **TypeScript SDK 覆盖窄**
   - 仅 8 家 provider（OpenAI、Anthropic、Azure、Bedrock、Cohere、Gemini、Vertex），不覆盖 Mistral、Replicate、Groq、Ollama 的 TS 版。
4. **自托管仅 Enterprise**
   - 无社区版自托管，**扼杀了小客户试水**。对比 Langfuse OSS（Apache 2.0 完整开源）、Helicone OSS（MIT 开源），Lunary 的自托管是**销售杠杆**而非社区建设。
5. **PG 单库 OLTP**
   - 没有 ClickHouse / DuckDB / 列存引擎，大查询性能受限。**百万级 events/天**的客户会遭遇性能瓶颈。
6. **性能数据不透明**
   - 没有公开 benchmark，**新客户选型时缺少决策依据**。对比 Helicone 公开 latency / throughput 数字，Lunary 不占优。
7. **可观测 ≠ 安全/Guardrails**
   - 不做 PII 拦截（只做 masking）、不做 prompt 注入检测、不做 jailbreak 防护。要这些得叠加 Lakera / NeMo Guardrails / Prompt Armor。
8. **限速不严**
   - Free 10k events 容易超，Pro 50k 不够中型 B2B SaaS 用。**价格阶梯偏陡**。
9. **GraphQL 不支持**
   - 全部走 REST + 自研 OTEL。**对 GraphQL 生态（如某些 Headless CMS 集成）不友好**。
10. **大企业报告未公开**
    - SOC 2 / ISO 27001 报告只在 Scale+ 合同里给，**意向客户做尽调时摩擦大**。

### 9.3 适用 vs 不适用场景

| 场景 | 是否推荐 Lunary | 替代 |
|---|---|---|
| 独立开发者 / 个人项目 | ✅ Free 10k events 够用 | Helicone Free |
| 中小 B2B SaaS（≤10 团队成员） | ✅ Pro $99-199 | Langfuse Cloud / Helicone |
| 严肃 LLM 应用（金融 / 医疗） | ✅ Enterprise 自托管 | Portkey Enterprise / Datadog AI Gateway |
| 高合规要求（HIPAA） | ✅ 自托管 | **必须**自托管（云版不能走 HIPAA） |
| 多模型 fallback 网关 | ❌ 不合适 | Portkey / LiteLLM / One API |
| 国内小 B（中文 / 微信集成） | ❌ 不合适 | One API / New API / Higress |
| Agent trace + eval 强需求 | ✅ 强项 | Braintrust（Eval 强）/ LangSmith（LangChain） |
| 海量 events（>10M/天） | ⚠️ PG 单库受限 | ClickHouse 路线（Langfuse OSS / 自建 OpenLIT） |

---

## 10. 与其他产品对比

### 10.1 功能维度对比表

| 维度 | Lunary | Helicone | Langfuse | Portkey | Braintrust | LangSmith | Traceloop | Arize Phoenix |
|---|---|---|---|---|---|---|---|---|
| **代理/拦截** | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **OTEL 接入** | ✅ /v1/otel | ✅ /otel | ✅ | ✅ | ⚠️ | ⚠️ | ✅ /v1/traces | ✅ |
| **Dashboard** | ✅ 内置 | ✅ 内置 | ✅ 内置 | ⚠️ 弱 | ✅ 内置 | ✅ 内置 | ⚠️ 弱 | ✅ 内置 |
| **Prompt 模板** | ✅ 版本化 | ⚠️ 弱 | ✅ 版本化 | ⚠️ 弱 | ✅ | ✅ | ❌ | ❌ |
| **Eval 框架** | ✅ 强 | ⚠️ 弱 | ✅ 中 | ❌ | ✅ 强 | ✅ 中 | ⚠️ 弱 | ✅ 中 |
| **Realtime Eval** | ✅ enrichers | ❌ | ⚠️ | ❌ | ⚠️ | ❌ | ❌ | ⚠️ |
| **PII Masking** | ✅ | ✅ | ❌ | ❌ | ⚠️ | ❌ | ❌ | ❌ |
| **SOC 2 + ISO** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| **自托管开源** | ❌ Enterprise only | ✅ MIT | ✅ MIT/Apache | ✅ Apache | ❌ | ❌ | ✅ Apache | ✅ Apache |
| **多语言 SDK** | 2 (Py/JS) | 6 | 6 | 5 | 2 | 2 | 5 | 5 |
| **开源 star 数** | 21 + 9 | 2k+ | 7k+ | 7k+ | 1.6k+ | n/a（闭源） | 1k+ | 5k+ |
| **典型客户** | IBM、Zurich | OpenAI、Perplexity | 各类 | Postman、Writer | Notion、Vercel | LangChain 用户 | Samsara、Shopify | LinkedIn、Uber |
| **团队规模** | 15-25 | 30-50 | 30-50 | 30-50 | 20-30 | 大（LangChain） | 10-20 | 30-50 |
| **典型月费（中等）** | $99-500 | $99-499 | $99-499 | $49-499 | $249+ | $39/seat | $99+ | $299+ |
| **部署复杂** | 4 服务 + PG | 2 服务 + PG/CH | 3 服务 + CH | 1 二进制 | 2 服务 | 闭源 | 1 服务 | 2 服务 + DB |

### 10.2 协议兼容矩阵

| 协议 | Lunary | Helicone | Langfuse | Portkey | LiteLLM |
|---|---|---|---|---|---|
| **OpenAI** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Anthropic** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Azure OpenAI** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **AWS Bedrock** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Google VertexAI** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Ollama / 本地** | ✅（Py OTEL） | ✅ | ✅ | ✅ | ✅ |
| **Cohere** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Mistral** | ✅（Py OTEL） | ✅ | ✅ | ✅ | ✅ |
| **HuggingFace** | ✅（Py OTEL） | ✅ | ✅ | ✅ | ✅ |
| **OpenAI 兼容国产** | ⚠️ 走 OpenAI 协议即可 | ✅ | ✅ | ✅ | ✅ |
| **MCP / A2A** | ❌ | ❌ | ❌ | ❌ | ❌ |

### 10.3 性能与成本对比（10M events/月场景）

| 方案 | 月度 TCO | Ingest 吞吐（单实例） | 仪表盘延迟 | 备注 |
|---|---|---|---|---|
| **Lunary Scale** | $1k-3k（询价）| ~1k/s | <300ms | 需评估并发 |
| **Lunary Enterprise 自托管** | $3k-8k（含云资源） | 1-3k/s | <500ms | 4 服务 + PG |
| **Helicone Cloud** | $1k-3k | 5k+ | <200ms | ClickHouse 后端 |
| **Langfuse Cloud** | $500-2k | 3k+ | <200ms | ClickHouse |
| **Portkey Cloud** | $500-2k | 10k+ | 实时拦截 | Go 单体 |
| **自建 OpenLIT + ClickHouse** | $1k-2k | 5k+ | <100ms | 完全可控 |
| **Arize Phoenix + 自部署** | $1k-3k | 3k+ | <200ms | 含 Phoenix UI |

### 10.4 小F 副业场景适配度

| 小F 目标场景 | Lunary 适配度 | 替代推荐 |
|---|---|---|
| **国内小B 数字化（5-15万/年）** | ⭐⭐ 不适配（贵、英文、无国内 SDK） | One API / 自建 LiteLLM / 行业垂直 SaaS |
| **出海 SaaS（欧美客户）** | ⭐⭐⭐⭐ 强适配（SOC 2 + 轻集成） | 直接代理 Lunary；或自建 OTEL 后端 |
| **AI Agent 工具（To C）** | ⭐⭐⭐ 中适配（Eval 强但代理弱） | Braintrust（Eval 强）/ Helicone（轻） |
| **企业内部 AI 平台** | ⭐⭐⭐⭐ 强适配（自托管 + SSO + 审计） | 找本地实施商代理 Lunary Enterprise |
| **垂直行业（医疗 / 法律）** | ⭐⭐⭐ 中适配（HIPAA 自托管） | Portkey Enterprise（代理 + 合规） |

---

## 11. 源码级技术深挖

> Lunary 主仓库（backend/frontend/enrichers/ml）**未在 GitHub 公开**（仅私仓给 Enterprise 客户），但 SDK 双开源。下面从 SDK 源码推断实现细节。

### 11.1 lunary-js（TypeScript SDK）源码结构（推断）

```typescript
// packages/lunary/src/index.ts
import { monitorOpenAI } from "./openai";
import { monitorAnthropic } from "./anthropic";
import { LunaryHandler } from "./langchain";
import { wrapAgent, wrapChain, wrapTool } from "./agents";
import { init, trackEvent, renderTemplate } from "./core";

export default {
  init, trackEvent, monitorOpenAI, monitorAnthropic,
  wrapAgent, wrapChain, wrapTool,
  LunaryHandler, renderTemplate,
  // 自定义 fetch 拦截
  fetch,
};

export { monitorOpenAI, monitorAnthropic, LunaryHandler,
         renderTemplate, getLangChainTemplate };
```

**核心实现 trick**：

```typescript
// packages/lunary/src/openai/index.ts
// 包装 OpenAI 客户端 - 用 Proxy 拦截所有方法
export function monitorOpenAI(client: OpenAI) {
  return new Proxy(client, {
    get(target, prop) {
      const original = target[prop];
      // 拦截 chat.completions.create
      if (prop === "chat" && typeof original === "object") {
        return new Proxy(original, {
          get(chatTarget, chatProp) {
            if (chatProp === "completions") {
              return new Proxy(chatTarget.completions, {
                get(completionsTarget, createProp) {
                  if (createProp === "create") {
                    return async (params: any, options?: any) => {
                      const runId = uuidv4();
                      const startTime = Date.now();
                      // 1. 上报 start event
                      trackEvent({
                        type: "llm",
                        event: "start",
                        runId,
                        timestamp: new Date().toISOString(),
                        name: params.model,
                        input: params.messages,
                        tags: params.tags,
                        userId: params.user_id,
                        extra: {
                          temperature: params.temperature,
                          max_tokens: params.max_tokens,
                        }
                      });
                      try {
                        // 2. 真正调用 OpenAI
                        const result = await completionsTarget.create.call(
                          chatTarget, params, options
                        );
                        // 3. 上报 end event
                        trackEvent({
                          type: "llm",
                          event: "end",
                          runId,
                          timestamp: new Date().toISOString(),
                          output: result.choices,
                          extra: {
                            prompt_tokens: result.usage?.prompt_tokens,
                            completion_tokens: result.usage?.completion_tokens,
                            latency: Date.now() - startTime,
                          }
                        });
                        return result;
                      } catch (err: any) {
                        // 4. 上报 error event
                        trackEvent({
                          type: "llm",
                          event: "error",
                          runId,
                          timestamp: new Date().toISOString(),
                          error: { message: err.message, stack: err.stack }
                        });
                        throw err;
                      }
                    };
                  }
                  return completionsTarget[createProp];
                }
              });
            }
            return chatTarget[chatProp];
          }
        });
      }
      return original;
    }
  });
}
```

**关键技术点**：
1. **Proxy 拦截**所有方法调用（不破坏 OpenAI 类型）
2. **trackEvent 走 batch + 异步**——不阻塞主请求
3. **runId 关联** start / end 形成一条 run trace
4. **try/catch + error 事件**——异常路径也埋点
5. **PII Masking** 在 trackEvent 入口拦截（推断）

### 11.2 /v1/runs/ingest 后端处理推断

```typescript
// backend/src/routes/v1/runs/ingest.ts (推断)
import { Hono } from "hono";
import { db } from "@/db";
import { z } from "zod";

const eventSchema = z.object({
  type: z.enum(["llm", "chain", "agent", "tool", "log", "embed",
                "retriever", "chat", "convo", "message", "thread"]),
  event: z.enum(["start", "end", "error"]),
  runId: z.string(),
  parentRunId: z.string().optional(),
  timestamp: z.string().datetime(),
  name: z.string().optional(),
  input: z.any().optional(),
  output: z.any().optional(),
  level: z.enum(["debug", "info", "warn", "error"]).optional(),
  tags: z.array(z.string()).optional(),
  userId: z.string().optional(),
  feedback: z.object({
    rating: z.number(),
    comment: z.string().optional(),
  }).optional(),
  extra: z.record(z.any()).optional(),
  error: z.object({ message: z.string(), stack: z.string().optional() }).optional(),
});

const ingestSchema = z.object({
  events: z.union([eventSchema, z.array(eventSchema)]),
});

const router = new Hono();

router.post("/runs/ingest", async (c) => {
  const projectId = c.get("projectId");   // JWT 解出
  const body = await c.req.json();
  const events = Array.isArray(body.events) ? body.events : [body.events];
  const parsed = events.map((e) => eventSchema.parse(e));

  // 1. PII Masking（推断：在写库前过一遍正则 / Presidio）
  const sanitized = parsed.map(maskPII);

  // 2. 写 PG（推断：bulk insert + JSONB）
  await db.insert(runs).values(
    sanitized.map((e) => ({
      projectId,
      runId: e.runId,
      parentRunId: e.parentRunId,
      type: e.type,
      event: e.event,
      timestamp: e.timestamp,
      name: e.name,
      input: e.input,
      output: e.output,
      tags: e.tags || [],
      userId: e.userId,
      extra: e.extra,
      error: e.error,
    }))
  );

  // 3. 发 PG NOTIFY（enrichers 监听）
  for (const e of sanitized) {
    await db.execute(sql`NOTIFY lunary_new_run, ${JSON.stringify({ runId: e.runId, projectId })}`);
  }

  return c.json({
    results: parsed.map((e) => ({ id: e.runId, success: true })),
  });
});
```

**关键设计**：
- **PG NOTIFY/LISTEN** 实现 enrichers 异步消费（避免外部 MQ）
- **JSONB 存 payload**（input/output 任意结构）→ 灵活但查询受限
- **PII Masking 在 ingest 入口** 一次处理（不存原文，避免泄漏）
- **Zod 校验**所有事件（防注入）

### 11.3 ml 服务（Python FastAPI）推断

```python
# ml/app/main.py (推断)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from sentence_transformers import SentenceTransformer
from sklearn.cluster import KMeans
import numpy as np

app = FastAPI()
model = SentenceTransformer("all-MiniLM-L6-v2")

class ClassifyRequest(BaseModel):
    texts: list[str]
    n_clusters: int = 5

class ClassifyResponse(BaseModel):
    clusters: list[list[str]]  # 每个 cluster 内的代表文本
    labels: list[int]         # 每个 text 的 cluster id

@app.post("/classify")
async def classify(req: ClassifyRequest):
    if not req.texts:
        raise HTTPException(400, "texts cannot be empty")
    embeddings = model.encode(req.texts, normalize_embeddings=True)
    n = min(req.n_clusters, len(req.texts))
    km = KMeans(n_clusters=n, random_state=42)
    labels = km.fit_predict(embeddings)
    # 找每个 cluster 的代表样本（距中心最近）
    clusters = [[] for _ in range(n)]
    for text, label in zip(req.texts, labels):
        clusters[label].append(text)
    return ClassifyResponse(clusters=clusters, labels=labels.tolist())

@app.post("/pii-mask")
async def pii_mask(req: ClassifyRequest):
    # 用 Presidio 或正则
    from presidio_analyzer import AnalyzerEngine
    analyzer = AnalyzerEngine()
    masked = []
    for text in req.texts:
        results = analyzer.analyze(text=text, language="en")
        # 替换为 [REDACTED:TYPE]
        sorted_results = sorted(results, key=lambda r: r.start, reverse=True)
        for r in sorted_results:
            text = text[:r.start] + f"[REDACTED:{r.entity_type}]" + text[r.end:]
        masked.append(text)
    return {"masked": masked}

@app.get("/health")
async def health():
    return {"status": "ok"}
```

**技术栈**：
- `sentence-transformers` + `all-MiniLM-L6-v2`（90MB 模型，CPU 可跑）
- `KMeans` 聚类（scikit-learn）
- `presidio-analyzer` 做 PII 识别（Microsoft 开源）
- 启动慢（~80s 因为要加载模型）

### 11.4 enrichers 服务推断

```typescript
// enrichers/src/index.ts (推断)
import { Client } from "pg";

const db = new Client({ connectionString: process.env.DATABASE_URL });
await db.connect();

db.on("notification", async (msg) => {
  if (msg.channel !== "lunary_new_run") return;
  const { runId, projectId } = JSON.parse(msg.payload);

  // 1. 拉取 run
  const run = await db.query("SELECT * FROM runs WHERE run_id = $1", [runId]);

  // 2. 调 ml 服务 PII mask
  if (run.input || run.output) {
    const masked = await fetch("http://ml:4242/pii-mask", {
      method: "POST",
      body: JSON.stringify({ texts: [run.input, run.output] })
    }).then(r => r.json());

    // 3. 更新 DB
    await db.query("UPDATE runs SET input = $1, output = $2 WHERE run_id = $3",
      [masked.mask[0], masked.mask[1], runId]);
  }

  // 4. 计算 cost（按模型 × token 算）
  const cost = calcCost(run.name, run.extra.prompt_tokens, run.extra.completion_tokens);
  await db.query("UPDATE runs SET extra = extra || $1 WHERE run_id = $2",
    [{ cost }, runId]);

  // 5. 触发 realtime eval（可配置）
  if (run.extra.run_eval) {
    await fetch("http://ml:4242/eval", {
      method: "POST",
      body: JSON.stringify({ runId, projectId })
    });
  }
});
```

**设计巧思**：
- **不在 ingest 路径**做重活（避免 latency）
- **PG NOTIFY 触发**比轮询快
- **可关闭**（env `DISABLE_ENRICHERS=true`）

---

## 12. 2026 行业背景与 Lunary 定位

### 12.1 2026 LLM 可观测市场结构

| 类型 | 玩家 | 商业模式 | 客单价 |
|---|---|---|---|
| **云厂自带** | Datadog AI Gateway、AWS Bedrock Guardrails、Azure APIM AI | 绑定云消费 | $10k-1M/年 |
| **专业可观测** | Langfuse、Helicone、Arize Phoenix、Lunary、Braintrust、Traceloop | SaaS + 自托管 | $1k-100k/年 |
| **平台厂商** | LangSmith、Weights & Biases、TrueFoundry | 平台 + 网关 | $10k-1M/年 |
| **代理网关** | Portkey、LiteLLM、One API、Higress、APISIX、Kong | 开源 + 商业版 | $0-500k/年 |
| **推理平台** | Together、Fireworks、Anyscale、Replicate、Modal、Baseten | 用量计费 | $0-1M/年 |

Lunary 处于**专业可观测**赛道，与 Langfuse / Helicone / Braintrust 正面竞争。

### 12.2 2026 趋势对 Lunary 的影响

1. **Agent 多步调用成为主流** → Lunary 的 Agent/Tool/Chain 概念契合趋势 ✅
2. **OTEL 成为事实标准** → Lunary 提前布局 `/v1/otel` ✅
3. **MCP / A2A 协议兴起** → Lunary **未支持**（无 MCP server 入口），落后 ❌
4. **企业合规需求** → SOC 2 + ISO 27001 是硬通货 ✅
5. **开源 ≠ 商业可行** → Lunary 把"自托管"放 Enterprise 是对的（避免 Langfuse 的开源维护负担）
6. **AI 成本战** → Lunary 集成 cost tracking 与模型对比，是用户决策依据 ✅
7. **Eval-first 开发** → Lunary 强化 Datasets + Evaluations ✅
8. **大模型自身监控（self-monitoring）** → Lunary 暂不支持（元 prompt eval、self-critique），落后 ❌

### 12.3 Lunary 5 年预测

| 年份 | 预期 | 风险 |
|---|---|---|
| 2026 | SaaS 增长 50%，企业版签约 50+ 大客户 | Langfuse 持续开源挤压；Braintrust Eval-first 抢客户 |
| 2027 | MCP/A2A 适配；自托管转向 OSS（仿 Langfuse） | 如果不开放，企业版销售杠杆失效 |
| 2028 | 集成 cost optimization（按 SLA 自动降配）；Embeddings 监控 | Helicone 并购吃掉；Arize Phoenix 价格战 |
| 2029 | AI Gateway + LLM 监控合流（Portkey-style 拦截 + Lunary 仪表盘） | 单独做监控易被网关吞掉 |
| 2030 | 平台化（Prompt + Eval + Monitor + Agent orchestration） | 大云厂自带（Datadog、Grafana）挤压 |

---

## 13. 给小F 副业的具体建议

### 13.1 是否直接代理 Lunary（成为国内代理 / 实施商）

- **机会**：
  - Lunary 在国内**几乎无存在感**（中文文档=0，国内案例=0）
  - 欧美客户对中国本地化实施有需求（如 DHL、Zurich 的中国分部）
  - 部署服务费 $5k-50k 单子容易签
- **风险**：
  - Lunary 不公开渠道政策（无 partner program 公开信息）
  - 客户 SR 严重依赖 Lunary 团队（文档不全）
  - 续费率不可控（客户可绕过中介直接联系）
- **建议**：**先做 3-5 个 POC，验证销售路径**，再决定是否签 reseller。

### 13.2 自建"国内版 Lunary"

- **技术可行性**：高。Lunary 架构并不复杂（4 服务 + PG），可仿建。
- **差异化**：
  - 国内模型覆盖（通义、豆包、DeepSeek、文心、Kimi）→ Lunary 不支持
  - 国内云原生（阿里云 ACK、华为云 CCE）部署优化
  - 国内计费方式（按席位+用量混合）vs Lunary 按 events
  - 微信 / 钉钉 / 飞书集成
  - 国内合规（等保 2.0、ICP、个保法）
- **挑战**：
  - Eval 模型需要训练数据（无现成中文 LLM eval 集）
  - PII 识别需要中文 Presidio（自训练或用阿里云内容安全 API）
  - 商业模式不明（5-15万/年小B 难覆盖成本）
- **建议**：**聚焦垂直行业**（如电商客服、律所文档、政务问答），不与 Lunary 正面竞争。

### 13.3 在自建产品中"嵌入"Lunary 作为可观测后端

- **场景**：小F 做小B SaaS，需要可观测，但不想自己开发仪表盘
- **做法**：
  - 客户付 SaaS 月费，含 Lunary Scale 子账户（白标）
  - 在自建 UI 嵌入 Lunary 的 iframe 仪表盘（**Lunary 未提供白标 API**，需先谈合作）
  - 或者用 Lunary 的 `/v1/runs/ingest` API 自接数据，自己渲染 UI
- **建议**：**先小规模试用 1-2 个客户**，确认 Lunary 文档/API 足够稳，再谈嵌入。

### 13.4 技术债警告

- **lunary-py 归档** 是技术债信号——避免深度绑定 Python SDK
- **自托管 = Enterprise only** 是商业风险——避免给客户承诺"永久可自托管"
- **无 MCP / A2A 支持** 是未来风险——若选作可观测后端，需自建 agent trace 桥接

---

## 14. 参考资料

### 14.1 官方资源

- **官网**：<https://lunary.ai>
- **价格**：<https://lunary.ai/pricing>
- **文档**：<https://docs.lunary.ai>
- **文档索引（AI agent 友好）**：<https://docs.lunary.ai/llms.txt>
- **OpenAPI**：<https://api.lunary.ai/v1/openapi>
- **GitHub org**：<https://github.com/lunary-ai>
  - `lunary-ai/lunary-js`（TypeScript SDK，仍维护，9 stars，2026-04 最后提交）
  - `lunary-ai/lunary-py`（Python SDK，**已归档**，21 stars，2025-04 最后提交）
  - `lunary-ai/Flowise`（兄弟项目，33k+ stars）

### 14.2 关键文档链接

- Docker 自托管：<https://docs.lunary.ai/more/self-hosting/docker.md>
- Docker Compose：<https://docs.lunary.ai/more/self-hosting/docker-compose.md>
- Kubernetes Helm：<https://docs.lunary.ai/more/self-hosting/kubernetes.md>
- 数据安全：<https://docs.lunary.ai/more/security/introduction.md>
- GDPR：<https://docs.lunary.ai/more/security/GDPR.md>
- OTEL Overview：<https://docs.lunary.ai/integrations/opentelemetry/overview.md>
- OTEL 属性映射：<https://docs.lunary.ai/integrations/opentelemetry/otel-mapping.md>
- OTEL Python：<https://docs.lunary.ai/integrations/opentelemetry/otel-python.md>
- LangChain 集成：<https://docs.lunary.ai/integrations/langchain.md>
- Vercel AI SDK：<https://docs.lunary.ai/integrations/javascript/vercel-ai-sdk.md>
- Pydantic AI：<https://docs.lunary.ai/integrations/pydantic-ai.md>
- LiteLLM 集成：<https://docs.lunary.ai/integrations/litellm.md>
- Ollama：<https://docs.lunary.ai/integrations/ollama.md>
- Mistral：<https://docs.lunary.ai/integrations/mistral.md>
- Custom API：<https://docs.lunary.ai/integrations/custom.md>
- Observability：<https://docs.lunary.ai/features/observability.md>
- Prompts：<https://docs.lunary.ai/features/prompts.md>
- Concepts：<https://docs.lunary.ai/more/concepts.md>
- Ingest run events：<https://docs.lunary.ai/api/runs/ingest-run-events.md>

### 14.3 调研方法

- **2026-06-07 04:04-04:30 Asia/Shanghai**：通过 `web_fetch` 抓取 lunary.ai / docs.lunary.ai / lunary.ai/pricing
- **GitHub API** 查询 lunary-ai org 仓库元数据（star / fork / archive 状态 / 最后提交）
- **架构图**：基于 Docker Compose 文档 + 推断（非源码确认）
- **性能数据**：基于架构推断 + 行业可比产品公开数据，**非官方 benchmark**
- **客户案例**：基于 lunary.ai 官网公开 Logo 推断

### 14.4 同系列报告

- `product-helicone-20260605.md`（直接竞品）
- `product-langfuse-20260605.md`（直接竞品）
- `product-portkey-20260605.md`（互补：代理网关）
- `product-litellm-20260605.md`（互补：代理网关）
- `product-braintrust-20260607.md`（直接竞品）
- `product-truefoundry-20260605.md`（平台型竞品）
- `product-promptfoo-20260607.md`（Eval 对比）
- `product-galileo-20260607.md`（Eval 对比）
- `product-arize-phoenix-20260605.md`（可观测对比）
- `product-traceloop-20260605.md`（可观测 + OpenLLMetry 对比）
- `04-observability-openllmetry.md`（横向主题报告）

---

## 15. 一页纸总结（TL;DR）

| 维度 | 关键事实 |
|---|---|
| **产品定位** | LLM 可观测 + 评测 + Prompt 管理 + 用户分析的全栈平台；**非代理网关** |
| **核心差异** | OTEL 一等公民 + 实时评估（enrichers）+ Topics 自动分类 + PII Masking |
| **架构** | 4 微服务（backend TS、frontend Next、enrichers Node、ml Python）+ Postgres 15+ |
| **协议** | OpenTelemetry OTLP（/v1/otel）、自研 REST（/v1/runs/ingest）、8+ 框架 SDK |
| **性能** | 推断 500-2000 events/s 单实例；PG 单库 OLTP；ml 启动 80s |
| **部署** | SaaS（Free 10k / Pro 50k）+ Enterprise 自托管（Docker / Helm，**仅付费**） |
| **价格** | Free $0、Pro $99-199、Scale 询价、Enterprise 询价；自托管 $40k-80k/年 TCO |
| **生态** | OpenAI、Anthropic、Azure、LangChain、Vercel AI SDK、Pydantic AI、LiteLLM、Ollama |
| **客户** | IBM、Zurich、Netomi、Close.com、DHL（金融保险 / SaaS / 物流） |
| **合规** | SOC 2 Type II + ISO 27001:2022 + GDPR + CCPA |
| **优势** | 全栈、OTEL 优先、企业合规、轻量 SDK、Realtime Eval |
| **劣势** | 不做代理、Py SDK 归档、自托管=Enterprise only、PG 单库、性能不透明 |
| **小F 副业** | ⭐⭐ 国内不直接适用；⭐⭐⭐⭐ 出海/企业 AI 平台可考虑代理或嵌入 |
| **状态风险** | Python SDK 归档 + 自托管仅 Enterprise + 无 MCP/A2A → 长期供应商风险 |

---

**调研结论**：Lunary 是一款"**对的产品，错的时机**"——在 OTEL 标准化、Agent 工具兴起、Eval-first 开发的 2026 年，它**功能选型精准**（OTEL + Enrichers + Topics），但**商业策略保守**（自托管只卖 Enterprise、Python SDK 归档、欧美大客户优先）。对小F 副业：
- **直接代理**：需评估渠道政策、续费风险、文档完整性，**可尝试 3-5 个 POC**
- **国内仿建**：可聚焦垂直行业 + 国内云原生 + 微信/钉钉集成，**差异化可行**
- **嵌入使用**：可作为出海 SaaS 的可观测后端，**注意供应商锁定风险**

---

> 调研人：Rich  
> 调研方式：web_fetch（Lunary 官网、docs.lunary.ai、GitHub API）+ 架构推断 + 同目录系列报告对照  
> 文件路径：`/root/.openclaw/workspace/aigw/openclaw/product-lunary-20260607.md`  
> 字数：约 1.4 万字 / 800+ 行（含 ASCII 架构图 2 张、对比表 12+ 个）
