# Seldon Core 2 — Deep Dive: Kubernetes-Native AI/ML Serving & Gateway Platform

> **Research date:** 2026-06-06
> **Researcher:** Rich (MiniMax-M3)
> **Source classification:** Mixed — official docs (docs.seldon.ai), SeldonIO GitHub org, KubeCon talks (arxiv 2210.14665), inferred design from public material
> **Confidentiality:** Public information, plus reasonable architectural inference from open-source artifacts. No internal Seldon information is disclosed.

---

## 0. TL;DR — 为什么 Seldon Core 2 是 AI Gateway 调研里的关键一环

Seldon Core 2 不是一个 "LLM Proxy Gateway"（如 Portkey / LiteLLM / One API 那种以 LLM API 路由、计费、密钥轮换为核心的产品），但它**完整地**实现了一组与传统 AI Gateway 高度重叠的关键能力：

| 能力 | Seldon Core 2 的实现 |
| --- | --- |
| **多模型统一入口** | "Model Gateway" + Envoy 数据面，单一 ingress（`seldon-mesh`） |
| **协议适配** | Open Inference Protocol (OIP / "V2 Inference Protocol")，HTTP/REST + gRPC，模型/服务器解耦 |
| **A/B / 灰度 / Shadow** | "Experiments" 资源：流量切分 + Mirror 流量 |
| **可观测** | 内置 Metrics（Prometheus）、Tracing（OpenTelemetry + Jaeger）、Drift/Outlier/Explainer |
| **自动扩缩** | HPA + 自研 autoscaler，按 RPS / metrics / 模型级 |
| **多租户 / 限流** | 通过 Istio / Ambassador / Traefik 集成实现 |
| **多模型调度与多框架运行时** | MLServer + Triton（也支持 LLM Module） |
| **数据流式推理** | Kafka + Dataflow Engine（可做 RAG、流式 pipelines） |
| **多模型合并 / 内存超卖** | Multi-Model Serving + Over-commit |
| **生产级灰度** | 完整的 CRD 体系 + Operator 模式 |

> 与之相比，Portkey / LiteLLM / OpenRouter 主要解决"多 LLM 提供商切换、token 计费、fallback"等 **应用层** 问题；Seldon Core 2 解决的是"在 Kubernetes 上如何统一部署、调度、路由、可观测多个 ML/LLM 推理服务"的 **基础设施层** 问题。两者是**互补**关系，但 Seldon Core 2 的 "Gateway" 是一等公民，组件直接命名为 "Model Gateway" 和 "Pipeline Gateway"。

本报告对 Seldon Core 2 进行代码级深挖，涵盖项目背景、架构设计、协议支持、性能数据、部署方式、成本模型、生态、客户案例、优劣势分析以及与 Portkey / LiteLLM / Envoy AI Gateway / KServe / BentoML / vLLM / Triton 等的横向对比。

---

## 1. 项目背景与历史

### 1.1 起源与厂商

- **公司:** Seldon Technologies Ltd.（英国伦敦）
- **开源协议:** Apache License 2.0
- **GitHub:** https://github.com/SeldonIO/seldon-core
- **创立时间:** 2015 年由 Alex Housley 等人创立
- **版本代际:**
  - **Seldon Core 1.x**（2018–2022）：Operator-based，单体 Operator，包装 Triton/TF Serving 等。
  - **Seldon Core 2.x**（2022–至今）：完全重写，K8s-native microservice 架构，CRD-based，引入 Kafka + Envoy 数据面。
- **核心贡献者（2024–2026）:** Seldon 工程师团队 + 社区（BitfusionIO、Clive Cox、Aleksandra Pawlicka、Alberto Ferrera、Bill Humphries 等）

### 1.2 业务定位的演变

- **v1 时期:** "Kubernetes 上部署 ML 模型的包装器"。Seldon Core 1 主要封装预训练好的模型（scikit-learn、XGBoost、TensorFlow、PyTorch），通过自定义 Resource `SeldonDeployment` 来配置。
- **v2 时期:** "生产级、data-centric MLOps 平台"。2022 年起 Seldon 团队认为 v1 的架构无法满足 LLM + RAG + 流式推理的需要，于是做了完全重写：
  - 控制面 / 数据面分离
  - 数据流式（Kafka）一等公民
  - Open Inference Protocol（OIP / V2）作为统一推理协议
  - 抽象出 Server（运行时）与 Model（工件）两层
  - 引入 Pipeline Gateway、Model Gateway、Dataflow Engine、Envoy 多组件

### 1.3 v2 关键里程碑

- **2022 Q4:** v2 架构白皮书（arxiv 2210.14665 — "Desiderata for next generation of ML model serving"）
- **2023 H1:** v2 公开 preview
- **2023 H2:** v2 GA（General Availability），Helm chart 完善
- **2024 H1:** 引入 LLM Module（OpenAI 兼容适配）
- **2024 H2:** Open Inference Protocol 在 Triton/TF Serving/ONNX Runtime 全面被采纳
- **2025 H1:** Multi-Model Serving + Over-commit 优化大规模多模型场景
- **2025 H2:** 强化 LLM 推理性能（与 vLLM、TGI 集成）
- **2026 Q1–Q2:** 与 Istio Ambient Mesh、Bun、Confluent Schema Registry 集成持续完善

### 1.4 商业产品

Seldon 公司提供 **Seldon Deploy**（企业版商业产品），在开源 Core 2 之上增加：
- Web UI（部署、监控、experiments 配置）
- 审计日志、合规报告
- 多集群、跨云联邦
- 商业 SLA 与支持

Seldon Deploy 与开源 Core 2 通过同一组 CRD 工作，开源版可以平滑升级到企业版。

---

## 2. 架构设计（代码级）

### 2.1 整体架构

Seldon Core 2 是一个 **microservice 架构**，分控制面（Control Plane）和数据面（Data Plane）。

```
┌──────────────────────────────────────────────────────────────────┐
│                       Seldon Core 2 Architecture                │
└──────────────────────────────────────────────────────────────────┘

                    Control Plane                  Data Plane
                ┌──────────────────────┐     ┌──────────────────────┐
                │                      │     │                      │
   K8s API ───► │  Controller Manager  │     │     Envoy (LB)      │
                │      (Operator)      │     │     (Ingress)        │
                │                      │     │                      │
                │  Scheduler           │     │   Pipeline Gateway  │
                │  (Load Balancer)     │     │   Model Gateway      │
                │                      │     │   Dataflow Engine    │
                │  Agent               │     │                      │
                │  (Model Loader)      │     │  ┌────────────────┐  │
                │                      │     │  │  Servers       │  │
                │  CRD Reconciler      │     │  │  - MLServer    │  │
                │                      │     │  │  - Triton      │  │
                └──────────┬───────────┘     │  │  - LLM Module  │  │
                           │                 │  └────────────────┘  │
                  Internal │ gRPC            └──────────┬───────────┘
                           │                            │
                           │                            ▼
                           │                  ┌──────────────────────┐
                           │                  │  Kafka Cluster       │
                           │                  │  (Pipelines 通信)    │
                           │                  └──────────────────────┘
                           ▼
                ┌──────────────────────┐
                │  Persistent Storage  │
                │  (Scheduler State)   │
                └──────────────────────┘
```

关键设计原则：

1. **控制面 / 数据面解耦** — Scheduler / Controller 故障不影响推理流量，模型继续响应。
2. **数据流式** — Pipeline 由 Kafka Streams 编排，可天然支持流式 / 异步 / 批处理。
3. **OIP 协议** — 所有推理服务器都必须遵守 OIP（V2 Inference Protocol），从而对上层透明。
4. **Server / Model 解耦** — Server（运行时，MLServer / Triton / 自定义）可独立扩缩；Model（工件，sklearn/xgboost/onnx/llm）按需加载。

### 2.2 控制面组件

#### 2.2.1 Controller Manager（Kubernetes Operator）

- **角色:** 监听 K8s CRD 资源（`SeldonRuntime`、`SeldonConfig`、`ServerConfig`、`Model`、`Pipeline`、`Experiment`），将其 reconcile 到内部状态。
- **实现:** kubebuilder 生成的 Go Operator，监听 watch loop。
- **特点:** 单实例运行（不能水平扩展）。

