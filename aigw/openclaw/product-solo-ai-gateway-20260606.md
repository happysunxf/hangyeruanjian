# Solo.io agentgateway（前 Gloo AI Gateway）深度调研报告

> 调研日期：2026-06-06
> 调研对象：Solo.io 主推的 AI/Agent 原生网关 **agentgateway**（含开源版 + Solo Enterprise 商业版），以及它的"前身" **Gloo AI Gateway** / **Gloo Gateway**
> 调研定位：单产品深挖，覆盖背景、架构、协议、性能、部署、成本、生态、案例、优劣势、对比 10 个维度
> 报告版本：v1.0
> 一句话总结：**agentgateway = Rust 写、原 AAIF（Linux Foundation 子基金会）托管、把 MCP / A2A / LLM / 传统 API 流量统一在一个数据平面里跑的网关；2026-06-04 才刚加入 AAIF，2026-06-05 才发布官方"设计哲学"博客，是目前 AI Gateway 赛道里"叙事最新、节奏最密"的产品之一**。

---

## 0. TL;DR

| 维度 | agentgateway（含 Solo Enterprise） | Envoy AI Gateway | Higress | Portkey | LiteLLM | Kong AI Gateway |
|------|-------------------------------------|------------------|---------|---------|---------|-----------------|
| 主体语言 | **Rust**（Tokio + Hyper + Tonic） | Go（控制面）+ Envoy（数据面） | Go + Envoy（WASM） | Node.js / TS | Python | Go + Lua/OpenResty |
| 协议广度 | HTTP / gRPC / TCP + **MCP + A2A + LLM** | HTTP + LLM | HTTP + LLM | HTTP + LLM | HTTP + LLM | HTTP + LLM |
| MCP / A2A 支持 | **原生一等公民** | 仅 HTTP 转发 + 社区扩展 | 仅 HTTP 转发 | 仅 HTTP 转发 | 仅 HTTP 转发 | 仅 HTTP 转发 |
| 性能（其官方基准） | 500k QPS / P99 < 0.2ms @ 30k QPS | 未公开 | 1.5k RPS（社区实测） | 5k RPS | SDK 层 ~0 开销 | ~3-5k RPS |
| 治理归属 | **AAIF（Linux Foundation）** | CNCF（Envoy 社区） | Apache 2.0（阿里） | Apache 2.0（Portkey） | MIT（BerriAI） | Apache 2.0（Kong） |
| 商业模式 | 开源 Apache 2.0 + Solo Enterprise（订阅） | 全开源 + 商业支持 | 全开源 + 商业版 | 开源 + SaaS | 开源 + 企业版 | 开源 + 企业版 |
| 主要客户 | Microsoft、Apple、Adobe、Amdocs、T-Mobile、Expedia、CoreWeave、Akamai、Dell、Salesforce、Red Hat、Solo.io 自家 | Solo.io 用户、AWS、Linux 基金会 | 阿里集团、字节、众多国内 SaaS | CleverTap、Autodesk、Waters、Travelgate | OpenAI 客户群（间接）、Uber、Mailchimp | 大量金融/政府/制造业 |
| 定位 | 面向 **Agentic AI 时代**的统一网关（AI + 非 AI 同数据面） | 面向 **K8s Gateway API** 的 AI 扩展 | 面向 **国内云原生 + 通义生态** 的多模网关 | 面向 **企业级 LLM 应用** 的可观测+路由 | 面向 **Python 应用** 的 SDK 兼容层 | 面向 **企业合规 + 既有 Kong 用户** 的 AI 插件 |
| 关键差异 | **Rust 性能 + MCP/A2A 一等公民 + 中立基金会治理** | K8s 集成最标准，CNCF 旗手 | 国内云厂商绑定 + WASM 插件 | 体验最好、可观测最全 | 协议兼容最广 | 老牌 API 网关，AI 插件层 |

**一句话总结**：agentgateway 押注的是"**Agent 时代需要的不只是 LLM 代理，而是 AI + 非 AI 流量统一 + 协议感知的网关**"——这一点和 Portkey（偏可观测）、LiteLLM（偏 SDK 兼容）、Kong（偏传统 API 网关）形成显著差异。它的关键风险是：**用户基数仍小、文档相对简略、商业版和企业生态还在建立**。

---

## 1. 项目背景

### 1.1 公司：Solo.io

Solo.io 成立于 **2017 年**（总部美国，CTO 是著名的 Christian Posta，Idit Levine 是 CEO/创始人），最初是 Istio 商业化的主要厂商之一，旗下 Gloo 系列是 Istio 生态里最早做"API Gateway 化"的产品之一。它在 2022-2024 年是 Istio Ambient Mesh（ztunnel）的核心贡献者。

- 团队规模：约 200-300 人（公开数据为 2024 年 Glassdoor 估算，未官方公开）
- 总部：Lexington, MA, USA
- 核心人物：
  - **Idit Levine**（Founder & CEO）— Service Mesh 圈 KOL
  - **Christian Posta**（Field CTO）— 多本 Service Mesh 书籍作者
  - **Lin Sun**（Head of Open Source）— Istio TOC 成员、agentgateway 项目核心贡献者
  - **Pete Muir**（PM Lead，agentgateway 商业版）
  - **Sam Heilbron**（agentgateway 商业版 PM）

### 1.2 产品演进时间线

| 时间 | 事件 | 备注 |
|------|------|------|
| 2017 | Solo.io 成立 | 服务网格/API 网关公司 |
| 2018-2019 | **Gloo Edge** 发布 | 基于 Envoy 的 API 网关，企业 API 路由 |
| 2020 | **Gloo Gateway**（开源） | Kubernetes 原生 Gateway |
| 2021 | **Gloo Mesh**（服务网格） | 多集群 Istio 管理 |
| 2023 | **Gloo AI Gateway**（第一代） | 在 Gloo Gateway 之上加 LLM 路由、token 限流、模型 fallback |
| 2024 | **Gloo AI Gateway** GA 1.0 | 商业版 + 开源版，支持 OpenAI / Anthropic / Bedrock 等 |
| 2025-03 | **agentgateway** 项目创建 | Solo.io 决定另起炉灶，Rust 重写，专注 Agent 场景 |
| 2025-08-25 | agentgateway 捐赠给 **Linux Foundation** | 第一阶段中立治理 |
| 2026-04-08 | 提交加入 **AAIF**（Agentic AI Foundation）提案 | 寻找更对口的中立基金会 |
| 2026-04-15 | **Solo Enterprise for agentgateway 2.3** GA | 首个企业版 |
| 2026-05-13 | AAIF TC 批准 | |
| 2026-05-21 | AAIF GB 批准 | Growth-stage 项目 |
| **2026-06-04** | **agentgateway 正式加入 AAIF** | **官方公告**（"Agentgateway Joins AAIF as an Open Gateway for Agentic AI Infrastructure"） |
| **2026-06-05** | **Designing agentgateway** 设计哲学博客发布 | Lin Sun 主笔，最权威架构解释 |
| 2026-05（同期） | **Solo Enterprise 2026.5.x** | 最新季度稳定版（docs.solo.io 显示 2026.5.2） |

### 1.3 项目性质变化

- **Gloo AI Gateway**：传统 Envoy-based AI 网关，偏 LLM 路由/fallback
- **agentgateway**：Rust 重写，AI-native 协议（MCP、A2A）一等公民，与 LLM 流量统一在**同一数据面**处理
- **Gloo Gateway**：继续维护，作为 K8s API 网关使用（基于 Envoy）

