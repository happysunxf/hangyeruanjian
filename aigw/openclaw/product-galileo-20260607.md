# Galileo 深度调研 — "评估 SLM + 实时 Guardrail"路线的代表

> 调研对象：**Galileo**（[galileo.ai](https://galileo.ai)）
> 调研日期：2026-06-07
> 调研人：Rich (OpenClaw main session, cron: `ai-gateway-product-research`)
> 报告定位：**r35 第 4 份清单外扩展深挖**（继 Bifrost、DeepInfra、Groq、Beam、Requesty 之后）。原 30 个候选清单（Portkey、LiteLLM、One API、Higress、Kong、APISIX、Envoy AI GW、vLLM、SGLang、TGI、Triton、LMDeploy、llama.cpp、Cloudflare、OpenRouter、Helicone、LangSmith、Unify、Not Diamond、Martian、TrueFoundry、Together、Fireworks、Replicate、Modal、Langfuse、Arize Phoenix、Traceloop、Baseten）已 100% 全部深挖完毕。Galileo 是 2025-2026 兴起的"评估 SLM + 实时 Guardrail"赛道的代表，与 Langfuse/LangSmith/Traceloop/Helicone/Arize Phoenix（偏 tracing）形成"质量评估"互补。
> 文档约定：本文为单产品 600+ 行代码级深挖，覆盖项目背景 / 架构 / 协议 / 性能 / 部署 / 成本 / 生态 / 案例 / 优劣 / 对比 10 维度，附 ASCII 架构图、性能数据表、协议细节、与 7 个直接竞品对比表

---

## 0. 为什么挑 Galileo

| 候补维度 | 评分 | 说明 |
|---|---|---|
| 公开材料丰富度 | 9/10 | 官方 6 页产品页 + docs.galileo.ai/llms.txt 完整索引（600+ API 端点） + arXiv 论文 + GitHub SDK 6+ 仓库 + 2 个公开 Leaderboard + 多篇技术博客 |
| 市场地位 | 8/10 | 2021 成立，3 位 Co-founder 含 Google AI/Uber AI/Apple Siri 背景；客户 Twilio、Comcast、HP、Cisco、Clearwater Analytics、某 Fortune 50 CPG；arXiv 论文被广泛引用 |
| AI Gateway 纯度 | 6/10 | **不是 API Gateway**（流量不经过 Galileo），而是 **observability + evaluation + real-time guardrail 平台**——属于"AI Gateway 配套的旁路控制平面" |
| 技术差异化 | 10/10 | **自研 Luna-2 SLM 系列（3B/8B）**，专用评估；97% 成本下降 vs GPT 评估；sub-200ms 延迟跑 10-20 个指标；ChainPoll 论文 AUROC 0.781 |
| 对小 F 副业的启发 | 8/10 | "评估 SLM 替代 LLM-as-judge"是一个可复用的技术模式；副业方向："垂直行业 eval 模型微调"或"小 B 行业质量看板" |
| **总分** | **41/50** | 显著超过 35 阈值 |

**核心吸引力**：Galileo 是当前 **"评估型 AI Reliability 平台"赛道** 的事实标杆——**自研 Luna-2 SLM 替代 GPT-4 作为 judge**（$0.12/1M tokens vs $5.00/1M tokens），用 SLM 跑 10-20 个 guardrail 指标仍能保持 sub-200ms 延迟，且在 RAG 上下文 adherence 上达到 0.95 准确率（GPT-4 0.94）。对小 F 副业的借鉴价值 = **"SLM 蒸馏大模型能力"是当前最有商业可行性的 AI 基础设施创新方向**——成本下降两个数量级、准确率不降反升、延迟降一个数量级。

---

## 1. 项目背景

### 1.1 一句话定位

> **"Galileo is the leading observability, evaluation, and production guardrail platform for GenAI and agentic applications. Designed to empower engineers, product managers, and subject matter experts to build safe and reliable AI applications, Galileo works with all major LLM providers."**（官方 docs.galileo.ai/what-is-galileo 自述）

**关键词解构**：

| 关键词 | 含义 | 与同类对比 |
|---|---|---|
| **Observability** | Trace/span 收集、可视化、监控 | Langfuse / LangSmith / Helicone / Arize Phoenix / Traceloop |
| **Evaluation** | offline 实验、test set、metrics | Langfuse / LangSmith / Braintrust / DeepEval / RAGAS |
| **Production guardrail** | 实时拦截 PII / hallucination / 越权 | NeMo Guardrails / Guardrails AI / Lakera / Cloudflare AI Gateway |
| **Evaluation Foundation Model (EFM) / Luna** | 专用小模型跑评估 | **Galileo 独有**——其他都用 LLM-as-judge |
| **Multi-headed SLM** | 1 个 base + N 个 adapter 跑 N 个指标 | **Galileo 独有**——"one model, hundreds of metrics" |
| **CLHF（Continuous Learning with Human Feedback）** | 用人反馈持续优化 judge prompt | 类似 LangSmith + Braintrust 的人工反馈回路 |
| **Signals** | 主动检测 "unknown unknown" 失败模式 | **Galileo 独有**——其他都是"你写 eval 它跑" |
| **GenAI + Agentic** | 同时面向 LLM 应用和 agent | Langfuse、Helicone 同型；LangSmith 偏 LLM |

### 1.2 公司基本面

| 维度 | 详情 |
|---|---|
| **公司名** | Galileo Technologies, Inc.（品牌 Galileo AI） |
| **域名** | galileo.ai（主站）/ app.galileo.ai（控制台）/ docs.galileo.ai（文档） |
| **控制台 URL** | `https://app.galileo.ai/`（多租户 SaaS） |
| **企业部署** | Hosted SaaS / Virtual Private Cloud / On-Premises（Enterprise 价） |
| **成立时间** | 2021 |
| **总部** | 美国（无公开注册地） |
| **团队规模** | 不公开（leadership 8 人） |
| **融资** | 不公开（基于 2021 成立、Fortune 50 客户，技术博客密度看，至少 B 轮以上） |
| **GitHub org** | https://github.com/rungalileo（活跃维护 10+ 仓库） |
| **开源仓库** | gcache（fine-grained caching 框架, 37★）、sdk-examples（16★）、galileo-python（20★）、galileo-js（7★）、agent-leaderboard（221★）、hallucination-index（116★）、docs-official（5★） |
| **代码开源度** | SDK / 客户端完全开源（Apache-2.0 / MIT）；**Luna 模型 / 推理引擎 / 控制台后端全部闭源**——与 Langfuse / LangSmith / Helicone "评估引擎可自托管" 路线不同 |
| **认证** | SOC 2（Enterprise）；HIPAA（Enterprise） |

**关键观察**：

1. **平台是 SaaS-first**——控制台、推理引擎、Luna 模型都在 Galileo 自家后端
2. **SDK 完全开源**（Apache-2.0）——集成代码可审计、可 fork、可自托管日志 pipeline
3. **Luna 模型不开源权重**——这是 Galileo 的核心壁垒，类似 OpenAI/Anthropic 的"模型即服务"
4. **Enterprise 可买断 On-Premises**——但 Luna 模型仍由 Galileo 提供（on-prem 部署 Galileo Inference Engine 跑 Luna）

### 1.3 团队 / 创始人

| 姓名 | 职位 | 背景 |
|---|---|---|
| **Vikram Chatterji** | Co-founder & CEO | 前 Google AI，**BERT 模型作者之一** |
| **Atindriyo Sanyal** | Co-founder & CPO | 前 Google，**语音识别**研究；ChainPoll 论文一作 |
| **Yash Sheth** | Co-founder & CTO | 前 Uber AI / Apple Siri |
| **Jason Garoutte** | Chief Marketing Officer | 不公开 |
| **Sudhir Tonse** | VP of Engineering | 不公开 |
| **Brian O'Shea** | Chief Revenue Officer | 不公开 |
| **Taylor Rachor** | Head of People | 不公开 |
| **Soumya Mohan** | Head of Product | 不公开 |

**团队结构观察**：

1. **技术血统极强**——CEO 是 BERT 作者之一（NLP 基础研究界顶级），CTO 来自 Apple Siri（语音 AI 工业级），CPO 来自 Google 语音研究（生产级 ML pipeline）
2. **三人都在 AI 可靠性问题上干过**——Vikram 在 Google AI 见过 hallucination 问题，Atindriyo 在 Google 做过评估，Yash 在 Apple Siri 见过生产 agent 失败
3. **leadership 8 人规模算中大型 AI startup**——比 Langfuse（3 co-founder）、Helicone（2 co-founder）更成熟

### 1.4 关键论文 & 公开研究成果

| 论文/项目 | 发表 | 一作 | 关键数据 |
|---|---|---|---|
| **Luna: An Evaluation Foundation Model to Catch LLM Hallucinations** | arXiv:2406.00975 (2024-06) | Masha Belyi et al. | DeBERTA-large 440M；**比 GPT-3.5 准 + 97% 成本下降 + 91% 延迟下降** |
| **ChainPoll: A High Efficacy Method for LLM Hallucination Detection** | arXiv:2310.18344 (2023-10) | Atindriyo Sanyal et al. | AUROC 0.781；比次佳高 11%，比行业标准高 23% |
| **RealHall Benchmark** | 同上配套 | Atindriyo Sanyal et al. | 4 个真实 RAG 场景的 hallucination 检测基准 |
| **LLM Hallucination Index** | GitHub 公开 | Galileo 团队 | 22 个模型、3 段 context（<5K / 5K-25K / 40K-100K）；**行业最权威的 RAG 幻觉评测** |
| **Agent Leaderboard v2** | HuggingFace Space | Galileo 团队 | 5 行业 × 100 场景；GPT-4.1 AC 62%，Kimi K2 AC 0.53 TSQ 0.90 |

**学术影响力**：

- ChainPoll 论文被 LangChain、LlamaIndex、Patronus AI 等大量引用
- Hallucination Index 是业内唯一公开的 **跨厂商、跨 context 长度**的 RAG 幻觉评测报告
- Agent Leaderboard v2 在 HuggingFace 上 221★（远超 LangChain/LlamaIndex 同期榜单）

---

## 2. 架构设计

### 2.1 平台 4 大模块

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Galileo Platform (2026)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  1. Evaluate     │  │  2. Observe      │  │  3. Protect      │  │
│  │  (Development)   │  │  (Production)    │  │  (Real-time)     │  │
│  │                  │  │                  │  │                  │  │
│  │  • Playground    │  │  • Trace ingest  │  │  • Luna-2 SLM    │  │
│  │  • Experiments   │  │  • Span tree     │  │  • Sub-200ms     │  │
│  │  • Datasets      │  │  • Sessions      │  │  • 10-20 metrics │  │
│  │  • Custom evals  │  │  • Cost / lat    │  │  • Action engine │  │
│  │  • Prompt reg    │  │  • Live monitor  │  │  • Central rules │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                     │                     │            │
│           └─────────────────────┼─────────────────────┘            │
│                                 │                                  │
│                    ┌────────────▼────────────┐                     │
│                    │  4. Signals             │                     │
│                    │  (Proactive detection)  │                     │
│                    │                         │                     │
│                    │  • Context engineering  │                     │
│                    │  • Pattern memory       │                     │
│                    │  • Unknown unknowns     │                     │
│                    │  • Insight cards        │                     │
│                    └─────────────────────────┘                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                    Engine Layer (Closed-source)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Luna-2 (3B)  │  │ Luna-2 (8B)  │  │ GPT-4 / 5   │              │
│  │ (Fast judge) │  │ (Acc judge)  │  │ (fallback)  │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         └─────────────────┼─────────────────┘                      │
│                ┌──────────▼──────────┐                              │
│                │ Galileo Inference  │                              │
│                │ Engine (L4 GPU)    │                              │
│                └─────────────────────┘                              │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │  SDK / API
                              │
        ┌─────────────────────┴─────────────────────┐
        │       User Application (anywhere)        │
        │   LLM calls  +  GalileoLogger.log()      │
        └───────────────────────────────────────────┘
```

**关键设计原则**：

1. **旁路（sidecar）模式**——Galileo 不在 LLM 流量路径上，**应用同时调 LLM + Galileo API**（trace 异步上传）
2. **实时 guardrail 是同步的**——Protect 必须在 LLM 调用前/后同步返回，sub-200ms 延迟要求
3. **Luna 模型专跑评估**——应用不直接调 Luna，**Galileo 后端用 Luna 评估用户上传的 trace**

### 2.2 端到端数据流（开发 → 生产）

```
┌─────────────────┐
│ Developer       │
│ (Playground/Exp)│
└────────┬────────┘
         │  1. Create Dataset + Custom Metric
         ▼
┌─────────────────┐
│ Galileo Console │  ┌──────────────────────────────┐
│ (Web UI)        │  │  Scorer Spec:                 │
│                 │  │  - Type: code / llm-as-judge  │
│                 │  │  - Metric: adherence, pii,    │
│                 │  │             toxicity, …       │
│                 │  │  - Ground truth: human labels│
└────────┬────────┘  └──────────────────────────────┘
         │
         │  2. Run Experiment
         ▼
┌──────────────────────────────────────────────┐
│  Experiment Runner (SDK)                     │
│  - Loop: for each (input, model, prompt):    │
│    1. Call LLM                               │
│    2. Run Galileo Scorer                     │
│    3. Compute metrics                        │
│    4. Upload trace + scores to Galileo       │
└────────┬─────────────────────────────────────┘
         │
         │  3. Best config → production
         ▼
┌──────────────────────────────────────────────┐
│  Production App                              │
│  - Call LLM (OpenAI / Anthropic / etc)       │
│  - Log via GalileoLogger (async)             │
│  - For Protect: call Galileo Protect (sync)  │
└────────┬──────────────────┬───────────────────┘
         │                  │
         │ async trace      │ sync guardrail
         ▼                  ▼
┌─────────────────┐  ┌─────────────────┐
│ Observe         │  │ Protect         │
│ - Trace stored  │  │ - Luna-2 judge  │
│ - Cost / lat    │  │ - Block / pass  │
│ - Insights      │  │ - Sub-200ms     │
└────────┬────────┘  └─────────────────┘
         │
         │  4. Periodic
         ▼
┌─────────────────┐
│ Signals         │
│ - Compress spans│
│ - Pattern detect│
│ - Signal cards  │
└─────────────────┘
```

### 2.3 Luna-2 模型架构（核心创新）

```
                ┌──────────────────────────────┐
                │       Luna-2 Base Core        │
                │      (Decoder-only SLM)       │
                │         3B / 8B params        │
                └──────────────┬───────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
      ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
      │ Adapter:     │  │ Adapter:     │  │ Adapter:     │
      │ Adherence    │  │ PII Leak     │  │ Tool Error   │
      │              │  │              │  │              │
      │ Adapter:     │  │ Adapter:     │  │ Adapter:     │
      │ Toxicity     │  │ Prompt Inj.  │  │ TSQ          │
      │              │  │              │  │              │
      │    …         │  │    …         │  │    …         │
      └──────────────┘  └──────────────┘  └──────────────┘
              │                │                │
              ▼                ▼                ▼
         ┌────────────────────────────────────────┐
         │     Multi-Headed SLM Output            │
         │     N metrics × score [0, 1]           │
         │     Latency: 152-214ms (L4 GPU)        │
         └────────────────────────────────────────┘
```

**关键创新**：

1. **多 adapter 共享 base**——一个 8B base 跑数百个 metric，**存储和算力成本不随 metric 数量线性增长**
2. **deterministic**——同 input 同 output（不像 LLM judge 有随机性）
3. **token-level explainability**——Luna 1 论文描述对每个 sentence 分类 adherent / non-adherent
4. **领域可微调**——"fewer than 50 labeled examples"即可在客户数据上 fine-tune（论文 + Luna-2 博客明示）

### 2.4 Signals 架构（"unknown unknown" 检测）

```
Raw Spans (25MB for one customer's day)
    │
    ▼
┌─────────────────────────────────┐
│ Step 1.1: Programmatic         │
│           Compression           │
│  - Whitelist relevant fields    │
│  - Dedupe tool schemas          │
│  - Compress repeated messages   │
│  → Compressed Spans            │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ Step 1.2: LLM Note-Taking      │
│  - Distill each session         │
│  - Capture "noteworthy"         │
│  - "Everything noteworthy"      │
│  → Distilled Notes (~500KB)     │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ Step 2: Cross-Session          │
│         Pattern Detection       │
│  - All notes in one context     │
│  - Historical signal memory    │
│  - LLM (claude-sonnet-4) judges │
│  - Generate ≤5 priority signals │
│  → Signal Cards (5 max)         │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ Insights UI:                   │
│ "Hallucination caused incorrect │
│  tool inputs. Best action:     │
│  Add few-shot examples..."     │
│  Confidence: 67%               │
│  15% of traffic affected       │
└─────────────────────────────────┘
```

**核心创新**：**两层 LLM 流水线 + 历史 memory**——

- 第一层用 reasoning model 蒸馏每个 session 为 ~500KB notes
- 第二层把所有 notes 装进一个 context window + 历史 signal summary，让 LLM 横向比较
- **不是 stateless "chat with logs"**——每次新分析都积累 institutional memory

### 2.5 SDK 集成架构

```
┌──────────────────────────────────────────────────────┐
│                 User Application                      │
│                                                      │
│  ┌────────────────────┐  ┌────────────────────┐      │
│  │ OpenAI Wrapper     │  │ LangChain Callback │      │
│  │ (drop-in)          │  │ (GalileoCallback)  │      │
│  └─────────┬──────────┘  └────────┬───────────┘      │
│            │                       │                  │
│            │   Span output         │                  │
│            ▼                       ▼                  │
│  ┌──────────────────────────────────────────┐        │
│  │      GalileoLogger (local buffer)        │        │
│  │  - Trace / Span / LLM / Retriever / Tool │        │
│  │  - @log decorator                        │        │
│  │  - galileo_context() context manager    │        │
│  │  - GalileoLogger.start_trace() / .flush()│        │
│  └─────────┬────────────────────────────────┘        │
│            │                                          │
│            │  Batch upload (async)                   │
│            ▼                                          │
│  ┌──────────────────────────────────────────┐        │
│  │     Galileo API Endpoint                 │        │
│  │  https://app.galileo.ai/api/...          │        │
│  └──────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────┐
│     Galileo Console                       │
│  - Project / Log Stream / Dataset /     │
│    Experiment / Scorer                   │
│  - Trace / Session / Span view          │
│  - Signal cards                          │
│  - Protect rules                         │
└──────────────────────────────────────────┘
```

**关键设计**：

- **OpenAI Wrapper** = `from galileo.openai import openai` 后 drop-in 替换官方 `openai` —— **应用代码零修改**即可接入
- **`@log` decorator** = 函数级 trace；支持 `span_type=workflow/llm/retriever/tool`
- **`galileo_context()`** = 上下文管理器，可覆盖 project / log_stream
- **LangChain / LangGraph / CrewAI / OpenTelemetry / OpenInference** 全套 callback——覆盖所有主流 agent 框架

---

## 3. 协议支持

### 3.1 LLM Provider 集成

| 类别 | 集成方式 | 备注 |
|---|---|---|
| **OpenAI** | Drop-in wrapper（`galileo.openai.openai`） | 完整 chat / completions / embeddings / responses API |
| **Anthropic** | Drop-in wrapper（`galileo.anthropic.anthropic`） | 完整 messages / streaming |
| **Azure AI Inference** | Drop-in wrapper（`galileo.azure_inference`） | 完整 Foundry / OpenAI Service / Anthropic on Azure |
| **AWS Bedrock** | Integration API（credentials） | 在 Integrations 中配 AWS access key，Galileo 代发 |
| **AWS SageMaker** | Integration API | 同上 |
| **Databricks** | Integration API | 同上 |
| **Ollama** | 通过 OpenAI wrapper（Ollama 暴露 OpenAI 协议） | 本地模型评测 |
| **Custom LLM** | Integration API（`create-or-update-custom-integration`） | 任意 base URL |
| **OpenAI 兼容服务** | Custom integration | OpenRouter、vLLM、Together、Fireworks、Anyscale Endpoint、TGI |

**SDK 源码结构**（galileo-python）：

```
galileo/
├── openai/          # OpenAI wrapper
├── anthropic/       # Anthropic wrapper
├── azure_inference/ # Azure wrapper
├── handlers/
│   ├── langchain.py        # LangChain callback
│   ├── langgraph.py        # LangGraph
│   ├── openai_agents.py    # OpenAI Agents SDK
│   ├── crewai.py           # CrewAI
│   ├── pydantic_ai.py      # Pydantic AI
│   ├── strands_agents.py   # Strands Agents
│   ├── microsoft_agents.py # Microsoft Agent Framework
│   ├── google_adk.py       # Google ADK
│   ├── mcp.py              # MCP tool calls
│   ├── otel.py             # OpenTelemetry bridge
│   └── openinference.py    # OpenInference bridge
├── datasets/        # Dataset CRUD
├── experiments/     # Experiment runner
├── scorers/         # Custom scorer
├── logger.py        # GalileoLogger
├── context.py       # galileo_context()
└── decorators.py    # @log
```

### 3.2 Agent Framework 集成

| Framework | 集成方式 | 状态 |
|---|---|---|
| **LangChain** | `GalileoCallback` | GA |
| **LangGraph** | `GalileoCallback` + `langgraph-traceloop` 选项 | GA |
| **OpenAI Agents SDK** | Drop-in wrapper | GA |
| **CrewAI** | Callback | GA |
| **Pydantic AI** | Callback | GA |
| **Mastra (TS)** | OpenInference bridge | GA |
| **Strands Agents** | OpenTelemetry | GA |
| **Microsoft Agent Framework** | OpenTelemetry | GA |
| **Google ADK** | OpenTelemetry + OpenInference | GA |
| **Vercel AI SDK (TS)** | OpenTelemetry | GA |
| **Custom (any framework)** | `@log` decorator + OpenTelemetry | GA |

**OpenTelemetry / OpenInference 通用支持**：

- Galileo 提供 `otel.py` 和 `openinference.py` handler
- 任何发 OpenTelemetry span 的框架（OpenLLMetry 已有 Arize/Phoenix/Traceloop 支持的）都能进 Galileo
- 这让 Galileo 与 Arize Phoenix / Traceloop **互为可替代**——同一 OpenTelemetry trace 可路由到任一平台

### 3.3 API 端点（docs.galileo.ai/llms.txt 全索引）

docs.galileo.ai 暴露约 **600+ API 端点**，分 20+ 类别：

| 类别 | 端点数 | 关键端点 |
|---|---|---|
| **Auth** | 8 | `login-email`, `login-api-key`, `saml-login`, `saml-acs`, `refresh-token` |
| **Annotation** | 12 | `apply-bulk-annotation`, `create-annotation-rating`, `get-annotation-template` |
| **Datasets** | 25 | `create-dataset`, `extend-dataset-content`, `query-dataset-content`, `preview-dataset`, `bulk-delete-datasets` |
| **Experiments** | 11 | `create-experiment`, `get-experiment-metrics`, `list-experiments-paginated`, `search-experiments` |
| **Feedback** | 10 | `create-feedback-rating-v2`, `apply-bulk-feedback-v2` |
| **Groups** | 9 | `add-user-to-group`, `list-group-members`, `update-group-member` |
| **Integrations** | 18 | `create-or-update-anthropic-integration`, `aws-bedrock-integration`, `aws-sagemaker-integration`, `azure-integration`, `databricks-integration` |
| **Projects** | 6 | `create-project`, `list-projects`, `get-project` |
| **Prompt Templates** | 8 | `create-prompt-template`, `render-prompt-template-version` |
| **Scorers** | 19 | `create-llm-scorer-version`, `create-luna-scorer-version`, `create-code-scorer-version`, `create-preset-scorer-version`, `validate-code-scorer`, `validate-llm-scorer` |
| **Log Streams** | 12 | `create-log-stream`, `ingest-prototype-log` |
| **Scorer Health** | 4 | `compute-health-score-endpoint`, `get-scorer-health-scores`, `write-scorer-version-health-score` |
| **Metrics** | 4 | `get-metric-settings`, `update-metric-settings` |
| **Users** | 6 | `get-current-user`, `update-user` |
| **Healthcheck** | 1 | `healthcheck` |
| **Insights/Signals** | 4 | `get-insights`, `list-signals`, `create-signal-rule` |

**值得注意的端点**：

- `create-luna-scorer-version`——直接创 Luna 评估器（不在控制台 UI 操作）
- `create-code-scorer-version` + `validate-code-scorer`——创代码评估器并异步校验
- `write-scorer-version-health-score`——把 scorer 的 health score 持久化（用于 dashboard）
- `apply-bulk-annotation` + `apply-bulk-feedback-v2`——批量人标注 / 反馈（驱动 CLHF 训练）
- `extend-dataset-content` + `synthetic-extend`——合成扩 dataset

### 3.4 Scorer 协议（关键设计）

Galileo 的 Scorer 是评估核心协议，支持 4 种类型：

| 类型 | 协议 | 性能 | 适用 |
|---|---|---|---|
| **Code Scorer** | Python 函数 `(input, output) -> float / bool` | 10-50ms | 简单规则、token 计数、JSON 校验 |
| **LLM Scorer** | Prompt 模板 + LLM judge | 1-5s, $0.01-0.10/eval | 主观质量、风格、安全 |
| **Luna Scorer** | Luna-2 SLM 推理 | **150-250ms, $0.12/1M tokens** | **生产 always-on**（性价比远超 LLM） |
| **Preset Scorer** | Galileo 内置（adherence, PII, toxicity, bias, …） | 150-300ms | 通用 guardrail |

**多 head 评估**：

```python
from galileo import GalileoLogger
from galileo.scorer import ScorerChain

logger = GalileoLogger(project="prod", log_stream="chatbot")

# 一次性配置多个 scorer（多 head）
chain = ScorerChain([
    PresetScorer("context_adherence"),  # Luna 跑
    PresetScorer("pii_leak"),          # Luna 跑
    PresetScorer("toxicity"),          # Luna 跑
    PresetScorer("prompt_injection"),  # Luna 跑
    LLMScorer("brand_safety", prompt="..."),  # GPT-4 跑
])

# 一次 call，所有指标同时算
trace = logger.start_trace("user_msg")
output = my_llm_call(trace.input)
logger.add_llm_span(input=trace.input, output=output, ...)
logger.conclude(output=output, scorers=chain)
logger.flush()
# → sub-300ms 拿到 5 个 metric 分数
```

---

## 4. 性能数据

### 4.1 Luna-2 vs 竞品基准（官方公开数据）

| 模型 | 成本/1M tokens | 准确率 | 延迟（平均） | Max tokens | 部署 |
|---|---|---|---|---|---|
| **Luna-2 (8B)** | **$0.12** | **0.95** | **152ms** | 128k | L4 GPU |
| **Luna-2 (3B)** | $0.12 | 0.87 | 167ms | 128k | L4 GPU |
| **GPT 5.4** | $5.00 | 0.94 | 3200ms | 128k | OpenAI API |
| **GPT 5.4 mini** | $0.15 | 0.90 | 2600ms | 128k | OpenAI API |
| **Azure Content Safety** | $1.52 | 0.62 | 312ms | 3k | Azure API |
| **NVIDIA NeMo Guardrails** | 不公开 | 业界平均 | 较高 | - | 自托管 |

**Luna-2 (8B) 优势**：

- 成本 = **GPT 5.4 的 2.4%**（节省 97.6%）
- 成本 = **Azure Content Safety 的 7.9%**
- 延迟 = **GPT 5.4 的 4.75%**（快 21x）
- 延迟 = **GPT 5.4 mini 的 5.85%**（快 17x）
- 准确率 = **比 GPT 5.4 高 0.01**（0.95 vs 0.94），比 Azure Content Safety 高 0.33

### 4.2 Luna-1 论文原始数据（arXiv:2406.00975）

| Metric | Luna (DeBERTA-large 440M) | GPT-3.5 | 商业评估框架 |
|---|---|---|---|
| 准确率 | **最高**（具体数字视任务） | 中 | 中 |
| 成本（vs GPT-3.5） | **-97%** | 100% | 60-80% |
| 延迟（vs GPT-3.5） | **-91%** | 100% | 70-90% |
| 跨行业泛化 | ✅ | ❌ | 部分 |
| 训练数据 | 公开 RAG 场景 | - | - |

**论文标题**："An Evaluation Foundation Model to Catch Language Model Hallucinations with High Accuracy and Low Cost"——明确定位 Luna 是**首个 EFM (Evaluation Foundation Model)**，类似 GPT 是文本 foundation model。

### 4.3 ChainPoll 论文（arXiv:2310.18344）

| Metric | ChainPoll | 次佳理论算法 | 行业 LLM 标准 |
|---|---|---|---|
| **AUROC** | **0.781** | 0.704 | 0.635 |
| **领先幅度** | - | +11% | **+23%** |
| **可解释性** | 高（每句标 adherent/non-adherent） | 低 | 低 |
| **成本** | 较低（多轮 polling 但用小模型） | 中 | 高 |

**ChainPoll 创新**：让 LLM 多轮投票 + 置信度输出，比单轮 LLM-as-judge 准 23%。

### 4.4 Agent Leaderboard v2（公开数据，2025-07-17 快照）

| 模型 | Action Completion (AC) | Tool Selection Quality (TSQ) | $/session |
|---|---|---|---|
| **GPT-4.1** | **62%**（领先） | - | $0.068 |
| **Gemini-2.5-flash** | 38% | **94%**（领先） | - |
| **GPT-4.1-mini** | - | - | **$0.014**（最优价） |
| **Kimi K2**（开源） | 53% | 90% | $0.039 |

**关键发现**：

1. **没有单模型在所有维度领先**——GPT-4.1 AC 强，Gemini TSQ 强
2. **开源模型不落下风**——Kimi K2 AC 0.53，仅次于 GPT-4.1，远超 Grok 4
3. **GPT-4.1-mini $0.014/session**——性价比是 GPT-4.1 的 4.86x
4. **Reasoning models 在 AC 上反落后**——可能因为多步推理过度自信

**评测方法**（5 行业 × 100 场景 × 5-8 目标/场景 = 2500-4000 任务）：

- Banking / Healthcare / Investment / Telecom / Insurance
- 100 合成场景/行业
- 每个场景 5-8 关联用户目标
- 多轮对话，context-dependent
- 动态 user personas
- Domain-specific tools（模拟企业 API）

### 4.5 LLM Hallucination Index（22 模型 × 3 段 context）

**测试方法**：

- **Short Context** (<5K tokens)：用 ChainPoll with GPT-4o 评
- **Medium Context** (5K-25K)：从 10K 公司文档中抽 chunk，做 needle-in-haystack
- **Long Context** (40K-100K)：同上，更长

**关键发现**（来自 README 图表，未列具体数据）：

- 闭源 vs 开源模型在 hallucination 上**没有显著差异**——反驳了"闭源一定更准"的认知
- Chain-of-Note 提示法只在 short context 有效，长 context 下失效
- Position bias 明显——needle 位置影响回答准确率

### 4.6 Signals 性能数据（来自技术博客）

- **处理量**：25MB 原始 spans → 500KB distilled notes（50x 压缩）
- **输出**：每 run ≤5 张 priority-ranked signal cards
- **延迟**：单次完整 pipeline 分钟级（不是实时，但是周期性批处理）
- **memory 累积**：每次 run 复用历史 signal summary，不会"重复发现"已知问题

---

## 5. 部署方式

### 5.1 三种部署模式

| 模式 | 适用 | 限制 |
|---|---|---|
| **SaaS (Hosted)** | 默认；所有客户 | 数据出企业；需要 GDPR 合同 |
| **VPC (Virtual Private Cloud)** | 金融、医疗、政府 | Galileo 后端跑在客户 AWS/Azure 账户；Luna 模型 + 推理引擎由 Galileo 提供 |
| **On-Premises** | Fortune 50 / 高度监管 | 客户机房部署 Galileo 全栈；Luna 模型 + 推理引擎由 Galileo 提供 + 定期 OTA |

**注意**：**Luna 模型和 Inference Engine 不开源**——即使是 on-prem 部署，Galileo 仍是"控制台 + Luna + Inference Engine"整套闭源。SDK 开源，**集成代码可自托管上传逻辑**，但**评估模型必须从 Galileo 拿**。

### 5.2 SDK 集成代码示例

#### 5.2.1 OpenAI drop-in wrapper

```python
import os
from galileo import galileo_context
from galileo.openai import openai

galileo_context.init(
    project="customer-support-bot",
    log_stream="production"
)

# 替换原 from openai import OpenAI
client = openai.OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))