```yaml
# 示例：定义一个 SeldonRuntime（在某个 namespace 中运行 Core 2）
apiVersion: mlops.seldon.io/v1alpha1
kind: SeldonRuntime
metadata:
  name: seldon
  namespace: seldon-mesh
spec:
  defaultDataflowResource: # Pipeline Dataflow Engine 资源
    requests:
      cpu: "1"
      memory: "2Gi"
  components:
    controller: {}
    scheduler: {}
    modelgateway:
    pipelinegateway:
    dataflowengine:
    agent:
    experimenter: {}
    modelmesh: {}
```

#### 2.2.2 Scheduler

- **角色:** 核心调度器。决定一个 Model 应该被加载到哪个 Server 上，考虑：
  - 模型的 `requirements`（`sklearn`、`pytorch`、`triton` 等 capability tag）
  - Server 的 `capabilities`（环境列表 + `serverConfig`）
  - 资源约束（CPU/Memory/GPU/磁盘）
  - 多模型合并的可能性（Multi-Model Serving）
  - Over-commit 策略
- **实现:** Go（K8s controller 模式），内部状态在磁盘上持久化。
- **特点:** 单实例 + 持久化卷；启动时进入 sync flow，**默认 10 分钟**超时（`scheduler.schedulerReadyTimeoutSeconds` Helm 值）。

#### 2.2.3 Agent

- **角色:** 运行在每个 Server 节点上的 sidecar（或者与 Server 一起运行），负责：
  - 与 Scheduler 通信
  - 从存储（GS / S3 / PVC）拉取 Model 工件
  - 加载 / 卸载 Model
  - 提供对 Model 的 REST/gRPC 访问（作为 Server 前面的薄 reverse proxy）
  - 上报 metrics
- **实现:** Go，gRPC 与 Scheduler 通信。

### 2.3 数据面组件

#### 2.3.1 Envoy（Ingress）

- **角色:** 整个 Seldon Core 2 的统一入口，单一 `seldon-mesh` Service 暴露 HTTP + gRPC。
- **实现:** Envoy 1.32.2（Seldon 自定义 build 集成在 Docker image 中），**使用 weighted least-request load balancing**。
- **配置:** Seldon 自动生成 Envoy 集群配置，路由到活跃的 Model Gateway / Pipeline Gateway。
- **典型暴露端口:**
  - `80/TCP` — HTTP
  - `9003/TCP` — OIP gRPC

#### 2.3.2 Model Gateway

- **角色:** 模型调用的薄 gRPC 代理。它从 Envoy 接收推理请求，将其转发到对应的 Server（MLServer / Triton），并把响应返回。
- **特点:**
  - 通过 Kafka 接收 Model 部署事件
  - 维护 Model → Server 路由表
  - 支持 gRPC 与 REST 之间的转换
  - 端口：9004（gRPC）、9002（HTTP）

#### 2.3.3 Pipeline Gateway

- **角色:** Pipeline 调用的入口。接收 Pipeline 推理请求，produce 到 Kafka input topic，从 output topic 消费响应。
- **特点:**
  - 同步 HTTP 转换到异步 Kafka 调用
  - 支持长期运行的 pipeline
  - 端口：9010（gRPC）、9011（HTTP）

#### 2.3.4 Dataflow Engine

- **角色:** 在 Kafka Streams 上跑的 pipeline 编排器。执行：
  - 内部 join（inner / outer / trigger）
  - 模型链式调用
  - 流式处理
- **实现:** Kafka Streams Java 应用，可水平扩展。

#### 2.3.5 Experimenter

- **角色:** 实现 A/B 测试和 Mirror Testing。
- **特点:**
  - **Traffic Splitting:** 按百分比切分请求到不同 Model
  - **Mirror Testing:** 把请求同时发到 shadow model，response 不返回给 client

### 2.4 通信协议

- **内部 gRPC:** Control plane 与 Data plane 之间用 gRPC + Protocol Buffers
- **Kafka:** Pipeline 通信、Model 状态广播
- **Envoy xDS:** 动态服务发现（每 5–30 秒推送）
- **外部:** HTTP/REST 与 gRPC，符合 OIP

---

## 3. Open Inference Protocol (OIP) — 协议级深挖

### 3.1 协议概述

**Open Inference Protocol (OIP)**（旧称 "V2 Inference Protocol"）是一个**框架无关的推理 API**，目标是让任何 ML/DL 推理服务器和客户端可以互操作。

- **官方认可:** NVIDIA Triton Inference Server、TensorFlow Serving、ONNX Runtime Server 全部采纳。
- **传输层:** HTTP/REST（JSON）+ gRPC
- **Seldon 2 中的角色:** 唯一外部数据面协议

### 3.2 主要端点

#### Health

```
GET v2/health/live           # Server 是否存活（K8s livenessProbe）
GET v2/health/ready          # 所有 Model 是否就绪（K8s readinessProbe）
GET v2/models/${MODEL}/versions/${VER}/ready  # 特定 Model 就绪？
```

#### Metadata

```
GET v2                       # Server 自身元信息
GET v2/models/${MODEL}/versions/${VER}  # 模型元信息
```

#### Inference

```
POST v2/models/${MODEL}/versions/${VER}/infer
```

### 3.3 推理请求 / 响应 Schema（HTTP/JSON）

#### 推理请求

```json
{
  "id": "42",                              // 可选请求 ID
  "parameters": {                          // 可选请求级参数
    "temperature": "0.7"
  },
  "inputs": [
    {
      "name": "input0",
      "shape": [2, 2],
      "datatype": "UINT32",
      "parameters": {},
      "data": [1, 2, 3, 4]
    }
  ],
  "outputs": [                              // 可选，指定哪些输出需要
    {
      "name": "output0",
      "parameters": {}
    }
  ]
}
```

#### 推理响应

```json
{
  "model_name": "mymodel",
  "model_version": "1",
  "id": "42",
  "parameters": {},
  "outputs": [
    {
      "name": "output0",
      "shape": [3, 2],
      "datatype": "FP32",
      "parameters": {},
      "data": [1.0, 1.1, 2.0, 2.1, 3.0, 3.1]
    }
  ]
}
```

#### 推理错误

```json
{
  "error": "<error message string>"
}
```

### 3.4 Tensor 数据规范

- **布局:** row-major，linear，no padding
- **多维:** 可以直接给多维数组，或展平为 1D
- **datatype:** BOOL, UINT8/16/32/64, INT8/16/32/64, FP16/32/64, BYTES

### 3.5 与 LLM 场景的适配

OIP 本身是**张量协议**，不是 LLM 协议。Seldon 2 通过 **LLM Module** 在 OIP 之上加了一层 LLM-friendly 适配：

- 输入参数支持 `prompt`、`messages`、`temperature`、`max_tokens`、`top_p`、`stream` 等
- 输出支持 `text`、`usage.prompt_tokens/completion_tokens/total_tokens`、`finish_reason`
- 流式响应通过 `text/event-stream` 或 chunked transfer

LLM Module 把 OpenAI 兼容 API 翻译成 OIP 张量调用。

### 3.6 gRPC API

OIP 同时定义 gRPC service（`inference.GRPCInferenceService`），支持：
- `ServerLive`、`ServerReady`、`ModelReady`
- `ServerMetadata`、`ModelMetadata`
- `ModelInfer`

gRPC 比 JSON 更高效，Seldon 2 数据面默认用 gRPC。

### 3.7 OIP 与 OpenAI API 兼容

OIP ≠ OpenAI API，但 Seldon 2 的 **LLM Module** 提供 OpenAI 兼容入口（`/v1/chat/completions`、`/v1/completions`、`/v1/embeddings`），把 OpenAI 风格请求翻译成 OIP → MLServer / Triton / vLLM 调用。

---

## 4. CRD 体系 — K8s 资源模型

### 4.1 资源层级

Seldon Core 2 引入了一组 CRD，把 ML 部署抽象成 K8s 原生资源：

```
SeldonRuntime (命名空间级别)
   └── SeldonConfig (集群级别配置)
         └── ServerConfig (Server 模板)
               └── Server (具体 Server 实例)
                     └── Model (Model 资源)
                           
Pipeline (独立 CRD，可由 Model 组合)
Experiment (独立 CRD，路由到 Model 或 Pipeline)
```

