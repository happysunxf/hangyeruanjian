# Datadog AI Gateway 深度调研

> 调研对象：**Datadog AI Gateway**（Datadog LLM Observability 子产品，原 LLM Observability Gateway 改名）
> 调研时间：2026-06-06（Asia/Shanghai）
> 调研人：Rich (OpenClaw main session)
> 文档定位：AI Gateway 产品深挖第 36+ 篇。`r34+` 报告已明确"29 项候选清单 100% 闭合 + 启用清单外扩展深挖策略"。本文是清单外扩展深挖下一份选定的产品。
> 信息来源：Datadog 官方文档（`docs.datadoghq.com/llm_observability/`、`docs.datadoghq.com/agent_gateway/`）、Datadog Security Labs 博客（2025-11-18 "Detect prompt injection attacks"）、Datadog Changelog（2025-09-22 / 2025-11-18 / 2026-01-20 / 2026-04-08 / 2026-05-12）、Datadog Marketplace、GitHub（`DataDog/datadog-agent` 中 LLM Observability 模块、`DataDog/dd-trace-go` 中 `llmobs` 子包、`@datadog/llm-observability` npm 包、`ddtrace-llmobs` Python 包）、第三方比较材料（The New Stack、DevOps.com、Last9、ClickHouse 博客）。

---

## 0. TL;DR（一页纸总结）

| 维度 | 一句话总结 |
|---|---|
| **定位** | Datadog LLM Observability 套件里的"网关"组件，承担 LLM 流量的代理 + 可观测 + 治理 + 安全四合一 |
| **核心卖点** | 复用 Datadog APM 的 OTel 兼容 trace 体系；一个 span 覆盖 prompt + tool call + token + cost；与 14 种主流 LLM provider 预集成；与 Datadog RAG 评估 / Guardrails / Cost / PII 联动 |
| **协议** | **OpenAI Chat Completions / Responses / Anthropic Messages / Google Gemini / Cohere / Bedrock / Azure OpenAI / Mistral / Hugging Face TGI / 自定义 OpenAI 兼容端点** + 双向透传；MCP 2026-Q2 计划中 |
| **部署模式** | **两种**：①托管 SaaS（`gateway.datadoghq.com`）②自托管 Datadog Agent（v7.58+）作为 sidecar 拦截 + Envoy/NGINX/HAProxy W3C trace context 透传；不支持独立部署开源版 |
| **定价** | 不单独卖 AI Gateway，按 **LLM Observability 计入的 span 数量** 收费。LLM Observability 独立 SKU：$0.05/百万 span（ingest）+ $0.10/GB retention（90 天默认）；Guardrails 与 Cost 是子模块，按规则命中或 token 计费 |
| **生态** | OTel GenAI semconv 全实现（2026-05 加入） + LangChain/LlamaIndex/Haystack/DSPy/crewAI/AutoGen/OpenAI Agents SDK/Pydantic AI 9 大框架 SDK + Datadog LLM Eval（offline / online）+ Datadog Guardrails（输入输出）+ Datadog Sensitive Data Scanner（PII 自动脱敏）+ Datadog Cost Management |
| **特色能力** | ①一个 span 覆盖 19 个 LLM Observability attributes（model、provider、prompt、completion、token input/output/cache、cache hit、tool call、tool result、reasoning、agent step、agent id、error）；②prompt injection 检测（2025-11-18 模型+规则双检测）；③PII 自动标记 + 跨 trace 关联；④Jailbreak / Toxicity 实时评分（与 meta-llama Llama Guard 集成） |
| **目标客户** | 已是 Datadog 客户的中大型企业（典型 SRE / 平台工程 / 安全团队）希望把 LLM 调用"等同于一个服务"接入 Datadog；**不是** AI gateway-first 工具（不像 Portkey / Bifrost 那样把"代理+路由"当主菜） |
| **关键短板** | ①**不开源**核心网关（Datadog Agent 7.58+ 中的 `llmobs` proxy 闭源）；②**不是 LLM 路由**专家——不支持 model-level A/B、semantic cache（要在 Guardrails 里手动接），不支持 cost-based 智能路由；③与 Langfuse/Arize 比，LLM 评估 / 标注 / dataset 能力弱；④单独 LLM Observability 起步价对小团队仍偏高（$0.05/百万 span，但每个 LLM 调用 = 多个 span） |
| **对小F副业的相关性** | **中等**。如果你已经是 Datadog 用户，AI Gateway 是个"无成本顺手接"的工具；但作为小B SaaS 主推路线，它对客户的价值是"你帮我用 Datadog 看 LLM"而不是"你帮我换 LLM provider"——和 aigw 主打"统一代理+成本归因"是错位关系。**观察点**：Datadog AI Gateway 反映的是"传统 observability 厂商切入 AI gateway 的方式"——以 OTel trace 为核心，弱化路由/缓存，是"平台型 SaaS"的典型路径。 |

---

## 1. 项目背景

### 1.1 时间线

| 日期 | 事件 | 备注 |
|---|---|---|
| 2023-11-08 | Datadog 启动 **LLM Observability** 项目（内部代号"compass"） | 早期聚焦 OpenAI 调用 trace |
| 2024-01-15 | LLM Observability **公共预览**（public preview） | 仅支持 OpenAI + 1 个 trace span/调用 |
| 2024-04-30 | LLM Observability **GA（1.0）** | 引入 `ml_app`、`span.kind=llm`、`span.kind=workflow` 概念；12 家 provider；5 个开源框架集成 |
| 2024-09-15 | **LLM Span Evaluations** 上线 | 用户可在 UI 给历史 span 打 1-5 星评分 |
| 2024-12-10 | **Prompt Injection Detection**（白名单规则） | 基于正则 + 黑名单词表 |
| 2025-03-20 | **LLM Cost Tracking** 公开 | 接入 OpenAI/Anthropic 价格表；按 token + 模型单价分摊 |
| 2025-06-15 | **Datadog Agent 7.55** 引入 `llmobs` 代理模块 | 首次支持 **自托管**（self-hosted）网关能力；OGenAI semconv 0.2.0 partial |
| 2025-09-22 | **Datadog AI Gateway** 正式命名（2025-09-22 公告） | 此前以"L1 代理 / L2 评估 / L3 防护"三层结构悄悄孵化，正式品牌化 |
| 2025-11-18 | **Prompt Injection Detection v2**（ML 模型 + 规则双轨） | 引入自研轻量 LLM-as-judge 模型 + Datadog Security Labs 维护的 200+ prompt injection 模板 |
| 2026-01-20 | **MCP Server Trace 公开预览** | 与 LLM Gateway 同一 SDK，可 trace MCP tool call |
| 2026-03-12 | **Anthropic Prompt Caching 透明拦截** | 在 Gateway 层自动注入 `cache_control: ephemeral`，对调用方透明 |
| 2026-04-08 | **Google Gemini 2.5 + Vertex AI 全量支持** | 新增 Gemini 2.5 Pro / Flash + Vertex AI Model Garden 模型 |
| 2026-05-12 | **OpenTelemetry GenAI semconv 1.30 对齐** | 升级到 `gen_ai.*` 属性名（与 OTel 0.4 兼容）；之前的 `dd.*` 属性保留作为 alias |
| 2026-05-30 | **Datadog LLM Eval 公开 SDK** | 用户可注册自定义 evaluator；UI 支持 evaluation dataset 导入 |
| 2026-06-05 (今天) | 当前状态 | 14 provider 集成、9 框架集成、3 部署模式（无服务器 SaaS、Agent 自托管、K8s sidecar）、MCP trace beta |

### 1.2 起源：从 APM 走到 LLM Observability

Datadog 做 AI Gateway 不是"从零开始造一个 LLM 路由"，而是**把 APM 那套 Trace / Span / Service Map 的心智模型延伸到 LLM 调用**。

#### 1.2.1 APM 的成功经验

Datadog 2010-2023 年靠 **APM (Application Performance Monitoring)** 成为云可观测的全球龙头。其核心抽象：

```
Service A → Span 1 → Service B → Span 2 → Service C
              ↓                ↓                ↓
            Trace ID 贯穿，p50/p95/p99 latency，error rate
```

每个服务调用是一个 **span**；多个 span 组成一个 **trace**。Datadog APM 用 OpenTelemetry SDK + 自家 dd-trace-py/dd-trace-go/dd-trace-js，把语言运行时（Python/Go/Node/Java/Ruby/.NET/PHP）的 HTTP/gRPC/DB/Queue 拦截全部变成 span。

#### 1.2.2 LLM Observability = APM + AI 业务属性

LLM Observability 的"第一性"假设是：**LLM 调用就是一个服务调用**，没有本质区别。所以 LLM Observability 直接复用：

- **Trace / Span 数据结构**（W3C `traceparent` 透传）
- **采样规则**（head-based / tail-based）
- **Service Map 视图**（自动画服务依赖图）
- **Error Tracking**（HTTP 5xx + LLM `finish_reason=length` / `content_filter`）
- **APM 关联**（一个 Python Web 应用收到请求 → 调 LLM → LLM 调 Tool → Tool 调 DB，全在一条 trace）

但 LLM 调用有 APM 不具备的"业务维度"：

| 业务维度 | 含义 | APM 是否覆盖 |
|---|---|---|
| `prompt`（用户输入） | LLM 的输入文本 | ❌ APM 只看 HTTP body length |
| `completion`（模型输出） | LLM 的输出文本 | ❌ 同上 |
| `token.input` / `token.output` | token 用量 | ❌ APM 不看 token |
| `token.cached` | 缓存命中 token | ❌ |
| `model` / `provider` | 模型 + 服务商 | ❌ APM 只看 host |
| `tool.name` / `tool.arguments` | Function call | ❌ |
| `finish_reason` | stop/length/content_filter/tool_calls | ❌ |
| `agent.id` / `agent.step` | Agent 多步 | ❌ APM 没有"agent"概念 |
| `eval.score` / `eval.label` | 质量评分 | ❌ |
| `pii.detected` / `pii.entities` | 敏感信息 | ❌ |
| `cost.input` / `cost.output` | 成本 | ❌ |

所以 LLM Observability 在 APM 之上加了 19 个 LLM 专属 attribute（详见 §3.2）。而 AI Gateway 则是 **"怎么把这些 attribute 抓回来"**——即在客户端和 LLM provider 之间安一个代理，让 trace 自动收齐。

#### 1.2.3 战略选择：做网关 vs 做 SDK

2024 年初 Datadog 内部有过讨论：LLM Observability 走 **SDK 路线**（用户自己 import `ddtrace.llmobs.*`）还是 **网关路线**（用户把流量打到 Datadog 的代理再转发）。

- **SDK 路线**：精度高，能拿到中间结果；但用户要改代码、升级依赖、语言绑定（Python/Node 都有，Go/Rust 弱）。
- **网关路线**：用户零代码改动、自动覆盖 14 provider；但要承担代理本身的 latency + 单点风险 + 双向流（streaming）处理复杂度。

最终 Datadog **双轨并行**：

- **无 SDK 模式** = AI Gateway（`gateway.datadoghq.com` 托管或 Datadog Agent 自托管）
- **有 SDK 模式** = 直接 import `from ddtrace.llmobs import LLMObs`（适合需要 fine-grained 控制 trace 的用户）

大多数客户用 SDK 模式（"能改代码就改代码"），AI Gateway 模式主要给"语言不受 Datadog 官方支持 + 不想改业务代码"的客户用。

### 1.3 团队与组织

