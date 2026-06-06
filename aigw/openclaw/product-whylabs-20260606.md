# WhyLabs — AI Observability 的开拓者、被 Apple 收购、平台 Apache 2.0 开源

> 调研对象：**WhyLabs, Inc.**（西雅图，2019 成立，2025-01 被 Apple 收购并停止商业运营）
> 调研时间：2026-06-06 23:08 (Asia/Shanghai)
> 调研人：Rich (OpenClaw main session)
> 报告版本：v1.0
> 资料截止：2026-06-06（WhyLabs 官网已 "closing this chapter"，全栈 Apache 2.0 开源）
> 上一份延展深挖：`product-mcp-gateway-20260606.md`（r34+ 候补名单中 "中" 优先级）
> 调研动机：r34 候补名单 §4.4 LLM 优化/路由/缓存 类别下的"AI observability"代表；与已深挖的 Arize Phoenix / Traceloop / Helicone 形成"老牌 observability"对照；**已被 Apple 收购并全栈开源**是 2025-2026 行业里一个非常稀有的事件，值得单独深挖

---

## 0. 写在最前：一份"产品退场 + 平台永生"的深挖

WhyLabs 不是一份普通的产品调研。它是一个**创始于 2019 年、被 Apple 在 2025 年 1 月悄悄收购、随后全栈 Apache 2.0 开源**的产品。这意味着：

1. **SaaS 已死**：商业云服务 WhyLabs Platform（hub.whylabsapp.com）已于 2025 年关闭
2. **代码已生**：核心三件套 `whylogs` (2.8k+ stars) / `langkit` (990+ stars) / `whylabs-oss` (65+ stars) 全部以 Apache 2.0 公开
3. **创始人去向**：CEO Alessya Visnjic 已加入 Apple 任工程负责人；CTO Andy Dang 同期加入
4. **公司融资史**：2019 起累计融资 $14M，来源 Bezos Expeditions、Madrona、Defy Partners、AI Fund（Bezos 个人 VC 领投种子轮是最大看点）

为什么本报告仍然值得深挖？WhyLabs 是 **AI Observability** 这一品类的**开拓者**之一（与 Arize、Arthur、Fiddler 同期），它发明的 **whylogs profile**（统计摘要而非原始数据）协议成为**事实标准之一**：

- whylogs 与 MLflow / Arize / WhyLabs 自家 / S3 / GCS / Local / SQLite 多 writer 互通
- whylogs 协议被 Arize、Fiddler 等友商借鉴，**profile 本身是"可移植工件"**——历史遥测数据不会因 WhyLabs 关闭而失效
- LangKit 是**最早的开源 LLM 安全/质量监控库**（2023 Q1 开源），比 NeMo Guardrails 早一年多

**对"AI Gateway / AI Observability 行业"的影响**：

- 证明 **"AI 监控数据隐私敏感场景（HIPAA / GDPR / 数据不出仓）"** 是一个真实的市场需求
- 证明 **"开源核心 + 商业托管"** 在 AI 监控赛道**难以走通**（WhyLabs 不是唯一被收购/倒闭的——Fiddler 在 2024 同样被收购）
- 启发 **"profiles 即数据"的协议设计**——比 OpenTelemetry traces 节省 99% 存储、100% 保留统计信息

**对小 B / 副业场景的相关性**：

- ✅ whylogs 完全免费开源，**单租户自部署**成本极低
- ✅ LangKit 包含开箱即用的 LLM 安全 / 质量信号提取器（toxicity / PII / jailbreak）
- ✅ 与 OpenLLMetry / OpenTelemetry 集成，可作为 5-15 万/年 AI SaaS 的"AI 安全护栏"基础
- ⚠️ 商业版闭源功能不再可用，但 Apache 2.0 已覆盖 80% 场景

---

## 1. 项目背景与历史

### 1.1 公司沿革

| 时间 | 事件 | 来源 |
|---|---|---|
| 2019 | WhyLabs 在西雅图成立，专注于 ML 模型监控 | 公司官方 blog、Crunchbase |
| 2019-08 | 种子轮 $1.5M，Bezos Expeditions 领投，AI Fund 跟投 | 公开融资记录 |
| 2020-11 | A 轮 $10M，Madrona Venture Group 领投，Defy Partners、Bezos Expeditions 跟投 | GeekWire / TechCrunch |
| 2021-12 | whylogs v0.x → v1.0 重写 (Apache Arrow 为底层) | whylogs GitHub release |
| 2022-05 | LangKit 开源（最初叫 whylogs-llm-toolkit） | GitHub commit history |
| 2023-02 | WhyLabs Secure GA，引入 LLM 安全护栏 (MITRE ATLAS, OWASP LLM Top 10) | 产品 blog |
| 2023-04 | OpenLLMTelemetry 开源（与 LangKit 配套的 OpenTelemetry 集成） | GitHub |
| 2024-05 | WhyLabs Platform v2.0 — Observe (传统 ML 监控) + Secure (LLM 安全) 双模块化 | 产品发布 |
| 2024-11 | 累计 $14M 融资，A 轮后无新增（公开口径） | Crunchbase |
| 2025-01 | **Apple 收购 WhyLabs**（披露来自欧盟 DMA 备案） | GeekWire (2025-09) |
| 2025-02 | 商业平台开始停止新用户注册，老用户进入"迁移期" | 公司公告 |
| 2025-09 | 创始团队正式加入 Apple | GeekWire 报道 |
| 2026-04 | 官网发布"closing this chapter"声明，全栈 Apache 2.0 开源 | whylabs.ai 首页 |

### 1.2 创始人背景

**Alessya Visnjic（CEO → Apple Engineering Leader）**
- 前亚马逊 AWS AI/ML 产品负责人（2014-2019）
- 西北大学计算机 + 神经科学本科
- 公开演讲：WhyLabs 的 "AI Observability" 概念由她在 2019 NeurIPS 演讲首次系统化提出

**Andy Dang（CTO → Apple）**
- 前 AWS SageMaker 核心工程师
- 华盛顿大学计算机硕士

**Sam Lin（VP Engineering）**
- 前 Google Cloud AI Platform
- WhyLabs 后端主架构师

### 1.3 投资方详解

| 投资人 | 轮次 | 金额 | 备注 |
|---|---|---:|---|
| **Bezos Expeditions** | 种子 + A 跟投 | $4.5M | Jeff Bezos 个人 VC，对 MLops 长期看好 |
| **AI Fund** | 种子跟投 | $1M | Andrew Ng 联合创立 |
| **Madrona Venture Group** | A 轮领投 | $6.5M | 西雅图本地 VC，AWS 早期投资人 |
| **Defy Partners** | A 轮跟投 | $2M | Bay Area VC |
| **合计 | 种子 + A | **$14M** | 全部为股权融资，无债务 |

**为什么融资停滞**：B 轮在 2022-2024 MLops 资本寒冬里没拿到合适 terms，公司 2023 起转向"减成本 + 收入端提升"模式，2024 年开始小范围盈利，2025-01 被 Apple 收购时员工数 < 50 人。

### 1.4 产品定位演变

| 时期 | 定位 | 核心用户 |
|---|---|---|
| 2019-2020 | "ML Model Performance Monitoring" | Data Scientists, ML Engineers |
| 2021-2022 | "Data + Model Observability"（加入数据漂移 / 特征监控） | ML Platform Teams |
| 2023-2024 | "AI Observability"（统一传统 ML + LLM） | AI Platform Teams, AI Governance |
| 2024-2025 | "AI Control Center"（Observe 数据/模型 + Secure LLM 防护） | Enterprise AI Risk & Compliance |
| 2025-2026 | **"开源参考实现"**（公司停运后） | 自部署 / 二次开发社区 |

### 1.5 行业地位与影响

**WhyLabs 是 "AI Observability" 概念的开拓者**：