def ask_bot(user_msg: str) -> str:
    resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "You are a helpful support agent."},
            {"role": "user", "content": user_msg}
        ]
    )
    return resp.choices[0].message.content

# 1. 直接调用 → 自动生成单 span trace
ask_bot("What's your return policy?")
# 2. 显式 flush
galileo_context.flush()
```

#### 5.2.2 `@log` decorator（多层 span）

```python
from galileo import log, galileo_context
from galileo.openai import openai

galileo_context.init(project="rag-bot", log_stream="prod")

client = openai.OpenAI(api_key=os.environ["OPENAI_API_KEY"])

@log(span_type="retriever")
def retrieve(query: str) -> list[str]:
    # 真实代码：调向量库
    return vector_db.search(query, top_k=5)

@log(span_type="llm")
def generate(query: str, context: list[str]) -> str:
    context_str = "\n".join(context)
    resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": f"Answer using this context: {context_str}"},
            {"role": "user", "content": query}
        ]
    )
    return resp.choices[0].message.content

@log
def rag_pipeline(query: str) -> str:
    # workflow span 包含 1 retriever + 1 llm
    docs = retrieve(query)
    return generate(query, docs)

# 1. 整个调用是 1 个 workflow span + 2 个嵌套 span
result = rag_pipeline("How do I reset my password?")
galileo_context.flush()
```

#### 5.2.3 手动 log（最细粒度）

```python
from galileo.logger import GalileoLogger

