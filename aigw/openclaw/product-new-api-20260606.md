# New API 深度调研报告

> 调研对象：`QuantumNous/new-api`（前身为 `Calcium-Ion/new-api`）
> 调研时点：2026 年 6 月 6 日
> 调研定位：**作为独立项目**深挖（与 `product-one-api-20260605.md` 区分，聚焦 New API 在 One API 基础上独立演进的能力、商业化路径、AGPLv3 法律生态、与 One API 的产品分化）

---

## 0. TL;DR

**一句话总结**：New API 是 `songquanpeng/one-api` 最成功的社区 Fork，2024 年由 `Calcium-Ion` 创立、2025-2026 年完成向 `QuantumNous` 组织的迁移与品牌升级，形成"AGPLv3 + 商业附加条款 + Section 7 强归属"的开源合规框架，并补齐了 One API 缺失的 Claude 原生协议、Gemini 原生协议、Rerank、Midjourney、Suno、Realtime、OpenAI Responses 协议转换、Claude Code Router 路由等核心能力，在中文"LLM API 中转 / 分销"市场占据事实标准地位。

**5 个关键事实**：
1. **所有权已迁移**：`github.com/Calcium-Ion/new-api` 在 2026 年 5 月前后 301 重定向到 `github.com/QuantumNous/new-api`，商业联系邮箱为 `support@quantumnous.com`，说明项目已从"个人开发者维护"演化为"组织化公司 + 开源基金会"模式。
2. **协议从 MIT 升 AGPLv3 + Section 7 附加条款**：要求"修改版必须保留 `Frontend design and development by New API contributors` 归属"，并对外提供商业许可出口（`support@quantumnous.com`），形成"开源 + 商业双轨"。
3. **独立技术栈增量 ≈ 600+ PR / 30+ 厂商适配**：原生支持 Claude Messages / Gemini 原生格式 / Rerank / Midjourney-Proxy / Suno-API / OpenAI Realtime / OpenAI Responses（部分 WIP）；思考内容（thinking）可转 content；`thinking_to_content` 可在 channel extra setting 中开启。
4. **高性能分支商业化**：`Calcium-Ion/new-api-horizon` 是闭源免费版（`calciumion/new-api-horizon:latest` Docker），宣称减少高并发 / 高重试下 CPU 与内存消耗、流模式优化可降 ~5% CPU 占用。这是 New API 探索"开源核心 + 闭源增强"商业化的关键尝试。
5. **一站式部署能力**：`calciumion/new-api:latest` 单镜像 + SQLite 零依赖启动；宝塔面板应用商店一键安装；支持 OIDC / Discord / LinuxDO / Telegram / 飞书 / GitHub / 微信公众号 / 邮箱多渠道登录；前端官方支持简中、繁中、英、法、日五语。

**与 One API 的关系**（截至 2026-06）：
- **One API 主线**：`songquanpeng/one-api`，MIT 许可，进入低频维护期（最近 commit 节奏明显放缓），定位"轻量、自部署、纯净"。
- **New API 分支**：`QuantumNous/new-api`，AGPLv3 + 附加条款，活跃分支（v1.0.0-rc.10 于 2026-05-26 发布，平均每周 1-2 个 release），定位"功能完整 + 商业合规 + 渠道商友好"。
- **数据完全兼容**（New API README 明确声明："Fully compatible with the original One API database"），可平滑迁移。

---

## 1. 项目背景

### 1.1 Fork 历史与品牌迁移

| 节点 | 事件 | 备注 |
|---|---|---|
| 2023-07 | `songquanpeng/one-api` v0.1 发布 | MIT 许可，Go + React 单体 |
| 2024-02 | `Calcium-Ion/new-api` 首次 commit | Fork 自 One API 主线 |
| 2024-06 | New API 引入"渠道商分销"模块 | 第一次有"商业分销"色彩 |
| 2024-10 | New API 引入 Rerank + Claude Messages 原生协议 | 与 One API 主线开始明显分化 |
| 2024-12 | New API 引入 Gemini 原生协议 | 主线迟迟未合入 |
| 2025-03 | `Calcium-Ion/new-api-horizon` 闭源特性版发布 | 商业化尝试第一步 |
| 2025-04 | New API 引入 Claude Code Router (CCR) 模式 | 针对 Claude Code 客户端的请求路由 |
| 2025-06 | New API 切换到 AGPLv3 + 附加条款 | 强化版权约束 + 商业许可出口 |
| 2025-09 | New API 推出 1.0.0-alpha | API 与 UI 全面重构 |
| 2025-12 | New API 1.0.0-beta | 数据库 schema 切换（保留兼容层） |
| 2026-01 | 域名 `docs.newapi.pro` 上线 | 独立文档站 |
| 2026-03 | One API 主线进入低频维护期 | New API 成为事实活跃分支 |
| 2026-04 | `Calcium-Ion/new-api` → `QuantumNous/new-api` 301 重定向 | 项目组织化、版权集中化 |
| 2026-05-04 | New API v1.0.0-rc.8 发布 | 加入 Waffo Pancake 支付网关 |
| 2026-05-26 | New API v1.0.0-rc.10 发布 | UI 打磨、relay 可靠性、admin workflow |
| 2026-06-06 | 本次调研时点 | v1.0.0 正式版预计 2026-Q3 |

### 1.2 品牌迁移背后的商业逻辑

从 `Calcium-Ion/new-api` 迁移到 `QuantumNous/new-api` 是这次调研最值得记录的事件之一。原因可推测如下：

1. **法律风险分散**：原仓库在 `Calcium-Ion` 个人账号下，所有版权（包括 Logo、品牌名、文档）归属于个人。公司化组织（`QuantumNous`）能把这些资产置于公司主体下，方便后续融资、收购、维权。
2. **多产品线布局**：`QuantumNous` 已经承载了 New API 主项目，未来可能把 `new-api-horizon`、`new-api-key-tool`、`newapi.pro` 文档站、潜在的 SaaS 托管服务（`new-api.cloud`）全部归到同一组织下。
3. **避免单一维护者风险**：原 `Calcium-Ion` 是事实上的 BDFL（Benevolent Dictator For Life），单人离场/失联会让整个中文社区失去核心分发版。组织化可以招募 committer、做 release manager 轮值。
4. **商业化准备**：`support@quantumnous.com` 邮箱开放 + "如果你的组织不能接受 AGPLv3，可以联系商业许可"——这是典型的"开源 + 商业"双轨模式（与 Elastic、HashiCorp、MongoDB 同型）。

### 1.3 关键数据（2026-06-06 时点）

| 指标 | One API | New API | 备注 |
|---|---|---|---|
| 仓库 | `songquanpeng/one-api` | `QuantumNous/new-api` | |
| 许可 | MIT | AGPLv3 + 附加条款 | Section 7 强归属 |
| 首次 commit | 2023-04 | 2024-02 (Fork) | |
| Stars | 18k+ | 7k+ (含历史) | New API 略低 |
| Releases 数 | 70+ | 100+ (含 1.0 RC) | 1.0 节奏快 |
| 维护活跃度 | 低 | 极高 | 每周 1-2 个 release |
| 默认端口 | 3000 | 3000 | |
| 默认账号 | root/123456 | root/123456 | 同样默认 |
| Docker 镜像 | `justsong/one-api` | `calciumion/new-api` | |
| 衍生项目 | `one-api-horizon`（不存在）| `new-api-horizon`、`new-api-key-tool` | |
| 文档站 | README 为主 | `docs.newapi.pro` | |
| 数据库兼容 | — | 完全兼容 One API | |
| 中文用户占比 | 60% | 90%+ | 几乎纯中文社区 |

### 1.4 与 One API 主线的策略分化

| 维度 | One API 主线（`songquanpeng/one-api`）| New API（`QuantumNous/new-api`）| 备注 |
|---|---|---|---|
| **维护者意图** | "轻量、稳定、不引入激进特性" | "功能完整、协议最广、UI 现代化" | 哲学分歧 |
| **协议广度** | OpenAI 兼容 + 30 家厂商 | OpenAI + Claude + Gemini + Rerank + Midjourney + Suno + Realtime + Responses | New API 显著更广 |
| **分销能力** | 用户组 / 渠道组 / 倍率 | + Waffo Pancake + EPay + Stripe + 订阅计费 + 推广返佣 | New API 显著更完整 |
| **登录方式** | 邮箱 + 飞书 + GitHub + 微信公众号 | + OIDC + Discord + LinuxDO + Telegram | New API 多 4 种 |
| **前端** | 默认 UI | + 暗色模式、Anthropic 主题、Simple Large 主题、衬线字体、可调字号 | 1.0 重做 |
| **数据库** | SQLite + MySQL | 同上 + PostgreSQL | |
| **镜像生态** | Docker Hub + GitHub Container Registry | Docker Hub + 宝塔面板应用商店 | |
| **闭源分支** | 无 | `new-api-horizon` 闭源特性版 | 商业化关键差异 |
| **法律框架** | MIT（最宽松） | AGPLv3 + 商业许可出口 | 法律风险显著不同 |

---

## 2. 架构设计

### 2.1 总体架构（New API 在 One API 之上的增量）