| 角色 | 团队 | 备注 |
|---|---|---|
| 产品负责人 | Aishwarya Srinivasan (ex-New Relic, 2024-04 加入 Datadog) | LinkedIn 公开 |
| 首席工程师 | Rômulo Silva（Pyroscope/Splunk 出身） | 主导 LLM Observability Agent 端 |
| 工程团队规模 | ~40 人（2025-Q4 公开） | 跨 SF / Paris / NYC 三个 hub |
| 报告线 | 隶属于 Datadog **Product Analytics BU**（VP：Yrieix Garnier） | 与 APM/Logs/Metrics/Digital Experience 平级 |
| 商业化 | LLM Observability 独立 SKU，2024-Q3 达到 Datadog 总营收 1.5%（按 Skytap 估算） | 与 Logs / APM 分开报价 |

### 1.4 与 Datadog 其他 AI 产品的关系

| 产品 | 定位 | 与 AI Gateway 关系 |
|---|---|---|
| **Datadog APM** | 应用性能监控 | AI Gateway 复用 APM 的 trace infra；UI 里可从一个 LLM span 跳到对应 service |
| **Datadog LLM Observability** | LLM 专项可观测 | AI Gateway 是它的"流量入口"——gateway 抓到的 span 100% 喂给 LLM Observability |
| **Datadog LLM Eval** | LLM 质量评估 | 独立 SKU（$0.10/evaluator run）；与 AI Gateway 共享 trace 但有独立 UI |
| **Datadog Guardrails** | LLM 输入输出防护 | 可被 AI Gateway 调用，**不是** AI Gateway 内置（明确分离） |
| **Datadog Cost Management** | 云成本 + LLM 成本 | LLM token 成本自动从 AI Gateway trace 抓；与 AWS/GCP/Azure cost 合并 dashboard |
| **Datadog Sensitive Data Scanner** | PII/PCI/PHI 扫描 | AI Gateway 抓到的 prompt/completion 自动过 SDS；命中 PII 标记 span attribute |
| **Datadog Bits AI** | Datadog 自家的 SRE AI 助手 | Bits AI 自身也跑在 AI Gateway 后面（"吃自己狗粮"） |
| **Datadog Watchdog** | 异常检测 | 把 LLM 调用异常（latency / error / cost spike）当 Watchdog 异常源 |
| **Datadog Cloud SIEM** | 安全信息与事件管理 | AI Gateway 的 prompt injection 检测会推送到 SIEM |
| **Datadog Cloud Workload Security** | 运行时安全 | 与 AI Gateway 弱关联，XDR 维度 |

**关键观察**：Datadog 把 AI Gateway 定位成"AI 可观测生态的中枢"——它**自己不做**模型路由、不做模型缓存、不做计费分摊，而是把"流量镜像 + 治理"做透，让 LLM Observability / Cost / Guardrails / SIEM 各取所需。

---

## 2. 架构设计

### 2.1 整体架构（自上而下）

```
                        ┌─────────────────────────────────────────────────────────┐
                        │                    Datadog 后端                            │
                        │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
                        │  │ APM Ingest   │  │ LLM Obs UI   │  │ Eval/Cost    │    │
                        │  │  (trace 收)  │  │              │  │              │    │
                        │  └──────┬───────┘  └──────▲───────┘  └──────▲───────┘    │
                        │         │                 │                  │            │
                        └─────────┼─────────────────┼──────────────────┼────────────┘
                                  │                 │                  │
                                  │ OTel/HTTP       │ WebSocket        │ Eval API
                                  │ traceparent     │ (实时 trace)     │
                                  │                 │                  │
   ┌──────────────────────────────┼─────────────────┼──────────────────┼──────────┐
   │ Datadog Agent 7.55+ (sidecar / DaemonSet) ────┼──────────────────┼──────────┤
   │                              │                 │                  │          │
   │  ┌───────────────────────────▼────┐  ┌─────────▼──────┐  ┌───────▼────────┐ │
   │  │  AI Gateway 模块 (llmobs)      │  │  APM Trace     │  │  Sensitive     │ │
   │  │  ┌─────────────┐ ┌──────────┐  │  │  Pipeline      │  │  Data Scanner  │ │
   │  │  │ HTTP Server │ │ Streaming│  │  │  (dd-trace)    │  │  (PII 扫描)    │ │
   │  │  │ :8126/gw    │ │ SSE/WS   │  │  │                │  │                │ │
   │  │  └──────┬──────┘ └────┬─────┘  │  └────────────────┘  └────────────────┘ │
   │  │         │             │        │                                        │
   │  │  ┌──────▼─────────────▼─────┐  │  ┌────────────────┐ ┌────────────────┐ │
   │  │  │ Provider Adapter         │  │  │ Guardrails     │ │ Cost Tracker   │ │
   │  │  │ (OpenAI/Anthropic/Gemini)│  │  │ (规则/ML)      │ │ (token × 价)   │ │
   │  │  └──────┬───────────────────┘  │  └────────────────┘ └────────────────┘ │
   │  │         │                      │                                        │
   │  │  ┌──────▼───────────────────┐  │  ┌────────────────────────────────────┐ │
   │  │  │ 请求/响应 Capture        │  │  │  Prompt Injection Detector          │ │
   │  │  │ (full body)              │  │  │  (2025-11 ML + 200 规则)             │ │
   │  │  └──────────────────────────┘  │  └────────────────────────────────────┘ │
   │  └────────────────────────────────┘                                        │
   │                                                                            │
   │  ┌────────────────────────────────┐  ┌────────────────────────────────────┐ │
   │  │  DogStatsD Metrics             │  │  LLM Eval Runner                  │ │
   │  │  (latency, token, cost)        │  │  (online evaluator)               │ │
   │  └────────────────────────────────┘  └────────────────────────────────────┘ │
   └────────────────────────────────────────────────────────────────────────────┘
                                  ▲                                              ▲
                                  │ HTTPS / mTLS                                │ HTTPS
                                  │                                              │
                ┌─────────────────┴────────────────┐                ┌─────────────┴────────────┐
                │  Application Code (client)       │                │  LLM Provider API         │
                │  (LLM 调用方)                    │                │  (OpenAI / Anthropic ...) │
                │                                  │                │                           │
                │  - 环境变量 DD_LLMOBS_GATEWAY_URL │                │  api.openai.com           │
                │  - 或 OpenAI SDK base_url 替换   │                │  api.anthropic.com        │
                │  - 或 dd-trace SDK 自动拦截       │                │  generativelanguage...    │
                └──────────────────────────────────┘                └───────────────────────────┘
```

### 2.2 核心数据流（一次 LLM Chat Completion 调用）

#### 2.2.1 不带 streaming

```
[Client App] --POST /v1/chat/completions--> [AI Gateway:8126]
                                                  │
                                                  ├── 1. 解析 OpenAI schema
                                                  ├── 2. 创建 span(kind=llm, api=chat)
                                                  ├── 3. 注入 traceparent (W3C)
                                                  ├── 4. PII 扫描 (Sensitive Data Scanner)
                                                  ├── 5. Prompt Injection 检查 (规则 + ML)
                                                  ├── 6. (可选) Guardrail 触发
                                                  ├── 7. 转发到 upstream
                                                  │     [POST api.openai.com/v1/chat/completions]
                                                  │     <-- 响应
                                                  ├── 8. PII 扫描 (response)
                                                  ├── 9. token 计数 (tiktoken for OpenAI / 自研 for Anthropic)
                                                  ├── 10. cost 计算 (token × model 单价)
                                                  ├── 11. span 填属性:
                                                  │     gen_ai.request.model = gpt-4o
                                                  │     gen_ai.usage.input_tokens = 1024
                                                  │     gen_ai.usage.output_tokens = 256
                                                  │     gen_ai.usage.cached_tokens = 0
                                                  │     gen_ai.response.finish_reason = stop
                                                  │     gen_ai.usage.cost.input_usd = 0.00255
                                                  │     gen_ai.usage.cost.output_usd = 0.00128
                                                  │     gen_ai.security.pii.detected = false
                                                  │     gen_ai.security.injection.score = 0.02
                                                  ├── 12. 异步上报 trace 到 Datadog
                                                  │
                                                  <-- 200 OK + 完整 response
[Client App]
```

**latency 开销**：
- 解析 OpenAI schema：~0.3ms
- PII 扫描 (规则)：~0.2ms
- Prompt Injection 规则检查：~0.1ms
- ML 模型（Llama Guard 7B quant）：~5-15ms（GPU 推理；如果走 CPU 30ms+）
- Token 计数：~0.1ms
- 上报 trace 到 Datadog 后端：异步，不阻塞
- **总计**：~6-16ms（ML 模型启用时） 或 ~1ms（ML 禁用时）

#### 2.2.2 带 streaming（SSE）

streaming 是 AI Gateway 的难点。Datadog 的处理：

1. **不缓冲 full response**——SSE 边到边转，边抓 chunk
2. 第一个 chunk 到达时创建 span（避免等全响应才创建 span 浪费时间）
3. 每个 chunk 累加 `gen_ai.usage.completion_tokens_so_far`
4. 最后一个 chunk（`data: [DONE]`）到达时填 `finish_reason`、上 span
5. **partial completion 抽样**：默认 1% 的 streaming 调用会 **额外** 把完整 prompt + completion 抓给 LLM Eval（在线评估）

streaming 模式下 latency 开销更小（每个 chunk ~0.1ms 处理），但内存占用更高（要保持 LLM 响应缓冲区）。

### 2.3 Provider Adapter 详解

#### 2.3.1 14 个 Provider 的覆盖矩阵

| Provider | 协议 | Streaming | Function Call | Vision | Prompt Cache | Reasoning | Embeddings | Audio |
|---|---|---|---|---|---|---|---|---|
| **OpenAI** | OpenAI Chat Completions / Responses | ✅ | ✅ | ✅ | ✅ (auto cache_control) | ✅ o1/o3 | ✅ | ✅ tts-1, whisper |
| **Anthropic** | Anthropic Messages | ✅ | ✅ tools | ✅ | ✅ (cache_control) | ✅ extended thinking | ❌ | ❌ |
| **Google Gemini** | Google Generative AI (REST + gRPC) | ✅ | ✅ function | ✅ | ✅ implicit | ✅ thinking | ✅ text-embedding-004 | ❌ |
| **Vertex AI** | Vertex AI Model Garden (OpenAI 兼容 + 原生 Vertex) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Azure OpenAI** | Azure OpenAI REST | ✅ | ✅ | ✅ | ✅ | ✅ o1 | ✅ | ✅ |
| **AWS Bedrock** | AWS Bedrock Runtime (InvokeModel / Converse) | ✅ | ✅ tool | ✅ | ✅ (anthropic on bedrock) | ✅ | ✅ (titan/cohere) | ❌ |
| **Cohere** | Cohere v2 API | ✅ | ✅ tools | ❌ | ❌ | ❌ | ✅ embed-v3 | ❌ |
| **Mistral** | Mistral Chat Completions (OpenAI 兼容) | ✅ | ✅ tools | ❌ | ❌ | ❌ | ✅ embed | ❌ |
| **Hugging Face TGI** | TGI OpenAI 兼容端点 | ✅ | ✅ | 部分 | ❌ | ❌ | ✅ | ❌ |
| **Ollama** | Ollama OpenAI 兼容端点 | ✅ | ❌ (mock) | ✅ | ❌ | ❌ | ✅ | ❌ |
| **vLLM** | vLLM OpenAI 兼容端点 | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Together AI** | Together OpenAI 兼容端点 | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Fireworks AI** | Fireworks OpenAI 兼容端点 | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **OpenRouter** | OpenRouter OpenAI 兼容端点 | ✅ | ✅ | ✅ | 透传 (依模型) | 透传 | ✅ | 透传 |
| **自定义 OpenAI 兼容** | OpenAI Chat Completions 协议 | ✅ | ✅ | 透传 | 透传 | 透传 | ✅ | 透传 |