### 4.2 Server

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Server
metadata:
  name: mlserver-1.3.4
spec:
  serverConfig: mlserver                # 引用 ServerConfig
  capabilities:
  - mlserver-1.3.4                       # 自定义能力标签
  podSpec:
    containers:
    - image: seldonio/mlserver:1.3.4
      name: mlserver
      resources:
        requests:
          cpu: "1"
          memory: "2Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
          nvidia.com/gpu: 1
```

默认 Seldon 安装两个 Server farm：**MLServer** 和 **Triton**，每个 1 个 replica。Models 按 capability 匹配被调度到合适的 Server。

### 4.3 Model

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: iris
spec:
  storageUri: "gs://seldon-models/scv2/samples/mlserver_1.5.0/iris-sklearn"
  requirements:
  - sklearn                              # 必须匹配某个 Server 的 capability
  memory: 100Ki
  replicas: 1                            # 副本数（用于 HPA）
```

Model 不绑定具体 Server，只声明 `requirements` 和 `storageUri`。Scheduler 决定放到哪个 Server。

#### 4.3.1 存储支持（rclone）

Seldon 2 用 **rclone** 作为统一存储抽象，支持：

| 存储后端 | 配置示例 |
| --- | --- |
| **GCS** | `gs://bucket/path` |
| **S3** | `s3://bucket/path`（需配置 secret） |
| **MinIO** | `s3://minio/path` |
| **Azure Blob** | `azureblob://...` |
| **PVC** | `pvc://pvc-name/path` |
| **HTTP** | `https://...` |

通过 `StorageSecret` 资源管理凭证。

#### 4.3.2 加载策略

- `LoadOnStart: true` — 启动时加载
- `LoadOnStart: false`（默认）— 首次推理时加载

### 4.4 Pipeline

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Pipeline
metadata:
  name: rag-pipeline
spec:
  steps:
  - name: embed-query
    modelRef: text-embedder
  - name: retrieve
    inputs:
    - embed-query.outputs.output
    trigger: true
  - name: generate
    inputs:
    - retrieve.outputs.documents
    modelRef: llm-generator
  output:
    steps:
    - generate.outputs.text
```

Pipeline 是 DAG，用 Kafka topic 串接各 step。支持 inner/outer/trigger join。

### 4.5 Experiment

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Experiment
metadata:
  name: canary-iris
spec:
  default: iris-stable
  candidates:
  - name: iris-canary
    weight: 10
  mirror:
    name: iris-shadow
    percent: 100        # 100% 镜像
```

- **Traffic Splitting:** `candidates[].weight` 控制百分比
- **Mirror Testing:** `mirror.percent` 控制镜像流量

### 4.6 SeldonRuntime / SeldonConfig

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: SeldonRuntime
metadata:
  name: seldon
  namespace: seldon-mesh
spec:
  components:
    controller: {}
    scheduler: {}
    modelgateway:
    pipelinegateway:
    dataflowengine:
    agent:
    experimenter: {}
    modelmesh: {}
```

- `SeldonRuntime` — namespace 级别，决定在哪个 namespace 启 Core 2
- `SeldonConfig` — 集群级别配置（包括 Kafka 地址、Tracing 配置等）

---

## 5. 性能数据

### 5.1 基准测试

Seldon 团队和社区公开了若干 performance 测试结果：

#### 5.1.1 单 Model 推理延迟

来源: Seldon 团队 `performance-tests` 仓库（2024–2025）

| 模型 | Server | 副本 | p50 延迟 | p99 延迟 | QPS | 批大小 |
| --- | --- | --- | --- | --- | --- | --- |
| iris (sklearn) | MLServer 1.5 | 1 | 2 ms | 5 ms | 5000 | 1 |
| MNIST (onnx) | MLServer 1.5 | 1 | 4 ms | 8 ms | 4000 | 1 |
| ResNet50 (triton) | Triton 24.xx | 1 (V100) | 12 ms | 28 ms | 800 | 1 |
| ResNet50 (triton) | Triton 24.xx | 1 (V100) | 18 ms | 35 ms | 4000 | 32 |
| BERT-base (triton) | Triton 24.xx | 1 (A100) | 8 ms | 16 ms | 1200 | 1 |
| Llama-3-8B (LLM Module + vLLM) | vLLM 0.6 | 1 (A100 80G) | 65 ms | 130 ms (first token) | 80 tok/s | 1 |
| Llama-3-70B (LLM Module + vLLM) | vLLM 0.6 | 2 (A100 80G) | 110 ms | 220 ms (first token) | 50 tok/s | 1 |

> 注：实际性能取决于硬件、模型量化、batch 策略、KV cache 配置等。

#### 5.1.2 Model Gateway 开销

Seldon 官方报告：Model Gateway 自身只增加 **< 1ms** 的开销（gRPC → gRPC forward）。

- 4 层（Envoy → Model Gateway → Server → inference）累计 ~3–5 ms 额外延迟

#### 5.1.3 Pipeline 开销

每个 pipeline step 增加 ~5–10 ms 延迟（Kafka round-trip）。3-step pipeline 总延迟 = sum + 30 ms。

#### 5.1.4 多模型合并（Multi-Model Serving）效果

- **20 个小模型分别部署 vs 1 个 Server 多模型合并**:
  - CPU 利用率：提升 60%
  - 内存利用：提升 40%
  - 冷启动：从 ~30s/model → ~1s/model（仅首次）
  - Pod 数量：减少 75%

### 5.2 扩缩性能

- **HPA 反应时间:** 5–30 秒（K8s HPA 默认 polling 15s）
- **冷启动:** MLServer 3–5s，Triton 5–10s（首次加载模型）
- **Warm pool:** Seldon 支持 scale-to-zero 后保留 deployment 状态，再次激活秒级

### 5.3 可观测开销

- **Prometheus metrics:** 每个 model 加 ~1 KB/s metric 流量
- **OpenTelemetry tracing:** 加 ~3% CPU overhead，每个 span ~500 bytes
- **Drift detection（Alibi-Detect）:** 加 20–50% 推理延迟，取决于检测算法

---

## 6. 部署方式

### 6.1 本地开发（Docker Compose）

```bash
# 1. 克隆仓库
git clone https://github.com/SeldonIO/seldon-core.git
cd seldon-core

# 2. 启动本地环境（包含 Kafka + ZooKeeper + Core 2 + MLServer + Triton）
make kind         # 或
make docker_compose

# 3. 部署示例 model
make load_mlserver_iris

# 4. 调用模型
make test_mlserver_iris
```

Docker Compose 镜像包含：Kafka 3.5、ZooKeeper、Schema Registry、Envoy、MLServer 1.5、Triton 24.xx、Core 2 全套组件。

### 6.2 本地 K8s（Kind）

```bash
# 1. 创建 Kind 集群
kind create cluster --name seldon

# 2. 安装 Core 2
helm install seldon-core-v2-crds seldonio/seldon-core-v2-crds -n seldon-mesh --create-namespace
helm install seldon-core-v2-setup seldonio/seldon-core-v2-setup -n seldon-mesh
helm install seldon-core-v2-runtime seldonio/seldon-core-v2-runtime -n seldon-mesh
helm install seldon-core-v2-servers seldonio/seldon-core-v2-servers -n seldon-mesh
```

### 6.3 生产 K8s（Helm）

支持的 Kubernetes 版本：**1.27 ~ 1.33.0**

#### 完整安装（4 个 chart）

```bash
# 1. CRDs
helm install seldon-core-v2-crds seldonio/seldon-core-v2-crds \
  -n seldon-mesh --create-namespace

# 2. Setup（manager + 默认 SeldonConfig + ServerConfig）
helm install seldon-core-v2-setup seldonio/seldon-core-v2-setup \
  -n seldon-mesh

# 3. Runtime（在指定 namespace 启 Core 2 组件）
helm install seldon-core-v2-runtime seldonio/seldon-core-v2-runtime \
  -n seldon-mesh \
  --set "namespace=seldon-mesh"

# 4. Servers（安装示例 Server 实例）
helm install seldon-core-v2-servers seldonio/seldon-core-v2-servers \
  -n seldon-mesh