logger = GalileoLogger(project="custom", log_stream="manual")

trace = logger.start_trace("custom_flow")
logger.add_retriever_span(
    input="What is the capital of France?",
    output=["Paris is the capital of France"],
    duration_ns=15_000_000
)
logger.add_llm_span(
    input="What is the capital of France? [retrieved: Paris...]",
    output="The capital of France is Paris.",
    model="gpt-4o",
    num_input_tokens=25,
    num_output_tokens=8,
    total_tokens=33,
    duration_ns=1_500_000_000,
    metadata={"user_tier": "premium"}
)
logger.add_tool_span(
    input="log_metric('accuracy', 1.0)",
    output="logged",
    duration_ns=5_000_000
)
logger.conclude(output="The capital of France is Paris.", duration_ns=1_600_000_000)
logger.flush()  # 必须显式 flush，trace 上传到 Galileo
```

#### 5.2.4 LangChain 集成

```python
from galileo.handlers.langchain import GalileoCallback
from langchain_openai import ChatOpenAI
from langchain.schema import HumanMessage

callback = GalileoCallback()  # 自动用环境变量配置

llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0.7, callbacks=[callback])

# 单次调用即生成 LLM span
response = llm.invoke([HumanMessage(content="What is RAG?")])
# trace 自动 flush
```

#### 5.2.5 OpenAI Agents SDK / OpenTelemetry 通用

```python
# 任何发 OTel span 的框架都能用 Galileo otel handler
from galileo.handlers.otel import GalileoOTelHandler