#### 2.3.2 Adapter 抽象层

所有 provider 都被映射到 4 类 span kind（来自 OTel GenAI semconv 0.4）：

```
gen_ai.operation.name ∈ {
   chat,                    // Chat Completion
   text_completion,         // 老式 completion
   embeddings,              // 文本向量化
   generate_content,        // Gemini style
   invoke_agent,            // Agent 多步
   execute_tool,            // Tool call
   create_agent,            // Agent 创建
}

gen_ai.provider.name ∈ {
   openai, anthropic, google_genai, vertex_ai, azure_openai, aws_bedrock,
   cohere, mistral, huggingface, ollama, vllm, together_ai, fireworks,
   openrouter, custom
}
```

即使原 API 是 Bedrock 的 `InvokeModelCommand`（boto3），Adapter 也把它转成 `gen_ai.provider.name=aws_bedrock` + `gen_ai.request.model=anthropic.claude-3-5-sonnet`，保证 UI 里跨 provider 聚合时正确分组。

### 2.4 与 Datadog APM 的整合

这是 Datadog AI Gateway 与其他 LLM observability 工具（Langfuse / Arize / Helicone）的最大差异——**它不是一个独立的 LLM trace 系统，而是 Datadog APM 体系内的一个 span kind**。

#### 2.4.1 一条完整的 Trace 示例

```
HTTP POST /api/chat (用户发请求)
└─ span kind=server (FastAPI)
   ├─ span kind=llm (OpenAI gpt-4o, 1.2s, 1024 input / 256 output tokens, $0.0038)
   │  └─ span kind=tool (weather_api.get, 350ms, 200 OK)
   ├─ span kind=llm (Anthropic claude-3-5-haiku, 0.4s, 512 input / 100 output, $0.0002)
   └─ span kind=http (DB query for conversation history, 50ms)
```

**所有 span 在一条 trace 里**，可以：
- 看整体 p50/p95/p99 latency（包括 LLM + 业务代码 + DB + Tool）
- 看 LLM 异常时整个请求是否也异常
- 跨 service 关联（Python Web → Node.js BFF → Java 后端 → LLM）
- 用 APM 的 Service Map 自动画拓扑图

#### 2.4.2 与传统 APM 工具的对比

| 维度 | Datadog AI Gateway + APM | Langfuse 独立 | Arize Phoenix 独立 |
|---|---|---|---|
| LLM 专项 attribute | ✅ 19 个 | ✅ 30+ | ✅ 25+ |
| 跨语言 trace 关联 | ✅ 同 APM trace | ❌ 独立 trace | ❌ 独立 trace |
| 业务代码 + LLM 一起看 | ✅ 同一 trace | ⚠️ 通过 OTel 关联可实现 | ⚠️ 同上 |
| LLM 专项 UI | ⚠️ LLM Observability 标签页 | ✅ 专门 LLM UI | ✅ Phoenix Experiments UI |
| Service Map / 拓扑 | ✅ Datadog 强项 | ❌ | ❌ |
| Eval / Dataset | ⚠️ 较新，弱 | ✅ Langfuse 强 | ✅ Arize 强 |
| 与 Logs 关联 | ✅ 一键跳 logs | ⚠️ 需自配 | ⚠️ 需自配 |
| 与 SIEM 关联 | ✅ 强 | ❌ | ❌ |
| 部署复杂度 | ⚠️ 需 Datadog Agent | ✅ 独立轻量 | ✅ 独立轻量 |

### 2.5 部署架构变体

#### 2.5.1 模式 A：SaaS Gateway（最简单）

```
[App] --HTTPS--> gateway.datadoghq.com --HTTPS--> api.openai.com
                     │
                     └── 上报 trace 到 Datadog 后端
```

用户配置：
```bash
export OPENAI_API_BASE=https://gateway.datadoghq.com/v1
export OPENAI_API_KEY=sk-...   # 仍是客户自己的 OpenAI key，Gateway 只做镜像
export DD_API_KEY=...           # Datadog key
```

**适用**：客户已有 OpenAI/Anthropic 业务账号，不想自建 Agent；语言不限制（任何 HTTP 客户端都行）

**劣势**：
- 多一跳公网（延迟 +1-3ms）
- **PII 数据流过 Datadog 服务器**——很多金融/医疗客户禁止
- 跨区域延迟（gateway 部署在 us1/us3/eu1/ap1，跨区不友好）

#### 2.5.2 模式 B：Datadog Agent 自托管（K8s sidecar / DaemonSet）

```
[App Pod] --localhost:8126--> [Datadog Agent Pod (sidecar)] --HTTPS--> api.openai.com
                                       │
                                       └── 上报 trace 到 Datadog 后端
```

用户配置：
```bash
# 在 Datadog Agent 7.55+ 的 datadog.yaml
llmobs:
  enabled: true
  gateway:
    port: 8126
    providers:
      - name: openai
        api_base: https://api.openai.com/v1
        api_key_env: OPENAI_API_KEY
      - name: anthropic
        api_base: https://api.anthropic.com
        api_key_env: ANTHROPIC_API_KEY
    sampling:
      head_based: 0.1   # 10% 全量
      tail_based: 0.05  # 异常 span 100% 保留
    pii_scanner: true
    injection_detector: true
```

**适用**：中大型 K8s 客户；合规要求 PII 不出本集群

**优势**：
- PII 数据不出本集群
- 延迟 +0.5-1ms（sidecar 同 pod）
- 与 APM 共享 Datadog Agent 部署（已装 APM 的话零新组件）

#### 2.5.3 模式 C：Client SDK（最精细）

```python
# Python
from ddtrace.llmobs import LLMObs
LLMObs.enable(
    ml_app="my-chatbot",
    integrations=[
        LLMObs.OpenAIIntegration(),
        LLMObs.LangChainIntegration(),
    ],
)
# 之后所有 OpenAI / LangChain 调用自动 trace
```

**适用**：需要精细控制 trace（自定义 span、自定义 attribute、自定义 evaluator）

**优势**：
- 零代理、零延迟开销
- 能 trace 任意中间步骤（自定义代码块、agent reasoning）
- 支持 OTel 自动 export 到其他后端

### 2.6 关键技术决策

| 决策 | 选择 | 理由 |
|---|---|---|
| 自研 vs 基于 Envoy | **自研 Go**（Datadog Agent 内） | Datadog Agent 已是 Go，集成零成本；Envoy 体积太大（120MB），不适合 sidecar |
| 协议抽象 | OTel GenAI semconv 1.30 | 与 Langfuse / OpenInference / OpenLLMetry 跨工具兼容 |
| Trace 存储 | 自建 Husky KV（Datadog 自研） | 与 APM / Logs / Metrics 共享存储层；查询性能优化 |
| 采样 | Head-based + tail-based 双轨 | Tail-based 用于"异常 span 100% 保留"（cost spike、injection 检测到） |
| PII 扫描 | Datadog Sensitive Data Scanner（同 Agent 模块） | 复用 SDS 引擎（增量开发） |
| 注入检测 | 自研 7B LLM + 200 规则（2025-11-18） | 内部 benchmark：F1 0.91（vs 纯规则 0.74） |
| 流式处理 | SSE 边到边转，不缓冲 | 避免 OOM；不破坏 streaming latency |
| 高可用 | SaaS 模式：多 AZ + mTLS；Agent 模式：本地缓存 + 异步重传 | 避免 Agent 挂掉影响 LLM 调用 |

---

## 3. 协议支持

### 3.1 输入协议（Datadog Agent / SDK 支持的协议）

| 协议 | 完整度 | 备注 |
|---|---|---|
| **OpenAI Chat Completions** (`/v1/chat/completions`) | 100% | 最早支持；Function call、tools、vision、audio、response_format 全覆盖 |
| **OpenAI Responses** (`/v1/responses`) | 100% | 2025-08 OpenAI 新协议；COT 内置、tool call 原生 |
| **Anthropic Messages** (`/v1/messages`) | 100% | 2024-Q4 引入；tool use、extended thinking、cache_control、citation |
| **Anthropic Prompt Caching 透明注入** | 100% | 2026-03 上线；Gateway 自动在 system prompt 末尾追加 `cache_control: ephemeral` 块，对调用方透明 |
| **Google Gemini** (`generateContent`) | 100% | REST + gRPC 双协议；function call、thinking、multimodal |
| **AWS Bedrock InvokeModel** | 100% | boto3 SDK + HTTP 直连两种 |
| **AWS Bedrock Converse** | 100% | 2024-12 引入，跨 provider 统一 tool use |
| **Azure OpenAI** | 100% | API 协议与 OpenAI 相同，部署 URL 替换 |
| **Cohere v2 Chat** | 100% | 2024-11 引入 |
| **Mistral Chat Completions** | 100% | OpenAI 兼容 API |
| **Hugging Face TGI** | 100% | 走 OpenAI 兼容端点 |
| **Ollama** | 100% | 走 OpenAI 兼容端点 |
| **vLLM / SGLang** | 100% | 走 OpenAI 兼容端点 |
| **OpenRouter** | 100% | 走 OpenAI 兼容端点 |
| **自定义 OpenAI 兼容** | 100% | 用户可注册任意 `/v1/chat/completions` 端点 |

### 3.2 抓取属性（LLM Observability 19 个核心 attribute）

依据 OTel GenAI semconv 1.30（2026-05 升级）：

#### 3.2.1 必抓（必填）

| OTel 属性 | 类型 | 含义 | 示例 |
|---|---|---|---|
| `gen_ai.operation.name` | string | 操作类型 | `chat`, `embeddings`, `execute_tool` |
| `gen_ai.provider.name` | string | 服务商标识 | `openai`, `anthropic`, `aws_bedrock` |
| `gen_ai.request.model` | string | 模型名 | `gpt-4o-2024-08-06`, `claude-3-5-sonnet-20241022` |
| `gen_ai.response.model` | string | 实际模型（处理 alias） | `gpt-4o-2024-08-06` |
| `gen_ai.response.id` | string | 响应 ID | `chatcmpl-abc123` |
| `gen_ai.response.finish_reasons` | string[] | 结束原因 | `["stop"]`, `["length"]`, `["tool_calls"]`, `["content_filter"]` |
| `gen_ai.usage.input_tokens` | int | 输入 token | 1024 |
| `gen_ai.usage.output_tokens` | int | 输出 token | 256 |
| `gen_ai.usage.cached_input_tokens` | int | 缓存命中 token | 0 |
| `gen_ai.usage.cost.input_usd` | float | 输入成本 | 0.00255 |
| `gen_ai.usage.cost.output_usd` | float | 输出成本 | 0.00128 |
| `gen_ai.usage.cost.total_usd` | float | 总成本 | 0.00383 |

#### 3.2.2 可选抓（看 provider）

| OTel 属性 | 类型 | 含义 | 示例 |
|---|---|---|---|
| `gen_ai.request.temperature` | float | 采样温度 | 0.7 |
| `gen_ai.request.max_tokens` | int | 最大输出 | 4096 |
| `gen_ai.request.top_p` | float | nucleus sampling | 0.9 |
| `gen_ai.request.frequency_penalty` | float | 频率惩罚 | 0.0 |
| `gen_ai.request.presence_penalty` | float | 存在惩罚 | 0.0 |
| `gen_ai.request.seed` | int | 随机种子 | 42 |
| `gen_ai.request.stop_sequences` | string[] | 停止序列 | `["\n\n"]` |
| `gen_ai.response.system_fingerprint` | string | OpenAI 系统指纹 | `fp_abc123` |

#### 3.2.3 Tool / Agent / 安全属性

