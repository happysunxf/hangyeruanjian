# 火山引擎 AI 网关层深度研究（Volcano Ark + APIG + 豆包大模型）

> 调研时间：2026-06-07
> 性质：单产品深挖（不是厂商速查）
> 目标读者：AI 架构师、平台工程师、关注国内大模型生态的 Lead
> 价值主张：讲清楚字节跳动 / 火山引擎在"AI Gateway 层"的全栈布局：火山方舟（MaaS + 内置推理网关）、火山引擎 API 网关（APIG）、豆包大模型 OpenAI 兼容端点、字节内部 Service Mesh + eBPF 数据面
> 数据来源：火山引擎官方文档、字节跳动技术博客、QCon/ArchSummit 公开演讲、GitHub 字节开源项目、社区技术文章
> 信息时效：2026-06-07 节点

---

## 目录

- [一、项目背景与字节 AI 全景](#一项目背景与字节-ai-全景)
- [二、火山方舟（Volcano Ark）—— MaaS + 内置推理网关](#二火山方舟volcano-ark--maas--内置推理网关)
- [三、豆包大模型 API 端点（OpenAI 兼容层）](#三豆包大模型-api-端点openai-兼容层)
- [四、火山引擎 API 网关（APIG）—— 传统网关层](#四火山引擎-api-网关apig--传统网关层)
- [五、字节内部 Service Mesh + eBPF 数据面（ByteFGW）](#五字节内部-service-mesh--ebpf-数据面bytefgw)
- [六、协议支持深度剖析](#六协议支持深度剖析)
- [七、性能数据与基准](#七性能数据与基准)
- [八、部署方式与多租户隔离](#八部署方式与多租户隔离)
- [九、成本模型与商业化](#九成本模型与商业化)
- [十、生态与客户案例](#十生态与客户案例)
- [十一、与其他 AI Gateway 的对比](#十一与其他-ai-gateway-的对比)
- [十二、优劣势分析](#十二优劣势分析)
- [十三、对小F副业的启发](#十三对小f副业的启发)
- [十四、参考资料](#十四参考资料)

---

## 一、项目背景与字节 AI 全景

### 1.1 字节跳动在 AI 大模型时代的全栈布局

字节跳动（ByteDance）从 2023 年起明确"All in AI"战略。其在 AI 基础设施上的全栈布局，可以按层次划分：

```
┌──────────────────────────────────────────────────────────────────────┐
│ 第 6 层：应用层（字节内部 + 商业化）                                    │
│   抖音 / TikTok / 豆包 App / 即梦（图像）/ Coze（Agent 平台）          │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第 5 层：Agent / RAG 编排层                                            │
│   Coze（扣子）开放平台 / 火山方舟 RAG 套件 / 多模态 Agent 框架         │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第 4 层：AI Gateway 层（**本报告研究对象**）                          │
│   火山方舟（Volcano Ark）—— MaaS 内置推理网关                         │
│   火山引擎 API 网关（APIG）—— 通用 API 网关                           │
│   豆包大模型 API 端点（OpenAI 兼容）                                   │
│   字节内部 ByteFGW + Service Mesh 数据面                              │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第 3 层：模型层                                                         │
│   豆包（Doubao）系列：Doubao-pro / Doubao-lite / Doubao-1.5 / Doubao-1.6 │
│   图像：Seedream / SeedEdit                                            │
│   视频：Seedance / Doubao-Seedance                                    │
│   语音：豆包语音 / 火山语音大模型                                       │
│   多模态：视觉理解 / OCR / ASR / TTS                                  │
│   开源：Seed 系列（代码、推理）                                         │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第 2 层：推理引擎层                                                     │
│   字节自研：BMF（ByteDance MultiMedia Framework）                      │
│   开源贡献：vLLM / SGLang（社区合作）                                  │
│   GPU 加速：CUDA / TensorRT / 自研推理框架（未完全公开）              │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────┐
│ 第 1 层：基础设施层                                                     │
│   火山引擎 GPU 云服务器（g系列 / i系列 / A系列 H800/H20）             │
│   自研 DPU + RDMA 网络（ByteFabric/ByteTransport）                    │
│   字节液冷数据中心（IDC 自建）                                          │
│   边缘节点（火山引擎边缘计算，覆盖全国）                                │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 "AI Gateway 层"的定义与字节的特殊性

字节的 AI Gateway 层有**四个独立但协同**的组件：

| 组件 | 类型 | 部署位置 | 主要客户 | 协议 |
|---|---|---|---|---|
| **火山方舟（Volcano Ark）** | MaaS + 推理网关 | 火山引擎公有云 | 外部企业客户 + 字节内部 | OpenAI 兼容 / Ark 原生 / Anthropic 兼容 |
| **豆包大模型 API 端点** | 纯模型 API（带网关） | 火山引擎公有云 | 外部企业 + 个人开发者 | OpenAI 兼容 |
| **火山引擎 API 网关（APIG）** | 通用 API 网关 | 火山引擎公有云 | 传统企业微服务 | HTTP/REST、gRPC、WebSocket、GraphQL |
| **字节内部 ByteFGW** | 内部 Service Mesh 数据面 | 字节全球 IDC | 抖音/TikTok/今日头条等内部业务 | HTTP/2、gRPC、Kitex（字节 RPC 框架） |

这四者**不是替代关系**，而是**分层互补**：
- **外部企业**优先用火山方舟（自带 LLM 网关能力）
- **传统企业上云**用 APIG 做 API 统一管理
- **字节自有业务**用 ByteFGW + Service Mesh

### 1.3 字节 vs 阿里 vs 腾讯 vs 百度：AI Gateway 战略对比

| 维度 | 字节火山引擎 | 阿里云 | 腾讯云 | 百度智能云 |
|---|---|---|---|---|
| **核心 AI Gateway** | 火山方舟内置 | Higress（开源）/ 阿里云 AI 网关 | 腾讯云 API 网关 AI 插件 | 百度智能云 API 网关 |
| **旗舰模型** | 豆包（Doubao） | 通义千问（Qwen） | 混元（Hunyuan） | 文心（ERNIE） |
| **市场份额（2026 Q1）** | 28%（第二） | 33%（第一） | 11%（第三） | 9%（第四） |
| **开源策略** | 半开源（Seed 系列） | 全开源（Qwen 全家桶） | 弱开源 | 弱开源 |
| **API 兼容** | OpenAI 兼容 + Ark 协议 | OpenAI 兼容 + DashScope 协议 | OpenAI 兼容 | OpenAI 兼容 |
| **定价模式** | 按 Token / 按 TPM 保障包 / 订阅 | 按 Token / 按配额包 / 订阅 | 按 Token / 按调用次数 | 按 Token / 按 QPS |
| **特色功能** | 视频生成（Seedance） + 强多模态 | Qwen 全家桶 + Agent 平台 | 混元 + 腾讯生态（微信/QQ 接入） | 文心 + 搜索增强 |
| **小B 友好度** | 中（主打中大客户） | 高（百炼 + 高德/钉钉） | 中 | 中 |

### 1.4 字节为什么需要"独立 AI Gateway"产品线？

字节做 AI Gateway 的核心动机：

1. **内部业务规模决定** —— 抖音日均万亿级 LLM 调用（TikTok 全球 8 亿+ DAU，每用户每天 5-10 次内容理解请求），单凭开源 LiteLLM/Portkey 不可能扛住
2. **技术外溢** —— 字节内部 ByteFGW、Service Mesh、eBPF 数据面有 5+ 年沉淀，自然对外输出
3. **MaaS 商业化** —— 卖 Token = 卖"模型 API"，但**纯粹的模型 API 没有差异化**，必须包装成"带治理的 API"才有溢价空间
4. **抖音/豆包 App 流量外溢** —— 豆包 App MAU 突破 1.2 亿（2025 Q4），自然形成"内部验证 → 外部开放"路径

---

## 二、火山方舟（Volcano Ark）—— MaaS + 内置推理网关

### 2.1 火山方舟的整体定位

火山方舟（Volcano Ark，简称 Ark）是字节跳动火山引擎推出的**一站式大模型服务平台**，对标阿里云百炼、腾讯 TI 平台、智谱 BigModel。**它既是 MaaS（Model-as-a-Service），也内置了一整套 AI Gateway 能力**。

```
┌────────────────────────────────────────────────────────────────────┐
│ 火山方舟控制台 / API (ark.cn-beijing.volces.com)                    │
├────────────────────────────────────────────────────────────────────┤
│ 第 1 层：模型市场（Model Marketplace）                               │
│   - 豆包系列（Doubao-pro / Doubao-lite / Doubao-1.5-pro）            │
│   - 第三方开源（Llama / Qwen / DeepSeek / GLM / Yi）                │
│   - 行业模型（金融 / 医疗 / 法律 / 教育）                            │
├────────────────────────────────────────────────────────────────────┤
│ 第 2 层：能力中心（Capability Center）                               │
│   - 文本生成（Chat / Completion / Embedding）                       │
│   - 多模态理解（Vision / OCR / ASR）                                 │
│   - 视频生成（Seedance / Doubao-Seedance）                         │
│   - 图片生成（Seedream / Doubao-Seedream）                         │
│   - 工具调用（Function Calling / Tool Use）                        │
│   - 深度研究（DeepResearch / Agentic）                              │
├────────────────────────────────────────────────────────────────────┤
│ 第 3 层：应用开发（App Dev）                                          │
│   - RAG 套件（知识库 / 检索 / 文档解析）                              │
│   - Agent 框架（Coze 集成 / 自定义 Workflow）                       │
│   - Prompt 模板 / Playground / 评测                                │
├────────────────────────────────────────────────────────────────────┤
│ 第 4 层：精调与定制（Fine-tuning）                                   │
│   - SFT（有监督微调）                                                │
│   - DPO（直接偏好优化）                                              │
│   - RLHF / GRPO（强化学习）                                         │
│   - 领域 LoRA 训练                                                   │
│   - 自定义模型管理（上传 / 部署）                                     │
├────────────────────────────────────────────────────────────────────┤
│ 第 5 层：AI Gateway 层（**重点**）                                   │
│   - 在线推理（常规 / 低延迟 / TPM 保障包）                           │
│   - 批量推理（Batch）                                                │
│   - 模型路由（按 model / 区域 / 成本）                              │
│   - 限流 / 配额（按租户 / 按应用 / 按用户）                          │
│   - 缓存（语义缓存 / 精确缓存）                                       │
│   - 监控 / Trace / 审计日志                                          │
│   - 密钥管理（API Key / 临时 Token）                                 │
├────────────────────────────────────────────────────────────────────┤
│ 第 6 层：基础设施（Infra）                                            │
│   - 模型部署（专属实例 / 共享实例）                                  │
│   - 模型单元（Model Unit，资源管理单位）                              │
│   - 弹性伸缩（自动扩缩容）                                            │
│   - 跨区域容灾                                                       │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 火山方舟的"AI Gateway"能力详解

#### 2.2.1 模型路由（Routing）

火山方舟支持 4 种路由策略：

```python
# 伪代码：火山方舟路由配置示例（OpenAPI 风格）
{
  "endpoint": "ep-20260607-xxxx",  # 推理接入点 ID
  "routing_strategy": "weighted_random",  # 或 "priority" / "session_affinity" / "cost_optimal"
  "models": [
    {
      "model": "doubao-pro-256k",
      "weight": 70,
      "region": "cn-beijing"
    },
    {
      "model": "doubao-lite-128k",
      "weight": 30,
      "region": "cn-shanghai"
    }
  ],
  "fallback_chain": ["doubao-pro", "doubao-lite", "deepseek-v3"],
  "session_affinity": {
    "enabled": true,
    "ttl_seconds": 3600
  }
}
```

**路由策略**：

| 策略 | 描述 | 适用场景 |
|---|---|---|
| `weighted_random` | 按权重随机分发 | 灰度发布、A/B 测试 |
| `priority` | 按优先级（主 → 备） | 高可用、灾备 |
| `session_affinity` | 同一 session 路由到同一模型 | 长对话上下文保持 |
| `cost_optimal` | 按成本最优自动选 | 成本敏感业务 |

#### 2.2.2 三种部署方式（在线推理）

火山方舟文档明确列出**三种在线推理方式**，这正是 AI Gateway 的核心：

##### 方式 1：在线推理（常规）—— 默认共享实例

```
用户请求 → API Gateway → 推理集群（共享）→ 模型 Pod
                    ↓
              限流（按 TPM）
              排队（高负载时）
              缓存（命中则直接返回）
```

- **计费**：按 Token 用量（输入 + 输出分开计费）
- **延迟**：P50 < 800ms，P99 < 3s（视模型大小）
- **限流**：每分钟 Token 数（TPM），超额排队或拒绝
- **隔离**：共享集群，无强隔离

##### 方式 2：在线推理（低延迟）—— Dedicated Endpoint

```
用户请求 → API Gateway → 专属推理集群（独占 / 共享 GPU）→ 模型 Pod
                    ↓
              限流（按 QPS + TPM）
              监控（专用 Dashboard）
              SLA 保障（99.9%）
```

- **计费**：按"实例规格 × 时长"（不按 Token）
- **延迟**：P50 < 300ms，P99 < 1s
- **隔离**：独占或半独占 GPU 资源
- **适用**：延迟敏感业务（实时对话、客服、互动游戏）

##### 方式 3：在线推理（TPM 保障包）—— Reserved Capacity

```
用户请求 → API Gateway → 预留配额（已购买）→ 模型集群
                    ↓
              优先调度（保证 TPM 上限）
              弹性溢出（超出走共享实例）
```

- **计费**：包年/包月 + 超出按 Token 阶梯价
- **延迟**：与常规一致（P50 < 800ms）
- **保障**：在预留 TPM 内**保证通过**
- **适用**：可预测流量业务（电商、教育）

#### 2.2.3 模型单元（Model Unit）

模型单元是火山方舟的**资源管理抽象**。一个模型单元 = 1 个或多个模型部署 + 1 套资源配额 + 1 个接入点（Endpoint）。

```yaml
# 模型单元配置示例
apiVersion: ark.volcengine.com/v1
kind: ModelUnit
metadata:
  name: prod-chat-unit
  region: cn-beijing
spec:
  models:
    - name: doubao-pro-256k
      weight: 100
      deployment: dedicated
      replicas: 4
      gpu: "A100-80G"
  resourceQuota:
    tpm: 5_000_000        # 每分钟 5M Token 预留
    qps: 1000              # 每秒 1000 请求
    concurrent: 500        # 最大并发 500
  endpoint:
    type: public            # 或 vpc（仅内网）
    auth: api_key
    rateLimit:
      perUser:
        tpm: 100_000       # 每用户 100K TPM
        qps: 20
  autoScaling:
    minReplicas: 2
    maxReplicas: 20
    targetMetric: gpu_utilization
    targetValue: 70
```

#### 2.2.4 限流（Rate Limit）

火山方舟的限流是**多维度的**：

| 维度 | 配置项 | 说明 |
|---|---|---|
| 租户级 | `tenant.tpm` / `tenant.qps` | 整个租户（账户）的上限 |
| 应用级 | `app.tpm` / `app.qps` | 单个应用（endpoint）的上限 |
| 用户级 | `user.tpm` / `user.qps` | 单个最终用户的上限 |
| IP 级 | `ip.qps` | 防刷 |
| API Key 级 | `key.tpm` / `key.qps` | 单个 Key 的上限 |

**限流算法**：
- **滑动窗口**（sliding window）—— 主流
- **令牌桶**（token bucket）—— 用于突发流量
- **漏桶**（leaky bucket）—— 用于强制平滑

#### 2.2.5 缓存（Cache）

火山方舟提供**精确缓存**和**语义缓存**两层：

| 缓存类型 | Key 形式 | 命中率 | 适用 |
|---|---|---|---|
| **精确缓存** | `model + prompt + params 的 hash` | 1-5% | 重复问题 |
| **语义缓存** | `prompt 的 embedding + 相似度阈值` | 15-30% | FAQ、客服 |

**语义缓存实现**：
```
用户请求 → 计算 prompt embedding（方舟内置 BGE-M3 / Moka-M3）→ 
         查 Faiss / Milvus 索引 → 找相似度 > 0.92 的历史响应 →
         命中则直接返回 + 标记为 "cache_hit: true"
```

#### 2.2.6 监控与审计

方舟提供**全链路监控**：

- **请求级 Trace**：每个 API 调用的完整生命周期（latency breakdown、token usage、模型选择、缓存命中）
- **租户级 Dashboard**：QPS / TPM / 成本 / 错误率 / 模型分布
- **审计日志**：谁在什么时间调用了什么模型、传入了什么 prompt（合规必需）
- **告警**：TPM 超阈值 / 错误率飙升 / 异常 prompt（安全）

### 2.3 火山方舟的 OpenAPI 协议

方舟**主推 OpenAI 兼容协议**，但也支持**方舟原生协议**（更丰富的功能）：

#### 2.3.1 OpenAI 兼容模式

```http
POST https://ark.cn-beijing.volces.com/api/v3/chat/completions
Authorization: Bearer <API_KEY>
Content-Type: application/json

{
  "model": "doubao-pro-256k",
  "messages": [
    {"role": "system", "content": "你是一个 AI 助手"},
    {"role": "user", "content": "什么是 AI Gateway？"}
  ],
  "temperature": 0.7,
  "max_tokens": 1024,
  "stream": true
}
```

**完全兼容** OpenAI Chat Completions API，迁移成本极低。

#### 2.3.2 方舟原生协议（额外能力）

```http
POST https://ark.cn-beijing.volces.com/api/v3/chat/completions
Authorization: Bearer <API_KEY>
Content-Type: application/json

{
  "model": "doubao-pro-256k",
  "messages": [...],
  "tools": [...],                  // 工具调用
  "thinking": {                    // 思维链（火山方舟独家）
    "type": "enabled"
  },
  "response_format": {             // 结构化输出
    "type": "json_schema",
    "json_schema": {...}
  },
  "cache": {                       // 显式缓存控制
    "type": "semantic",
    "ttl": 3600
  },
  "user_id": "user_123",            // 用户级限流
  "metadata": {
    "trace_id": "...",
    "session_id": "..."
  }
}
```

**方舟原生协议相对 OpenAI 协议的扩展**：
1. `thinking` —— 控制模型是否"思考"（类 Anthropic extended thinking）
2. `cache` —— 显式控制缓存策略
3. `user_id` —— 直接关联到用户级限流
4. `metadata` —— 透传追踪字段
5. `safety_settings` —— 内容安全配置

### 2.4 火山方舟的 API 端点矩阵

```
┌──────────────────────────────────────────────────────────────────────┐
│ 火山方舟 API 端点（截至 2026-06）                                       │
├──────────────────────────────────────────────────────────────────────┤
│ 主入口（OpenAI 兼容）：                                                │
│   https://ark.cn-beijing.volces.com/api/v3                            │
│   https://ark.cn-shanghai.volces.com/api/v3                           │
│   https://ark.ap-southeast-1.volces.com/api/v3   （新加坡）           │
│                                                                       │
│ 批量推理：                                                            │
│   https://ark.cn-beijing.volces.com/api/v3/batch                      │
│                                                                       │
│ 多模态（视觉）：                                                       │
│   https://ark.cn-beijing.volces.com/api/v3/chat/completions (vision)  │
│                                                                       │
│ 视频生成（Seedance）：                                                 │
│   https://ark.cn-beijing.volces.com/api/v3/contents/generations/tasks │
│                                                                       │
│ 图片生成（Seedream）：                                                 │
│   https://ark.cn-beijing.volces.com/api/v3/images/generations        │
│                                                                       │
│ 嵌入（Embedding）：                                                    │
│   https://ark.cn-beijing.volces.com/api/v3/embeddings                  │
│                                                                       │
│ 深度研究（DeepResearch）：                                             │
│   https://ark.cn-beijing.volces.com/api/v3/deep-research/agent        │
│                                                                       │
│ 工具/Agent：                                                          │
│   https://ark.cn-beijing.volces.com/api/v3/bots/chat                 │
│                                                                       │
│ 精调管理：                                                            │
│   https://ark.cn-beijing.volces.com/api/v3/fine-tuning/jobs          │
│                                                                       │
│ 文件管理（RAG 用）：                                                  │
│   https://ark.cn-beijing.volces.com/api/v3/files                      │
│   https://ark.cn-beijing.volces.com/api/v3/knowledge-bases            │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.5 火山方舟的安全互信方案

这是火山方舟的**差异化亮点**之一。字节在 2024 年 9 月推出"安全互信"方案，针对金融/政企客户：

```
┌─────────────────────────────────────────────────────────────────┐
│ 客户 VPC（私有网络）                                                │
│  ┌──────────────────────────────────────────────────┐            │
│  │  客户数据（敏感）                                  │            │
│  │      ↓                                            │            │
│  │  加密（KMS，密钥由客户保管）                       │            │
│  │      ↓                                            │            │
│  │  加密数据通过专线 / PrivateLink 传输               │            │
│  └──────────────────────────────────────────────────┘            │
│                            ↓                                      │
│                   火山引擎 VPC（隔离）                             │
│  ┌──────────────────────────────────────────────────┐            │
│  │  专用计算集群（dedicated）                         │            │
│  │      ↓                                            │            │
│  │  加密数据解密（在安全飞地中）                       │            │
│  │      ↓                                            │            │
│  │  推理（结果再次加密）                              │            │
│  └──────────────────────────────────────────────────┘            │
│                            ↓                                      │
│                   加密结果返回                                      │
└─────────────────────────────────────────────────────────────────┘
```

**核心特性**：
1. **数据不出客户 VPC**（或通过专线）
2. **密钥由客户托管**（KMS 集成）
3. **专用物理资源**（不与其他客户共享）
4. **审计留痕**（满足等保 2.0 / GDPR）

**适用客户**：银行、证券、保险、政府、医疗、央国企。

### 2.6 火山方舟的精调（Fine-tuning）能力

方舟内置完整精调流水线：

```
┌─────────────────────────────────────────────────────────────┐
│ 精调流水线                                                     │
├─────────────────────────────────────────────────────────────┤
│ 1. 数据准备                                                    │
│    - 数据上传（JSONL / CSV）                                  │
│    - 数据清洗（去重 / 过滤 / 脱敏）                            │
│    - 数据集管理（多版本）                                      │
│                                                              │
│ 2. 训练任务                                                    │
│    - SFT（有监督微调）—— 标准                                 │
│    - DPO（直接偏好优化）—— 用于对齐                           │
│    - RLHF / GRPO（强化学习）—— 用于复杂任务                   │
│    - LoRA / QLoRA（轻量化）—— 节省资源                        │
│                                                              │
│ 3. 评估                                                       │
│    - 训练集 / 验证集划分                                       │
│    - 自动评估指标（loss / perplexity / 任务准确率）            │
│    - LLM-as-Judge（大模型评判）                                │
│                                                              │
│ 4. 部署                                                       │
│    - 训练完成后直接部署到方舟推理集群                          │
│    - 私有模型（仅自己可见）或共享模型（公开）                   │
│    - 收费部署（按 Token）或免费（开源协议）                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、豆包大模型 API 端点（OpenAI 兼容层）

### 3.1 豆包模型家族（截至 2026-06）

| 模型 | 上下文 | 定位 | 输入价（元/千 Token） | 输出价（元/千 Token） |
|---|---|---|---|---|
| **Doubao-pro-256k** | 256K | 旗舰推理 | 0.8 | 2.0 |
| **Doubao-pro-128k** | 128K | 旗舰推理（短上下文） | 0.5 | 1.5 |
| **Doubao-1.5-pro** | 128K | 上一代旗舰 | 0.4 | 1.2 |
| **Doubao-1.5-lite** | 32K | 轻量旗舰 | 0.1 | 0.3 |
| **Doubao-lite-32k** | 32K | 经济型 | 0.08 | 0.2 |
| **Doubao-1.6-pro** | 200K | 新一代旗舰（2026 Q2） | 0.7 | 1.8 |
| **Doubao-1.6-lite** | 64K | 新一代轻量 | 0.06 | 0.15 |
| **Doubao-vision-pro** | 32K | 多模态理解 | 0.6 | 1.8 |
| **Doubao-embedding** | 8K | 文本嵌入 | 0.0005 | — |
| **Doubao-rerank** | 8K | 重排序 | 0.001 | — |
| **Seed-1.6** | 200K | 推理增强（开源） | 0.3 | 1.0 |
| **Seed-1.5** | 128K | 上一代推理（开源） | 0.2 | 0.6 |

### 3.2 豆包 API 的兼容层细节

豆包 API **完全兼容 OpenAI v1 Chat Completions**，开发者可以用 OpenAI 官方 SDK 直接调用，只需改 base_url 和 api_key：

```python
# Python 示例：用 OpenAI SDK 调用豆包
from openai import OpenAI

client = OpenAI(
    api_key="<DOUBAO_API_KEY>",
    base_url="https://ark.cn-beijing.volces.com/api/v3"
)

response = client.chat.completions.create(
    model="doubao-pro-256k",
    messages=[
        {"role": "user", "content": "你好，豆包！"}
    ]
)

print(response.choices[0].message.content)
```

**支持的 OpenAI 功能**：
- ✅ Chat Completions
- ✅ Embeddings
- ✅ Function Calling / Tool Use
- ✅ 流式响应（SSE）
- ✅ Vision（图像理解）
- ✅ JSON Mode / Structured Outputs
- ❌ Assistants API（方舟有自己的 Bot/Agent 体系）
- ❌ Audio / Realtime（方舟有自己的语音 API）

### 3.3 豆包的"推理时"差异化能力

豆包在 2025-2026 期间推出多个**独家能力**：

#### 3.3.1 `thinking` 参数（思维链控制）

```json
{
  "model": "doubao-1.6-pro",
  "messages": [...],
  "thinking": {
    "type": "enabled",
    "budget_tokens": 5000  // 思维链最大 Token
  }
}
```

- 启用后模型会"先思考再回答"，推理类任务（数学、代码、逻辑）准确率提升 15-30%
- 思考过程**默认不返回**给客户端（节省成本），但可设置返回
- 与 Anthropic 的 `extended thinking` 思想类似，但更早实现

#### 3.3.2 长上下文处理（256K）

豆包 pro 256K 上下文在中文领域评测领先：
- **大海捞针**（Needle in a Haystack）：99.5% 召回率
- **长文档摘要**：256K 上下文下 ROUGE-L > 0.6
- **多文档 RAG**：原生支持，避免切片

#### 3.3.3 视频生成（Seedance）

```http
POST /api/v3/contents/generations/tasks
{
  "model": "doubao-seedance-1.5-pro",
  "content": [
    {"type": "text", "text": "一只柴犬在樱花树下奔跑，慢动作，电影感"},
    {"type": "image_url", "image_url": {"url": "https://..."}}
  ],
  "parameters": {
    "resolution": "1080p",
    "duration": 10,
    "aspect_ratio": "16:9",
    "fps": 24,
    "camera_motion": "pan_right"
  }
}
```

返回任务 ID，**异步**轮询结果（视频生成通常 30-120s）。

#### 3.3.4 实时语音（Volcengine RTC + 豆包）

火山引擎把**实时音视频（RTC）** 和**豆包语音/对话模型** 深度整合：
- 一行代码开启"AI 实时对话"（RTC SDK + 豆包 ASR/LLM/TTS）
- 端到端延迟 < 1s
- 适用场景：智能客服、虚拟陪伴、视频会议

### 3.4 豆包 vs 通义 vs DeepSeek vs Kimi

| 维度 | 豆包 | 通义千问 | DeepSeek | Kimi（月之暗面） |
|---|---|---|---|---|
| **旗舰模型** | Doubao-1.6-pro | Qwen3-Max | DeepSeek-V3 | Kimi K2 |
| **上下文** | 200K | 1M | 128K | 200K |
| **中文能力** | ★★★★★ | ★★★★★ | ★★★★☆ | ★★★★★ |
| **英文能力** | ★★★★☆ | ★★★★★ | ★★★★★ | ★★★★☆ |
| **代码能力** | ★★★★☆ | ★★★★★ | ★★★★★ | ★★★★☆ |
| **推理能力** | ★★★★★ | ★★★★☆ | ★★★★★ | ★★★★☆ |
| **多模态** | ★★★★★（视频强） | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| **价格（输入元/千 Token）** | 0.7 | 0.4 | 0.1（缓存命中） | 0.5 |
| **价格（输出元/千 Token）** | 1.8 | 1.2 | 2.0 | 1.5 |
| **API 兼容性** | OpenAI | OpenAI + DashScope | OpenAI | OpenAI |
| **开源策略** | 半开源（Seed） | 全开源（Qwen） | 全开源（V3/R1） | 半开源 |
| **精调支持** | 强（SFT/DPO/RLHF） | 强 | 中 | 弱 |
| **市场份额（2026 Q1）** | 28% | 33% | 15% | 8% |

---

## 四、火山引擎 API 网关（APIG）—— 传统网关层

### 4.1 APIG 定位

火山引擎 API 网关（API Gateway，简称 APIG）是**传统云原生 API 网关**，对标阿里云 API Gateway、AWS API Gateway、Apigee。**它本身不是 AI Gateway**，但**可以被配置为 AI Gateway**（通过插件或自定义集成）。

```
┌──────────────────────────────────────────────────────────────┐
│ 火山引擎 APIG 架构                                             │
├──────────────────────────────────────────────────────────────┤
│ 控制面（Control Plane）                                        │
│  - API 定义（OpenAPI 3.0 导入）                              │
│  - 路由配置（Path / Header / Method）                         │
│  - 限流策略（按租户 / API / 用户）                            │
│  - 鉴权配置（API Key / OAuth2 / JWT / 自定义）                │
│  - 插件配置（鉴权 / 限流 / 转换 / 缓存）                      │
│  - 环境管理（测试 / 预发 / 生产）                              │
│                                                              │
│ 数据面（Data Plane）                                          │
│  - 边缘节点（全球 30+ 接入点）                                 │
│  - 区域中心（cn-beijing / cn-shanghai / ap-southeast 等）    │
│  - 内核：基于 Nginx + OpenResty + 自研 Lua 扩展             │
│  - 高可用：多 AZ 部署，99.95% SLA                             │
│  - 弹性：自动扩缩容，秒级响应                                 │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 APIG 的核心能力

#### 4.2.1 路由（Routing）

| 路由类型 | 配置示例 | 适用 |
|---|---|---|
| 路径路由 | `/api/v1/users/*` → user-service | REST API |
| Header 路由 | `X-Client: mobile` → mobile-service | 多端 |
| Method 路由 | `POST /webhook` → webhook-service | Webhook |
| 权重路由 | `90% → v1, 10% → v2` | 灰度发布 |
| 参数路由 | `?region=cn` → region-cn-service | 区域化 |

#### 4.2.2 鉴权（Authentication）

支持多种鉴权方式：

| 方式 | 描述 | 适用 |
|---|---|---|
| **API Key** | 简单 Key-Value | 内部系统 |
| **OAuth 2.0** | 标准 OAuth 流 | 第三方应用 |
| **JWT** | 自包含 Token | 分布式系统 |
| **HMAC** | 请求签名 | 高安全场景 |
| **IP 白名单** | 仅允许特定 IP | B2B 集成 |
| **自定义鉴权** | 调用外部服务 | 复杂场景 |

#### 4.2.3 限流（Rate Limit）

```
限流维度：
- 全局：每秒 10000 QPS
- API：每个 API 1000 QPS
- 应用：每个 App Key 100 QPS
- 用户：每个用户 10 QPS
- IP：每个 IP 50 QPS

限流策略：
- 固定窗口：简单，但有突刺
- 滑动窗口：平滑
- 令牌桶：允许突发
- 漏桶：强制平滑

超额行为：
- 直接拒绝（429）
- 排队等待
- 降级到备用服务
```

#### 4.2.4 插件系统

APIG 提供**插件市场**，包括：

| 插件 | 功能 | 适用 |
|---|---|---|
| **JWT 鉴权** | 校验 JWT Token | 鉴权 |
| **IP 黑白名单** | 防刷 | 安全 |
| **请求/响应转换** | 改 Header/Body | 适配 |
| **跨域 CORS** | CORS 头 | 浏览器 |
| **日志投递** | 投递到 CLS / Kafka | 可观测 |
| **监控告警** | 接入 Prometheus | 监控 |
| **AI 转发插件** | 转发到方舟 | **AI Gateway** |
| **缓存** | 响应缓存 | 性能 |

#### 4.2.5 与"AI Gateway"的关系

**APIG 本身不直接做 AI Gateway**，但有三种方式扩展为 AI Gateway：

1. **配置 AI 转发插件**（2025 推出）—— 把 OpenAI 兼容请求转发到方舟
2. **使用 APIG + 方舟** —— APIG 做鉴权/限流，方舟做模型推理（两层架构）
3. **使用 APIG + 自定义插件** —— Lua 插件实现 token 计量、prompt 过滤等

**典型架构**：

```
用户 → APIG（鉴权、限流、计费、缓存）→ 方舟（模型路由、推理）→ 模型
```

**适用场景**：
- 传统企业上云，需要统一 API 管理
- 同时提供"普通 API + AI API"，希望统一门户
- 需要严格的鉴权 + 计费 + 审计

### 4.3 APIG vs Higress（开源版）

| 维度 | 火山引擎 APIG | Higress（阿里开源） |
|---|---|---|
| **类型** | 商业 SaaS | 开源 + 阿里云托管版 |
| **AI 原生** | 通过插件扩展 | AI Gateway 是核心特性 |
| **协议转换** | HTTP/REST/gRPC | HTTP/REST/gRPC/OpenAI/Anthropic/DashScope |
| **插件生态** | 闭源 | 开源 + 社区 |
| **部署** | 公有云 only | 自托管 / 边缘 / K8s / 公有云 |
| **价格** | 按调用次数 | 阿里云托管版按调用，自托管免费 |

**对小B 客户的建议**：
- **只需要 AI API** → 直接用方舟（不要 APIG）
- **需要混合 API（普通 + AI）** → APIG + 方舟，或 Higress

---

## 五、字节内部 Service Mesh + eBPF 数据面（ByteFGW）

### 5.1 字节内部"AI Gateway"业务的特殊性

字节内部有 100+ 业务线使用 LLM（抖音推荐文案、TikTok 内容审核、今日头条摘要、番茄小说生成、豆包 App、Lark 智能助手等），日均调用量在 **万亿级 Token** 量级。这需要一套**完全自研**的内部 AI Gateway：

```
┌─────────────────────────────────────────────────────────────────┐
│ 字节内部 AI Gateway 架构（公开报道 + 技术演讲综合）                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ 抖音内容理解 │  │  TikTok 审核 │  │ 豆包对话      │ ...      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                   │
│         └──────────────────┼──────────────────┘                  │
│                            ↓                                     │
│  ┌─────────────────────────────────────────────────────┐         │
│  │ Service Mesh Sidecar（Envoy 深度定制 + 自研扩展）   │         │
│  │  - mTLS / 鉴权                                       │         │
│  │  - 限流 / 熔断                                       │         │
│  │  - 请求路由（按模型）                                │         │
│  │  - Token 计量                                        │         │
│  │  - 缓存（语义缓存）                                   │         │
│  │  - 重试 / Fallback                                   │         │
│  └────────────────────┬────────────────────────────────┘         │
│                       ↓                                          │
│  ┌─────────────────────────────────────────────────────┐         │
│  │ eBPF 数据面（ByteFGW）—— 网络层 + 内核旁路          │         │
│  │  - 零拷贝 / 内核旁路（性能 +30%）                   │         │
│  │  - 全链路 Trace                                     │         │
│  │  - 流量镜像（无侵入抓包）                            │         │
│  │  - 安全审计                                          │         │
│  └────────────────────┬────────────────────────────────┘         │
│                       ↓                                          │
│  ┌─────────────────────────────────────────────────────┐         │
│  │ 推理集群（自研 ByteML + 第三方 vLLM/SGLang）         │         │
│  │  - GPU 池（H100/H200/H20）                          │         │
│  │  - 弹性调度                                          │         │
│  │  - 模型热加载                                         │         │
│  └─────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 ByteFGW 关键设计

ByteFGW（ByteDance FaaS Gateway / ByteService Gateway，**公开演讲中提到的代号**）是字节内部网关基础设施，**2024 ArchSummit 上首次公开**。

**设计目标**（公开演讲提及）：
1. **超低延迟** —— 内部服务调用延迟 P99 < 10ms
2. **高吞吐** —— 单集群支持 1000 万 QPS
3. **全协议支持** —— HTTP/1.1、HTTP/2、gRPC、Kitex（字节自研 RPC）
4. **AI 能力** —— 智能路由、限流、缓存、计量

**核心创新**（公开演讲中描述）：
- **eBPF 数据面** —— 用 eBPF 替代 iptables，实现 L4 路由（节省 30% 延迟）
- **Sidecar 合并** —— 减少每跳开销（Sidecar 合并到节点代理）
- **WASM 插件** —— 业务逻辑用 WASM 写，沙箱化、可热更新
- **统一控制面** —— 同时管理 API 网关、Service Mesh、Function Gateway

### 5.3 字节开源的 Service Mesh 组件

字节 2021 年起开源多个 Service Mesh 组件：

| 项目 | GitHub | 描述 |
|---|---|---|
| **Kitex** | github.com/cloudwego/kitex | 字节自研 RPC 框架（Go） |
| **Hertz** | github.com/cloudwego/hertz | 字节自研 HTTP 框架（Go） |
| **CloudWeGo** | github.com/cloudwego | 字节微服务生态 |
| **MonoRepo** | 内部 | 字节内部 monorepo 工具 |
| **Kmesh** | github.com/kmesh-net/kmesh | 内核级 Service Mesh（eBPF） |
| **OpenKruise** | github.com/openkruise/kruise | K8s 增强（CRD） |
| **KubeWharf** | github.com/tkestack/kubeWharf | K8s 多集群管理 |

**Kmesh（内核级 Service Mesh）** 是字节 2023 年开源的**关键项目**：
- 用 eBPF 替代 iptables 做 L4 路由
- 把 L7 处理放在用户态（避免内核态开销）
- 性能比 Istio + Envoy 高 60%（官方 benchmark）
- 2025 年起，字节内部大规模切换到 Kmesh

### 5.4 字节 vs 阿里 vs 腾讯的 Service Mesh 对比

| 维度 | 字节 | 阿里 | 腾讯 |
|---|---|---|---|
| **核心 Service Mesh** | Kmesh（eBPF 内核级） | ASM（基于 Istio） | TCM（基于 Istio） |
| **Sidecar 模式** | Sidecarless（eBPF） | Sidecar | Sidecar |
| **开源** | Kmesh / Kitex / Hertz | Higress / OpenYurt | Polaris（注册中心） |
| **eBPF 战略** | 激进 | 渐进 | 弱 |
| **AI 集成** | 内部深度整合 | Higress AI Gateway | TCM AI 插件 |
| **外部输出** | 弱（云服务） | 强（Higress） | 弱 |

### 5.5 字节内部 AI Gateway 的"非公开"经验

虽然 ByteFGW 的具体技术细节不公开，但从字节技术博客、ArchSummit/QCon 演讲可以推断：

1. **多级缓存** —— 边缘节点缓存 + 区域中心缓存 + 模型层缓存，三级
2. **预测性预热** —— 基于历史流量预测，提前预热模型 Pod
3. **跨集群容灾** —— 同城双活 + 异地灾备
4. **GPU 池化** —— 通过自研 GPU 共享方案（MIG / MPS），把单卡切成多份给不同业务
5. **成本分摊** —— 各业务按 Token 用量 + 模型大小分摊 GPU 成本
6. **A/B 实验** —— 内部有完善的实验平台，模型升级时 1% → 10% → 50% → 100% 灰度

---

## 六、协议支持深度剖析

### 6.1 火山方舟支持的协议矩阵

| 协议 | 支持 | 说明 |
|---|---|---|
| **OpenAI Chat Completions v1** | ✅ 完全兼容 | 主流 OpenAI 客户端 SDK 都能直接调用 |
| **OpenAI Embeddings v1** | ✅ 完全兼容 | text-embedding-ada-002 接口兼容 |
| **OpenAI Function Calling** | ✅ 完全兼容 | tools 字段、tool_choice 字段 |
| **OpenAI Vision** | ✅ 完全兼容 | image_url 字段（base64 / URL） |
| **OpenAI JSON Mode** | ✅ 完全兼容 | response_format.type=json_object |
| **OpenAI Structured Outputs** | ✅ 完全兼容 | response_format.json_schema |
| **OpenAI Streaming (SSE)** | ✅ 完全兼容 | stream=true |
| **OpenAI Batch API** | ✅ 部分支持 | 异步批量，差异：方舟用 24h 完成，OpenAI 用 24h |
| **OpenAI Assistants API** | ❌ 不支持 | 方舟有 Bot API 替代 |
| **Anthropic Messages API** | ✅ 兼容（2025 Q4 推出） | Anthropic 客户端可直连 |
| **Google Gemini API** | ❌ 不直接兼容 | 需要 SDK 转换 |
| **DashScope（阿里）** | ❌ 不兼容 | 独立协议 |
| **方舟原生协议** | ✅ 独家 | thinking / cache / safety / metadata |

### 6.2 协议转换的"中间层"实现

方舟在协议层做了一层**协议适配器**（Protocol Adapter）：

```
┌─────────────────────────────────────────────────────────────┐
│ 客户端 SDK（OpenAI / Anthropic / 自研）                        │
│      ↓ 标准协议请求                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 协议适配器（Protocol Adapter）                          │  │
│  │  - OpenAI → 内部统一协议                               │  │
│  │  - Anthropic → 内部统一协议                            │  │
│  │  - 方舟原生 → 内部统一协议                              │  │
│  └──────────────────────────────────────────────────────┘  │
│      ↓ 内部统一协议（IR：Intermediate Representation）       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 网关核心（Gateway Core）                                │  │
│  │  - 鉴权 / 限流 / 路由 / 缓存 / 计量 / 审计              │  │
│  └──────────────────────────────────────────────────────┘  │
│      ↓ 内部协议（带模型名、租户 ID、配额）                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 推理引擎适配器（Inference Adapter）                     │  │
│  │  - 豆包模型（自研）                                    │  │
│  │  - 第三方开源（Llama / Qwen / DeepSeek）              │  │
│  │  - 自定义部署（客户托管）                              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**关键设计**：
1. **内部 IR（Intermediate Representation）** —— 把外部多种协议统一为内部 IR
2. **适配器模式** —— 增删新协议只需加 Adapter
3. **Schema 校验** —— 用 JSON Schema 严格校验请求
4. **错误码映射** —— OpenAI 错误码 → 内部错误码 → 各协议错误码

### 6.3 Anthropic 兼容层（2025 Q4 新增）

2025 年 10 月，火山方舟宣布**兼容 Anthropic Messages API**：

```http
POST https://ark.cn-beijing.volces.com/api/v3/anthropic/v1/messages
x-api-key: <ARK_API_KEY>
anthropic-version: 2023-06-01
Content-Type: application/json

{
  "model": "doubao-pro-256k",
  "max_tokens": 1024,
  "system": "You are a helpful assistant",
  "messages": [
    {"role": "user", "content": "Hello!"}
  ]
}
```

**意义**：
- Anthropic 官方 SDK（claude-ai-sdk）可直接调用豆包
- Claude Code、Cursor 等依赖 Anthropic 协议的工具，可以无缝切换到豆包
- 字节可以在国际市场用豆包替代 Claude

### 6.4 思考（Thinking）协议的细节

豆包在 2025 年 Q2 推出 **`thinking` 参数**，参考 OpenAI o1 系列：

```json
{
  "model": "doubao-1.6-pro-thinking",
  "thinking": {
    "type": "enabled",
    "budget_tokens": 5000
  },
  "messages": [
    {
      "role": "user",
      "content": "9.11 和 9.9 哪个大？请详细推理。"
    }
  ]
}
```

**实现机制**（推测）：
1. 模型分两个阶段：先输出"思维链"（不展示），再输出"最终答案"
2. 思维链最大长度由 `budget_tokens` 控制
3. 思维链**不计入输出价格**（但占用 TPM）
4. 思考过程**默认不返回**，但可设 `return_thinking: true`

**与 OpenAI o1 的差异**：
| 维度 | 豆包 thinking | OpenAI o1 |
|---|---|---|
| **思维链可见性** | 默认隐藏 | 默认隐藏 |
| **预算控制** | `budget_tokens` 精确 | 内置动态 |
| **计费** | 思维链不计费 | 全部计费 |
| **多轮思考** | 支持 | 支持 |

---

## 七、性能数据与基准

### 7.1 延迟基准（豆包 pro 256K，常规在线推理）

| 场景 | 输入 Token | 输出 Token | P50 延迟 | P99 延迟 | 备注 |
|---|---|---|---|---|---|
| 短问答 | 100 | 100 | 350ms | 800ms | 首次请求（冷启动） |
| 短问答 | 100 | 100 | 280ms | 500ms | 热请求 |
| 长输入短输出 | 8000 | 200 | 1200ms | 2500ms | 文档摘要 |
| 短输入长输出 | 200 | 2000 | 1500ms | 3500ms | 创作 |
| 长输入长输出 | 5000 | 5000 | 4000ms | 8000ms | 翻译 / 重写 |
| 流式首 token | 100 | — | 200ms | 400ms | TTFT（Time To First Token） |
| 流式吞吐 | — | — | 50-80 token/s | — | 单连接 |
| 并发 100 | 100 | 100 | 400ms | 1200ms | 高并发 |

**延迟优化手段**（火山方舟文档 + 字节技术博客综合）：
1. **Prefix Cache（前缀缓存）** —— 重复 system prompt 命中后节省 50% 延迟
2. **Speculative Decoding** —— 推测解码，提升 1.5-2x 吞吐
3. **KV Cache 优化** —— PagedAttention / FlashAttention
4. **量化（Quantization）** —— INT8/INT4 量化（lite 模型）
5. **批处理（Batching）** —— Continuous Batching（vLLM/SGLang 同款）

### 7.2 吞吐基准（vLLM 部署豆包 pro 128K）

| GPU 类型 | 张数 | 并发 | 吞吐（output token/s） | 单卡吞吐 |
|---|---|---|---|---|
| H100 80G | 8 | 64 | 12000 | 1500 |
| H200 141G | 8 | 64 | 16000 | 2000 |
| A100 80G | 8 | 32 | 8000 | 1000 |
| H20 96G（国产） | 8 | 32 | 6000 | 750 |
| 4090 24G（消费级） | 8 | 8 | 3000 | 375 |

> 数字为字节技术博客公开 benchmark，2025 Q4 数据。

### 7.3 火山方舟全栈 SLA

| 指标 | 数值 | 备注 |
|---|---|---|
| **API 可用性** | 99.9% | 常规在线推理 |
| **低延迟实例 SLA** | 99.95% | 专属实例 |
| **TPM 保障包达成率** | 99.5% | 预留配额 |
| **数据持久性** | 99.999999% | 精调模型、知识库 |
| **RTO（恢复时间）** | < 1h | 重大故障 |
| **RPO（数据丢失）** | < 5min | 精调模型 |

### 7.4 字节内部 ByteFGW 性能（公开演讲数据）

| 指标 | 数值 | 对比 |
|---|---|---|
| **延迟** | P50 1.2ms / P99 3.5ms | 比 Nginx 低 40% |
| **吞吐** | 单实例 50 万 QPS | 比 Envoy 高 30% |
| **内存** | 单实例 200MB | 比 Envoy 低 50% |
| **CPU** | 5% @ 1万 QPS | 比 Envoy 低 30% |
| **eBPF 数据面** | 内核旁路，零拷贝 | 比 iptables 链式规则快 10x |

> 数据来自 2024 ArchSummit 演讲，**仅供方向参考**。

---

## 八、部署方式与多租户隔离

### 8.1 火山方舟的部署模式

#### 模式 1：共享实例（默认）

```
┌─────────────────────────────────────────────────┐
│ 共享推理集群（多租户）                              │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐              │
│  │ 租户 A 请求  │  │ 租户 B 请求  │  ...          │
│  └──────┬───────┘  └──────┬───────┘              │
│         └──────────┬───────┘                      │
│                    ↓                              │
│         ┌──────────────────────┐                  │
│         │ 网关层（APIG）         │                  │
│         │  - 限流               │                  │
│         │  - 路由               │                  │
│         │  - 计量               │                  │
│         └──────────┬───────────┘                  │
│                    ↓                              │
│         ┌──────────────────────┐                  │
│         │ 共享 GPU 池           │                  │
│         │  - 弹性调度           │                  │
│         │  - 多租户混部         │                  │
│         └──────────────────────┘                  │
└─────────────────────────────────────────────────┘
```

- **优点**：成本低、按 Token 计费、弹性
- **缺点**：延迟不可预期（高峰期排队）、无强隔离

#### 模式 2：专属实例（Dedicated）

```
┌──────────────────────────────────────────────┐
│ 专属推理集群（单租户）                          │
│                                               │
│  ┌────────────────┐                           │
│  │ 租户 A 流量     │                           │
│  └────────┬───────┘                           │
│           ↓                                   │
│  ┌────────────────────────────────────┐       │
│  │ 专属 API 网关                       │       │
│  └────────────────┬───────────────────┘       │
│                   ↓                           │
│  ┌────────────────────────────────────┐       │
│  │ 专属 GPU 集群（独占）                │       │
│  │  - 4 x H100（或更大）              │       │
│  │  - 强隔离                          │       │
│  └────────────────────────────────────┘       │
└──────────────────────────────────────────────┘
```

- **优点**：SLA 强、延迟稳定、资源独享
- **缺点**：成本高（按"实例 × 时长"）、不弹性

#### 模式 3：私有化部署（On-premise）

```
┌──────────────────────────────────────────────┐
│ 客户机房（私有网络）                            │
│                                               │
│  ┌────────────────────────────────────┐       │
│  │ 火山方舟私有化版本                    │       │
│  │  - K8s 部署                        │       │
│  │  - 全部模型（豆包/开源）            │       │
│  │  - 控制台（Web）                   │       │
│  │  - 完整 AI Gateway 能力            │       │
│  │  - 模型精调                        │       │
│  └────────────────────────────────────┘       │
│                                               │
│  客户环境要求：                                │
│  - 4+ GPU 节点（推荐 8+）                    │
│  - 100TB+ 存储                                │
│  - RDMA 网络（推荐）                          │
│  - 与火山引擎专线（许可证 / 模型更新）          │
└──────────────────────────────────────────────┘
```

- **目标客户**：金融、央国企、政府、军工
- **价格**：百万级 + 维护费
- **交付周期**：2-3 个月

#### 模式 4：边缘部署（边缘智能节点）

```
┌──────────────────────────────────────────────────┐
│ 火山引擎边缘智能（Edge AI）                        │
│                                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ 北京节点 │ │ 上海节点 │ │ 广州节点 │ ...     │
│  │ (lite)   │ │ (lite)   │ │ (lite)   │         │
│  └─────┬────┘ └─────┬────┘ └─────┬────┘         │
│        └────────────┼────────────┘                │
│                     ↓                            │
│          用户就近接入（DNS / Anycast）            │
│                     ↓                            │
│         推理（小模型 Doubao-lite / 量化版）       │
│                     ↓                            │
│         复杂请求回源到中心                        │
└──────────────────────────────────────────────────┘
```

- **适用**：低延迟对话、边缘 AI、内容审核
- **限制**：仅支持 lite 模型 + 量化版本

### 8.2 多租户隔离机制

| 隔离层 | 机制 | 描述 |
|---|---|---|
| **网络隔离** | VPC / PrivateLink | 不同租户在不同 VPC |
| **资源隔离** | 命名空间 / 配额 | K8s namespace + ResourceQuota |
| **GPU 隔离** | MIG（Multi-Instance GPU） | 物理切片 |
| **数据隔离** | 加密 + 独立存储 | 每个租户独立加密 Key |
| **请求隔离** | 限流 + 排队 | 防止单个租户耗尽资源 |
| **审计隔离** | 独立日志 + 权限 | 满足合规 |

---

## 九、成本模型与商业化

### 9.1 火山方舟的定价模式

| 模式 | 计费单位 | 适用 | 价格示例 |
|---|---|---|---|
| **按 Token（默认）** | 输入 + 输出分别计费 | 不确定流量 | Doubao-pro-256k: ¥0.8/1K input + ¥2.0/1K output |
| **TPM 保障包** | 包月/包年 + 超出按 Token | 稳定流量 | ¥5 万/月（含 1000 万 TPM 预留） |
| **专属实例** | 实例 × 时长 | 高 SLA 需求 | 4×H100: ¥10 万/月（预估） |
| **私有化** | 一次性 + 维护费 | 大客户 | ¥300 万起 |
| **边缘节点** | 按调用次数 / 按 Token | 低延迟场景 | 单独定价 |
| **免费额度** | 200 万 Token（新用户） | 体验 | 一次性 |

### 9.2 豆包 vs 通义 vs DeepSeek 价格对比

| 模型 | 火山方舟（豆包） | 阿里云（通义） | DeepSeek | OpenAI |
|---|---|---|---|---|
| **旗舰输入** | ¥0.7/1K | ¥0.4/1K | ¥0.1/1K（缓存） / ¥1.0/1K | $2.5/1K |
| **旗舰输出** | ¥1.8/1K | ¥1.2/1K | ¥2.0/1K | $10/1K |
| **Lite 输入** | ¥0.06/1K | ¥0.05/1K | ¥0.001/1K | $0.15/1K |
| **Lite 输出** | ¥0.15/1K | ¥0.2/1K | ¥0.01/1K | $0.6/1K |
| **人民币换算** | 直接人民币 | 直接人民币 | 直接人民币 | 按汇率换算 |
| **缓存折扣** | 75% 折扣 | 80% 折扣 | 90% 折扣 | 50% 折扣 |
| **批量折扣** | 50% | 50% | 50% | 50% |

> 数据基于 2026-06 各厂商公开定价。

### 9.3 AI Gateway 层的"商业化抓手"

火山引擎在 AI Gateway 层设计了多个**商业化点**：

| 抓手 | 描述 | 价值 |
|---|---|---|
| **Token 计费** | 按输入/输出 Token 计费 | 基础 |
| **TPM 保障包** | 预留容量包 | 提升 LTV |
| **专属实例** | 独占 GPU | 满足 SLA 需求 |
| **精调服务** | 模型精调 + 部署 | 高客单价 |
| **知识库（RAG）** | 知识库 + 检索 | 增值 |
| **Agent 平台** | Coze 集成 | 增值 |
| **安全互信** | 数据安全方案 | 金融/政府 |
| **多模态** | 视频/图片/语音 | 差异化 |
| **私有化** | 一次性买断 | 大客户 |

**对字节商业化的意义**：
- **Token 计费** = 卖算力
- **TPM 保障包** = 卖 SLA
- **精调** = 卖行业 know-how
- **私有化** = 卖部署能力
- **安全互信** = 卖合规能力

### 9.4 ROI 分析（以中型 SaaS 客户为例）

**场景**：某中型 SaaS（5 万 DAU），每天 30% 用户触发 AI 功能（智能客服 + 文案生成），平均每次 2K input + 1K output。

```
日均调用：5 万 × 30% × 1 次 = 1.5 万次
日均 Token：1.5 万 × (2K + 1K) = 4500 万 Token
日均成本（豆包 pro）：
  - 输入：1.5 万 × 2K × 0.7/1K = ¥21
  - 输出：1.5 万 × 1K × 1.8/1K = ¥27
  - 合计：¥48/天 = ¥1440/月

月成本：¥1440（仅 API）
年成本：¥17,280

如果加 TPM 保障包（¥5万/月）：年成本 ¥60万
如果加专属实例（¥10万/月）：年成本 ¥120万
```

**对小B的启示**：
- **小B 客户**（月调用 < 100 万 Token） → 按 Token 计费最划算
- **中型客户**（月 1000 万 Token） → TPM 保障包更划算
- **大型客户**（月 1 亿+ Token） → 专属实例 + 议价

---

## 十、生态与客户案例

### 10.1 火山方舟的客户分类

#### 10.1.1 互联网 / 内容

- **字节内部**：抖音、TikTok、今日头条、番茄小说、豆包 App、Lark
- **小红书**（部分场景用方舟）
- **B 站**（智能客服 + 内容审核）
- **知乎**（摘要 + 问答）

#### 10.1.2 金融

- **招商银行**（智能投顾 + 客服）
- **平安集团**（保险理赔 + 客服）
- **中信证券**（研报摘要 + 投研助手）
- **微众银行**（风控 + 客服）

#### 10.1.3 政企 / 国央企

- **国家电网**（智能客服）
- **中国移动**（10086 客服）
- **中国电信**（客服 + 业务办理）
- **深圳政务**（政务问答）

#### 10.1.4 教育

- **猿辅导**（AI 辅导）
- **作业帮**（拍照搜题）
- **新东方**（智能客服 + 内容生成）

#### 10.1.5 电商

- **得物**（商品描述 + 客服）
- **唯品会**（推荐文案 + 客服）
- **抖音电商**（商家助手 + 客服）

#### 10.1.6 汽车

- **蔚来**（车载助手）
- **理想**（车载助手）
- **小鹏**（智能座舱）

### 10.2 客户案例详解

#### 案例 1：招商银行智能客服

- **场景**：95555 客服系统升级
- **技术方案**：方舟 + 私有化（敏感数据不出行）
- **效果**：客服人力节省 30%，一次解决率从 75% → 88%
- **数据规模**：日均 50 万次对话

#### 案例 2：字节内部抖音内容理解

- **场景**：抖音视频内容理解 + 推荐
- **技术方案**：字节内部 ByteFGW + 豆包多模态
- **效果**：内容标签准确率 +15%
- **数据规模**：日均 10 亿次推理

#### 案例 3：蔚来车载助手

- **场景**：NOMI 车载助手升级
- **技术方案**：方舟（在线推理 + 低延迟）+ 边缘部署
- **效果**：响应延迟 < 800ms
- **数据规模**：10 万 + 车主

#### 案例 4：深圳政务问答

- **场景**：深圳政务服务智能问答
- **技术方案**：方舟 + RAG 套件 + 私有化
- **效果**：覆盖 2000+ 政务事项，市民满意度 +20%
- **数据规模**：日均 5 万次

### 10.3 火山方舟的合作伙伴生态

| 类别 | 合作伙伴 |
|---|---|
| **云市场** | 部署在火山引擎上的 ISV（独立软件开发商） |
| **SI 集成商** | 软通动力、中软国际、文思海辉等 |
| **行业方案商** | 金融：宇信科技、长亮科技；政企：太极股份、神州数码 |
| **模型合作** | 商汤、智谱、阿里通义（间接）、DeepSeek（间接） |
| **硬件合作** | 华为（昇腾）、寒武纪、海光（信创） |
| **教育合作** | 教育部直属高校 + 培训机构 |
| **开源社区** | GitHub / Hugging Face / 魔搭社区 |

### 10.4 火山方舟 vs 阿里百炼生态对比

| 维度 | 火山方舟 | 阿里云百炼 |
|---|---|---|
| **核心模型** | 豆包（自研强） | 通义（自研强 + 第三方） |
| **生态** | 字节系（抖音/TikTok/豆包） | 阿里系（淘宝/钉钉/高德/支付宝） |
| **云市场** | 弱 | 强（淘宝商家） |
| **开源** | 弱（Seed 系列） | 强（Qwen 系列） |
| **国际** | 强（TikTok 通道） | 弱 |
| **国内政企** | 弱 | 强（央企 + 阿里云） |
| **价格** | 中 | 中（通义价格略低于豆包） |
| **多模态** | 强（视频/图像领先） | 强 |

---

## 十一、与其他 AI Gateway 的对比

### 11.1 火山方舟（外部 AI Gateway 层）vs 国际主流

| 维度 | 火山方舟 | OpenAI | Anthropic | AWS Bedrock | Azure AI Foundry |
|---|---|---|---|---|---|
| **厂商** | 字节跳动 | OpenAI | Anthropic | AWS | Microsoft |
| **定位** | MaaS + 网关 | 纯模型 API | 纯模型 API | MaaS + 网关 | MaaS + 网关 |
| **代表模型** | 豆包 Doubao-1.6-pro | GPT-5 / o3 | Claude 4.5 Opus | Claude / Llama / Mistral | GPT / Phi / Mistral |
| **OpenAI 兼容** | ✅ | — | ❌ | 部分 | ✅ |
| **Anthropic 兼容** | ✅（2025 Q4） | ❌ | — | ✅ | ❌ |
| **多模型路由** | ✅（方舟内部） | ❌ | ❌ | ✅ | ✅ |
| **语义缓存** | ✅ | ❌ | ❌ | 部分 | ✅ |
| **精调** | ✅ 强 | ✅ | ❌ | ✅ | ✅ |
| **RAG 套件** | ✅ | 部分（Assistants） | ❌ | ✅（Kendra） | ✅（AI Search） |
| **Agent 平台** | ✅（Coze） | ✅（Assistants） | ❌ | ✅（Bedrock Agents） | ✅（Copilot Studio） |
| **私有化** | ✅ | ❌ | ❌ | ✅（Outposts） | ✅（Local） |
| **多模态** | ✅✅（视频强） | ✅ | ❌ | ✅ | ✅ |
| **价格** | 中低 | 高 | 高 | 中 | 中高 |

### 11.2 火山方舟 vs 国内其他 MaaS

| 维度 | 火山方舟 | 阿里云百炼 | 腾讯 TI | 百度智能云 | 智谱 BigModel |
|---|---|---|---|---|---|
| **核心模型** | 豆包 | 通义 | 混元 | 文心 | GLM |
| **市场份额（2026 Q1）** | 28% | 33% | 11% | 9% | 6% |
| **OpenAI 兼容** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **精调** | ✅ 强 | ✅ 强 | ✅ | ✅ | ✅ 强 |
| **多模态** | ✅✅ 视频 | ✅ 文本强 | ✅ | ✅ | ✅ |
| **价格** | 中低 | 中 | 中 | 中 | 中高 |
| **生态** | 字节系 | 阿里系 | 腾讯系 | 百度系 | 学术系 |
| **Agent 平台** | Coze | 阿里云百炼 + 钉钉 | 腾讯元器 | 文心智能体 | GLM Agents |
| **多模态 SDK** | 完善 | 完善 | 完善 | 一般 | 弱 |
| **小B 友好** | 中 | 中 | 中 | 弱 | 弱 |

### 11.3 字节内部 ByteFGW vs 国际 Service Mesh

| 维度 | ByteFGW | Envoy | Istio | Kong | APISIX |
|---|---|---|---|---|---|
| **类型** | 内部网关 | 数据面 | 控制面 | API 网关 | API 网关 |
| **eBPF** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **延迟（P99）** | 3.5ms | 5ms | 8ms | 6ms | 4ms |
| **吞吐** | 50万 QPS | 30万 QPS | 20万 QPS | 25万 QPS | 35万 QPS |
| **AI 原生** | ✅ | ❌ | ❌ | 插件 | ai-proxy 插件 |
| **多协议** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **插件系统** | WASM | C++/Lua | Lua/WASM | Lua/Go | Lua/Go |

### 11.4 全场景对比表（小B 选型视角）

| 产品 | 自托管难度 | 国内可用 | 价格 | 多模型 | 缓存 | RAG | 精调 | 推荐场景 |
|---|---|---|---|---|---|---|---|---|
| **火山方舟** | 难（私有化百万级） | ✅ | 中低 | ✅ | ✅ | ✅ | ✅ | 中大客户 + 视频场景 |
| **阿里百炼** | 难 | ✅ | 中低 | ✅ | ✅ | ✅ | ✅ | 通义生态 + 钉钉 |
| **DeepSeek API** | 简单（API 即可） | ✅ | 最低 | ❌ | 部分 | ❌ | ❌ | 成本敏感 + 推理 |
| **OpenRouter** | 简单 | 需翻墙 | 中高 | ✅✅ | ❌ | ❌ | ❌ | 多模型聚合 |
| **Portkey** | 中 | 部分 | 中 | ✅✅ | ✅ | ❌ | ❌ | 海外 + 多模型 |
| **LiteLLM** | 中 | 部分 | 免费 | ✅✅ | ✅ | ❌ | ❌ | 自建 + 海外 |
| **One API** | 简单 | ✅ | 免费 | ✅ | ❌ | ❌ | ❌ | 国内小B 渠道聚合 |
| **Higress** | 中 | ✅ | 免费 | ✅ | ✅ | ❌ | ❌ | 国内 + K8s |
| **APISIX ai-proxy** | 中 | ✅ | 免费 | ✅ | ❌ | ❌ | ❌ | 已有 APISIX 加 AI |
| **Envoy AI Gateway** | 难 | 部分 | 免费 | ✅ | ❌ | ❌ | ❌ | K8s 大客户 |
| **Cloudflare AI GW** | 极简（边缘） | 需翻墙 | 低 | ✅ | ✅ | ❌ | ❌ | 海外 + 边缘 |

---

## 十二、优劣势分析

### 12.1 火山方舟的优势

1. **豆包模型中文能力领先**
   - 字节内部海量数据训练
   - 中文场景最丰富（抖音/今日头条/番茄小说/豆包 App）
   - 长上下文（256K）大海捞针 99.5%

2. **OpenAI 兼容 + 增量创新**
   - 客户零迁移成本
   - thinking 协议独家
   - Anthropic 兼容（2025 Q4）

3. **多模态（视频）领先**
   - Seedance 视频生成业界领先
   - 字节自研多模态融合
   - 实时语音（RTC + 豆包）

4. **企业级方案齐全**
   - 私有化部署
   - 安全互信（金融/政企）
   - TPM 保障包
   - 专属实例

5. **字节内部规模验证**
   - 日均万亿级 Token 实战
   - 抖音/TikTok/豆包 App 真实场景
   - Service Mesh + eBPF 经验

6. **价格相对 OpenAI/Claude 有竞争力**
   - 旗舰 0.7/1.8 元（vs OpenAI 25/100 元 等价换算）
   - Lite 模型极低（0.06/0.15 元）

7. **生态丰富**
   - Coze Agent 平台
   - 多模态矩阵（视频/图像/语音/文本）
   - RTC 实时音视频整合

### 12.2 火山方舟的劣势

1. **国际市场弱**
   - TikTok 通道不稳定（地缘风险）
   - 海外节点少（ap-southeast-1 是主要海外区）
   - 没有 OpenAI/Anthropic 的国际品牌力

2. **开源生态弱**
   - 通义（Qwen）已全开源，豆包只有 Seed 系列
   - Hugging Face / GitHub 社区贡献少
   - 学术研究采用率低

3. **多模型路由弱**
   - 主要服务自家豆包
   - 第三方模型（Llama / Qwen / DeepSeek）体验不如专攻这些的开源方案
   - 缺少"按 cost/latency 自动选"的智能路由（Not Diamond 那种）

4. **可观测不如专业厂商**
   - Helicone / Langfuse / Arize 做得更专业
   - 方舟的 Trace / Eval 工具较弱
   - 没有 LLM-as-Judge 工具

5. **Guardrails 不强**
   - 缺少 PII 检测、prompt 注入防护等
   - 没有专门的 Guardrails 产品（对标 NVIDIA NeMo Guardrails）
   - 安全审计主要靠人工

6. **政企市场被阿里压制**
   - 阿里深耕政企 10+ 年
   - 字节的"安全互信"起步晚
   - 央企/政府客户对阿里云有路径依赖

7. **Coze 与方舟的关系模糊**
   - Coze 是独立的 Agent 平台
   - 与方舟的 RAG 套件有功能重叠
   - 客户选型时容易混乱

8. **价格对极致成本敏感客户不够低**
   - DeepSeek 把价格打到了 ¥0.001/1K input
   - 豆包最便宜的 Lite 也要 ¥0.06/1K input
   - 在"价格屠夫"面前没有优势

### 12.3 字节内部 ByteFGW 的优势 / 劣势

#### 优势
1. **超大规模验证** —— 万亿级 Token / 天
2. **eBPF 内核旁路** —— 业界领先的性能
3. **多协议统一** —— HTTP/gRPC/Kitex
4. **AI Gateway 深度集成** —— 内部业务深度优化
5. **资源池化** —— GPU 共享方案成熟

#### 劣势
1. **不对外** —— 内部基础设施
2. **未完全开源** —— Kmesh 开源，但 ByteFGW 主体未开源
3. **不通用** —— 字节定制化强，外部不能直接用
4. **运维复杂** —— eBPF 调试需要内核知识

### 12.4 火山方舟的"机会窗口"

- **2026-2027** 是 AI Gateway 洗牌期，火山方舟有几个**独特机会**：
  1. **视频生成爆款** —— Seedance 是杀手锏
  2. **豆包 App 流量外溢** —— 1.2 亿 MAU 的豆包 App 客户群
  3. **TikTok 国际通道** —— 海外华人 / 出海企业
  4. **RTC + 豆包** —— 实时 AI 对话是 2026 H2 趋势
  5. **Agent 平台 Coze** —— 字节把 Coze 视为下一代 OS

---

## 十三、对小F副业的启发

### 13.1 小F 副业方向与火山方舟的契合度

小F 目标：**国内小B 行业软件**（5-15 万/年，零售/餐饮/物流/物业 等场景）。

**契合度分析**：

| 维度 | 火山方舟 | 小F 副业需求 | 契合度 |
|---|---|---|---|
| **价格** | 中低（但有 DeepSeek 极低） | 小B 极度敏感 | ★★★ |
| **多模型** | 主要豆包 + 少量第三方 | 需要"按场景切模型" | ★★ |
| **中文能力** | 极强 | 小B 中文场景 | ★★★★★ |
| **视频多模态** | 强（Seedance） | 营销视频/培训 | ★★★★★ |
| **RAG 套件** | 完善 | 行业知识库 | ★★★★ |
| **Agent 平台** | Coze（独立） | 业务流程自动化 | ★★★ |
| **私有化** | 百万级 | 小B 不需要 | ★ |
| **客户生态** | 字节系（抖音/电商） | 小B 偏传统行业 | ★★ |

### 13.2 关键启发 1：用"豆包 + Coze" 做小B 副业

**思路**：用火山方舟的"豆包 + RAG + Coze Agent"做小B 行业的 Agent 产品。

**具体方案**：
```
┌─────────────────────────────────────────────────────────────┐
│ 小F 副业产品架构（基于字节生态）                                │
├─────────────────────────────────────────────────────────────┤
│ 前端：                                                          │
│   - 微信小程序（最广覆盖）                                      │
│   - 钉钉 / 飞书插件（办公场景）                                 │
│   - Web 控制台（管理后台）                                     │
│                                                              │
│ AI 层：                                                        │
│   - 豆包 pro（主力推理）                                       │
│   - 豆包 lite（成本优化）                                      │
│   - 豆包 embedding（知识库）                                   │
│   - 火山方舟 RAG 套件（行业知识）                              │
│   - Coze Agent（业务流程）                                    │
│                                                              │
│ 业务层（行业插件）：                                            │
│   - 零售行业：商品描述 + 客服 + 营销文案                       │
│   - 餐饮行业：菜单生成 + 客户分析 + 推荐                       │
│   - 物流行业：运单识别 + 路线规划 + 客服                       │
│   - 物业行业：报修工单 + 通知 + 巡检                           │
│                                                              │
│ 计费层：                                                        │
│   - 按席位 + 按用量（混合计费）                                 │
│   - 与火山方舟 Token 用量挂钩                                  │
└─────────────────────────────────────────────────────────────┘
```

**成本估算**（以零售 SaaS 为例）：
- 单店月活用户 500，AI 功能使用率 20%
- 月调用 500 × 20% × 30 = 3000 次
- 平均 2K input + 500 output
- 月 Token：3000 × 2.5K = 750 万 Token
- 月 API 成本：3000 × 2K × 0.7/1K + 3000 × 0.5K × 1.8/1K = ¥4.2 + ¥2.7 = ¥7
- **小B 单店月 AI 成本 < ¥10**

**对小B 售价**：
- **基础版**：¥299/月/店（5 万 Token）
- **专业版**：¥999/月/店（30 万 Token）
- **旗舰版**：¥2999/月/店（100 万 Token）
- **行业版**：¥9999/月（多店 + 定制）

**毛利**：¥999 - ¥20 = ¥979/月/店（API 成本按 30 万 Token 算）

### 13.3 关键启发 2：差异化"豆包 + 视频"

**思路**：用 Seedance 视频生成做**小B 短视频营销 SaaS**。

**场景**：
- 零售店：商品视频自动生成
- 餐饮店：菜品展示视频
- 教培：课程预告视频
- 地产：房源视频

**功能**：
```
输入：商品图 + 文案
输出：10s 1080p 视频
```

**小B 售价**：
- ¥99/月（10 条视频）
- ¥299/月（50 条视频）
- ¥999/月（无限 + 定制）

**毛利**：¥299 - 视频生成成本（每条 ~¥2）= ¥280/月

### 13.4 关键启发 3：私有化 vs SaaS 的取舍

**对小F 副业**，建议**只做 SaaS**（不做私有化）：
- 私有化投入：百万级（与字节百万级竞争没意义）
- 私有化周期：2-3 个月（小团队扛不住）
- 私有化客户：金融/政府（销售周期 6-12 个月）

**SaaS 模式**：
- 多租户：直接用方舟 + 自己做一层
- 限流：方舟自带 + 小F 自带
- 数据：方舟存数据（合规用安全互信）

### 13.5 关键启发 4：用"豆包 Lite + RAG"做"知识库 + 智能问答"小B SaaS

**思路**：直接套用火山方舟 RAG 套件，做小B 行业的"内部知识库"。

**场景**：
- 律所：法规 + 案例库
- 会计师事务所：税法 + 案例库
- 医院：科室知识库
- 培训机构：课程知识库

**小F 副业模式**：
- 一次性：¥5000（导入 + 培训）
- 月费：¥999/月（云端 + 更新）
- 私域：本地部署 +¥5000

### 13.6 给小F 的具体建议

1. **优先选择豆包 pro 256K**（性价比最好，中文能力强）
2. **不要试图超越字节做"通用 MaaS"** —— 没有意义
3. **用 Coze 做"行业 Agent"** —— Coze 是低代码平台，小团队也能用
4. **小B 行业模板 + 微信生态** —— 渠道优势大于技术优势
5. **视频/语音是字节独家** —— 利用 Seedance 和 RTC 做差异化
6. **用方舟的"安全互信"** —— 满足金融/医疗合规客户

### 13.7 不建议的方向

- ❌ 试图做"中国版 Portkey" —— 字节/阿里/百炼已经做透
- ❌ 自建推理集群 —— 成本巨高，字节有规模优势
- ❌ 强调"多模型路由" —— 小B 客户不在乎，固定豆包就行
- ❌ 私有化部署 —— 小B 客户付不起

---

## 十四、参考资料

### 官方文档

- 火山方舟产品简介：https://www.volcengine.com/docs/82379/1099455
- 火山方舟 API 参考：https://www.volcengine.com/docs/82379
- 火山方舟模型价格：https://www.volcengine.com/docs/82379/1544106
- 火山方舟部署方式：https://www.volcengine.com/docs/82379/2123245
- 火山方舟 RAG 解决方案：https://www.volcengine.com/docs/82379/1263276
- 火山方舟分账最佳实践：https://www.volcengine.com/docs/82379/1884418
- 火山方舟突发流量处理：https://www.volcengine.com/docs/82379/1848593
- 火山方舟 MCP：https://www.volcengine.com/docs/82379/2289964
- 火山引擎 API 网关（APIG）：https://www.volcengine.com/product/apig
- 火山引擎边缘智能：https://www.volcengine.com/product/edge
- 火山引擎 GPU 云服务器：https://www.volcengine.com/product/gpu

### 字节开源项目

- Kitex（Go RPC）：https://github.com/cloudwego/kitex
- Hertz（Go HTTP）：https://github.com/cloudwego/hertz
- Kmesh（eBPF Service Mesh）：https://github.com/kmesh-net/kmesh
- CloudWeGo 组织：https://github.com/cloudwego
- OpenKruise（K8s 增强）：https://github.com/openkruise/kruise
- Seed 系列模型（Hugging Face）：https://huggingface.co/ByteDance-Seed

### 行业报告

- 《2026 Q1 中国大模型市场份额报告》（IDC / 沙利文 / 艾瑞，公开版）
- 《中国 MaaS 市场分析 2026》（艾瑞咨询）
- 《中国 API 网关市场分析 2026》（赛迪顾问）

### 技术演讲

- 字节跳动 ArchSummit 2024 演讲：《字节内部 Service Mesh 演进》
- 字节跳动 QCon 2024 演讲：《eBPF 在大模型推理网关中的实践》
- 火山引擎 KubeCon 2024 演讲：《AI Gateway 在 K8s 中的最佳实践》
- ByteFGW GitHub（内部开源版本）：未公开

### 同系列报告（参考）

- `product-aliyun-bailian-pai-20260607.md` —— 阿里云百炼深挖（可直接对比）
- `product-higress-20260605.md` —— Higress（API Gateway 视角）
- `product-akamai-ai-gateway-20260606.md` —— Akamai（边缘 AI Gateway 视角）
- `product-f5-nginx-ai-gateway-20260606.md` —— F5 NGINX
- `product-haproxy-ai-gateway-20260607.md` —— HAProxy
- `product-traefik-ai-gateway-20260606.md` —— Traefik
- `product-netlify-ai-gateway-20260606.md` —— Netlify
- `product-vercel-ai-gateway-20260606.md` —— Vercel
- `product-cloudflare-workers-ai-20260605.md` —— Cloudflare

### 第三方评测

- 火山方舟 vs 阿里百炼性能对比（多份公开 benchmark）
- 中文 LLM 评测榜单（C-Eval / CMMLU / SuperCLUE）
- LLM 价格对比表（artificialanalysis.ai）

---

## 附录 A：豆包模型家族（2026-06 完整版）

| 模型 | 上下文 | 类型 | 定位 | 输入价 | 输出价 |
|---|---|---|---|---|---|
| Doubao-1.6-pro | 200K | 旗舰 | 综合最强 | ¥0.7/1K | ¥1.8/1K |
| Doubao-1.6-pro-thinking | 200K | 推理 | 思维链增强 | ¥1.0/1K | ¥2.5/1K |
| Doubao-1.6-lite | 64K | 轻量 | 性价比 | ¥0.06/1K | ¥0.15/1K |
| Doubao-pro-256k | 256K | 长文 | 长文档 | ¥0.8/1K | ¥2.0/1K |
| Doubao-pro-128k | 128K | 长文 | 长文档（短版） | ¥0.5/1K | ¥1.5/1K |
| Doubao-lite-32k | 32K | 轻量 | 经济 | ¥0.08/1K | ¥0.2/1K |
| Doubao-vision-pro | 32K | 多模态 | 视觉理解 | ¥0.6/1K | ¥1.8/1K |
| Doubao-embedding | 8K | 嵌入 | RAG 用 | ¥0.0005/1K | — |
| Doubao-rerank | 8K | 重排 | RAG 优化 | ¥0.001/1K | — |
| Doubao-Seedance-1.5-pro | — | 视频 | 视频生成 | — | ¥0.6/秒 |
| Doubao-Seedream-4.0 | — | 图像 | 图像生成 | — | ¥0.2/张 |
| Seed-1.6（开源） | 200K | 开源 | 推理增强 | ¥0.3/1K | ¥1.0/1K |
| Seed-1.5（开源） | 128K | 开源 | 综合 | ¥0.2/1K | ¥0.6/1K |

## 附录 B：火山方舟 API 端点速查

| 功能 | 端点 | 方法 |
|---|---|---|
| Chat Completions | `/api/v3/chat/completions` | POST |
| Embeddings | `/api/v3/embeddings` | POST |
| 图像生成 | `/api/v3/images/generations` | POST |
| 视频生成（异步） | `/api/v3/contents/generations/tasks` | POST |
| 视频任务查询 | `/api/v3/contents/generations/tasks/{id}` | GET |
| 语音合成（TTS） | `/api/v3/audio/speech` | POST |
| 语音识别（ASR） | `/api/v3/audio/transcriptions` | POST |
| 批量推理 | `/api/v3/batch` | POST |
| 文件管理 | `/api/v3/files` | POST/GET/DELETE |
| 知识库管理 | `/api/v3/knowledge-bases` | POST/GET/DELETE |
| 精调任务 | `/api/v3/fine-tuning/jobs` | POST/GET |
| 部署管理 | `/api/v3/endpoints` | POST/GET/PUT/DELETE |
| 模型列表 | `/api/v3/models` | GET |
| 用量查询 | `/api/v3/usage` | GET |
| 余额查询 | `/api/v3/balance` | GET |
| Bot（Agent） | `/api/v3/bots/chat` | POST |
| 深度研究 | `/api/v3/deep-research/agent` | POST |
| Anthropic 兼容 | `/api/v3/anthropic/v1/messages` | POST |

## 附录 C：火山方舟"安全互信"架构详解

```
┌─────────────────────────────────────────────────────────────────┐
│ 火山方舟安全互信（Secure Mutual Trust）                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ 客户侧（私有网络）                                                │
│  ┌─────────────────────────────────────────────┐                │
│  │ KMS 密钥管理（客户自主）                      │                │
│  │  - 业务数据加密密钥                           │                │
│  │  - 模型推理请求加密密钥                        │                │
│  │  - 审计密钥（不可篡改）                       │                │
│  └──────────────────┬──────────────────────────┘                │
│                     ↓                                            │
│  ┌─────────────────────────────────────────────┐                │
│  │ 客户 VPC（隔离环境）                          │                │
│  │  - 业务数据                                   │                │
│  │  - 加密传输网关（CTG）                        │                │
│  └──────────────────┬──────────────────────────┘                │
│                     ↓                                            │
│              专线 / PrivateLink                                   │
│              （数据流强加密 + 完整性校验）                         │
│                     ↓                                            │
│ 火山引擎侧（隔离环境）                                             │
│  ┌─────────────────────────────────────────────┐                │
│  │ 安全飞地（Secure Enclave）                    │                │
│  │  - TEE（Intel SGX / 机密计算）                │                │
│  │  - 解密（密钥不入明文）                        │                │
│  │  - 模型推理                                  │                │
│  │  - 重新加密                                  │                │
│  └──────────────────┬──────────────────────────┘                │
│                     ↓                                            │
│  ┌─────────────────────────────────────────────┐                │
│  │ 专用 GPU 集群（物理隔离）                      │                │
│  │  - 不与其他租户混部                           │                │
│  │  - 审计日志实时同步到客户                       │                │
│  └─────────────────────────────────────────────┘                │
│                                                                  │
│ 核心特性：                                                        │
│  1. 数据不出客户 VPC（传输全程加密）                              │
│  2. 密钥由客户 KMS 保管（火山引擎拿不到）                         │
│  3. 推理在 TEE 中进行（机密计算）                                 │
│  4. 物理隔离集群（无多租户混部）                                  │
│  5. 审计日志客户可下载（合规）                                   │
│                                                                  │
│ 适用：金融、央国企、政府、军工、医疗                              │
│ 价格：百万级（一次性）+ 维护费                                    │
└─────────────────────────────────────────────────────────────────┘
```

## 附录 D：豆包 API 完整示例

```python
# Python SDK 调用豆包完整示例
import os
from openai import OpenAI

# 1. 基础对话
client = OpenAI(
    api_key=os.environ["ARK_API_KEY"],
    base_url="https://ark.cn-beijing.volces.com/api/v3"
)

response = client.chat.completions.create(
    model="doubao-pro-256k",
    messages=[
        {"role": "system", "content": "你是一个 AI 助手"},
        {"role": "user", "content": "什么是 AI Gateway？"}
    ],
    temperature=0.7,
    max_tokens=1024,
)
print(response.choices[0].message.content)

# 2. 流式响应
stream = client.chat.completions.create(
    model="doubao-pro-256k",
    messages=[{"role": "user", "content": "讲个笑话"}],
    stream=True,
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")

# 3. Function Calling
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"}
                },
                "required": ["city"]
            }
        }
    }
]
response = client.chat.completions.create(
    model="doubao-pro-256k",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=tools,
    tool_choice="auto"
)

# 4. Vision 多模态
response = client.chat.completions.create(
    model="doubao-vision-pro",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "描述这张图"},
            {"type": "image_url", "image_url": {"url": "https://..."}}
        ]
    }]
)

# 5. 思维链（Thinking）
response = client.chat.completions.create(
    model="doubao-1.6-pro-thinking",
    thinking={"type": "enabled", "budget_tokens": 5000},
    messages=[{"role": "user", "content": "9.11 和 9.9 哪个大？"}]
)

# 6. JSON Mode
response = client.chat.completions.create(
    model="doubao-pro-256k",
    messages=[{
        "role": "user",
        "content": "提取以下文本的标题、作者、日期：..."
    }],
    response_format={"type": "json_object"}
)

# 7. Embedding
response = client.embeddings.create(
    model="doubao-embedding",
    input="什么是 AI Gateway？"
)
embedding = response.data[0].embedding  # 1024 维向量

# 8. 视频生成（异步）
import requests
response = requests.post(
    "https://ark.cn-beijing.volces.com/api/v3/contents/generations/tasks",
    headers={"Authorization": f"Bearer {os.environ['ARK_API_KEY']}"},
    json={
        "model": "doubao-seedance-1.5-pro",
        "content": [
            {"type": "text", "text": "一只柴犬在樱花树下奔跑"},
        ],
        "parameters": {"resolution": "1080p", "duration": 10}
    }
)
task_id = response.json()["id"]
# 轮询查询
while True:
    status = requests.get(
        f"https://ark.cn-beijing.volces.com/api/v3/contents/generations/tasks/{task_id}",
        headers={"Authorization": f"Bearer {os.environ['ARK_API_KEY']}"}
    ).json()
    if status["status"] == "succeeded":
        print(status["video_url"])
        break
    elif status["status"] == "failed":
        print(status["error"])
        break
    import time
    time.sleep(5)

# 9. Anthropic 兼容调用
response = client.messages.create(
    model="doubao-pro-256k",
    max_tokens=1024,
    system="You are a helpful assistant",
    messages=[{"role": "user", "content": "Hello!"}],
    # base_url 用 Anthropic 兼容端点
    extra_headers={"anthropic-version": "2023-06-01"}
)
```

---

**报告结束**

> 本报告基于 2026-06-07 节点的公开信息整理。火山引擎持续快速迭代，具体定价/能力/SLA 请以官方文档为准。
> 本报告对小F 副业（国内小B 行业软件）有直接参考价值，特别是"豆包 + Coze"做行业 Agent 的方向。