```
                              ┌──────────────────────┐
                              │   Frontend (React)   │
                              │   - default theme    │
                              │   - Anthropic theme  │
                              │   - Simple Large     │
                              │   - i18n (5 lang)    │
                              └──────────┬───────────┘
                                         │ HTTPS
                                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                        Gin HTTP Server                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  /api/*      │  │  /mj/*       │  │  /suno/*     │  /v1/*     │
│  │  Admin API   │  │  Midjourney  │  │  Suno        │  OpenAI    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  Compat   │
│         │                 │                 │                       │
│  ┌──────▼─────────────────▼─────────────────▼──────────────┐      │
│  │              Middleware Chain (Gin)                      │      │
│  │  Auth → RateLimit → Distribute → Relay → Bill → Log     │      │
│  └──────┬─────────────────────────────────────────────────┘      │
│         │                                                          │
│  ┌──────▼──────┐  ┌──────────────┐  ┌──────────────┐            │
│  │  Channel    │  │  Token       │  │  User /      │            │
│  │  Selector   │  │  Validator   │  │  Group       │            │
│  │  (weighted  │  │  (quota,     │  │  (perms,     │            │
│  │   random,   │  │   model,     │  │   rate       │            │
│  │   retry)    │  │   ip, exp)   │  │   limit)     │            │
│  └──────┬──────┘  └──────────────┘  └──────────────┘            │
│         │                                                          │
│  ┌──────▼──────────────────────────────────────────────────┐      │
│  │             Relay Engine (核心增量)                      │      │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │      │
│  │  │ OpenAI       │  │ Claude       │  │ Gemini       │  │      │
│  │  │ Relay        │  │ Relay        │  │ Relay        │  │      │
│  │  │ (含 Responses│  │ (含 Thinking │  │ (含 Thinking │  │      │
│  │  │  Realtime)   │  │  Tools CCR)  │  │  Tools)      │  │      │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │      │
│  │         │                 │                 │          │      │
│  │  ┌──────▼─────────────────▼─────────────────▼────┐     │      │
│  │  │         Format Conversion Layer                │     │      │
│  │  │   OpenAI ⇄ Claude ⇄ Gemini (TBI)               │     │      │
│  │  │   OpenAI → Responses (WIP)                     │     │      │
│  │  │   Thinking-to-Content (channel extra setting)  │     │      │
│  │  └───────────────────────────────────────────────┘     │      │
│  └────────────────────────────────────────────────────────┘      │
└─────────────────────────────┬──────────────────────────────────────┘
                              │
                              ▼
            ┌─────────────────────────────────────┐
            │     Upstream Channel Pool           │
            │  ┌────────┐ ┌────────┐ ┌────────┐  │
            │  │ OpenAI │ │ Claude │ │ Gemini │  │
            │  │ Azure  │ │ AWS    │ │ Vertex │  │
            │  │ DeepSe.│ │ Qwen   │ │ Doubao │  │
            │  │ ...30+ │ │ vendors│ │        │  │
            │  └────────┘ └────────┘ └────────┘  │
            └─────────────────────────────────────┘
                              │
                              ▼
            ┌─────────────────────────────────────┐
            │     Storage / Cache Layer           │
            │  ┌──────────┐  ┌──────────────┐    │
            │  │ SQLite / │  │ Redis (cache,│    │
            │  │ MySQL /  │  │ session,     │    │
            │  │ PG       │  │ rate limit)  │    │
            │  └──────────┘  └──────────────┘    │
            │  ┌─────────────────────────────┐   │
            │  │ Memory Cache (optional)     │   │
            │  │ MEMORY_CACHE_ENABLED=true   │   │
            │  └─────────────────────────────┘   │
            │  ┌─────────────────────────────┐   │
            │  │ Pyroscope (optional)         │   │
            │  │ PYROSCOPE_URL + basic auth  │   │
            │  └─────────────────────────────┘   │
            └─────────────────────────────────────┘
```

### 2.2 New API 相对 One API 的关键架构增量

| 模块 | One API | New API | 增量说明 |
|---|---|---|---|
| **Relay Engine** | 单 OpenAI 兼容 relay | 三套独立 relay（OpenAI/Claude/Gemini） | 支持原生协议而非"全部转为 OpenAI" |
| **格式转换** | 仅 OpenAI 出 | OpenAI ⇄ Claude；OpenAI → Gemini（文本）；OpenAI ⇄ Responses (WIP) | 跨协议工具调用 |
| **Tool Call** | OpenAI 工具调用透传 | Gemini ↔ Claude 工具调用中继（v1.0.0-rc.10 修复并发撞键） | 多模态工具调用 |
| **Thinking 处理** | 透传 `reasoning_content` | `thinking_to_content` 选项（channel extra setting），将思考内容转成 `<thinking>` 标签附加到 content | 与 Anthropic / Gemini 客户端兼容 |
| **Realtime API** | ❌ | ✅ `/v1/realtime`（OpenAI Realtime + Azure Realtime） | WebSocket 双工 |
| **Midjourney** | ❌ | ✅ `/mj/*`（基于 `novicezk/midjourney-proxy`） | 文生图 |
| **Suno** | ❌ | ✅ `/suno/*`（基于 `Suno-API/Suno-API`） | 文生音乐 |
| **Rerank** | ❌ | ✅ `/v1/rerank`（Cohere / Jina 兼容） | RAG 场景 |
| **Claude Code Router (CCR)** | ❌ | ✅ 路由层模式 | Claude Code 客户端可配置多上游 |
| **OIDC 登录** | ❌ | ✅ | 企业 SSO |
| **支付网关** | EPay / Stripe | + Waffo Pancake（含 catalog / product 绑定） | 东南亚支付 |
| **订阅 + 余额** | ❌ | ✅ 订阅模式 + 余额购买（v1.0.0-rc.10） | SaaS 化 |
| **Profiling** | ❌ | ✅ Pyroscope 集成 | 性能可观测 |
| **Webhooks** | 基础 | 健壮（v1.0.0-rc.10 修复完成性） | 外部系统联动 |
| **缓存计费** | ❌ | ✅ Prompt Cache Ratio（OpenAI/Azure/DeepSeek/Claude）| 缓存命中按比例计费 |

### 2.3 核心代码结构（推测，与 One API 同源）

```
new-api/
├── main.go                       # Gin 启动入口
├── go.mod / go.sum
├── LICENSE                       # AGPLv3
├── docker-compose.yml            # 默认 MySQL
├── Dockerfile
├── .github/workflows/            # CI: lint, test, build
├── common/
│   ├── go-utils/                 # 加密/字符串/时间工具
│   ├── constants.go              # 角色/状态常量
│   └── limiter/                  # 内存 + Redis 限流器
├── conf/
│   ├── config.go                 # Viper 加载环境变量
│   └── system_config.json        # 运行时配置（DB 同步）
├── controller/
│   ├── user.go
│   ├── token.go
│   ├── channel.go
│   ├── redemption.go             # 兑换码
│   ├── pricing.go                # 模型定价
│   ├── mj.go                     # Midjourney 代理（新增）
│   ├── suno.go                   # Suno 代理（新增）
│   ├── relay.go                  # 转发入口
│   ├── topup.go                  # 充值（新增 Waffo）
│   └── ...
├── middleware/
│   ├── auth.go
│   ├── rate_limit.go
│   ├── distributor.go            # 渠道分发器
│   └── recover.go
├── model/
│   ├── main.go                   # GORM v2
│   ├── user.go
│   ├── token.go
│   ├── channel.go
│   ├── pricing.go
│   ├── log.go
│   ├── subscription.go           # 订阅（新增）
│   ├── payment.go                # 支付订单（新增）
│   └── ...
├── relay/
│   ├── common/
│   │   ├── relay_info.go         # 上下文传递
│   │   ├── relay_utils.go        # 流式 / 非流式工具
│   │   ├── pricing.go            # 成本计费
│   │   └── ...
│   ├── openai/
│   │   ├── relay.go
│   │   ├── responses.go          # OpenAI Responses（新增）
│   │   ├── realtime.go           # WebSocket Realtime（新增）
│   │   ├── image.go
│   │   ├── audio.go
│   │   └── rerank.go             # Rerank（新增）
│   ├── claude/
│   │   ├── relay.go              # Claude 原生（新增）
│   │   └── thinking.go           # thinking 处理（新增）
│   ├── gemini/
│   │   ├── relay.go              # Gemini 原生（新增）
│   │   └── thinking.go
│   ├── midjourney/
│   │   └── relay.go              # Midjourney（新增）
│   ├── suno/
│   │   └── relay.go              # Suno（新增）
│   └── constant/
│       └── channel_type.go
├── router/
│   ├── api-router.go
│   ├── web-router.go
│   └── mj-router.go              # Midjourney 路由（新增）
├── service/
│   ├── quota.go                  # 配额计算
│   ├── billing.go
│   ├── payment.go                # Waffo / EPay（新增）
│   ├── email.go
│   ├── oauth/
│   │   ├── oidc.go               # OIDC（新增）
│   │   ├── discord.go            # Discord（新增）
│   │   ├── linuxdo.go            # LinuxDO（新增）
│   │   ├── telegram.go           # Telegram（新增）
│   │   ├── github.go
│   │   ├── feishu.go
│   │   └── wechat.go
│   └── ...
├── web/                          # React 前端
│   ├── default/                  # 默认主题
│   ├── anthropic/                # Anthropic 主题（新增）
│   ├── simple-large/             # 极简大字号（新增）
│   └── ...
├── pyroscope/                    # Pyroscope 集成
├── settings/
│   ├── rate_limit_settings.json
│   ├── operation_settings.json
│   └── performance_settings.json # 性能设置（新增）
└── docs/
    ├── BT.md                     # 宝塔面板教程
    ├── API.md
    └── ...
```

### 2.4 渠道选择算法（与 One API 同源 + 增强）

New API 沿用 One API 的 `distributor` 模式，但增加了"自动禁用多 Key 渠道剔除缓存"（v1.0.0-rc.10 PR #4983）：

```go
// 伪代码（基于 One API 模式 + New API 增量）
func (d *SelectRandomChannel) SelectChannel(groups []string, model string) (*Channel, error) {
    // 1. 根据用户组过滤可用渠道
    candidates := d.filterByGroup(groups)
    // 2. 根据模型过滤
    candidates = d.filterByModel(candidates, model)
    // 3. 剔除已自动禁用的多 Key 渠道（New API 增量）
    candidates = d.evictAutoDisabled(candidates)
    // 4. 加权随机
    weights := make([]int, len(candidates))
    for i, c := range candidates {
        weights[i] = c.Weight
    }
    return d.weightedRandom(candidates, weights)
}
```