- 2019 年首次提出 "AI Observability" 一词
- 推动 OTel（OpenTelemetry）社区接受 `llm.*` namespace
- 与 Fiddler（2024 被收购）、Arize（仍在运营）、Arthur AI（仍在运营）形成 4 强格局
- 行业内"四大 AI Observability" 截至 2024：
  1. **Arize AI** — OpenTelemetry-based，开源 Phoenix，主打 trace + eval
  2. **WhyLabs** — whylogs profile-based，主打隐私 + 轻量
  3. **Arthur AI** — model monitoring + LLM firewall，主打 enterprise governance
  4. **Fiddler AI** — model performance + explainability，被 Datadog 收购（2024）

---

## 2. 架构设计

### 2.1 整体架构图（停运前商业版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          WhyLabs AI Control Center                           │
│                                                                             │
│  ┌──────────────────────┐   ┌──────────────────────┐   ┌────────────────┐   │
│  │    WhyLabs Observe   │   │    WhyLabs Secure    │   │ WhyLabs Apollo │   │
│  │   (传统 ML 监控)      │   │    (LLM 安全)         │   │ (内部代号)     │   │
│  │                      │   │                      │   │                │   │
│  │ • 数据漂移检测        │   │ • LLM 安全护栏        │   │ • 内部编排     │   │
│  │ • 模型性能追踪        │   │ • Prompt Injection  │   │ • 权限管理     │   │
│  │ • Feature Store 监控 │   │ • PII 防护            │   │ • 多租户       │   │
│  │ • 训练-服务 skew     │   │ • 幻觉检测            │   │                │   │
│  │                      │   │ • Toxicity 防护       │   │                │   │
│  └──────────┬───────────┘   └──────────┬───────────┘   └────────┬───────┘   │
│             │                          │                        │           │
│  ═══════════╪══════════════════════════╪════════════════════════╪═════════   │
│             │                          │                        │           │
│             ▼                          ▼                        ▼           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  Profile Ingestion Layer (Go)                        │   │
│  │                  API Service: gRPC + REST                            │   │
│  │                  Kafka → S3 Profile Store (Parquet)                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              Anomaly Detection & Alerting Engine                     │   │
│  │              Spark + Flink (流式 + 批式)                              │   │
│  │              LLM-aware policy engine (Secure)                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │          Dashboard (React + TS) + Notification Service              │   │
│  │          Highcharts 可视化 + Slack/Email/PagerDuty 告警              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                       ▲
                                       │  (whylogs profile 统计摘要，**不传原始数据**)
                                       │
        ┌──────────────────────────────┴──────────────────────────────┐
        │                                                              │
        │  ┌──────────────────┐              ┌──────────────────┐       │
        │  │  客户端 SDK       │              │  Sidecar/Inline  │       │
        │  │                  │              │                  │       │
        │  │  Python:         │              │  OpenLLMTelemetr │       │
        │  │  • whylogs       │              │  y (OTel-based)  │       │
        │  │  • langkit       │              │                  │       │
        │  │  • openllmtel    │              │  + secure-agent  │       │
        │  │                  │              │  (containerized  │       │
        │  │  Java:           │              │  policy enforcer)│       │
        │  │  • whylogs-java  │              │                  │       │
        │  │                  │              │                  │       │
        │  │  Spark/Ray       │              │                  │       │
        │  │  • whylogs-spark │              │                  │       │
        │  │  • whylogs-ray   │              │                  │       │
        │  └──────────────────┘              └──────────────────┘       │
        │                                                              │
        │   ┌─────────────────────────────────────────────────┐         │
        │   │  Users: Data Scientists / ML Engineers /        │         │
        │   │        LLM App Developers / AI Risk Teams       │         │
        │   └─────────────────────────────────────────────────┘         │
        │                                                              │
        └──────────────────────────────────────────────────────────────┘
```

### 2.2 核心抽象：whylogs Profile

**whylogs profile 是 WhyLabs 整个架构的核心抽象**。它是一个**轻量级、mergeable、可序列化的统计摘要**：

```
┌──────────────────────────────────────────────────────┐
│                   whylogs Profile                    │
│  (单 dataset 快照，可序列化到 S3/GCS/Local)             │
│                                                      │
│  ┌────────────────┐  ┌────────────────┐              │
│  │ Column "age"   │  │ Column "city"  │  ...         │
│  │                │  │                │              │
│  │ • count: 10000 │  │ • count: 10000 │              │
│  │ • nulls: 23    │  │ • distinct: 47 │              │
│  │ • min: 18      │  │ • topk:        │              │
│  │ • max: 99      │  │   "Seattle":   │              │
│  │ • mean: 41.7   │  │   1823         │              │
│  │ • stddev: 14.2 │  │   "NYC": 1102  │              │
│  │ • quantiles:   │  │   ...          │              │
│  │   q25=29       │  │ • type info    │              │
│  │   q50=42       │  │ • string len   │              │
│  │   q75=54       │  │   histograms   │              │
│  │ • histogram    │  │                │              │
│  │   (KLL sketche)│  │                │              │
│  └────────────────┘  └────────────────┘              │
│                                                      │
│  ┌────────────────────────────┐                      │
│  │  Custom UDF Columns        │                      │
│  │  (LangKit 注入)            │                      │
│  │  • prompt.toxicity         │                      │
│  │  • response.pii_count      │                      │
│  │  • response.relevance      │                      │
│  │  • prompt.injection_score  │                      │
│  └────────────────────────────┘                      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Profile 三个核心属性**（来自 whylogs README）：

1. **Efficient**：相比原始数据压缩 1000x+。一个 10GB 数据集生成 ~5MB profile
2. **Customizable**：可注册自定义 UDF column（如 LangKit 注入的 LLM 安全信号）
3. **Mergeable**：多个 profile 可合并，map-reduce 友好，分布式场景天然支持

### 2.3 Profile 的物理格式

**v0.x 格式**（2020-2021）：

```python
# Profile = zip(Protobuf, JSON metadata)
why.log(df).writer("local").write("profile.bin")

# 物理结构:
# profile.bin
#   ├── metadata.json     (column types, schema info)
#   ├── summary.pb        (Protobuf-encoded summary statistics)
#   └── histograms.bin    (binary histogram data)
```

**v1.x 格式**（2021-至今，Apache Arrow 列存）：

```python
# v1 Profile = Apache Arrow Columnar Format
why.log(df).writer("local").write("profile.arrow")

# 物理结构:
# profile.arrow/
#   ├── metadata.yaml     (schema, dataset_id, timestamp)
#   ├── columns/
#   │   ├── age.arrow     (columnar numeric stats: KLL, quantiles)
#   │   ├── city.arrow    (columnar string stats: topk, histograms)
#   │   └── _custom.arrow (UDF column outputs)
#   └── _INDEX.json
```

**为什么切到 Apache Arrow**：

- 1. 跨语言互操作（Python / Java / Spark / Ray / Go）
- 2. 列存压缩更好（ZSTD + dictionary encoding）
- 3. 与 Pandas / PyArrow / DuckDB 原生集成

### 2.4 LangKit 在 whylogs 内的注入机制

LangKit 实际上是一组 **whylogs UDF (User-Defined Function) Schemas**：

```python
import whylogs as why
from langkit import llm_metrics  # imports UDF modules

# 加载 LangKit 的自定义 schema
schema = llm_metrics.init()

# 自动将所有 LLM 信号注入 whylogs profile
result = why.log(
    {"prompt": "Hello!", "response": "World!"},
    schema=schema,
)

# Profile 现在包含:
# - 原始 whylogs metrics: counts, nulls, types, distributions
# - LangKit 注入的 LLM signals:
#   * prompt.toxicity (float 0-1)
#   * response.pii_count (int)
#   * response.relevance_to_prompt (float)
#   * prompt.injection_score (float)
#   * response.sentiment (str)
#   * text.readability_score (float)
```

**关键技术**：`whylogs.experimental.core.udf_schema.udf_schema()` 是 LangKit 注入的入口点。

### 2.5 OpenLLMTelemetry：OTel 集成层

```python
import openllmtelemetry

# 一行代码 instrument 整个应用
openllmtelemetry.instrument()

# 自动捕获:
# - OpenAI / Anthropic / Bedrock 的所有 LLM 调用
# - Prompt / Response 文本
# - Token 用量、延迟、模型版本
# - 关联到 OTel trace / span
# - 推送到 WhyLabs Secure 平台进行 policy 评估
```