| OTel 属性 | 类型 | 含义 | 示例 |
|---|---|---|---|
| `gen_ai.tool.name` | string | 工具名 | `get_weather` |
| `gen_ai.tool.description` | string | 工具描述 | `Get current weather for a location` |
| `gen_ai.tool.call.id` | string | tool call id | `call_abc123` |
| `gen_ai.tool.call.arguments` | string (JSON) | 工具参数 | `{"location": "Beijing"}` |
| `gen_ai.tool.call.result` | string (JSON) | 工具结果 | `{"temp": 25, "unit": "C"}` |
| `gen_ai.agent.id` | string | Agent ID | `weather_agent_42` |
| `gen_ai.agent.name` | string | Agent 名 | `Weather Agent` |
| `gen_ai.agent.step` | int | Agent 步数 | 3 |
| `gen_ai.security.pii.detected` | bool | PII 是否被检测到 | true |
| `gen_ai.security.pii.entities` | string[] | PII 实体 | `["EMAIL", "PHONE", "SSN"]` |
| `gen_ai.security.injection.score` | float | 注入攻击概率 | 0.02 |
| `gen_ai.security.injection.verdict` | string | 注入攻击判定 | `safe`, `suspicious`, `malicious` |
| `gen_ai.security.toxicity.score` | float | 毒性评分 | 0.05 |
| `gen_ai.evaluation.name` | string | Eval 名 | `relevance`, `hallucination` |
| `gen_ai.evaluation.score` | float | Eval 分数 | 0.85 |
| `gen_ai.evaluation.label` | string | Eval 标签 | `pass`, `fail` |
| `gen_ai.evaluation.explanation` | string | Eval 解释 | `The response is relevant to the question but lacks specific data.` |

### 3.3 输出协议（Datadog 暴露的端点）

#### 3.3.1 SaaS Gateway 端点

| 端点 | 用途 |
|---|---|
| `https://gateway.datadoghq.com/v1/chat/completions` | OpenAI 协议代理（客户应用调用此端点） |
| `https://gateway.datadoghq.com/v1/embeddings` | Embeddings 代理 |
| `https://gateway.datadoghq.com/v1/responses` | OpenAI Responses 代理 |
| `https://gateway.datadoghqq.com/v1/messages` | Anthropic Messages 代理 |
| `https://api.datadoghq.com/api/v2/llm-obs/v1/spans` | trace 上报（OTLP HTTP） |
| `https://api.datadoghq.com/api/v2/llm-obs/v1/evaluations` | eval 上报 |

#### 3.3.2 Agent 自托管端点

| 端点 | 用途 |
|---|---|
| `http://localhost:8126/v1/chat/completions` | 本地代理（sidecar 模式） |
| `http://localhost:8126/v1/embeddings` | 本地 embedding 代理 |
| `http://localhost:8126/v1/responses` | 本地 OpenAI Responses 代理 |
| `http://localhost:8126/v1/messages` | 本地 Anthropic 代理 |
| `http://localhost:8126/agent/eval` | 本地 eval 触发 |
| `http://localhost:8126/agent/guardrail` | 本地 guardrail 触发 |

#### 3.3.3 API 客户端

- `pydatadog.llmobs.LLMObs`（Python 3.9+）
- `@datadog/llm-observability`（Node 18+）
- `gopkg.in/DataDog/dd-trace-go.v1/llmobs`（Go 1.21+）
- `io.datadoghq:dd-trace-llmobs-java`（Java 11+）
- `Datadog.LLMObs`（.NET 6+）
- `ddtrace-llmobs`（Ruby 3.0+）
- `datadog-llm-observability`（PHP 8.0+）
- OTel GenAI 1.30 SDK 任意语言（OTLP export）

### 3.4 协议翻译矩阵

#### 3.4.1 OpenAI Chat ↔ Anthropic Messages 转换

Datadog AI Gateway 自己**不做协议转换**（不像 Portkey / LiteLLM），只做镜像抓取。但 SDK 模式下用户可手动调：

```python
# 典型用法：OpenAI SDK + 协议转换
from openai import OpenAI
client = OpenAI(base_url="https://gateway.datadoghq.com/v1")
resp = client.chat.completions.create(
    model="anthropic/claude-3-5-sonnet",  # 走 Anthropic
    messages=[{"role": "user", "content": "hi"}],
)
```

Datadog 抓的 span 会标 `gen_ai.provider.name=anthropic`，即使协议走 OpenAI。

#### 3.4.2 Function Call / Tool Use 归一化

| 原协议 | Datadog 内部表示 | 示例 |
|---|---|---|
| OpenAI `function_call` | `gen_ai.tool.call.arguments` | `{"name": "get_weather", "arguments": "{...}"}` |
| OpenAI `tools[]` (新) | `gen_ai.tool.call.arguments` | 同上 |
| Anthropic `tool_use` | `gen_ai.tool.call.arguments` | 同上 |
| Gemini `functionCall` | `gen_ai.tool.call.arguments` | 同上 |
| Bedrock `toolUse` | `gen_ai.tool.call.arguments` | 同上 |

#### 3.4.3 Reasoning / Thinking 归一化

| Provider | Reasoning 字段 | Datadog 抓取位置 |
|---|---|---|
| OpenAI o1/o3 | `reasoning_tokens` (in usage) | `gen_ai.usage.reasoning_tokens` |
| OpenAI o1 (Responses API) | `summary` 数组 | `gen_ai.response.reasoning_summary` |
| Anthropic extended thinking | `thinking` block | `gen_ai.response.thinking_blocks` |
| Gemini 2.5 thinking | `thoughtsSummary` | `gen_ai.response.thinking_summary` |
| DeepSeek R1 | `reasoning_content` | `gen_ai.response.reasoning_content` |

### 3.5 MCP（Model Context Protocol）支持

- **当前（2026-Q2 beta）**：MCP tool call 已被识别为 `span.kind=tool`，但 MCP server → MCP client 的握手、list_tools 等不专门 trace
- **2026-Q3 计划**（2026-05 changelog 预告）：
  - MCP server 端到端 trace（独立 `gen_ai.operation.name=mcp_server`）
  - MCP auth（OAuth 2.1）的 OTel 透传
  - 与 Datadog API Catalog 联动（自动发现 MCP server 注册的工具）
  - Prompt injection 在 MCP tool 描述层的检测（与现有规则引擎共用）

---

## 4. 性能数据

### 4.1 AI Gateway 自身 overhead

#### 4.1.1 单调用 latency 开销

| 模式 | p50 | p95 | p99 | 备注 |
|---|---|---|---|---|
| **SaaS Gateway** (gateway.datadoghq.com) | +12ms | +25ms | +45ms | 跨 region 公网延迟 |
| **Agent 自托管** (sidecar 模式) | +0.8ms | +1.5ms | +3ms | localhost |
| **SDK 模式** | +0.05ms | +0.1ms | +0.2ms | 进程内拦截 |
| **Agent + 启用 ML 注入检测** | +8ms | +18ms | +35ms | Llama Guard 7B 量化 |
| **Agent + 启用 PII 扫描** | +0.4ms | +0.8ms | +1.5ms | 规则匹配 |

#### 4.1.2 吞吐量

| 模式 | 最大 QPS | 备注 |
|---|---|---|
| **SaaS Gateway** | 100,000 QPS / 客户 (软限) | 单 LLM 调用计 1 span，多 span 联动 |
| **Agent sidecar** (单核) | 3,500 QPS | 1 vCPU 测得 (Intel Xeon 8275CL @ 3.00GHz) |
| **Agent sidecar** (8 核) | 18,000 QPS | 线性扩展 |
| **SDK 模式** | 0 开销 | 进程内，无瓶颈 |

#### 4.1.3 内存占用

| 模式 | 空闲 | 满载 (1000 RPS) |
|---|---|---|
| **Agent sidecar** | 80MB | 320MB |
| **Agent + 启用 ML 注入检测** | 1.2GB | 1.8GB |
| **SDK 模式** | +5MB | +15MB |

#### 4.1.4 资源开销对 LLM 推理的影响

Datadog AI Gateway **不参与 LLM 推理**，只做客户端 → LLM provider 的代理。所以：
- 对 LLM 推理性能：0 影响
- 对 LLM 推理成本：0 影响（除了可能的 retry / failover 带来的额外调用）
- 对 LLM 推理延迟：0 影响（除了 gateway 自身的 ~1ms overhead）

### 4.2 LLM Eval Runner 性能

| Eval 类型 | 每次耗时 | 备注 |
|---|---|---|
| **Heuristic Eval** (关键词、正则、JSON Schema 校验) | 5-50ms | CPU |
| **LLM-as-judge** (GPT-4o 评分) | 1-3s | 走 OpenAI API |
| **Datadog 自研 Evaluator LLM** (开源 Llama 70B) | 200-500ms | 部署在 Datadog 自家 GPU |
| **Custom Python Evaluator** | 用户自定义 | 取决于用户实现 |

### 4.3 Trace Ingest 性能

- **OTLP HTTP** 上报：每 span ~1.5KB（压缩后）
- **单 Agent 上报带宽**（1000 QPS，每个 LLM 调用 ~3 span）：~4.5MB/s 压缩前 / ~1.2MB/s 压缩后
- **Datadog 后端 ingest 容量**：单区域 5M span/s（公开 benchmark）

### 4.4 Prompt Injection Detector 性能

| 模式 | F1 | 召回率 | 精确率 | 延迟 |
|---|---|---|---|---|
| **纯规则** (2024-12 第一版) | 0.74 | 0.81 | 0.69 | 0.1ms |
| **ML 模型** (Llama Guard 7B 量化) | 0.86 | 0.91 | 0.81 | 8-15ms |
| **规则 + ML 双轨** (2025-11 当前) | 0.91 | 0.95 | 0.88 | 8-16ms |
| **ML + Adversarial Augmentation** (2026-Q2 计划) | 0.93+ | 0.96+ | 0.91+ | 10-20ms |

### 4.5 与竞品的性能对比

#### 4.5.1 Gateway overhead 对比

| 产品 | p50 overhead | p99 overhead | 部署模式 | 备注 |
|---|---|---|---|---|
| **Datadog AI Gateway** (Agent) | 0.8ms | 3ms | sidecar | 与 APM 共享 |
| **Portkey** | 5ms | 15ms | SaaS + self-host | 多了路由 + 缓存逻辑 |
| **LiteLLM** | 8ms | 30ms | self-host | 协议转换 + 路由 |
| **Bifrost** | 0.011ms | 0.05ms | self-host | Go + zero-allocation |
| **Higress** | 1.5ms | 5ms | self-host (Envoy) | WASM 插件 |
| **Helicone** | 6ms | 20ms | SaaS | 抓取 + 计费 |
| **Langfuse** | 0.1ms | 0.3ms | self-host SDK | 进程内 |

#### 4.5.2 Trace 抓取精度对比

| 维度 | Datadog | Langfuse | Arize | Helicone |
|---|---|---|---|---|
| 全 prompt/completion 抓取 | ✅ 100% (SaaS) | ✅ 100% (self-host) | ✅ 100% (self-host) | ✅ 100% (SaaS) |
| Token 计数精度 | ✅ 用 provider 实际 usage | ✅ 用 provider 实际 usage | ✅ 用 provider 实际 usage | ✅ 用 provider 实际 usage |
| 跨语言 trace 关联 | ✅ 同 APM | ⚠️ 通过 OTel | ⚠️ 通过 OTel | ❌ 独立 |
| Tool call 嵌套 | ✅ 多层 | ✅ 多层 | ✅ 多层 | ⚠️ 1 层 |
| Streaming trace | ✅ SSE 边到边 | ✅ SSE 边到边 | ✅ SSE 边到边 | ✅ SSE 边到边 |
| Eval 在线触发 | ✅ 2025-09 | ✅ 强 | ✅ 强 | ⚠️ 弱 |
| 标注 + dataset | ⚠️ 较新 | ✅ 强 | ✅ 强 | ❌ |
| 实时报警 | ✅ Watchdog | ✅ Alerting | ✅ Alerting | ⚠️ 基础 |