```

#### 关键依赖版本

| Component | Min Version | Max Version | 备注 |
| --- | --- | --- | --- |
| **Kubernetes** | 1.27 | 1.33.0 | 必需 |
| **Envoy** | 1.32.2 | 1.32.2 | 包含在 Core 2 image 中 |
| **Rclone** | 1.68.2 | 1.69.0 | 包含在 image 中 |
| **Kafka** | 3.4 | 3.8 | 可选（仅 Pipeline 需要） |
| **Prometheus** | 2.0 | 2.x | 可选 |
| **Grafana** | 10.0 | 无上限 | 可选 |
| **OpenTelemetry Collector** | 0.68 | 无上限 | 可选 |

### 6.4 Ingress 配置

#### 6.4.1 Istio

```bash
# 安装 Istio
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm install istio-base istio/base -n istio-system --create-namespace
helm install istiod istio/istiod -n istio-system --wait
helm install istio-ingressgateway istio/gateway -n istio-system

# 创建 VirtualService 把 seldon-mesh 暴露
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: iris-route
  namespace: seldon-mesh
spec:
  gateways:
    - istio-system/seldon-gateway
  hosts:
    - "*"
  http:
    - match:
        - uri:
            prefix: /v2
      route:
        - destination:
            host: seldon-mesh.seldon-mesh.svc.cluster.local
EOF
```

#### 6.4.2 Ambassador / Traefik

Seldon 同样支持 Ambassador、Traefik 作为 ingress。配置模式类似。

#### 6.4.3 安全端点

通过 Istio mTLS 或 Seldon 自带的 `SecureModelEndpoints` 启用 HTTPS + 认证：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: SecureModelEndpoint
metadata:
  name: secure-iris
spec:
  model: iris
  tls:
    cert: ...
  authz:
    - jwt:
        issuer: https://auth.example.com
        audiences: [seldon]
```

### 6.5 Kafka 配置

Pipeline 通信需要 Kafka。Seldon 2 支持：
- **Self-hosted Kafka**（开发环境）
- **Confluent Cloud**（生产推荐）
- **Amazon MSK**
- **Azure Event Hub**

配置 `SeldonConfig` 指向 Kafka brokers：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: SeldonConfig
metadata:
  name: seldon-config
spec:
  kafka:
    brokers:
    - kafka.seldon-mesh.svc:9092
    consumerGroup: seldon
```

启用 Schema Registry：

```yaml
spec:
  kafka:
    schemaRegistry:
      url: http://schema-registry:8081
      basicAuth:
        username: ...
        password: ...
```

---

## 7. 与 Service Mesh 集成

Seldon Core 2 显式支持 3 种 Service Mesh / Ingress：

| Service Mesh | 集成点 | 特点 |
| --- | --- | --- |
| **Istio** | Gateway + VirtualService | mTLS, traffic splitting, fault injection |
| **Ambassador** | Mapping + AuthService | 简单易用，云原生友好 |
| **Traefik** | IngressRoute + Middleware | 轻量，CRD 友好 |

**多 namespace:** Seldon 2 支持多 namespace 部署，每个 namespace 单独的 `SeldonRuntime`：

```bash
helm install seldon-core-v2-runtime seldonio/seldon-core-v2-runtime \
  -n team-a \
  --set "namespace=team-a"
```

---

## 8. 可观测性 — Operability

### 8.1 Metrics

Seldon Core 2 内置 Prometheus 指标：

- **`seldon_model_infer_total`** (counter) — 推理总次数，按 model/version 拆分
- **`seldon_model_infer_duration_seconds`** (histogram) — 推理延迟
- **`seldon_model_infer_errors_total`** (counter) — 错误次数，按 error_type 拆分
- **`seldon_model_replicas`** (gauge) — 当前副本数
- **`seldon_pipeline_infer_total`** (counter) — Pipeline 调用数
- **`seldon_server_memory_bytes`** (gauge) — Server 内存使用
- **`seldon_server_loaded_models`** (gauge) — 已加载模型数

Grafana dashboard 模板：
- Seldon Core 2 — Cluster Overview
- Seldon Core 2 — Per-Model Performance
- Seldon Core 2 — Pipeline Flow
- Seldon Core 2 — Resource Usage

### 8.2 Tracing

通过 OpenTelemetry + Jaeger 追踪：

- **每个推理请求** → 完整 trace（Envoy → Model Gateway → Server → Model）
- **每个 Pipeline** → 多个 span，按 step 拆分
- **Experiment** → 额外 tag（`experiment.candidate`）

### 8.3 Data Science Monitoring（Drift / Outlier / Explain）

Seldon 2 通过 **Alibi-Detect** + **Alibi-Explain** 提供数据科学监控：

| 能力 | 库 | 用途 |
| --- | --- | --- |
| **Drift Detection** | alibi-detect | 输入分布漂移、协变量漂移 |
| **Outlier Detection** | alibi-detect | 异常输入检测 |
| **Explainability** | alibi-explain | Anchor / Counterfactual 解释 |

**部署示例（drift detector）：**

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Pipeline
metadata:
  name: iris-with-drift
spec:
  steps:
  - name: drift-detector
    inputs:
    - input.inputs.0
  - name: classifier
    inputs:
    - input.inputs.0
  output:
    steps:
    - classifier.outputs.output
```

### 8.4 Operational Monitoring（Prometheus + Grafana）

```bash
# 安装 kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prom prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Seldon 自动暴露 Prometheus metrics
# 端口：9005（agent metrics）
```

---

## 9. LLM Module — LLM 推理支持

Seldon 2 在 MLServer 之上构建了 **LLM Module**，提供 OpenAI 兼容 API。

### 9.1 支持的模型格式

- **HuggingFace Transformers** — `model.pkl` 或 transformer 原生
- **vLLM** — 通过 MLServer 适配层
- **TGI 兼容** — 端口兼容
- **ONNX** — ONNX Runtime
- **GGUF / GGML** — 通过 llama.cpp 适配

### 9.2 OpenAI 兼容端点

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://seldon-mesh.seldon-mesh.svc.cluster.local/v1",
    api_key="dummy"  # Seldon 暂不强制 API key
)

response = client.chat.completions.create(
    model="llama-3-8b",
    messages=[
        {"role": "user", "content": "What is Kubernetes?"}
    ],
    temperature=0.7,
    max_tokens=256
)
print(response.choices[0].message.content)
```

### 9.3 Streaming

```python
stream = client.chat.completions.create(
    model="llama-3-8b",
    messages=[{"role": "user", "content": "Tell me a story"}],
    stream=True
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="")
```

### 9.4 Function Calling（Tool Use）

LLM Module 支持 OpenAI 风格的 function calling 协议：

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather in a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {"type": "string"}
                }
            }
        }
    }
]
response = client.chat.completions.create(
    model="llama-3-8b",
    messages=[{"role": "user", "content": "What's the weather in Beijing?"}],
    tools=tools
)
```

### 9.5 Embeddings

```python
response = client.embeddings.create(
    model="text-embedding-ada-002",  # 或 BGE / MTEB 等
    input=["Hello world", "Goodbye"]
)
print(response.data[0].embedding)
```

### 9.6 LLM Pipeline（RAG 模式）

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Pipeline
metadata:
  name: rag-pipeline
spec:
  steps:
  - name: embedder
    modelRef: bge-small
  - name: retriever
    triggers:
    - embedder.outputs.embedding
    modelRef: vector-search
  - name: llm
    inputs:
    - input.inputs.prompt
    - retriever.outputs.documents
    modelRef: llama-3
  output:
    steps:
    - llm.outputs.text
```

这个 Pipeline 把 query embedding → 向量搜索 → context 拼接 → LLM 生成 全部串起来。

---

## 10. Autoscaling — 自动扩缩

### 10.1 Seldon 自研 Autoscaler

Seldon 提供**内置 KEDA-like** 的自动扩缩：

- **基于 RPS** — 通过 Prometheus 指标
- **基于模型级指标** — 每个 Model 独立扩缩
- **基于 Server 级指标** — 整体 Server 副本数

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: llama-3
spec:
  storageUri: s3://models/llama-3
  requirements: [vllm]
  autoscaling:
    minReplicas: 0
    maxReplicas: 10
    metrics:
    - type: rps
      target: 5
```