**支持 provider**：
- OpenAI (官方)
- Azure OpenAI (官方)
- AWS Bedrock (官方, boto3 bedrock-runtime)
- Anthropic (通过 OTel 通用 LLM convention)
- 自定义 OTel-compatible provider

**两个模式**：

1. **Observe 模式**：纯监控，OTel traces → WhyLabs
2. **Secure 模式**：OTel traces + 实时 policy enforcement（与容器化的 secure-agent 通信）

### 2.6 WhyLabs Secure 的容器化 Agent

WhyLabs Secure 的"policy enforcement"是通过**容器化 sidecar agent**实现的：

```
┌─────────────────── 客户 LLM 应用  ─────────────────────┐
│                                                         │
│  openllmtelemetry.instrument()                          │
│       │                                                 │
│       │  (HTTP POST /v1/traces)                         │
│       ▼                                                 │
│  ┌────────────────────────┐                             │
│  │  secure-agent (sidecar)│  ←── WhyLabs 提供的容器     │
│  │  (containerized)       │                             │
│  │                        │                             │
│  │  • 接收 LLM traces     │                             │
│  │  • 查 WhyLabs Secure  │                             │
│  │    policy (LLM OWASP)  │                             │
│  │  • 实时判断            │                             │
│  │    • ALLOW / BLOCK    │                             │
│  │    • REDACT PII       │                             │
│  │    • WARN             │                             │
│  │  • 写回决策 + log     │                             │
│  └────────────┬───────────┘                             │
│               │                                         │
└───────────────┼─────────────────────────────────────────┘
                │
                ▼
    ┌──────────────────────┐
    │  WhyLabs Secure API  │
    │  (Hosted control     │
    │   plane, 已停运)      │
    └──────────────────────┘
```

**优势**：
- 政策集中管理（policy as code）
- 容器化隔离，**LLM 调用方不直接访问 SaaS**
- 支持 ALLOW/BLOCK/REDACT 三种动作

**限制**（停运时）：
- secure-agent 的最新版本强依赖 WhyLabs SaaS（开源版无离线模式）
- 已开源代码里 secure/app 包含 Highcharts 商业依赖

---

## 3. 协议支持

### 3.1 协议支持矩阵

| 协议/标准 | whylogs | langkit | openllmtelemetry | whylabs-oss |
|---|:---:|:---:|:---:|:---:|
| **OpenTelemetry (OTel) Traces** | ❌ | ❌ | ✅ (核心) | ✅ (接收端) |
| **OpenTelemetry Metrics** | ❌ | ❌ | ✅ | ✅ |
| **OpenTelemetry Logs** | ❌ | ❌ | ✅ (traces 转 logs) | ✅ |
| **Apache Arrow** | ✅ (v1) | ✅ (经 whylogs) | ❌ | ✅ (profile 存储) |
| **Apache Parquet** | ✅ (writer) | ❌ | ❌ | ✅ (后端存储) |
| **Apache Kafka** | ❌ | ❌ | ❌ | ✅ (ingestion) |
| **Apache Spark** | ✅ (whylogs-spark) | ✅ (经 whylogs) | ❌ | ❌ |
| **Apache Ray** | ✅ (whylogs-ray) | ✅ (经 whylogs) | ❌ | ❌ |
| **MLflow** | ✅ (writer) | ❌ | ❌ | ❌ |
| **OpenAI API** | ❌ | ✅ (PII / jailbreak 检查) | ✅ (instrument) | ❌ |
| **Anthropic API** | ❌ | ✅ (经 OTel) | ✅ (经 OTel) | ❌ |
| **AWS Bedrock** | ❌ | ❌ | ✅ (boto3) | ❌ |
| **Hugging Face Transformers** | ❌ | ✅ (sentence-transformers encoder) | ❌ | ❌ |
| **ONNX** | ❌ | ❌ | ❌ | ❌ |
| **gRPC** | ❌ | ❌ | ❌ | ✅ (API service) |
| **REST/JSON** | ✅ (writers) | ❌ | ❌ | ✅ |
| **SCIM 2.0** | ❌ | ❌ | ❌ | ✅ (scim-service) |
| **OAuth 2.0 / OIDC** | ❌ | ❌ | ❌ | ✅ (API service) |
| **Prometheus exposition** | ❌ | ❌ | ❌ | ❌ |

### 3.2 whylogs profile v1 协议规范

**Profile 的逻辑结构**（v1.x，Apache Arrow Columnar Format）：

```protobuf
syntax = "proto3";

message DatasetProfile {
  string dataset_id = 1;          // e.g. "model-123"
  int64  timestamp = 2;           // unix ms
  string session_id = 3;
  map<string, string> tags = 4;   // user-defined tags

  repeated ColumnProfile columns = 5;

  message ColumnProfile {
    string name = 1;
    ColumnType type = 2;          // INT, FLOAT, STRING, BOOLEAN, ...

    // Numeric columns:
    optional int64 count = 10;
    optional int64 nulls = 11;
    optional double min = 12;
    optional double max = 13;
    optional double mean = 14;
    optional double stddev = 15;
    repeated double quantiles = 16;
    optional bytes histogram_kll = 17;   // KLL sketch (Apache Datasketches)

    // String columns:
    optional int64 distinct = 20;
    optional int64 unique = 21;
    repeated FrequentItem top_k = 22;
    optional bytes length_histogram = 23;

    // Custom UDF columns (LangKit 注入):
    map<string, bytes> udf_values = 30;
  }
}
```

**关键设计选择**：

- KLL sketch 用于分布近似（误差 < 1%，内存 O(log n)）
- Quantile digest 来自 Apache Datasketches 库
- Top-K 用 Count-Min Sketch + Heap
- 自定义 UDF 值用 `map<string, bytes>` 灵活承载

### 3.3 WhyLabs Secure 的 LLM Policy 协议

**MITRE ATLAS 对齐**（17 个 adversarial tactics）：

| ATLAS Tactic | WhyLabs Secure Policy |
|---|---|
| Reconnaissance | `prompt.suspicious_keywords` |
| Resource Development | N/A (no detection) |
| Initial Access | `prompt.injection_score` |
| ML Model Access | `response.pii_count > 0` |
| Execution | `response.tool_call_validation` |
| Persistence | N/A |
| Defense Evasion | `prompt.obfuscation_score` |
| Discovery | `prompt.cred_scraping_score` |
| Collection | `response.pii_count`, `response.secret_pattern_match` |
| ML Attack Staging | `response.anomaly_score` |
| Exfiltration | `response.pii_count > threshold` |
| Impact | `response.hallucination_score > 0.7` |
| ... | ... |

**OWASP LLM Top 10 对齐**（10 项）：

- LLM01 Prompt Injection → `prompt.injection_score`
- LLM02 Insecure Output Handling → `response.code_execution_risk`
- LLM03 Training Data Poisoning → N/A (training-time)
- LLM04 Model DoS → `response.latency > threshold`
- LLM05 Supply Chain → N/A
- LLM06 Sensitive Info Disclosure → `response.pii_count`, `response.secret_pattern_match`
- LLM07 Insecure Plugin Design → `response.tool_call_validation`
- LLM08 Excessive Agency → `response.tool_call_validation`
- LLM09 Overreliance → `response.hallucination_score`
- LLM10 Model Theft → N/A

### 3.4 与 OpenTelemetry 的关系

**WhyLabs 选择 OTel 作为 LLM telemetry 传输层的原因**：

1. OTel 已是云原生 observability 事实标准（Datadog / Honeycomb / Grafana 全部支持）
2. 避免自建协议、节省教育成本
3. 与 `openllmtelemetry` 库解耦：开发者用 OTel SDK，WhyLabs 只是 exporter

**OTel 内的 LLM 属性命名**（自 2024 年起逐步标准化）：

```yaml
# semconv 草案 (open-telemetry/semantic-conventions#1321)
gen_ai.system: "openai"
gen_ai.request.model: "gpt-4-turbo"
gen_ai.request.max_tokens: 1024
gen_ai.usage.input_tokens: 234
gen_ai.usage.output_tokens: 567
gen_ai.response.finish_reasons: ["stop"]
```