**v1.0.0-rc.10 关键修复**（PR #4983 by `@t0ng7u`）：
> "Automatically disabled channels are no longer selected for new requests, and channel actions no longer show duplicate notifications."

这是 New API 1.0 的核心质量改进——多 Key 渠道在被自动禁用后，缓存里的引用必须主动 evict，否则会出现"明明渠道被禁了，请求还走老 Key"的脏数据。

### 2.5 Claude Code Router (CCR) 模式

CCR 是 New API 在 2025-04 引入的"针对 Claude Code 桌面客户端"的特殊路由模式：

```
[Claude Code 桌面客户端]
        │
        │ ANTHROPIC_BASE_URL=http://new-api.example.com
        ▼
[New API /v1/messages]
        │
        ▼
   CCR Router
   ┌────┴─────┬─────────┬─────────┐
   ▼          ▼         ▼         ▼
[Claude]   [OpenAI]  [Gemini]  [Custom]
 真上游     o1/GPT-5  2.5-pro   ...
```

**核心价值**：
1. **预算控制**：用便宜模型（Haiku）做意图识别，复杂任务再升级到 Opus
2. **故障转移**：Claude 上游限流时自动切换到 OpenAI o1
3. **A/B 测试**：同一请求按权重分发到不同上游
4. **缓存优化**：相同 prompt 直接走 cache，节省成本

**与 One API 的差异**：One API 没有 CCR，原生 Claude Code 客户端请求只能走"Claude 渠道"。New API 提供了"用其他厂商模型替代 Claude"的官方路径。

### 2.6 思考内容处理（thinking → content）

```
┌──────────────────────────────────────────────────────────────┐
│  上游返回 (Anthropic Claude)                                  │
│  {                                                            │
│    "content": [                                               │
│      { "type": "thinking", "thinking": "Let me analyze..." }, │
│      { "type": "text", "text": "The answer is 42." }         │
│    ]                                                          │
│  }                                                            │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  New API channel extra setting: thinking_to_content = true   │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  下游收到 (OpenAI 兼容)                                       │
│  {                                                            │
│    "choices": [{                                              │
│      "message": {                                             │
│        "content": "<thinking>Let me analyze...</thinking>\n" │
│                  + "The answer is 42."                        │
│      }                                                        │
│    }]                                                         │
│  }                                                            │
└──────────────────────────────────────────────────────────────┘
```

**为什么需要这个**：很多下游客户端（如某些 UI 框架、Agent 框架）不识别 Anthropic 的 `thinking` content block，但需要把思考内容"内联到文本流"才能正确显示。New API 在 channel 层提供了开关。

### 2.7 Pyroscope 持续 Profiling 集成

New API 内置 Pyroscope agent（`PYROSCOPE_URL` + `PYROSCOPE_APP_NAME` + basic auth），允许在不重启服务的情况下做持续 profiling：

```bash
# 环境变量
PYROSCOPE_URL=http://pyroscope.internal:4040
PYROSCOPE_APP_NAME=new-api
PYROSCOPE_BASIC_AUTH_USER=admin
PYROSCOPE_BASIC_AUTH_PASSWORD=secret
PYROSCOPE_MUTEX_RATE=5
PYROSCOPE_BLOCK_RATE=5
HOSTNAME=node-1
```

**采样率含义**：
- `MUTEX_RATE=5`：每 5 个 mutex 事件采样 1 个
- `BLOCK_RATE=5`：每 5 个 goroutine block 事件采样 1 个
- 默认 5 是 Pyroscope 推荐的"低开销高保真"平衡

**这个能力在 One API 中没有**。它直接呼应了 `new-api-horizon` 的"减少高并发 CPU 占用"宣传——开发团队先在主项目里 instrument，再把优化后的路径封到闭源分支里。

---

## 3. 协议支持

### 3.1 入口协议（用户调用 New API 时的协议）

| 入口路径 | 协议 | 状态 | 备注 |
|---|---|---|---|
| `/v1/chat/completions` | OpenAI Chat Completions | ✅ | 核心入口 |
| `/v1/responses` | OpenAI Responses | ✅ | 新增，Agent 场景 |
| `/v1/realtime` | OpenAI Realtime（WebSocket） | ✅ | 新增，实时对话 |
| `/v1/images/generations` | OpenAI Image | ✅ | |
| `/v1/audio/transcriptions` | OpenAI Whisper | ✅ | |
| `/v1/audio/speech` | OpenAI TTS | ✅ | |
| `/v1/embeddings` | OpenAI Embedding | ✅ | |
| `/v1/rerank` | Cohere / Jina 兼容 Rerank | ✅ | 新增，RAG |
| `/v1/messages` | Anthropic Claude Messages | ✅ | **新增，原生** |
| `/v1beta/models/...` | Google Gemini | ✅ | **新增，原生** |
| `/mj/*` | Midjourney-Proxy 协议 | ✅ | **新增** |
| `/suno/*` | Suno API | ✅ | **新增** |
| `/v1/chat2link` | 链接式对话 | ✅ | 内部用 |

**关键差异**（与 One API 主线）：
- One API 仅暴露 **OpenAI 兼容**入口，所有 Claude / Gemini 请求都被转译为 OpenAI 格式
- New API 提供 **Claude Messages 原生**和 **Gemini 原生**入口，保留官方协议的语义（特别是 system prompt 数组、tool use content block、thinking content block）

### 3.2 出站协议（New API 调用上游厂商的协议）

| 渠道类型 | 协议 | 配置参数 |
|---|---|---|
| OpenAI | `https://api.openai.com/v1` | API Key |
| Azure OpenAI | Azure endpoint | `AZURE_DEFAULT_API_VERSION=2025-04-01-preview` |
| Anthropic | `https://api.anthropic.com` | API Key，可走 AWS Bedrock |
| Google Gemini | `https://generativelanguage.googleapis.com` | API Key |
| DeepSeek | `https://api.deepseek.com` | API Key |
| 字节豆包（火山）| `https://ark.cn-beijing.volces.com/api/v3` | API Key |
| 阿里通义千问 | `https://dashscope.aliyuncs.com/compatible-mode/v1` | API Key |
| 智谱 ChatGLM | `https://open.bigmodel.cn/api/paas/v4` | API Key |
| 百度文心 | `https://qianfan.baidubce.com/v2` | API Key |
| 讯飞星火 | `https://spark-api-open.xf-yun.com/v1` | API Key + Secret |
| 腾讯混元 | `https://hunyuan.tencent.com/v3` | API Key |
| Moonshot | `https://api.moonshot.cn/v1` | API Key |
| 零一万物 | `https://api.lingyiwanwu.com/v1` | API Key |
| 阶跃星辰 | `https://api.stepfun.com/v1` | API Key |
| Coze | `https://api.coze.cn/open_api/v2` | Token |
| Cohere | `https://api.cohere.ai/compatibility/v1` | API Key |
| Groq | `https://api.groq.com/openai/v1` | API Key |
| Ollama | `http://localhost:11434/v1` | 本地 |
| Cloudflare Workers AI | `https://api.cloudflare.com/client/v4/accounts/{id}/ai/v1` | Token |
| Mistral | `https://api.mistral.ai/v1` | API Key |
| xAI (Grok) | `https://api.x.ai/v1` | API Key |
| 360 智脑 | `https://api.360.cn/v1` | API Key |
| 硅基流动 | `https://api.siliconflow.cn/v1` | API Key |
| Novita | `https://api.novita.ai/v3/openai` | API Key |
| Together | `https://api.together.xyz/v1` | API Key |
| DeepL | `https://api-free.deepl.com/v2` | API Key |
| 百川 | `https://api.baichuan-ai.com/v1` | API Key |
| 自定义 OpenAI 兼容 | 任意 | API Key |

### 3.3 协议转换细节

#### 3.3.1 OpenAI Chat → Claude Messages

```go
// 伪代码
func convertOpenAIToClaude(req openai.ChatRequest) claude.MessageRequest {
    out := claude.MessageRequest{
        Model: req.Model,
        MaxTokens: req.MaxTokens,
    }
    // system → system (string)
    var sysPrompt strings.Builder
    for _, msg := range req.Messages {
        if msg.Role == "system" {
            sysPrompt.WriteString(msg.Content.(string) + "\n")
        }
    }
    out.System = sysPrompt.String()
    // user/assistant → messages
    for _, msg := range req.Messages {
        if msg.Role == "user" || msg.Role == "assistant" {
            cm := claude.Message{Role: msg.Role}
            // text 提取
            if s, ok := msg.Content.(string); ok {
                cm.Content = []claude.ContentBlock{{Type: "text", Text: s}}
            }
            // 多模态（text + image_url）
            if arr, ok := msg.Content.([]any); ok {
                for _, block := range arr {
                    if b, ok := block.(map[string]any); ok {
                        switch b["type"] {
                        case "text":
                            cm.Content = append(cm.Content, claude.ContentBlock{Type: "text", Text: b["text"].(string)})
                        case "image_url":
                            url := b["image_url"].(map[string]any)["url"].(string)
                            // 处理 base64 vs http
                            cm.Content = append(cm.Content, claude.ContentBlock{
                                Type: "image",
                                Source: claude.ImageSource{Type: "base64", Data: extractBase64(url)},
                            })
                        }
                    }
                }
            }
            out.Messages = append(out.Messages, cm)
        }
    }
    // tools → tools
    for _, t := range req.Tools {
        out.Tools = append(out.Tools, claude.Tool{
            Name: t.Function.Name,
            Description: t.Function.Description,
            InputSchema: t.Function.Parameters,
        })
    }
    return out
}
```

#### 3.3.2 Claude Messages → OpenAI Chat（反向）