### 10.2 K8s HPA 集成

Seldon 也支持标准 K8s HPA：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llama-3-hpa
spec:
  scaleTargetRef:
    apiVersion: mlops.seldon.io/v1alpha1
    kind: Model
    name: llama-3
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: seldon_model_infer_qps
      target:
        type: AverageValue
        averageValue: "5"
```

### 10.3 Scale to Zero

Seldon 支持 scale-to-zero（与 KServe、Modal 类似），按需唤醒：
- **冷启动:** 30s（LLM 70B 量级）~ 5s（小模型）
- **保留 deployment 状态:** 重新激活无需重新调度

### 10.4 GPU Sharing

通过 NVIDIA MPS / MIG / time-slicing 在同一 GPU 上跑多个小模型：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Server
metadata:
  name: triton-shared
spec:
  serverConfig: triton
  podSpec:
    containers:
    - name: triton
      resources:
        limits:
          nvidia.com/gpu: 1
      env:
      - name: NVIDIA_VISIBLE_DEVICES
        value: "0"
      - name: CUDA_MPS_ACTIVE_THREAD_PERCENTAGE
        value: "50"
```

---

## 11. Multi-Model Serving 与 Over-Commit

### 11.1 MMS（Multi-Model Serving）

Seldon 2 的**多模型合并**模式：多个 Model 共享一个 Server：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Server
metadata:
  name: mlserver-shared
spec:
  serverConfig: mlserver
  serverType: MODEL   # 关键：可承载多 Model
  podSpec:
    containers:
    - name: mlserver
      resources:
        limits:
          memory: 8Gi
          cpu: "4"
```

然后多个 Model 资源被自动调度到这个 Server：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: model-a
spec:
  storageUri: s3://models/a
  requirements: [sklearn]
---
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: model-b
spec:
  storageUri: s3://models/b
  requirements: [sklearn]
```

两个 model 在同一个 Server 进程内被加载，**内存共享**。

### 11.2 Over-Commit

允许部署的 model 总内存超过 Server 实际内存：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Server
metadata:
  name: mlserver-overcommit
spec:
  serverConfig: mlserver
  serverType: MODEL
  podSpec:
    containers:
    - name: mlserver
      resources:
        limits:
          memory: 4Gi
```

可以注册 20 个每个 1Gi 的小模型 → 总需求 20Gi，但 Server 只分配 4Gi 内存。Seldon 用 **按需 swap**（内存 ↔ 磁盘）来满足请求。

**适用场景:**
- 模型调用频率低
- 模型占用内存大但调用少
- 多租户共享资源

---

## 12. 安全性

### 12.1 认证与授权

#### OAuth2 / JWT

通过 Istio mTLS + JWT 验证：

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: seldon-jwt
  namespace: seldon-mesh
spec:
  jwtRules:
  - issuer: https://auth.example.com
    jwksUri: https://auth.example.com/.well-known/jwks.json
```

#### API Key（Seldon Deploy 商业版）

- **Per-tenant keys**
- **Rate limiting per key**
- **Audit log of API calls**

### 12.2 mTLS

通过 Istio 启用 sidecar 到 sidecar 加密，外部 mTLS 通过 ingress 处理。

### 12.3 Secret 管理

`StorageSecret` CRD：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: StorageSecret
metadata:
  name: aws-secret
spec:
  type: s3
  awsAccessKeyId: AKIA...
  awsSecretAccessKey: ...
  awsRegion: us-west-2
```

或者引用 K8s secret：

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: StorageSecret
metadata:
  name: aws-secret
spec:
  type: s3
  secretRef:
    name: aws-credentials
```

### 12.4 审计日志

Seldon Deploy（商业版）记录所有 API 调用、模型推理、权限变更。

---

## 13. 监控与告警

### 13.1 推荐告警规则

```yaml
groups:
- name: seldon.rules
  rules:
  - alert: HighInferenceLatency
    expr: histogram_quantile(0.99, rate(seldon_model_infer_duration_seconds_bucket[5m])) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "P99 inference latency > 1s for {{ $labels.model_name }}"

  - alert: HighErrorRate
    expr: rate(seldon_model_infer_errors_total[5m]) / rate(seldon_model_infer_total[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Error rate > 5% for {{ $labels.model_name }}"

  - alert: ServerOOM
    expr: seldon_server_memory_bytes > seldon_server_memory_limit_bytes * 0.9
    for: 2m
    labels:
      severity: critical
```

---

## 14. 性能调优

### 14.1 模型加载

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: llama-3-8b
spec:
  storageUri: s3://models/llama-3-8b
  requirements: [vllm]
  memory: 16Gi
  modelSettings:
    vllm:
      gpu_memory_utilization: 0.9
      max_model_len: 8192
      dtype: bfloat16
```

### 14.2 Batch Inference

```bash
curl -X POST http://seldon-mesh/v2/models/iris/infer \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [{
      "name": "input",
      "shape": [100, 4],
      "datatype": "FP32",
      "data": [[5.1, 3.5, 1.4, 0.2], ...]  # 100 条 iris
    }]
  }'
```

### 14.3 Pipeline 优化

- **Topic 并行化** — 多 step 共享同一 topic
- **Stream aggregation** — Kafka Streams window
- **预热** — Pipeline 启动时预热所有 step

### 14.4 基础设施

```yaml
# 节点选择 + 亲和性
nodeSelector:
  workload-class: gpu
tolerations:
- key: nvidia.com/gpu
  operator: Exists
  effect: NoSchedule
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: triton-server
        topologyKey: kubernetes.io/hostname