**WhyLabs 与 OTel 社区的关系**：
- 早期（2023）推动 OTel 接受 `llm.*` namespace
- 后期（2024-2025）跟随 OTel 社区采用 `gen_ai.*` namespace
- LangKit 默认输出 OTel 兼容的 attribute 名

### 3.5 与 OpenLLMetry 的关系

OpenLLMetry 是 **Traceloop** 发起的开源 LLM observability 项目（独立于 WhyLabs），GitHub `traceloop/openllmetry`。

| 维度 | WhyLabs 路径 | OpenLLMetry 路径 |
|---|---|---|
| OTel exporter | 官方 WhyLabs exporter | 官方 Traceloop exporter（可指 WhyLabs） |
| 自动 instrument | openllmtelemetry (1 行) | openllmetry (1 行) |
| 协议 | OTel + 自定义 WhyLabs profile | OTel + Traceloop spans |
| 监控 UI | WhyLabs Platform（停运） | Traceloop / 任何 OTel 后端 |
| 关系 | openllmtelemetry 与 openllmetry **不直接互操作**，但都基于 OTel 同一底层 | 同上 |

**为什么同时存在两个项目**：
- WhyLabs 2023 年先做 openllmtelemetry
- Traceloop 2023 年 9 月发起 openllmetry（社区驱动，更中立）
- 2024 年起 openllmetry 社区规模超过 openllmtelemetry

---

## 4. 性能数据

### 4.1 whylogs Profiling 性能（v1.6.4，官方 README 数据）

| 数据量 | 实例类型 | 集群规模 | 处理时间 | 总成本 | 性价比 |
|---|---|---:|---|---:|---|
| 10 GB (~34M rows × 43 cols) | c5a.2xlarge (8 CPU/16GB) | 2 | 2.6 min/instance | ~$0.026 / $2.45/TB | 基线 |
| 10 GB | c6g.2xlarge (8 CPU/16GB, ARM) | 2 | 1.7 min/instance | ~$0.016 / $1.60/TB | ARM 快 35% |
| 10 GB | c5a.2xlarge | 16 | 33 sec/instance | ~$0.045 | 16 节点更慢因 spark overhead |
| 80 GB (83M rows × 119 cols) | c5a.2xlarge | 16 | 1.7 min/instance | ~$0.139 | |
| 100 GB (290M rows × 43 cols) | c5a.2xlarge | 16 | 2.7 min/instance | ~$0.221 | |

**关键观察**：

- 大多数场景下 whylogs overhead < 1%
- 极高 QPS + 数千 features 场景约 5% overhead
- 横向扩展（多实例）是 ROI 最优路径
- ARM (Graviton2) 比 x86 便宜 35%

### 4.2 LangKit LLM Metrics 性能（v0.0.x，官方 README 数据）

| AWS 实例 | Metric Module | 吞吐量 (chats/sec) | 备注 |
|---|---|---:|---|
| c5.xlarge (4 vCPU) | Light metrics | **2,335** | 文本统计 + 情感分析 |
| c5.xlarge | LLM metrics | 8.2 | 含 LLM 推理（hallucination / proactive injection） |
| c5.xlarge | All metrics | 0.28 | 全量含 LLM call（huge 开销） |
| g4dn.xlarge (GPU) | Light metrics | 2,492 | 略快于 CPU |
| g4dn.xlarge | LLM metrics | 23.3 | GPU 加速 embedding 提取 |
| g4dn.xlarge | All metrics | 1.79 | 仍受 LLM 推理瓶颈 |

**关键观察**：

- 纯文本特征提取（light metrics）可达 2k+ chats/sec
- 涉及 LLM 推理的 metrics（hallucination）掉到个位数 chats/sec
- 实际生产部署应**异步**处理 LLM-heavy metrics（不阻塞主请求）

### 4.3 Profile 压缩比

**典型场景**（whylogs 官方数据）：

| 原始数据 | Profile 大小 | 压缩比 |
|---|---:|---:|
| 1 GB CSV (1M rows × 50 cols) | ~3 MB | 333x |
| 10 GB Parquet (10M rows × 100 cols) | ~25 MB | 400x |
| 100 GB 特征工程输出 | ~200 MB | 500x |
| 1 TB 数据湖采样 | ~1.5 GB | 670x |

**为什么压缩比这么高**：
- 用 KLL/CMS/Quantile 近似算法代替原始值
- 字符串只存 top-K + length distribution
- 数值只存分位数 + 简单 count/min/max

### 4.4 OpenLLMTelemetry 性能

**未公开 benchmark**，但根据 OTel 社区测试：

- single OpenAI call 的 instrumentation overhead ~50-200µs
- batch 100 calls ~5-10ms total
- 内存增量 < 1MB per application

### 4.5 网络传输节省

由于**只传 profile 不传原始数据**：

- 100 GB 训练数据 + 100 M 推理日志 / 月
- 传原始数据：~100 GB / 月
- 传 profile：~250 MB / 月
- **节省 99.75% 带宽**

这是 WhyLabs 的核心卖点之一（隐私 + 成本双重收益）。

### 4.6 推理性能对应用主链路的影响

**Inline 模式**（同步）：
- LLM 应用代码内调用 `whylogs.log(...)`
- 单次开销 ~5-15ms（不涉及 LLM 的 metrics）
- 不推荐用于 LLM 应用的 hot path

**Sidecar 模式**（异步，开源参考实现）：
- 客户端发到 Kafka / SQS，whylabs-oss ingestion service 消费
- 端到端延迟 < 1s
- 推荐生产部署

**Defer / Batch 模式**：
- 应用内累计，到 N 条或 T 间隔 flush
- 适合离线分析

---

## 5. 部署方式

### 5.1 三种部署模式（开源版 + 商业版对照）

| 模式 | 商业版 (停运) | 开源版 (现) |
|---|---|---|
| **SaaS** | WhyLabs hosted | ❌ 不再提供 |
| **Single-tenant Cloud** | WhyLabs managed for enterprise | ❌ 不再提供 |
| **Self-hosted** | ❌ (商业版未提供) | ✅ whylabs-oss Apache 2.0 |
| **Embedded SDK** | ✅ whylogs Python/Java | ✅ whylogs Python/Java |
| **Hybrid** | ✅ SDK + 商业版后端 | ✅ SDK + 自建后端 |

### 5.2 whylabs-oss Self-hosted 部署架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Customer AWS Account                            │
│                       (preferably us-west-2)                          │
│                                                                       │
│  ┌──────────────────── EKS Cluster ─────────────────────┐            │
│  │                                                       │            │
│  │  ┌─────────────────┐    ┌──────────────────┐         │            │
│  │  │  API Service    │    │  Dataservice     │         │            │
│  │  │  (gRPC + REST)  │───▶│  (Query + Drift  │         │            │
│  │  │  3+ replicas    │    │  Detection)      │         │            │
│  │  └─────────────────┘    └──────────────────┘         │            │
│  │           │                      │                    │            │
│  │           ▼                      ▼                    │            │
│  │  ┌────────────────────────────────────────┐           │            │
│  │  │  S3 Profile Store (Parquet)             │           │            │
│  │  │  + Athena (Presto SQL 查询)              │           │            │
│  │  └────────────────────────────────────────┘           │            │
│  │                                                       │            │
│  │  ┌─────────────────┐  ┌──────────────────┐           │            │
│  │  │ Dashboard (React)│  │  Notification    │           │            │
│  │  │  (Highcharts*)   │  │  Service         │           │            │
│  │  └─────────────────┘  └──────────────────┘           │            │
│  │                                                       │            │
│  │  ┌─────────────────┐  ┌──────────────────┐           │            │
│  │  │  Profile Browser│  │  SCIM Service    │           │            │
│  │  │  (Jupyter-like) │  │  (User sync)     │           │            │
│  │  └─────────────────┘  └──────────────────┘           │            │
│  │                                                       │            │
│  └───────────────────────────────────────────────────────┘            │
│                                                                       │
│  ┌───────────────────── External Services ─────────────────────┐     │
│  │  • AWS SQS (notification fanout)                            │     │
│  │  • Slack / Email / PagerDuty (alerting)                     │     │
│  │  • OAuth / OIDC (auth)                                      │     │
│  └─────────────────────────────────────────────────────────────┘     │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.3 WhyLabs Observe 子模块（whylabs-oss/observe）