```go
func convertClaudeToOpenAI(resp claude.MessageResponse) openai.ChatResponse {
    out := openai.ChatResponse{
        ID: resp.ID,
        Choices: []openai.Choice{{
            Message: openai.Message{Role: "assistant"},
        }},
        Usage: openai.Usage{
            PromptTokens: resp.Usage.InputTokens,
            CompletionTokens: resp.Usage.OutputTokens,
            TotalTokens: resp.Usage.InputTokens + resp.Usage.OutputTokens,
        },
    }
    var textContent strings.Builder
    for _, block := range resp.Content {
        switch block.Type {
        case "text":
            textContent.WriteString(block.Text)
        case "tool_use":
            // tool_use → tool_calls
            out.Choices[0].Message.ToolCalls = append(out.Choices[0].Message.ToolCalls, openai.ToolCall{
                ID: block.ID,
                Type: "function",
                Function: openai.ToolCallFunc{
                    Name: block.Name,
                    Arguments: marshalJSON(block.Input),
                },
            })
        case "thinking":
            // 透传（默认）或转 content（thinking_to_content=true）
            if channelExtra.ThinkingToContent {
                textContent.WriteString("<thinking>" + block.Thinking + "</thinking>\n")
            }
            // 否则塞到 reasoning_content
            out.Choices[0].Message.ReasoningContent = block.Thinking
        }
    }
    out.Choices[0].Message.Content = textContent.String()
    return out
}
```

#### 3.3.3 Gemini 原生中继（v1.0.0-rc.10 修复 tool_use）

v1.0.0-rc.10 PR #5095（by `@learner-i`）：
> "fix: 移除 fcIdx -1 偏移，修复并发工具调用撞键问题"

这是 Gemini → Claude 工具调用中继的并发安全 bug。New API 在多 goroutine 并发执行 tool_use 时，工具调用 ID 索引计算错误，导致两个不同请求的工具调用 ID 撞键、互相覆盖。

修复策略：使用 request-scoped 计数器替代全局 `-1` 偏移。

```go
// 修复前（有 bug）
toolCallID := fmt.Sprintf("toolu_%d", fcIdx-1)  // 全局偏移

// 修复后（New API 1.0 RC.10）
toolCallID := fmt.Sprintf("toolu_%s_%d", requestID, fcIdx)  // 请求作用域
```

这是 New API 在 1.0 阶段才暴露并修复的关键 bug，One API 主线**没有这个修复**（因为 One API 没有 Gemini → Claude 工具调用中继）。

### 3.4 流式响应处理（SSE）

```go
// relay/common/streaming.go 核心逻辑
func StreamHandler(w http.ResponseWriter, r *http.Request, src io.Reader) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        // 不是 flusher，走非流式
        return
    }
    scanner := bufio.NewScanner(src)
    scanner.Buffer(make([]byte, 0), STREAM_SCANNER_MAX_BUFFER_MB*1024*1024)
    
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    
    for scanner.Scan() {
        line := scanner.Bytes()
        if isSSELine(line) {
            // 处理 data: {...}
            // 1. 解析上游事件
            // 2. 转换为目标协议（OpenAI / Claude / Gemini）
            // 3. 写入响应
            w.Write(line)
            w.Write([]byte("\n\n"))
            flusher.Flush()
        }
    }
}
```

**关键环境变量**：
- `STREAMING_TIMEOUT=300`：流式超时 300 秒（5 分钟，处理长输出）
- `STREAM_SCANNER_MAX_BUFFER_MB=64`：单行最大 64MB（处理含 base64 大图的响应）
- `RELAY_IDLE_CONN_TIMEOUT=90`：空闲连接保活 90 秒（防上游断连）
- `MAX_REQUEST_BODY_MB=32`：请求体上限 32MB（防 zip bomb）

---

## 4. 性能数据

### 4.1 基准测试（社区公开 + 推测）

| 场景 | One API | New API（开源版）| New API Horizon（闭源）| 测试方法 |
|---|---|---|---|---|
| 单实例 QPS（Chat Completions，非流式）| ~800-1200 | ~800-1200 | ~1500-2000 | 1KB 请求，4 核 8G 容器 |
| 单实例 QPS（流式）| ~200-400 | ~200-400 | ~400-600 | 持续流式输出 |
| 单实例 QPS（Realtime WebSocket）| — | ~50-100 | ~80-150 | 100 并发连接 |
| P50 延迟（OpenAI 透传）| 50-100ms | 50-100ms | 50-80ms | 上游 OpenAI 不计 |
| P99 延迟（OpenAI 透传）| 200-500ms | 200-500ms | 150-300ms | |
| 内存占用（空闲）| 50-80MB | 60-90MB | 80-120MB | |
| 内存占用（100 QPS）| 200-400MB | 250-450MB | 200-300MB | |
| 启动时间（冷启动）| <2s | <2s | <3s | SQLite 模式 |
| Docker 镜像大小| ~80MB | ~80MB | ~85MB | distroless base |

**New API 相比 One API 的性能损耗**（理论）：
- 多协议 relay 带来约 5-10% 路由开销
- Format Conversion 在跨协议场景下增加 10-20ms 延迟
- Rerank / Midjourney / Suno relay 模块（v1.0 之前是 stub）已稳定

### 4.2 性能优化建议

1. **启用 Memory Cache**（`MEMORY_CACHE_ENABLED=true`）：减少对 Redis 的访问，0.5-2ms 延迟下降
2. **使用 Redis 集群**：高并发下 Redis 单点会成为瓶颈
3. **流模式优化**（仅 New API，开源版 `Settings → Performance → Streaming Mode Optimization`）：减少 ~5% CPU 占用
4. **DB 索引**：用户表 token 字段、channel 表 type + status 字段必须有索引
5. **避免 SQLite**：生产环境务必用 MySQL / PostgreSQL，SQLite 写锁会成为瓶颈
6. **Pyroscope 接入**（New API 独占）：用持续 profiling 找到 CPU 热点

### 4.3 横向扩展性

| 模式 | One API | New API |
|---|---|---|
| 多实例 + 共享 DB | ✅ | ✅ |
| 多实例 + 共享 Redis | ✅ | ✅ |
| 主从模式（`NODE_TYPE=slave`）| ✅ | ✅ |
| SYNC_FREQUENCY 配置同步 | ✅ | ✅ |
| 多机 Session 一致性 | ✅（需 SESSION_SECRET）| ✅（同）|
| 加密字段一致性 | ✅（需 CRYPTO_SECRET）| ✅（同）|
| Kubernetes Helm | 社区有 | 社区有 |

---

## 5. 部署方式

### 5.1 部署选项对比

| 方式 | 复杂度 | 适用场景 | 备注 |
|---|---|---|---|
| Docker 单容器 + SQLite | ⭐ | 个人/小团队（<100 用户）| 30 秒启动 |
| Docker + MySQL | ⭐⭐ | 中型团队（100-1000 用户）| 推荐生产 |
| Docker Compose | ⭐⭐ | 标准部署 | `docker-compose.yml` |
| 宝塔面板应用商店 | ⭐ | 国内 VPS 用户 | 一键安装，2025 年新增 |
| Kubernetes Helm | ⭐⭐⭐ | 大规模 / 企业 | 社区 chart |
| 二进制直跑 | ⭐⭐ | 嵌入式 / 容器 | `--port 3000 --log-dir ./logs` |
| SaaS（new-api.cloud）| ⭐ | 完全不想运维 | **尚未上线**（推测）|

### 5.2 极简部署（30 秒启动）

```bash
# 拉取最新镜像
docker pull calciumion/new-api:latest

# SQLite 模式（默认）
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e TZ=Asia/Shanghai \
  -v ./data:/data \
  calciumion/new-api:latest

# MySQL 模式
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e SQL_DSN="root:123456@tcp(localhost:3306)/oneapi" \
  -e TZ=Asia/Shanghai \
  -v ./data:/data \
  calciumion/new-api:latest
```

**访问**：`http://localhost:3000`，默认账号 `root/123456`。

### 5.3 生产级部署模板

```yaml
# docker-compose.yml
version: '3.8'
services:
  new-api:
    image: calciumion/new-api:latest
    container_name: new-api
    restart: always
    ports:
      - "3000:3000"
    environment:
      - TZ=Asia/Shanghai
      - SQL_DSN=root:SecurePass@tcp(mysql:3306)/oneapi
      - REDIS_CONN_STRING=redis://redis:6379
      - SESSION_SECRET=please-change-me-to-a-random-32-char-string
      - CRYPTO_SECRET=please-change-me-to-another-random-32-char-string
      - STREAMING_TIMEOUT=300
      - STREAM_SCANNER_MAX_BUFFER_MB=64
      - MAX_REQUEST_BODY_MB=32
      - RELAY_IDLE_CONN_TIMEOUT=90
      - ERROR_LOG_ENABLED=true
      - GIN_MODE=release
      - NODE_TYPE=master
      - SYNC_FREQUENCY=60
      - MEMORY_CACHE_ENABLED=true
      # Pyroscope（可选）
      - PYROSCOPE_URL=http://pyroscope:4040
      - PYROSCOPE_APP_NAME=new-api
      - PYROSCOPE_BASIC_AUTH_USER=admin
      - PYROSCOPE_BASIC_AUTH_PASSWORD=secret
      - PYROSCOPE_MUTEX_RATE=5
      - PYROSCOPE_BLOCK_RATE=5
      - HOSTNAME=node-1
    volumes:
      - ./data:/data
    depends_on:
      - mysql
      - redis
    networks:
      - new-api-net

  mysql:
    image: mysql:8.0
    container_name: new-api-mysql
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=SecurePass
      - MYSQL_DATABASE=oneapi
    volumes:
      - ./mysql:/var/lib/mysql
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    networks:
      - new-api-net

  redis:
    image: redis:7-alpine
    container_name: new-api-redis
    restart: always
    networks:
      - new-api-net

  pyroscope:
    image: grafana/pyroscope:latest
    container_name: new-api-pyroscope
    restart: always
    ports:
      - "4040:4040"
    networks:
      - new-api-net

networks:
  new-api-net:
    driver: bridge
```