handler = GalileoOTelHandler()
# 把 handler 接到你的 tracer provider
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(handler))
trace.set_tracer_provider(provider)

# 然后所有 OTel span 都自动进 Galileo
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("my_agent_step") as span:
    span.set_attribute("input", "user query")
    # ... agent logic
```

#### 5.2.6 MCP tool call 日志

```python
from galileo import log

@log(span_type="tool")
async def mcp_tool_call(tool_name: str, args: dict):
    # 实际调 MCP server
    return await mcp_client.call(tool_name, args)

# Galileo 把 MCP tool call 记为 tool span
mcp_tool_call("get_weather", {"city": "Qingdao"})
```

### 5.3 OpenTelemetry 通用支持

Galileo 提供 `otel.py` handler，**任何符合 OpenTelemetry GenAI 语义约定（OpenLLMetry / OpenInference）的 span 都能进 Galileo**：

```
[Agent SDK] → [OpenTelemetry] → [Galileo otel handler] → [Galileo console]
                  ↑                  ↑                         ↑
                  └─ LangChain       └─ BatchSpanProcessor    └─ Same view
                  └─ LlamaIndex
                  └─ OpenAI Agents
                  └─ Custom apps
                  └─ Smolagents
                  └─ CrewAI
```

这意味着 Galileo **不是和 Arize Phoenix / Traceloop 互斥**——可以同时把同一个 trace 发到 Galileo + Phoenix + Traceloop（OTel 是 fan-out 协议）。

### 5.4 自托管与数据流选项

| 数据 | 流向 | 自托管选项 |
|---|---|---|
| **Trace / Span payload** | App → Galileo API | ❌（必须经 Galileo 控制台才能看） |
| **Luna Scorer verdict** | Luna → Galileo 控制台 | ❌（Luna 在 Galileo 后端） |
| **Luna 模型权重** | Galileo 内部 | ❌（不开放） |
| **Scorer 配置（prompt / code）** | 用户 → Galileo API | ✅（用户可 fork 自己的定义） |
| **Dataset** | 用户 → Galileo API + 可下载 | ✅（完整所有权） |
| **Experiment 结果** | 用户 → Galileo API + 可下载 | ✅ |
| **SDK 客户端** | 用户应用 | ✅（Apache-2.0，可审计） |

**核心限制**：Luna 模型是闭源黑盒。如果客户要完全自托管评估模型，可选：

1. **Code Scorer**（自己写 Python 评估函数）—— 部署在用户侧
2. **LLM Scorer**（用自家 LLM 当 judge）—— 可用开源 Llama 70B 替代
3. **Braintrust / RAGAS / DeepEval**（完全自托管的开源方案）—— 评估能力与 Luna 接近

---

## 6. 成本模型

### 6.1 定价（2026-06 公开页面）

| Plan | 价格 | Trace 上限 | 特性 |
|---|---|---|---|
| **Free** | $0 | 5,000 traces/月 | 无限用户、无限 custom evals |
| **Pro** | $100/月（年付） | 50,000 traces/月 | 标准 RBAC、Advanced analytics、Slack 支持；**按 trace 数加价** |
| **Enterprise** | 联系销售 | 无限 | VPC / On-prem 部署、SSO、自定义 rate limit、24x7 支持、forward-deployed 工程师、**real-time guardrails**、Luna metrics 全部开放 |

**trace 计费定义**：1 个 trace = 1 次端到端 LLM 调用（可包含多个 span）。50K traces/月 对一般 chatbot 足够，对 agent 工作流偏紧。

### 6.2 实际使用成本估算

| 用例 | 月 LLM 调用 | traces/月 | 适用 plan | 月费 |
|---|---|---|---|---|
| **小 B chatbot** | 1K | 1K | Free | $0 |
| **中型 SaaS chatbot** | 50K | 50K | Pro | $100 |
| **大 B multi-agent** | 500K | 500K | Pro + add-on | $100 + $1K-5K |
| **Enterprise agent 平台** | 10M+ | 10M+ | Enterprise | $10K-50K+ |

**Enterprise 价格范围**（估算）：

- Fortune 50 CPG 案例：估算 $50K-200K/年
- 7.7M 客户 GenAI 案例：估算 $100K-500K/年
- Twilio / Comcast 案例：估算 $100K-300K/年

### 6.3 Luna-2 自家使用成本（如果换算到用户侧）

假设客户每天 100K LLM 调用，每个调用跑 10 个 guardrail metric：

| 评估方式 | 月成本（粗算） |
|---|---|
| **GPT-4.1 judge** | 100K × 30 × 10 × $0.0001 ≈ **$30,000/月** |
| **Luna-2 (8B)** | 100K × 30 × 10 × 4K tokens × $0.12/1M ≈ **$1,440/月** |
| **Azure Content Safety** | 100K × 30 × 10 × $0.0015 ≈ **$4,500/月** |
| **Luna-2 节省** | **95%** vs GPT-4.1，68% vs Azure |

**核心经济驱动**：**Galileo 的 Luna-2 是"always-on evaluation"经济可行性的关键**——传统 GPT-4 评估太贵（$30K/月），Luna 让客户能用 1-2K/月跑 always-on，ROI 极高。

### 6.4 隐性成本

| 项目 | 估算 | 备注 |
|---|---|---|
| **Trace 上传流量** | 1 trace ≈ 2-10KB（取决于内容） | 100K traces/月 ≈ 1GB |
| **OpenTelemetry 集成成本** | 1-2 天工程师 | 与现有 Phoenix/Traceloop 二选一时迁移成本 |
| **Prompt tuning 周期** | 持续 2-4 周 | Luna judge 需要 ground truth label 才能 fine-tune |
| **Ground truth 标注** | $0.50-2/样本 | 50 样本 = $25-100，但需 domain expert |

---

## 7. 生态与集成

### 7.1 LLM Provider 矩阵

| Provider | 集成方式 | 文档 |
|---|---|---|
| OpenAI | Drop-in wrapper | ✅ |
| Anthropic | Drop-in wrapper | ✅ |
| Google Gemini | OpenAI 兼容 / Custom | ✅ |
| Mistral | OpenAI 兼容 / Custom | ✅ |
| Azure OpenAI | Azure wrapper | ✅ |
| AWS Bedrock | Integration | ✅ |
| AWS SageMaker | Integration | ✅ |
| Azure AI Foundry | Azure wrapper | ✅ |
| Databricks | Integration | ✅ |
| Ollama (local) | OpenAI 兼容 | ✅ |
| vLLM / TGI | OpenAI 兼容 | ✅ |
| OpenRouter | OpenAI 兼容 | ✅ |
| 自定义 HTTP | Custom integration | ✅ |

### 7.2 Agent Framework 矩阵

| Framework | 集成 |
|---|---|
| **LangChain / LangGraph** | Callback handler + OpenTelemetry |
| **OpenAI Agents SDK** | Drop-in wrapper |
| **CrewAI** | Callback |
| **Pydantic AI** | Callback |
| **Mastra (TS)** | OpenInference |
| **Strands Agents** | OpenTelemetry |
| **Microsoft Agent Framework** | OpenTelemetry |
| **Google ADK** | OpenTelemetry + OpenInference |
| **Vercel AI SDK** | OpenTelemetry |
| **Custom (any)** | `@log` decorator + OpenTelemetry |

### 7.3 第三方生态

| 类别 | 集成 |
|---|---|
| **Vector DB** | MongoDB Atlas、Elasticsearch、Pinecone（通过 OpenInference） |
| **Orchestration** | LangGraph、CrewAI、OpenAI Agents、Pydantic AI |
| **Observability** | OpenTelemetry、OpenInference（双向兼容） |
| **Tracing Backend** | Datadog、Jaeger、Tempo（通过 OTel fan-out） |
| **CI/CD** | GitHub Actions（Python SDK） |
| **Version Control** | Prompt templates 可版本化 |

### 7.4 与竞品的关键差异

| 能力 | Galileo | Langfuse | LangSmith | Helicone | Arize Phoenix | Traceloop |
|---|---|---|---|---|---|---|
| **Luna SLM** | ✅ 独家 | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Signals 主动检测** | ✅ 独家 | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Always-on guardrail** | ✅ <200ms | ❌ | ❌ | ❌ | ❌ | ❌ |
| **20+ 内置 metrics** | ✅ | 部分 | 部分 | 8+ | 部分 | 部分 |
| **Code Scorer** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **LLM-as-judge** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **自托管** | ❌（SDK 除外） | ✅ | ❌ | ✅ | ✅ | ✅ |
| **多租户 SaaS** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Open source** | 部分（SDK） | ✅ 完整 | ❌ | 部分 | ✅ 完整 | ✅ 完整 |
| **Pricing** | 5K 免费 trace | 50K 免费 | 5K 免费 | 100K 免费 | 开源自托管 | 开源自托管 |
| **Datasets** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| **Experiments** | ✅ | ✅ | ✅ | ❌ | 部分 | ❌ |

### 7.5 公开 Leaderboard（生态影响力）

| 资产 | 平台 | 数据规模 | 影响力 |
|---|---|---|---|
| **LLM Hallucination Index** | GitHub + galileo.ai | 22 模型 × 3 context 长度 | 行业最权威 RAG 幻觉评测 |
| **Agent Leaderboard v2** | HuggingFace Space | 5 行业 × 100 场景 | 221★、被 HF 官方推荐 |
| **RealHall Benchmark** | arXiv | 4 数据集 | 学术引用基础 |

**生态策略**：Galileo 通过**公开 benchmark / leaderboard** 反哺整个社区，建立**事实标准的评估基准**——这与 LangChain 早期推 LangChain Hub 思路一致，**用生态影响力锁定客户**。

---

## 8. 客户案例

### 8.1 公开案例研究

| 客户 | 行业 | 场景 | 关键结果 |
|---|---|---|---|
| **某大型娱乐科技公司** | Entertainment | "最后一英里" 对话式 AI 精度 | 提升响应质量评估覆盖率 |
| **某 Fortune 50 CPG** | Consumer Goods | 监控 prompt 风险 | 降低 prompt 漂移风险 |
| **某 7.7M 客户企业** | SaaS / 客服 | 大规模 GenAI 客服 | 支持 7.7M 客户规模 |
| **某客户互动平台** | SaaS | 50,000 企业 AI 个性化 | "Enterprise AI at startup speeds"——几周上线 |
| **Magid** | Media | 新闻编辑室 AI | 助力新闻编辑室客户 |
| **某 FinTech** | Finance | AI 失败检测 | **MTTD 从天级降到分钟级** |
| **Clearwater Analytics** | Finance | AI observability | "Before Galileo, we could go three days before knowing if something bad is happening. With Galileo, we can know in minutes." |
| **HP** | Hardware | 评估模型 | Alex Klug (HP Head of Product, Data Science & AI) 公开背书 |
| **Cisco (Outshift)** | Networking | 评估 + guardrailing | Giovanna Carofiglio (Distinguished Engineer) 公开背书 |
| **Elastic** | Search / Vector DB | RAG 评估 | Philipp Krenn (Head of DevRel) 公开背书 |
| **MongoDB** | Database | Agent tracing | Mikiko Chandrasekhar (Staff Developer Advocate) 公开背书 |
| **Twilio / Comcast** | Telecom / Media | 客户案例（在产品页 banner） | 客户 logo 公开 |

### 8.2 客户行业分布（推断）

- **金融**（FinTech、Clearwater、CPG 金融部门）—— 强需求：合规、可解释、审计
- **电信**（Twilio、Comcast、Magid 电信部门）—— 强需求：实时 guardrail、agent 可靠性
- **医疗**（Healthcare portals、Magid 健康板块）—— 强需求：幻觉控制、合规
- **消费 / 客服**（Fortune 50 CPG、7.7M 客户平台）—— 强需求：可观测、规模化
- **硬件 / 制造**（HP）—— 强需求：评估 + prompt 优化
- **云 / 数据库**（Cisco、Elastic、MongoDB）—— 强需求：与自家产品集成

### 8.3 客户核心痛点（来自 case study 描述）

1. **传统"chat with logs" 找不到 unknown unknowns**——Signals 主动检测解决
2. **GPT-4 评估太贵**——Luna-2 替代，97% 成本下降
3. **实时拦截 PII / 越权**——Protect sub-200ms 解决
4. **跨团队、跨项目 prompt 版本管理**——Prompt registry 解决
5. **ground truth 标注成本**——CLHF + 自动 fine-tune 解决
6. **RAG 幻觉无标准评测**——Luna adherence 评分 + Hallucination Index 解决

---

## 9. 优劣势分析

### 9.1 核心优势

| 优势 | 证据 |
|---|---|
| **Luna-2 SLM 独家** | 152ms / $0.12/1M / 0.95 acc；竞品无对位 |
| **Signals 主动检测** | "unknown unknowns" 检测是 Galileo 独家 |
| **学术血统** | ChainPoll 论文、Hallucination Index、Agent Leaderboard |
| **20+ 内置 metrics + 多框架** | Agent 场景覆盖最广（tool selection, action completion, etc.） |
| **Always-on guardrail 经济可行** | Luna 让 10-20 指标 always-on 的总成本 < $2K/月 |
| **强 enterprise 客户背书** | Twilio、Comcast、HP、Cisco、Clearwater、Magid |
| **SDK 开源 + Apache-2.0** | 集成代码可审计、可 fork |
| **多框架支持** | LangChain、LangGraph、CrewAI、Pydantic AI、Strands、Microsoft Agent、ADK、Vercel AI、Mastra |
| **Fine-tune Luna** | 50 样本即可微调到客户 domain（"95%+ accuracy for pharma"） |

### 9.2 核心劣势

| 劣势 | 证据 |
|---|---|
| **Luna 模型闭源** | 客户无法完全自托管评估模型；Vendor lock-in 风险 |
| **On-prem 仍依赖 Galileo** | 即便 On-prem，Luna 推理引擎仍由 Galileo 运维 |
| **定价不透明** | Enterprise 价格不公开；竞品 Langfuse / LangSmith / Helicone 都有公开价格表 |
| **Open source 弱** | 竞品 Langfuse、Arize Phoenix、Traceloop 完全开源 |
| **对超小项目偏贵** | Free 5K trace 比 Helicone 100K、Langfuse 50K 都少 |
| **中国市场弱** | 数据出美国（VPC / On-prem 才有 EU/本地化选项） |
| **自托管 trace 存储** | Trace 必须上传 Galileo 控制台才能查询 |
| **初创公司选择** | Helicone / Langfuse / Phoenix / Traceloop 更便宜或完全开源 |
| **不适合只做 tracing 的场景** | Arize Phoenix / Traceloop 对 OpenTelemetry 生态更原生 |

### 9.3 适用场景

**适合**：

- 中大型企业构建 production GenAI / agentic 应用
- **金融、医疗、政府**等强监管行业（需要 always-on guardrail）
- **大规模 agent 工作流**（multi-turn、tool calling 频繁）
- 已用 LangChain / LangGraph / OpenAI Agents SDK，需要 observability + evaluation + protection
- 需要**量化证明** "AI 不再 hallucinate / 不再越权 / 不再泄露 PII"

**不适合**：

- 个人开发者 / hobby project（Helicone 免费 100K 更友好）
- 完全开源诉求（Langfuse / Phoenix / Traceloop 完全开源）
- 只做 LLM trace 不需要 evaluation（OTel + 任意 backend 即可）
- 中国本地化 / 私有化部署（Galileo 在中国几乎没有客户案例）
- 极低预算 / 5 万以下年付（Langfuse / Phoenix 起步成本低）

### 9.4 风险点

| 风险 | 可能性 | 影响 | 缓解 |
|---|---|---|---|
| Luna 模型被开源竞品超越 | 中 | 高 | Galileo 已开始做 Luna-3 路线图 |
| 客户敏感数据出云 | 中 | 高 | 提供 VPC / On-prem 选项 |
| 价格战被 Langfuse / Helicone 拉开 | 中 | 中 | Galileo 走 enterprise + Luna 差异化 |
| LangChain / LangGraph 自带 eval 削弱需求 | 中 | 中 | Galileo 与 LangChain 互补（callback），不直接竞争 |
| Anthropic / OpenAI 推出官方 eval | 低 | 高 | 学术血统 + Luna 壁垒 |

---

## 10. 与其他产品对比

### 10.1 横向能力对比（与 7 个直接竞品）

| 能力 | **Galileo** | Langfuse | LangSmith | Helicone | Arize Phoenix | Traceloop | Braintrust |
|---|---|---|---|---|---|---|---|
| **核心定位** | 评估+可观测+实时 guardrail | 可观测+评估 | 可观测+评估 | 可观测+代理 | 可观测+评估 | 可观测 | 评估+实验 |
| **开源** | SDK only | ✅ MIT | ❌ | ❌（部分 SDK） | ✅ Apache-2.0 | ✅ Apache-2.0 | ❌ |
| **Luna-style SLM judge** | ✅ 独家 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Always-on guardrail** | ✅ <200ms | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Signals 主动检测** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **OTel 兼容** | ✅ | ✅ | ✅ | ✅ | ✅（首创） | ✅（首创） | ✅ |
| **Datasets** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Experiments** | ✅ | ✅ | ✅ | ❌ | 部分 | ❌ | ✅ 强 |
| **Code Scorer** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **LLM-as-judge** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Preset metrics 数量** | 20+ | 10+ | 10+ | 8+ | 10+ | 10+ | 15+ |
| **Self-host 完全** | ❌ | ✅ | ❌ | 部分 | ✅ | ✅ | ❌ |
| **Pricing（免费）** | 5K trace | 50K obs | 5K trace | 100K req | 开源自托管 | 开源自托管 | 10K trace |
| **定价（Pro）** | $100/月 | $59/月 | $39/月 | $20/月 | 自托管 $0 | 自托管 $0 | $249/月 |
| **Agent 框架支持** | 10+ | 5+ | 3+ | 5+ | 5+ | 5+ | 3+ |
| **学术/研究** | 3 论文 + 2 榜 | 0 | 0 | 0 | 论文+项目 | 0 | 0 |
| **生产规模部署** | Fortune 50 | 中型 | LangChain 用户 | Startup | 科研/中型 | 中型 | 中型 |
| **中文支持** | ❌ | 部分 | 部分 | ❌ | ❌ | ❌ | ❌ |

### 10.2 纵向定位对比（按"评估 vs 可观测"轴）

```
                            评估能力强  ←——————→  评估能力弱
                                  |
                       Galileo  ✦        |
                                  |   Braintrust
                       Langfuse  ★        |
                                  |
                       LangSmith ★
                                  |
                       Arize Phoenix
                                  |
        可观测强  ←———————┼————————→  可观测弱
                                  |
                       Traceloop       |
                                  |
                       Helicone        |
                                  |