> **关键认知**：2025-2026 年 Solo.io 的战略重心从 "Gloo AI Gateway" 全面转向 "agentgateway"，两者的产品定位**有重叠但不完全等价**——agentgateway 是更"激进"的下一代设计，Gloo AI Gateway 则是更"稳"的传统路径。本报告以 **agentgateway** 为主线，附带 Gloo AI Gateway 对比。

### 1.4 AAIF 与 Linux Foundation 背景

**AAIF**（Agentic AI Infrastructure Foundation）是 Linux Foundation 下的子基金会，专注于 Agentic AI 基础设施的开源治理。已知托管项目：

1. **MCP**（Model Context Protocol）— Anthropic 主导的事实标准协议
2. **Goose** — Block 开源的 AI agent 编程框架
3. **agentregistry** — Agent/MCP 服务发现注册中心
4. **agentgateway**（2026-06-04 加入）— 第四个

AAIF 联合创始成员：Microsoft、Anthropic、Block、AWS、Cloudflare 等。

### 1.5 战略意义

Solo.io 之所以把 agentgateway 捐给 AAIF，而不是继续留在 CNCF 或自营：
- **目标受众匹配**：Agent 场景的厂商大多关注 AI 生态（Anthropic、OpenAI、Block）而非 K8s 生态（CNCF）
- **避免 vendor lock-in**：和 Envoy AI Gateway、Kong、Higress 等竞品形成"中立"差异化
- **承接 MCP 红利**：MCP 已是 AAIF 托管，agentgateway 在网关层做"安全+可观测+治理"，形成完整链路

---

## 2. 架构设计

### 2.1 整体架构图（ASCII）

```
┌────────────────────────────────────────────────────────────────────┐
│                      agentgateway 部署形态                          │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  agentgateway│  │  agentgateway│  │  agentgateway│  (xN)       │
│  │  Standalone  │  │     K8s      │  │  Docker      │             │
│  │  (二进制)    │  │  (Gateway API)│  │              │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                 │                       │
│         │ xDS / static   │ K8s Gateway API│  static config       │
│         │ config         │ CRDs            │  YAML                 │
│         └────────────────┴────────────────┘                       │
│                            │                                      │
│                            ▼                                      │
│                  ┌──────────────────────┐                         │
│                  │   xDS Control Plane  │  (Solo Enterprise       │
│                  │   (动态配置下发)     │   提供企业级控制面)      │
│                  └──────────────────────┘                         │
│                            │                                      │
│                            ▼                                      │
│         ┌─────────────────────────────────────┐                   │
│         │  Rust Data Plane (Tokio + Hyper)    │                   │
│         │  ┌─────────────────────────────┐    │                   │
│         │  │ Listener → Router → Filter  │    │                   │
│         │  │  chain → Backend (LLM/MCP/  │    │                   │
│         │  │   A2A/HTTP/gRPC/TCP)        │    │                   │
│         │  └─────────────────────────────┘    │                   │
│         │  ┌───── 内建能力 ─────────────┐    │                   │
│         │  │ • JWT/OAuth/mTLS/API key   │    │                   │
│         │  │ • CEL 策略引擎             │    │                   │
│         │  │ • OPA 外部授权             │    │                   │
│         │  │ • Token 限流 / Budget 控   │    │                   │
│         │  │ • OBO Token Exchange       │    │                   │
│         │  │ • OpenTelemetry 出口       │    │                   │
│         │  └────────────────────────────┘    │                   │
│         └──────────────┬──────────────────────┘                   │
│                        │                                          │
└────────────────────────┼──────────────────────────────────────────┘
                         │
        ┌────────────────┼─────────────────┐
        │                │                 │
        ▼                ▼                 ▼
  ┌──────────┐    ┌──────────┐    ┌──────────────┐
  │  LLM 后端 │   │ MCP 服务器│   │ A2A Agent 集群│
  │ OpenAI   │   │ 内置/外部 │   │ LangChain    │
  │ Anthropic│   │ (stdio/   │   │ CrewAI       │
  │ Bedrock  │   │  HTTP/    │   │ ADK          │
  │ Gemini   │   │  SSE/     │   │ 自家 Runtime │
  │ Vertex   │   │ Streamable│   │              │
  │ vLLM/TGI │   │  HTTP)    │   │              │
  │ 自托管池 │   │           │   │              │
  └──────────┘    └──────────┘    └──────────────┘
```

### 2.2 数据面：Rust + Tokio + Hyper + Tonic

设计哲学来自 Lin Sun 2026-06-05 博客：

> "We built agentgateway in Rust because performance and memory safety are non-negotiable for this kind of system."
> — Lin Sun, Head of Open Source, Solo.io

技术栈选型：
- **Tokio**：异步运行时（与 ztunnel 共用经验）
- **Hyper**：HTTP/1.1 + HTTP/2 实现
- **Tonic**：gRPC 框架（用于 xDS）
- **cel-rust**：Common Expression Language 解析器（用于动态策略）
- **wg / kube-rs**：可选的 K8s Gateway API CRD 客户端

和 Envoy 的关系：**不是 Envoy 的 wrapper 或 filter**——agentgateway 是一套独立的数据面，**借鉴**了 Envoy 的多协议代理设计思想（xDS、Listener/Route/Cluster 三段式）但**完全重写**。Solo.io 拥有 Istio ambient（ztunnel）的 Rust 经验，是这次"再写一个 Rust 代理"的基础。

### 2.3 控制面：Solo Enterprise 才有，开源版用静态 YAML

| 部署模式 | 控制面 | 数据面 | 适用场景 |
|----------|--------|--------|----------|
| **Standalone**（开源） | 无（静态 YAML） | Rust 二进制 | 开发/小规模自托管 |
| **Kubernetes 开源** | K8s Gateway API 标准 CRD（GatewayClass / Gateway / HTTPRoute / TCPRoute 等） | Rust DaemonSet/Deployment | 任何 K8s 用户 |
| **Kubernetes 商业** | **Solo Enterprise CRD 扩展**（EnterpriseAgentgatewayBackend 等） | 同上 | 大型企业，需要企业级控制面 |
| **Solo Cloud** | Solo.io 托管控制面 + 客户数据面 | 客户侧数据面 | 不想运维控制面的客户 |

Solo Enterprise 的 CRD 扩展代表：
- `EnterpriseAgentgatewayBackend`：支持 inline OpenAPI schema 直接暴露为 MCP 工具（2.3 新增）
- `AgentgatewayPolicy`：企业级策略（OBO Token、声明式授权）
- `AgentgatewayTelemetry`：OpenTelemetry 出口策略

### 2.4 协议分层处理

```
                ┌─────────────────────────────────┐
   Request  →   │  Listener (TLS/HTTP detection)  │
                │  ↓                              │
                │  Router (path/method/header)    │
                │  ↓                              │
                │  Filter chain:                   │
                │   • Auth (JWT/Key/OAuth)        │
                │   • Rate limit / Budget         │
                │   • CEL policy                  │
                │   • Prompt guard                │
                │   • PII shield                  │
                │   • OBO token exchange          │
                │  ↓                              │
                │  Backend protocol translator:    │
                │   • OpenAI-compatible ↔ Anthropic│
                │   • OpenAPI → MCP tool          │
                │   • stdio/HTTP/SSE/Streamable   │
                │  ↓                              │
                │  Upstream (LLM / MCP / A2A /   │
                │   HTTP / gRPC / TCP)            │
                └─────────────────────────────────┘
                          ↓
                       Response
                          ↓
                • Token usage 计量
                • TTFT / TTLB 记录
                • OpenTelemetry 埋点
                • CEL 表达式的 deny 拦截
                • Stream 透明转发
```

### 2.5 关键架构特性

**1. 协议感知 vs. 协议透明**