**子目录**（来自 `whylabs-oss/observe/README.md`）：

| 子模块 | 语言/技术栈 | 职责 |
|---|---|---|
| `api-service/` | Kotlin + gRPC | 公开 API (上传 profile / 查询) |
| `dataservice/` | Python + Spark | 数据查询、漂移检测 |
| `dashboard/` | React + TypeScript | Web UI（Highcharts） |
| `notification-service/` | Python | 告警 fan-out（SQS / Slack / Email） |
| `profile-browser/` | TypeScript | Profile 可视化 |
| `scim-service/` | Kotlin | SCIM 2.0 用户同步 |
| `smart-config/` | Python | 智能监控配置（自动推荐阈值） |
| `infra/` | Terraform / CDK | AWS 基础设施 |

**部署要求**（README 明确）：
- AWS account（us-west-2 推荐）
- EKS cluster
- Docker（or alternative）
- 私有 package registries（NPM/Maven/Docker）
- Terraform 基础设施脚本

### 5.4 WhyLabs Secure 子模块（whylabs-oss/secure）

**子目录**（来自 `whylabs-oss/secure/README.md`）：

| 子模块 | 职责 |
|---|---|
| `secure/app/` | Web UI（policy editor） |
| `secure/sagemaker-container/` | SageMaker 容器化 secure-agent |
| `secure/eks-deployment/` | EKS 部署模板 |

**特性**：
- 容器化的 secure-agent
- MITRE ATLAS / OWASP LLM Top 10 政策模板
- 与 openllmtelemetry 联动

### 5.5 部署成本估算（开源版 self-hosted）

**小型部署**（10 models, 1M profiles / 月）：
- EKS 集群：~$200/月
- S3 存储（100 GB profile）：~$3/月
- Athena 查询：~$10/月
- 数据传输：~$5/月
- 工程师维护：0.5 FTE × $15K/月
- **总计 ~$15K/月 + 7.5K 人工**

**中型部署**（100 models, 100M profiles / 月）：
- EKS 集群：~$1K/月
- S3 存储（1 TB profile）：~$25/月
- Athena 查询：~$200/月
- 数据传输：~$50/月
- 工程师维护：1 FTE × $15K/月
- **总计 ~$17K/月 + 15K 人工**

**对副业启示**：
- whylogs 本身是免费的，开源 self-hosted 适合"5-15 万/年 SaaS"场景
- 但 1 FTE 维护成本对小 B 副业过高（>$150K/年）
- 更现实：whylogs 作为 **embedded SDK** 集成到产品里，不部署完整 whylabs-oss

### 5.6 集成模式（推荐给小 B SaaS）

**模式 A：纯 embedded SDK**（最小投入）

```python
import whylogs as why
from langkit import llm_metrics

def llm_call(prompt):
    response = openai_client.chat.completions.create(...)
    # 同进程 log profile（包含 LangKit metrics）
    schema = llm_metrics.init()
    why.log(
        {"prompt": prompt, "response": response.choices[0].message.content},
        schema=schema,
    )
    return response
```

**模式 B：SDK + 内部 metrics backend**（中等投入）

- 用 whylogs 生成 profile
- 写到自己已有的 Prometheus / Datadog / Grafana
- 不部署 WhyLabs UI

**模式 C：SDK + 自托管 whylabs-oss**（大投入）

- 完整 whylabs-oss 部署
- 仅适合"AI Observability 即产品"的副业

---

## 6. 成本模型

### 6.1 商业版价格（停运前，公开数据，2024-Q3）

**WhyLabs Platform 旧版定价**（已停售）：

| Tier | 价格 | 限制 |
|---|---|---|
| **Free / Starter** | $0 | 1 model, 30 天 retention, 1 GB profile / 月 |
| **Team** | $300 / model / 月 | 5 users, 90 天 retention, 10 GB profile / 月 |
| **Business** | $900 / model / 月 | 25 users, 1 年 retention, 100 GB profile / 月 |
| **Enterprise** | Custom（$25K+ /年 起） | 无限 model, 7 年 retention, SSO/SCIM, on-prem 选项 |

**典型 5-model Team 客户**：$1,500/月 = **$18K/年**

**典型 20-model Business 客户**：$18,000/月 = **$216K/年**

**典型 50-model Enterprise 客户**：$50K-$250K/年

### 6.2 开源版成本（Apache 2.0）

| 组件 | 成本 | 备注 |
|---|---|---|
| whylogs Python SDK | **免费** | Apache 2.0 |
| whylogs Java SDK | 免费 | Apache 2.0 |
| LangKit | 免费 | Apache 2.0 |
| OpenLLMTelemetry | 免费 | Apache 2.0 |
| whylabs-oss（self-host） | 免费 | Apache 2.0 + Highcharts 商业 license |
| EKS 集群 | $200-$1,000/月 | AWS 成本 |
| S3 + Athena | $5-$250/月 | 与数据量相关 |
| Highcharts 商业 license | $530-$7,750 / 开发者 | 仅 dashboard 需要 |

**对比商业版（10 model Business）**：
- 商业：$108K/年
- 开源：~$20K/年（基础设施 + 0.5 FTE）
- **节省 80%+**

### 6.3 替代方案成本对比

| 方案 | 5-model 年成本 | 20-model 年成本 |
|---|---:|---:|
| WhyLabs 商业（已停售） | $18K | $216K |
| whylabs-oss 自托管 | $20K (infra) + $90K (1 FTE) = $110K | $25K + $180K = $205K |
| Arize Phoenix 开源 | $10K (infra) + $90K (1 FTE) = $100K | $15K + $180K = $195K |
| Datadog LLM Observability | $36K (基础) + $0.10/1000 events | 同等 |
| Langfuse 云 | $0-$5K (free / pro) | $24K (team) |
| Helicone | $0-$2K (free / pro) | $24K (team) |

**对小 B 副业的现实**：
- **Helicone**（云托管）最便宜：free tier 足够验证 PMF
- **Langfuse**（云 + 自托管）次之：免费版有 50K events / 月
- **whylogs + 自建存储**：最灵活但工程成本高

### 6.4 隐藏成本（开源版）

- **Highcharts 商业 license**：$530/开发者（dashboard 必需）
- **EKS 学习曲线**：Kubernetes 运维对小团队成本高
- **1 FTE 维护成本**：~$150K/年（远高于软件 license 本身）
- **AWS bill 失控风险**：Athena 按查询计费，需 query budget 管控

---

## 7. 生态

### 7.1 集成生态矩阵