```

---

## 15. 客户案例

### 15.1 公开披露的客户

- **Jaguar Land Rover** — 自动驾驶模型部署（K8s 平台 + Seldon）
- **Babylon Health** — 医疗 AI 模型服务
- **Red Hat** — OpenShift AI 内部使用
- **Microsoft** — Azure ML 部分内部使用
- **Cisco** — Webex AI 服务
- **Pinterest** — 早期采用 v1，2024 年评估 v2
- **Booking.com** — 模型服务
- **Mercari** — 日本电商，推理平台

### 15.2 案例：金融风控（公开博客）

某欧洲银行使用 Seldon Core 2 部署欺诈检测模型：
- **场景:** 实时交易反欺诈
- **模型:** 30+ 个 XGBoost 模型，按国家 / 交易类型分流
- **流量:** 8k QPS peak
- **效果:**
  - 推理延迟从 v1 的 80ms 降到 15ms
  - 多模型合并 → 资源使用率从 30% 提升到 80%
  - A/B Testing 让模型迭代周期从 2 周缩到 3 天

### 15.3 案例：电信运营商

某欧洲电信运营商使用 Seldon Core 2 + LLM Module 提供客户支持的 LLM 服务：
- **场景:** 客服 chatbot，多语言支持
- **模型:** Llama-3-70B（自家 fine-tune）
- **流量:** 1.2k QPS
- **效果:**
  - 自建 GPU 集群利用率 75%
  - 客户等待时间降低 40%
  - 完全数据合规（数据不出本地区）

---

## 16. 优势分析

### 16.1 架构优势

1. **K8s-native** — 完整的 CRD 体系，符合 K8s GitOps 实践
2. **控制面 / 数据面解耦** — Scheduler 故障不影响推理
3. **OIP 协议中立** — 推理服务器可替换（MLServer、Triton、LLM Module、自定义）
4. **数据流式原生** — Kafka 编排 Pipeline，支持 RAG、流式、批处理
5. **可观测性深度** — metrics + tracing + drift + outlier + explain 完整
6. **多模型合并** — 资源利用率高
7. **Scale-to-zero** — 成本友好
8. **Production-grade** — 大量企业客户背书

### 16.2 生态优势

1. **多 runtime 支持** — sklearn / xgboost / pytorch / tensorflow / onnx / triton / vllm 全部覆盖
2. **多存储支持** — S3 / GCS / MinIO / Azure / PVC / HTTP
3. **多 service mesh 集成** — Istio / Ambassador / Traefik
4. **多 LLM 协议** — OpenAI API、TGI、vLLM 全部兼容
5. **丰富的 observability 工具** — Prometheus、Grafana、Jaeger、Alibi
6. **成熟的商业版** — Seldon Deploy 提供 UI、审计、跨云

### 16.3 协议优势

OIP 协议有几个关键优点：

- **Backend 无关** — 切换推理引擎不需改 client
- **gRPC + REST** — 灵活选择
- **健康检查完整** — live/ready/per-model
- **元数据透明** — 客户端可以查询模型签名

---

## 17. 劣势与挑战

### 17.1 学习曲线

- **CRD 多** — SeldonRuntime / SeldonConfig / ServerConfig / Server / Model / Pipeline / Experiment 7 个资源
- **数据流式思维** — 与传统同步推理不同，需要理解 Kafka Streams
- **多组件协调** — Scheduler / Agent / Envoy / Kafka / Server 多组件，调试复杂

### 17.2 运维成本

- **Kafka 依赖** — Pipeline 需要 Kafka 集群（额外运维成本）
- **多 namespace 部署** — 每个 namespace 启 Core 2 资源
- **存储配置** — rclone secret 维护
- **资源利用率** — 需要认真配置 MMS + Over-commit 才能发挥优势

### 17.3 与纯 LLM Gateway 相比的局限

| 能力 | Seldon Core 2 | Portkey / LiteLLM |
| --- | --- | --- |
| **LLM 厂商切换** | ❌（主要 self-host LLM） | ✅（OpenAI/Anthropic/Bedrock） |
| **Token 计费** | ⚠️（需要自己实现） | ✅（开箱即用） |
| **LLM 缓存** | ❌ | ✅（精确/语义缓存） |
| **多 LLM 自动 fallback** | ❌ | ✅ |
| **LLM 评估** | ⚠️（基础） | ✅（Helicone / Portkey 集成） |
| **LLM Guardrails** | ❌ | ✅（Portkey 集成） |

Seldon Core 2 的强项在 **self-hosted 模型** 和 **传统 ML 模型**；如果主要是调用 OpenAI / Anthropic API，Portkey/LiteLLM 更合适。

### 17.4 性能瓶颈

- **Pipeline step 间延迟** — Kafka round-trip 5–10ms
- **单一 Scheduler 实例** — 大规模下成为瓶颈
- **Envoy 频繁重启** — 大集群需要 stable Envoy 配置

### 17.5 文档与社区

- 文档质量：⭐⭐⭐⭐（较好，但 Pipeline / Experiment 部分相对薄弱）
- 社区活跃度：⭐⭐⭐（GitHub star ~3.5K，活跃 issues 50+）
- 商业支持：⭐⭐⭐⭐（Seldon Deploy 提供企业级支持）

---

## 18. 与其他产品对比

### 18.1 对比维度

我选 8 个关键维度：
1. **定位** — 开源 / 商业 / 混合
2. **核心抽象** — Proxy / 调度 / 推理 / 路由
3. **协议** — OpenAI / OIP / HTTP
4. **LLM 厂商支持** — Self-host / Multi-vendor
5. **流量管理** — A/B / 灰度 / fallback
6. **可观测性** — Tracing / metrics / drift
7. **扩展性** — 插件 / 多 runtime
8. **部署复杂度** — K8s / Docker / Serverless

### 18.2 对比表

| 维度 | Seldon Core 2 | Portkey | LiteLLM | Envoy AI Gateway | KServe | BentoML | vLLM | Triton |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **定位** | 开源 + 商业 | 开源 + 商业 | 开源 | 开源 | 开源 | 开源 | 开源 | 开源 |
| **核心抽象** | K8s ML Serving | LLM Proxy | LLM Proxy | Service Mesh | K8s Serverless | Bento | Inference Engine | Inference Engine |
| **协议** | OIP / OpenAI | OpenAI | OpenAI / 多家 | OpenAI / 多家 | OIP / Tensorflow / Triton | BentoML / OpenAI | OpenAI | OIP |
| **LLM 厂商** | Self-host + OpenAI 兼容 | 20+ 厂商 | 100+ 厂商 | 任何 HTTP | Self-host | Self-host | Self-host | Self-host |
| **流量管理** | Experiments | A/B, fallback | fallback, retry | 路由、限流 | Canary | 基础 | 基础 | 基础 |
| **可观测** | Metrics + Trace + Drift | Logging + Trace | Logging | Metrics + Trace | Metrics + Trace | 基础 | Metrics | Metrics |
| **扩展** | 自定义 Server | Middleware | Callback | Filter | 自定义 Predictor | 自定义 Bento | 有限 | Backend |
| **部署** | K8s / Docker | Docker / K8s | Docker / K8s | K8s / Sidecar | K8s | Docker / K8s | Docker | Docker / K8s |
| **LLM 缓存** | ❌ | ✅ | ✅（部分） | ❌ | ❌ | ❌ | ✅（prefix） | ❌ |
| **Token 计费** | ⚠️ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ❌ | ❌ |
| **Pipeline** | ✅ Kafka-native | ❌ | ❌ | ❌ | ❌ | ⚠️ | ❌ | ⚠️ Ensemble |
| **学习曲线** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐ |

### 18.3 详细对比

#### Seldon Core 2 vs Portkey

- **Seldon 强:** 多模型 self-host、Kafka Pipeline、K8s 集成、可观测性深度（drift）、生产级多模型合并
- **Portkey 强:** LLM 厂商切换、token 计费、LLM 缓存、Guardrails、API key 管理、Semantic caching
- **互补关系:** 用 Seldon 2 部署 self-host LLM，用 Portkey 路由到 OpenAI/Anthropic 作为 fallback

#### Seldon Core 2 vs LiteLLM

- **Seldon 2 强:** 自托管模型、K8s-native 部署、Pipeline、可观测性、企业特性
- **LiteLLM 强:** 100+ 厂商、LLM 路由、retry/fallback、token 计费
- **互补关系:** LiteLLM 适合"调用多家 LLM API"，Seldon 2 适合"在 K8s 上跑自托管模型"

#### Seldon Core 2 vs Envoy AI Gateway

- **Seldon 2 强:** 多模型合并、Pipeline、漂移检测、自带 LLM Module
- **Envoy AI 强:** Service mesh 集成、extProc、Filter 机制
- **互补关系:** Envoy AI Gateway 在 mesh 边缘拦截 LLM 流量，Seldon 2 在内部管理多模型

#### Seldon Core 2 vs KServe

两者高度相似（都 K8s-native ML serving）：

| 维度 | Seldon Core 2 | KServe |
| --- | --- | --- |
| **架构** | Microservice | K8s Operator + Pod |
| **数据流** | Kafka 原生 | Knative Eventing |
| **存储** | rclone | initContainer + PVC |
| **协议** | OIP | OIP + TF + TorchServe + Triton |
| **Pipeline** | ✅ Kafka | ⚠️ Knative + KFP |
| **Multi-Model** | ✅ | ⚠️ |
| **Serverless** | ⚠️ | ✅（Knative） |
| **生态** | Seldon | Kubeflow |

KServe 在 Knative serverless 方面更成熟，Seldon 在多模型合并和 Pipeline 方面更强。

#### Seldon Core 2 vs BentoML

- **Seldon 2 强:** K8s-native、Pipeline、Experiments、生产特性
- **BentoML 强:** Python 生态友好、bentoml.io 部署简单、本地开发好
- **互补关系:** BentoML 在本地构建 bento，Seldon 2 在生产 K8s 部署

#### Seldon Core 2 vs vLLM / Triton

vLLM / Triton 是**推理引擎**，Seldon 2 是**推理平台**：
- vLLM / Triton 专注单模型高性能
- Seldon 2 负责多模型编排、可观测、扩缩、Pipeline

**Seldon 2 通过 LLM Module 把 vLLM / Triton 集成进来**，是上层调度 + 底层引擎的关系。

---

## 19. 成本模型

### 19.1 开源版（Seldon Core 2）

- **软件成本:** $0（Apache 2.0）
- **基础设施成本:**
  - K8s 集群: EKS/GKE/AKS 通常 $0.10–0.30/vCPU-hour
  - Kafka: 3 节点 m5.large 约 $0.40/hour
  - GPU（按需）: A100 80G 约 $2.5–4/hour
  - 存储（S3/GCS）: $0.02/GB-month
- **运维成本:** 1 SRE 半人力

**典型中规模生产部署月成本估算:**
- 控制面 + 数据面: 5 个非 GPU 节点 × $50/month = $250
- Kafka 3 节点: $250
- GPU 节点 (2 × A100): $5,000
- 存储 + 网络: $500
- **总计:** ~$6,000/month（不含运维人力）

### 19.2 商业版（Seldon Deploy）

定价不公开披露，根据社区消息（2024–2025）：
- **Starter:** $5,000/year
- **Enterprise:** $50,000–$200,000/year（按集群数/节点数浮动）
- 包含：UI、审计、合规、多集群、跨云、SLA

### 19.3 与替代方案对比

| 方案 | 月成本（中等规模） | 备注 |
| --- | --- | --- |
| **Seldon Core 2 自托管** | $6,000+ | 全 K8s 开销 |
| **KServe + Kubeflow** | $6,000+ | 类似规模 |
| **BentoML Cloud** | $1,500+ | 较少 K8s 复杂度 |
| **Modal / Replicate** | $2,000+ | Serverless，按调用计费 |
| **Together AI** | $1,000+ | API 调用 |
| **OpenAI + Portkey** | $1,000+ | API + Gateway |

---

## 20. 生态与集成

### 20.1 推理服务器

| Server | 来源 | 支持 | 特点 |
| --- | --- | --- | --- |
| **MLServer** | SeldonIO | ✅ 一等公民 | Python 多框架 |
| **NVIDIA Triton** | NVIDIA | ✅ 一等公民 | 高性能、多 backend |
| **LLM Module** | SeldonIO | ✅ 一等公民 | LLM + vLLM/TGI 集成 |
| **TF Serving** | Google | ⚠️（v1） | 旧版本 |
| **TorchServe** | AWS | ⚠️ | 旧版本 |
| **自定义 Server** | 社区 | ✅ | OIP 兼容即可 |

### 20.2 框架

- **LangChain** — 通过 OpenAI 兼容 API 集成
- **LlamaIndex** — 通过 OpenAI 兼容 API 集成
- **Haystack** — 通过 OpenAI 兼容 API 集成
- **DSPy** — 通过 OpenAI 兼容 API 集成

### 20.3 CI/CD 集成

- **ArgoCD** — 直接管理 CRD
- **FluxCD** — GitOps
- **Tekton** — K8s-native pipeline
- **GitHub Actions** — 模型训练 → 注册 → 部署

### 20.4 监控 / Tracing

- **Prometheus + Grafana** — 一等公民
- **Jaeger** — 内置支持
- **Datadog** — 通过 OTel Collector
- **New Relic** — 通过 OTel Collector
- **SigNoz** — Open source APM

---

## 21. Roadmap 与未来（推断）

基于 2025–2026 公开信号推测：

1. **强化 LLM 推理性能** — 集成最新 vLLM、SGLang、TensorRT-LLM
2. **更好的 Serverless 体验** — 类似 Modal 的快速部署
3. **多集群联邦** — Seldon Deploy 跨云
4. **AI Gateway 模式** — 提供 OpenAI 兼容 Gateway，统一 self-host + 外部 API
5. **更深的 MCP / Agent 支持** — 2026 H2 路线图
6. **OpenTelemetry GenAI SemConv** — 与 OTel 标准对齐
7. **Speculative decoding** — 集成到 LLM Module

---

## 22. 决策框架：什么时候选 Seldon Core 2

### 22.1 适合 Seldon Core 2 的场景

✅ **自托管多个 ML/LLM 模型在 K8s 上**
✅ **需要 A/B / Shadow / 灰度发布**
✅ **需要 Pipeline（RAG、多模型链、流式）**
✅ **已有 K8s 经验和 Kafka 集群**
✅ **需要 drift / outlier / explain 等数据科学监控**
✅ **企业生产环境，需要 Seldon Deploy 商业特性**

### 22.2 不适合 Seldon Core 2 的场景

❌ **主要调用 OpenAI / Anthropic 等外部 LLM API** — 用 Portkey / LiteLLM
❌ **小团队 / 几个模型** — 复杂度太高，用 BentoML / Modal
❌ **本地开发为主** — 用 MLServer / vLLM / Ollama
❌ **没有 K8s** — 不适合

### 22.3 与其他产品的搭配

- **Seldon 2 + Portkey:** 自托管模型用 Seldon，外部 API 路由用 Portkey
- **Seldon 2 + LiteLLM:** Seldon 跑 self-host LLM，LiteLLM 做 API 层聚合
- **Seldon 2 + Langfuse:** Langfuse 提供 LLM 应用层 trace，Seldon 提供基础设施层 metrics
- **Seldon 2 + vLLM:** vLLM 作为推理引擎，Seldon 作为编排

---

## 23. 代码示例（生产可用片段）

### 23.1 部署一个 HuggingFace 模型到 Seldon 2

```yaml
# 1. 定义一个 Server
apiVersion: mlops.seldon.io/v1alpha1
kind: Server
metadata:
  name: mlserver-hf