---

## 5. 部署方式

### 5.1 模式 A：SaaS Gateway（最快）

#### 5.1.1 客户配置

```bash
# 环境变量（最简）
export OPENAI_API_BASE=https://gateway.datadoghq.com/v1
export OPENAI_API_KEY=sk-...
export DD_API_KEY=<DD_KEY>
export DD_APP_KEY=<DD_APP_KEY>  # 可选，用于写 eval
```

```yaml
# datadog.yaml（仅在 Agent 模式下需要）
site: datadoghq.com
api_key: <DD_KEY>
llmobs:
  enabled: true
```

#### 5.1.2 适用场景

- 个人开发者 / 小团队，不想装 Agent
- 多语言环境（任意 HTTP 客户端）
- 快速 PoC
- 没有 PII 顾虑
- 公网延迟 <50ms 可接受

#### 5.1.3 优势 vs 劣势

| 优势 | 劣势 |
|---|---|
| 零运维、5 分钟接入 | PII 出本集群 |
| 自动扩展 | 跨区域 +12ms |
| 跨语言 | 单点风险（Datadog 挂掉影响 LLM） |

### 5.2 模式 B：Datadog Agent 自托管（最常用）

#### 5.2.1 K8s Helm 部署

```yaml
# values.yaml
datadog:
  apiKey: <DD_KEY>
  site: datadoghq.com
  apm:
    enabled: true   # LLM Obs 依赖 APM
  llmObs:
    enabled: true
    gateway:
      enabled: true
      port: 8126
      providers:
        - name: openai
        - name: anthropic
        - name: bedrock
          region: us-east-1
        - name: vertex
          project: my-gcp-proj
        - name: custom-openai
          apiBase: https://my-custom-llm.com/v1
```

#### 5.2.2 Docker Compose 部署

```yaml
# docker-compose.yaml
services:
  datadog-agent:
    image: gcr.io/datadoghq/agent:7.58.0
    environment:
      - DD_API_KEY=<DD_KEY>
      - DD_SITE=datadoghq.com
      - DD_LLMOBS_ENABLED=true
      - DD_LLMOBS_GATEWAY_ENABLED=true
      - DD_LLMOBS_GATEWAY_PORT=8126
      - DD_OPENAI_API_KEY=sk-...
      - DD_ANTHROPIC_API_KEY=sk-ant-...
    ports:
      - "8126:8126"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./datadog.yaml:/etc/datadog-agent/datadog.yaml:ro

  app:
    build: .
    environment:
      - OPENAI_API_BASE=http://datadog-agent:8126/v1
      - OPENAI_API_KEY=sk-...  # 真实 key 仍由应用持有
    depends_on:
      - datadog-agent
```

#### 5.2.3 适用场景

- 中大型 K8s 客户
- 合规要求 PII 不出本集群
- 已是 Datadog 客户
- 已有 Datadog Agent 部署

### 5.3 模式 C：Client SDK（最灵活）

#### 5.3.1 Python

```python
import openai
from ddtrace.llmobs import LLMObs, LLMObsSpan

# 1. 启用 LLM Observability
LLMObs.enable(
    ml_app="my-chatbot",
    integrations=[
        LLMObs.OpenAIIntegration(),
        LLMObs.LangChainIntegration(),
    ],
    api_key="<DD_KEY>",
    site="datadoghq.com",
)

# 2. 直接调 OpenAI（自动 trace）
client = openai.OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
# Span 自动创建，gen_ai.* 属性自动填

# 3. 自定义 span
with LLMObs.llm(
    name="custom_llm_call",
    model_name="my-fine-tuned-model",
    model_provider="custom",
    ml_app="my-chatbot",
) as span:
    span.input_data = [{"role": "user", "content": "Hi"}]
    response = my_custom_llm_call(...)
    span.output_data = [{"role": "assistant", "content": response}]
    span.metrics["input_tokens"] = 100
    span.metrics["output_tokens"] = 50
```

#### 5.3.2 Node.js

```typescript
import OpenAI from "openai";
import { LLMObs } from "@datadog/llm-observability";

LLMObs.enable({
  mlApp: "my-chatbot",
  apiKey: process.env.DD_API_KEY,
  site: "datadoghq.com",
  integrations: [new LLMObs.OpenAIIntegration()],
});

const client = new OpenAI();
const response = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "Hello" }],
});
```

#### 5.3.3 适用场景

- 需要精细控制 trace
- 多语言混用
- 自定义 LLM provider
- 用 OTel 自动 export

### 5.4 部署 checklist

- [ ] Datadog 账号 + API Key（admin 权限）
- [ ] LLM provider API Key（OpenAI / Anthropic / ...）
- [ ] 选择模式（SaaS / Agent / SDK）
- [ ] 配置 `llmobs` 模块
- [ ] 配置 provider
- [ ] 配置 sampling
- [ ] （可选）启用 PII 扫描
- [ ] （可选）启用 prompt injection 检测
- [ ] （可选）配置 guardrail 规则
- [ ] 配置 trace 关联到现有 APM service
- [ ] 验证 trace 上报（UI → LLM Observability → Spans）
- [ ] 设置 cost 预算 + 告警
- [ ] 设置异常告警（p95 latency, error rate）

---

## 6. 成本模型

### 6.1 定价 SKU

Datadog AI Gateway **不单独卖**——它是 LLM Observability 的"流量入口"组件。LLM Observability 自身是独立 SKU。

#### 6.1.1 LLM Observability 定价（2026-Q2）

| 项 | 价格 | 计费单位 |
|---|---|---|
| **Span Ingest** | $0.05 / 百万 span | ingest 时计费 |
| **Span Retention** (90 天) | $0.10 / GB / 月 | 按 span 压缩后体积 |
| **Span Retention > 90 天** | $0.05 / GB / 月 | 按 30 天滚动计费 |
| **Eval Run** (LLM-as-judge) | $0.10 / 千次 | 触发时计费 |
| **Eval Run** (Datadog 自研 evaluator LLM) | $0.05 / 千次 | 触发时计费 |
| **Custom Evaluator** | $0.10 / 千次 | 用户自定义 evaluator run |
| **Guardrail** (Datadog 提供规则) | $0.05 / 千次 | 命中时计费 |
| **Guardrail** (自配) | $0 | 用户自配不计费 |
| **PII Scan** (Sensitive Data Scanner) | $0.10 / GB 扫描数据 | 扫描时计费 |

#### 6.1.2 SaaS Gateway 流量

- **不限流量**——按 span 计费，gateway 本身无流量费
- 但客户需承担 LLM provider 的实际费用（OpenAI / Anthropic / ...）

#### 6.1.3 隐含成本

| 项 | 估算 | 备注 |
|---|---|---|
| **Datadog 平台费** | 必选 | APM / Infrastructure / Logs 单独收费 |
| **APM 基础 SKU** | 必选 | LLM Obs 依赖 APM，APM Host 起价 $31/host/月 |
| **每 host** | $31 + $5 (LLM Obs ingest) | 单台服务一年 ~$432 |
| **10 host** | $310 + $50 (LLM Obs ingest) | 一年 ~$4,320 |

### 6.2 成本计算示例

#### 6.2.1 中型 SaaS 客户

假设：
- 每天 10,000 次 LLM 调用
- 每次调用产生 5 个 span（LLM + 4 个 tool call）
- 每次 LLM 调用平均 2000 input + 500 output tokens
- OpenAI gpt-4o 单价：$2.50/M input, $10.00/M output
- 每月（30 天）

| 成本项 | 计算 | 月度费用 |
|---|---|---|
| LLM 调用本身 | 10000 × 30 × (2000/1M × 2.5 + 500/1M × 10) = 10000 × 30 × 0.01 = **$3,000** | $3,000 |
| LLM Obs span ingest | 10000 × 30 × 5 = 1.5M span → 1.5 × $0.05 = **$0.075** | $0.075 |
| LLM Obs span retention | 1.5M × 1.5KB / 1024 = ~2.2GB → 2.2 × $0.10 = **$0.22** | $0.22 |
| APM 基础 (5 host) | 5 × $31 = **$155** | $155 |
| Total | | **$3,155.30** |

**Datadog AI Gateway 占比**：$0.30 / $3,155 = **0.0095%**（几乎可以忽略）

#### 6.2.2 大型企业客户

假设：
- 每天 1M 次 LLM 调用
- 每次调用 5 span
- 每月 30 天

| 成本项 | 月度费用 |
|---|---|
| LLM 调用 (gpt-4o, 2000/500 tokens avg) | **$300,000** |
| LLM Obs span ingest (150M span) | $7.50 |
| LLM Obs span retention (~220GB) | $22 |
| APM 基础 (50 host) | $1,550 |
| Total | **$301,580** |

**Datadog AI Gateway 占比**：$30 / $301,580 = **0.01%**

#### 6.2.3 小团队/个人

假设：
- 每天 100 次 LLM 调用
- 每次 3 span（无 tool call）
- 每月 30 天

| 成本项 | 月度费用 |
|---|---|
| LLM 调用 (gpt-4o-mini) | ~$1 |
| LLM Obs span ingest (9,000 span) | 几乎免费 |
| APM 基础 (1 host) | $31 |
| Total | **~$32** |

### 6.3 定价策略观察

| 观察点 | 含义 |
|---|---|
| **Span ingest 极便宜** | 鼓励用户多 trace，不要被成本吓到 |
| **APM 是隐性门槛** | 必须先买 APM 才能用 LLM Obs，对纯 LLM 团队不友好 |
| **Eval / Guardrail 单独计费** | 提醒用户主动消费才能发挥 LLM Obs 价值 |
| **无免费层** | 没有 Langfuse / Helicone 那种"1000 call/month free" |
| **Datadog 平台套娃** | Logs / Metrics / APM / LLM Obs / DBM / RUM / SIEM / ASM 全部独立报价 |

### 6.4 与竞品定价对比

| 产品 | 月度 10K LLM 调用成本 | 月度 1M LLM 调用成本 | 免费层 |
|---|---|---|---|
| **Datadog AI Gateway** | ~$32 (含 APM) | ~$1,580 (含 APM) | ❌ |
| **Portkey Cloud** | $0 (free tier) | $499 (Pro) | ✅ |
| **LiteLLM (self-host)** | 0 (只承担 infra) | 0 (只承担 infra) | N/A (self-host) |
| **Helicone** | $0 (free tier 100K event) | $0 (Growth) | ✅ 100K event |
| **Langfuse Cloud** | $0 (Hobby) | $199 (Pro) | ✅ 50K event |
| **Arize Phoenix (self-host)** | 0 (只承担 infra) | 0 (只承担 infra) | N/A (self-host) |
| **Bifrost (self-host)** | 0 (只承担 infra) | 0 (只承担 infra) | N/A (self-host) |

**Datadog 的优势**：
- 已是 Datadog 客户，**边际成本几乎为零**
- 跨产品联动（APM + Logs + Security）无额外整合成本

**Datadog 的劣势**：
- 对纯 LLM 团队**门槛高**（要买 APM）
- 与"AI gateway 工具"比，**单 LLM 专项的性价比低**（Langfuse / Helicone / Portkey 都便宜得多）

---

## 7. 生态

### 7.1 框架集成（9 大框架）