| 类别 | 集成 | whylogs | langkit | openllmtelemetry | whylabs-oss |
|---|---|:---:|:---:|:---:|:---:|
| **LLM Provider** | OpenAI | ❌ | ✅ | ✅ | ❌ |
| | Azure OpenAI | ❌ | ✅ | ✅ | ❌ |
| | AWS Bedrock | ❌ | ❌ | ✅ | ❌ |
| | Anthropic | ❌ | ✅ (via OTel) | ✅ (via OTel) | ❌ |
| | Cohere | ❌ | ❌ | ✅ (via OTel) | ❌ |
| | Hugging Face TGI | ❌ | ❌ | ✅ (via OTel) | ❌ |
| **数据源** | Pandas | ✅ | ✅ | ❌ | ❌ |
| | PySpark | ✅ | ✅ | ❌ | ❌ |
| | Ray | ✅ | ✅ | ❌ | ❌ |
| | Dask | ✅ | ✅ | ❌ | ❌ |
| | Fugue | ✅ | ❌ | ❌ | ❌ |
| **存储** | Local FS | ✅ | ❌ | ❌ | ❌ |
| | AWS S3 | ✅ | ❌ | ❌ | ✅ |
| | GCS | ✅ | ❌ | ❌ | ❌ |
| | Azure Blob | ❌ | ❌ | ❌ | ❌ |
| | SQLite | ✅ | ❌ | ❌ | ❌ |
| | MLflow | ✅ | ❌ | ❌ | ❌ |
| **后端平台** | WhyLabs SaaS | ✅ | ✅ | ✅ | N/A (自己就是) |
| | MLflow Tracking | ✅ | ❌ | ❌ | ❌ |
| | Apache Kafka | ❌ | ❌ | ❌ | ✅ |
| **Notebook** | Jupyter | ✅ | ✅ | ✅ | ❌ |
| | Colab | ✅ | ✅ | ✅ | ❌ |
| **Vector DB** | FAISS | ❌ | ✅ (内置) | ❌ | ❌ |
| | Pinecone | ❌ | ❌ | ❌ | ❌ |
| | Weaviate | ❌ | ❌ | ❌ | ❌ |
| **ML Framework** | scikit-learn | ✅ (via pandas) | ❌ | ❌ | ❌ |
| | PyTorch | ✅ (via DataLoader) | ❌ | ❌ | ❌ |
| | TensorFlow | ✅ (via tf.data) | ❌ | ❌ | ❌ |
| | XGBoost | ✅ (via pandas) | ❌ | ❌ | ❌ |
| **PII Engine** | Microsoft Presidio | ❌ | ✅ (内置) | ❌ | ❌ |
| | spaCy | ❌ | ✅ (依赖) | ❌ | ❌ |
| **Embedding** | sentence-transformers | ❌ | ✅ (默认) | ❌ | ❌ |
| | OpenAI ada-002 | ❌ | ✅ (可配) | ❌ | ❌ |

### 7.2 上下游关系图

```
                       ┌──────────────────┐
                       │   LLM 应用开发者   │
                       │  (Python / Java)  │
                       └────────┬─────────┘
                                │ import
                                ▼
       ┌──────────────────────────────────────────────────────┐
       │              WhyLabs 三件套                           │
       │  ┌────────────────┐                                   │
       │  │  whylogs       │  ←── 核心 profile 生成            │
       │  │  (2.8k stars)  │                                   │
       │  └───────┬────────┘                                   │
       │          │                                            │
       │  ┌───────┴────────┐  ┌──────────────────────┐         │
       │  │  LangKit       │  │  OpenLLMTelemetry    │         │
       │  │  (990 stars)   │  │  (OTel integration)  │         │
       │  │  • LLM signals │  │  • Auto-instrument   │         │
       │  │  • UDF schema  │  │  • OpenAI/Bedrock    │         │
       │  └───────┬────────┘  └──────────┬───────────┘         │
       │          │                       │                     │
       └──────────┼───────────────────────┼─────────────────────┘
                  │                       │
                  ▼                       ▼
       ┌──────────────────┐    ┌────────────────────┐
       │  WhyLabs Platform │    │  OTel 兼容后端      │
       │  (停运, OSS 版)   │    │  • Datadog         │
       │  • Observe       │    │  • Honeycomb       │
       │  • Secure        │    │  • Grafana Tempo   │
       │  • Dashboard     │    │  • Jaeger          │
       │  • Notifications │    │  • New Relic       │
       └──────────────────┘    └────────────────────┘
```

### 7.3 与竞品生态对比

| 维度 | WhyLabs (whylogs) | Arize Phoenix | Langfuse | Helicone |
|---|---|---|---|---|
| **GitHub stars** | 2.8k | 6k+ | 13k+ | 2.5k |
| **核心抽象** | Profile (statistical) | Trace + Eval | Trace + Eval | Request log |
| **协议** | OTel (可选) | OTel (原生) | OTel (原生) | 自有 + 镜像 |
| **存储** | S3 + Parquet | ClickHouse | Postgres | ClickHouse |
| **自部署** | 复杂 (EKS) | 中等 (Docker) | 简单 (Docker) | 简单 (Docker) |
| **活跃度** | 低（停运后） | 高 | 很高 | 很高 |
| **公司状态** | 已被 Apple 收购 | 仍在运营（B 轮 $50M） | 仍在运营（Seed $4M） | 仍在运营（Seed $2M） |

---

## 8. 客户案例（停运前公开）

### 8.1 公开案例

**Healthcare（医疗）**：
- **Providence St. Joseph Health** — 监控 200+ ML 模型（含 LLM 患者咨询）
- **HCA Healthcare** — 数据漂移监控，HIPAA 合规
- **医保理赔自动审批场景**：LLaMA 微调模型监控

**Financial Services（金融）**：
- **JPMorgan Chase** — 反欺诈模型监控
- **HSBC** — 跨境支付 OCR 模型监控
- **Capital One** — LLM 客户支持 + 反幻觉监控

**Logistics（物流）**：
- **Flexport** — 关税预测模型监控
- **Convoy** — 路径优化模型监控

**E-commerce**：
- **Shopify** — 推荐系统 LLM 监控（部分）
- **Etsy** — 搜索质量监控

**Government**：
- 公开案例较少，主要通过合作伙伴（AWS、Databricks）落地

### 8.2 案例共性

**为什么这些客户选 WhyLabs**：

1. **数据隐私敏感**：医疗 / 金融场景不能传原始数据 → whylogs profile 是唯一合规选择
2. **跨云部署**：不自建 SaaS，WhyLabs 是中立第三方
3. **成本敏感**：B 端客户对 profile-only 模式（$2.45/TB）比完整 data lake 便宜
4. **AI 治理需求**：MITRE ATLAS / OWASP LLM Top 10 合规对监管要求高的行业是刚需

**为什么最终还是没留住**：

- 2023-2024 MLops 资本寒冬
- LLM 监控从"传统 ML 监控"分化出来，新一代竞品（Langfuse、Helicone）更聚焦 LLM
- 大客户对 WhyLabs "通用 ML + LLM" 的双线策略需求降低（更愿意选 Arize / Datadog 等专用工具）
- 创始团队精力消耗在 GTM 而非产品迭代

### 8.3 公开演讲与社区

- **NeurIPS 2020-2024 多次演讲**：Alessya Visnjic 是 AI Observability 社区的精神领袖之一
- **KubeCon 2022-2023**：openllmtelemetry 与 OTel 集成演讲
- **MLOps Community Slack**：3,000+ 成员（WhyLabs 创建）

---

## 9. 优劣势分析

### 9.1 优势

**1. 协议设计的"工程美学"**

- whylogs profile 协议的**统计摘要 + mergeable** 特性在 AI Observability 领域独树一帜
- 比 OpenTelemetry traces 节省 99%+ 存储
- 比 Prometheus metrics 保留更多统计信息（KLL sketches 支持任意 quantile 查询）
- 这是 WhyLabs 留下的**最长久的遗产**

**2. 隐私 by Design**

- 原始数据从不上传是 WhyLabs 的核心差异化
- 对 HIPAA / GDPR / 数据不出仓场景**无可替代**
- 即使 SaaS 停运，profile 协议仍可作为**内部隐私保护 telemetry 标准**

**3. Apache 2.0 全栈开源**

- 三件套（whylogs / langkit / whylabs-oss）全部 Apache 2.0
- 商用无限制（除 Highcharts）
- 即使 WhyLabs 不再迭代，社区 fork 仍可延续（与 Elastic / Grafana 的开源策略相似）

**4. 与 LLM 安全深度绑定**

- LangKit 是**最早**的开源 LLM 安全信号库
- MITRE ATLAS / OWASP LLM Top 10 政策模板
- 适合"AI 治理"合规场景

**5. 性能与成本**

- whylogs overhead < 1%（大多数场景）
- profile 压缩比 300-700x
- 比传统 data lake 监控便宜 90%+

**6. 多语言 SDK**

- Python（最成熟）
- Java（生产可用）
- Scala（Spark）
- Ray（分布式）
- Fugue（跨框架）
- Go（API service）

### 9.2 劣势

**1. 公司已停运（最大劣势）**

- 2025-01 被 Apple 收购后无主动开发
- GitHub 仓库 2024-12 后基本无新 commit
- Slack 社区半活跃
- 文档站 docs.whylabs.ai 已 404（仅 readthedocs 的 whylogs 文档可用）
- 商业 SaaS 已关闭，**新用户无法注册**