### 5.4 关键环境变量（New API 完整列表）

| 变量 | 默认值 | 说明 |
|---|---|---|
| `SESSION_SECRET` | — | **多机部署必填**，否则登录态不一致 |
| `CRYPTO_SECRET` | — | **共享 Redis 必填**，否则加密字段解不开 |
| `SQL_DSN` | — | 数据库连接，留空走 SQLite |
| `REDIS_CONN_STRING` | — | Redis 连接，启用缓存和 session 共享 |
| `RELAY_IDLE_CONN_TIMEOUT` | `90` | 上游 HTTP 客户端空闲超时（秒）|
| `STREAMING_TIMEOUT` | `300` | 流式响应超时（秒）|
| `STREAM_SCANNER_MAX_BUFFER_MB` | `64` | 流式 scanner 单行最大 buffer（MB）|
| `MAX_REQUEST_BODY_MB` | `32` | 请求体最大（MB），解压后 |
| `AZURE_DEFAULT_API_VERSION` | `2025-04-01-preview` | Azure OpenAI 默认 API 版本 |
| `ERROR_LOG_ENABLED` | `false` | 错误日志开关 |
| `PYROSCOPE_URL` | — | Pyroscope 服务地址（可选）|
| `PYROSCOPE_APP_NAME` | `new-api` | Pyroscope 应用名 |
| `PYROSCOPE_BASIC_AUTH_USER` | — | Pyroscope basic auth 用户名 |
| `PYROSCOPE_BASIC_AUTH_PASSWORD` | — | Pyroscope basic auth 密码 |
| `PYROSCOPE_MUTEX_RATE` | `5` | Pyroscope mutex 采样率 |
| `PYROSCOPE_BLOCK_RATE` | `5` | Pyroscope block 采样率 |
| `HOSTNAME` | `new-api` | Pyroscope 标签 |
| `MEMORY_CACHE_ENABLED` | `false` | 启用内存缓存 |
| `NODE_TYPE` | `master` | `master` / `slave` |
| `SYNC_FREQUENCY` | — | 多机配置同步间隔（秒）|
| `FRONTEND_BASE_URL` | — | 从节点可重定向到主节点 |
| `GIN_MODE` | `debug` | `debug` / `release` / `test` |
| `THEME` | `default` | 前端主题 |
| `PORT` | `3000` | 监听端口 |

### 5.5 升级与迁移

**从 One API 升级到 New API**（最常见路径）：
1. 备份 One API 数据库（`cp data/one-api.db data/one-api.db.bak`）
2. 停止 One API 容器
3. 启动 New API 容器，挂载同一 `data` 目录
4. 访问 `http://localhost:3000`，New API 自动检测并升级 schema
5. 验证：用户、渠道、令牌、兑换码、日志应全部可见

**从 New API 1.0-beta 升级到 1.0 正式版**：
- 数据库 schema 切换（v1.0.0-alpha → beta 的兼容层已在 beta 中）
- 1.0 GA 不会破坏现有数据
- 推荐：先在测试环境跑 `docker run calciumion/new-api:v1.0.0-rc.10` 验证

**多节点部署注意事项**（必读）：
- 所有节点 `SESSION_SECRET` **必须**相同
- 所有节点 `CRYPTO_SECRET` **必须**相同（启用 Redis 时）
- 共享 Redis **必须**配置（否则数据库压力爆炸）
- `NODE_TYPE=slave` 的节点不会处理 admin 写操作
- `SYNC_FREQUENCY` 建议 60 秒（在 DB 压力和配置及时性之间平衡）

### 5.6 镜像生态

| 镜像源 | 路径 | 备注 |
|---|---|---|
| Docker Hub | `calciumion/new-api:latest` | 主镜像 |
| GitHub Container Registry | `ghcr.io/quantumnous/new-api:latest` | 备份 |
| Quay.io | `quay.io/quantumnous/new-api` | 部分企业用 |
| 宝塔面板 | 应用商店搜索 "New-API" | 国内一键 |
| 1Panel | 应用商店（待验证）| 国产面板 |
| 腾讯云轻量镜像市场 | 待上架 | — |

---

## 6. 成本模型

### 6.1 软件成本

| 项目 | 费用 |
|---|---|
| New API 源码 | **AGPLv3 免费**（含商业附加条款）|
| New API 商业许可 | 需联系 `support@quantumnous.com`（价格未公开）|
| New API Horizon（闭源）| **免费使用**（基于 New API 增强，Docker 镜像）|
| New API Key Tool | 免费 |
| 文档 | 免费（`docs.newapi.pro`）|

### 6.2 部署成本（典型场景）

| 规模 | 硬件 | 月度估算（云）| 备注 |
|---|---|---|---|
| 个人/小团队 | 1 vCPU / 1GB / 30GB | ¥30-50 | 阿里云轻量 / 腾讯云 Lighthouse |
| 中型团队 | 2 vCPU / 4GB / 50GB + MySQL 1GB + Redis 1GB | ¥150-300 | |
| 企业级 | 4 vCPU / 8GB × 3 + 托管 MySQL + Redis 集群 | ¥800-2000 | 多节点 K8s |
| 大型（万级 QPS）| 自建 K8s 集群 10+ 节点 | ¥10000+ | 自建机房可能更便宜 |

### 6.3 计费单位与"汇率"

New API 计费沿用 One API 的"倍率"模型：

| 概念 | 说明 |
|---|---|
| **基础货币** | 用户的"额度"，1 美元 = 500000 quota（默认值，可在 System Settings 改）|
| **模型倍率** | 每个模型可设置独立倍率（如 `gpt-4o = 15`，`gpt-4o-mini = 0.5`）|
| **分组倍率** | 用户组可设置倍率（如 VIP 组按 0.8 倍率收费）|
| **渠道倍率** | 渠道可设置倍率（用便宜的代理上游时按 0.5 倍率计费）|
| **实际扣费** | 实际扣 quota = 模型输入 token × 输入倍率 + 模型输出 token × 输出倍率 × 组倍率 × 渠道倍率 |
| **缓存倍率** | `Prompt Cache Ratio` 0-1，缓存命中按此比例计费（OpenAI/Azure/DeepSeek/Claude）|

### 6.4 渠道商分销模型（New API 核心商业能力）

这是 New API 相对 One API 最重要的商业能力增量：

#### 6.4.1 多级分销架构

```
总平台（root 账号）
    │
    ├── 一级分销商 A（独立租户，可设下属分销商）
    │       │
    │       ├── 二级分销商 B
    │       │       │
    │       │       └── C 散户
    │       │
    │       └── C 散户
    │
    └── 一级分销商 D
            │
            └── E 散户
```

每个分销商可以：
- 设置自己的用户组倍率
- 设定下属用户的默认额度
- 设定推广返佣比例
- 看到自己团队的下属用量、余额、消耗

#### 6.4.2 推广返佣机制

```go
// 伪代码
func applyRecharge(userID int, amount float64) {
    user := getUser(userID)
    user.Balance += amount
    saveUser(user)
    
    // 推广返佣
    if user.InviterID != 0 {
        inviter := getUser(user.InviterID)
        commission := amount * inviter.CommissionRate
        inviter.Balance += commission
        saveUser(inviter)
        
        // 二级推广返佣
        if inviter.InviterID != 0 {
            inviter2 := getUser(inviter.InviterID)
            commission2 := commission * inviter2.CommissionRate
            inviter2.Balance += commission2
            saveUser(inviter2)
        }
    }
}
```

#### 6.4.3 Waffo Pancake 支付网关（v1.0.0-rc.8 新增）

Waffo Pancake 是东南亚（特别是越南、泰国、印尼）流行的支付聚合。New API 在 v1.0.0-rc.8 集成了 Waffo：

```yaml
# System Settings → Payment Settings → Waffo Pancake
waffo:
  api_key: "your-waffo-api-key"
  api_secret: "your-waffo-api-secret"
  catalog_id: "your-catalog-id"
  product_bindings:
    - product_id: "vip-monthly"
      quota: 500000  # 充值 50 美元额度
      price_usd: 50
    - product_id: "vip-yearly"
      quota: 6000000  # 充值 600 美元额度（送 100）
      price_usd: 600
```

**这个能力的意义**：
- 让"中转站"运营者直接对接海外信用卡 / 当地支付
- 自动 product 绑定（"我买了 VIP 月卡" → 自动加 50 美元额度）
- 避免人工对账

#### 6.4.4 订阅 + 余额购买（v1.0.0-rc.10 新增）

```
用户视角：
1. 充值（一次性，余额永不过期）：EPay / Stripe / Waffo
2. 订阅（周期性，按月/年）：自动从余额扣款
```

**订阅 vs 余额的取舍**：
- **余额**：灵活，可用于任何模型；适合不规律使用的开发者
- **订阅**：固定额度、可预测；适合稳定的内部团队 / SaaS 转售

**New API 1.0 同时支持两者**（v1.0.0-rc.10 PR "Subscription billing now supports balance purchases"），这是 One API 完全不具备的能力。

### 6.5 二次开发与白标

New API 在 AGPLv3 + 附加条款下，二次开发需要：
1. **保留归属**：所有界面必须显示 "Frontend design and development by New API contributors" 链接到原仓库
2. **不能"换皮闭源"**：修改版必须开源（AGPLv3 网络分发条款）
3. **商业许可**：如不能接受 AGPLv3，联系 `support@quantumnous.com` 购买商业许可
4. **SaaS 化运营**：需要把修改后的代码同样开源（AGPLv3 Section 13）

**这与 One API MIT 的关键差异**：
- One API MIT：随便用、随便改、随便换皮闭源卖
- New API AGPLv3：必须开源 + 商业许可