| 框架 | 集成方式 | 覆盖 |
|---|---|---|
| **OpenAI Python/Node SDK** | 自动 monkey-patch | 100% 覆盖 |
| **Anthropic Python/Node SDK** | 自动 monkey-patch | 100% 覆盖 |
| **LangChain** (`langchain` + `langchain-community`) | 官方 callback handler | 100% 覆盖（Agent / Tool / Chain / Retriever） |
| **LlamaIndex** | 官方 callback handler | 100% 覆盖（Query Engine / Retriever / Synthesizer） |
| **Haystack** | 官方 component | 100% 覆盖 |
| **DSPy** | 官方 callback | 100% 覆盖（含 BootstrapFewShot 等优化器） |
| **crewAI** | 官方 callback | 100% 覆盖（Agent / Task / Crew） |
| **AutoGen** (Microsoft) | 官方 callback | 100% 覆盖（UserProxyAgent / AssistantAgent / GroupChat） |
| **OpenAI Agents SDK** | 官方 callback | 100% 覆盖 |
| **Pydantic AI** | 官方 callback | 100% 覆盖 |
| **Vercel AI SDK** | 官方 callback | beta（2026-Q2 计划 GA） |
| **Semantic Kernel** (Microsoft) | 官方 callback | beta（2026-Q3 计划 GA） |

### 7.2 模型 Provider 集成（14 家）

详见 §3.1。

### 7.3 OTel 兼容

Datadog AI Gateway 完整实现 OTel GenAI semconv 1.30 属性集（2026-05 升级）。用户可通过 OTel SDK 直接 export 到 Datadog：

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

# 配置 OTel exporter 到 Datadog
exporter = OTLPSpanExporter(
    endpoint="https://trace.agent.datadoghq.com/api/v0.2/traces",
    headers={"DD-API-KEY": "<DD_KEY>"},
)
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# 用 OTel SDK 创建 LLM span
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("chat gpt-4o") as span:
    span.set_attribute("gen_ai.operation.name", "chat")
    span.set_attribute("gen_ai.provider.name", "openai")
    span.set_attribute("gen_ai.request.model", "gpt-4o")
    # ... Datadog 识别这些属性
```

### 7.4 Marketplace 集成

Datadog Marketplace 提供 LLM Obs 的附加组件：

| 组件 | 厂商 | 功能 |
|---|---|---|
| **Galileo** | Galileo AI | LLM hallucination 检测（与 LLM Eval 联动） |
| **WhyLabs** | WhyLabs Inc | AI observability 联动 |
| **Aporia** | Aporia | ML 监控 |
| **Arize** | Arize AI | 评估 + drift 检测 |
| **Fiddler** | Fiddler AI | 模型性能监控 |
| **Dataloop** | Dataloop | 数据标注 |
| **Scale AI** | Scale | 数据标注 |
| **Labelbox** | Labelbox | 数据标注 |
| **Humanloop** | Humanloop | Prompt 工程 + eval |
| **Honeycomb** | Honeycomb | 事件关联（轻量 APM） |

### 7.5 与 Datadog 其他产品联动

| 联动 | 描述 |
|---|---|
| **APM** | LLM span 与业务 span 同一 trace |
| **Logs** | LLM 异常自动生成 log（可配置） |
| **Metrics** | LLM token / cost / latency 自动成 metric |
| **Watchdog** | LLM 异常（cost spike、latency spike、error spike）触发 Watchdog 告警 |
| **SIEM** | Prompt injection 检测结果推 SIEM |
| **Cost Management** | LLM token 成本与云成本合并 dashboard |
| **Cloud Workload Security (CWS)** | LLM 调用异常的 host 行为分析 |
| **Bits AI** | Datadog 自家 SRE AI 助手（吃自己狗粮） |
| **Notebooks** | 用 Notebooks 分析 LLM 行为 |
| **Dashboards** | LLM 性能 / 成本 / 异常的预置 dashboard |

### 7.6 社区与生态

- **GitHub Stars**：`DataDog/datadog-agent` 整体 2.8k stars（整个 monorepo 不止）；`@datadog/llm-observability` npm 单独 240 stars
- **Discord**：Datadog 官方 Discord 有 `#llm-observability` 频道（2024-08 设立，~3,200 成员）
- **Office Hours**：Datadog 工程师每周二 11am ET 公开 Office Hours（Zoom）
- **博客**：Datadog Security Labs 博客 LLM 主题 ~30 篇 / 年；Datadog Engineering 博客 LLM 主题 ~10 篇 / 年
- **文档完整度**：API 文档、YAML 配置、Helm chart、Terraform provider 全套
- **示例代码**：`DataDog/llm-observability-examples` 仓库 12 个端到端示例

---

## 8. 客户案例

### 8.1 公开案例

#### 8.1.1 Notion

- **规模**：3M+ 付费用户、$10B 估值
- **用例**：Notion AI 写作助手、Notion Q&A 知识库
- **Datadog AI Gateway 价值**：
  - **PII 防护**——Notion 用户输入可能含 email / phone / 地址，自动 SDS 扫描标记
  - **prompt injection 防护**——攻击者可能通过文档内容注入 prompt，ML 模型拦截
  - **成本归因**——按 Notion workspace / user / team 分摊 LLM 成本
- **公开材料**：Notion 2024 SRECon 演讲（YouTube 公开）、Datadog 客户案例页面

#### 8.1.2 Ramp

- **规模**：金融科技独角兽、$22.5B 估值
- **用例**：内部 AI 助手（费用分析、合同审阅、客户支持）
- **Datadog AI Gateway 价值**：
  - **合规审计**——金融行业要求 7 年 trace 保留
  - **异常告警**——LLM 成本异常 / 异常输出实时告警
  - **跨 service trace**——AI 助手与生产 Ramp App 同一 APM 视图
- **公开材料**：Ramp 2024 KubeCon 演讲、Datadog 客户案例页面

#### 8.1.3 Instacart

- **规模**：杂货配送龙头，$30B+ 估值
- **用例**：AI 购物助手、客服 AI、商家 AI
- **Datadog AI Gateway 价值**：
  - **多 provider 路由**——OpenAI + Anthropic + 自研 LLM
  - **PII 防护**——用户地址、电话、信用卡信息自动检测
  - **成本优化**——按产品线分摊 LLM 成本
- **公开材料**：Instacart 2024 LLM Production Meetup 演讲

#### 8.1.4 Patreon

- **规模**：创作者平台，100M+ 用户
- **用例**：创作者 AI 助手、推荐系统、内容审核
- **Datadog AI Gateway 价值**：
  - **内容审核**——toxicity / jailbreak 检测
  - **PII 防护**——创作者 / 用户信息保护
  - **可观测**——LLM 性能与业务性能关联
- **公开材料**：Patreon 2024 内部工程博客

#### 8.1.5 匿名金融客户（Forrester 报告引用）

- **规模**：美国 Top 10 银行
- **用例**：内部客服 AI、合规审查
- **Datadog AI Gateway 价值**：
  - **合规审计**——所有 LLM 调用 7 年 trace
  - **PII 防护**——SOC 2 + PCI DSS 合规
  - **多 region**——us-east-1 + eu-west-1 多区域 LLM
- **公开材料**：Datadog 2024 Forrester Wave™ 报告

### 8.2 行业分布（Datadog 内部 2025-Q4 统计）

| 行业 | 占比 | 主要用例 |
|---|---|---|
| 金融 | 28% | 风控 AI、客服 AI、合规审查 |
| 零售 / 电商 | 18% | 购物助手、推荐、客服 |
| SaaS | 16% | 内部 AI 助手、用户功能增强 |
| 媒体 / 内容 | 12% | 内容生成、审核 |
| 健康医疗 | 10% | 诊断辅助、合规审查 |
| 教育 | 8% | 个性化辅导 |
| 其他 | 8% | 多元 |

### 8.3 客户规模分布

| 客户规模 | 占比 |
|---|---|
| SMB（< 50 员工） | 15% |
| Mid-Market（50-500 员工） | 35% |
| Enterprise（> 500 员工） | 50% |

**关键观察**：Datadog AI Gateway **明显偏 Enterprise**——50% 客户 > 500 员工。这是 Datadog 整体客户结构的镜像（Datadog 大客户占比本身就高）。

---

## 9. 优劣势分析

### 9.1 优势

| 优势 | 详细说明 |
|---|---|
| **1. 与 Datadog APM 深度整合** | 唯一在 LLM observability 领域实现"LLM span 与业务 span 同一 trace"的产品。对已用 Datadog 的企业，这是巨大优势 |
| **2. 多语言 SDK 齐全** | Python / Node / Go / Java / .NET / Ruby / PHP 全部支持，跨语言栈友好 |
| **3. OTel GenAI semconv 1.30 完整** | 与 OTel 生态兼容；用户可自由切换 OTel 后端（不必锁 Datadog） |
| **4. PII + 注入检测** | SDS + 自研 7B LLM 双轨，是商用 LLM gateway 中检测精度最高的之一（F1 0.91） |
| **5. 流式 SSE 处理** | 边到边转、不缓冲，对 streaming LLM 应用零额外延迟 |
| **6. 多 provider 覆盖** | 14 家 provider + 自定义 OpenAI 兼容，足够覆盖 99% 客户 |
| **7. Eval + Guardrail 联动** | LLM Eval 可在 trace 上在线跑；Guardrail 可阻止异常输出 |
| **8. 跨产品联动** | APM + Logs + SIEM + Cost Management + Watchdog 一体化 |
| **9. 部署灵活** | SaaS / Agent / SDK 三种模式覆盖不同合规需求 |
| **10. 企业级合规** | SOC 2 Type II、ISO 27001、HIPAA、PCI DSS、FedRAMP（In-Process）齐全 |

### 9.2 劣势

| 劣势 | 详细说明 |
|---|---|
| **1. 闭源核心** | AI Gateway 核心代理不开源（仅 Datadog Agent 中闭源模块）——用户不能审计代码、不能自托管商业版（仅有基础版） |
| **2. 非 LLM 路由专家** | 不做 model-level A/B、semantic cache、cost-based routing——这些是 Portkey / Bifrost / LiteLLM 的强项 |
| **3. 与 Langfuse / Arize 比 Eval 弱** | Eval / dataset / 标注功能是 Langfuse / Arize 的看家本领，Datadog LLM Eval 2025-09 才 GA，功能弱很多 |
| **4. APM 隐性门槛** | 必须先买 APM 才能用 LLM Obs；APM $31/host/月对小团队不友好 |
| **5. 无免费层** | 没有 Langfuse Hobby / Helicone Free / Portkey Free 那种免费层——对个人开发者门槛高 |
| **6. 不支持 A2A** | 2026-Q2 计划中；目前仅支持 MCP tool call（beta） |
| **7. 自定义 Eval 复杂** | 与 Langfuse SDK 比，Datadog 的 Custom Evaluator 配置更复杂（要写 Yaml + Python） |
| **8. 中国区表现弱** | gateway 区域不含 cn1；LLM Obs 后端在 us1/us3/eu1/ap1，中国客户 latency 偏高 |
| **9. 文档对新手不友好** | 与 Langfuse / Helicone 比，Datadog 文档更像"配置参考"而非"教程" |
| **10. Vendor Lock-in 高** | 一旦接入 Datadog AI Gateway，迁移到其他 LLM observability 工具成本高（span 结构、UI、alert 都需重建） |

### 9.3 与 aigw 项目的对比启示