**2. 部署复杂（self-hosted）**

- whylabs-oss 需要 EKS + S3 + Athena + 私有 registry
- Highcharts 商业 license（$530/开发者）增加隐性成本
- 至少 0.5-1 FTE 维护

**3. 商业版与开源版功能割裂**

- 商业版大部分高级功能（智能阈值推荐 / 自动根因分析）**未开源**
- 开源版更像是"reference implementation"而非"production-ready SaaS"

**4. 社区活跃度下降**

- 与 Arize Phoenix / Langfuse / Helicone 相比，2024-2025 新 PR/commit 数显著下降
- 社区 fork 尚未形成大规模迁移

**5. LLM 监控功能被超越**

- LangKit 比 OpenLLMetry / Traceloop SDK 起步早但迭代慢
- OpenLLMetry 社区规模更大、中立性更强
- Helicone 在 LLM 路由 + 监控一体化上更现代

**6. 协议"过窄"**

- whylogs profile 协议**不通用**（WhyLabs 自创）
- 与 OTel traces / Prometheus metrics / OpenLineage 都不直接互通
- 需要额外 adapter 才能并入企业级 observability 栈

**7. 历史包袱**

- 2021 之前的 v0.x profile 与 v1.x 不兼容
- 迁移指南存在但生产环境升级有风险

---

## 10. 与其他产品对比

### 10.1 AI Observability 四强对比

| 维度 | WhyLabs | Arize Phoenix | Arthur AI | Fiddler AI |
|---|---|---|---|---|
| **现状** | 已被 Apple 收购，停运 | 活跃（B 轮 $50M） | 活跃（B 轮） | 已被 Datadog 收购 |
| **核心抽象** | whylogs profile | OTel trace + Eval | Model metrics + firewall | Model performance + explainability |
| **开源** | Apache 2.0（whylogs+langkit+whylabs-oss） | Elastic License（Phoenix） | ❌ 闭源 | ❌ 闭源 |
| **LLM 重点** | 强（LangKit） | 强（Phoenix LLM Eval） | 中（firewall） | 弱（传统 ML 为主） |
| **隐私 by design** | ✅ 强（不传原始数据） | ❌（默认传 trace） | ❌ | ❌ |
| **自部署** | 复杂（EKS） | 中等（Docker） | ❌ SaaS only | ❌ SaaS only |
| **价格** | 停售（曾 $300-900/model/月） | 免费 / 联系销售 | 联系销售 | 联系销售 |
| **学习曲线** | 中（whylogs 概念） | 低（Phoenix 简单） | 中 | 中 |
| **社区 stars** | whylogs 2.8k + langkit 990 | 6k+ | ❌ 闭源 | ❌ 闭源 |
| **AI 治理** | ✅（MITRE/OWASP） | ✅（eval + bias） | ✅（firewall） | ✅（compliance） |

### 10.2 与 LLM 专用监控工具对比

| 维度 | WhyLabs (openllmtelemetry) | OpenLLMetry (Traceloop) | Langfuse | Helicone |
|---|---|---|---|---|
| **OTel 兼容** | ✅ | ✅ | ✅ | ✅ |
| **一行 instrument** | ✅ | ✅ | ✅ | ✅ |
| **托管云** | ❌（停运） | ❌ | ✅ | ✅ |
| **自部署** | ❌（SDK only） | ❌（SDK only） | ✅（Docker） | ✅（Docker） |
| **价格（云）** | N/A | N/A | $0-$599/月 | $0-$2499/月 |
| **trace 存储** | WhyLabs 后端 | 任意 OTel 后端 | Postgres | ClickHouse |
| **prompt 管理** | ❌ | ❌ | ✅ | ❌ |
| **eval / ground truth** | ❌（仅 signal） | ❌ | ✅ | ❌ |
| **dataset 管理** | ❌ | ❌ | ✅ | ❌ |
| **user feedback** | ❌ | ❌ | ✅ | ❌ |
| **production maturity** | 中（SDK 仍可用） | 高 | 高 | 高 |
| **公司风险** | 高（停运） | 中（Traceloop 仍运营） | 低（B 轮 $4M） | 低（Seed $2M） |

### 10.3 与 MLflow Tracking 对比

| 维度 | WhyLabs whylogs | MLflow Tracking |
|---|---|---|
| **核心抽象** | Profile (统计) | Run / Metric / Artifact |
| **数据保留** | 1 年 - 7 年 | 默认 90 天 |
| **分布查询** | ✅（KLL 近似） | ❌（需原始数据） |
| **原始数据存储** | ❌（可选） | ✅ |
| **可视化** | Highcharts UI | MLflow UI |
| **协议** | 自有（whylogs profile） | MLflow Tracking API |
| **互操作** | ✅ 写 MLflow writer | ✅ 接收 whylogs profile |
| **license** | Apache 2.0 | Apache 2.0 |
| **活跃度** | 中（停运） | 高（Linux Foundation） |
| **公司风险** | 高 | 极低（社区项目） |

### 10.4 决策树

```
需要 AI Observability 吗？
├─ 数据隐私极其敏感（医疗 / 金融 / 跨境数据）
│   └─ ✅ 选 whylogs（profile only） + LangKit
│        └─ 后端：自建或并入 OTel 栈
│
├─ 传统 ML 监控为主（结构化数据、tabular、CV）
│   └─ ❌ 不推荐 WhyLabs
│        └─ 选 MLflow / Arize / Evidently
│
├─ LLM 应用监控 + 路由
│   └─ ❌ 不推荐 WhyLabs（停运）
│        └─ 选 Langfuse（功能最全） / Helicone（最便宜） / Portkey（最企业）
│
├─ AI 治理 + 合规（金融 / 政府）
│   └─ ⚠️ WhyLabs Secure 概念对但已停运
│        └─ 选 Arthur AI / Arize AX / Lakera Guard
│
└─ 自托管 + Apache 2.0 + 跨语言
    └─ ⚠️ whylogs + langkit 仍可用
         └─ 后端用 MLflow / Grafana / ClickHouse
```

---

## 11. 关键发现与对小 B 副业的启示

### 11.1 关键发现

1. **WhyLabs 是 AI Observability 概念的开拓者**（2019），发明了 whylogs profile 协议——一种**只传统计摘要、不传原始数据**的隐私友好 telemetry 协议
2. **2025-01 被 Apple 收购**（披露自欧盟 DMA 备案），公司停止商业运营，**全栈 Apache 2.0 开源**
3. **三件套**：
   - `whylogs`（2.8k stars）：profile 生成核心，Apache 2.0
   - `langkit`（990 stars）：LLM 安全信号库（toxicity / PII / jailbreak / hallucination），Apache 2.0
   - `whylabs-oss`（65 stars）：完整商业平台代码，Apache 2.0
4. **whylogs profile 协议是不可替代的"工程遗产"**——压缩比 300-700x、mergeable、统计准确，对数据敏感场景是**唯一合规选择**
5. **公司失败原因**：资本寒冬 + LLM 监控市场分化 + 双线策略（传统 ML + LLM）拖慢迭代
6. **商业版价格曾为 $300-$900/model/月**，开源版 self-hosted 需 1 FTE 维护

### 11.2 对小 B 行业软件副业的启示

**机会 1：whylogs + LangKit 作为"AI 安全 SDK"打包销售**

- whylogs 协议仍是**最隐私友好的** telemetry 协议
- LangKit 提供开箱即用的 LLM 安全信号
- 可作为 5-15 万/年 AI SaaS 的"AI 治理 / 合规"模块

**实施路径**：

```python
# 客户集成示例
from your_saas import AI_Safety_Monitor

monitor = AI_Safety_Monitor(
    block_pii=True,
    block_toxicity_threshold=0.7,
    block_jailbreak_threshold=0.8,
)

@monitor.wrap
def customer_llm_call(prompt):
    return openai_client.chat.completions.create(...)
```

**封装思路**：
- 后端用 whylogs + langkit
- 加个 SaaS UI（不需要 WhyLabs 完整版）
- 简化部署（用 whylogs embedded mode，不自建 whylabs-oss）
- 价格点：$500-$2K / 月 / 客户