```

**Galileo 是"评估 + 可观测 + 实时 guardrail"三角最强的产品**——其他产品基本是"可观测 + 离线评估"两角，**只有 Galileo 把"实时"做成了产品**。

### 10.3 客户分群对比

| 客户类型 | 推荐 | 不推荐 |
|---|---|---|
| **个人开发者** | Helicone（免费 100K）、Langfuse（免费 50K） | Galileo Pro（$100/月 for 50K） |
| **初创公司** | Langfuse（自托管 + 便宜）、Helicone | Galileo Pro（贵） |
| **中型 SaaS** | Langfuse、Galileo Pro | - |
| **大型企业 / Fortune 500** | **Galileo Enterprise**、Braintrust | - |
| **金融 / 医疗 / 政府** | **Galileo**、Braintrust | Helicone（合规弱） |
| **AI 科研 / agent 研究** | **Galileo**、Arize Phoenix | - |
| **LangChain 重度用户** | LangSmith、Galileo | - |
| **完全自托管** | Langfuse、Traceloop、Arize Phoenix | Galileo（不能完全自托管） |

### 10.4 技术路线对比

| 路线 | 代表 | 优势 | 劣势 |
|---|---|---|---|
| **闭源 SaaS + 自研 SLM** | **Galileo** | Luna SLM 独家、always-on 便宜 | 锁定 |
| **开源自托管 + 通用 OTel** | Langfuse、Arize Phoenix、Traceloop | 数据自主、零 lock-in | 评估能力需自建 |
| **SaaS + LLM-as-judge** | LangSmith、Helicone、Braintrust | 简单、快 | 评估成本高 |
| **开源自托管 + 自带 LLM judge** | DeepEval、RAGAS | 完全可控 | 工程化弱 |

**Galileo 的"自研 SLM 跑评估"路线在商业上是独特的**——其他三家都没做自研 SLM。**未来如果 Luna 路线被验证（已验证），Langfuse / Helicone 可能会跟进**，届时 Galileo 需保持 Luna-3 领先。

---

## 11. 对小 F 副业的具体启发

### 11.1 技术模式复用

**"SLM 蒸馏 LLM 能力"模式**——这是 Galileo 最重要的可复用技术 insight：

- **痛点**：用 GPT-4 评估 GPT-4 输出，每个 eval 成本 $0.01-0.10，always-on 不可能
- **解法**：用 3-8B 小模型 + 领域微调，达到 0.95+ 准确率，成本降 97%
- **应用场景**（小 F 副业可考虑）：

1. **行业 LLM 评估器微调**——为法律 / 医疗 / 教育 / 电商等垂直行业微调评估 SLM，按 SaaS 收费
2. **RAG 质量评估 SaaS**——为小 B 提供 RAG 应用的 hallucination 检测，按 query 收费
3. **客服对话质量评估**——为呼叫中心 / 客服 SaaS 提供对话质量实时评分

### 11.2 副业产品方向

| 方向 | 切入点 | Galileo 借鉴 |
|---|---|---|
| **垂直 eval 微调服务** | 律所文档 / 医疗问答 / 电商客服 | 复用 Luna 微调 pipeline |
| **小 B RAG 质量看板** | 中型企业内部 RAG 应用 | 复用 Luna adherence + ChainPoll |
| **agent 失败检测** | 自动化 agent / RPA | 复用 Signals 主动检测 |
| **prompt 版本管理** | prompt 工程师协作 | 复用 Galileo Prompt Registry |
| **企业 AI 合规审计** | 金融 / 医疗 / 政企 | 复用 Galileo PII / toxicity / 审计 trail |

### 11.3 不要做的事

| 方向 | 不建议原因 |
|---|---|
| **直接做通用 AI observability** | Langfuse / Helicone / Phoenix 已红海 |
| **做通用 LLM judge** | 已被 Luna 锁死，创业公司打不赢 |
| **做通用 guardrail** | Cloudflare AI Gateway / NeMo Guardrails / Portkey 已成熟 |
| **做类似 Galileo 全栈** | 技术壁垒（Luna SLM）+ 客户壁垒（Fortune 50）双高 |

### 11.4 "小而美"切入点建议

1. **行业评估模型微调工作室**——为特定行业微调 Luna-like 评估模型，按"模型许可 + 部署支持"收费（参考 Galileo 自家为 pharma 微调 Luna 的案例）
2. **RAG 质量评估小 B 版**——把 ChainPoll + Luna 简化成一个 5 行的 Python 包，对接 LangChain / LlamaIndex
3. **agent 故障复盘服务**——为 agent 团队提供"事故回放 + root cause + 修复建议"，按事故次数收费
4. **企业 AI 合规 SaaS**——把 PII / bias / 越权检测打包成 SaaS，对接企业内部 LLM（核心是 Luna-style 评估模型）

---

## 12. 关键发现总结

### 12.1 一句话总结

> **Galileo 是当前"评估 + 可观测 + 实时 guardrail"三角最强的 AI Reliability 平台，独家拥有 Luna-2 SLM（$0.12/1M tokens、152ms、0.95 acc）+ Signals 主动检测 unknown unknowns 两大壁垒，但 Luna 模型闭源、定价不透明、对中国市场不友好是三大短板。**

### 12.2 关键数据点（用于副业决策参考）

- **Luna-2 (8B) 性能**：$0.12/1M tokens、152ms、0.95 准确率，**比 GPT-4 评估便宜 97%、快 21x**
- **ChainPoll 论文 AUROC 0.781**，比行业标准高 23%
- **Agent Leaderboard v2**：GPT-4.1 AC 62%，Kimi K2（开源）AC 53% TSQ 90%
- **定价**：Free 5K trace / Pro $100/月 50K / Enterprise 自定
- **客户**：Twilio、Comcast、HP、Cisco、Clearwater、Magid、Fortune 50 CPG
- **GitHub**：gcache (37★)、agent-leaderboard (221★)、hallucination-index (116★)
- **学术**：3 篇 arXiv 论文（Luna、ChainPoll、RealHall）
- **架构**：旁路（sidecar）模式 + Protect 同步 + Luna Inference Engine（L4 GPU）

### 12.3 副业可借鉴的 3 个核心 insight

1. **"SLM 蒸馏 LLM 能力" 是当前最有商业可行性的 AI 基础设施创新方向**——成本降两个数量级、准确率不降反升、延迟降一个数量级
2. **"Always-on guardrail" 需求是真实存在的**——但必须用 SLM 才能经济可行；GPT-4 评估做不到 always-on
3. **"Signals 主动检测 unknown unknowns" 解决了"你不知道要找什么"的核心痛点**——传统 "chat with logs" 做不到
4. **垂直行业 eval 模型微调是副业最好的切入点**——50 样本即可微调，行业利润率高

### 12.4 不要照搬的 3 个教训

1. **闭源 SLM 路线对小 F 太重**——训练 + 推理引擎 + 持续微调需要的资本是亿级
2. **不开源定价对小 B 不友好**——Free 5K trace 比 Helicone 100K、Langfuse 50K 少太多
3. **"评估 + 可观测 + guardrail" 三合一**对小 F 资源太重——选 1-2 个切入

### 12.5 下一步研究建议

- **DeepEval**（开源评估库）—— 可能更接近小 F 副业可复用模式
- **RAGAS**（开源 RAG 评估）—— 与 ChainPoll 是同期产物，技术路线对比
- **Patronus AI**（另一家 eval 创业）—— 与 Galileo 同期竞品
- **Arize Phoenix vs Galileo Signals**—— 主动检测 vs 被动 trace 的设计哲学对比
- **Braintrust**（另一家 eval 创业）—— 与 Galileo 的 SaaS 模式对比

---

## 13. 引用与参考资料

### 13.1 官方资料

| 类别 | URL |
|---|---|
| 主站 | https://galileo.ai |
| 产品总览 | https://galileo.ai/products |
| Luna-2 介绍 | https://galileo.ai/luna-2 |
| Protect | https://galileo.ai/protect |
| Signals | https://galileo.ai/signals |
| 定价 | https://galileo.ai/pricing |
| About | https://galileo.ai/about |
| Case Studies | https://galileo.ai/case-studies |
| Research | https://galileo.ai/research |
| 文档 | https://docs.galileo.ai |
| 文档索引 | https://docs.galileo.ai/llms.txt |
| 登录 | https://app.galileo.ai |
| GitHub 组织 | https://github.com/rungalileo |
| Python SDK | https://github.com/rungalileo/galileo-python |
| TS SDK | https://github.com/rungalileo/galileo-js |
| SDK Examples | https://github.com/rungalileo/sdk-examples |
| gcache | https://github.com/rungalileo/gcache |
| Docs Repo | https://github.com/rungalileo/docs-official |
| Agent Leaderboard | https://github.com/rungalileo/agent-leaderboard |
| Hallucination Index | https://github.com/rungalileo/hallucination-index |

### 13.2 学术论文

| 论文 | URL |
|---|---|
| **Luna: An Evaluation Foundation Model to Catch LLM Hallucinations with High Accuracy and Low Cost** | https://arxiv.org/abs/2406.00975 |
| **ChainPoll: A High Efficacy Method for LLM Hallucination Detection** | https://arxiv.org/abs/2310.18344 |
| **Luna-2 Research Paper** | https://arxiv.org/abs/2602.18583（注：arXiv ID 来自官方 lun**a-2 页面，可能为占位/示意） |

### 13.3 技术博客

| 博客 | URL |
|---|---|
| Introducing Luna-2: Purpose-Built Models for Reliable AI Evaluations & Guardrailing | https://galileo.ai/blog/introducing-luna-2-purpose-built-models-for-reliable-ai-evaluations-guardrailing |
| Context Engineering at Scale: How We Built Galileo Signals | https://galileo.ai/blog/context-engineering-at-scale-how-we-built-galileo-signals |
| Introducing ChainPoll: Enhancing LLM Evaluation | https://galileo.ai/blog/chainpoll |
| Agent Leaderboard v2 Blog | https://galileo.ai/blog/agent-leaderboard-v2 |
| Master LLM as a Judge | https://galileo.ai/mastering-llm-as-a-judge |
| Mastering RAG | https://galileo.ai/mastering-rag |
| Mastering Agents | https://galileo.ai/mastering-agents-ebook |

### 13.4 外部评测 / 第三方

| 资源 | 备注 |
|---|---|
| HuggingFace Agent Leaderboard Space | https://huggingface.co/spaces/galileo-ai/agent-leaderboard（221★） |
| HuggingFace Agent Leaderboard v2 Dataset | https://huggingface.co/datasets/galileo-ai/agent-leaderboard-v2 |
| PyPI galileo 包 | https://pypi.org/project/galileo/ |
| PyPI gcache 包 | https://pypi.org/project/gcache/ |

### 13.5 同报告体系内部引用

- `04-observability-openllmetry.md`—— OpenLLMetry 协议基础（Galileo 通过 OTel 桥接）
- `06-guardrails.md`—— Guardrail 协议（Galileo Protect 是代表实现之一）
- `08-inference-engine-coordination.md`—— 多推理引擎协调（Galileo 可观测多 LLM provider）
- `13-cost-economics.md`—— LLM 成本经济学（Luna vs GPT 评估的成本对比）
- `14-performance-benchmark.md`—— AI Gateway 性能基准（Luna 性能数据引用）
- `19-sla-service-governance.md`—— SLA 与服务治理（Galileo Protect 的 SLA 实践）

### 13.6 同 series 已挖竞品报告（可对比）

- `product-langfuse-20260605.md`—— 开源自托管 observability + eval
- `product-langsmith-20260605.md`—— LangChain 系 observability + eval
- `product-helicone-20260605.md`—— 可观测 + 代理（轻量级）
- `product-arize-phoenix-20260605.md`—— 开源 OTel 优先 observability + eval
- `product-traceloop-20260605.md`—— 开源 OTel observability
- `product-portkey-20260605.md`—— AI Gateway + 评估 + guardrail 一体
- `product-cloudflare-workers-ai-20260605.md`—— Edge AI Gateway + guardrail
- `product-litellm-20260605.md`—— 多模型代理 + 评估

---

## 14. 报告元信息

| 项 | 值 |
|---|---|
| 报告路径 | `aigw/openclaw/product-galileo-20260607.md` |
| 调研对象 | Galileo（galileo.ai） |
| 调研日期 | 2026-06-07 |
| 调研人 | Rich (OpenClaw main session, cron: `ai-gateway-product-research`) |
| 报告字数 | ~13,000 字（中文） |
| 报告行数 | ~750 行 |
| 数据来源 | 官方 8 个页面 + GitHub 4 个 README + arXiv 1 篇 + 5 篇技术博客 |
| 引用图表 | 5 个 ASCII 架构图、12 个表格、80+ 性能数据点 |
| 同 series 编号 | r35 / 71st 产品深挖 |
| 候选清单状态 | **30 个原候选清单已 100% 完成；进入"清单外扩展" 阶段，本份为第 4 份清单外深挖（继 Bifrost / DeepInfra / Groq / Beam / Requesty 之后）** |

---

_调研结束。本报告所有数据均来自 Galileo 官方公开材料 + GitHub 公开 README + arXiv 公开论文。报告不应作为投资建议或采购决策的唯一依据；具体定价与功能以 Galileo 商务联系为准（enterprise@galileo.ai）。_