传统 API 网关是"协议透明"的——HTTP 进 HTTP 出。agentgateway 强调"**协议感知**"：

- **MCP 感知**：能解析 MCP 协议的 `tools/list`、`tools/call`、`resources/read`，对工具调用做**单独的 RBAC、限流、审计**
- **A2A 感知**：理解 Agent Card、Task、Artifact 等 A2A 概念，做 capability discovery
- **LLM 感知**：能识别 prompt/completion 边界，按 token 而非 request 限流，按 model 名而非 host 路由

**2. 单二进制 = 单数据面**

> "Agentgateway was designed as a unified gateway control plane and proxy data plane that can handle HTTP, gRPC, MCP, A2A, and LLM traffic together through the same operational surface."
> — Lin Sun, 2026-06-05

这一点是和 Envoy AI Gateway 最大的区别：
- **Envoy AI Gateway**：用 Envoy 作为数据面，AI 路由是 Envoy 之上的扩展 filter
- **agentgateway**：**完全重写**了一个数据面来同时处理 AI + 非 AI

好处：避免"AI 流量走 AI 网关 + 传统流量走 API 网关"的双重运维。坏处：要维护两个生态（agentgateway 自己 vs. Envoy 社区）。

**3. 渐进式发现（Progressive Disclosure）**

2026-04-22 博客《Keeping Context and Tokens Low With Progressive Disclosure In Agentgateway》介绍了 MCP 工具的"按需描述"能力——MCP 工具太多时，全部塞进 LLM context 会爆 token。agentgateway 用 MCP 协议的 `resources/list` + `resources/read` 模式让 agent 按需加载工具元信息，可减少 91% 的 MCP token 消耗。

**4. OpenAPI → MCP 桥**

2026-05-13 博客《Agentgateway Code Mode for OpenAPI to MCP》介绍了三种暴露方式：
- **Direct exposure**：OpenAPI operation 直接映射为 MCP tool
- **Custom exposure + API chaining**：自定义映射 + 串联调用
- **Code mode**：让 LLM 生成代码来编排多个 API（最大灵活度，但需要沙箱）

---

## 3. 协议支持

### 3.1 完整支持矩阵

| 协议 | 入口支持 | 出口支持 | 备注 |
|------|----------|----------|------|
| **HTTP/1.1** | ✅ | ✅ | Auto-TLS 协议嗅探（2.3 新增） |
| **HTTP/2** | ✅ | ✅ | 包含 gRPC |
| **HTTP/3 (QUIC)** | ⚠️ 计划中 | ⚠️ 计划中 | 路线图 |
| **gRPC** | ✅ | ✅ | Tonic 实现 |
| **TCP** | ✅（L4 透传） | ✅ | 简单四层 |
| **TLS 1.3** | ✅ | ✅ | mTLS 轮换内置 |
| **OpenAI Chat Completions** | ✅ | ✅ | 事实标准 |
| **OpenAI Responses API** | ✅ | ✅ | 2025+ 新标准 |
| **OpenAI Assistants** | ⚠️ 弃用 | ⚠️ 弃用 | OpenAI 已弃用，agentgateway 不重点支持 |
| **Anthropic Messages** | ✅ | ✅ | Anthropic 原生协议 |
| **Anthropic /v1/messages 兼容层** | ✅ | ✅ | 通过 OpenAI 兼容 API 桥接 |
| **Google Gemini** | ✅ | ✅ | |
| **AWS Bedrock** | ✅ | ✅ | **2.3 加入 AgentCore 深度支持** |
| **Azure OpenAI** | ✅ | ✅ | |
| **Vertex AI** | ✅ | ✅ | |
| **Cohere** | ✅ | ✅ | |
| **Hugging Face Inference Endpoints** | ✅ | ✅ | |
| **vLLM / TGI / Triton（自托管）** | ✅ | ✅ | Inference Gateway 模式 |
| **MCP（stdio）** | ✅ | ✅ | 本地进程 |
| **MCP（HTTP/SSE）** | ✅ | ✅ | 旧版 |
| **MCP（Streamable HTTP）** | ✅ | ✅ | 2025 主流 |
| **MCP（Stateless）** | ⚠️ 进行中 | ⚠️ 进行中 | 路线图 |
| **A2A（Agent-to-Agent）** | ✅ | ✅ | Google 主导 |
| **OpenAPI → MCP** | N/A | ✅ | 2.3 新增（EnterpriseAgentgatewayBackend CRD） |
| **OpenTelemetry（OTLP）** | N/A | ✅ | 出口可观测 |
| **OAuth 2.0 / 2.1** | ✅ | ✅ | MCP 接入有专门优化 |
| **JWT / OIDC** | ✅ | ✅ | 含 claim 转换 |
| **RFC 8693 OBO Token Exchange** | ✅ | ✅ | 2.3 新增，AWS Bedrock AgentCore 场景 |
| **MCP OAuth 2.1** | ✅ | ✅ | 2026 第一个完整支持 MCP OAuth 的网关 |

### 3.2 MCP 协议处理详解

MCP（Model Context Protocol）有四种传输方式，agentgateway 全部支持：

```
┌──────────────────────────────────────────────────────────┐
│ MCP Transport Support in agentgateway                     │
├────────────────────┬─────────────────────────────────────┤
│ stdio              │ 本地 MCP server（agentgateway 起进程│
│                    │ 与之通信）；适合开发和小规模          │
├────────────────────┼─────────────────────────────────────┤
│ HTTP+SSE (legacy)  │ 已支持，但官方推荐迁移到 Streamable  │
│                    │ HTTP                                │
├────────────────────┼─────────────────────────────────────┤
│ Streamable HTTP    │ 当前推荐；agentgateway 内置完整     │
│                    │ client 和 server 模式              │
├────────────────────┼─────────────────────────────────────┤
│ Stateless MCP      │ 路线图中（2026 中）                │
└────────────────────┴─────────────────────────────────────┘
```

**关键能力**：
- **MCP 联邦（MCP Federation）**：把多个 MCP server 聚合成一个逻辑端点
- **MCP 虚拟化**：客户端只需对接一个虚拟 MCP endpoint，agentgateway 内部路由到真实 server
- **MCP 沙箱**：shadow MCP access（servers 动态请求 backend URL）由 agentgateway 验证+沙箱
- **MCP OAuth 2.1**：解决 desktop agent 频繁弹 OAuth 窗的问题（在 connect time 完成）
- **MCP 工具级 RBAC**：基于 CEL 表达式控制 agent 可调用的 tool 子集
- **MCP 审计**：每个 tool call 都有完整调用链、用户身份、token 消耗审计
- **Progressive Disclosure**：MCP 工具描述按需加载（节省 91% token）

### 3.3 A2A 协议处理

A2A（Agent-to-Agent）是 Google 在 2025 年主导推出的 agent 互操作协议。agentgateway 是**第一个原生支持 A2A 路由的网关**：

- **Agent Card 发现**：通过 `.well-known/agent.json` 自动发现其他 agent 的能力
- **Modality 协商**：理解不同 agent 支持的 input/output 模态（text/image/audio）
- **Task 协作**：支持长任务（long-running task）跨 agent 传递
- **Identity 透传**：保持 agent 调用链中的身份可追溯
- **Framework 兼容**：LangChain、CrewAI、ADK（Google Agent Development Kit）、自家 runtime 都能接入

### 3.4 LLM 协议处理

agentgateway 在 LLM 协议层做的不只是"协议兼容"——它在每层都有专门处理：

**Token 级别**：
- 按 model + user/team/api-key 三维度限流
- 实时预算控制（Budget enforcement），可"denial-of-wallet"防爆
- Prompt cache token 类型（`input_cache_read` / `input_cache_write`）单独计量（2.3 新增）

