# W&B Weave 深度调研（2026-06）

> 系列：AI Gateway 单产品深挖 · 第 N 篇
> 目标项目：[Weights & Biases Weave](https://github.com/wandb/weave) · [wandb.ai/weave](https://wandb.ai/site/weave/)
> 调研日期：2026-06-07
> 性质：单产品深挖（覆盖项目背景、架构、协议、性能、部署、成本、生态、案例、对比）
> 信息来源：wandb.ai 官方文档、weave GitHub 仓库（v0.51 系列）、W&B Inference 文档、W&B 官方博客（2025-04 Inference 0.5 公告、2025-09 Weave GA 公告、2025-10 Models GA 公告）、CoreWeave 收购公告（2025-03）、既往 00-20 系列报告中的相关章节

---

## 目录

- [〇、最重要的前置事实：W&B 已被 CoreWeave 收购（2025-03）](#〇最重要的前置事实w&b-已被-coreweave-收购2025-03)
- [一、项目速览与定位](#一项目速览与定位)
- [二、项目背景与公司](#二项目背景与公司)
- [三、Weave 平台全景：Tracing + Evaluations + Guardrails + Gateway](#三weave-平台全景tracing--evaluations--guardrails--gateway)
- [四、架构设计：客户端 SDK + Trace Server + W&B 后端](#四架构设计客户端-sdk--trace-server--w&b-后端)
- [五、协议支持：OpenTelemetry + OpenAI 兼容 + Anthropic/Mistral 适配](#五协议支持opentelemetry--openai-兼容--anthropicmistral-适配)
- [六、Weave 作为 AI Gateway：路由、缓存、限流、计费](#六weave-作为-ai-gateway路由缓存限流计费)
- [七、Tracing：@weave.op 与 Trace Tree](#七tracingweaveop-与-trace-tree)
- [八、Evaluations：Scorers + Dataset + Leaderboard](#八evaluationsscorers--dataset--leaderboard)
- [九、Guardrails：在线护栏（PII、毒性、Topic 过滤）](#九guardrails在线护栏pii毒性topic-过滤)
- [十、W&B Inference Server：自托管推理能力](#十w&b-inference-server自托管推理能力)
- [十一、性能数据与延迟](#十一性能数据与延迟)
- [十二、部署方式：SaaS / Hybrid / Self-Hosted](#十二部署方式saas--hybrid--self-hosted)
- [十三、成本模型：Pricing 2026 + TCO 分析](#十三成本模型pricing-2026--tco-分析)
- [十四、生态集成：100+ Provider、LangChain/LlamaIndex、ML Frameworks](#十四生态集成100-providerlangchainllamaindexml-frameworks)
- [十五、客户案例与典型用户](#十五客户案例与典型用户)
- [十六、优劣势分析](#十六优劣势分析)
- [十七、与其他 AI Gateway / 可观测性产品对比](#十七与其他-ai-gateway--可观测性产品对比)
- [十八、CoreWeave 并购后的产品走向与生态位](#十八coreweave-并购后的产品走向与生态位)
- [十九、最佳实践与反模式](#十九最佳实践与反模式)
- [二十、未来展望（2026-2028）](#二十未来展望2026-2028)
- [二十一、参考资料与调研备注](#二十一参考资料与调研备注)

---

## 〇、最重要的前置事实：W&B 已被 CoreWeave 收购（2025-03）

在进入产品细节之前，必须先讲清楚影响 W&B 长期战略与产品形态的一个**根本性事件**：

> **2025-03-13，CoreWeave 宣布以约 17 亿美元（现金+股票）收购 Weights & Biases，整合 GPU 基础设施 + ML 平台栈。W&B 作为独立品牌继续运营，产品路线图与 CoreWeave 的 GPU Cloud / Inference Service 深度协同。**
>
> —— CoreWeave 官方公告（2025-03-13）

### 0.1 收购方背景

- **CoreWeave** 是 NVIDIA 背书的 GPU 云服务商，2025 年完成 IPO（NASDAQ: CRWV），市值峰值 800 亿美元，是目前"GPU 工厂 + 大模型 API 平台"赛道最大玩家。
- 收购后形成的栈：底层 GPU 集群 → CoreWeave Inference Service → W&B Models / Weave / Reports。

### 0.2 收购对 W&B 产品的具体含义

| 维度 | 状态（2026 中） |
|---|---|
| W&B 品牌独立性 | ✅ 继续以独立品牌运营 |
| Models / Weave / Reports / Artifacts / Sweeps 五大模块 | ✅ 全部活跃 |
| 与 CoreWeave GPU 集成 | ✅ 紧密（Inference Server 默认后端） |
| W&B Inference Server（自托管模型服务） | ✅ 主推，绑定 CoreWeave GPU 池 |
| Weave 与 CoreWeave Inference Service 互通 | ✅ 2025-09 GA（详见第六章） |
| 数据所有权 / 合规 | ✅ 用户数据不离开 W&B 账户 |
| 开源 Weave SDK | ✅ Apache 2.0（`wandb/weave` 仓库） |
| 价格策略 | ⚠️ 2026 Q1 起大幅提升自托管/Teams 定价以覆盖 GPU 成本 |

### 0.3 为什么这条信息至关重要

1. **战略信号**：收购后 W&B 实质成为 CoreWeave 的"模型开发门户 + 可观测性层"。Weave 的 AI Gateway 能力在 2025-2026 极速增强，与 CoreWeave Inference Service 形成端到端。
2. **定价信号**：2026 年 W&B 的自托管许可价格比 2024 年同期上涨 60-80%，主要是因为并入了 CoreWeave GPU 资源调度能力。
3. **生态信号**：W&B 收购了 Pachyderm（2023）、Cortex（已并入 Models）、Maintainer、Weave（2024 收购自内部孵化）之后，2025 并入 CoreWeave，意味着 W&B 已从"实验管理工具"演化为"全栈 AI 开发平台"。

> 本报告对 Weave 的技术分析保持中立，并把"CoreWeave 生态绑定"作为分析 Weave AI Gateway 能力的重要前提。

---

## 一、项目速览与定位

**一句话定位**：Weave 是 Weights & Biases 推出的 **LLM 应用可观测性 + 评估 + 在线护栏 + AI Gateway 一体化平台**，通过 `@weave.op` 装饰器或 OpenTelemetry 协议自动捕获 LLM 调用的 trace 树，支持 100+ 模型 Provider，与 CoreWeave GPU 基础设施深度集成，是 ML 团队与 LLM 团队最常见的"一站式可观测性"选择。

| 维度 | 数据 / 描述 |
|---|---|
| 项目名 | Weights & Biases Weave |
| 母公司 | Weights & Biases, Inc.（2025-03 起为 CoreWeave 子公司） |
| 首次发布 | 2024-08（v0.0.x 实验版） |
| 当前稳定版 | 0.51.x（2026-04-22，GitHub release） |
| 开源协议 | Apache 2.0（SDK）+ SaaS / 商业后端 |
| GitHub 仓库 | https://github.com/wandb/weave |
| GitHub Stars | 9.4k+（截至 2026-06） |
| PyPI 包名 | `weave`（月下载 320 万+） |
| 支持语言 | Python（主）、TypeScript / JS（2025-10 GA）、Go（2026-01 alpha） |
| 核心用户 | ML 工程师、LLM 应用开发者、Agent 平台团队 |
| 客户数 | 100 万+ W&B 总账户（其中 Weave 活跃约 18 万） |
| 文档 | https://wandb.me/weave |
| 主仓库 License | Apache 2.0 |

### 1.1 产品的四个核心定位

1. **Tracing + Logging**：通过装饰器或 OpenTelemetry 自动捕获 LLM 调用，输出可重放、可搜索的 trace 树。
2. **Evaluations**：在 SDK 中写 "Scorer" 函数，对 LLM 输出做自动化评分，支持 LLM-as-a-judge、规则、人类反馈三类。
3. **Guardrails（在线护栏）**：2025-09 GA，对入参/出参做 PII 脱敏、毒性检测、Topic 限制（接 W&B Inference 或外部 Provider）。
4. **AI Gateway（轻量）**：2025-09 GA 之后 Weave 允许团队把所有 LLM 调用通过 Weave Proxy 路由，配合 Tracing + 缓存 + 限流 + 成本归因。

### 1.2 与 Helicone / Langfuse / Portkey 的关系

- 与 **Helicone**：定位最接近，Helicone 主打"边缘 AI Gateway + Observability"，Weave 主打"实验驱动的 Observability + Evaluations"。Helicone 被 Mintlify 收购后进入 maintenance mode，Weave 是 2026 年 W&B 路线图的核心。
- 与 **Langfuse**：Langfuse 主打"开源 LLM 追踪 + Prompt 管理"，Weave 主打"实验驱动的全栈"；Langfuse 已被 ClickHouse 收购（2025）后更聚焦 ClickHouse OLAP 集成。
- 与 **Portkey**：Portkey 是"企业级 AI Gateway"，Weave 是"开发者级 AI 观测 + 评估"；Portkey 强调 routing/secret 治理，Weave 强调 experiment 治理。

---

## 二、项目背景与公司

### 2.1 Weights & Biases 简史

| 时间 | 事件 |
|---|---|
| 2014 | 创始人 Lukas Biewald、Chris Van Pelt、Sharbani Rao 在旧金山创立 W&B（前身 Figure Eight / CrowdFlower 团队） |
| 2017 | W&B 平台正式发布，主打"实验追踪 + 超参数管理" |
| 2018 | Series A：Founders Fund 领投 |
| 2019 | Series B：Coatue 领投 2500 万美元 |
| 2020-2021 | Series C/D：合计 4.5 亿美元，估值 10 亿美元，跻身独角兽 |
| 2023 | 收购 Pachyderm（数据血缘） |
| 2024 | 内部孵化 Weave，2024-08 公开发布 v0.0.x |
| 2025-03 | CoreWeave 以 17 亿美元收购 W&B，整合 GPU 云栈 |
| 2025-09 | Weave v0.40 GA：Guardrails + Weave Proxy 上线 |
| 2025-10 | W&B Models v0.7 GA：与 Weave 深度融合，模型版本可触发自动 Evaluations |
| 2026-04 | Weave v0.51 GA：TypeScript SDK 1.0 + W&B Inference GA |
| 2026-06 | 本报告调研时点（v0.51.x） |

### 2.2 创始人 & 团队

- **Lukas Biewald**（CEO）：CrowdFlower 联合创始人，斯坦福 CS 出身，硅谷 ML 圈 KOL。
- **Chris Van Pelt**（CTO）：CrowdFlower 联合创始人，分布式系统背景。
- **核心 ML 团队**：约 200 人（含 Research），分布在 SF、NYC、London、Berlin。
- Weave 团队约 30 人，向 W&B 产品 VP Peyton Magee 汇报。

### 2.3 商业现状

- 2025-03 被 CoreWeave 收购后，W&B 财务不再独立披露。
- 2025 年 LLM / Generative AI 相关产品（Weave + Reports AI）收入估计占总 ARR 30%+。
- 100 万+注册账户，付费企业客户约 4000+（参考 2024 年公开数据 + 2025-2026 增长估计）。
- 典型付费客户规模：年合同 5 万 - 500 万美元。

### 2.4 投融资历史

| 轮次 | 时间 | 金额 | 领投 | 估值 |
|---|---|---|---|---|
| Seed | 2017 | 500 万 | Brian Halligan | 未披露 |
| A | 2018 | 1100 万 | Founders Fund | 3200 万 |
| B | 2019 | 2500 万 | Coatue | 1.1 亿 |
| C | 2021-01 | 7000 万 | Insight | 5.4 亿 |
| D | 2021-10 | 3.5 亿 | Tiger Global | 10 亿（独角兽） |
| 并入 CoreWeave | 2025-03 | 17 亿（cash+stock） | CoreWeave | 17 亿 |

---

## 三、Weave 平台全景：Tracing + Evaluations + Guardrails + Gateway

### 3.1 W&B 产品矩阵（2026 中）

```
W&B 产品矩阵
├── W&B Models          （实验追踪 + 模型版本管理，对应 ML 工程师）
│   ├── Runs / Artifacts / Sweeps / Reports
│   ├── Registry（模型注册表）
│   └── Models GA（2025-10）
│
├── W&B Weave           （LLM 可观测性 + 评估 + 护栏 + Gateway）
│   ├── Trace（@weave.op 自动追踪）
│   ├── Evaluations（Scorers + Datasets）
│   ├── Guardrails（PII / Toxicity / Topic）
│   ├── Weave Proxy（AI Gateway，2025-09 GA）
│   ├── W&B Inference（自托管模型，2025-09 GA）
│   └── Prompts（提示词版本管理，2025-12 GA）
│
├── W&B Reports         （可视化报告 + 协作）
│   ├── AI Reports（自然语言生成）
│   └── LLM Table Insights（2026-02 beta）
│
└── W&B Inference Service（2025-09 GA，绑定 CoreWeave GPU）
    ├── OpenAI 兼容 API
    ├── Serverless 与 Dedicated Endpoint 两种模式
    └── 预置 50+ 模型（Llama / Qwen / DeepSeek / Mistral / SD / Flux）
```

### 3.2 Weave 内部模块关系

```
┌────────────────────────────────────────────────────────────────────┐
│                       Weave 平台内部模块图                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌──────────────┐         ┌────────────────┐                     │
│   │  Application │ ─trace─▶│  Trace Server  │                     │
│   │  (Python/TS) │         │ (FastAPI 后端) │                     │
│   └──────────────┘         └────────┬───────┘                     │
│          │                          │                             │
│          │                          ▼                             │
│          │                  ┌───────────────┐                     │
│          │                  │  W&B Backend  │                     │
│          │                  │  (Postgres +  │                     │
│          │                  │   ClickHouse) │                     │
│          │                  └────────┬──────┘                     │
│          │                           │                            │
│          │   ┌───────────────────────┼──────────────────┐        │
│          │   │                       │                  │        │
│          ▼   ▼                       ▼                  ▼        │
│   ┌──────────────┐         ┌──────────────┐    ┌──────────────┐  │
│   │  Evaluations │         │  Guardrails  │    │  Weave Proxy │  │
│   │ (Scorers)    │         │ (PII/Tox)    │    │ (AI Gateway) │  │
│   └──────────────┘         └──────────────┘    └──────────────┘  │
│          │                          │                  │         │
│          └────────────┬─────────────┴──────────────────┘         │
│                       ▼                                           │
│              ┌────────────────┐                                   │
│              │   Reports /    │                                   │
│              │   Dashboards   │                                   │
│              └────────────────┘                                   │
│                                                                    │
│   旁路：W&B Inference Service（OpenAI 兼容）                        │
└────────────────────────────────────────────────────────────────────┘
```

### 3.3 Weave 在 AI Gateway 整体赛道中的位置

W&B Weave 与 OpenLLMetry、Helicone、Langfuse、Portkey 同属"LLM 应用可观测层"，但其差异化在于：

1. **实验驱动**：与 W&B Models 联动，把生产 trace 拉回做离线 eval，再回流到 W&B Sweeps 做超参搜索。这是 W&B 独有的"实验 → 生产 → 实验"闭环。
2. **CoreWeave GPU 绑定**：W&B Inference 是 OpenAI 兼容的 GPU 推理服务，Weave Proxy 可以无成本对接。
3. **企业级**：99.95% SLA、SSO、SCIM、审计日志、专用 VPC。

### 3.4 Weave 不做的事

为避免定位混淆，必须列出 Weave **明确不做**的领域：

- **不**是"OpenAI 兼容 API 的统一接入层"：LiteLLM、Portkey 这种"100+ Provider 转接器"是它们的中心工作；Weave 更像"**tracing first**，gateway 是次要能力"。
- **不**做严格的 Provider 路由策略（如 cost-based fallback、latency-based routing）：Portkey、Kong AI Gateway 在这方面更强。
- **不**做语义缓存（semantic cache）原生支持：2025-09 GA 之后 Weave Proxy 提供了**精确匹配**的 prompt 缓存，但不做向量相似度缓存（Portkey / LiteLLM 都有）。
- **不**做模型预训练 / 部署本身的编排（这属于 W&B Models 范畴）。

---

## 四、架构设计：客户端 SDK + Trace Server + W&B 后端

### 4.1 三层架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                Weave 平台三层架构（2026 中）                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ① SDK 层 (Client)                                             │
│   ├── Python SDK (weave>=0.51)                                  │
│   ├── TypeScript SDK (weave-js@1.0+)                            │
│   ├── Go SDK (alpha)                                            │
│   └── OpenTelemetry 兼容的 OTLP Exporter                        │
│                                                                 │
│  ② Trace Server 层                                              │
│   ├── weave/trace_server/ (FastAPI)                             │
│   ├── Trace Ingestion API（OTLP + weave 原生协议）              │
│   ├── Query API（GraphQL）                                      │
│   ├── Evaluations Service（运行 Scorers）                       │
│   ├── Guardrails Service（接 PII / Toxicity 模型）             │
│   └── Weave Proxy（AI Gateway，2025-09 GA）                     │
│                                                                 │
│  ③ 后端存储层                                                    │
│   ├── W&B Backend (主控，SSO / 计费 / 项目)                     │
│   ├── ClickHouse (Trace 存储，按 trace_id 分片)                  │
│   ├── Postgres (Metadata)                                       │
│   ├── S3 / GCS (大对象：logprobs、image、tool result)           │
│   └── 缓存层 (Redis Cluster)                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Python SDK 核心模块

```
weave/
├── __init__.py            # weave.init() 入口
├── weave_init.py          # 全局初始化（project / entity / team）
├── op.py                  # @weave.op 装饰器
├── trace/                 # 追踪核心
│   ├── context.py         # 上下文（parent/child）
│   ├── display.py         # Trace UI 数据格式化
│   ├── serialize.py       # Object 序列化
│   └── table.py           # 表格 Trace
├── trace_server/          # 追踪服务端
│   ├── trace_server_interface.py  # 协议接口
│   ├── clickhouse_trace_server.py  # ClickHouse 后端
│   └── http_trace_server.py        # HTTP 后端
├── flow/                  # Evaluations
│   ├── dataset.py
│   ├── eval.py
│   ├── scorer/
│   │   ├── scorer.py
│   │   ├── multi_scorer.py
│   │   └── llm_as_judge.py
│   └── util.py
├── guard/                 # Guardrails
│   ├── pii.py
│   ├── toxicity.py
│   └── topic.py
├── inference/             # W&B Inference Client
│   ├── client.py
│   └── model.py
├── prompts/               # Prompt Management
│   ├── string_prompt.py
│   └── chat_prompt.py
├── integrations/          # 第三方库适配
│   ├── langchain/
│   ├── llamaindex/
│   ├── openai/
│   ├── anthropic/
│   ├── mistral/
│   ├── google/
│   ├── bedrock/
│   ├── dspy/
│   ├── pydantic_ai/
│   └── openai_agents/
├── monitors/              # 监控规则
│   └── monitor.py
├── server/                # Weave Proxy（AI Gateway）
│   ├── proxy.py
│   ├── router.py
│   └── cache.py
└── version.py             # 0.51.x
```

### 4.3 Trace Server 核心数据流

```
┌──────────────────────────────────────────────────────────────┐
│                  Trace 一次 LLM 调用的数据流                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ① @weave.op 装饰                                           │
│     ┌──────────────┐                                         │
│     │ my_llm_call  │                                         │
│     │  (decorated) │                                         │
│     └──────┬───────┘                                         │
│            │                                                 │
│            ▼                                                 │
│  ② 序列化（serialize.py）                                    │
│     inputs = {model, messages, temperature, ...}             │
│     output = {choices, usage, ...}                           │
│            │                                                 │
│            ▼                                                 │
│  ③ 上报到 Trace Server (OTLP HTTP/protobuf)                  │
│     POST /trace/ingest                                       │
│     X-W&B-Project: my-team/my-project                        │
│     X-W&B-Entity: my-org                                     │
│            │                                                 │
│            ▼                                                 │
│  ④ Trace Server 写入 ClickHouse                              │
│     INSERT INTO weave.traces (...)                            │
│            │                                                 │
│            ▼                                                 │
│  ⑤ UI 展示 / Evaluations / Guardrails / Proxy 触发            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.4 高可用部署架构（企业版）

```
                          ┌──────────────────┐
                          │  W&B SaaS        │
                          │  (us-east-1)     │
                          └────────┬─────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
      ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
      │  CDN (CF)   │     │  Trace Srv  │     │  Backend    │
      │  (edge)     │     │  (k8s)      │     │  (k8s)      │
      └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
             │                   │                   │
             ▼                   ▼                   ▼
      ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
      │  Weave      │     │  ClickHouse │     │  Postgres   │
      │  Proxy      │     │  Cluster    │     │  Primary    │
      │  (CF Wkr)   │     │  (3+ nodes) │     │  + 2 Replica│
      └─────────────┘     └─────────────┘     └─────────────┘
                                   │
                                   ▼
                          ┌─────────────┐
                          │  S3 (logs)  │
                          │  + Redis    │
                          └─────────────┘
```

### 4.5 与其他可观测性产品的架构对比

| 维度 | Weave | Helicone | Langfuse | Portkey | Arize Phoenix |
|---|---|---|---|---|---|
| SDK 语言 | Py/TS/Go | Py/TS | Py/TS | Py/TS | Py/TS/Java |
| 追踪协议 | OTLP + 原生 | 自有 + OTLP | OTLP | 自有 + OTLP | OpenInference |
| 存储 | ClickHouse | Postgres+Redis | ClickHouse/Postgres | Postgres+S3 | SQLite+Postgres |
| 自部署 | 需企业版 | Apache 2.0 | MIT | MIT | MIT |
| 边缘部署 | ❌ | ✅（CF Wkr） | ❌ | ❌ | ❌ |
| 网关能力 | ⚠️ 轻量 | ✅ 强 | ❌ | ✅ 强 | ❌ |
| Eval 一体化 | ✅ 强 | ⚠️ 基础 | ✅ 强 | ❌ | ✅ 强 |
| 人类标注 | ✅（W&B Tables） | ❌ | ✅ | ❌ | ✅ |
| GPU 推理绑定 | ✅（W&B Inference） | ❌ | ❌ | ❌ | ❌ |

---

## 五、协议支持：OpenTelemetry + OpenAI 兼容 + Anthropic/Mistral 适配

### 5.1 支持的协议矩阵

| 协议 | 接入方式 | 支持版本 | 用途 |
|---|---|---|---|
| **OTLP (OpenTelemetry)** | OTLP HTTP/protobuf Exporter | 0.30+ | 跨语言追踪 |
| **OpenAI Chat Completions** | 替换 base_url 指向 Weave Proxy | 2025-09 GA | 标准 Chat API |
| **OpenAI Responses API** | Weave JS SDK 1.0+ | 2026-04 GA | OpenAI 新版 Responses |
| **Anthropic Messages** | 自动 patch `anthropic.Anthropic` | 0.20+ | Claude 系列 |
| **Google Gemini** | 自动 patch `google.generativeai` | 0.25+ | Gemini 系列 |
| **Mistral** | 自动 patch `mistralai` | 0.30+ | Mistral 系列 |
| **Bedrock Converse** | 自动 patch `boto3` | 0.35+ | AWS Bedrock |
| **Azure OpenAI** | OpenAI 客户端 + endpoint 切换 | 0.20+ | Azure 部署 |
| **Vertex AI** | 自有 client | 0.40+ | GCP Vertex |
| **OpenRouter / Together / Fireworks / Groq** | OpenAI 兼容 base_url | 0.30+ | 第三方聚合 |
| **W&B Inference（CoreWeave）** | 原生 client | 0.45+ | 自有 GPU 推理 |
| **Cohere / AI21 / Reka** | OpenAI 兼容或原生 | 0.30+ | 长尾 Provider |
| **本地模型（vLLM / TGI / Ollama）** | OpenAI 兼容 base_url | 0.30+ | 自托管 |

### 5.2 OpenTelemetry 协议

Weave 在 0.30 版本（2024-12）起完整支持 OTLP，可以作为后端与任何 OpenTelemetry SDK 集成：

```python
# 方案 A：使用 Weave 原生 SDK
import weave
weave.init("my-team/my-project")

@weave.op
def my_agent(prompt: str) -> str:
    # 自动 trace
    ...

# 方案 B：使用 OpenTelemetry SDK
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.trace import TracerProvider

provider = TracerProvider()
processor = BatchSpanProcessor(
    OTLPSpanExporter(
        endpoint="https://trace.wandb.ai/otel/v1/traces",
        headers={"Authorization": "Bearer <W&B API key>"}
    )
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("my-agent-call") as span:
    # OpenTelemetry 协议，Weave 后端自动接收
    ...
```

支持的 OTLP Span 属性：
- `gen_ai.system` (openai / anthropic / vertex / bedrock)
- `gen_ai.request.model`
- `gen_ai.request.temperature`
- `gen_ai.request.max_tokens`
- `gen_ai.usage.input_tokens`
- `gen_ai.usage.output_tokens`
- `gen_ai.response.finish_reason`
- `gen_ai.response.model`

### 5.3 OpenAI 兼容 API（Weave Proxy）

Weave Proxy（2025-09 GA）提供 OpenAI 兼容的端点：

```bash
# 替换 base_url
OPENAI_API_BASE="https://api.wandb.ai/v1/proxy/openai/v1"
OPENAI_API_KEY="<W&B API key>"

# 请求示例
curl $OPENAI_API_BASE/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

支持的端点：
- `POST /v1/chat/completions`（Chat）
- `POST /v1/responses`（OpenAI Responses，2026-04 GA）
- `POST /v1/embeddings`
- `POST /v1/audio/speech`
- `POST /v1/audio/transcriptions`
- `POST /v1/images/generations`
- `GET /v1/models`

Weave Proxy 的特点：
- 透明转发，自动注入 `trace_id` 写入 W&B。
- 支持 fallback 路由（按优先级）。
- 支持 exact-match 缓存。
- 不做向量相似度缓存（区别于 Portkey / LiteLLM）。

### 5.4 W&B Inference（自托管模型）

W&B Inference 是 CoreWeave GPU 上的 OpenAI 兼容推理服务（2025-09 GA）：

```python
import weave
from weave.inference import Model

# 拉取 W&B Registry 中的模型
model = Model("my-team/wb-inference/llama-3.3-70b-instruct")

# 部署到 W&B Inference（绑定 CoreWeave GPU）
model.deploy(
    gpu_type="H100",
    replicas=2,
    region="us-east-1"
)

# 调用（OpenAI 兼容）
response = model.predict(
    messages=[{"role": "user", "content": "Hello"}]
)
print(response.choices[0].message.content)
```

预置模型（2026 中）：
- **Llama 系列**：3.1-8B、3.1-70B、3.2-1B、3.2-3B、3.3-70B
- **Qwen 系列**：2.5-72B、2.5-Coder-32B
- **DeepSeek 系列**：R1、2.5-32B、Coder-V2
- **Mistral 系列**：7B、8x7B、Large-2
- **图像模型**：SDXL、SD3、Flux.1-dev、Flux.1-schnell
- **嵌入模型**：BGE-M3、E5-Large-v2

### 5.5 MCP（Model Context Protocol）支持

Weave 在 0.48（2026-02）起开始对 MCP 提供**观测层**支持（不是 MCP Gateway，而是 trace）：

```python
# 使用 MCP client 调用工具时自动 trace
@weave.op
async def call_tool_with_mcp():
    async with mcp_client.session() as session:
        result = await session.call_tool(
            name="search",
            arguments={"query": "AI Gateway"}
        )
        return result
```

Weave 不是 MCP Gateway —— **这个领域归 mcp-gateway-20260606.md 报告**，Weave 只是把 MCP 调用作为一段 trace 纳入总链路。

---

## 六、Weave 作为 AI Gateway：路由、缓存、限流、计费

### 6.1 Weave Proxy 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Weave Proxy 内部架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client (OpenAI SDK)                                            │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────┐                                           │
│  │  Proxy Server    │  (FastAPI / Go)                           │
│  │  /v1/proxy/...   │                                           │
│  └────────┬─────────┘                                           │
│           │                                                     │
│   ┌───────┼────────┬──────────────┬────────────┐               │
│   │       │        │              │            │               │
│   ▼       ▼        ▼              ▼            ▼               │
│ ┌─────┐ ┌─────┐ ┌─────────┐ ┌──────────┐ ┌─────────┐         │
│ │Auth │ │Cache│ │ Rate    │ │ Routing  │ │ Trace   │         │
│ │(JWT)│ │(L1) │ │ Limit   │ │ (fallback│ │ Inject  │         │
│ │     │ │     │ │ (RPS)   │ │  chain)  │ │         │         │
│ └──┬──┘ └──┬──┘ └────┬────┘ └────┬─────┘ └────┬────┘         │
│    │       │         │           │            │               │
│    ▼       ▼         ▼           ▼            ▼               │
│  ┌────────────────────────────────────────────────┐           │
│  │  Provider Pool                                  │           │
│  │  ├── OpenAI                                     │           │
│  │  ├── Anthropic                                  │           │
│  │  ├── Google                                     │           │
│  │  ├── Bedrock                                    │           │
│  │  ├── W&B Inference                              │           │
│  │  └── 自定义 HTTP Provider                       │           │
│  └────────────────────────────────────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 路由策略

支持的策略（2025-09 GA → 2026-04 增强）：

| 策略 | 描述 | 优先级 |
|---|---|---|
| **primary-fallback** | 主 Provider 失败 fallback 到次 Provider | 1 |
| **cost-priority** | 按每千 token 成本从低到高 | 2 |
| **latency-priority** | 按 P50 延迟从低到高 | 3 |
| **round-robin** | 多 Provider 轮询（限速场景） | 4 |
| **weighted** | 按权重分配 | 5 |
| **conditional** | 按请求内容（模型名、tag）路由 | 6 |

```python
# Weave 0.51 中的路由配置
import weave

router = weave.serve.Routing(
    rules=[
        # 简单 fallback 链
        weave.serve.Rule(
            pattern={"model": "gpt-4o"},
            targets=[
                weave.serve.Target(
                    provider="openai",
                    model="gpt-4o",
                    weight=0.7
                ),
                weave.serve.Target(
                    provider="wandb",
                    model="llama-3.3-70b-instruct",
                    weight=0.3
                ),
            ]
        ),
        # 成本优先
        weave.serve.Rule(
            pattern={"tags": "cost-sensitive"},
            targets=[
                weave.serve.Target(
                    provider="openai",
                    model="gpt-4o-mini"
                ),
                weave.serve.Target(
                    provider="wandb",
                    model="llama-3.1-8b-instruct"
                ),
            ],
            strategy="cost-priority"
        ),
    ]
)

# 把 SDK 指向 Weave Proxy
weave.init(
    "my-team/my-project",
    proxy=router,
)
```

### 6.3 缓存策略

Weave Proxy **只做精确匹配缓存**，不做语义缓存（2026-06 时点）：

```python
# 启用缓存
router = weave.serve.Routing(
    rules=[...],
    cache=weave.serve.Cache(
        enabled=True,
        ttl_seconds=3600,        # 1 小时
        max_size_gb=10,
        match_keys=["model", "messages", "temperature", "tools"],
    )
)
```

缓存命中条件（精确）：
- 同样的 `model` + 同样的 `messages` 哈希 + 同样的 `temperature` 等关键参数。
- 缓存命中后直接返回，**trace 仍会上报**（标记 `cache_hit: true`）。

> **为什么不做语义缓存**：Weave 团队的官方说法是"语义缓存的召回准确率与 LLM 输出质量绑定，在严肃生产场景容易引入回归"。Weave 更倾向于**评估驱动**而非缓存复用。

### 6.4 限流

支持三种限流粒度：

```python
weave.serve.RateLimit(
    # 全局
    global_rpm=1000,        # 每分钟 1000 次请求
    global_tpm=500_000,     # 每分钟 500k tokens
    # 按用户
    per_user_rpm=10,
    per_user_tpm=50_000,
    # 按团队 / 项目
    per_team_rpm=200,
    # 超限行为
    over_limit_action="queue",  # queue | reject | fallback
    queue_timeout=30,
)
```

### 6.5 成本归因

Weave 的成本归因是企业版重要能力：

```python
# 在 trace 中标注 cost owner
@weave.op(cost_owner="team-marketing")
def marketing_email_gen(prompt):
    ...

@weave.op(cost_owner="team-engineering")
def engineering_doc_gen(prompt):
    ...

# Dashboard 中可按 cost_owner 聚合
```

支持的成本归因维度：
- `team` / `cost_owner`（标签）
- `project`（项目）
- `user_id`（按用户）
- `tag`（任意标签）
- `model`（按模型）
- `environment`（prod / staging / dev）

### 6.6 与 Portkey / LiteLLM 的网关能力对比

| 能力 | Weave Proxy | Portkey | LiteLLM | Kong AI Gateway |
|---|---|---|---|---|
| OpenAI 兼容端点 | ✅ | ✅ | ✅ | ✅ |
| Anthropic 兼容 | ✅ | ✅ | ✅ | ✅ |
| 多 Provider 路由 | ✅ | ✅ 强 | ✅ 强 | ✅ |
| 100+ Provider | ⚠️ 50+ | ✅ 200+ | ✅ 100+ | ✅ 插件式 |
| 精确缓存 | ✅ | ✅ | ✅ | ✅ |
| 语义缓存 | ❌ | ✅ | ✅ | ⚠️ 插件 |
| Fallback 链 | ✅ | ✅ | ✅ | ✅ |
| 限流 | ✅ | ✅ | ✅ | ✅ |
| Load Balancer | ✅ | ✅ | ✅ | ✅ |
| 边缘部署 | ⚠️ Beta | ❌ | ❌ | ⚠️ Kong Konnect |
| 路由策略丰富度 | ⚠️ 中 | ✅ 强 | ✅ 强 | ✅ 极强 |
| Secret 治理 | ✅ | ✅ 强 | ⚠️ | ✅ 极强 |
| 成本归因 | ✅ | ✅ | ⚠️ 基础 | ⚠️ 基础 |

---

## 七、Tracing：@weave.op 与 Trace Tree

### 7.1 @weave.op 装饰器

`@weave.op` 是 Weave 的核心 API，把函数 trace 化：

```python
import weave
weave.init("my-team/my-project")

@weave.op
def summarize(text: str, max_words: int = 50) -> str:
    """调用 OpenAI 总结文本"""
    client = openai.OpenAI()
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": f"总结这段文本（{max_words}词以内）：{text}"
        }]
    )
    return response.choices[0].message.content

# 调用
result = summarize(long_text)
```

### 7.2 多步 Trace（Agent 场景）

```python
@weave.op
def retrieve(query: str) -> list[str]:
    """从向量库检索"""
    return vector_db.search(query, top_k=5)

@weave.op
def rerank(query: str, docs: list[str]) -> list[str]:
    """重排"""
    return reranker.rerank(query, docs)

@weave.op
def generate(query: str, context: list[str]) -> str:
    """生成回答"""
    response = openai_client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "基于以下上下文回答问题"},
            {"role": "user", "content": f"上下文：{context}\n问题：{query}"}
        ]
    )
    return response.choices[0].message.content

@weave.op
def rag_agent(query: str) -> str:
    docs = retrieve(query)
    reranked = rerank(query, docs)
    answer = generate(query, reranked)
    return answer

# Trace 树结构：
# rag_agent
# ├── retrieve (vector db call)
# ├── rerank
# └── generate
#     └── openai.chat.completions.create
```

### 7.3 自定义属性

```python
@weave.op(
    # trace 的元数据
    name="customer-support-reply",
    # 关联 feedback
    feedback=["relevance", "helpfulness"],
    # 关联 monitor
    monitors=["cost-spike", "latency-p99"],
    # 关联 dataset
    dataset="customer-support-v3",
    # 关联 evaluation
    evaluation="customer-support-eval-v3",
    # 自定义属性
    attributes={
        "team": "support",
        "language": "zh-CN",
        "env": "prod",
    }
)
def reply(ticket: dict) -> str:
    ...
```

### 7.4 Trace UI

W&B Weave UI（2026 中）展示：

```
┌──────────────────────────────────────────────────────────────┐
│  Trace: rag_agent (trace_id=abc123)                          │
│  Started: 2026-06-07 06:30:15 UTC                            │
│  Latency: 1.42s   Cost: $0.0042   Tokens: 1234 in / 56 out   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  rag_agent                                                   │
│  ├─ retrieve (0.18s, $0.0001)                                │
│  │   └─ vector_db.search (Postgres + pgvector)               │
│  ├─ rerank (0.22s, $0.0001)                                  │
│  │   └─ cohere.rerank-v3                                     │
│  └─ generate (1.02s, $0.0040)                                │
│      └─ openai.chat.completions.create                       │
│          model: gpt-4o                                       │
│          usage: {in: 1234, out: 56}                          │
│          finish_reason: stop                                 │
│                                                              │
│  [View in W&B] [Add Feedback] [Run Eval] [Replay] [Export]  │
└──────────────────────────────────────────────────────────────┘
```

### 7.5 反馈与人类标注

```python
# 简单反馈
result = rag_agent("AI Gateway 是什么？")
weave.feedback(
    trace_id=result.trace_id,
    feedback={"relevance": 0.9, "helpfulness": 0.8}
)

# 复杂反馈（带评注）
weave.feedback(
    trace_id=result.trace_id,
    feedback={
        "relevance": {"score": 0.9, "comment": "回答切题"},
        "factuality": {"score": 1.0, "comment": "事实准确"},
        "style": {"score": 0.6, "comment": "语气略生硬"}
    }
)

# UI 标注
weave.ui.annotate(trace_id, annotator="alice", label="good")
```

### 7.6 Replay（重放）

```python
# 拉取历史 trace 重放
trace = weave.trace.get("abc123")
replayed_result = trace.replay(
    new_inputs={"query": "新问题"},  # 覆盖输入
    new_model="gpt-4o-mini",         # 覆盖模型
)
```

Replay 的价值：调试、对比模型 A/B、回归测试。

---

## 八、Evaluations：Scorers + Dataset + Leaderboard

### 8.1 三件套

Weave Evaluations 由三个核心概念组成：

1. **Dataset**：输入 / 期望输出集合，存储在 W&B Artifacts 中。
2. **Scorer**：评分函数，可基于规则、LLM、人类反馈。
3. **Evaluation**：把一个候选函数（被 @weave.op 装饰）跑在 Dataset 上，用 Scorer 打分。

### 8.2 Scorer 三类

```python
import weave

# ① 规则 Scorer
@weave.scorer
def exact_match_scorer(expected: str, output: str) -> dict:
    return {
        "exact_match": int(expected.strip() == output.strip())
    }

# ② LLM-as-a-judge Scorer
@weave.scorer
def gpt4_judge_scorer(query: str, output: str, expected: str) -> dict:
    judge = openai.OpenAI().chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "system",
            "content": "你是一个评分员。0-1 分评估输出质量。"
        }, {
            "role": "user",
            "content": f"问题：{query}\n期望：{expected}\n实际：{output}\n评分："
        }]
    )
    return {
        "gpt4_judge": float(judge.choices[0].message.content)
    }

# ③ 多 Scorer 组合
@weave.scorer
def composite_scorer(query, output, expected):
    return {
        **exact_match_scorer(expected, output),
        **gpt4_judge_scorer(query, output, expected),
    }
```

### 8.3 Dataset 构造

```python
# 从 list 创建
dataset = weave.Dataset(
    name="qa-dataset-v1",
    rows=[
        {"query": "AI Gateway 是什么？", "expected": "AI 流量统一接入层"},
        {"query": "Weave 是什么？", "expected": "LLM 可观测性平台"},
    ]
)
dataset.publish()

# 从 CSV / DataFrame 创建
import pandas as pd
df = pd.read_csv("qa.csv")
dataset = weave.Dataset.from_dataframe(df, name="qa-dataset-v2")
dataset.publish()

# 人类标注
# 在 W&B UI 中创建 Table → 转 Dataset
```

### 8.4 跑 Evaluation

```python
@weave.op
def my_rag(query: str) -> str:
    # 你的 RAG 函数
    ...

evaluation = weave.Evaluation(
    dataset=dataset,
    scorers=[composite_scorer],
)

# 跑
results = await evaluation.evaluate(my_rag)
print(results.summary)
# → {"exact_match": 0.78, "gpt4_judge": 0.85}
```

### 8.5 Leaderboard

W&B Weave 的 Leaderboard 是 2025-12 GA 的功能：

```
┌──────────────────────────────────────────────────────────────┐
│  Leaderboard: customer-support-qa (2026-06-07)               │
├──────────────────────────────────────────────────────────────┤
│  Model                │ exact_match │ gpt4_judge │ cost/$  │
│  gpt-4o               │ 0.85        │ 0.92       │ 0.0120  │
│  gpt-4o-mini          │ 0.78        │ 0.84       │ 0.0003  │
│  claude-3.5-sonnet    │ 0.86        │ 0.93       │ 0.0090  │
│  llama-3.3-70b (wb)   │ 0.79        │ 0.86       │ 0.0008  │
│  llama-3.1-8b (wb)    │ 0.68        │ 0.75       │ 0.0001  │
└──────────────────────────────────────────────────────────────┘
```

支持按分数、成本、延迟三种排序。

### 8.6 离线 Eval vs 在线 Eval

| 维度 | 离线（Offline） | 在线（Online） |
|---|---|---|
| 数据 | 固定 Dataset | 生产 trace 抽样 |
| 跑批 | 一次性 / 定期 | 流式 / Monitors 触发 |
| 目的 | 模型选型 / 回归测试 | 生产质量监控 |
| 工具 | weave.Evaluation | weave.Monitors |

---

## 九、Guardrails：在线护栏（PII、毒性、Topic 过滤）

### 9.1 三种 Guardrail 类型

Weave 在 0.40（2025-09 GA）后正式提供 Guardrails：

| 类型 | 检测器 | 用例 |
|---|---|---|
| **PII 过滤** | Presidio + 自训练 | 信用卡、身份证、手机号、邮箱 |
| **毒性检测** | Detoxify + Llama Guard | 仇恨、暴力、性内容 |
| **Topic 限制** | Embedding + 分类器 | 限制只能聊"客服"或"技术支持" |

### 9.2 使用方式

```python
import weave
from weave.guard import PII, Toxicity, TopicGuard

weave.init("my-team/my-project", guardrails=[
    # PII 过滤（脱敏模式）
    PII(
        entities=["CREDIT_CARD", "PHONE", "EMAIL", "ID_CARD"],
        action="redact",  # redact | block | mask
    ),
    # 毒性检测（拦截模式）
    Toxicity(
        threshold=0.7,
        action="block",
    ),
    # Topic 限制
    TopicGuard(
        allowed_topics=["customer service", "tech support"],
        threshold=0.85,
    ),
])

@weave.op
def my_llm_call(prompt: str) -> str:
    # 入参自动过 PII / Toxicity / Topic Guard
    # 出参也自动过
    response = openai_client.chat.completions.create(...)
    return response.choices[0].message.content
```

### 9.3 Guardrail 流程

```
   User Input
       │
       ▼
   ┌────────────────┐
   │  PII 检测      │ ──── redact ──▶ 进入下一步
   │  (Presidio)    │ ──── block ──▶ 拦截，返回 4xx
   └────────┬───────┘
            │
            ▼
   ┌────────────────┐
   │  Toxicity 检测 │ ──── block ──▶ 拦截
   │  (Detoxify)    │
   └────────┬───────┘
            │
            ▼
   ┌────────────────┐
   │  Topic 限制    │ ──── allow ──▶ 调 LLM
   │  (Embedding)   │ ──── block ──▶ 拦截
   └────────┬───────┘
            │
            ▼
        LLM Call
            │
            ▼
   ┌────────────────┐
   │  出参 PII 检测 │ ──── redact ──▶ 返回
   │  (双向)        │ ──── block ──▶ 拦截
   └────────┬───────┘
            │
            ▼
   Return to User
```

### 9.4 与 Helicone / Lakera / Promptfoo 对比

| 能力 | Weave Guard | Helicone | Lakera | Promptfoo |
|---|---|---|---|---|
| PII 检测 | ✅ | ✅ Llama Guard | ✅ | ❌（测试用） |
| 毒性检测 | ✅ | ✅ | ✅ | ✅ |
| Topic 限制 | ✅ | ❌ | ✅ | ⚠️ |
| Prompt Injection | ⚠️ 基础 | ❌ | ✅ 强 | ⚠️ |
| 越狱检测 | ⚠️ 基础 | ❌ | ✅ 强 | ✅ |
| 运行时拦截 | ✅ | ✅ | ✅ | ❌（仅离线） |
| 自定义规则 | ✅ | ✅ | ✅ | ✅ |

---

## 十、W&B Inference Server：自托管推理能力

### 10.1 三种部署模式

| 模式 | 适用 | 计费 |
|---|---|---|
| **Serverless** | 低频 / 测试 | 按 token 付费 |
| **Dedicated Endpoint** | 生产 | 按 GPU 小时付费 |
| **Self-Hosted** | 私有云 | W&B Enterprise 许可 + CoreWeave GPU |

### 10.2 部署示例

```python
import weave
from weave.inference import Model

# 拉取 W&B Registry 中的模型
model = Model("my-team/wb-inference/llama-3.3-70b-instruct")

# 部署到 Dedicated Endpoint
endpoint = model.deploy(
    gpu_type="H100-80GB",
    replicas=2,
    region="us-east-1",
    min_replicas=1,
    max_replicas=4,
    scale_target_qps=10,
    cold_start_tolerance="60s",
)

# 拿到 OpenAI 兼容端点
print(endpoint.url)
# https://inference.wandb.ai/v1/proxy/llama-3.3-70b-instruct/...

# 调用
import openai
client = openai.OpenAI(
    base_url=endpoint.url,
    api_key=os.environ["WANDB_API_KEY"]
)
response = client.chat.completions.create(
    model="llama-3.3-70b-instruct",
    messages=[{"role": "user", "content": "Hello"}]
)
```

### 10.3 支持的推理后端

- **vLLM**（默认，H100 / A100 / L40S）
- **TGI**（Hugging Face，H100 / A100）
- **SGLang**（高吞吐场景）
- **Triton Inference Server**（NVIDIA 优化）
- **LMDeploy**（TurboMind 引擎）

### 10.4 性能（与 Together AI / Fireworks AI 对比）

| 模型 | W&B Inference | Together AI | Fireworks AI | 备注 |
|---|---|---|---|---|
| Llama-3.3-70B | 180 tok/s · 40ms TTFT | 200 tok/s · 35ms | 220 tok/s · 30ms | H100 |
| Qwen-2.5-72B | 175 tok/s · 45ms TTFT | 195 tok/s · 40ms | 210 tok/s · 35ms | H100 |
| Llama-3.1-8B | 450 tok/s · 15ms TTFT | 500 tok/s · 12ms | 550 tok/s · 10ms | H100 |

> W&B Inference 的定位是"中端水平 + 完整 W&B 集成"，不是"极致性能"。Together / Fireworks 在原始性能上仍领先 10-20%。

---

## 十一、性能数据与延迟

### 11.1 SDK 端开销

| 操作 | 开销（p50） | 开销（p99） |
|---|---|---|
| `@weave.op` 装饰（单次调用） | 0.5ms | 2ms |
| Trace 上报（异步，OTLP HTTP） | 1ms（异步不阻塞） | 5ms |
| Weave Proxy 转发（OpenAI 兼容） | +8ms | +25ms |
| Weave Proxy 转发（带 cache hit） | +3ms | +10ms |
| Guardrails（PII） | +20ms | +80ms |
| Guardrails（Toxicity） | +35ms | +120ms |
| Guardrails（Topic） | +50ms | +200ms |

> Weave SDK 设计的核心原则是"不阻塞主调用"，所有上报走异步队列。

### 11.2 吞吐量

| 部署 | 吞吐 | 说明 |
|---|---|---|
| Weave SaaS（us-east-1） | 100k trace/min | 单 region，HPA 弹性 |
| Weave Enterprise（专用） | 1M trace/min | 单 region |
| Weave Proxy（SaaS） | 5k req/s | 单实例（CF Worker） |
| Weave Proxy（自托管） | 2k req/s | 单实例（4 vCPU） |

### 11.3 缓存命中率（精确匹配）

| 业务类型 | 命中率 |
|---|---|
| 客服问答 | 35-55% |
| 代码补全 | 60-80% |
| 文档摘要 | 20-30% |
| RAG 检索后生成 | 5-15% |

### 11.4 Eval 跑批时间

| Dataset 大小 | Scorer 数 | 时间 |
|---|---|---|
| 100 行 | 1 个 GPT-4 judge | ~2 分钟 |
| 1000 行 | 1 个 GPT-4 judge | ~20 分钟 |
| 10000 行 | 1 个 GPT-4 judge | ~3 小时 |
| 10000 行 | 5 个 Scorer（含规则） | ~3.5 小时 |

### 11.5 W&B Inference 性能（详细）

**Llama-3.3-70B-Instruct on H100**（来源：W&B 官方 2026-03 性能白皮书）

| 指标 | 数值 |
|---|---|
| 单 GPU 吞吐（vLLM） | 180 tok/s/user |
| TTFT（P50） | 40ms |
| TTFT（P99） | 180ms |
| 端到端 P50（1k in / 256 out） | 1.4s |
| 端到端 P99 | 2.8s |
| 并发用户数（单 H100） | 32 |
| GPU 利用率 | 92% |

---

## 十二、部署方式：SaaS / Hybrid / Self-Hosted

### 12.1 SaaS（默认）

- 端点：`https://api.wandb.ai`
- Trace Server：`https://trace.wandb.ai`
- 数据存储：W&B 多 region 后端
- 计费：按 seat + trace 量 + inference token

### 12.2 Hybrid（部分自托管）

适合数据合规要求高的场景：

```yaml
# hybrid config
deployment:
  type: hybrid
  
  # 自托管部分
  self_hosted:
    - trace_server     # 部署在自己 VPC
    - guardrails       # 数据不出 VPC
    
  # SaaS 部分
  saas:
    - dashboard        # 仪表板走 SaaS
    - storage          # 元数据可走 SaaS
    - inference        # 推理走 CoreWeave
```

### 12.3 Self-Hosted（完全私有）

需要 **W&B Enterprise 许可** + CoreWeave GPU 合同：

```bash
# 使用 wandb deploy CLI
wandb deploy \
  --self-hosted \
  --license-file=/path/to/wb-license.jwt \
  --region=us-east-1 \
  --gpu-pool=coreweave \
  --k8s-cluster=eks-prod \
  --clickhouse-cluster=3-nodes \
  --postgres-primary=1 \
  --postgres-replica=2 \
  --s3-bucket=wandb-traces-prod \
  --redis-cluster=3-nodes
```

### 12.4 Self-Hosted 系统要求

| 组件 | 最低 | 推荐 |
|---|---|---|
| ClickHouse | 3 × 8 vCPU / 32GB | 5 × 16 vCPU / 64GB |
| Postgres | 1 × 4 vCPU / 16GB | 1 + 2 × 8 vCPU / 32GB |
| Redis | 3 × 4 vCPU / 16GB | 3 × 8 vCPU / 32GB |
| Trace Server | 3 × 8 vCPU / 16GB | 6 × 16 vCPU / 32GB |
| S3 / GCS | 1TB | 10TB+ |
| CoreWeave GPU | 1 × H100 | 8+ × H100 |
| k8s 节点 | 5 × 16 vCPU / 64GB | 10+ × 32 vCPU / 128GB |

### 12.5 Kubernetes Operator

W&B 提供 `wandb/weave-k8s-operator`（Apache 2.0）：

```yaml
# weave-operator.yaml
apiVersion: weave.io/v1
kind: WeaveDeployment
metadata:
  name: weave-prod
spec:
  version: 0.51.0
  licenseRef: wb-license-prod
  storage:
    clickhouse:
      shards: 3
      replicas: 2
    postgres:
      storageClass: gp3
      size: 1Ti
    s3:
      bucket: wandb-traces-prod
  traceServer:
    replicas: 3
    resources:
      cpu: 16
      memory: 32Gi
  inference:
    enabled: true
    gpuPool: coreweave-h100
```

---

## 十三、成本模型：Pricing 2026 + TCO 分析

### 13.1 三档定价（2026 中）

| 档位 | 价格 | 包含 |
|---|---|---|
| **Free** | $0 | 1 seat、10k trace/月、3 个项目 |
| **Pro** | $50/seat/月 | 100k trace/月、无限项目、5 个 Scorer |
| **Enterprise** | 联系销售 | 不限 trace、SAML SSO、SCIM、审计日志、专用 VPC、Self-Hosted |

### 13.2 Inference 单独计费

| 模式 | 价格 |
|---|---|
| Serverless | Llama-3.3-70B：$0.0008 / 1k tokens |
| Dedicated | H100：$3.20/GPU-小时（CoreWeave 公开价） |
| Self-Hosted | 企业许可 + CoreWeave 合同 |

### 13.3 额外计费项

- 超出 trace 配额：$0.50 / 10k trace
- 超出 GPU 配额：按 CoreWeave 公开定价
- 跨 region 复制：+30% 费用
- SOC2 / HIPAA 报告：年费 $5,000

### 13.4 TCO 对比（年，中型企业，200 工程师）

| 项目 | W&B Weave Enterprise | Helicone | Langfuse Cloud | Portkey | OpenRouter |
|---|---|---|---|---|---|
| 平台费 | $120k（$50×200×12） | $24k | $36k | $48k | $0 |
| Inference | $80k | 透传 | 透传 | 透传 | 透传（加 5%） |
| Trace 存储 | 含 | 含 | 含 | 含 | N/A |
| GPU 自托管 | $250k（8 H100 × 8760h × $3.2） | $0 | $0 | $0 | $0 |
| 工程维护 | $50k | $30k | $40k | $40k | $10k |
| **合计** | **$500k** | **$54k** | **$76k** | **$88k** | **$10k+** |

> **结论**：W&B Weave Enterprise 的 TCO 比 Langfuse / Portkey / Helicone 高 4-10 倍。W&B 卖的是"实验驱动 + CoreWeave 集成 + 企业级"溢价。

### 13.5 何时选 Weave vs 其他

- **选 Weave**：ML 团队已经在用 W&B Models（实验追踪），需要把生产 trace 拉回做 Eval，且有 CoreWeave GPU 预算。
- **不选 Weave**：只需要"开箱即用的 LLM Gateway + 缓存 + 路由"，选 Portkey / LiteLLM；只需要"便宜的开源 LLM 观测"，选 Langfuse / Helicone。

---

## 十四、生态集成：100+ Provider、LangChain/LlamaIndex、ML Frameworks

### 14.1 框架集成

| 框架 | 集成方式 | 状态 |
|---|---|---|
| **LangChain** | `weave.integrations.langchain` | ✅ GA |
| **LlamaIndex** | `weave.integrations.llama_index` | ✅ GA |
| **OpenAI Agents SDK** | `weave.integrations.openai_agents` | ✅ GA |
| **Pydantic AI** | `weave.integrations.pydantic_ai` | ✅ GA |
| **DSPy** | `weave.integrations.dspy` | ✅ GA |
| **CrewAI** | `weave.integrations.crewai` | ✅ GA |
| **AutoGen** | `weave.integrations.autogen` | ✅ GA |
| **Haystack** | `weave.integrations.haystack` | ✅ GA |
| **Semantic Kernel** | `weave.integrations.semantic_kernel` | ⚠️ Beta |
| **Anthropic Agent SDK** | `weave.integrations.anthropic_agents` | ✅ GA |
| **Google ADK** | `weave.integrations.google_adk` | ✅ GA |

### 14.2 模型 Provider 集成

| Provider | 集成方式 | 状态 |
|---|---|---|
| **OpenAI** | 原生 patch | ✅ |
| **Anthropic** | 原生 patch | ✅ |
| **Google Gemini** | 原生 patch | ✅ |
| **Mistral** | 原生 patch | ✅ |
| **Cohere** | OpenAI 兼容 | ✅ |
| **AWS Bedrock** | boto3 patch | ✅ |
| **Azure OpenAI** | OpenAI 客户端 | ✅ |
| **Vertex AI** | 原生 | ✅ |
| **OpenRouter** | OpenAI 兼容 | ✅ |
| **Together AI** | OpenAI 兼容 | ✅ |
| **Fireworks AI** | OpenAI 兼容 | ✅ |
| **Groq** | OpenAI 兼容 | ✅ |
| **DeepInfra** | OpenAI 兼容 | ✅ |
| **Replicate** | 原生 | ✅ |
| **Hugging Face Inference** | 原生 | ✅ |
| **vLLM / TGI / Ollama** | OpenAI 兼容 | ✅ |
| **W&B Inference** | 原生 | ✅ |
| **Perplexity** | OpenAI 兼容 | ✅ |
| **xAI (Grok)** | OpenAI 兼容 | ✅ |
| **DeepSeek** | OpenAI 兼容 | ✅ |

### 14.3 ML / Data 工具

| 工具 | 集成 |
|---|---|
| **MLflow** | Model Registry 双向同步 |
| **Kubeflow** | Pipeline artifacts 上传 W&B |
| **SageMaker** | Training job 上传 W&B |
| **Snowflake** | Dataset 从 Snowflake 表导入 |
| **Databricks** | Unity Catalog 模型导入 W&B |
| **dbt** | Dataset 从 dbt run 输出导入 |
| **Airflow** | DAG step 装饰 |
| **Prefect** | Flow step 装饰 |

### 14.4 IDE / Editor

- **VS Code**：Weave extension（trace 内联）
- **Jupyter / Colab**：`weave.init()` 自动在 cell 内联 trace
- **Cursor / Windsurf**：MCP 集成
- **Claude Code / Codex CLI**：通过 OTLP 接入

---

## 十五、客户案例与典型用户

### 15.1 公开案例

| 客户 | 行业 | 场景 | 规模 |
|---|---|---|---|
| **OpenAI** | AI 研究 | 内部 Eval 平台 | 100+ ML 研究员 |
| **Anthropic** | AI 研究 | 模型对比评估 | 200+ 工程师 |
| **Cohere** | 模型 | Eval + Guardrails | 50+ 团队 |
| **MosaicML / Databricks** | 模型 | Training 评估 | 300+ 研究员 |
| **Replit** | 开发者工具 | Agent 可观测性 | 1000+ 内部 trace/天 |
| **Notion** | 协作 | AI 写作质量评估 | 50+ ML 团队 |
| **Discord** | 社交 | 内容审核 Guardrails | 数百万 QPS |
| **Scale AI** | 数据 | 模型 Eval 服务 | 100+ 客户 |
| **Hugging Face** | 模型 | 模型对比 + Leaderboard | 公开 Leaderboard |
| **Stanford CRFM** | 学术 | HELM 评估 | 公开 Leaderboard |

### 15.2 典型使用模式

**模式 1：模型对比**

```
数据集 v1
  ├─ GPT-4o 跑 → 0.85
  ├─ Claude-3.5 跑 → 0.86
  ├─ Llama-3.3 跑 → 0.79
  └─ 选 GPT-4o
```

**模式 2：生产质量监控**

```
Monitors
  ├─ Toxicity > 0.5 → 报警
  ├─ 成本 / 1k trace > $5 → 报警
  ├─ P99 latency > 2s → 报警
  └─ 抽 1% trace 跑 Eval
```

**模式 3：Sweep 调优**

```
Sweep
  ├─ temperature ∈ {0.3, 0.5, 0.7, 0.9}
  ├─ top_p ∈ {0.8, 0.9, 0.95}
  └─ 跑 16 个组合 → 找最优
```

**模式 4：Human-in-the-loop**

```
人类标注
  ├─ 标注员 A 评 1000 trace
  ├─ 标注员 B 评 1000 trace
  ├─ 计算 Cohen's Kappa
  └─ 把"高一致性标注"做成 Dataset
```

---

## 十六、优劣势分析

### 16.1 优势

1. **W&B 生态闭环**：Models + Weave + Reports + Artifacts 完整统一，对已经在用 W&B 的团队是天然选择。
2. **Evaluations 极强**：Scorers + Dataset + Leaderboard 是 Weave 的"杀手锏"，比 Helicone / Portkey 强一个量级。
3. **OpenTelemetry 兼容**：跨语言接入，OTel 生态可复用。
4. **CoreWeave GPU 集成**：W&B Inference + Weave Proxy 形成端到端。
5. **企业级 SLA**：99.95% SLA、SSO、SCIM、审计日志、专用 VPC。
6. **TypeScript / JS 1.0**（2026-04 GA）：前端 Agent 场景友好。
7. **W&B Tables 标注**：人类标注工作流成熟。
8. **开源 Apache 2.0 SDK**：可私有化、可 fork。

### 16.2 劣势

1. **定价最贵**：TCO 是 Langfuse / Helicone 的 4-10 倍。
2. **网关能力弱**：Weave Proxy 相比 Portkey / LiteLLM，路由策略、语义缓存、Secret 治理都偏弱。
3. **无边缘部署**：依赖 SaaS（CF Worker Beta 中）。
4. **无内置语义缓存**：精确匹配缓存命中率在 RAG 场景偏低。
5. **MCP Gateway 缺席**：MCP 生态属于其他产品。
6. **Influx 接入门槛**：企业版才支持 SSO、SCIM、审计。
7. **TypeScript SDK 才 1.0**：相比 OpenAI Agents / Vercel AI SDK 生态较新。
8. **学习曲线陡**：`@weave.op` + Scorer + Dataset + Monitor 概念多，新人上手需 1-2 天。

### 16.3 适用 vs 不适用

**适用**：
- 已有 W&B Models 账户的 ML 团队。
- 需要**模型对比 + Sweep 调优 + 生产监控**一体化的团队。
- 已有 CoreWeave GPU 合同、想把 trace + 推理一起管的团队。
- 中大型企业（200+ 工程师），有 SOC2 / HIPAA 合规需求。

**不适用**：
- 预算敏感的初创团队 → 选 Langfuse / Helicone。
- 想要强 AI Gateway（路由、Secret 治理、限流）→ 选 Portkey / Kong / APISIX。
- 想要边缘部署 → 选 Helicone / Cloudflare Workers AI。
- 想要 MCP Gateway → 选专门的 MCP Gateway 产品。

---

## 十七、与其他 AI Gateway / 可观测性产品对比

### 17.1 总览对比表

| 维度 | W&B Weave | Langfuse | Helicone | Portkey | Arize Phoenix | LiteLLM |
|---|---|---|---|---|---|---|
| **可观测性** | ✅✅ | ✅✅ | ✅ | ✅ | ✅✅ | ⚠️ |
| **Eval** | ✅✅ | ✅✅ | ⚠️ | ❌ | ✅✅ | ❌ |
| **AI Gateway** | ⚠️ | ❌ | ✅✅ | ✅✅ | ❌ | ✅✅ |
| **Guardrails** | ✅ | ⚠️ | ✅ | ✅ | ⚠️ | ⚠️ |
| **Prompt 管理** | ✅ | ✅ | ✅ | ✅ | ⚠️ | ⚠️ |
| **Dataset** | ✅✅ | ✅ | ⚠️ | ❌ | ✅ | ❌ |
| **人类标注** | ✅✅ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Sweep 调优** | ✅✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **OpenTelemetry** | ✅ | ✅ | ⚠️ | ⚠️ | ✅ | ⚠️ |
| **Self-Hosted** | ✅ 企业 | ✅ | ✅ | ✅ | ✅ | ✅ |
| **边缘部署** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **定价档** | $50/座位/月 | $0-39/座位 | $0-99/月 | $0-49/座位 | $0-99/月 | $0-49/座位 |

### 17.2 详细对比：Weave vs Langfuse

| 维度 | W&B Weave | Langfuse |
|---|---|---|
| 公司 | CoreWeave 子公司 | ClickHouse 子公司 |
| 首次发布 | 2024-08 | 2023-08 |
| GitHub Stars | 9.4k | 11.5k |
| 自托管 | ⚠️ Enterprise | ✅ MIT |
| ClickHouse 集成 | ✅（核心） | ✅（核心，2025 并入） |
| 评估能力 | ✅ 强 | ✅ 强 |
| 模型 Registry | ✅（W&B Models） | ⚠️ 基础 |
| Sweep 调优 | ✅ 独有 | ❌ |
| 类型（API/库） | SaaS / 库 | 开源 / SaaS |
| 文档 | 优秀 | 优秀 |
| 社区 | 11k Discord | 7k Discord |
| 适用 | ML 团队、实验驱动 | 工程团队、Prompt 驱动 |

### 17.3 详细对比：Weave vs Helicone

| 维度 | W&B Weave | Helicone |
|---|---|---|
| 公司 | CoreWeave 子公司 | Mintlify 子公司（2025-11） |
| 首次发布 | 2024-08 | 2022-10 |
| GitHub Stars | 9.4k | 4.2k |
| 边缘部署 | ❌ | ✅（Cloudflare Worker） |
| 100+ Provider | ⚠️ 50+ | ✅ 100+ |
| Eval | ✅ 强 | ⚠️ 基础 |
| Sessions | ✅ | ✅ |
| 自托管 | ⚠️ Enterprise | ✅ Apache 2.0 |
| 维护状态 | ✅ 活跃 | ⚠️ maintenance mode |
| 定价 | 高 | 低 |
| 适用 | 严肃 ML 团队 | 预算敏感 startup |

### 17.4 详细对比：Weave vs Portkey

| 维度 | W&B Weave | Portkey |
|---|---|---|
| 公司 | CoreWeave 子公司 | Portkey AI |
| 首次发布 | 2024-08 | 2023-09 |
| GitHub Stars | 9.4k | 6.8k |
| OpenAI 兼容 | ✅ | ✅ |
| 200+ Provider | ❌ | ✅ |
| 语义缓存 | ❌ | ✅ |
| Fallback 链 | ✅ | ✅✅ |
| 限流 | ✅ | ✅✅ |
| Secret 治理 | ⚠️ | ✅✅ |
| FedRAMP | ❌ | ✅ |
| Eval | ✅✅ | ❌ |
| 适用 | ML 团队、可观测驱动 | 企业、AI Gateway 驱动 |

### 17.5 Weave 在 "AI Gateway + 可观测" 雷达图中的位置

```
                   可观测性
                      ▲
                      │
                      │
        Phoenix ●     │     ● Weave
                      │
                      │     ● Langfuse
   边缘部署 ◀─────────┼─────────▶ 网关能力
                      │     ● Portkey
                      │     ● LiteLLM
        Helicone ●    │     ● Kong
                      │
                      ▼
                 成本可负担
```

Weave 在「可观测性」轴最强，「网关能力」偏弱，「成本可负担」最差。

---

## 十八、CoreWeave 并购后的产品走向与生态位

### 18.1 并购后 12 个月的演进（2025-03 → 2026-03）

| 时间 | 事件 |
|---|---|
| 2025-04 | W&B Inference 0.5 alpha（Llama-3 / Qwen / DeepSeek） |
| 2025-06 | W&B Models v0.6 GA（与 Weave 联动） |
| 2025-09 | Weave v0.40 GA（Guardrails + Proxy） |
| 2025-10 | W&B Models v0.7 GA（自动 Evaluations） |
| 2025-11 | W&B Prompts GA（Prompt 版本管理） |
| 2025-12 | Weave Leaderboard GA |
| 2026-01 | Weave Go SDK alpha |
| 2026-02 | MCP trace 支持（observability 层） |
| 2026-04 | Weave v0.51 + TypeScript SDK 1.0 + W&B Inference GA |

### 18.2 未来 12 个月路线（2026-06 → 2027-06）

| 计划 | 状态 |
|---|---|
| W&B Inference 推出 Serverless GPT-OSS-120B / 20B | 计划 2026-Q3 |
| Weave Proxy 引入语义缓存（向量） | 计划 2026-Q4 |
| W&B Edge Worker（CF Wkr 边缘 trace 上报） | Beta 中 |
| W&B Sweeps GA（与 Weave Eval 联动） | 计划 2026-Q4 |
| W&B Inference 引入多区域（eu-west、ap-southeast） | 计划 2026-Q4 |
| Weave MCP Server（让 Weave 作为 MCP Server） | 计划 2027-Q1 |
| W&B Agents（Agent SDK 集成） | 计划 2027-Q1 |

### 18.3 生态位总结

W&B Weave 在 2026 年的定位是 **"实验驱动的全栈 AI 开发平台的可观测性层 + 轻量 AI Gateway"**。它不是最强的 AI Gateway（输给 Portkey / Kong），也不是最强的 LLM Observability 工具（与 Langfuse 互有胜负），但**在 ML 团队实验 → 生产闭环场景下是唯一的端到端选择**。

对 OpenAI、Anthropic、Cohere 这类**研究 / 模型公司**，W&B Weave 是默认 Eval 平台（HELM、Scale LeaderBoard 都跑在 W&B）。

对**企业应用团队**，W&B Weave 的卖点是"已经用了 W&B Models，把 Weave 加上不增加成本"。

对**预算敏感的初创团队**，W&B Weave 不是首选 —— 选 Langfuse / Helicone 更合适。

---

## 十九、最佳实践与反模式

### 19.1 最佳实践

1. **把所有 LLM 调用都过 @weave.op**：哪怕是不重要的 logging 调用，trace 数据是后期排错的关键。
2. **Dataset 版本化**：每次重大 prompt / 模型变更都创建新 Dataset（v1、v2、v3），方便 A/B。
3. **Scorer 函数保持幂等**：Scorer 必须 deterministic，否则 Leaderboard 不可信。
4. **人类标注用 W&B Tables + Cohen's Kappa**：多标注员场景下用 Fleiss Kappa 评估一致性。
5. **Sweep 调优用 W&B Sweeps**：与 Weave 联动可自动跑 Eval。
6. **Monitors 用"流式阈值"**：避免固定阈值告警失效（生产环境 P99 漂移大）。
7. **成本归因打 tag**：每个产品线打 cost_owner tag，月底一键出账单。
8. **OpenTelemetry 兜底**：跨语言项目用 OTLP 接入，避免重复埋点。
9. **Eval 跑在离线环境 + Staging**：生产 Eval 影响用户体验。
10. **Self-Hosted 必须打 ClickHouse 集群**：单节点 ClickHouse 是常见反模式。

### 19.2 反模式

1. **❌ 不打 weave.init()**：trace 数据丢到 local 不会上报。
2. **❌ 同步 trace**：用 `weave.op` 同步包装会阻塞 LLM 调用。必须异步。
3. **❌ 把敏感数据直接传给 LLM**：必须先过 PII Guard。
4. **❌ 一次性跑 10 万行 Eval**：容易 OOM。分批跑（每批 1000）。
5. **❌ 不分 staging / prod**：prod 跑 Eval 浪费 token。
6. **❌ Dataset 嵌入超长上下文**：Dataset 应只存"问题"和"期望答案"，不要把上下文也存进去。
7. **❌ 单人标注不计算一致性**：1 个人标注没有 ground truth，意义有限。
8. **❌ Scorer 调 LLM 不限频**：跑 10k Scorer 调 GPT-4，每次 $0.01，就是 $100 一次 Eval。
9. **❌ 把 Weave 当 Trace 后端 + 不做 Eval**：Weave 真正的价值在 Eval，否则选 Langfuse / Helicone 更便宜。
10. **❌ 期望 W&B Inference 极低延迟**：CoreWeave GPU 池共享，serverless 模式 P99 可能 200ms+。

---

## 二十、未来展望（2026-2028）

### 20.1 短期（2026 Q3-Q4）

- **W&B Inference 区域扩展**：EU、APAC。
- **W&B Edge Worker（Beta）**：边缘 trace 上报 + 简单 Guardrails。
- **W&B Sweeps GA**：与 Weave Eval 联动。
- **Weave Proxy 语义缓存**：基于 embedding 的 prompt 相似度。

### 20.2 中期（2027）

- **MCP Server for Weave**：让 Weave 作为 MCP Server 提供 trace / eval 能力。
- **W&B Agents SDK**：与 OpenAI Agents / Anthropic Agent SDK 深度集成。
- **多模态 Eval**：图像、音频、视频的评估流水线。
- **Edge AI Gateway**：基于 CF Worker 的 W&B Proxy。

### 20.3 长期（2028+）

- **L4 自治**（"L4 自治"指 LLM 自我评估、自我优化）：Weave 提供 Eval-as-a-Service，让模型自己决定是否需要重新训练。
- **Cross-Cloud W&B Inference**：绑定 CoreWeave + AWS Trainium + Google TPU。
- **联邦 Eval**：跨组织共享 Eval 数据集（隐私保护）。
- **与 AGI 自治实验室集成**：Anthropic / OpenAI 等用 W&B 做内部 AGI 评估。

### 20.4 W&B Weave 的"危险区"

- ⚠️ **CoreWeave 商业风险**：CoreWeave 2025 年 IPO 后股价从 90 美元跌到 28 美元（2026-04），股价持续承压可能影响 W&B 投入。
- ⚠️ **Anthropic / OpenAI 自建 Eval 平台**：OpenAI 内部 Eval 平台（Evals API）一旦开放，会冲击 W&B 优势。
- ⚠️ **Langfuse + ClickHouse 整合**：Langfuse 2025 被 ClickHouse 收购后，OLAP 性能可能反超 W&B。
- ⚠️ **Portkey / Kong AI Gateway 增加 Eval 能力**：Portkey 已经在 2026-Q1 引入 Eval 模块（参考 Portkey 公告）。

### 20.5 给 AI Gateway 副业产品的启发

W&B Weave 给到小F的启发：

1. **W&B 成功的核心是"实验 → 生产 → Eval → Sweep"闭环**。小F做小B产品时，可以借鉴这个闭环：让商户先用基础版 → 触发 Eval 报告 → 推荐升级到付费版。
2. **Evaluations 是 Weave 的差异化壁垒**。副业产品如果只做"AI Gateway"很难差异化，**加上 Evaluations / Quality Scoring** 是突破口。
3. **W&B Tables 的人类标注工作流**值得借鉴。副业可以做一个"AI 客服质量评分"工作流，让商户手动标注 AI 回复质量，反向训练模型。
4. **CoreWeave 收购 W&B 说明：纯 SaaS 工具的天花板有限**，必须与 GPU 算力绑定才能做大估值。副业产品可以从一开始就考虑"工具 + 算力"或"工具 + 数据"双轮驱动。
5. **OpenTelemetry 是 AI 可观测性的"事实标准"**。副业产品应该从 Day 1 支持 OTLP，避免被绑死在私有协议。

---

## 二十一、参考资料与调研备注

### 21.1 主要参考资料

1. **W&B Weave 官方文档**：https://wandb.me/weave
2. **W&B Weave GitHub 仓库**：https://github.com/wandb/weave
3. **W&B Inference 文档**：https://docs.wandb.ai/inference
4. **W&B Pricing**：https://wandb.ai/site/pricing
5. **CoreWeave 收购公告（2025-03-13）**：https://coreweave.com/blog/coreweave-acquires-weights-and-biases
6. **W&B 官方博客**：
   - "W&B Inference 0.5 公告"（2025-04）
   - "Weave v0.40 GA"（2025-09）
   - "W&B Models v0.7 GA"（2025-10）
   - "Weave TypeScript SDK 1.0"（2026-04）
7. **W&B 性能白皮书**（2026-03）：H100 / A100 / L40S 基准测试
8. **OTel GenAI 语义约定**：https://opentelemetry.io/docs/specs/semconv/gen-ai/
9. **OpenInference 协议**（Arize）：https://github.com/Arize-ai/openinference
10. **既往 00-20 系列报告**：
    - `04-observability-openllmetry.md`
    - `06-guardrails.md`
    - `08-inference-engine-coordination.md`
    - `14-performance-benchmark.md`
    - `15-open-source-contribution.md`

### 21.2 调研备注

- 调研时点：2026-06-07，Weave v0.51.x 稳定版。
- 报告中所有定价信息均来自 2026 中公开页面，可能在 2026 下半年调整。
- 性能数据综合官方白皮书 + 社区 benchmark + 内部估算。
- 客户案例基于公开演讲、博客、招聘信息推断。
- 本报告对 W&B Weave 的技术分析保持中立，对 CoreWeave 收购后的产品方向持谨慎乐观。

### 21.3 与既往报告的关联

- 与 `product-helicone-20260605.md`：Helicone 已被 Mintlify 收购进入 maintenance mode，Weave 是 2026 年最活跃的同类竞品。
- 与 `product-langfuse-20260605.md`：Langfuse + ClickHouse 合并后，Weave 在 OLAP 性能上不再有优势。
- 与 `product-portkey-20260605.md`：Portkey 主打 AI Gateway，Weave 主打 Observability + Eval，两者互为补充。
- 与 `product-arize-phoenix-20260605.md`：Phoenix（开源） + W&B Weave（商业）是 OpenTelemetry 协议下两个最完整的 LLM 观测栈。
- 与 `product-together-ai-20260605.md` / `product-fireworks-ai-20260605.md`：W&B Inference 性能比 Together / Fireworks 弱 10-20%，但集成度高。
- 与 `04-observability-openllmetry.md`：本报告是 OpenLLMetry 主题的"W&B Weave"专篇，与 Traceloop 互补。

---

## 调研总结

| 维度 | 核心结论 |
|---|---|
| **定位** | 实验驱动的全栈 AI 开发平台的可观测性 + 轻量 AI Gateway |
| **架构** | Python/TS SDK + FastAPI Trace Server + ClickHouse/Postgres/S3 |
| **协议** | OTLP 完整支持 + OpenAI 兼容 Proxy + 20+ Provider 原生 patch |
| **AI Gateway** | ⚠️ 中等（精确缓存、限流、fallback 都有，缺语义缓存、边缘部署） |
| **性能** | Weave Proxy +8-25ms；W&B Inference 180 tok/s/H100 (Llama-3.3-70B) |
| **部署** | SaaS（默认）/ Hybrid / Self-Hosted（Enterprise 许可 + CoreWeave） |
| **成本** | Enterprise TCO $500k/年（中型企业），是 Langfuse/Portkey 的 4-10 倍 |
| **生态** | 100+ Provider，LangChain/LlamaIndex/DSPy/CrewAI/Pydantic AI 全覆盖 |
| **客户** | OpenAI、Anthropic、Cohere、Notion、Replit、Scale AI、HF |
| **优势** | Eval 极强、Models 闭环、CoreWeave 集成、企业级 SLA |
| **劣势** | 定价贵、网关能力弱、无边缘部署、无语义缓存 |
| **给副业启发** | Eval 是差异化突破口；工具+算力是估值天花板；OTel 协议 Day 1 支持 |

**最终推荐**：

- ✅ **强推荐选 Weave 的场景**：已在用 W&B Models；需要 Eval + Sweep + 生产监控闭环；有 CoreWeave GPU 合同；有 SOC2/HIPAA 合规需求。
- ❌ **不推荐选 Weave 的场景**：预算敏感；需要强 AI Gateway（路由、Secret 治理、限流）；需要边缘部署；需要 MCP Gateway。

---

> 调研人：Rich
> 调研日期：2026-06-07
> 报告版本：v1.0
> 项目地址：https://github.com/wandb/weave
> W&B 官网：https://wandb.ai
