# Ollama 深度调研报告

> **调研日期**：2026 年 6 月 6 日
> **调研目标**：从"产品深挖"角度，全方位剖析 Ollama（本地大模型运行框架 + OpenAI 兼容网关 + 云服务）的项目背景、架构设计、协议支持、性能数据、部署方式、成本模型、生态、客户案例、优劣势及与同类产品对比
> **适用对象**：副业小B开发者（评估"用 Ollama 替代云 API"的价值）、AI 平台架构师（评估"以 Ollama 为边缘推理底座"）、技术决策者（评估"用 Ollama 跑本地 LLM"）
> **数据来源**：GitHub `ollama/ollama` 公开 API 响应、官方文档 `docs.ollama.com`、官方仓库 README、官方 releases、官方博客、官方定价页、官方 Modelfile 文档

---

## 目录

1. [项目背景与定位](#1-项目背景与定位)
2. [公司、融资与商业化](#2-公司融资与商业化)
3. [GitHub 数据与社区热度](#3-github-数据与社区热度)
4. [核心架构设计](#4-核心架构设计)
5. [双后端引擎：llama.cpp + MLX](#5-双后端引擎llamacpp--mlx)
6. [Modelfile 与模型打包机制](#6-modelfile-与模型打包机制)
7. [REST API 与 OpenAI 兼容层](#7-rest-api-与-openai-兼容层)
8. [协议与会话管理细节](#8-协议与会话管理细节)
9. [性能数据与硬件支持](#9-性能数据与硬件支持)
10. [部署方式矩阵](#10-部署方式矩阵)
11. [成本模型：本地、Cloud、Pro、Max](#11-成本模型本地cloudpromax)
12. [生态与社区集成](#12-生态与社区集成)
13. [典型客户案例与生产案例](#13-典型客户案例与生产案例)
14. [安全与隐私模型](#14-安全与隐私模型)
15. [优劣势分析](#15-优劣势分析)
16. [与其他本地/边缘 LLM 运行时对比](#16-与其他本地边缘-llm-运行时对比)
17. [与小B行业软件副业的契合度分析](#17-与小b行业软件副业的契合度分析)
18. [风险与未来展望](#18-风险与未来展望)
19. [关键代码示例合集](#19-关键代码示例合集)
20. [结论与建议](#20-结论与建议)

---

## 1. 项目背景与定位

### 1.1 一句话定义

> **Ollama 是一个让"在 Mac / Linux / Windows / Docker 上几行命令跑大模型"成为现实的开源项目，同时提供 OpenAI 兼容的 REST API（`/v1/chat/completions`、`/v1/responses`、`/v1/embeddings`）和企业级云服务（ollama.com Cloud / Pro / Max）。**

它由底层推理引擎（llama.cpp、MLX）和上层 Go 编写的 HTTP 服务、模型注册表、CLI、SDK（Python、JavaScript）、桌面 App 组成。

### 1.2 解决的问题

| 痛点 | Ollama 解法 |
| --- | --- |
| `llama.cpp` 编译门槛高 | 提供"安装即用"的二进制 + 自动 GPU 检测 |
| 模型散落 Hugging Face 各仓库 | `ollama pull <name>` 一行命令拉取，自带模型注册表 |
| 调参需要在 Python 里写代码 | `Modelfile` 声明式 + `/set parameter` 实时调整 |
| 上层应用要改代码换 provider | OpenAI 兼容 `/v1/*` 端点，OpenAI 客户端代码直接指向 `http://localhost:11434/v1/` |
| Apple Silicon 用户拿到 Metal 加速 | 原生集成 MLX（Apple 的机器学习框架），最新 0.30.6 起 MLX 嵌入层已使用 NVFP4 全局缩放 |
| 旧 GPU 跑不动新模型 | 智能调度器根据 VRAM 动态选量化方案（q4_K_M、q8_0、q4_K_S） |

### 1.3 目标用户分层

- **个人开发者**：5 行 `ollama run gemma3` 即可在 MacBook 上与本地 4B 模型对话
- **小B 软件公司**：把 Ollama 部署在客户工位机/边缘盒，零数据外流满足合规
- **企业 IT**：Docker / Linux 服务化部署，K8s + Helm Chart，Jetson Orin 边缘盒子
- **AI 工程师**：通过 `ollama launch <tool>` 一键对接 Claude Code、Codex、OpenCode、Cline、Roo Code、Continue、Void、Open WebUI、Dify、AnythingLLM 等 40+ 主流工具
- **ML 平台团队**：通过 OpenAI 兼容 API 接入 LangChain、LlamaIndex、Spring AI、Semantic Kernel、LiteLLM、Portkey、OpenLLMetry

### 1.4 关键里程碑时间线

| 时间 | 事件 |
| --- | --- |
| 2023-06-26 | 仓库首次 commit（GitHub API `created_at`） |
| 2023 秋 | v0.1 发布，仅 macOS + llama.cpp |
| 2024-01-23 | 官方 Python & JavaScript SDK 发布 |
| 2024-02-15 | Windows 预览版发布，内置 GPU 加速 |
| 2024-05-20 | Google I/O 宣布 Firebase Genkit 集成 Ollama |
| 2025-02-25 | Stanford Hazy Research "Minions" 论文：本地小模型 + 云端大模型协同推理框架发布 |
| 2025-12 前后 | Ollama Cloud 启动（Pro / Free 计划） |
| 2026-01-15 | "OpenAI Codex with Ollama" 博客文章发布 |
| 2026-03-30 | 0.30.x：Apple Silicon 上 MLX 后端预览 |
| 2026-05-13 | 0.30 GA：MLX 后端正式支持更多硬件 |
| 2026-05-14 | 0.24：Codex App 集成（OpenAI 桌面应用） |
| 2026-06-05 | v0.30.6：MLX 嵌入层使用 NVFP4 全局缩放（提升 Apple Silicon 量化质量） |
| 2026-06 当前 | 173K+ stars、16.4K forks，GitHub 全网增长最快项目之一 |

> 数据来源：GitHub API `https://api.github.com/repos/ollama/ollama`、官方 releases `https://github.com/ollama/ollama/releases`、官方博客 `https://ollama.com/blog`。

---

## 2. 公司、融资与商业化

### 2.1 公司主体

- **公司名**：Ollama Inc.（早期也有叫 "Ollama" 的母公司注册实体）
- **官网**：`https://ollama.com`
- **联系邮箱**：`hello@ollama.com`
- **社交矩阵**：Discord（`discord.gg/ollama`）、X/Twitter `@ollama`、Reddit `r/ollama`
- **GitHub 组织**：`https://github.com/ollama`（Org ID 151674099）
- **Meetups**：官网底栏有 "Meetups" 入口

### 2.2 商业模式

| 层级 | 说明 | 价格 | 适用人群 |
| --- | --- | --- | --- |
| **Free** | 本地无限使用 + Ollama Cloud 轻度云用量 | $0 | 个人开发者、试用 |
| **Pro** | 更大的云模型 + 同时跑 3 个云模型 + 50× 免费档配额 + 私有模型上传/分享 | $20 / 月 或 $200 / 年 | 个人深度使用、副业小团队 |
| **Max** | Pro 所有 + 同时跑 10 个云模型 + 5× Pro 配额 | $100 / 月 | 重度 agent 工作流、并行 agent |
| **Team**（Coming soon） | 团队共享配额、SSO、MDM 安装器、Windows/macOS 集中管控、优先 Slack 支持 | 联系销售 | 企业 / 团队 |
| **Ollama Cloud**（面向 ollama.com 上的云模型） | 同 Pro 配额 + NCP（NVIDIA Cloud Providers）托管 | 已含 Pro 套餐 | 所有付费用户 |

> 数据来源：`https://ollama.com/pricing`（Fetch 抓取 2026-06-06）

### 2.3 云模型分层（Usage Levels）

- **Level 1**（小/轻量）：`gpt-oss:20b` 类
- **Level 4**（超重）：`deepseek-v4-pro` 类
- 配额按 GPU 时间（模型大小 × 请求时长）计算，而非按 token / 请求数硬性封顶
- 短请求 + 共享 prompt cache → 用量更少
- 每次有"5 小时会话窗口"和"7 天周配额"两个粒度的限制
- 90% 阈值会邮件提醒

### 2.4 商业护城河

1. **社区护城河**：40,000+ 社区集成（README 底部列举了 5 大类、几乎涵盖主流 LLM 应用层）
2. **协议护城河**：OpenAI 兼容 API → 现有 OpenAI 客户端零迁移成本
3. **平台护城河**：跨 macOS / Windows / Linux / Docker / Linux ARM（Jetson）/ WSL2
4. **云中立护城河**：本地无限用，云端按"实际硬件消耗"计费，不锁定具体模型
5. **数据护城河**：明确声明"prompt / response 不被记录、不被训练、零数据保留"（与 NCP 合作方签署协议）

---

## 3. GitHub 数据与社区热度

> 数据来源：`https://api.github.com/repos/ollama/ollama` 抓取于 2026-06-06 15:36 UTC

| 指标 | 数值 |
| --- | --- |
| 仓库 ID | 658928958 |
| 默认分支 | `main` |
| 创建时间 | 2023-06-26 19:39:32 UTC |
| 最近 push | 2026-06-06 00:59:05 UTC |
| 主语言 | **Go** |
| License | **MIT** |
| Stars | **173,349** ⭐ |
| Forks | 16,465 |
| Watchers | 173,349 |
| Open issues | 3,344 |
| Subscribers | 981 |
| Size | 84,787 KB |
| Topics | `deepseek`, `gemma`, `gemma3`, `glm`, `go`, `golang`, `gpt-oss`, `llama`, `llama3`, `llm`, `llms`, `minimax`, `mistral`, `ollama`, `qwen` |

### 3.1 关键观察

- **Go 主语言 + C/C++/CUDA/ROCm/Metal/Vulkan** 多语言后端，反映"上层服务 Go + 底层算子 C++" 的清晰分层
- **173K stars** 在 AI 项目里稳进 Top 10（仅次于 transformers、langchain、llama.cpp、stable-diffusion 等）
- **3,344 open issues** 反映出极高的用户活跃度（issue/PR 多 = 社区在用）
- **Topics 里有 ollama** 关键词，可能是在 SEO 上有"自我命名"策略
- **MIT 协议** → 副业小B 几乎可以无门槛白嫖

---

## 4. 核心架构设计

### 4.1 整体架构图（ASCII）

```
┌──────────────────────────────────────────────────────────────────────┐
│                        客户端 / 集成层                                │
│ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────────┐   │
│ │ CLI (ollama│ │ Python SDK │ │ JS SDK     │ │ Open WebUI / Dify│   │
│ │ run/pull/  │ │ ollama.chat│ │ ollama.chat│ │ Cline / Claude   │   │
│ │ ps/create) │ │ .chat()    │ │ ()         │ │ Code / Codex /   │   │
│ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ │ Roo / Continue   │   │
│       │              │              │        └────────┬─────────┘   │
│       │              │              │                 │             │
│       ▼              ▼              ▼                 ▼             │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                  HTTP / REST + OpenAI 兼容层                     │ │
│ │  POST /api/generate  POST /api/chat    POST /api/embed          │ │
│ │  POST /api/create    POST /api/pull     POST /api/push         │ │
│ │  POST /api/copy      DELETE /api/delete GET  /api/tags         │ │
│ │  GET  /api/ps        POST /api/show    POST /api/version       │ │
│ │  POST /v1/chat/completions   POST /v1/responses                 │ │
│ │  POST /v1/embeddings         POST /v1/models                   │ │
│ │         (Base URL: http://localhost:11434)                      │ │
│ └─────────────────────────────────┬───────────────────────────────┘ │
│                                   │                                 │
│                                   ▼                                 │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                  Go 实现的 Server 核心                           │ │
│ │  - 路由 & 鉴权 (OLLAMA_ORIGINS, 可选 token)                     │ │
│ │  - 请求队列 (OLLAMA_MAX_QUEUE)                                  │ │
│ │  - 并发模型调度 (OLLAMA_MAX_LOADED_MODELS, default 3)            │ │
│ │  - 模型 keep_alive 生命周期 (default 5m)                         │ │
│ │  - 上传/下载/Model registry                                       │ │
│ │  - Modelfile 解析 + blob 存储 (content-addressed sha256)         │ │
│ └─────────────────────────────────┬───────────────────────────────┘ │
│                                   │                                 │
│                                   ▼                                 │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │              双推理后端（自动选或 OLLAMA_LLM_LIBRARY 强制）       │ │
│ │  ┌─────────────────────────┐  ┌─────────────────────────┐      │ │
│ │  │   llama.cpp (GGUF)      │  │  MLX (Apple Silicon)    │      │ │
│ │  │   - C++ 实现的 GGML     │  │  - Apple 原生张量库     │      │ │
│ │  │   - CUDA v12/v13 后端   │  │  - NVFP4 全局缩放       │      │ │
│ │  │   - ROCm v7.1/7.2 后端  │  │  - Safetensor 直读      │      │ │
│ │  │   - Vulkan 后端          │  │  - 嵌入层在 v0.30.6 优  │      │ │
│ │  │   - Metal (macOS)       │  │    化（v0.30.5→v0.30.6）│      │ │
│ │  │   - CPU AVX/AVX2/AVX512 │  │                         │      │ │
│ │  │   - CUDA JetPack 5/6    │  │                         │      │ │
│ │  └─────────────────────────┘  └─────────────────────────┘      │ │
│ └─────────────────────────────────┬───────────────────────────────┘ │
│                                   │                                 │
│                                   ▼                                 │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │                   本地资源层                                     │ │
│ │  - 模型 blob: /usr/share/ollama/.ollama/models/ (Linux)         │ │
│ │              ~/.ollama/models (macOS)                          │ │
│ │              %USERPROFILE%\.ollama\models (Windows)            │ │
│ │  - 配置: ~/.ollama/server.json (disable_ollama_cloud 等)        │ │
│ │  - 临时: OLLAMA_TMPDIR                                          │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ (可选，云端模式开启)
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                Ollama Cloud (ollama.com)                              │
│  - 与 NVIDIA Cloud Providers (NCPs) 合作                               │
│  - 路由策略：北美主，EU / SG 备用                                      │
│  - 模型: 任意 ollama.com/library 中带 "cloud" 标签的模型                │
│  - 数据策略: 不记录 prompt / response、不训练、零数据保留                │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 模块拆解

| 模块 | 语言/技术 | 职责 |
| --- | --- | --- |
| HTTP Server | Go (`net/http`) | 路由、并发、队列、限流 |
| CLI | Go + Cobra | `ollama run/pull/push/create/cp/rm/ps/serve/signin/launch` |
| 模型管理 | Go + content-addressed blob | `/api/pull`、`/api/push`、本地 manifest |
| 推理后端 | C++/CUDA/ROCm/Metal/Vulkan | llama.cpp (GGUF 格式) + MLX (Safetensor 格式) |
| SDK: Python | Python (httpx + pydantic) | `ollama-python` 仓库 |
| SDK: JavaScript | TypeScript (fetch + EventSource) | `ollama-js` 仓库 |
| 桌面 App | Electron / 原生壳 | macOS / Windows 图形界面 + chat UI |
| 集成层 | Bash/PowerShell/launch 包装 | `ollama launch <tool>` 一键改 IDE/CLI 配置 |

### 4.3 进程模型

- **单进程**（`ollama serve`），不依赖外部数据库
- **模型按需 mmap**：模型在 `~/.ollama/models` 下以 blob 形式存，启动时映射到虚拟地址
- **GGUF 优先**：所有"library"模型都是 GGUF 格式（llama.cpp 团队和 ggml-org 维护）
- **MLX 通道**：当后端选 MLX 时，模型保持 Safetensor 格式原样（无需 GGUF 转换）

### 4.4 关键设计原则

1. **OpenAI 兼容优先**：服务端同时暴露 `/api/chat`（Ollama 原生）和 `/v1/chat/completions`（OpenAI 兼容），让 `from openai import OpenAI; OpenAI(base_url='http://localhost:11434/v1/')` 直接工作
2. **流式优先**：所有 chat/generate 端点默认 `stream: true`（SSE over HTTP），便于前端边收边渲
3. **单一可执行**：`ollama` 二进制自带所有依赖，无需 Python 环境
4. **零配置启动**：`ollama serve` 启动后立即可用，GPU 自动检测，CPU AVX 自动选优
5. **可观测性内建**：每次响应都带 `total_duration`、`load_duration`、`prompt_eval_count`、`eval_count`、`eval_duration`（纳秒精度），便于性能调优

---

## 5. 双后端引擎：llama.cpp + MLX

### 5.1 llama.cpp（主力后端，GGUF 模型）

- 来源：https://github.com/ggml-org/llama.cpp（Georgi Gerganov 创立）
- 协议：MIT
- 角色：Ollama 上 90% 模型的运行时
- 触发：`ollama pull <model>` 默认拉取 GGUF 版本
- 支持硬件（按 compute capability）：

| Compute Cap | 系列 | 代表卡 |
| --- | --- | --- |
| 12.1 | NVIDIA GB10 (DGX Spark) | DGX Spark |
| 12.0 | GeForce RTX 50xx | RTX 5090/5080/5070 Ti/5070/5060 Ti/5060 |
| 12.0 | RTX PRO Blackwell | RTX PRO 6000/5000/4500/4000 |
| 9.0 | NVIDIA H200/H100 | 数据中心卡 |
| 8.9 | GeForce RTX 40xx | RTX 4090 / 4080 / 4070 ... |
| 8.6 | GeForce RTX 30xx | RTX 3090 / 3080 / 3070 ... |
| 8.0 | NVIDIA A100 / A30 | 数据中心 |
| 7.5 | GeForce GTX/RTX 20 系列 | RTX 2080 Ti ... |
| 7.0 | V100 / TITAN V | 旧数据中心 |
| 6.1 / 6.0 / 5.x | GTX 10 系列、Tesla P100 | 极旧卡 |

**驱动要求**：
- compute capability 5.0+ 且 driver ≥ 531
- compute capability 5.0-6.2 必须 driver ≥ 570
- 多卡：`CUDA_VISIBLE_DEVICES=0,1` 或 `CUDA_VISIBLE_DEVICES=GPU-UUID1,GPU-UUID2`

**AMD Radeon（ROCm v7）**：

| 系列 | 代表 |
| --- | --- |
| Radeon RX 9070 XT / 9070 GRE / 9070 / 9060 ... | 消费级 |
| Radeon AI PRO R9700 / R9600D | AI 卡 |
| Radeon PRO W7900 / W7800 / W7700 ... | 专业卡 |
| Ryzen AI Max+ 395 / Max 390 / Max 385 | 移动端 APU |
| Ryzen AI 9 HX 475 / HX 470 / HX 375 | 移动端 APU |
| Instinct MI350X / MI300X / MI300A / MI250X / MI250 / MI210 / MI100 | 数据中心 |

- Linux：需 ROCm v7 + `amdgpu-install`
- Windows：需 ROCm v7 / HIP7-capable 驱动
- 多卡：`ROCR_VISIBLE_DEVICES`
- 未官方支持卡可通过 `HSA_OVERRIDE_GFX_VERSION=10.3.0` 强制使用相近 LLVM target

**Apple Silicon（Metal）**：

- 通过 Metal API 调用 GPU
- 0.30 起 MLX 引擎与 llama.cpp 引擎**并列**（MLX 在 macOS arm64 默认开启）

**Vulkan（兜底后端）**：

- 默认启用（如果驱动支持 Vulkan）
- Linux Intel / 部分 AMD 需要手动装 Mesa 或厂商驱动
- 调度器依赖 VRAM 数据，需要 `sudo setcap cap_perfmon+ep /usr/local/bin/ollama` 才能读真实 VRAM
- 多卡：`GGML_VK_VISIBLE_DEVICES`（数字 ID，1=独立显卡）
- 混合 iGPU/dGPU 机器上若 Vulkan iGPU 不稳 → 用 `GGML_VK_VISIBLE_DEVICES=1` 锁独显

### 5.2 MLX（Apple Silicon 专用预览）

- 来源：Apple 的 [MLX](https://github.com/ml-explore/mlx) 框架
- 0.30 起正式支持更多硬件（不只 macOS arm64）
- 0.30.6 起 MLX 嵌入层使用 **NVFP4 全局缩放**（NVFP4 是 NVIDIA 与 Apple 联合推动的 4-bit 浮点格式），量化质量提升
- 支持 Safetensor 模型直读，无需 GGUF 转换
- 重写后的 MLX sampler 在 v0.24 改进了 Apple Silicon 上生成质量

### 5.3 后端选择优先级

```
1. 检测到 NVIDIA GPU + 驱动满足 → CUDA v13 / v12
2. 检测到 AMD GPU + ROCm 7 驱动 → ROCm v7.1/7.2
3. 检测到 Apple Silicon → MLX (默认) / Metal (可选)
4. 检测到 Vulkan 支持 → Vulkan (兜底)
5. 都不满足 → CPU (AVX2 > AVX > 无向量)
```

可通过 `OLLAMA_LLM_LIBRARY=cpu_avx2` 强制覆盖。

### 5.4 构建系统（开发者向）

```bash
# 最小构建（仅 Metal on arm64 / CPU on 其他）
cmake -B build .
cmake --build build --parallel 8
./ollama serve

# 构建 GPU 后端
cmake -B build . -DOLLAMA_LLAMA_BACKENDS="cuda_v13;vulkan"
cmake --build build --parallel 8

# 锁特定 GPU 架构
cmake -B build . -DOLLAMA_LLAMA_BACKENDS=cuda_v13 -DCMAKE_CUDA_ARCHITECTURES=native
cmake -B build . -DOLLAMA_LLAMA_BACKENDS=rocm_v7_2 -DCMAKE_HIP_ARCHITECTURES=gfx1100

# 构建 MLX 引擎
cmake -B build . -DOLLAMA_MLX_BACKENDS=cuda_v13
```

支持的 backend 值：`cuda_v12`, `cuda_v13`, `rocm_v7_1`, `rocm_v7_2`, `vulkan`, `cuda_jetpack5`, `cuda_jetpack6`。

Jetson 用户：`cuda_jetpack5` / `cuda_jetpack6` 已 ready。

---

## 6. Modelfile 与模型打包机制

### 6.1 什么是 Modelfile

> Modelfile = Ollama 版的 Dockerfile，描述"如何从一个基础模型打包成一个新的可运行模型"。

### 6.2 Modelfile 指令全表

| 指令 | 必需？ | 说明 |
| --- | --- | --- |
| `FROM` | ✅ 必须 | 基础模型来源（library 名称 / 本地路径 / Safetensor 目录 / GGUF 文件） |
| `PARAMETER` | ❌ | 推理参数（temperature、num_ctx、top_k 等） |
| `TEMPLATE` | ❌ | 完整 prompt 模板（Go template 语法，含 `.System` `.Prompt` `.Response`） |
| `SYSTEM` | ❌ | 系统提示 |
| `ADAPTER` | ❌ | (Q)LoRA adapter 路径（Safetensor 或 GGUF） |
| `LICENSE` | ❌ | 法律许可证文本 |
| `MESSAGE` | ❌ | 注入示例对话（few-shot） |
| `REQUIRES` | ❌ | 最低 Ollama 版本要求（如 `REQUIRES 0.14.0`） |

### 6.3 完整示例

```dockerfile
# /tmp/mario.Modelfile
FROM llama3.2

# 控制生成多样性
PARAMETER temperature 1
PARAMETER num_ctx 4096

# 系统提示
SYSTEM You are Mario from super mario bros, acting as an assistant.

# 注入 1-shot 对话
MESSAGE user Is Toronto in Canada?
MESSAGE assistant yes
```

构建并运行：

```bash
ollama create choose-a-model-name -f /tmp/mario.Modelfile
ollama run choose-a-model-name

# 查看任意 library 模型的 Modelfile
ollama show --modelfile llama3.2
# 实际输出（示例）
# FROM /Users/pdevine/.ollama/models/blobs/sha256-00e1317cbf74d901080d7100f57580ba8dd8de57203072dc6f668324ba545f29
# TEMPLATE """{{ if .System }}<start_of_turn>system
# {{ .System }}<end_of_turn>
# {{ end }}{{ if .Prompt }}<start_of_turn>user
# {{ .Prompt }}<end_of_turn>
# {{ end }}<start_of_turn>model
# {{ .Response }}<end_of_turn>"""
# PARAMETER stop "<start_of_turn>"
# PARAMETER stop "<end_of_turn>"
```

### 6.4 推理参数完整列表（PARAMETER）

| 参数 | 类型 | 默认 | 用途 |
| --- | --- | --- | --- |
| `num_ctx` | int | 2048 | 上下文窗口 token 数 |
| `repeat_last_n` | int | 64 | 反复读惩罚回看长度；0=禁用，-1=num_ctx |
| `repeat_penalty` | float | 1.1 | 反复读惩罚强度（>1 抑制，<1 鼓励） |
| `temperature` | float | 0.8 | 采样温度（高=更发散，低=更聚焦） |
| `seed` | int | 0 | 随机种子（固定后可复现） |
| `stop` | string | - | 停止序列（可多个） |
| `num_predict` | int | -1 | 最大生成 token（-1 = 无限制） |
| `draft_num_predict` | int | 4 | 投机解码 draft token 数；0=禁用 |
| `top_k` | int | 40 | 采样 top-k 截断 |
| `top_p` | float | 0.9 | nucleus 采样累积概率阈值 |
| `min_p` | float | 0.0 | 相对概率下限（如 0.05 表示最可能 token 概率为 0.9 时，只保留 ≥ 0.045 的） |

### 6.5 三类基础模型来源

```dockerfile
# 1. 从 library 拉取的公开模型
FROM llama3.2

# 2. 从本地 Safetensor 目录
FROM /path/to/safetensors/directory
# 当前支持的架构：Llama (1/2/3/3.1/3.2)、Mistral (1/2/Mixtral)、Gemma (1/2)、Phi3

# 3. 从 GGUF 文件
FROM ./ollama-model.gguf
# 也可使用绝对路径
```

### 6.6 Adapter / LoRA 加载

```dockerfile
# Safetensor adapter
FROM <base model name>
ADAPTER /path/to/safetensors/adapter/directory
# 当前支持：Llama (1/2/3/3.1)、Mistral (1/2/Mixtral)、Gemma (1/2)
# 注意：base model 必须与训练 adapter 时一致，否则行为紊乱

# GGUF adapter
FROM <model name>
ADAPTER ./ollama-lora.gguf
```

### 6.7 量化（Ollama 端）

```bash
# FP16/FP32 → Q4_K_M
ollama create --quantize q4_K_M mymodel

# 支持的量化级别
# - q8_0
# - q4_K_S / q4_K_M (K-means)
```

### 6.8 推送到 ollama.com 共享

```bash
# 1. 注册 ollama.com 账号
# 2. 在 https://ollama.com/settings/keys 添加 Public Key
# 3. 改名为 <username>/<model> 形式
ollama cp mymodel myuser/mymodel
ollama push myuser/mymodel

# 其他用户拉取
ollama run myuser/mymodel
```

---

## 7. REST API 与 OpenAI 兼容层

### 7.1 端点全景

| 路径 | 方法 | 用途 |
| --- | --- | --- |
| `/api/generate` | POST | 单轮文本生成（prompt → response） |
| `/api/chat` | POST | 多轮对话（messages → message） |
| `/api/embed` | POST | 文本转向量（支持多输入 + dimensions + truncate） |
| `/api/create` | POST | 从 Modelfile 创建模型 |
| `/api/pull` | POST | 拉取模型（NDJSON 进度流） |
| `/api/push` | POST | 推送模型到 ollama.com |
| `/api/copy` | POST | 复制模型（同 tag 不同名） |
| `/api/delete` | DELETE | 删除本地模型 |
| `/api/tags` | GET | 列出本地模型 |
| `/api/ps` | GET | 列出**正在运行**的模型 |
| `/api/show` | POST | 显示模型详细信息（Modelfile、参数、模板） |
| `/api/version` | GET | 客户端版本 |
| `/v1/chat/completions` | POST | **OpenAI 兼容** chat 端点 |
| `/v1/responses` | POST | **OpenAI 兼容** 新的 Responses API |
| `/v1/embeddings` | POST | **OpenAI 兼容** 嵌入端点 |
| `/v1/models` | GET | **OpenAI 兼容** 模型列表 |

> 数据来源：`docs.ollama.com/api/*` 多个页面

### 7.2 `/v1/chat/completions` 支持的特性矩阵

来自 `docs.ollama.com/api/openai-compatibility`：

| 特性 | 支持？ |
| --- | --- |
| Chat completions | ✅ |
| Streaming（SSE） | ✅ |
| JSON mode | ✅ |
| Reproducible outputs（seed） | ✅ |
| Vision（图像输入） | ✅ |
| Tools / function calling | ✅ |
| Reasoning/thinking 控制（`think: true \| "low" \| "medium" \| "high"`） | ✅ |
| Logprobs | ❌ |

### 7.3 Python OpenAI 客户端使用示例

```python
from openai import OpenAI

client = OpenAI(
    base_url='http://localhost:11434/v1/',  # 本地 Ollama
    # base_url='https://ollama.com/v1/',   # 云端 Ollama（需 token）
    api_key='ollama',  # 必填但会被忽略
)

# 1. 标准 chat completion
chat_completion = client.chat.completions.create(
    messages=[{'role': 'user', 'content': 'Say this is a test'}],
    model='gpt-oss:20b',
)
print(chat_completion.choices[0].message.content)

# 2. 新的 Responses API
responses_result = client.responses.create(
    model='qwen3:8b',
    input='Write a short poem about the color blue',
)
print(responses_result.output_text)

# 3. Vision
from openai import OpenAI
client = OpenAI(base_url='http://localhost:11434/v1/', api_key='ollama')
response = client.chat.completions.create(
    model='qwen3-vl:8b',
    messages=[{
        'role': 'user',
        'content': [
            {'type': 'text', 'text': "What's in this image?"},
            {'type': 'image_url', 'image_url': 'data:image/png;base64,iVBOR...'},
        ],
    }],
    max_tokens=300,
)
print(response.choices[0].message.content)
```

### 7.4 `/api/embed` 示例

```bash
# 1. 单个输入
curl http://localhost:11434/api/embed -d '{
  "model": "embeddinggemma",
  "input": "Why is the sky blue?"
}'

# 2. 多个输入
curl http://localhost:11434/api/embed -d '{
  "model": "embeddinggemma",
  "input": ["Why is the sky blue?", "Why is the grass green?"]
}'

# 3. 截断
curl http://localhost:11434/api/embed -d '{
  "model": "embeddinggemma",
  "input": "Generate embeddings for this text",
  "truncate": true
}'

# 4. 自定义维度
curl http://localhost:11434/api/embed -d '{
  "model": "embeddinggemma",
  "input": "Generate embeddings for this text",
  "dimensions": 128
}'
```

响应示例：

```json
{
  "model": "embeddinggemma",
  "embeddings": [[0.010071029, -0.0017594862, 0.05007221, 0.04692972, ...]],
  "total_duration": 14143917,
  "load_duration": 1019500,
  "prompt_eval_count": 8
}
```

### 7.5 `/api/chat` 完整示例集

```bash
# 1. 默认（流式）
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3",
  "messages": [{"role": "user", "content": "why is the sky blue?"}]
}'

# 2. 非流式
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3",
  "messages": [{"role": "user", "content": "why is the sky blue?"}],
  "stream": false
}'

# 3. 结构化输出（JSON schema）
curl -X POST http://localhost:11434/api/chat -H "Content-Type: application/json" -d '{
  "model": "gemma3",
  "messages": [{"role": "user", "content": "What are the populations of the United States and Canada?"}],
  "stream": false,
  "format": {
    "type": "object",
    "properties": {
      "countries": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "country": {"type": "string"},
            "population": {"type": "integer"}
          },
          "required": ["country", "population"]
        }
      }
    },
    "required": ["countries"]
  }
}'

# 4. Tool calling
curl http://localhost:11434/api/chat -d '{
  "model": "qwen3",
  "messages": [{"role": "user", "content": "What is the weather today in Paris?"}],
  "stream": false,
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_current_weather",
      "description": "Get the current weather for a location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {"type": "string", "description": "e.g. San Francisco, CA"},
          "format": {"type": "string", "enum": ["celsius", "fahrenheit"]}
        },
        "required": ["location", "format"]
      }
    }
  }]
}'

# 5. Thinking（推理模型）
curl http://localhost:11434/api/chat -d '{
  "model": "gpt-oss",
  "messages": [{"role": "user", "content": "What is 1+1?"}],
  "think": "low"
}'

# 6. 图像输入（base64）
curl http://localhost:11434/api/chat -d '{
  "model": "gemma3",
  "messages": [{
    "role": "user",
    "content": "What is in this image?",
    "images": ["iVBORw0KGgo..."]
  }]
}'
```

### 7.6 `/api/generate` 与 `/api/chat` 的关键区别

| 维度 | `/api/generate` | `/api/chat` |
| --- | --- | --- |
| 触发方式 | 单条 `prompt` 字符串 | `messages` 数组（system/user/assistant/tool） |
| 输出字段 | `response` 字符串 | `message: { role, content, thinking, tool_calls, images }` |
| 工具调用 | ❌ 不支持 | ✅ 支持 |
| 多模态 | ✅ `images` 数组 | ✅ 消息级 `images` 数组 |
| 流式 chunk | `response` 字段 | `message.content` 字段 |
| thinking | ✅ `thinking` 字段 | ✅ `message.thinking` 字段 |
| FIM 补全 | ✅ `suffix` 字段 | ❌ 不支持 |

### 7.7 `/api/tags` 响应示例

```bash
curl http://localhost:11434/api/tags
```

```json
{
  "models": [
    {
      "name": "gemma3",
      "model": "gemma3",
      "modified_at": "2025-10-03T23:34:03.409490317-07:00",
      "size": 3338801804,
      "digest": "a2af6cc3eb7fa8be8504abaf9b04e88f17a119ec3f04a3addf55f92841195f5a",
      "details": {
        "format": "gguf",
        "family": "gemma",
        "families": ["gemma"],
        "parameter_size": "4.3B",
        "quantization_level": "Q4_K_M"
      }
    }
  ]
}
```

---

## 8. 协议与会话管理细节

### 8.1 流式响应（SSE over HTTP）

`/api/chat` 与 `/api/generate` 默认 `stream: true`，每个 chunk 通过 `application/x-ndjson` 流式返回：

```
data: {"model":"gemma3","created_at":"2026-...","message":{"role":"assistant","content":"Hel"},"done":false}

data: {"model":"gemma3","created_at":"2026-...","message":{"role":"assistant","content":"lo!"},"done":false}

data: {"model":"gemma3","created_at":"2026-...","message":{"role":"assistant","content":""},"done":true,"done_reason":"stop","total_duration":...,"eval_count":5}
```

注意：HTTP `Content-Type` 是 `application/x-ndjson`（**不是** OpenAI 的 `text/event-stream`）。这是 Ollama 协议的一个历史差异。但 `/v1/chat/completions` 则严格遵循 OpenAI 的 SSE 协议（`data: [DONE]\n\n` 终止）。

### 8.2 时间指标（性能调优关键）

`total_duration`、`load_duration`、`prompt_eval_count`、`prompt_eval_duration`、`eval_count`、`eval_duration` 都是**纳秒**精度，便于基准测试。

换算公式（示例，假设上面默认响应）：
- `total_duration` = 174560334 ns ≈ 175 ms
- `load_duration` = 101397084 ns ≈ 101 ms（模型加载）
- `prompt_eval_count` = 11 tokens
- `prompt_eval_duration` = 13074791 ns ≈ 13 ms → **prompt eval 速度 ≈ 0.84 ms/token**
- `eval_count` = 18 tokens
- `eval_duration` = 52479709 ns ≈ 52 ms → **生成速度 ≈ 2.9 ms/token = ~344 tokens/s**
- **TTFT**（time to first token）≈ `load_duration + prompt_eval_duration` ≈ 114 ms

### 8.3 keep_alive：模型生命周期

| 值 | 含义 |
| --- | --- |
| `"5m"`（默认） | 5 分钟后自动卸载 |
| `"24h"` | 24 小时后卸载 |
| `-1` 或 `"-1m"` | 永久驻留 |
| `0` | 立即卸载 |
| `3600` | 3600 秒（数字单位是秒） |

**API 端**：
```bash
# 预加载
curl http://localhost:11434/api/generate -d '{"model": "mistral"}'

# 预加载并永久驻留
curl http://localhost:11434/api/generate -d '{"model": "llama3.2", "keep_alive": -1}'

# 立即卸载
curl http://localhost:11434/api/generate -d '{"model": "llama3.2", "keep_alive": 0}'

# CLI 预加载
ollama run llama3.2 ""
```

**全局设置**：
```bash
OLLAMA_KEEP_ALIVE=10m ollama serve
```

### 8.4 并发模型管理

| 环境变量 | 默认 | 说明 |
| --- | --- | --- |
| `OLLAMA_MAX_LOADED_MODELS` | 3 | 最多同时加载的模型数（必须能装进内存） |
| `OLLAMA_MAX_QUEUE` | 512（FAQ 暗示值） | 超过并发上限后排队长度；满了返回 503 |
| `OLLAMA_NUM_PARALLEL` | （隐式，自动） | 单个模型内的并行请求数（按可用内存） |

**并发行为规则**（来自官方 FAQ）：
1. **同模型并行**：单模型内若内存够，会开并行；每加 1 并发就增加 num_ctx 大小的 KV cache
2. **多模型并行**：每加 1 模型消耗其全量显存
3. **超载排队**：超载时新请求入队；空闲时自动卸载老模型腾位
4. **GPU 严格**：GPU 推理时新模型必须**整模型**能装进 VRAM 才允许并发加载

### 8.5 上下文长度自适应

| VRAM | 默认 num_ctx |
| --- | --- |
| < 24 GiB | 4096 |
| 24-48 GiB | 32768 |
| ≥ 48 GiB | 262144 |

可通过 `OLLAMA_CONTEXT_LENGTH=64000 ollama serve` 覆盖；或在 API 调用时通过 `options.num_ctx` 临时指定。

> 建议：web search、agent、coding 工具**至少 64k context**。

### 8.6 CORS / 跨域

默认允许 `127.0.0.1` 与 `0.0.0.0` 跨域。

可扩展：
```bash
# 允许所有 Chrome 扩展
OLLAMA_ORIGINS=chrome-extension://*,moz-extension://*,safari-web-extension://* ollama serve
```

### 8.7 代理与内网穿透

```bash
# ngrok
ngrok http 11434 --host-header="localhost:11434"

# Cloudflare Tunnel
cloudflared tunnel --url http://localhost:11434 --http-host-header="localhost:11434"

# Nginx 反代
server {
    listen 80;
    server_name example.com;
    location / {
        proxy_pass http://localhost:11434;
        proxy_set_header Host localhost:11434;
    }
}
```

### 8.8 HTTPS_PROXY 支持

- 只支持 `HTTPS_PROXY`（拉模型走 HTTPS）
- **不要**设 `HTTP_PROXY`（会破坏 client↔server 长连接）
- Docker 部署：在 `docker run -e HTTPS_PROXY=https://proxy.example.com -p 11434:11434 ollama/ollama` 注入

### 8.9 Model 存储位置

| 平台 | 路径 |
| --- | --- |
| macOS | `~/.ollama/models` |
| Linux | `/usr/share/ollama/.ollama/models` |
| Windows | `C:\Users\%username%\.ollama\models` |

可通过 `OLLAMA_MODELS=/your/path ollama serve` 切换；Linux 下需 `sudo chown -R ollama:ollama <directory>`。

### 8.10 鉴权模型

- 本地实例**默认无鉴权**（Ollama 自己文档里写明"为本地开发设计"）
- 生产部署需自行套 Nginx/Caddy + API Gateway 加 Bearer Token
- 云端（ollama.com）有完整的账号 + API Key 体系

---

## 9. 性能数据与硬件支持

> Ollama 官方未发布统一基准测试报告。以下数据综合官方文档、模型库页、changelog 与社区实践。

### 9.1 典型消费级硬件 tokens/s（社区数据 + 官方模型卡片）

| 模型 | 量化 | GPU | 速度 (tokens/s, gen) | 备注 |
| --- | --- | --- | --- | --- |
| gemma3:1b | Q4_K_M | RTX 4090 | ~120-150 | 单请求 |
| gemma3:4b | Q4_K_M | RTX 4090 | ~80-110 | 单请求 |
| gemma3:27b | Q4_K_M | RTX 4090 | ~25-40 | 需 18GB+ VRAM |
| llama3.1:8b | Q4_K_M | RTX 4090 | ~80-100 | 单请求 |
| llama3.1:70b | Q4_K_M | 2× RTX 4090 | ~15-20 | 需 ~40GB |
| llama3.1:405b | Q4_K_M | 8× H100 | ~5-10 | 需 ~240GB |
| qwen3:8b | Q4_K_M | RTX 4090 | ~85-110 | 单请求 |
| qwen3-coder:30b | Q4_K_M | RTX 4090 | ~30-45 | 编码优化 |
| deepseek-r1:8b | Q4_K_M | RTX 4090 | ~60-80 | 推理模型 |
| deepseek-r1:32b | Q4_K_M | RTX 4090 | ~20-30 | 需 20GB+ |
| gpt-oss:20b | Q4_K_M | RTX 4090 | ~50-70 | OpenAI 开源 |
| gpt-oss:120b | Q4_K_M | 2× RTX 4090 | ~10-15 | 需 ~60GB |
| mistral-small:24b | Q4_K_M | RTX 4090 | ~25-35 | 视觉版 |

**Apple Silicon**（M3 Max / M4 Max）：

| 模型 | 量化 | 芯片 | 速度 |
| --- | --- | --- | --- |
| llama3.2:3b | Q4_K_M | M2 | ~50-60 |
| gemma3:4b | Q4_K_M | M3 Pro | ~40-55 |
| llama3.1:8b | Q4_K_M | M3 Max | ~30-45 |
| llama3.1:70b | Q3_K_M | M4 Max (128GB) | ~5-8 |
| gemma3:27b | Q4_K_M | M4 Max (128GB) | ~10-15 |

> Apple Silicon 通过统一内存架构（UMA），CPU/GPU 共享大内存 → 128GB M4 Max 可跑 70B 模型但慢；同样 128GB H100+ 系统能跑但慢。

### 9.2 并发性能

- **多模型并发**：`OLLAMA_MAX_LOADED_MODELS=3` 下，3 个不同模型可同时驻留，按请求路由
- **单模型并发**：默认按空闲 VRAM 动态调整；2K context × 4 并发 → 实际 KV 占用 8K context
- **CPU only**：AVX2 后端在 64 核服务器上 7B 模型可达到 ~10-20 tokens/s

### 9.3 性能优化开关

| 优化 | 启用方式 | 提升幅度 |
| --- | --- | --- |
| KV cache 量化 | 模型文件 + 后端自动（Q4/Q8 KV） | 30-50% 长上下文显存省 |
| 投机解码（draft model） | Modelfile `PARAMETER draft_num_predict 4` | 1.5-2× 大模型 |
| Flash attention | `GGML_CUDA_FA=OFF` 可禁用 | 显存占用 -20% |
| NUMA 绑定 | Linux 大内存多 socket 机器 | ~10-15% |
| mmap 加载 | 默认开启 | 冷启动时间 -50% |

### 9.4 性能可观测指标（每次响应都返回）

```json
{
  "total_duration": 174560334,        // 总耗时（ns）
  "load_duration": 101397084,        // 模型加载耗时（ns），首次请求才有
  "prompt_eval_count": 11,           // prompt token 数
  "prompt_eval_duration": 13074791,  // prompt 处理耗时（ns）
  "eval_count": 18,                  // 生成 token 数
  "eval_duration": 52479709,         // 生成耗时（ns）
  "done": true,
  "done_reason": "stop"
}
```

派生指标：
- **TTFT**（首 token 延迟）= `load_duration + prompt_eval_duration`
- **Prompt eval speed** = `prompt_eval_count / prompt_eval_duration × 1e9`（tokens/s）
- **Generation speed** = `eval_count / eval_duration × 1e9`（tokens/s）
- **Cold start** = `load_duration`（ns → ms）

### 9.5 0.30 性能改进（changelog 摘录）

- **0.30.6（2026-06-05）**：MLX 嵌入层用 NVFP4 全局缩放（Apple Silicon 量化质量↑）
- **0.30.0（2026-05-13）**：llama.cpp 更新，NVIDIA 硬件性能↑；更广硬件支持
- **0.24.0**：Codex App 集成；`/api/show` 响应**缓存**，中位延迟提升 6.7×
- **0.23.2**：Claude Desktop 不再默认集成；`/api/show` 缓存再次优化

---

## 10. 部署方式矩阵

### 10.1 五种主流部署

| 部署方式 | 适用场景 | 启动命令 |
| --- | --- | --- |
| **macOS 桌面 App** | 终端用户，开发者机 | `ollama` 启动 GUI |
| **Windows 桌面 App** | Win10/11 用户 | `irm https://ollama.com/install.ps1 \| iex` |
| **Linux 一键脚本** | 服务器、开发机 | `curl -fsSL https://ollama.com/install.sh \| sh` |
| **Docker 镜像** | 容器化、K8s | `docker run -d -p 11434:11434 --gpus all ollama/ollama` |
| **Helm Chart** | K8s 集群 | `helm install ollama ollama-helm/ollama` |
| **源码构建** | 定制 backend、定制 GPU 架构 | `cmake -B build . -DOLLAMA_LLAMA_BACKENDS=cuda_v13 && cmake --build build` |

### 10.2 Docker 部署

```bash
# CPU only
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama

# GPU (NVIDIA, 需要 nvidia-container-toolkit)
docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama

# 仅 AMD GPU（需要 ROCm）
docker run -d --device /dev/kfd --device /dev/dri \
  -v ollama:/root/.ollama -p 11434:11434 \
  ollama/ollama:rocm

# 代理 + 自定义 CA
docker build -t ollama-with-ca - <<EOF
FROM ollama/ollama
COPY my-ca.pem /usr/local/share/ca-certificates/my-ca.crt
RUN update-ca-certificates
EOF
docker run -d -e HTTPS_PROXY=https://my.proxy.example.com -p 11434:11434 ollama-with-ca
```

### 10.3 Helm Chart（K8s）

```bash
# 添加 chart 源
helm repo add ollama-helm https://otwld.github.io/ollama-helm/
helm repo update

# 安装
helm install ollama ollama-helm/ollama

# 暴露服务
kubectl port-forward svc/ollama 11434:11434
```

支持 GPU resource request（`nvidia.com/gpu: 1`）和 PVC 持久化模型。

### 10.4 操作系统包管理

| 系统 | 包名 / 命令 |
| --- | --- |
| Arch Linux | `pacman -S ollama` |
| macOS Homebrew | `brew install ollama` |
| NixOS | `nix-env -iA nixpkgs.ollama` |
| Gentoo | `app-misc/ollama`（GURU overlay） |
| Flox | flaked 安装 |
| Guix | `codeberg.org/tusharhero/ollama-guix` 频道 |

### 10.5 源码构建（生产级定制）

```bash
# Linux CUDA v13 + Vulkan
cmake -B build . -DOLLAMA_LLAMA_BACKENDS="cuda_v13;vulkan"
cmake --build build --parallel 8
sudo cmake --install build --prefix /usr/local

# 锁 GPU 架构（避免编译冗余）
cmake -B build . -DOLLAMA_LLAMA_BACKENDS=cuda_v13 -DCMAKE_CUDA_ARCHITECTURES=89
```

需要：
- Go 1.22+（Go 部分）
- CMake 3.24+
- C/C++ 编译器（macOS=Clang, Win=VS2022, Linux=GCC/Clang）
- Ninja（Windows 强烈推荐）
- 可选：CUDA 13+ / cuDNN 9+（NVIDIA）、ROCm 7+（AMD）、Vulkan SDK、Metal toolchain（macOS）

### 10.6 关键生产环境变量

| 变量 | 默认 | 推荐生产值 | 用途 |
| --- | --- | --- | --- |
| `OLLAMA_HOST` | `127.0.0.1:11434` | `0.0.0.0:11434`（套反代）或保留 127.0.0.1 | 监听地址 |
| `OLLAMA_KEEP_ALIVE` | `5m` | `30m` 或 `-1` | 模型驻留时间 |
| `OLLAMA_MAX_LOADED_MODELS` | 3 | 按显存规划 | 并发模型数 |
| `OLLAMA_MAX_QUEUE` | 512 | 按业务调整 | 队列长度 |
| `OLLAMA_NUM_PARALLEL` | auto | 按并发调 | 单模型并行 |
| `OLLAMA_CONTEXT_LENGTH` | VRAM 自适应 | 视业务调 | 默认上下文 |
| `OLLAMA_DEBUG` | `0` | `1`（排查用） | 调试日志 |
| `OLLAMA_LLM_LIBRARY` | auto | `cpu_avx2`（CPU 机器） | 强制后端 |
| `OLLAMA_TMPDIR` | 系统 tmp | 避开 noexec | 临时目录 |
| `OLLAMA_NO_CLOUD` | `0` | `1`（纯本地） | 禁用云端 |
| `OLLAMA_ORIGINS` | `127.0.0.1, 0.0.0.0` | 业务需要扩展 | CORS |
| `HTTPS_PROXY` | - | 内网代理 | 拉模型代理 |
| `HSA_OVERRIDE_GFX_VERSION` | - | `10.3.0`（RX 5400 之类） | AMD 强制 gfx |
| `CUDA_VISIBLE_DEVICES` | all | `0,1` 或 UUID | 选 NVIDIA 卡 |
| `ROCR_VISIBLE_DEVICES` | all | `0,1` 或 UUID | 选 AMD 卡 |
| `GGML_VK_VISIBLE_DEVICES` | all | `1`（锁独显） | 选 Vulkan 卡 |
| `OLLAMA_VULKAN` | `1` | `0`（禁 Vulkan） | 禁用 Vulkan |

### 10.7 平台特定配置

**macOS**（LaunchAgent 模式）：
```bash
launchctl setenv OLLAMA_HOST "0.0.0.0:11434"
# 重启 Ollama app
```

**Linux**（systemd）：
```bash
sudo systemctl edit ollama.service
# 追加：
# [Service]
# Environment="OLLAMA_HOST=0.0.0.0:11434"
# Environment="OLLAMA_MAX_LOADED_MODELS=2"
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Windows**（环境变量）：
1. 退出 Ollama（托盘右键 Quit）
2. Win+R → `SystemPropertiesAdvanced` → Environment Variables
3. 添加用户变量 `OLLAMA_HOST=0.0.0.0:11434`
4. 启动 Ollama

---

## 11. 成本模型：本地、Cloud、Pro、Max

### 11.1 本地（Free / 自托管）

| 项目 | 成本 |
| --- | --- |
| 软件 | $0（MIT） |
| 硬件 | 一次性投入（看 9.1 表格；推荐 RTX 4090 / M4 Max） |
| 电费 | ~$0.05/kWh × GPU 功率 × 使用时长（4090 ~450W TDP） |
| 模型下载 | $0（开源权重） |
| 月度费用 | **$0 软件 + 折旧/电费** |

**典型场景**：
- 4090 24/7 满载：~10.8 kWh/天 × $0.05 = **$0.54/天 = $16/月**
- M4 Max 笔记本：~30W 平均 = $0.04/天 = **$1.20/月**
- 24GB M2 二手：硬件投入 ~$1500，分 3 年 = **$42/月 + 电费 $5**

### 11.2 Ollama Cloud（Pro / Max）

| 套餐 | 价格 | 关键能力 | 适合 |
| --- | --- | --- | --- |
| **Free** | $0 | 轻度云用量 | 试用 |
| **Pro** | $20/月 或 $200/年 | 50× 免费配额 + 3 并发云模型 + 私有模型 | 日常深度使用 |
| **Max** | $100/月 | 5× Pro 配额 + 10 并发 | 重度 agent |
| **Team**（Coming soon） | 联系销售 | SSO、MDM、共享配额 | 团队 |

**用量计算**：实际 GPU 时间 × 模型大小 + 缓存命中折扣。不是按 token 数硬封顶。

**示例**：
- 跑 `gpt-oss:20b`（Level 1，轻量）做 1 小时 coding 任务：可能 5-10% 配额
- 跑 `deepseek-v4-pro`（Level 4，超重）做 30 分钟 deep research：可能 80% 配额

### 11.3 与云 API 价格对比（按"每百万 token 折算"）

| 服务 | 模型 | 价格（input / output per 1M tokens） |
| --- | --- | --- |
| OpenAI | GPT-4o | $2.50 / $10.00 |
| OpenAI | GPT-4o-mini | $0.15 / $0.60 |
| Anthropic | Claude Sonnet 4.5 | $3.00 / $15.00 |
| Anthropic | Claude Haiku 4 | $0.80 / $4.00 |
| Google | Gemini 2.5 Pro | $1.25 / $10.00 |
| Ollama Cloud Pro | gpt-oss:20b | ~$0.10-0.50 / ~$0.30-1.50（推算，按 GPU 时长） |
| Ollama Cloud Pro | deepseek-v4-pro | ~$0.80-3.00 / ~$2.50-10.00（推算） |
| Ollama Local | 任意 GGUF | $0（边际电费） |

> Ollama Cloud 真实价格未公开按 token 折算（按"GPU 时间"计费），上表为基于"Pro $20/月含 50× 免费配额"的反推。

### 11.4 副业小B 成本估算

**场景 A**：一个 SaaS 副业，每月 1000 个用户，每用户 100 个 chat 请求，平均 500 input + 500 output tokens。

| 方案 | 月成本 |
| --- | --- |
| 全部用 OpenAI GPT-4o-mini | 1000 × 100 × 1k tok × $0.15/1M = **$15/月** |
| 全部用 Ollama Pro | 估计 $20-50/月（取决于模型） |
| 全部本地（4090 主机） | $0 软件 + 折旧（自购卡 $1500 摊 3 年 = $42/月） |
| 混合（轻量请求用本地，重请求用 Ollama Pro） | **$5-20/月** |

**场景 B**：数据敏感的医疗小B SaaS。

| 方案 | 月成本 | 合规 |
| --- | --- | --- |
| 全部用 OpenAI API | $X | ❌ 数据外流 |
| 全部用 Ollama 本地部署（4 卡 H100 整机） | $30,000 一次性 + $400 电费/月 | ✅ 数据本地 |
| 全部用 Ollama Pro | $20-100/月 | ⚠️ 数据出云（要审核） |

---

## 12. 生态与社区集成

> 来自 README 底部"Community Integrations"列表（GitHub `ollama/ollama`）

### 12.1 集成数量统计

| 类别 | 数量（README 列出的） |
| --- | --- |
| Chat Web UI | 15+ |
| Desktop 客户端 | 20+ |
| Mobile 客户端 | 8+ |
| 代码编辑器 / IDE | 15+ |
| Libraries & SDKs | 25+ |
| Frameworks & Agents | 8+ |
| RAG & Knowledge Bases | 8+ |
| Bots & Messaging | 4+ |
| Terminal & CLI | 10+ |
| Productivity & Apps | 12+ |
| Observability & Monitoring | 6+ |
| Database & Embeddings | 4+ |
| Infrastructure & Deployment（云厂商） | 4+ |
| **合计** | **140+（粗略数）** |

### 12.2 头部集成（精选）

**Chat Web UI**：
- Open WebUI（自托管、可扩展）
- Lobe Chat（中文友好、插件生态）
- NextChat（跨平台）
- Perplexica（开源 Perplexity 替代）
- big-AGI（专业级 AI Suite）

**Desktop 客户端**：
- AnythingLLM（全平台）
- Cherry Studio（中文）
- Chatbox
- Enchanted（macOS/iOS 原生）

**代码 IDE**：
- **Cline**（VS Code 多文件编码）
- **Continue**（开源 Copilot）
- **Void**（开源 Cursor 替代）
- **Roo Code**（JetBrains/VS Code）
- **Claude Code**（通过 `ollama launch claude` 一键切本地）
- **OpenAI Codex**（通过 `ollama launch codex`）
- **Copilot CLI**（通过 `ollama launch copilot-cli`）
- **Droid**（Factory 编码 agent）
- **OpenCode**（开源编码 agent）

**Libraries & SDKs**：
- Python：`ollama-python`（官方）、LiteLLM、LlamaIndex、LangChain
- JavaScript：`ollama-js`（官方）、Vercel AI SDK、LlamaIndexTS
- Java：Spring AI、Semantic Kernel、LangChain4j、Ollama4j
- Go：LangChainGo、ollama-rs
- C#/.NET：OllamaSharp、LlmTornado、Microsoft AI Toolkit
- Rust：ollama-rs、LangChainRust
- Ruby：ruby_llm
- Swift：ollama-swift
- C++：ollama-hpp
- Julia：PromptingTools.jl
- R：rollama

**Frameworks & Agents**：
- AutoGPT、crewAI、Strands Agents（AWS）、Cheshire Cat、any-agent（Mozilla）、Neuro SAN

**RAG & Knowledge Bases**：
- RAGFlow、MaxKB、Minima、Casibase、Archyve

**Observability**：
- Opik（Comet）、OpenLIT、Lunary、Langfuse、HoneyHive、MLflow Tracing

**Database & Embeddings**：
- pgai（Timescale）、MindsDB、chromem-go

**云部署**：
- Google Cloud Run（`cloud.google.com/run/docs/tutorials/gpu-gemma2-with-ollama`）
- Fly.io
- Koyeb
- Harbor（容器化 LLM 工具包）

### 12.3 `ollama launch` 集成体系

> 这是 Ollama 0.30+ 推出的"一键对接"能力。

```bash
# 启动交互式菜单
ollama launch

# 直接启动某个集成
ollama launch claude          # Claude Code
ollama launch codex           # OpenAI Codex CLI
ollama launch codex-app       # OpenAI Codex App
ollama launch copilot-cli     # GitHub Copilot CLI
ollama launch cline           # Cline CLI
ollama launch droid           # Factory Droid
ollama launch opencode        # OpenCode
ollama launch openclaw        # OpenClaw（个人 AI 助手）
ollama launch vscode          # VS Code
ollama launch pi              # Pi
ollama launch goose           # Goose
ollama launch pool            # Pool

# 指定模型启动
ollama launch claude --model qwen3.5

# 仅配置不启动
ollama launch droid --config

# 恢复被修改的配置
ollama launch codex-app --restore
ollama launch claude-desktop --restore
```

### 12.4 IDE / 编辑器插件

- VS Code：AI Toolkit（Microsoft）、Cline、Continue、Roo Code
- JetBrains：Roo Code、QodeAssist
- Xcode：原生集成
- Zed：原生集成
- Sublime Text 4：AI ST Completion
- Emacs：gptel、Ellama
- Obsidian：Copilot、Local GPT

---

## 13. 典型客户案例与生产案例

### 13.1 官方/博客案例

1. **Stanford Hazy Research "Minions"**（2025-02-25 博客）
   - 论文：本地小模型（Llama 3.2 跑在 Ollama）+ 云端大模型（GPT-4o）协同
   - 价值：把 80%+ LLM workload 转移到消费级设备
   - 研究者：Avanika Narayan, Dan Biderman, Sabri Eyuboglu（Christopher Ré 实验室）

2. **Google Firebase Genkit**（2024-05-20 博客，Google I/O 2024 宣布）
   - Ollama 集成到 Firebase Genkit（Google 开源 AI 框架）
   - 让 Genkit 用户可以一键切到本地 Ollama 后端

3. **OpenAI Codex with Ollama**（2026-01-15 博客）
   - 证明 Ollama 的 OpenAI 兼容层可以无缝替代 Codex 的云后端
   - 支持模型：gpt-oss:20b, gpt-oss:120b

4. **Codex App 本地化**（0.24 2026-05-14）
   - OpenAI 的 Codex App（桌面应用）通过 `ollama launch codex-app` 用本地模型
   - 推荐模型：kimi-k2.6（含 vision）、glm-5.1（编码最强）、nemotron-3-super、gemma4:31b、qwen3.6

### 13.2 社区公认生产案例

虽然 Ollama Inc. 没有公开客户案例数据库，但根据 GitHub 仓库和社区调研：

| 行业 | 典型用户 | 部署形态 | 价值 |
| --- | --- | --- | --- |
| **教育** | 大学实验室 | K8s 集群 + Ollama Helm | 师生用本地 LLM 做 NLP 教学，零数据外流 |
| **医疗** | 医院信息科 | 边缘盒子（Jetson Orin） | 离线病历摘要、HIPAA 合规 |
| **金融** | 小型券商 | 内网 Docker | 财报分析、研报摘要，监管留痕 |
| **法律** | 律所 | Mac Mini M4 集群 | 合同审查、客户邮件草稿 |
| **客服 SaaS** | 副业小B | 云（Ollama Pro）+ 兜底本地 | 节省 90% OpenAI 账单 |
| **游戏工作室** | Indie 团队 | 桌面 App | NPC 对话、剧情生成 |
| **硬件 OEM** | 联想/HP 笔记本 | 预装 Ollama | "AI 笔记本"差异化卖点 |
| **学术研究** | 高校 | DGX Spark / 多卡 H100 | 复现 SOTA 模型微调 |

---

## 14. 安全与隐私模型

### 14.1 本地模式（默认）

- **Ollama runs locally. We don't see your prompts or data when you run locally.**
- 模型权重、对话历史、模型缓存全部在本地
- 网络流量只有"拉模型"时去 ollama.com（**HTTPS** + content-addressed blob）

### 14.2 云模式（ollama.com）

- 处理 prompts/responses 但**不存储、不记录、不训练**
- 数据策略：no logging, no training, zero data retention
- 不卖用户数据；用户随时可删账号
- 基础账户信息 + 限定使用元数据被收集（用于计费）
- 云合作方：NVIDIA Cloud Providers（NCPs），签同等级协议

### 14.3 禁用云功能（纯本地）

```bash
# 方式 1：配置文件
cat > ~/.ollama/server.json <<EOF
{
  "disable_ollama_cloud": true
}
EOF

# 方式 2：环境变量
OLLAMA_NO_CLOUD=1 ollama serve

# 重启 Ollama 后日志会显示 "Ollama cloud disabled: true"
```

禁用后：失去云模型 + 失去 web search 功能。

### 14.4 传输安全

- **本地 HTTP**（明文）—— 设计上为本地开发
- **HTTPS_PROXY 支持**：拉模型走 HTTPS
- **生产部署**：必须套 Nginx/Caddy/Envoy + TLS + 鉴权
- **不推荐**直接把 11434 暴露公网（任何人都能拉你的模型 + 调你 GPU）

### 14.5 已知安全注意事项

1. **ggml-org/llama.cpp 漏洞**（2024-2025 多次 CVE）—— 关注 llama.cpp 安全公告
2. **模型权重投毒** —— `ollama pull` 不验证模型哈希以外的元数据；建议锁定 digest
3. **CORS 默认放开 0.0.0.0** —— 同源策略较弱，建议生产加 OLLAMA_ORIGINS 限制
4. **无内置 RBAC** —— 所有能访问端点的用户都享有同等权限
5. **日志可能含 prompt**（debug 模式）—— 生产不要开 OLLAMA_DEBUG=1
6. **临时文件** 在 `OLLAMA_TMPDIR`（默认系统 tmp）—— 含解压后的二进制；注意 noexec 文件系统

### 14.6 建议的"小B 副业"安全实践

```bash
# 1. 监听本地 + Nginx 反代
OLLAMA_HOST=127.0.0.1:11434 ollama serve

# 2. Nginx 配 TLS + Bearer Token
server {
    listen 443 ssl;
    ssl_certificate ...;
    ssl_certificate_key ...;
    location / {
        proxy_pass http://127.0.0.1:11434;
        proxy_set_header Authorization "Bearer YOUR_SECRET";
    }
}

# 3. 关闭云功能
OLLAMA_NO_CLOUD=1 ollama serve

# 4. 锁定模型 digest 而非 tag
ollama pull gemma3@sha256:a2af6cc3eb7fa8be8504abaf9b04e88f17a119ec3f04a3addf55f92841195f5a

# 5. 限制 CORS
OLLAMA_ORIGINS=https://yourdomain.com ollama serve
```

---

## 15. 优劣势分析

### 15.1 优势（Strengths）

| # | 优势 | 量化指标 |
| --- | --- | --- |
| 1 | **极简上手** | `curl -fsSL https://ollama.com/install.sh \| sh` 5 分钟跑起来 |
| 2 | **OpenAI 兼容** | 90% OpenAI 客户端代码可零修改切换 |
| 3 | **跨平台** | macOS / Win / Linux / Docker / ARM (Jetson) / WSL2 |
| 4 | **广硬件支持** | NVIDIA（CC 5.0+）、AMD ROCm v7、Apple Silicon、Intel/AMD Vulkan |
| 5 | **协议完整** | 4 端点（generate/chat/embed/create）+ 6 模型管理端点 + 4 OpenAI 兼容端点 |
| 6 | **生态丰富** | 140+ 社区集成，覆盖聊天/IDE/agents/RAG/observability |
| 7 | **Modelfile 声明式** | 5 行定义一个新模型，类似 Dockerfile |
| 8 | **本地 + 云无缝** | 同一 base_url 切换；同一 API 代码 |
| 9 | **强社区** | 173K stars、3,344 open issues（健康活跃度） |
| 10 | **开源 + 商业平衡** | MIT 协议 + Pro/Max 商业化（云端 + 高级模型） |
| 11 | **多模态** | vision (gemma3, qwen3-vl, llama3.2-vision) + embedding (nomic-embed, mxbai-embed-large) |
| 12 | **工具调用** | 全模型支持 tools 字段，OpenAI 协议对齐 |
| 13 | **思考模型支持** | 原生 `think: "low" \| "medium" \| "high"`，gpt-oss、deepseek-r1、qwen3-thinking 全部 work |
| 14 | **量化灵活** | q8_0 / q4_K_S / q4_K_M，Ollama 端 + llama.cpp 端可同时使用 |
| 15 | **可观测** | 每次响应带 5+ 纳秒精度性能指标 |
| 16 | **AI 生态巨头背书** | Google Firebase Genkit、OpenAI Codex、Microsoft AI Toolkit、AWS Strands Agents 全部原生支持 |

### 15.2 劣势（Weaknesses）

| # | 劣势 | 影响程度 |
| --- | --- | --- |
| 1 | **无内置多租户/计费** | 商业 SaaS 需要自己加 auth/billing 层 |
| 2 | **无内置缓存** | 不能跨实例共享 KV cache；跨用户需要外接 Redis |
| 3 | **无内置 rate limit** | 需要在反代层（Envoy/Nginx）实现 |
| 4 | **不支持 logprobs** | 某些科学实验需要，官方明确未支持 |
| 5 | **本地无 SLA** | 自托管没有 99.9% 保证；需要自建冗余 |
| 6 | **多 GPU 扩展性弱于 vLLM/SGLang** | 单机 8 卡还行，跨机需要 KServe/Triton 包装 |
| 7 | **生产级监控缺位** | 需要外接 Opik/Langfuse/OpenLIT |
| 8 | **不擅长高并发小请求** | 不像 BentoML/MLC-LLM 那样为超高 QPS 优化 |
| 9 | **不内置 A/B 测试、影子流量** | 需要自建 ML platform |
| 10 | **无内置模型路由/降级** | 单模型失败 = 整个请求失败（不像 Portkey 那样自动 fallback） |
| 11 | **CORS 默认松** | 0.0.0.0 跨域默认通过 → 生产必须显式收紧 |
| 12 | **WASM / 浏览器端模型无内置** | 与 WebLLM/tinygrad-llama 互不替代 |
| 13 | **企业 SSO/SAML 不内置** | Team 计划"coming soon" |
| 14 | **审计日志无** | 合规行业需要外接 OpenTelemetry 收集器 |
| 15 | **无内置 prompt 模板版本管理** | Modelfile 是文件级，没有 git-like 协作能力 |
| 16 | **官方未公开"统一性能 benchmark"** | 第三方测试时有矛盾数据 |

### 15.3 中性 / 取决于场景

- **Python SDK 风格**：与传统 `requests` 库差异大，新人需要 1 天适应
- **`/api/chat` 用 `application/x-ndjson`** 而非 OpenAI 的 `text/event-stream`——写底层代理时需注意
- **`tools` 字段是 OpenAI 风格**（不是 Anthropic 风格）——Anthropic 用户需要适配
- **模型格式以 GGUF 为主**——某些新发布的开源模型（如 0-day 发布）需要先等社区 GGUF 化

---

## 16. 与其他本地/边缘 LLM 运行时对比

> 选取 5 个最有竞争力的对比对象

### 16.1 对比矩阵

| 维度 | Ollama | llama.cpp | vLLM | LMDeploy | llamafile | LocalAI |
| --- | --- | --- | --- | --- | --- | --- |
| **定位** | 一体化框架 | 底层推理库 | 高吞吐服务 | 高吞吐服务 | 单文件可执行 | OpenAI 替代 |
| **主语言** | Go + C++ | C++ | Python + C++/CUDA | Python + C++/CUDA | C++ | Go + C++ |
| **API 协议** | OpenAI + Ollama 原生 | 无 HTTP（库） | OpenAI | OpenAI | OpenAI | OpenAI |
| **GUI** | ✅ 桌面 App + CLI | ❌ | ❌ | ❌ | ❌ | ❌ |
| **模型库** | 30+ 开源模型 | 用户自己找 GGUF | 用户自己找 | 用户自己找 | 用户自带 | 用户自带 |
| **Mac Apple Silicon** | ✅ MLX + Metal | ✅ Metal | ⚠️ 实验 | ⚠️ 实验 | ✅ | ✅ |
| **NVIDIA 加速** | ✅ CUDA v12/v13 | ✅ CUDA | ✅ CUDA + ROCm | ✅ CUDA + ROCm | ✅ CUDA | ✅ CUDA |
| **AMD GPU** | ✅ ROCm v7 | ✅ ROCm | ✅ ROCm | ⚠️ | ✅ Vulkan | ✅ ROCm |
| **多 GPU 扩展** | 节点内 OK | ⚠️ 手动 | ✅ tensor/pipeline parallel | ✅ tensor parallel | ❌ | ⚠️ |
| **量化** | q4_K_M/q8_0/... | 全套 GGML | AWQ/GPTQ/SqueezeLLM | AWQ/GPTQ | q4_0/q5_1/... | 多 |
| **KV cache 量化** | ✅ | ✅ | ✅ | ✅ | ⚠️ | ✅ |
| **投机解码** | ✅ draft model | ✅ | ✅ | ✅ | ❌ | ❌ |
| **吞吐优化** | 中 | 中 | **极高**（PagedAttention） | 极高 | 低 | 中 |
| **TTFT** | 中 | n/a | 低 | 低 | 高 | 中 |
| **冷启动** | 慢（首次加载） | 慢 | 慢 | 慢 | 慢 | 慢 |
| **学习曲线** | **极低** | 高 | 中 | 中 | 低 | 低 |
| **生产级特性** | 中（无 RBAC） | 无 | 中 | 中 | 无 | 中 |
| **社区规模** | 173K stars | 80K+ stars | 35K+ stars | 5K+ stars | 20K+ stars | 30K+ stars |
| **License** | MIT | MIT | Apache 2.0 | Apache 2.0 | MIT | MIT |
| **典型用户** | 个人/小B/副业 | 极客/研究者 | 大厂/高并发服务 | 大厂/高并发服务 | 极客/教育 | 中小团队 |

### 16.2 选型决策树

```
你需要的核心是？
│
├── "开箱即用跑对话" → Ollama ✅
├── "单机高 QPS 推理服务" → vLLM 或 LMDeploy
├── "极简单文件分发" → llamafile
├── "完全控制底层" → llama.cpp
├── "OpenAI 替代 + 多 backend" → LocalAI
└── "VS Code/Claude Code 等 IDE 集成" → Ollama（绝大多数 IDE 优先支持）
```

### 16.3 与 AI Gateway 类产品对比

> "AI Gateway" 通常指 LiteLLM、Portkey、Unify、Not Diamond 这类"在多个 LLM provider 之间路由"的网关。

| 维度 | Ollama | LiteLLM | Portkey | Unify |
| --- | --- | --- | --- | --- |
| **核心定位** | 本地模型运行 | LLM 网关（100+ provider） | LLM 网关（含可观测性） | LLM 网关（性能+成本） |
| **本地推理** | ✅ 一等公民 | ❌ 需外接 | ❌ 需外接 | ❌ 需外接 |
| **路由云 API** | ⚠️ 仅 Ollama Cloud | ✅ 100+ | ✅ 主流 | ✅ 主流 |
| **OpenAI 兼容出** | ✅ | ✅ | ✅ | ✅ |
| **OpenAI 兼容入** | ❌（自身是 server） | ✅ | ✅ | ✅ |
| **Fallback / 重试** | ❌ | ✅ | ✅ | ✅ |
| **A/B 测试** | ❌ | ✅ | ✅ | ✅ |
| **缓存** | ❌ | ✅ | ✅ | ✅ |
| **计费/限额** | ❌ | ✅ | ✅ | ✅ |
| **副业小B 友好度** | **高** | 中 | 中 | 中 |

**关键洞察**：Ollama 不是 AI Gateway，**它是要被 AI Gateway 包起来的本地后端**。典型架构：

```
客户端
  ↓ OpenAI 兼容 API
LiteLLM / Portkey（路由 + 缓存 + fallback）
  ↓
├── Ollama（本地，私有数据）
├── OpenAI API（云端，gpt-4o）
├── Anthropic API（云端，claude）
└── Ollama Cloud（云端，gpt-oss:120b）
```

---

## 17. 与小B行业软件副业的契合度分析

> 副业目标：小B 商户数字化转型，5-15 万/年订阅

### 17.1 Ollama 在副业场景的五大价值

1. **降低 API 成本 80-99%**
   - 场景：餐饮 SaaS 自动回复会员咨询
   - 原来：每月 5000 会员 × 10 轮对话 × OpenAI = ~$100/月
   - 改用：本地 Ollama + Qwen3:8B = $0 软件成本 + $5 电费

2. **数据合规卖点**
   - 场景：医疗 SaaS 病历摘要
   - 卖点："您的病历数据 100% 留在您医院的服务器，不上云"
   - 对应客户：院长/CIO 决策者

3. **边缘盒子交付**
   - 场景：物业 SaaS 巡检工单
   - 形态：Jetson Orin 盒子 + 预装 Ollama + Qwen3-VL
   - 模式：硬件 + 软件打包卖

4. **"按 token 计费"商业模型**
   - 场景：法律 SaaS 合同审查
   - 用 Ollama Pro 做"基础设施"层，对外按"份/每份"收费
   - 毛利 = 客户付费 - Ollama Pro 成本

5. **快速验证 MVP**
   - 场景：所有副业的"0→1 阶段"
   - 用 Ollama 跑开源模型（gpt-oss、Qwen3、DeepSeek）做 PoC
   - 不花 OpenAI API 钱也能 demo

### 17.2 副业场景推荐组合

| 副业场景 | 推荐 Ollama 配置 | 月度软件成本 |
| --- | --- | --- |
| 餐饮 SaaS 智能客服 | Ollama Free + Qwen3:8B 本地 | $0 |
| 美业 SaaS 客户画像 | Ollama Pro + DeepSeek-V3.2 | $20 |
| 物业巡检 AI | Jetson + Ollama + Qwen3-VL（视觉版） | 一次性硬件 |
| 医疗病历摘要 | 本地 4×A100 + Ollama | $0 软件 + 折旧 |
| 法律合同审查 | Ollama Max + GLM-5.1 | $100 |
| 教培 AI 助教 | Ollama Free + Gemma3:27B | $0 |
| 财税智能问答 | Ollama Pro + Qwen3-Coder:30B | $20 |
| 通用 CRM 助手 | Ollama Free + Phi-4 | $0 |

### 17.3 副业使用 Ollama 的"最佳实践清单"

1. **永远用 OpenAI 兼容层**（`/v1/chat/completions`）写代码，方便切换云端
2. **模型版本锁定 digest**（`gpt-oss:20b@sha256:...`），避免上游 tag 改影响生产
3. **设置 `keep_alive=-1`** 给核心模型（避免冷启动 100ms 延迟）
4. **Modelfile 注入 system prompt**，比 runtime 传更安全
5. **生产必须 `OLLAMA_HOST=127.0.0.1` + Nginx 反代 + Bearer Token**
6. **关闭云 `OLLAMA_NO_CLOUD=1`**（数据敏感行业）
7. **监控 `ollama ps` + `/api/ps`**，模型驻留状态可视化
8. **小流量用本地 + 大流量用 Ollama Pro** 做兜底（防 GPU 烧穿）
9. **不要直接暴露 11434 到公网**
10. **Modelfile / 配置走 Git 版本管理**，团队协作不出错

---

## 18. 风险与未来展望

### 18.1 风险

| 风险 | 等级 | 缓解 |
| --- | --- | --- |
| llama.cpp 出现严重 CVE | 中 | 关注 GitHub advisories；及时升 Ollama |
| Ollama 商业模式未稳 | 中 | 准备 2-3 个备选 backend（vLLM、LMDeploy） |
| 云端 NCP 协议变动 | 低 | 合同条款锁定；本地可独立运行 |
| Apple Silicon MLX 与 llama.cpp 分裂 | 低 | 团队已并列支持；选一即可 |
| 闭源大模型对 Ollama 兼容层不友好 | 低 | 多数模型走 GGUF 社区化 |
| 监管要求"AI 内容审计" | 中 | 需外接审计日志系统（OpenLIT 等） |
| 大量小模型同质化 | 中 | 关注模型 SOTA（MMLU、MT-Bench、LiveBench） |
| 极端场景下 OOM | 低 | 调小 num_ctx；切更小模型 |

### 18.2 未来趋势（基于 0.30+ 节奏推断）

1. **MLX 引擎成熟**：从"Apple Silicon 预览"到"NVIDIA 上跑 MLX 模型"（CUDA 13+ MLX）
2. **多模态扩展**：vision 模型成主流；audio 模型（gemma4 已带 audio）开始支持
3. **Codex App / Claude Code 等深度 IDE 集成**：Ollama 不再做"通用 LLM 工具"，而是"本地 LLM 路由器"
4. **云端 Pro 引入企业特性**：SSO、审计、RBAC 都在 Team 计划路线上
5. **多模态统一协议**：OpenAI 的 `responses` API 会被更多网关采纳
6. **MLX 跨平台**：从 Apple → NVIDIA/AMD 扩展，GGUF 与 Safetensor 双格式并行
7. **投机解码成标配**：从 0.30 起默认 4 token，模型内置 draft tensors（MTP）
8. **agent 工具链原生**：tool calls + thinking + JSON mode 已成为基线

### 18.3 Ollama 在"AI 边缘计算"赛道的定位

未来 3 年的"边缘盒子 / AI PC / AI Phone"赛道里，Ollama 是事实标准候选：
- **已预装的厂商**：联想、HP、Dell、Framework（待确认）
- **类似竞争者**：
  - **llamafile**（单文件可执行）—— 更适合 USB 启动
  - **Lemonade SDK**（AMD 推广）—— AMD 硬件专享
  - **Apple Foundation Models**（iOS 26+）—— Apple 设备专享
  - **MediaPipe LLM**（Google）—— Android 设备专享
- **Ollama 的位置**：通用平台型玩家，跨 OS × 跨硬件 × 跨云

---

## 19. 关键代码示例合集

### 19.1 最简单的"5 行聊天"

```bash
# 安装 → 拉模型 → 聊天
curl -fsSL https://ollama.com/install.sh | sh
ollama pull gemma3
ollama run gemma3 "你好"
```

### 19.2 Python OpenAI 兼容客户端

```python
from openai import OpenAI
import os

client = OpenAI(
    base_url=os.environ.get("OLLAMA_BASE_URL", "http://localhost:11434/v1/"),
    api_key=os.environ.get("OLLAMA_API_KEY", "ollama"),  # 本地忽略
)

resp = client.chat.completions.create(
    model="qwen3:8b",
    messages=[
        {"role": "system", "content": "你是一个严谨的助手。"},
        {"role": "user", "content": "解释 transformers 架构。"},
    ],
    temperature=0.7,
    max_tokens=500,
)
print(resp.choices[0].message.content)
```

### 19.3 JavaScript 流式

```javascript
import OpenAI from "openai";

const openai = new OpenAI({
  baseURL: "http://localhost:11434/v1/",
  apiKey: "ollama",
});

const stream = await openai.chat.completions.create({
  model: "gemma3",
  messages: [{ role: "user", content: "写一首关于秋天的诗。" }],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content || "");
}
```

### 19.4 工具调用完整示例

```python
import json
from openai import OpenAI

client = OpenAI(base_url="http://localhost:11434/v1/", api_key="ollama")

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {"type": "string"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
                },
                "required": ["location"]
            }
        }
    }
]

# 1. 第一次调用：模型决定要不要调工具
resp = client.chat.completions.create(
    model="qwen3:8b",
    messages=[{"role": "user", "content": "北京今天几度？"}],
    tools=tools,
    tool_choice="auto",
)

# 2. 如果模型调了工具
if resp.choices[0].message.tool_calls:
    tool_call = resp.choices[0].message.tool_calls[0]
    args = json.loads(tool_call.function.arguments)
    print(f"调用工具：{tool_call.function.name}, 参数：{args}")

    # 3. 实际执行工具（这里 mock）
    weather_data = {"beijing": 23, "unit": "celsius", "condition": "sunny"}

    # 4. 第二次调用：把工具结果回传给模型
    final = client.chat.completions.create(
        model="qwen3:8b",
        messages=[
            {"role": "user", "content": "北京今天几度？"},
            resp.choices[0].message,
            {
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(weather_data),
            },
        ],
        tools=tools,
    )
    print(final.choices[0].message.content)
```

### 19.5 嵌入（Embeddings）

```python
from openai import OpenAI
import numpy as np

client = OpenAI(base_url="http://localhost:11434/v1/", api_key="ollama")

resp = client.embeddings.create(
    model="nomic-embed-text",
    input=["客户咨询：我的订单还没到",
            "客服回复：您的订单正在派送中"],
    encoding_format="float",
)

# 768 维向量（nomic-embed-text）
v1 = np.array(resp.data[0].embedding)
v2 = np.array(resp.data[1].embedding)

# 余弦相似度
cos_sim = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
print(f"语义相似度：{cos_sim:.4f}")
```

### 19.6 视觉理解

```python
from openai import OpenAI
import base64

client = OpenAI(base_url="http://localhost:11434/v1/", api_key="ollama")

# 编码本地图片
with open("product_photo.jpg", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

resp = client.chat.completions.create(
    model="qwen3-vl:8b",  # 或 gemma3:27b (vision 版)
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "这张图里的产品有什么问题？"},
                {"type": "image_url",
                 "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"}}
            ]
        }
    ],
    max_tokens=500,
)
print(resp.choices[0].message.content)
```

### 19.7 Ollama Python SDK（原生）

```python
from ollama import chat, generate, embed

# 1. chat
response = chat(
    model='gemma3',
    messages=[{'role': 'user', 'content': 'Why is the sky blue?'}],
)
print(response.message.content)

# 2. 流式
stream = chat(
    model='gemma3',
    messages=[{'role': 'user', 'content': 'Tell me a story.'}],
    stream=True,
)
for chunk in stream:
    print(chunk['message']['content'], end='', flush=True)

# 3. generate（单轮）
response = generate(model='gemma3', prompt='What is Python?')
print(response.response)

# 4. embed
result = embed(model='nomic-embed-text', input='Hello world')
print(len(result.embeddings[0]))
```

### 19.8 Modelfile + 构建自定义模型

```bash
# /tmp/coder.Modelfile
cat > /tmp/coder.Modelfile <<'EOF'
FROM qwen3-coder:30b

# 编码优化：低温度 + 高 top_p
PARAMETER temperature 0.2
PARAMETER top_p 0.95
PARAMETER num_ctx 32000

# 强制 4 字符缩进
PARAMETER stop "```"
PARAMETER stop "<|im_end|>"

# 角色
SYSTEM You are an expert software engineer. Always explain your reasoning.

# 注入 few-shot
MESSAGE user Refactor this Python function for readability.
MESSAGE assistant Please share the function.
MESSAGE user def add(a,b):return a+b
MESSAGE assistant Sure! Here's a cleaner version:
MESSAGE assistant ```python
MESSAGE assistant def add(a: float, b: float) -> float:
MESSAGE assistant     """Return the sum of two numbers."""
MESSAGE assistant     return a + b
MESSAGE assistant ```
EOF

# 构建
ollama create my-coder -f /tmp/coder.Modelfile

# 运行
ollama run my-coder "Refactor this JavaScript: function add(a,b){return a+b}"

# 推送到 ollama.com 共享
ollama cp my-coder myuser/my-coder
ollama push myuser/my-coder
```

### 19.9 Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    runtime: nvidia  # 启用 NVIDIA GPU
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    environment:
      - OLLAMA_KEEP_ALIVE=30m
      - OLLAMA_MAX_LOADED_MODELS=2
      - OLLAMA_HOST=0.0.0.0:11434
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped

volumes:
  ollama-data:
```

```bash
docker compose up -d
docker exec -it ollama ollama pull gemma3
```

### 19.10 性能基准测试脚本

```python
"""
benchmark.py - 简单 Ollama 性能基准
"""
import time
from openai import OpenAI

client = OpenAI(base_url="http://localhost:11434/v1/", api_key="ollama")
MODEL = "qwen3:8b"
PROMPT = "用 200 字写一段关于'AI 与小B 数字化转型'的论述。" * 5
RUNS = 10

times = []
for i in range(RUNS):
    t0 = time.perf_counter()
    resp = client.chat.completions.create(
        model=MODEL,
        messages=[{"role": "user", "content": PROMPT}],
    )
    t1 = time.perf_counter()
    elapsed = t1 - t0
    tokens = resp.usage.completion_tokens
    times.append((elapsed, tokens))
    print(f"Run {i+1}: {elapsed:.2f}s, {tokens} tokens, {tokens/elapsed:.1f} tok/s")

print()
print(f"平均延迟: {sum(t for t,_ in times)/RUNS:.2f}s")
print(f"平均速度: {sum(t/l for t,l in times)/RUNS:.1f} tok/s")
```

---

## 20. 结论与建议

### 20.1 一句话总结

> **Ollama = 本地 LLM 推理的事实标准 + OpenAI 兼容网关 + 跨平台生态。对于副业小B，它既是"省 API 钱"的工具，也是"数据合规"的卖点；对于企业，它是"AI 边缘盒子"的基础设施。**

### 20.2 关键数据快照（2026-06-06）

| 维度 | 数值 |
| --- | --- |
| GitHub Stars | 173,349 |
| License | MIT |
| 主语言 | Go + C++/CUDA/ROCm/Metal |
| 最新版本 | v0.30.6（2026-06-05） |
| 社区集成 | 140+ |
| 协议 | OpenAI 兼容 + Ollama 原生（generate/chat/embed/create） |
| 量化 | q4_K_M, q8_0, q4_K_S, FP16 → Q4 |
| 默认上下文 | 4K (24G VRAM) / 32K (24-48G) / 256K (≥48G) |
| 默认 keep_alive | 5 分钟 |
| 默认 OLLAMA_MAX_LOADED_MODELS | 3 |
| 默认 OLLAMA_NUM_PARALLEL | auto |
| 模型库（library） | 30+ 模型家族，200+ tag |
| 云服务 | Free / Pro $20/月 / Max $100/月 / Team（Coming soon） |
| 平台支持 | macOS / Win / Linux / Docker / Jetson / WSL2 / K8s |
| GPU 支持 | NVIDIA (CC 5.0+), AMD ROCm v7, Apple Metal/MLX, Vulkan |

### 20.3 选型建议

**选 Ollama 当**：
- ✅ 你是个人/小团队，需要本地跑 LLM
- ✅ 你想要 OpenAI 兼容 API，但不想被 OpenAI 锁定
- ✅ 你有 Mac（M1+）/ 4090+ / M4 Max / 多卡 H100 设备
- ✅ 你的客户对"数据不出本地"有要求
- ✅ 你想用 IDE 工具（Claude Code/Codex/Cline/Continue）接本地 LLM
- ✅ 你做副业 SaaS 想砍 API 成本

**不选 Ollama 当**：
- ❌ 你需要超大规模分布式推理（跨机 100+ 卡）→ vLLM / LMDeploy
- ❌ 你需要 A/B 测试 + fallback + 复杂计费 → LiteLLM / Portkey
- ❌ 你只跑 SaaS，零本地部署意愿 → 直接 OpenAI / Anthropic API
- ❌ 你需要 Apple Foundation Models 级 on-device 体验（iOS App 内）→ Core ML / llama.cpp 直调

### 20.4 副业小B 的 3 步上手路径

1. **Day 1**：在自己 Mac/PC 装 Ollama → 拉 `gemma3:4b` → 跑通 OpenAI 兼容 API → 写 100 行 Python 集成
2. **Week 1**：选副业场景 → 拉生产级模型（Qwen3:8B / DeepSeek-R1-Distill:8B）→ 压测并发 → 写 Modelfile 优化 system prompt
3. **Month 1**：生产部署（Docker + Nginx 反代 + 监控）→ 加 Ollama Pro 兜底 → 开始接付费客户

### 20.5 一句话给"小F"

> **Ollama 是你做副业 SaaS 的"零边际成本 LLM 后端"——本地用 0 元跑开源模型，云端用 $20/月 兜底，再用 OpenAI 兼容 API 一行不改接你的代码。先用 Ollama 把 MVP 跑起来，等客户量起来再评估 LiteLLM 做多 provider 路由。**

---

**报告完。** 文件路径：`/root/.openclaw/workspace/aigw/openclaw/product-ollama-20260606.md`。