**流式响应（SSE）**：
- 完全透明转发，不破坏流
- 在第一个 token（TTFT）和最后一个 token（TTLB）打点 OpenTelemetry
- 流式 cancel 能正确关闭上游连接

**模型路由**：
- Cost-aware：自动选最便宜的够用模型
- Latency-aware：自动选当前 P99 最低的 region
- Reliability-aware：基于 CEL 表达式的健康检查，evict 失败 provider
- Context-aware：按请求头/JWT claim 路由到 fine-tuned 变体

**Failover**（2.3 新增）：
- CEL eviction policy 表达能力强
- 支持 5xx / 429 / 自定义错误码触发切换
- 主 provider 恢复后自动切回

---

## 4. 性能数据

### 4.1 官方公开基准（2026-06-05 博客披露）

| 指标 | agentgateway | 对比对象（"interpreted alternatives"，未指明） | 优势倍数 |
|------|--------------|--------------------------------------------|----------|
| **内存占用** | 30 MB | 9 GB | **300×** |
| **吞吐量** | 165k QPS | 4.6k QPS | **35×** |
| **P99 延迟** | 0.09 ms | 11 ms | **122×** |
| **路由传播** | 实时（xDS 推送） | 需重启 | 显著 |

> **注意**：上述对比对象具体是哪个产品，官方未明确说明。从数量级推断（4.6k QPS / 9GB 内存 / 11ms P99）来看，**很可能是 Envoy** 或 **Kong**。但这一基准是 Solo.io 自测，且是在非常特定的 HTTP echo 场景下，不完全代表生产 LLM 工作负载。

### 4.2 第三方基准（gateway-api-bench v2）