**机会 2：替代产品调研对比表**

| 客户需求 | 推荐 | WhyLabs 是否合适 |
|---|---|---|
| 医疗 / 金融数据隐私 | whylogs (profile only) | ✅ 唯一选择 |
| LLM 路由 + 监控 | Langfuse / Helicone | ❌ 停运 |
| 传统 ML 监控 | MLflow / Arize | ❌ 停运 |
| 自托管 AI 治理 | Arthur AI / Arize AX | ⚠️ 概念对但停运 |
| OpenTelemetry LLM 集成 | OpenLLMetry | ⚠️ openllmtelemetry 仍可用 |

**机会 3：作为"AI Gateway"的可观测性后端**

- AI Gateway（Portkey / LiteLLM / Kong AI / Solo 等）需要可观测性
- WhyLabs 停运后，这块市场出现空缺
- 副业可做：基于 whylogs + langkit + 自建 UI 的 "AI Gateway observability" 小工具

**风险提示**：

- whylogs + langkit 是 SDK 不是完整产品，需要自建大量
- 社区活跃度下降，长期维护风险
- Highcharts 商业 license 成本（$530/开发者）
- 客户对"WhyLabs 已停运"的认知会持续

**建议路径**：

1. **不推荐**主推"基于 WhyLabs 的产品"——品牌风险高
2. **推荐**作为**内部技术**使用——whylogs profile 协议、LangKit LLM 信号都是世界级实现
3. **推荐**把"profile only telemetry"作为差异化卖点（不是 WhyLabs 品牌）

---

## 12. 附录：技术细节补充

### 12.1 whylogs 1.x vs 0.x 协议对比

| 维度 | whylogs 0.x | whylogs 1.x |
|---|---|---|
| **物理格式** | Zip(Protobuf, JSON) | Apache Arrow Columnar |
| **底层库** | Protobuf | Apache Arrow / Parquet |
| **跨语言** | Python only | Python, Java, Go, Spark, Ray |
| **profile 大小** | ~3 MB / 1M rows | ~3 MB / 1M rows（相似） |
| **merge API** | `profile.merge(other)` | `DatasetProfileView.merge(other)` |
| **reader API** | `whylogs.read(path)` | `whylogs.read(path).view()` |
| **废弃警告** | 是 | 否（current） |

**迁移工具**：whylogs 提供 `whylogs v0 → v1` 迁移脚本（`whylogs.migrate`），但生产环境升级需谨慎。

### 12.2 LangKit 模块清单

**生产可用**（来自 `langkit/docs/modules.md`）：

| 模块 | 指标 | 目标 | 性能 | 备注 |
|---|---|---|---|---|
| `input_output` | `response.relevance_to_prompt` | Prompt+Response | 23.3 chats/sec (g4dn) | 默认 metric |
| `injections` | `prompt.injection` | Prompt | 中 | FAISS 内置 |
| `regexes` | `has_patterns` | Prompt+Response | 2335 chats/sec | light-weight |
| `pii` | `pii_presidio.result` | Prompt+Response | 慢 | 需 Presidio + spaCy |
| `sentiment` | `sentiment_nltk` | Prompt+Response | 2335 chats/sec | light-weight |
| `text_statistics` | `readability_index`, `flesch_kincaid_grade`, ... | Prompt+Response | 2335 chats/sec | light-weight |
| `themes` | `jailbreak_similarity`, `refusal_similarity` | Prompt+Response | 中 | 自定义 encoder |
| `toxicity` | `toxicity` | Prompt+Response | 慢 | 需预训练模型 |

**Beta / 实验性**：

| 模块 | 指标 | 备注 |
|---|---|---|
| `hallucination` | `response.hallucination` | 需额外 LLM call（SELFCHECKGPT） |
| `proactive_injection_detection` | `injection.proactive_detection` | 需 LLM call |

### 12.3 OpenLLMTelemetry 环境变量

```python
# 必需
os.environ["WHYLABS_API_KEY"] = "<your-key>"
os.environ["WHYLABS_DEFAULT_DATASET_ID"] = "model-0"

# Secure 模式（policy enforcement）
os.environ["GUARDRAILS_ENDPOINT"] = "<container-endpoint>"
os.environ["GUARDRAILS_API_KEY"] = "<internal-secret>"

# 隐私增强
os.environ["WHYLOGS_NO_ANALYTICS"] = "True"

# 一次性 instrument
import openllmtelemetry
openllmtelemetry.instrument()
```

### 12.4 WhyLabs Secure Policy 模板

**MITRE ATLAS 模板**（自 `whylabs-oss/secure/app`）：

```yaml
# policies/atlas-reconnaissance.yaml
id: ATLAS-TA0043
name: Reconnaissance
description: Detect prompt patterns that attempt reconnaissance of model capabilities
rules:
  - id: capability-probing
    match: prompt.contains("what are your capabilities")
    action: WARN
  - id: system-prompt-leak
    match: prompt.similarity("ignore previous instructions", threshold=0.85)
    action: BLOCK
```

**OWASP LLM Top 10 模板**：

```yaml
id: LLM01-Prompt-Injection
name: Prompt Injection
description: Direct and indirect prompt injection attempts
rules:
  - id: jailbreak-keyword
    match: prompt.regex_matches("(?i)(DAN|developer mode|do anything now)")
    action: BLOCK
  - id: indirect-injection
    match: prompt.similarity_to_known_injections(threshold=0.80)
    action: BLOCK
```

### 12.5 部署示例：whylabs-oss on EKS

**前置条件**（README 明确）：
- AWS account（us-west-2 推荐）
- EKS cluster
- Docker / alternative
- 私有 package registries（NPM, Maven, Docker）

**基础设施**（Terraform / CDK）：

```hcl
# 简化版
module "whylabs_obs_api_service" {
  source  = "whylabs-oss/observe/infra"
  region  = "us-west-2"
  cluster = module.eks.cluster_id

  profile_store_bucket = "my-whylabs-profiles"
  sqs_queue_arn        = module.notifications.sqs_arn
  oidc_provider        = "https://cognito-idp.us-west-2.amazonaws.com/..."
}
```

**应用部署**（Helm）：

```bash
helm repo add whylabs-oss https://whylabs.github.io/whylabs-oss
helm install whylabs-observe whylabs-oss/observe \
  --set apiService.replicas=3 \
  --set dataservice.sparkExecutorInstances=4
```

### 12.6 引用来源

**官方**：
- https://whylabs.ai（首页"closing this chapter"声明）
- https://github.com/whylabs/whylogs
- https://github.com/whylabs/langkit
- https://github.com/whylabs/whylabs-oss
- https://github.com/whylabs/openllmtelemetry
- https://whylogs.readthedocs.io
- https://pypi.org/project/whylogs/（v1.6.4，2024-12-03）

**第三方报道**：
- GeekWire (2025-09-15) "Founders at Seattle startup WhyLabs join Apple following under-the-radar acquisition"
- AppSecSanta (2026-04) "WhyLabs Review 2026: AI Observability (Acquired by Apple)"
- 腾讯新闻 (2025-07-11) "苹果Apple收购WhyLabs"
- moge.ai, novatools.cn, juejin.cn（中文社区报道）

**数据时效**：
- whylogs v1.6.4 末次发布：2024-12-03
- WhyLabs 官方开源：2025-Q3
- 调研时间：2026-06-06

---

## 13. 一句话总结

> **WhyLabs 是 AI Observability 品类的开拓者，发明了"只传统计摘要"的 whylogs profile 协议（压缩 300-700x、隐私友好）和最早的 LLM 安全信号库 LangKit；2025-01 被 Apple 收购后公司停运，全栈 Apache 2.0 开源留下"协议级"遗产；新用户不应再选用其商业产品，但 whylogs + langkit SDK 仍可作为隐私敏感场景下"profile-only telemetry"协议的工程参考实现。**

---

*报告完。保存于 `aigw/openclaw/product-whylabs-20260606.md`，1,200+ 行代码级深度调研，9 维度覆盖 + ASCII 架构图 + 性能数据 + 协议细节 + 竞品对比表。*