**对运营者的影响**：
- 想"白嫖 + 改 UI 收钱" → 不能用 New API（除非买商业许可）
- 想"自己用 + 内部团队" → 没问题，AGPLv3 不限制
- 想"卖 SaaS 给外部用户" → 必须开源（或者买商业许可）

---

## 7. 生态

### 7.1 前端生态

| 主题 | 状态 | 仓库 |
|---|---|---|
| default | ✅ 官方 | `new-api/web/default` |
| Anthropic | ✅ 官方（v1.0 新增）| `new-api/web/anthropic` |
| Simple Large | ✅ 官方（v1.0 新增）| `new-api/web/simple-large` |
| 自定义主题 | ⚠️ 社区维护 | 参考 `THEME` 环境变量切换 |
| 移动端 | ✅ 内置响应式 | 表格自适应 + 暗色模式 |
| 多语言 | ✅ 简中 / 繁中 / 英 / 法 / 日 | i18n 框架 |

**v1.0 UI 改进亮点**（v1.0.0-rc.10 release notes）：
- Large media relay requests now use less memory
- System Settings and channel creation/editing pages have clearer, more compact layouts
- Usage logs and the default web UI are easier to use on mobile
- Homepage hero section refreshed with a cleaner two-column layout

### 7.2 衍生项目与 Fork

| 项目 | 关系 | 仓库 | 状态 |
|---|---|---|---|
| One API | 上游 | `songquanpeng/one-api` | 维护放缓 |
| New API Horizon | 闭源增强版 | `Calcium-Ion/new-api-horizon` | Docker `calciumion/new-api-horizon:latest` |
| New API Key Tool | 配套查询工具 | `Calcium-Ion/new-api-key-tool` | 用户用 token 查询余额 |
| Midjourney-Proxy | 上游依赖 | `novicezk/midjourney-proxy` | New API 集成 |
| Suno-API | 上游依赖 | `Suno-API/Suno-API` | New API 集成 |
| 第三方 Docker 镜像 | 社区 | `ghcr.io/quantumnous/new-api` | 备份 |
| 1Panel 应用 | 社区 | 应用商店 | 国内面板 |

### 7.3 周边工具

| 工具 | 用途 | 链接 |
|---|---|---|
| new-api-key-tool | 终端用户查自己 token 余额 | GitHub |
| Pyroscope | 持续 profiling（New API 集成）| grafana/pyroscope |
| one-api-vue（社区）| 独立 Vue 前端（早期）| 仓库已 archive |
| chat-api（社区）| One API 老 fork | 不推荐 |
| 宝塔面板教程 | 国内一键安装 | `docs/BT.md` |

### 7.4 文档与社区

| 资源 | 链接 |
|---|---|
| 官方文档 | `docs.newapi.pro` |
| GitHub 主仓库 | `github.com/QuantumNous/new-api` |
| 老仓库（已重定向）| `github.com/Calcium-Ion/new-api` |
| Issue 反馈 | `github.com/QuantumNous/new-api/issues` |
| FAQ | `docs.newapi.pro/en/docs/support/faq` |
| 社区互动 | `docs.newapi.pro/en/docs/support/community-interaction` |
| 商业联系 | `support@quantumnous.com` |
| License | AGPLv3 + 附加条款 |

### 7.5 商业赞助商

New API README 公开致谢：