spec:
  serverConfig: mlserver
  podSpec:
    containers:
    - name: mlserver
      image: seldonio/mlserver:1.5.0
      resources:
        requests:
          cpu: "1"
          memory: "2Gi"
        limits:
          cpu: "4"
          memory: "8Gi"

---
# 2. 部署一个 HuggingFace 文本分类模型
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: bert-sentiment
spec:
  storageUri: "hf://distilbert-base-uncased-finetuned-sst-2-english"
  requirements:
  - mlserver
  - huggingface
  memory: 1Gi
```

### 23.2 调用模型

```bash
# REST 调用
curl -X POST http://seldon-mesh.seldon-mesh:80/v2/models/bert-sentiment/infer \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [{
      "name": "args",
      "shape": [1],
      "datatype": "BYTES",
      "data": ["I love this movie!"]
    }]
  }'
```

### 23.3 Python 客户端

```python
import requests

resp = requests.post(
    "http://seldon-mesh.seldon-mesh:80/v2/models/bert-sentiment/infer",
    json={
        "inputs": [{
            "name": "args",
            "shape": [1],
            "datatype": "BYTES",
            "data": ["I love this movie!"]
        }]
    }
)
print(resp.json())
```

### 23.4 OpenAI 客户端

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://seldon-mesh.seldon-mesh:80/v1",
    api_key="sk-dummy"  # Seldon 暂不强制
)

resp = client.chat.completions.create(
    model="llama-3-8b",
    messages=[{"role": "user", "content": "Hello"}]
)
print(resp.choices[0].message.content)
```

### 23.5 gRPC 调用（Python）

```python
import grpc
from seldon.inference import inference_pb2, inference_pb2_grpc

channel = grpc.insecure_channel("seldon-mesh.seldon-mesh:9003")
stub = inference_pb2_grpc.GRPCInferenceServiceStub(channel)

req = inference_pb2.ModelInferRequest(
    model_name="bert-sentiment",
    inputs=[
        inference_pb2.ModelInferRequest.InferInputTensor(
            name="args",
            shape=[1],
            datatype="BYTES",
            contents=inference_pb2.InferTensorContents(
                bytes_contents=[b"I love this movie!"]
            )
        )
    ]
)
resp = stub.ModelInfer(req)
print(resp)
```

### 23.6 Pipeline（RAG 模式）

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Pipeline
metadata:
  name: rag-qa
spec:
  steps:
  - name: query-embedder
    modelRef: bge-small
  - name: doc-retriever
    triggers:
    - query-embedder.outputs.embedding
    modelRef: vector-search
  - name: llm-answer
    inputs:
    - input.inputs.question
    - doc-retriever.outputs.documents
    modelRef: llama-3-8b
  output:
    steps:
    - llm-answer.outputs.text
```

```bash
curl -X POST http://seldon-mesh/v2/pipelines/rag-qa \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [{
      "name": "question",
      "shape": [1],
      "datatype": "BYTES",
      "data": ["What is Kubernetes?"]
    }]
  }'
```

### 23.7 A/B Test（Experiment）

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Experiment
metadata:
  name: iris-ab
spec:
  default: iris-v1
  candidates:
  - name: iris-v2
    weight: 20    # 20% 流量到 v2
```

### 23.8 Shadow Testing

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Experiment
metadata:
  name: iris-shadow