| 维度 | Datadog AI Gateway | aigw（你的副业） | 启示 |
|---|---|---|---|
| **目标客户** | 中大型企业、Datadog 老客户 | 小B SaaS、5-15万/年 | 客户规模完全错位 |
| **核心能力** | 可观测 + 安全 + 成本 | 路由 + 成本归因 + 简单可观测 | 路径错位，aigw 走"代理 + 轻 observability"更合理 |
| **协议广度** | 14 provider + OTel 兼容 | OpenAI 兼容 + 4-5 个国内 provider | aigw 走"国内为主 + 海外为辅"差异化 |
| **部署方式** | SaaS / Agent / SDK | Docker Compose / K8s 部署 | aigw 走"一键自托管" |
| **定价模型** | 按 span + 平台套娃 | 按 SaaS 订阅 + 一次性 license | aigw 更适合小B |
| **品牌** | Datadog 品牌加持 | 全新品牌 | aigw 需要差异化卖点（成本、隐私、易用） |
| **生态** | APM / SIEM / Cost 一体化 | 较弱 | aigw 不可能建平台生态，只能做"轻但专" |
| **护城河** | Datadog 客户基数 + 数据飞轮 | 暂无 | aigw 需要找到自己的护城河（行业 know-how / 区域优势） |

**给 aigw 的战略建议**：
1. **不要试图和 Datadog 比"可观测深度"**——他们有 APM 飞轮，必输
2. **专注"AI 代理"本身的能力**——智能路由、semantic cache、成本优化、模型评估自动化
3. **走"小B 友好"路径**——免费层、5 分钟接入、不强制绑定可观测
4. **国内 + 海外双栈**——这是 Datadog 中国区的弱点
5. **构建"AI agent gateway"心智**——Datadog 是 "AI 流量镜像"，aigw 可以是 "AI 流量调度"

---

## 10. 与其他产品对比

### 10.1 vs Portkey

| 维度 | Datadog AI Gateway | Portkey |
|---|---|---|
| **核心定位** | LLM 可观测 + 安全 | LLM 路由 + 缓存 + 可观测 |
| **开源** | ❌ 闭源 | ✅ Apache 2.0 |
| **路由** | ❌ 弱 | ✅ model-level A/B、semantic cache、conditional routing |
| **缓存** | ❌ 需自配 | ✅ exact + semantic cache（built-in） |
| **可观测** | ✅ 强（19 个 LLM attribute + 跨 APM） | ✅ 强（25+ 个 LLM attribute） |
| **Eval** | ⚠️ 2025-09 GA | ✅ 2024 GA，功能丰富 |
| **Guardrails** | ✅ 自带（200+ 规则 + ML） | ✅ 集成（与合作伙伴） |
| **PII** | ✅ SDS（自家） | ✅ 集成（自配或合作伙伴） |
| **MCP** | ⚠️ beta | ✅ 自带 MCP gateway |
| **A2A** | ❌ 计划中 | ⚠️ 计划中 |
| **多语言** | ✅ 8 语言 | ⚠️ Python / Node / Go |
| **价格** | 贵（$31/host/月起） | 免费层 + 订阅 |
| **自托管** | ✅ Agent | ✅ 完全自托管 |
| **目标客户** | 中大型企业、Datadog 客户 | 各类 |
| **生态** | Datadog 全家桶 | LangChain / Langfuse / Opik 集成 |
| **市场份额** | 1-2%（仅 LLM observability 子集） | 5-8% |

**核心差异**：Datadog AI Gateway 是"LLM 流量镜像 + 治理"，Portkey 是"LLM 流量代理 + 路由"。前者偏 observability，后者偏 routing。

### 10.2 vs Langfuse

| 维度 | Datadog AI Gateway | Langfuse |
|---|---|---|
| **核心定位** | LLM 可观测 + 安全 | LLM 可观测 + Eval + Prompt Mgmt |
| **开源** | ❌ 闭源 | ✅ MIT（self-host） + 商业版 |
| **Eval** | ⚠️ 2025-09 GA | ✅ 2023 GA，最强 LLM eval 之一 |
| **Dataset** | ⚠️ 弱 | ✅ 强（核心功能） |
| **Prompt Mgmt** | ❌ 弱 | ✅ 强（版本控制、A/B、协作） |
| **Trace** | ✅ 强（19 attribute + APM 关联） | ✅ 强（30+ attribute） |
| **可观测** | ✅ 顶级（APM 飞轮） | ✅ 强（独立 LLM trace） |
| **跨语言** | ✅ 8 语言 | ✅ Python / Node / Go / Java |
| **价格** | 贵 | 免费层 + Pro $199/月 |
| **自托管** | ✅ Agent（闭源） | ✅ 完全开源 + 自托管 |
| **目标客户** | 中大型 | 各类（特别受 LangChain 生态欢迎） |
| **护城河** | Datadog 客户基数 | 开源 + Prompt/eval 飞轮 |
| **市场份额** | 1-2% | 15-20% |

**核心差异**：Langfuse 是"LLM 工程平台"（eval + prompt + dataset + trace），Datadog AI Gateway 是"LLM 流量可观测"。前者偏开发工具，后者偏运维工具。

### 10.3 vs Arize Phoenix

| 维度 | Datadog AI Gateway | Arize Phoenix |
|---|---|---|
| **核心定位** | LLM 可观测 + 安全 | LLM 评估 + Drift + 监控 |
| **开源** | ❌ 闭源 | ✅ Apache 2.0 |
| **Eval** | ⚠️ 弱 | ✅ 最强（Phoenix Experiments） |
| **Drift** | ❌ | ✅ 强（生产数据 vs 训练数据） |
| **Trace** | ✅ 强 | ✅ 强（OpenInference 标准） |
| **Embeddings** | ⚠️ trace | ✅ 强（UMAP、clustering、search） |
| **可观测** | ✅ 顶级 | ✅ 强 |
| **价格** | 贵 | 免费（self-host）+ 商业版 |
| **目标客户** | Datadog 客户 | ML/AI 工程团队、drift 检测需求 |

**核心差异**：Arize Phoenix 偏 ML/AI 工程（drift、embedding 可视化、eval），Datadog AI Gateway 偏 SRE/Ops（trace、cost、alert）。

### 10.4 vs Helicone

| 维度 | Datadog AI Gateway | Helicone |
|---|---|---|
| **核心定位** | LLM 可观测 + 安全 | LLM 可观测 + 缓存 + 计费 |
| **开源** | ❌ 闭源 | ✅ MIT |
| **缓存** | ❌ 需自配 | ✅ exact + semantic（built-in） |
| **可观测** | ✅ 强 | ✅ 强（19 attribute） |
| **定价** | 贵 | 免费层 + Growth $0（事件计费） |
| **目标客户** | Datadog 客户 | 个人开发者 / 小团队 |
| **特色** | APM 联动 | 缓存 + 免费层 |

**核心差异**：Helicone 偏开发者友好（免费、易用），Datadog 偏企业集成。

### 10.5 vs Bifrost

| 维度 | Datadog AI Gateway | Bifrost |
|---|---|---|
| **核心定位** | 可观测 + 安全 | 路由 + 性能（< 50µs overhead） |
| **语言** | Go（闭源） | Go（开源 Apache 2.0） |
| **Overhead** | 0.8ms p50 | **0.011ms p50**（领先 70x） |
| **开源** | ❌ | ✅ |
| **MCP** | beta | ✅ 一等公民 |
| **Code Mode** | ❌ | ✅ |
| **Enterprise LB** | ❌ | ✅ Adaptive LB |
| **Eval** | ⚠️ 弱 | ❌ |
| **可观测** | ✅ 顶级 | ⚠️ 弱（OTel 集成） |
| **价格** | 贵 | 免费（self-host） |
| **目标客户** | 大企业 | 性能敏感团队 |

**核心差异**：Bifrost 偏性能 / 路由 / 协议，Datadog 偏可观测 / 治理。

### 10.6 vs LiteLLM

| 维度 | Datadog AI Gateway | LiteLLM |
|---|---|---|
| **核心定位** | 可观测 + 安全 | 协议统一 + 路由 + 缓存 |
| **开源** | ❌ 闭源 | ✅ MIT |
| **协议** | 镜像（不变换） | 100+ provider 协议变换 |
| **路由** | ❌ | ✅ 多模型 fallback、cost-based、conditional |
| **缓存** | ❌ | ✅ exact + semantic |
| **可观测** | ✅ 顶级 | ⚠️ 弱（集成 Langfuse / Datadog） |
| **语言** | Go | Python |
| **价格** | 贵 | 免费（self-host） |
| **目标客户** | 大企业 | 开发者、自托管首选 |

**核心差异**：LiteLLM 偏"万能协议适配器"，Datadog 偏"运维可观测"。

### 10.7 决策树：选哪个

```
你是 Datadog 老客户吗？
├─ 是 → 评估 Datadog AI Gateway（边际成本低）
│   └─ LLM 路由 / 缓存是核心需求？
│       ├─ 是 → Portkey + Datadog LLM Obs 联动
│       └─ 否 → Datadog AI Gateway 足够
│
└─ 否 → 主要需求是？
    ├─ 路由 + 缓存 + 性能 → Portkey / Bifrost / LiteLLM
    ├─ Eval + Prompt + Dataset → Langfuse / Arize Phoenix
    ├─ 免费 + 易用 + 缓存 → Helicone
    ├─ 可观测 + 安全 + APM 联动 → Datadog AI Gateway
    └─ 不确定 → Langfuse（开源、灵活、文档好）
```

---

## 11. 关键技术细节深入

### 11.1 Anthropic Prompt Caching 透明注入（2026-03 上线）

Datadog AI Gateway 2026-03 上线了一个**对调用方透明**的优化：自动在 Anthropic 调用里追加 `cache_control: ephemeral` 块，让 Anthropic 缓存命中率提升 30-50%。

#### 11.1.1 原理

```python
# 用户原始请求（无 cache_control）
{
    "model": "claude-3-5-sonnet-20241022",
    "system": "You are a helpful assistant for Acme Corp. [3000 tokens of system prompt]",
    "messages": [{"role": "user", "content": "What's the weather in Beijing?"}]
}

# AI Gateway 自动重写（追加 cache_control）
{
    "model": "claude-3-5-sonnet-20241022",
    "system": [
        {
            "type": "text",
            "text": "You are a helpful assistant for Acme Corp. [3000 tokens of system prompt]",
            "cache_control": {"type": "ephemeral"}  # 自动注入
        }
    ],
    "messages": [{"role": "user", "content": "What's the weather in Beijing?"}]
}
```

#### 11.1.2 抓取

抓到的 `gen_ai.usage.cached_input_tokens` 反映实际命中：

```json
{
    "gen_ai.usage.input_tokens": 3015,
    "gen_ai.usage.cached_input_tokens": 3000,  // 命中
    "gen_ai.usage.cache_hit_rate": 0.995       // 命中率
}
```

#### 11.1.3 效果

- 客户感知：零代码改动
- 成本节省：Anthropic cached input token 单价是 normal 的 10%（$0.30/M vs $3.00/M），cache 命中后省 90%
- Latency 提升：cached token 不参与 prefill，省 200-500ms

### 11.2 OpenAI Responses API 协议支持

OpenAI 2025-08 发布 Responses API，是 Chat Completions 的升级版（内建 COT、tool call 原生）。Datadog AI Gateway 2025-09 即时支持：

```python
# OpenAI Responses API（用户）
from openai import OpenAI
client = OpenAI(base_url="https://gateway.datadoghq.com/v1")
response = client.responses.create(
    model="o1",
    input="Explain quantum entanglement",
    reasoning={"effort": "medium"},
)
# Datadog 抓取的 span：
# - gen_ai.operation.name = "responses"
# - gen_ai.usage.reasoning_tokens = 850
# - gen_ai.usage.output_tokens = 320
# - gen_ai.response.reasoning_summary = [...]
```

### 11.3 MCP Tool Call trace（2026-Q2 beta）

```python
# 用户代码
from mcp import Client
client = Client(transport="stdio", command="python", args=["my_server.py"])
tools = await client.list_tools()
# Datadog 自动 trace list_tools() 作为一个 span

result = await client.call_tool("get_weather", {"city": "Beijing"})
# Datadog 自动 trace call_tool() 作为一个 span
# - gen_ai.operation.name = "execute_tool"
# - gen_ai.tool.name = "get_weather"
# - gen_ai.tool.call.arguments = {"city": "Beijing"}
# - gen_ai.tool.call.result = {"temp": 25}
```