John Howard 的 [gateway-api-bench v2](https://github.com/howardjohn/gateway-api-bench/blob/main/README-v2.md) 提供了独立测试数据：

| 测试 | 数字 | 条件 |
|------|------|------|
| Traffic performance | **~500k QPS** | 512 concurrent connections |
| P99 latency | **< 0.2 ms** | 30k QPS, 512 connections |
| HTTP/2 多路复用 | ✅ | |

> 这个测试是标准 HTTP echo，**不模拟** LLM 实际负载（流式、长连接、TTFT）。生产 LLM 流量下，性能主要由"上游 LLM provider 速度"决定，agentgateway 自身的开销应 < 1ms。

### 4.3 与其他 AI Gateway 性能对比（社区估算）

| 产品 | 单实例 RPS（HTTP echo） | LLM 场景 P50 额外开销 | 备注 |
|------|------------------------|----------------------|------|
| **agentgateway** | **165k-500k** | **< 1ms** | Rust |
| Envoy AI Gateway | 100k+ | < 5ms | C++ (Envoy) |
| Higress | 30k+ | < 10ms | Go + WASM |
| Kong AI Gateway | 10-15k | 5-15ms | OpenResty/Lua |
| APISIX ai-proxy | 20-30k | 5-10ms | Go + Lua |
| Portkey Gateway | 5k | 5-20ms | Node.js + Redis |
| LiteLLM（proxy mode） | 1-2k | 10-30ms | Python |
| One API / New API | 1-2k | 10-20ms | Go + GORM |
| Cloudflare Workers AI | 边缘 100k+ | 边缘 5-20ms | V8 isolate |

> **重要 caveat**：以上数字多为**社区或厂商自测**，测试条件不同（payload 大小、并发模型、是否带 streaming、有无 Redis 依赖），**不可直接横向比较**。真实 LLM 工作负载下，瓶颈通常在 LLM provider 而非网关。

### 4.4 资源占用

- **二进制大小**：约 30-50 MB（包含完整 Rust 运行时）
- **运行时内存**：30-100 MB 静态，活跃流量下增长可控
- **启动时间**：< 1 秒（Rust 单二进制）
- **冷启动**：适合 K8s 快速扩缩容场景

---

## 5. 部署方式

### 5.1 三种部署模式

#### 5.1.1 Standalone（单机/裸机）

适用：开发测试、小规模自托管、单实例服务

```bash
# 一行安装
curl -sL https://agentgateway.dev/install.sh | bash

# 准备配置
cat > config.yaml <<'EOF'
binds:
- port: 3000
listeners:
- routes:
  - policies:
      cors:
        allowOrigins: ["*"]
        allowHeaders: [content-type, cache-control]
    a2a: {}
    backends:
    - host: localhost:9999
EOF

# 启动
agentgateway -f config.yaml
# INFO agentgateway: Listening on 0.0.0.0:3000
# INFO agentgateway: Admin UI at http://localhost:15000/ui/
```

**特点**：
- 单二进制，零外部依赖（除 OTLP exporter）
- 自带管理 UI（端口 15000）
- 配置是静态 YAML
- 适合：本地开发、单租户小流量、嵌入式场景

#### 5.1.2 Docker

```bash
docker run -p 3000:3000 -p 15000:15000 \
  -v $(pwd)/config.yaml:/etc/agentgateway/config.yaml \
  ghcr.io/agentgateway/agentgateway:latest \
  -f /etc/agentgateway/config.yaml
```

镜像位置：`ghcr.io/agentgateway/agentgateway`（开源版）/ Solo.io 私有 registry（企业版）

#### 5.1.3 Kubernetes（推荐生产）

```bash
# 1. 装 K8s Gateway API 标准 CRD
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

# 2. 装 agentgateway CRD（Helm OCI）
helm upgrade -i agentgateway-crds \
  oci://cr.agentgateway.dev/charts/agentgateway-crds \
  --create-namespace --namespace agentgateway-system \
  --version v1.1.0 \
  --set controller.image.pullPolicy=Always

# 3. 装 agentgateway controller
helm upgrade -i agentgateway \
  oci://cr.agentgateway.dev/charts/agentgateway \
  --namespace agentgateway-system \
  --version v1.1.0 \
  --set controller.image.pullPolicy=Always \
  --set controller.extraEnv.KGW_ENABLE_GATEWAY_API_EXPERIMENTAL_FEATURES=true \
  --wait
```

**特点**：
- 完全 conformant K8s Gateway API
- 多个 GatewayClass 实现（不同配置 profile）
- 路由由 `HTTPRoute` / `GRPCRoute` / `TCPRoute` 标准资源描述
- xDS 实时下发到 data plane
- 支持 HorizontalPodAutoscaler、PodDisruptionBudget

#### 5.1.4 Solo Cloud（企业版 SaaS 化）

Solo.io 2025 起推的托管控制面。客户只需部署数据面，控制面在 Solo.io 运维。适合"不想养网关团队"的客户。

### 5.2 K8s 上的资源消耗

| 模式 | CPU（典型） | Memory（典型） | 副本数（默认） |
|------|------------|----------------|----------------|
| 轻量 LLM 代理 | 100m-500m | 128-256Mi | 2 |
| 中等（带 OTLP + RBAC） | 500m-1 | 256-512Mi | 3-5 |
| 重载（流式 + 多协议） | 1-2 | 512Mi-1Gi | 5-10 |
| 边缘节点 DaemonSet | 100m | 64Mi | 1/节点 |

### 5.3 多集群 / 多区域

agentgateway 在 K8s 上通过 GatewayClass 区分多集群，配置由 Solo Enterprise 的 Global Control Plane 同步。**和 Istio 多集群、Consul 类似**，但更轻量——因为是"配置同步"而非"服务网格数据面同步"。

---

## 6. 成本模型

### 6.1 开源版（Apache 2.0）

- **许可证**：Apache 2.0
- **价格**：免费
- **包含**：所有核心数据面功能、Standalone/K8s 部署、社区支持
- **不含**：Solo Enterprise CRD（OpenAPI→MCP、OBO Token Exchange 等）、商业级控制面、SLA、官方技术支持

### 6.2 Solo Enterprise for agentgateway

价格**未在官网公开**（需要 sales 对接），但根据 Solo.io 之前 Gloo Enterprise 的市场行情估算：

| 组件 | 估算价格（年付） | 备注 |
|------|----------------|------|
| Solo Enterprise for agentgateway（基础） | **$20k-50k/年/集群** | 含企业 CRD、RBAC、OBO |
| Solo Enterprise Premium Support | **+ $10-20k/年** | 24/7 工单、专属 TAM |
| Solo Cloud（托管控制面） | **按 RPS 或节点数定价** | 类似 Kong Konnect 模式 |
| Solo University / Training | **$5-10k/人/课程** | 实施培训 |

> Solo.io 的定价策略一向**面向中大企业**（按集群/节点/支持等级），**不针对小 B 用户**——这和其目标客户（Microsoft、Apple、Adobe、T-Mobile、Expedia、Akamai、Dell、Salesforce、CoreWeave、Red Hat、Amdocs）相符。

### 6.3 总拥有成本（TCO）对比

考虑一个**中等规模企业 AI 网关场景**（10 个 K8s 集群、月 1B tokens、3 个 LLM provider + 5 个 MCP server）：

| 方案 | 一次性集成成本 | 年运维 | 年总 TCO | 备注 |
|------|---------------|--------|---------|------|
| **agentgateway 开源** | 2-4 FTE-月 | 1-2 FTE-年 | $300-500k/年 | 需要 Rust/K8s 专家 |
| **agentgateway 企业** | 1-2 FTE-月 | 0.5 FTE-年 | $250-400k/年 | 含支持和培训 |
| **Portkey Cloud** | < 0.5 FTE-月 | 0.2 FTE-年 | $100-300k/年 | 托管，省运维 |
| **Higress 自托管** | 1-2 FTE-月 | 0.5-1 FTE-年 | $150-300k/年 | 国内生态优势 |
| **Envoy AI Gateway 自托管** | 3-5 FTE-月 | 1-2 FTE-年 | $400-700k/年 | 集成成本高 |
| **Kong AI Gateway 企业** | 2-3 FTE-月 | 1-2 FTE-年 | $400-800k/年 | 含 Kong Gateway 总价 |

> FTE 成本按 $200k/年（美国资深工程师）估算。

### 6.4 自托管 vs 托管（SaaS）决策

| 维度 | 自托管（开源） | Solo Cloud |
|------|--------------|------------|
| 控制力 | 高 | 中 |
| 运维负担 | 高 | 低 |
| 成本可控 | 高（仅人力） | 中（订阅） |
| 升级敏捷 | 慢 | 快 |
| 合规可控 | 高 | 中 |
| 适合 | 大企业、严格合规 | 中型企业、想快速上线 |

---

## 7. 生态

### 7.1 上下游生态

#### 上游 LLM 集成（10+）

- OpenAI（Chat Completions、Responses、Embeddings）
- Anthropic（Messages API）
- Google Gemini（原生 + Vertex AI）
- AWS Bedrock（含 AgentCore 2.3）
- Azure OpenAI
- Cohere
- Mistral（通过 OpenAI 兼容 API）
- Hugging Face Inference Endpoints（自托管）
- 自托管 vLLM / TGI / Triton 池
- 其他 OpenAI 兼容服务（Together、Fireworks、OpenRouter 等）

#### 下游 MCP 服务集成

- 内置 stdio 模式可启动任何 MCP server 进程
- HTTP/SSE/Streamable HTTP 模式可连接远程 MCP 服务
- OpenAPI → MCP 自动桥接（2.3 新增）
- 与社区 MCP server 目录（mcpservers.org、mcp.so）兼容

#### 上下游 Agent 框架

- LangChain / LangGraph
- CrewAI
- Google ADK（Agent Development Kit）
- LlamaIndex Agents
- AWS Bedrock AgentCore（**2.3 重点集成**）
- 自家 runtime

#### 上下游基础设施

- **可观测性**：OpenTelemetry OTLP → Datadog / Honeycomb / Grafana Tempo / Jaeger
- **IAM**：Keycloak / Okta / Auth0 / Azure AD（OIDC）
- **策略**：OPA / Cedar
- **CI/CD**：Argo CD / Flux（GitOps 标准）

### 7.2 Solo.io 内部生态

Solo.io 在 agentgateway 周围构建了一组**AI 基础设施全家桶**：

| 项目 | 用途 | 关系 |
|------|------|------|
| **agentgateway** | AI/Agent 网关 | 本报告主角 |
| **kagent** | K8s 上的 agent runtime | 上游：agent 部署在 K8s |
| **kgateway** | K8s API 网关 | 兄弟项目：传统 API 网关注 K8s Gateway API |
| **kmcp** | MCP 服务部署到 K8s | 配套：让 MCP server 像 K8s 服务一样部署 |
| **agentregistry** | Agent/MCP 服务发现 | 配套：联邦多个 MCP 源 |
| **agentevals** | Agent 评估 | 配套：CI 中的 agent 评测 |
| **Gloo Gateway** | 传统 K8s API 网关 | 兄弟项目（基于 Envoy） |
| **Gloo AI Gateway** | 第一代 AI 网关 | 前身（基于 Envoy） |
| **Ambient Mesh** | Istio ambient mesh | 旁系：Solo.io 在 Istio 的核心贡献 |
| **Gloo Mesh** | 多集群服务网格 | 旁系：Istio 多集群商业版 |

> 这种"全家桶"策略是 Solo.io 的传统——Gloo 系列一直走"一组覆盖 K8s 网关/网格/AI 网关/agent runtime"的路子。

### 7.3 标准与组织参与

- **AAIF**：agentgateway 是 hosted project（2026-06-04 加入）
- **Linux Foundation**：通过 AAIF 间接关联
- **CNCF**：Istio 是 CNCF 项目；agentgateway **和 Istio 互操作**（作为 Istio 的 data plane 选项之一，2026-03 Istio 公告）
- **MCP**：直接支持 MCP 规范，AAIF 内的协议合作
- **Gateway API**：完全 conformant K8s Gateway API 实现
- **OpenTelemetry**：原生 OTLP 出口
- **A2A**：Google A2A 协议直接支持
- **W3C / IETF**：JWT、OAuth 2.1、OIDC、RFC 8693 等标准

### 7.4 竞品社区关系

| 竞品 | 与 agentgateway 关系 | 差异点 |
|------|---------------------|--------|
| Envoy AI Gateway（Envoy 社区） | **互操作**——Istio 已采用 agentgateway 作为 data plane 选项之一 | 治理：AIVEN/CNCF vs AAIF |
| Higress（阿里） | 独立竞品 | 国内云厂商绑定 vs 全球中立基金会 |
| Kong AI Gateway | 独立竞品 | 老牌 API 网关，AI 是扩展 |
| Portkey | 独立竞品 | SaaS 为主，vs 自托管优先 |
| LiteLLM | 独立竞品 | Python SDK 层，vs 网关层 |
| Cloudflare Workers AI Gateway | 独立竞品 | 边缘节点，vs 自托管 |

### 7.5 增长数据（官方披露）

来自 Lin Sun 2026-06-05 博客：

- 周下载量：从 ~100,000 → 1,000,000+（10× 增长）
- 总下载量：700 万+
- 贡献者：300+ 来自 60+ 组织
- 核心贡献组织：CoreWeave、Red Hat、Solo.io、Adobe、Salesforce、Amdocs、Microsoft
- 大客户：Microsoft、Apple、Adobe、Amdocs、T-Mobile、Expedia、CoreWeave、Akamai、Dell、Salesforce

> **数据 caveat**：Solo.io 自家披露数据，第三方独立验证有限。真实生产部署数应该比这个低一个数量级。

---

## 8. 客户案例

### 8.1 Solo.io 自家使用

> "We also use agentgateway extensively at Solo.io to mediate both LLM and MCP traffic, giving us consistent security, governance, and observability across these systems."
> — Lin Sun, 2026-06-05

Solo.io 内部全面使用 agentgateway 做 LLM + MCP 流量的统一管理（自举 dogfooding）。

### 8.2 已披露客户（2026-06 之前）

| 客户 | 行业 | 场景 | 披露程度 |
|------|------|------|----------|
| **Microsoft** | 科技 | 内部 AI 基础设施 | 名字（赞助商+贡献者） |
| **Apple** | 消费电子 | 内部 AI 基础设施 | 名字（合作方） |
| **Adobe** | 软件 | 创意 AI 工具后端 | 名字（合作方） |
| **Amdocs** | 电信 | 客服 agent 路由 | 名字（合作方） |
| **T-Mobile** | 电信 | 内部 AI 工具 | 名字（合作方） |
| **Expedia** | 旅游 | 旅行 agent | 名字（合作方） |
| **CoreWeave** | 云/AI 算力 | GPU 池 + AI 网关 | 名字（赞助商+核心贡献者） |
| **Akamai** | CDN | 边缘 AI | 名字（赞助商） |
| **Dell** | 硬件 | OEM AI 解决方案 | 名字（赞助商，CTO 公开背书） |
| **Salesforce** | SaaS | Agentforce 后端 | 名字（核心贡献者） |
| **Red Hat** | Linux/容器 | OpenShift 集成 | 名字（核心贡献者） |

> **公开案例详细程度**：除名字外，Solo.io **几乎不公开具体技术细节、ROI 数据、流量规模**。这在 B2B 基础设施软件里是常见做法（出于客户保密）。相比之下，Portkey、Cloudflare、Helicone 有更具体的客户故事。

### 8.3 Dell CTO 公开背书（来自 agentgateway.dev 首页）

> "The future won't be built by standalone agents, MCP servers or LLMs — it's shaped by their interconnection and ability to work together seamlessly. To unlock their full potential, we must apply policies, ensure control and maintain clear visibility into their interactions. This is where agentgateway plays a pivotal role — bridging not only agent-to-agent (A2A) communication but also agent-to-MCP servers, filling a critical gap in the ecosystem."
> — Global CTO & Chief AI Officer, Dell

### 8.4 推断的目标场景

基于客户行业和已发布博客：

| 场景 | 客户类型 | agentgateway 卖点 |
|------|----------|------------------|
| **客服 agent 路由** | 电信、零售 | A2A 联邦 + MCP 工具治理 |
| **内部 AI 工具平台** | 大型企业 IT | MCP OAuth + 工具级 RBAC + 审计 |
| **AI 算力调度** | 云厂商 (CoreWeave) | Inference Gateway + GPU 利用率优化 |
| **OEM AI 解决方案** | Dell | 标准化、可嵌入到 Dell 设备 |
| **CDN + 边缘 AI** | Akamai | 边缘节点 + 中心控制面 |
| **SaaS Agent 后端** | Salesforce、Adobe | 多租户 + 模型路由 + 成本控制 |

---

## 9. 优劣势分析

### 9.1 核心优势

#### 1. **性能领先**（Rust + Tokio）

300× 内存优势、35× 吞吐量、122× 延迟优势是 Solo.io 反复宣传的卖点。Rust 单二进制确实在网关这种"流量密集型 + 低延迟要求"的场景下有结构性优势。

#### 2. **协议覆盖最完整**（AI + 非 AI 统一数据面）

唯一一个在**同一数据面**处理 HTTP / gRPC / TCP + LLM + MCP + A2A 的网关。其他竞品都是"传统网关 + AI 扩展"或"AI 网关 + 简单 HTTP 转发"。

#### 3. **MCP / A2A 一等公民**

MCP 联邦、Progressive Disclosure、Code Mode、A2A 路由、Stateless MCP 路线图——**MCP 治理的最完整实现**。Portkey、LiteLLM、Higress 都还没把 MCP 当成一等公民。

#### 4. **中立基金会治理**（AAIF）

加入 AAIF 后，agentgateway 是**目前唯一在 AI 专门基金会下托管的开源 AI 网关**：
- CNCF：Envoy AI Gateway、Istio
- Apache：Higress、APISIX、Kong
- AAIF：**agentgateway**（独占）

对于担心 vendor lock-in 的大客户（金融、政府），这层中立性是显著优势。

#### 5. **Solo.io 团队的执行力**

Lin Sun（agentgateway 主架构师）是 Istio TOC 成员、ztunnel Rust 代理的核心作者。Solo.io 团队在 Rust + 服务网格 + API 网关三个领域的经验是**行业稀缺**的。

#### 6. **AWS Bedrock AgentCore 深度集成**（2.3 差异化）

OBO Token Exchange 三种模式是 2026-04 才有的能力，**比竞品早 6-12 个月**解决"agent 替用户调用 + 用户身份如何审计"的合规问题。

#### 7. **Inference Gateway 集成 llm-d**

llm-d 是 Red Hat 主导的 vLLM 分布式推理框架。agentgateway 直接集成，**对自托管 GPU 池最友好**。

### 9.2 核心劣势

#### 1. **生产案例少，文档相对单薄**

对比数据：
- **Portkey** 公开 1000+ 企业客户、详细 case study
- **Cloudflare Workers AI** 公开流量数据、SLA
- **agentgateway** 公开的 case study 几乎都是"赞助商 + 名字"

对于一个 2025-03 才创建的 Rust 项目来说，**生产验证周期还不够长**。大型企业（金融、医疗、政府）通常需要 2-3 年的生产观察才会采纳。

#### 2. **Solo.io 商业版定价不透明**

Kong、Apigee、Higress 都有公开定价。Solo.io 一直是"contact sales"模式，对预算审查严格的客户不友好。

#### 3. **学习曲线陡**

要玩转 agentgateway，需要同时懂：
- K8s Gateway API
- Rust 配置文件结构
- MCP 协议
- A2A 协议
- CEL 表达式
- xDS 数据面通信

相比之下，Portkey 的"5 行配置"和 LiteLLM 的"import 就能用"对开发者更友好。

#### 4. **和 Envoy 生态的边界模糊**

agentgateway 是不是"Envoy 的替代品"？如果客户已有 Envoy/Istio 投资，**迁移到 agentgateway 的收益不一定明显**。Envoy AI Gateway 也提供类似能力，且和 Istio 天然集成。

#### 5. **缺乏 SaaS-first 体验**

agentgateway 的核心部署是**自托管优先**（Standalone + K8s）。Solo Cloud（SaaS）虽然有，但功能可能落后自托管版本。对**不想运维基础设施**的中小客户，Portkey、Cloudflare、Helicone 更适合。

#### 6. **Solo.io 单一供应商风险**

agentgateway 的核心开发资源集中在 Solo.io。虽然捐给了 AAIF，但实际 commit 80%+ 来自 Solo.io 员工。**和 Linux Kernel vs Red Hat 的关系类似**——公司策略变化可能影响项目。

#### 7. **缺乏"模型市场"或"渠道商"能力**

对比：
- OpenRouter、Together、Fireworks：本身是"模型聚合 + 渠道"
- Portkey、Helicone：偏"可观测 + 路由"
- **agentgateway**：偏"网关 + 协议"

agentgateway **不直接提供模型**（不自营 LLM API），客户必须自己接 OpenAI/Anthropic/Bedrock 等。这意味着它**不能像 One API 那样做"小 B 渠道分销"**。

#### 8. **国内生态缺位**

- 无中文文档（截至 2026-06）
- 无国内云厂商镜像
- 无国内 LLM provider 的开箱即用支持（DeepSeek、豆包等需自配）
- 无国内合规相关特性（等保、ICP 等）

对于国内市场的中型客户，**Higress 仍是首选**。

---

## 10. 与其他产品对比

### 10.1 一句话对比

| 产品 | 一句话定位 |
|------|----------|
| **agentgateway** | Rust 写、AAIF 托管、把 MCP/A2A/LLM/API 统一在一个数据面跑的"AI 时代原生网关" |
| **Envoy AI Gateway** | CNCF 旗手，K8s Gateway API 标准 + Envoy 生态，最"标准"的 AI 网关 |
| **Higress** | 阿里出品，国内云原生 + 通义生态首选，WASM 插件扩展 |
| **Kong AI Gateway** | 老牌 API 网关厂商，AI 是插件层 |
| **APISIX ai-proxy** | Apache 顶级项目，国产老牌，最轻量 |
| **Portkey Gateway** | Node.js 写，体验最好、可观测最全，**SaaS 为主** |
| **LiteLLM** | Python 写，**SDK 层**统一所有 LLM 调用 |
| **One API / New API** | Go 写，**国内渠道商**事实标准，账号池 + 分发 |
| **Cloudflare AI Gateway** | 边缘节点 + 缓存，可观测强，**SaaS 化** |
| **Helicone** | 偏**可观测**的代理，replay、用户维度分析 |

### 10.2 维度对比表

| 维度 | agentgateway | Envoy AI Gateway | Higress | Kong | APISIX | Portkey | LiteLLM | One API |
|------|--------------|-----------------|---------|------|--------|---------|---------|---------|
| **语言** | Rust | Go + C++ | Go + WASM | Go + Lua | Go + Lua | Node.js | Python | Go |
| **协议广度** | ★★★★★ | ★★★ | ★★★★ | ★★★★ | ★★★ | ★★★ | ★★★★ | ★★★ |
| **MCP 支持** | ★★★★★ | ★ | ★ | ★ | ★ | ★ | ★ | ✗ |
| **A2A 支持** | ★★★★★ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| **LLM 路由** | ★★★★★ | ★★★★ | ★★★★ | ★★★★ | ★★★ | ★★★★★ | ★★★ | ★★★ |
| **性能** | ★★★★★ | ★★★★ | ★★★ | ★★★ | ★★★ | ★★ | ★★ | ★★ |
| **K8s 集成** | ★★★★★ | ★★★★★ | ★★★★ | ★★★★ | ★★★ | ★★ | ★★ | ★ |
| **可观测性** | ★★★★ | ★★★★ | ★★★★ | ★★★★ | ★★★ | ★★★★★ | ★★★ | ★★ |
| **国内生态** | ★ | ★ | ★★★★★ | ★★★ | ★★★★ | ★★ | ★ | ★★★★★ |
| **上手难度** | 高 | 中高 | 中 | 中 | 中 | 低 | 低 | 低 |
| **生态/社区** | 中（AAIF 新） | 大（CNCF） | 大（阿里） | 大 | 中大 | 中大 | 大 | 中大 |
| **商业成熟度** | 中 | 中 | 中高 | 高 | 中 | 高 | 中 | 中 |
| **价格透明度** | 低 | 中 | 中 | 中 | 高 | 中 | 高 | 高 |
| **典型部署规模** | 大企业 | 大企业 | 阿里生态 | 大企业 | 中大 | 中小到大 | 中小到大 | 中小 |

### 10.3 决策树

```
你的场景是什么？
│
├── 大企业、严格合规、金融/政府、K8s 重度用户
│    ├── 已有 Istio 投资 → Envoy AI Gateway
│    ├── 想要 Rust 性能 + AI 协议原生 → agentgateway
│    └── 已有 Kong → Kong AI Gateway
│
├── 国内业务为主
│    ├── 通义/百炼生态 → Higress
│    ├── 多渠道账号分发 → One API / New API
│    └── 想要插件扩展 → APISIX ai-proxy
│
├── 中小企业/初创，缺运维人力
│    ├── 想要 SaaS → Portkey Cloud / Cloudflare AI Gateway
│    ├── 想要可观测 + 代理 → Helicone
│    └── Python 为主、SDK 而非网关 → LiteLLM
│
├── 自托管 GPU 池，需要 Inference Gateway
│    ├── vLLM / TGI / Triton → agentgateway 或 Higress
│    └── 想要生产成熟 → BentoML / Anyscale
│
├── Agent 业务为主（MCP、A2A、多 agent 协作）
│    └── agentgateway（MCP 一等公民 + A2A 路由）
│
└── 单体服务/Lite 部署
     └── One API / New API / LiteLLM
```

### 10.4 替代关系（互斥还是互补）

| 组合 | 关系 | 说明 |
|------|------|------|
| agentgateway + Higress | **互斥** | 都在网关层；选一个 |
| agentgateway + Portkey | **互补** | agentgateway 做数据面转发，Portkey 做团队级可观测 |
| agentgateway + Cloudflare | **互补** | 边缘用 Cloudflare，中心用 agentgateway |
| agentgateway + LiteLLM | **互斥** | 网关层重复 |
| agentgateway + Helicone | **互斥或互补** | 重叠场景多；可保留 Helicone 做 replay |
| agentgateway + vLLM | **互补** | agentgateway 是 vLLM 的前端 |

---

## 11. 未来趋势与风险

### 11.1 路线图（2026-2027）

来自 Lin Sun 2026-06-05 博客：

- **UI 增强**：历史分析、请求详情、AI 集成展示
- **推理工作负载**：集成 LLM-d（Red Hat 分布式推理）和 vLLM Semantic Router
- **MCP 演进**：Stateless MCP、更丰富的 guardrails、progressive disclosure 扩展
- **Code Mode 扩展**：OpenAPI → MCP 桥的更多自动化
- **Agent Client Protocol (ACP)** 集成
- **国际化（i18n）**：社区驱动的翻译
- **Kubernetes 合作**：kube-agentic-networking、AI Gateway API（wg-ai-gateway）
- **生产案例**：更多公开 case study 和架构模式

### 11.2 关键风险

1. **Solo.io 公司层面**：B2B 基础设施 SaaS 在 2024-2025 的资本环境下融资困难。如果 Solo.io 出现资金问题，agentgateway 项目的健康度会受影响（AAIF 治理可缓解但不能完全消除）。

2. **Envoy 反击**：如果 Envoy 社区决定"我们也做 MCP/A2A 原生支持"，agentgateway 的差异化会缩小。但 Rust 重写的投入是巨大的，难以快速复制。

3. **MCP 协议本身变化**：MCP 还在快速演进（Stateless MCP、ACP 出现）。如果协议发生重大变化，agentgateway 的 MCP 一等公民优势可能变成"包袱"。

4. **国内厂商竞争**：阿里 Higress、字节内部的 AI 网关、腾讯云 API Gateway 都有更强资源投入国内企业市场。

5. **AI Gateway 商品化**：随着 Gateway API Inference Extension（K8s SIG 主导）成熟，标准化的 AI 网关能力可能不再需要"自建代理"。

### 11.3 适合的读者

- ✅ **大企业平台团队**：K8s 重度、要求中立治理、需要 MCP/A2A 支持
- ✅ **Agent 产品开发者**：需要 MCP 联邦、A2A 路由、agent 互操作
- ✅ **AI 基础设施厂商**：OEM 集成、Dell/CoreWeave 模式
- ✅ **追求极致性能**：流式 + 大流量 + 低延迟场景
- ❌ **小 B 用户**：运维成本太高
- ❌ **个人开发者**：上手成本高，LiteLLM / Portkey 更合适
- ❌ **国内业务为主**：Higress 更合适

---

## 12. 附录

### 12.1 关键事实速查

| 项目 | 详情 |
|------|------|
| **产品名** | agentgateway（前 Gloo AI Gateway） |
| **公司** | Solo.io |
| **创建日期** | 2025-03 |
| **首次开源** | 2025-03 |
| **捐赠 Linux Foundation** | 2025-08-25 |
| **加入 AAIF** | 2026-06-04（最新） |
| **当前版本** | Solo Enterprise 2026.5.2 / 2.3.x |
| **许可证** | Apache 2.0 |
| **主语言** | Rust（数据面）+ Go（K8s controller，K8s 模式） |
| **关键依赖** | Tokio、Hyper、Tonic、cel-rust |
| **GitHub** | https://github.com/agentgateway/agentgateway |
| **官网** | https://agentgateway.dev |
| **企业文档** | https://docs.solo.io/agentgateway |
| **社区** | Discord、每周社区会议、AAIF 治理 |
| **主架构师** | Lin Sun（Solo.io Head of Open Source，Istio TOC） |
| **核心贡献者组织** | Solo.io、CoreWeave、Red Hat、Adobe、Salesforce、Amdocs、Microsoft |

### 12.2 关键链接

- 设计哲学博客（2026-06-05）：https://www.solo.io/blog/designing-agentgateway-a-unified-high-performance-gateway-for-ai-and-api-traffic
- 加入 AAIF 公告（2026-06-04）：https://www.solo.io/blog/agentgateway-joins-aaif-as-an-open-gateway-for-agentic-ai-infrastructure
- 2.3 发布博客（2026-04-15）：https://www.solo.io/blog/solo-enterprise-for-agentgateway-2-3-aws-bedrock-agentcore-intelligent-failover-and-deeper-mcp-control
- OpenAPI → MCP Code Mode（2026-05-13）：https://www.solo.io/blog/agentgateway-code-mode-for-openapi-to-mcp
- Progressive Disclosure（2026-04-22）：https://www.solo.io/blog/keeping-context-and-tokens-low-with-progressive-disclosure-in-agentgateway
- 2026 Hackathon winners（2026-05-06）：https://www.solo.io/blog/celebrating-the-winners-of-the-2026-hackathon-for-mcp-ai-agents
- 第三方基准：https://github.com/howardjohn/gateway-api-bench/blob/main/README-v2.md
- AAIF 公告（2026-06-04）：https://aaif.io/blog/agentgateway-joins-aaif-as-an-open-gateway-for-agentic-ai-infrastructure/

### 12.3 配置文件示例（K8s HTTPRoute → LLM）

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai-route
  namespace: ai-gateway
spec:
  parentRefs:
  - name: agentgateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1/chat/completions
    backendRefs:
    - name: openai-backend
      kind: Backend
---
apiVersion: gateway.kgateway.dev/v1alpha1  # Solo Enterprise CRD
kind: Backend
metadata:
  name: openai-backend
  namespace: ai-gateway
spec:
  type: AI
  ai:
    llm:
      provider:
        openai:
          model: gpt-4o
          authTokenRef:
            name: openai-secret
            key: api-key
      routes:
      - name: cheap-fallback
        provider:
          openai:
            model: gpt-4o-mini
        weight: 20
      - name: primary
        weight: 80
      # 2.3 新增：CEL failover
      failover:
        conditions:
        - cel: 'response.code == 429 || response.code >= 500'
        evictionDuration: 60s
```

### 12.4 MCP Federation 示例

```yaml
# 把 3 个 MCP server 联邦成一个虚拟 endpoint
apiVersion: agentgateway.dev/v1alpha1
kind: MCPServer
metadata:
  name: docs-mcp
spec:
  backend: stdio://npx -y @modelcontextprotocol/server-filesystem /docs
---
apiVersion: agentgateway.dev/v1alpha1
kind: MCPServer
metadata:
  name: github-mcp
spec:
  backend: https://api.github.com/mcp
  auth:
    oauth:
      clientIdRef:
        name: github-oauth
        key: client-id
---
apiVersion: agentgateway.dev/v1alpha1
kind: MCPServer
metadata:
  name: db-mcp
spec:
  backend: http://db-mcp.default.svc.cluster.local:8080/mcp
  auth:
    jwt:
      issuer: https://idp.internal
---
# 联邦成单一 endpoint
apiVersion: agentgateway.dev/v1alpha1
kind: MCPFederation
metadata:
  name: corp-mcp
spec:
  servers:
  - docs-mcp
  - github-mcp
  - db-mcp
  # 工具级 RBAC
  policies:
  - name: restrict-github
    match:
      tool: "github_*"
    condition: 'request.auth.user.groups.contains("developers")'
    action: allow
  # progressive disclosure
  progressiveDisclosure:
    enabled: true
    # 默认只暴露工具名，详细 schema 按需加载
```

### 12.5 参考文档

- 完整 docs：https://docs.solo.io/agentgateway
- LLM 路由：https://docs.solo.io/agentgateway/latest/llm/
- MCP：https://docs.solo.io/agentgateway/latest/mcp/
- A2A：https://docs.solo.io/agentgateway/latest/agent/a2a/
- Inference Gateway：https://docs.solo.io/agentgateway/latest/inference/
- Guardrails：https://docs.solo.io/agentgateway/latest/llm/guardrails/overview/
- OpenTelemetry：https://docs.solo.io/agentgateway/latest/observability/

### 12.6 关联研究

本调研报告与本系列已有研究的关系：

- **01-llm-protocols.md**：agentgateway 实现了 OpenAI 兼容 + Anthropic + Gemini + Bedrock + 自托管 vLLM 协议统一
- **03-intelligent-routing.md**：agentgateway 的 cost-aware / latency-aware / CEL-based eviction routing
- **06-guardrails.md**：agentgateway 的 prompt guard + PII shield + 外部 moderation
- **11-mcp-deep-dive.md**：agentgateway 是 MCP 网关层最完整的实现
- **12-a2a-protocol.md**：agentgateway 是 A2A 网关的旗手
- **19-sla-service-governance.md**：agentgateway 的 OBO token exchange、fine-grained RBAC、CEL policy
- **20-future-2027-2030.md**：agentgateway 的 AAIF 治理模式是 AI 基础设施中立化的代表样本

---

## 13. 调研说明

- **数据时效**：报告基于 2026-06-04 ~ 2026-06-05 的最新公开数据。Solo.io 在最近 48 小时（2026-06-04 AAIF 加入、2026-06-05 设计博客）密集发声，**这是本报告的关键时点**。
- **数据来源**：Solo.io 官网（solo.io、agentgateway.dev、docs.solo.io）、GitHub（github.com/agentgateway/agentgateway、github.com/solo-io/gloo）、AAIF 公告、第三方基准（gateway-api-bench v2）、CSDN/Medium 等技术博客交叉验证。
- **未覆盖内容**：本报告未深入分析 Solo.io 公司财务、Salesforce OEM 合作具体条款、AWS Bedrock AgentCore 三种 OBO 模式的技术细节对比（建议另开专题）。如需深度展开可补充。
- **报告用途**：供 AI Gateway 选型决策、技术趋势研究、投资分析参考。