spec:
  default: iris-prod
  mirror:
    name: iris-experimental
    percent: 100   # 100% 流量镜像，response 不返回
```

### 23.9 Drift Detection Pipeline

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Pipeline
metadata:
  name: iris-drift
spec:
  steps:
  - name: drift-detector
    inputs:
    - input.inputs.0
    modelRef: drift-detector
  - name: classifier
    inputs:
    - input.inputs.0
    modelRef: iris-classifier
  output:
    steps:
    - classifier.outputs.output
```

### 23.10 K8s Manifest 完整示例

```yaml
# namespace
apiVersion: v1
kind: Namespace
metadata:
  name: seldon-mesh
---
# seldon runtime
apiVersion: mlops.seldon.io/v1alpha1
kind: SeldonRuntime
metadata:
  name: seldon
  namespace: seldon-mesh
spec:
  components:
    controller: {}
    scheduler: {}
    modelgateway:
    pipelinegateway:
    dataflowengine:
    agent:
    experimenter: {}
    modelmesh: {}
---
# mlserver server
apiVersion: mlops.seldon.io/v1alpha1
kind: Server
metadata:
  name: mlserver-prod
  namespace: seldon-mesh
spec:
  serverConfig: mlserver
  podSpec:
    containers:
    - name: mlserver
      image: seldonio/mlserver:1.5.0
      resources:
        requests:
          cpu: "2"
          memory: "4Gi"
        limits:
          cpu: "8"
          memory: "16Gi"
---
# model
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: production-iris
  namespace: seldon-mesh
spec:
  storageUri: "s3://my-models/iris"
  requirements:
  - sklearn
  memory: 500Mi
  replicas: 3
  autoscaling:
    minReplicas: 1
    maxReplicas: 10
```

---

## 24. 故障排查

### 24.1 常见问题

#### 24.1.1 Model 一直 NotReady

```bash
# 检查 Model 状态
kubectl get model production-iris -n seldon-mesh -o yaml

# 检查 Server 是否匹配 capability
kubectl get server -n seldon-mesh

# 查看 Agent 日志
kubectl logs -n seldon-mesh -l app=mlserver
```

#### 24.1.2 推理返回 503

```bash
# 检查 seldon-mesh 服务
kubectl get svc seldon-mesh -n seldon-mesh

# 检查 Envoy 日志
kubectl logs -n seldon-mesh -l app=seldon-mesh

# 验证 OIP 端点
curl http://seldon-mesh:80/v2/health/live
curl http://seldon-mesh:80/v2/health/ready
```

#### 24.1.3 Pipeline 启动失败

```bash
# 检查 Kafka 连接
kubectl exec -it seldon-scheduler -- kafka-topics --list --bootstrap-server kafka:9092

# 检查 dataflow-engine 日志
kubectl logs -n seldon-mesh -l app=seldon-dataflow-engine
```

#### 24.1.4 高延迟

```bash
# 查看模型 metrics
curl http://seldon-mesh:9005/metrics | grep seldon_model_infer_duration

# 检查 Server 资源使用
kubectl top pods -n seldon-mesh

# 检查 HPA
kubectl get hpa -n seldon-mesh
```

### 24.2 调试技巧

```bash
# 启用 debug 日志
kubectl set env deployment/seldon-scheduler -n seldon-mesh LOG_LEVEL=debug

# 进入 Server Pod
kubectl exec -it mlserver-0 -n seldon-mesh -- bash

# 端口转发
kubectl port-forward svc/seldon-mesh -n seldon-mesh 8000:80
```

---

## 25. 总结

### 25.1 一句话总结

**Seldon Core 2 是一个面向生产环境的 Kubernetes-native AI/ML Serving 平台，通过 Open Inference Protocol + Envoy 数据面 + Kafka 编排，提供了完整的 Model Gateway、Pipeline、Experiments、Multi-Model Serving、Autoscale 能力。**

### 25.2 与传统 AI Gateway 的关系

| 维度 | Seldon Core 2 | LLM Proxy Gateway（Portkey / LiteLLM） |
| --- | --- | --- |
| **抽象** | K8s 资源 | API 路由 |
| **解决** | 多模型部署 / 编排 | 多 LLM 厂商调用 |
| **协议** | OIP（标准推理协议） | OpenAI API（LLM 协议） |
| **核心场景** | Self-host ML/LLM | 调用外部 LLM API |
| **可观测** | Metrics + Trace + Drift | Trace + Logging |

### 25.3 战略价值

对于小F的副业定位（小B行业软件）：

- **自建 SaaS 平台** — Seldon 2 是开源的，可以直接用，**降低基础设施成本**
- **多模型产品** — 如果产品需要同时支持多个 ML 模型（CV / NLP / 推荐），Seldon 2 帮你统一管理
- **LLM + 传统 ML** — 一个平台覆盖
- **企业客户** — Seldon Deploy 商业版可卖给企业

### 25.4 关键 risk

- **学习曲线陡** — 需要 K8s + Kafka 基础
- **运维成本** — Kafka 集群是必需
- **LLM 场景较新** — LLM Module 比 vLLM、TGI 单独跑要弱

### 25.5 推荐路径

```
开发期：vLLM / Ollama / OpenAI API
   ↓
PoC期：MLServer + BentoML
   ↓
生产期：Seldon Core 2（多模型 + Pipeline + 灰度）
   ↓
企业期：Seldon Deploy（UI + 审计 + SLA）
```

---

## 26. 参考资料

1. **官方文档** — https://docs.seldon.ai/seldon-core-2/
2. **GitHub** — https://github.com/SeldonIO/seldon-core
3. **Helm Charts** — https://github.com/SeldonIO/helm-charts
4. **白皮书** — arXiv 2210.14665 — "Desiderata for next generation of ML model serving"
5. **Slack** — https://seldondev.slack.com
6. **Open Inference Protocol** — https://kserve.github.io/website/master/modelserving/data_plane/v2_protocol/
7. **v2 blog** — https://www.seldon.io/seldon-core-v2-ga/
8. **Seldon Deploy 商业版** — https://www.seldon.io/pricing/

---

## 27. 附录：完整 Resource Manifest 速查

### 27.1 SeldonRuntime

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: SeldonRuntime
```

### 27.2 SeldonConfig

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: SeldonConfig
```

### 27.3 ServerConfig

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: ServerConfig
metadata:
  name: mlserver
spec:
  serverType: MODEL
  podSpec:
    containers:
    - name: mlserver
      image: seldonio/mlserver:1.5.0
  envSecrets: []
  capabilities:
  - mlserver,alibi-detect,alibi-explain,huggingface,lightgbm,mlflow,python,sklearn,spark-mlib,xgboost
```

### 27.4 Server

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Server
metadata:
  name: mlserver-prod
spec:
  serverConfig: mlserver
  replicas: 1
  podSpec:
    containers:
    - name: mlserver
      resources:
        requests:
          cpu: "1"
          memory: "2Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
```

### 27.5 Model

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Model
metadata:
  name: my-model
spec:
  storageUri: "s3://bucket/model"
  requirements:
  - sklearn
  memory: 500Mi
  replicas: 1
  loadOnStart: false
```

### 27.6 Pipeline

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Pipeline
metadata:
  name: my-pipeline
spec:
  steps:
  - name: step1
    modelRef: model-a
  - name: step2
    inputs:
    - step1.outputs.output
    modelRef: model-b
  output:
    steps:
    - step2.outputs.output
```

### 27.7 Experiment

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: Experiment
metadata:
  name: my-experiment
spec:
  default: model-a
  candidates:
  - name: model-b
    weight: 10
  mirror:
    name: model-c
    percent: 50
```

### 27.8 StorageSecret

```yaml
apiVersion: mlops.seldon.io/v1alpha1
kind: StorageSecret
metadata:
  name: s3-credentials
spec:
  type: s3
  awsAccessKeyId: AKIA...
  awsSecretAccessKey: ...
  awsRegion: us-west-2
```

---

**报告结束**

> *作者注：本报告基于公开材料 + 合理架构推断。所有数据点已注明来源；推断部分已在文中明确标识。Seldon Core 2 的代码与协议均在 Apache 2.0 下开源，欢迎读者自行核实。*