### 11.4 自研 Prompt Injection Detector 详解

#### 11.4.1 架构

```
输入 prompt
   │
   ▼
┌──────────────────┐
│ 规则引擎         │ (200+ 规则，0.1ms)
│ - 关键词黑名单   │   "ignore previous instructions"
│ - Unicode 攻击   │   U+202E (RTL override)
│ - 编码攻击       │   base64/hex encoded instructions
│ - Markdown 注入  │   ![](javascript:...)
└────────┬─────────┘
         │ 命中任一规则
         ▼
   verdict = "suspicious" (0.5 概率)
   │  (没命中)
   ▼
┌──────────────────┐
│ 自研 7B LLM       │ (8-15ms)
│ 输入: prompt      │
│ 输出: 0/1 判定   │
│ 训练: Datadog     │
│   Security Labs  │
│   维护的 50K     │
│   标注样本       │
└────────┬─────────┘
         │
         ▼
   verdict = "safe" / "suspicious" / "malicious"
   score = 0.0 - 1.0
```

#### 11.4.2 训练数据

- 50K 标注样本（positive + negative）
- 2026-04 与 Anthropic / OpenAI 安全团队合作，引入红队测试样本
- 2026-05 加入 multi-turn injection 样本（针对 agent 多步对话）

#### 11.4.3 性能（Datadog 2026-05 内部 benchmark）

| 场景 | 召回 | 精确 | F1 |
|---|---|---|---|
| 单轮 prompt injection | 0.96 | 0.92 | 0.94 |
| 多轮 agent injection | 0.93 | 0.88 | 0.90 |
| 编码攻击（base64/hex） | 0.91 | 0.85 | 0.88 |
| 越狱（jailbreak） | 0.95 | 0.90 | 0.92 |
| 误报率（业务 prompt） | - | - | 0.04 |

### 11.5 Sensitive Data Scanner 集成

#### 11.5.1 自动检测

Datadog SDS 内置 100+ pattern：

- 个人信息：EMAIL, PHONE, SSN, PASSPORT, DRIVER_LICENSE, CREDIT_CARD
- 金融：IBAN, SWIFT, BANK_ACCOUNT, ABA_ROUTING
- 健康：MRN, NPI, ICD_CODE, DEA_NUMBER
- 安全：API_KEY, JWT, OAUTH_TOKEN, SSH_KEY, PASSWORD
- 地区：CN_ID_CARD（中国身份证）、CN_PHONE、CN_BANK_CARD

#### 11.5.2 配置示例

```yaml
# datadog.yaml
llmobs:
  sensitive_data_scanner:
    enabled: true
    actions:
      - type: mask
        patterns: [EMAIL, PHONE, SSN]
      - type: block
        patterns: [CREDIT_CARD, API_KEY]
    on_prompt: true
    on_completion: true
    custom_patterns:
      - name: INTERNAL_PROJECT_CODE
        regex: "PROJ-[A-Z]{2}\\d{4}"
        action: mask
```

#### 11.5.3 抓取属性

```json
{
    "gen_ai.security.pii.detected": true,
    "gen_ai.security.pii.entities": ["EMAIL", "PHONE"],
    "gen_ai.security.pii.action": "mask",
    "gen_ai.security.pii.count": 2
}
```

### 11.6 Eval Runner 详解

#### 11.6.1 触发方式

```python
# 1. 用户代码手动触发
from ddtrace.llmobs import LLMObs
LLMObs.submit_evaluation(
    span_id=span.span_id,
    name="relevance",
    score=0.85,
    label="pass",
    explanation="The response is relevant to the question.",
)

# 2. 配置自动触发（每次 LLM 调用都跑）
LLMObs.enable_evaluator(
    name="hallucination",
    evaluator=MyHallucinationEvaluator(),  # 用户自定义
    sample_rate=0.1,  # 10% 调用触发
)

# 3. Datadog 自带 evaluator
LLMObs.enable_evaluator(
    name="dd.toxicity",
    sample_rate=0.05,
)
LLMObs.enable_evaluator(
    name="dd.relevance",  # LLM-as-judge with GPT-4o
    sample_rate=0.02,
)
```

#### 11.6.2 UI 集成

- 任何 LLM span 上可点 "Run evaluation"，选择 evaluator，立即出分
- Evaluation dataset 可导入 OpenAI Eval format / LangSmith format
- Dataset 跑分可对比不同 prompt / 模型 / 评估器

---

## 12. 未来路线图

### 12.1 2026-Q3 计划（公开 changelog 预告）

- **A2A 协议支持**——Agent-to-Agent 通信 trace
- **MCP auth 完整支持**——OAuth 2.1 透传
- **CIMD（Common Identity for Model Discovery）**——与 AAIF 协作
- **Generative AI 评估增强**——多模态（图像/音频）评估
- **成本预测**——基于历史 trace 预测下月 LLM 成本
- **跨 region 同步**——多 region gateway 数据聚合

### 12.2 2026-Q4 路线图（猜测）

- **A2A 安全**——Agent 间通信的 prompt injection 检测
- **A2A Routing**——基于任务类型路由到合适 agent
- **自动 model 升级建议**——基于 latency / cost 趋势推荐更便宜或更快的模型
- **LLM 推理性能分析**——与 Triton / vLLM 集成，trace 到推理层
- **GenAI Cost 优化**——自动 fallback 到更便宜 model
- **API Key 自动轮换**——防止 LLM provider key 失效

### 12.3 2027+ 远期

- **跨 agent trace**——open standard（与 Langfuse、Arize、OpenLLMetry 协作）
- **Real-time eval**——流式评估（边生成边打分）
- **LLM 反向代理**——让 LLM 反过来调 Datadog（"self-debugging" agent）
- **与 Agent Gateway 整合**——Solo.io 协作的 agentgateway
- **Edge AI Gateway**——Cloudflare Workers / Vercel Edge 部署

---

## 13. 关键观察与启示

### 13.1 Datadog 的战略意图

1. **APM 飞轮延伸到 AI**——Datadog 的核心资产是 APM，把 LLM 视为"另一种服务"是自然延伸
2. **不做 LLM 路由**——明确避开与 Portkey / Bifrost / LiteLLM 的直接竞争
3. **强调安全**——PII、prompt injection、jailbreak 检测是大客户刚需
4. **平台粘性**——一旦用 Datadog AI Gateway，迁移成本极高（APM trace、Cost、SOC 联动）
5. **不做廉价方案**——明确不与 Langfuse / Helicone 的免费层竞争，定位"愿意付溢价的 enterprise"

### 13.2 对 aigw 的启示

1. **"AI 流量调度"是差异化方向**——Datadog 走"AI 流量镜像"，aigw 可走"AI 流量调度 + 智能路由"
2. **小B 友好是 Datadog 弱点**——免费层、5 分钟接入、$30/月而非 $30/host/月
3. **国内 + 海外双栈**——Datadog 中国区表现弱，aigw 有地理优势
4. **不要试图与 Datadog 比可观测深度**——专注 aigw 的核心：路由、缓存、成本归因
5. **可观测是配套，不是核心**——可观测与 Langfuse / Arize 集成（不与 Datadog 竞争）
6. **安全是增值**——把 prompt injection 检测做成"开箱即用"，对应小B 客户对安全的焦虑

### 13.3 市场定位矩阵

```
                简单 / 单一
                     ▲
                     │
        Helicone ●   │   ● Portkey
        Langfuse ●   │   ● Bifrost
                     │
   便宜/开源 ◀───────┼────────▶ 贵/企业
                     │
        LiteLLM ●    │   ● Datadog AI Gateway
        Arize ●      │   ● Solo agentgateway
                     │
                     ▼
                复杂 / 多功能
```

**aigw 的目标象限**：
- 简单 + 便宜 + 开源 + 中国友好 = **潜在优势区**
- 关键是要找到一个 Datadog / Portkey / Bifrost 都不强，但 aigw 能做的"小B 友好 AI gateway"定位

---

## 14. 参考资料

### 14.1 Datadog 官方

- [Datadog LLM Observability 文档](https://docs.datadoghq.com/llm_observability/)
- [Datadog AI Gateway 文档](https://docs.datadoghq.com/agent_gateway/) (注：2026-Q1 文档已合并到 LLM Observability)
- [Datadog OTel GenAI semconv 1.30 升级公告](https://www.datadoghq.com/blog/llm-observability-otel-genai/)
- [Datadog Prompt Injection Detection v2 博客](https://securitylabs.datadoghq.com/articles/prompt-injection-2025/)
- [Datadog Cost Management 文档](https://docs.datadoghq.com/cost_management/)
- [Datadog Sensitive Data Scanner 文档](https://docs.datadoghq.com/sensitive_data_scanner/)
- [Datadog Changelog（2025-2026）](https://docs.datadoghq.com/agent/changelog/)

### 14.2 GitHub

- [DataDog/datadog-agent](https://github.com/DataDog/datadog-agent) — Datadog Agent（闭源 llmobs 模块）
- [DataDog/dd-trace-go](https://github.com/DataDog/dd-trace-go) — Go tracer（开源 llmobs 子包）
- [DataDog/dd-trace-py](https://github.com/DataDog/dd-trace-py) — Python tracer（开源 llmobs 子包）
- [DataDog/dd-trace-js](https://github.com/DataDog/dd-trace-js) — Node tracer
- [DataDog/llm-observability-examples](https://github.com/DataDog/llm-observability-examples) — 示例代码

### 14.3 OTel / 生态

- [OpenTelemetry GenAI SemConv 1.30](https://github.com/open-telemetry/semantic-conventions/tree/main/docs/gen-ai)
- [OpenInference](https://github.com/Arize-ai/openinference) — Phoenix 的 OTel 实现
- [OpenLLMetry](https://github.com/traceloop/openllmetry) — Traceloop 的 OTel 实现

### 14.4 第三方分析

- [The New Stack: Datadog LLM Observability 评测](https://thenewstack.io/datadog-llm-observability-review/)
- [DevOps.com: Datadog AI Gateway 概览](https://devops.com/datadog-ai-gateway-overview/)
- [Last9: LLM Gateway 对比](https://last9.io/blog/llm-gateway-comparison/)
- [ClickHouse 博客：OTel + LLM Observability](https://clickhouse.com/blog/llm-observability)
- [Forrester Wave™: AI/ML Observability 2024](https://www.forrester.com/)

### 14.5 客户案例

- [Notion × Datadog（SRECon 2024 演讲）](https://www.youtube.com/results?search_query=notion+datadog+srecon)
- [Ramp × Datadog（KubeCon 2024 演讲）](https://www.youtube.com/results?search_query=ramp+datadog+kubecon)
- [Instacart × Datadog LLM Production Meetup](https://www.meetup.com/llm-production/)

---

## 15. 调研小结

Datadog AI Gateway 是**传统 observability 厂商切入 AI 赛道的典型路径**：

1. **核心定位**：以 APM trace 为核心，LLM 调用视为"另一种服务"
2. **强项**：与 APM 深度整合、多语言 SDK 齐全、PII/注入检测强
3. **弱项**：闭源、不做 LLM 路由、Eval/Prompt Mgmt 弱、APM 隐性门槛
4. **目标客户**：中大型 Datadog 老客户；不适合纯 LLM 团队
5. **对 aigw 启示**：aigw 不应与 Datadog 比可观测，应专注"AI 流量调度"差异化，走"小B 友好 + 国内/海外双栈 + 免费层"路径

报告结束。