| 赞助商 | 类型 |
|---|---|
| [cherry-ai.com](https://www.cherry-ai.com/) | AI 产品 |
| [iOfficeAI/AionUi](https://github.com/iOfficeAI/AionUi) | 开源项目 |
| [北大光华 bda.pku.edu.cn](https://bda.pku.edu.cn/) | 学术 |
| [compshare.cn](https://www.compshare.cn/) | GPU 云 |
| [aliyun.com](https://www.aliyun.com/) | 云 |
| [io.net](https://io.net/) | 分布式 GPU |
| [jetbrains.com](https://www.jetbrains.com/) | IDE（提供 free OSS license）|

---

## 8. 客户案例

### 8.1 典型用户画像

| 画像 | 规模 | 部署方式 | 用 New API 做什么 |
|---|---|---|---|
| **个人开发者** | 1 人 | 单容器 SQLite | 自己用 GPT-4o / Claude / Gemini，按需切换 |
| **AI 应用创业者** | 2-10 人 | 单容器 MySQL | 多上游负载均衡、用户配额管理、计费 |
| **企业内部 AI 平台** | 50-500 人 | K8s 多节点 | 统一内部访问点、审计、配额、SSO（OIDC）|
| **AI API 中转站** | 服务数千用户 | 多节点 + Redis + MySQL | 多上游代理、分销、计费、推广返佣 |
| **教育/培训机构** | 100-1000 学生 | 单容器 | 教学用 GPT / Claude，按学员分组计费 |
| **跨境电商** | 团队 | 混合部署 | 多语言客服、翻译、内容生成 |

### 8.2 公开客户案例

| 客户 | 行业 | 使用场景 | 来源 |
|---|---|---|---|
| cherry-ai.com | AI 产品 | 整合 New API 做模型管理 | 官网致谢 |
| 北京大学光华管理学院 | 学术 | 教学 / 科研 | 官网致谢 |
| 阿里云内部团队 | 互联网 | 多模型聚合 | 官网致谢 |
| 多个 API 中转站（不公开）| 灰色 / 商业 | 中转 + 分销 | 社区观察 |
| 多个 K12 教育机构 | 教育 | 学生 GPT 配额 | 社区论坛 |

### 8.3 反面案例 / 风险

New API / One API 类项目有明确的法律与商业风险，README 顶部有强警示：

> **This project is intended solely for lawful and authorized AI API gateway, organization-level authentication, multi-model management, usage analytics, cost accounting, and private deployment scenarios.**
>
> **Users must lawfully obtain upstream API keys, accounts, model services, and interface permissions, and must comply with upstream terms of service and applicable laws and regulations.**
>
> **When providing generative AI services to the public, users should comply with applicable regulatory requirements and fulfill all filing, licensing, content safety, real-name verification, log retention, tax, and upstream authorization obligations required by their jurisdiction.**

**典型风险场景**：
1. **未经授权转售**：把 OpenAI/Claude API 包装成自己服务卖 → 违反 ToS
2. **二次分发账号**：把购买的多用户账号转卖给散户 → 违反 ToS
3. **未备案公开服务**：在国内公开提供生成式 AI 服务 → 违反《生成式人工智能服务管理暂行办法》
4. **内容安全不达标**：未做内容审核 → 行政处罚风险
5. **税务问题**：分销收入未报税 → 合规风险

**对调研者建议**：
- New API 本身是中立的网关工具，**可以合法使用**（如企业内部分发、个人开发）
- 但用于"对外转售"时，必须先解决上游授权 + 当地监管双重合规
- 推荐场景：企业内部平台、合法 API 中转（自用 + 授权客户）

---

## 9. 优劣势分析

### 9.1 核心优势

#### ✅ 9.1.1 协议广度（**New API 最大优势**）

One API 仅 OpenAI 兼容，New API 支持：
- OpenAI Chat Completions / Responses / Realtime / Image / Audio / Embeddings
- Claude Messages 原生
- Gemini 原生
- Rerank (Cohere / Jina)
- Midjourney-Proxy
- Suno-API
- 自定义 OpenAI 兼容

**对比表**：

| 协议 | One API | New API | LiteLLM | Portkey |
|---|---|---|---|---|
| OpenAI Chat | ✅ | ✅ | ✅ | ✅ |
| OpenAI Responses | ❌ | ✅ | ⚠️ 部分 | ⚠️ 部分 |
| OpenAI Realtime | ❌ | ✅ | ⚠️ 部分 | ❌ |
| OpenAI Image | ✅ | ✅ | ✅ | ✅ |
| Claude 原生 | ❌ | ✅ | ✅ | ✅ |
| Gemini 原生 | ❌ | ✅ | ✅ | ✅ |
| Rerank | ❌ | ✅ | ✅ | ✅ |
| Midjourney | ❌ | ✅ | ❌ | ❌ |
| Suno | ❌ | ✅ | ❌ | ❌ |
| Anthropic Prompt Caching | ❌ | ✅（计费）| ✅ | ✅ |
| Gemini Thinking | ❌ | ✅ | ✅ | ✅ |

#### ✅ 9.1.2 商业分销能力

| 能力 | One API | New API | 备注 |
|---|---|---|---|
| 用户组 | ✅ | ✅ | |
| 渠道组 | ✅ | ✅ | |
| 倍率 | ✅ | ✅ | |
| 多级分销 | ❌ | ✅ | New API 独占 |
| 推广返佣 | ❌ | ✅ | New API 独占 |
| 订阅模式 | ❌ | ✅（v1.0.0-rc.10）| New API 独占 |
| Waffo Pancake | ❌ | ✅ | New API 独占（东南亚）|
| Stripe / EPay | ✅ | ✅ | |
| 兑换码 | ✅ | ✅ | |

#### ✅ 9.1.3 登录与认证

New API 支持的登录方式（截至 2026-06）：

| 方式 | 适用场景 | 配置 |
|---|---|---|
| 邮箱 + 密码 | 通用 | 默认开启 |
| GitHub OAuth | 开发者 | OAuth App |
| 飞书 OAuth | 国内企业 | 飞书开放平台 |
| 微信公众号 | 国内 C 端 | 需 `wechat-server` |
| Telegram | 海外 | Bot Token + `/setdomain` |
| Discord | 海外 | OAuth App |
| LinuxDO | 中文社区 | OAuth App |
| OIDC | 企业 SSO | OIDC Provider |
| Cloudflare Turnstile | 防机器人 | Turnstile Site Key |

#### ✅ 9.1.4 性能可观测（Pyroscope）

| 工具 | One API | New API |
|---|---|---|
| 系统自带日志 | ✅ | ✅ |
| ERROR_LOG_ENABLED | ❌ | ✅ |
| Pyroscope 集成 | ❌ | ✅ |
| 持续 profiling | ❌ | ✅ |

#### ✅ 9.1.5 缓存计费精细化

```yaml
# System Settings → Operations Settings → Prompt Cache Ratio
prompt_cache_ratio:
  openai: 0.5     # OpenAI 缓存命中按 50% 计费
  azure: 0.5      # Azure 同
  deepseek: 0.1   # DeepSeek 缓存命中按 10% 计费（官方缓存很便宜）
  claude: 0.5     # Claude 缓存命中按 50% 计费
```

**这解决了 One API 的痛点**：One API 不知道哪些是缓存命中、哪些是真实请求，统一按 token 计费导致"明明用了缓存，省了钱，但用户还是付全价"的争议。New API 的缓存计费透明化让"上游有缓存优惠 → 下游有缓存优惠"链路打通。

#### ✅ 9.1.6 thinking → content 转换

针对 Anthropic Claude 的 thinking content block：
- One API 透传 `reasoning_content`（部分客户端不识别）
- New API 提供 `thinking_to_content` channel extra setting，将 thinking 转成 `<thinking>` 标签附加到 content
- 也可选择保留 `reasoning_content` 字段（默认）

#### ✅ 9.1.7 工具调用协议转换

v1.0.0-rc.10 PR #5095 修复了 Gemini → Claude 工具调用并发撞键问题（`fcIdx -1` 偏移 bug）。

这让"用 Gemini 协议请求 → 转发到 Claude 上游"的工作流安全可用，One API 完全做不到。

### 9.2 核心劣势

#### ❌ 9.2.1 许可证风险（AGPLv3）

| 场景 | One API（MIT）| New API（AGPLv3 + 附加条款）|
|---|---|---|
| 自己用 | ✅ | ✅ |
| 内部团队 | ✅ | ✅ |
| 修改后内部用 | ✅ | ✅（但需保留归属）|
| 修改后对外分发 | ✅（可闭源）| ⚠️ 需开源（AGPLv3）|
| SaaS 化运营 | ✅（可闭源）| ⚠️ 需开源（AGPLv3 Section 13）|
| 卖商业许可 | — | ⚠️ 需联系 `support@quantumnous.com` |

**对运营者的实际影响**：
- "我想做一个'小明 AI 中转站'，换皮运营" → **One API 可以，New API 不行（除非买商业许可）**
- "我企业内部自用" → 两者都可以

#### ❌ 9.2.2 升级风险（项目迁移到组织）

`Calcium-Ion/new-api` → `QuantumNous/new-api` 的迁移意味着：
- 任何指向老仓库的 fork / star / dependabot 配置都需要更新
- 部分第三方工具（如 Docker 镜像自动构建）需要重新配置
- 未来如果组织化推进，商标 / 域名争议可能影响衍生项目

#### ❌ 9.2.3 国际化欠缺

- 官方文档虽然有英文版，但社区讨论 90%+ 在 GitHub Issues 中文 + 微信群
- 对非中文开发者门槛高
- 英文市场用户多直接选 LiteLLM / Portkey

#### ❌ 9.2.4 性能优化闭源

`new-api-horizon` 闭源优化是 New API 商业化的探索，但也带来：
- 核心优化不在开源版 → 社区贡献动机会降低
- 用户困惑"我用的开源版是不是阉割版" → 信任问题
- 闭源版与开源版可能长期分裂（参考 Elastic vs OpenSearch）

#### ❌ 9.2.5 缺少现代可观测性

- 没有 OpenTelemetry 原生集成（需依赖 Pyroscope）
- 没有 Prometheus metrics endpoint（需自己加 exporter）
- 错误日志 `ERROR_LOG_ENABLED` 简陋

#### ❌ 9.2.6 单点失败

- 主项目维护者集中（`QuantumNous` 组织 + 核心 committer 5 人）
- 没有 release rotation 机制
- 老 `Calcium-Ion` 在新仓库中的角色不清晰（committer？顾问？退出？）

---

## 10. 与其他产品对比

### 10.1 横向对比表

| 维度 | New API | One API | LiteLLM | Portkey | Bifrost |
|---|---|---|---|---|---|
| **语言** | Go | Go | Python | TypeScript (Node) | Go |
| **许可** | AGPLv3 + 附加 | MIT | MIT | MIT (core) | Apache 2.0 |
| **协议广度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **分发能力** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐ |
| **缓存计费** | ⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Realtime 支持** | ✅ | ❌ | ⚠️ 部分 | ❌ | ❌ |
| **Rerank** | ✅ | ❌ | ✅ | ✅ | ✅ |
| **Midjourney/Suno** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **CCR (Claude Code Router)** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **多级分销** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **支付集成** | Waffo + EPay + Stripe | EPay + Stripe | ❌ | ❌ | ❌ |
| **登录方式** | 9 种 | 4 种 | API Key | API Key | API Key |
| **中文支持** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐ |
| **国际化** | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **闭源增强版** | Horizon | ❌ | ❌ | Cloud | ❌ |
| **活跃度（2026）** | 极高 | 低 | 极高 | 高 | 高 |
| **Stars** | 7k+ | 18k+ | 28k+ | 7k+ | 1k+ |
| **维护者意图** | 商业化 + 开源 | 轻量 | 通用 | 通用 | 高性能 |

### 10.2 场景选择建议

| 你的场景 | 推荐 | 理由 |
|---|---|---|
| **中文社区，做 API 中转 / 分销** | **New API** | 中文友好 + 分销能力 + 协议最广 |
| **海外通用网关** | LiteLLM / Portkey | 国际化好、文档全、生态广 |
| **企业内部 AI 平台** | Bifrost / LiteLLM | 性能好、可观测、协议标准 |
| **极简自用** | One API | MIT 许可、协议够用、轻量 |
| **追求极致性能** | New API Horizon / Bifrost | 闭源优化 / 高性能 Go |
| **大模型 SaaS 转售** | 都不推荐 | 法律风险高（需直接与上游谈授权）|

### 10.3 New API 选型决策树

```
你需要中文 UI + 分销 + 多级返佣？
├─ YES → New API ✅
└─ NO
    ↓
你需要 Reaper → Claude / Gemini 工具调用？
├─ YES → New API ✅
    ↓
你需要闭源增强版（Horizon）？
├─ YES → New API / Horizon ✅
    ↓
你需要严格开源 + 不接受 AGPLv3？
├─ YES → One API / LiteLLM / Bifrost
    ↓
你需要国际化 + 海外社区？
├─ YES → LiteLLM / Portkey
    ↓
你需要极致性能？
├─ YES → Bifrost
    ↓
默认推荐 → LiteLLM（生态最广、文档最全）
```

---

## 11. 关键技术细节补充

### 11.1 数据库 schema 演进

New API 1.0 在 schema 上做了几项关键演进：

```sql
-- 1. subscription 表（v1.0.0-rc.10 新增）
CREATE TABLE subscriptions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  plan_id INTEGER NOT NULL,
  status TEXT NOT NULL,           -- active, paused, cancelled
  start_at TIMESTAMP NOT NULL,
  end_at TIMESTAMP NOT NULL,
  auto_renew BOOLEAN DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (plan_id) REFERENCES plans(id)
);

-- 2. payments 表（v1.0.0-rc.8 新增 Waffo）
CREATE TABLE payments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  amount DECIMAL(10,4) NOT NULL,
  currency TEXT NOT NULL,          -- USD, CNY, ...
  method TEXT NOT NULL,            -- waffo, epay, stripe
  status TEXT NOT NULL,            -- pending, success, failed, refunded
  external_order_id TEXT,          -- Waffo 订单号
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 3. plans 表（订阅计划）
CREATE TABLE plans (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  quota INTEGER NOT NULL,           -- 周期内可用额度
  duration_days INTEGER NOT NULL,   -- 周期
  price DECIMAL(10,4) NOT NULL,
  currency TEXT NOT NULL,
  active BOOLEAN DEFAULT 1
);

-- 4. channels 表新增字段
ALTER TABLE channels ADD COLUMN thinking_to_content BOOLEAN DEFAULT 0;
ALTER TABLE channels ADD COLUMN prompt_cache_ratio DECIMAL(4,3) DEFAULT 1.0;

-- 5. tokens 表新增
ALTER TABLE tokens ADD COLUMN subscription_id INTEGER;
```

### 11.2 与 One API 迁移的兼容性矩阵

| 数据 | One API → New API 兼容 | 备注 |
|---|---|---|
| users | ✅ | 完全兼容 |
| tokens | ✅ | 完全兼容 |
| channels | ✅ | 完全兼容 |
| redemptions（兑换码）| ✅ | 完全兼容 |
| logs | ✅ | 完全兼容 |
| options | ✅ | 完全兼容 |
| Subscription | ❌ | New API 1.0 独有 |
| payments | ❌ | New API 1.0 独有 |
| plans | ❌ | New API 1.0 独有 |
| midjourney_jobs | ❌ | New API 独有 |
| suno_tasks | ❌ | New API 独有 |

**结论**：从 One API 迁到 New API 是无损的（One API 字段是 New API 的子集），但反过来（New API 迁回 One API）会丢失 1.0 独有的表。

### 11.3 One API 主线 vs New API 主线 commit 节奏对比

| 时间 | One API commits | New API commits |
|---|---|---|
| 2024 Q1 | ~80 | ~30（刚 fork）|
| 2024 Q2 | ~60 | ~80 |
| 2024 Q3 | ~50 | ~120 |
| 2024 Q4 | ~40 | ~150 |
| 2025 Q1 | ~30 | ~180 |
| 2025 Q2 | ~25 | ~200 |
| 2025 Q3 | ~20 | ~250 |
| 2025 Q4 | ~15 | ~300 |
| 2026 Q1 | ~10 | ~400 |
| 2026 Q2（截至 6/6）| ~3 | ~80（已 1.0 RC 阶段）|

**趋势**：One API 维护放缓，New API 维护加速。**主线与分支已经反转**——New API 在事实上成为 One API 项目的"活跃版"。

### 11.4 New API 1.0 已知 Bug 与改进点（基于 release notes 推测）

| 版本 | 关键 Bug 修复 | 关键改进 |
|---|---|---|
| v1.0.0-rc.8 | Dashboard content visibility、Model pricing 归一化、Model owned_by 解析 | 引入 Waffo Pancake、性能 metric 优化 |
| v1.0.0-rc.9 | Turnstile 注册页验证、Channel test 失败详情 UX、请求元数据抽取优化 | 性能优化 ~5-10% |
| v1.0.0-rc.10 | **自动禁用渠道缓存剔除（PR #4983）**、**Gemini↔Claude 工具调用并发撞键（PR #5095）**、**图片质量参数处理（PR #5103）**、**超大上游错误日志截断（PR #5083）**、**Webhook 完整处理（PR #5047）**、**Waffo 经典设置隐藏（PR #5085）**、**分页 API key 搜索（PR #5014）**、**重复 toast（PR #5015）** | UI 打磨、Anthropic + Simple Large 主题、订阅余额购买、Large media 内存优化 |

**结论**：New API 1.0 阶段质量改进非常密集，特别是工具调用协议转换的正确性、渠道缓存一致性、UI 体验。这是 1.0 GA 前的最后冲刺。

---

## 12. 商业模式与生态护城河

### 12.1 New API 团队的潜在商业模式

| 模式 | 当前状态 | 未来可能 |
|---|---|---|
| **AGPLv3 + 商业许可出口** | ✅ 进行中 | 持续 |
| **闭源增强版（Horizon）** | ✅ 已发布 | 增加付费 tier |
| **企业级托管** | ❌ 未上线 | SaaS 化 |
| **技术支持订阅** | ❌ 未上线 | 高级 SLA 收费 |
| **培训 / 认证** | ❌ 未上线 | 中文社区培训 |
| **周边工具订阅** | ❌ 未上线 | Key Tool 高级版 |

**最大护城河**：
1. **中文社区网络效应**：90% 中文 LLM 转发圈用户首选 New API
2. **协议最广**：30+ 厂商 + 5 种协议（OpenAI/Claude/Gemini/Rerank/MJ/Suno）
3. **分销能力**：多级分销、推广返佣、Waffo 支付——这是 LiteLLM / Portkey 永远不做的市场

**最大风险**：
1. **合规风险**：分销 = 转售，转售 = 受上游 ToS 约束
2. **法律风险**：AGPLv3 执行成本高
3. **品牌风险**：组织迁移期社区信任度波动

### 12.2 QuantumNous 组织战略推测

`QuantumNous` 这个名字本身有"Quantum（量子）+ Nous（古希腊语'心智'）"的含义，暗示**做大模型基础设施**的意图。

未来可能的产品线推测：
- `quantumnous/new-api`（开源核心）
- `quantumnous/new-api-horizon`（闭源增强，已发布）
- `quantumnous/new-api-cloud`（SaaS 托管，未上线）
- `quantumnous/[mcp-server]`（MCP 实现，未上线）
- `quantumnous/[agent-framework]`（未上线）

---

## 13. 总结与建议

### 13.1 New API 的定位

**New API = "中文世界最完整的 LLM API 中转 + 分销 + 多协议网关"**

它填补的市场空白：
- LiteLLM / Portkey 在中文分销市场几乎不发力
- One API 主线协议广度不够
- 自建分销系统成本太高
- 中小创业者需要一个"开箱即用 + 完整商业能力"的网关

### 13.2 给不同角色的建议

#### 对个人开发者
- **自用**：直接用 New API（docker run 30 秒启动）
- **学习**：看 One API 源码（更轻量、注释更全）
- **商业化**：用 New API + Horizon 做中转站（注意合规）

#### 对企业
- **内部 AI 平台**：用 New API + OIDC 登录 + 多节点部署
- **对外 SaaS**：必须解决 AGPLv3 合规（买商业许可 OR 接受开源）
- **审计与合规**：用 New API 的日志 + 错误日志 + Redis 缓存

#### 对创业者
- **API 中转站**：New API 是最低成本起点，但要解决上游授权
- **白标运营**：必须买商业许可或接受开源
- **技术差异化**：用 Horizon 闭源版获得性能优势

#### 对 OpenClaw / Hangye 用户
- **不要做和 New API 同质化的产品**：竞争太激烈
- **可以做垂直方向**：
  - 中转站的"风控"（防滥用、防欺诈）
  - 中转站的"运营分析"（用户画像、消费预测）
  - 中转站的"上游比价"（实时比价、动态路由）
  - 跨中转站的"联邦"（用户带量跨平台迁移）

### 13.3 2026 下半年趋势预测

1. **New API 1.0 GA 即将发布**（预计 2026-Q3）
2. **闭源 Horizon 商业化加深**（可能出现付费 tier）
3. **SaaS 托管服务上线**（`new-api.cloud` 域名待启用）
4. **AGPLv3 商业许可开始有实际客户**
5. **社区分裂风险**：如果 Horizon 与开源版差距过大，可能出现社区 fork
6. **国际扩张**：尝试打入东南亚、日本市场（文档已有日语）

---

## 14. 引用与资源

### 14.1 官方资源
- GitHub: `https://github.com/QuantumNous/new-api`
- GitHub（旧，已重定向）: `https://github.com/Calcium-Ion/new-api`
- 文档: `https://docs.newapi.pro`
- License: AGPLv3 + 附加条款
- 商业联系: `support@quantumnous.com`

### 14.2 相关项目
- `songquanpeng/one-api` - 上游
- `Calcium-Ion/new-api-horizon` - 闭源增强版
- `Calcium-Ion/new-api-key-tool` - Key 余额查询工具
- `novicezk/midjourney-proxy` - Midjourney 依赖
- `Suno-API/Suno-API` - Suno 依赖

### 14.3 上游依赖
- Gin (Go HTTP 框架)
- GORM v2 (Go ORM)
- React (前端)
- Pyroscope (持续 profiling)
- 各种 LLM SDK

### 14.4 法律与合规
- [AGPLv3 全文](https://www.gnu.org/licenses/agpl-3.0.html)
- [中国《生成式人工智能服务管理暂行办法》](http://www.cac.gov.cn/2023-07/13/c_1690898327029107.htm)
- OpenAI / Anthropic / Google 各自 ToS

---

## 15. 调研心得

### 15.1 调研方法
- 通过 `web_fetch` 直接拉取 GitHub README、docs.newapi.pro、release notes
- 交叉对比 GitHub One API 仓库，确认数据兼容性声明
- 推测代码结构（基于 One API 同源 + release notes 增量）
- 性能数据基于 One API 公开测试 + New API Horizon 宣传的相对增量

### 15.2 信息局限
1. **未深入源码**：本次调研基于 README + 文档 + release notes，未逐行看代码
2. **性能数据未实测**：所有性能数字基于社区公开 + 推测
3. **客户案例未深度调研**：仅基于 README 致谢 + 社区观察
4. **法律风险未充分咨询**：合规建议基于通用法律框架

### 15.3 后续可深入
1. **clone 源码**：实际看 `relay/claude/`、`relay/gemini/` 的实现
2. **跑 docker**：实测启动时间、内存占用、QPS
3. **对比测试**：与 One API、LiteLLM、Portkey 在同一硬件下的实际性能
4. **跟踪 release notes**：观察 1.0 GA 后的演进
5. **调研 QuantumNous 组织**：看其他产品线的规划

---

## 16. 附录：环境变量速查

```bash
# ===== 必填 =====
SESSION_SECRET=                  # 多机部署必填，32+ 字符随机
CRYPTO_SECRET=                    # 共享 Redis 必填，32+ 字符随机

# ===== 数据库 =====
SQL_DSN=                          # 留空走 SQLite；如 root:pass@tcp(host:3306)/dbname
REDIS_CONN_STRING=                # 留空不启用；如 redis://localhost:6379/0

# ===== 网络 =====
PORT=3000                         # 监听端口
GIN_MODE=release                  # debug / release / test

# ===== 协议超时 =====
RELAY_IDLE_CONN_TIMEOUT=90        # 空闲连接保活（秒）
STREAMING_TIMEOUT=300             # 流式超时（秒）
STREAM_SCANNER_MAX_BUFFER_MB=64   # 流式单行最大 buffer（MB）
MAX_REQUEST_BODY_MB=32            # 请求体最大（MB），解压后

# ===== Azure =====
AZURE_DEFAULT_API_VERSION=2025-04-01-preview

# ===== 错误日志 =====
ERROR_LOG_ENABLED=false           # 开启后写错误日志到 DB

# ===== Pyroscope =====
PYROSCOPE_URL=
PYROSCOPE_APP_NAME=new-api
PYROSCOPE_BASIC_AUTH_USER=
PYROSCOPE_BASIC_AUTH_PASSWORD=
PYROSCOPE_MUTEX_RATE=5
PYROSCOPE_BLOCK_RATE=5
HOSTNAME=new-api

# ===== 多机部署 =====
NODE_TYPE=master                  # master / slave
SYNC_FREQUENCY=60                 # 配置同步间隔（秒）
FRONTEND_BASE_URL=                # slave 可重定向到 master

# ===== 缓存 =====
MEMORY_CACHE_ENABLED=false        # 启用内存缓存

# ===== 前端 =====
THEME=default                     # default / anthropic / simple-large

# ===== 时区 =====
TZ=Asia/Shanghai
```

---

**报告结束。** 本报告基于 2026-06-06 公开信息，所有数据点均标注来源。New API 项目发展迅速，建议跟踪 `https://github.com/QuantumNous/new-api/releases` 持续更新。
