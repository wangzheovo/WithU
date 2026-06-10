
# WithU — 个人 AI 伴侣机器人

## 一、项目背景

WithU 是一个**个人 AI 伴侣/助手**项目，目标是为用户打造一个具备人格化交互能力的桌面机器人。它既能作为日常陪伴的虚拟伙伴，也能作为高效的工作助手。

项目的核心出发点：
- **情感陪伴**：提供一个可交互的虚拟形象，缓解独处时的孤独感
- **智能对话**：接入大语言模型（LLM），支持自然语言交互，具备长期记忆能力
- **高度可定制**：通过 Web 后台自由配置机器人的人设、外观、行为、能力扩展等
- **渐进式演进**：从纯 Web 桌宠 → 带语音/视频的物理机器人 → 自动追随移动机器人

项目分为三期实现，当前一期聚焦于 **Web 桌面机器人 + 管理后台**。

---

## 二、整体实现思路

### 2.1 总体架构

```
┌─────────────────────────────────────────────────────────┐
│                     客户端层                              │
│  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │   桌面宠物端       │  │     Web 管理后台              │ │
│  │  (Canvas 2D动画)   │  │     (Vue 3 + Vite)           │ │
│  │  交互 / 展示 / 对话 │  │     配置 / 管理 / 监控        │ │
│  └────────┬─────────┘  └──────────────┬───────────────┘ │
│           │          WebSocket         │    REST API      │
└───────────┼───────────────────────────┼──────────────────┘
            │                           │
┌───────────▼───────────────────────────▼──────────────────┐
│                     服务端层 (FastAPI)                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ 对话引擎  │ │ 记忆系统  │ │ 任务管理  │ │  插件系统  │ │ RAG引擎  │ │ 情感引擎  │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│       │             │            │              │            │            │       │
│  ┌────▼─────────────▼────────────▼──────────────▼────────────▼────────────▼─────┐ │
│  │                             核心总线                                          │ │
│  │  LLM Provider 路由  /  上下文管理  /  消息路由  /  检索增强  /  成本追踪     │ │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│                          数据层                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │  PostgreSQL   │  │    Redis      │  │  文件存储     │  │  pgvector   │ │
│  │ + pgvector    │  │  缓存/会话     │  │  文档/日志    │  │  向量索引    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Claude Code 源码参考

本项目在交互设计、记忆系统、上下文管理等方面深度参考了 [Claude Code 的开源源码](D:\Wangz\ClaudeCode\claude-code)。Claude Code 是目前最优秀的 AI 编程助手 CLI，在以下方面有极高的工程参考价值。

#### 重点参考模块

| 模块 | Claude Code 路径 | 参考要点 | 本项目对应 |
|------|-----------------|---------|-----------|
| **记忆系统** | `src/memdir/` | 四种记忆类型（user/feedback/project/reference）、自动记忆提取、MEMORY.md 入口文件、记忆年龄衰减 | 机器人长期记忆存储与召回 |
| **上下文压缩** | `src/services/compact/` | 自动压缩(autoCompact)、反应式压缩(reactiveCompact)、上下文折叠(contextCollapse)、Token 预算管理 | 长对话上下文窗口管理 |
| **任务管理** | `src/Task.ts`, `src/commands/tasks/` | 任务状态机(pending→running→completed/failed/killed)、定时任务(CronCreateTool)、子任务派发 | 机器人定时提醒、自动化任务 |
| **插件系统** | `src/plugins/` | 内置插件+bundled、动态加载、插件生命周期 | 机器人能力扩展、第三方集成 |
| **语音交互** | `src/commands/voice/`, `hooks/useVoice.ts` | 语音输入链路、Hook 模式 | 后续物理机器人语音交互 |
| **查询引擎** | `src/QueryEngine.ts` | LLM API 调用编排、流式响应、重试逻辑、Token 计数 | DeepSeek API 调用封装 |
| **历史记录** | `src/history.ts` | 历史条目存储、粘贴内容引用、对话回放 | 对话记录存储与检索 |
| **成本追踪** | `src/cost-tracker.ts` | 每轮 token 用量与费用实时统计、`/cost` 命令 | Token 用量统计与 Dashboard 展示 |
| **启动优化** | `src/main.tsx` | 并行预取(MDM/keychain/GrowthBook)、懒加载、条件导入 | 服务启动速度优化 |
| **工具调用** | `src/Tool.ts`, `src/tools/` | 工具注册、权限检查、输入 Schema 定义 | Agent 工具调用体系（Phase 2） |
| **RAG 检索增强** | 独立设计 | Claude Code 无传统 RAG 系统，本项目参考企业 RAG 最佳实践自主设计（详见 6.6） | 知识库检索增强对话 |

#### 记忆系统详解（Claude Code `memdir/`）

Claude Code 的记忆系统是本项目最重要的参考对象，其核心设计包括：

- **四种记忆类型**：
  - `user` — 用户画像（角色、偏好、知识背景），始终私有
  - `feedback` — 用户反馈（纠正和建议），指导 AI 行为风格
  - `project` — 项目上下文（目标、进度、状态），偏向团队共享
  - `reference` — 参考信息（事实、约定、文档），可验证的知识

- **关键机制**：
  - `MEMORY.md` 作为入口文件，受限200行/25KB
  - 自动记忆提取（`extractMemories/`），从对话中自动识别应保存的记忆
  - 记忆年龄感知，区分新/旧记忆的时效性
  - 团队记忆同步（`teamMemorySync/`）

本项目将把这些设计适配为"用户画像 / 偏好习惯 / 会话摘要 / 知识库"四类记忆。

#### 上下文压缩机制详解（Claude Code `services/compact/`）

- **自动压缩**：当对话接近 token 上限的 90% 时触发，将历史消息压缩为摘要
- **反应式压缩**：基于收益递减检测——连续 3 次交互的 token 增量 < 500 且仍然接近预算上限时触发
- **上下文折叠**：更激进的上下文精简策略
- **Token 预算追踪**：`tokenBudget.ts` 记录每次轮次的 token 消耗，计算完成百分比，决定继续对话还是停止

本项目将在长对话场景中采用类似的"摘要+滑动窗口+动态预算"三层策略。

---

## 三、项目分期规划

### 第一期：Web 桌面机器人（当前阶段）

**目标**：实现一个可交互的 2D 桌宠 + 全功能 Web 管理后台

**核心功能**：
- 桌宠端（Canvas 2D 动画 + Electron 壳）：
  - 虚拟形象展示与动画（支持多套皮肤/动作）
  - 聊天对话窗口（文字 + 表情）+ RAG 知识检索入口
  - 语音输入/输出（可选，TTS/ASR）
  - 状态展示（待机 / 思考 / 说话 / 提醒）
  - 系统托盘驻留，可拖拽移动，窗口始终置顶
  - 断线自动重连 + 离线兜底动画

- Web 管理后台（Vue 3）：
  - Dashboard 总览（对话统计、活跃度、Token 费用面板）
  - LLM 配置管理（API Key、模型选择、Token、Temperature、Top-P、多模型路由策略）
  - 机器人人设编辑（System Prompt 可视化编辑 + 模板）
  - 外观配置（皮肤/主题/动画风格/颜色）
  - 语音配置（TTS 引擎选择、语速/音色）
  - 知识库管理（文档上传/解析/切片/向量化、知识库分类、检索测试）
  - 记忆管理（查看/编辑/删除记忆条目）
  - 对话历史（搜索/回溯/导出）
  - 定时任务（提醒事项、自动问候等）
  - 插件市场（安装/卸载/配置扩展）
  - 用户与权限管理（多用户、RBAC）
  - 系统设置（日志、备份、安全策略）
  - 配置热更新：后台修改配置后通过 WebSocket 实时推送至桌宠端，无需重启

- 安全防护：
  - JWT 认证 + Token 刷新
  - API 限流（Rate Limiting）
  - 输入校验与 XSS/SQL 注入防护
  - API Key 加密存储
  - CORS 白名单
  - 操作审计日志
  - HTTPS/TLS 强制

### 第二期：固定式物理机器人

**目标**：将桌宠迁移到物理设备，增加语音/视频交互

- 硬件方案：树莓派 + 触摸屏 + 麦克风阵列 + 摄像头 + 扬声器
- 语音交互：全双工语音对话（唤醒词 + ASR + LLM + TTS）
- 视觉能力：人脸检测/识别、表情识别、简单手势交互
- 软件适配：将 Web 前端适配为 Kiosk 模式全屏运行

### 第三期：自动追随移动机器人

**目标**：实现可自主移动的物理机器人

- 硬件方案：底盘电机控制 + SLAM/视觉导航 + 避障传感器
- 人体追踪：基于视觉的人体检测与跟随
- 自主巡航：室内定位、路径规划、自动回充
- 多模态交互：语音 + 视觉 + 触摸 + 表情反馈

---

## 四、技术选型

| 层次 | 技术 | 选型理由 |
|------|------|---------|
| **后端框架** | FastAPI (Python 3.11+) | 异步高性能、自动 OpenAPI 文档、类型安全、企业主流 |
| **ORM** | SQLAlchemy 2.0 (async) + Alembic | 成熟稳定、异步支持、迁移管理 |
| **数据库** | PostgreSQL 16 | 企业级关系型数据库，支持 JSON、全文搜索 |
| **缓存** | Redis 7 | 会话缓存、对话上下文缓存、API 限流、任务队列 |
| **任务队列** | Celery + Redis Broker | 异步任务（LLM调用重试、定时任务、语音合成） |
| **前端框架** | Vue 3 (Composition API) + Vite | 国内生态好、上手快、性能优秀 |
| **UI 组件库** | Element Plus / Naive UI | 企业级组件，Vue 3 原生支持 |
| **状态管理** | Pinia | Vue 3 官方推荐 |
| **桌宠渲染** | Canvas 2D / PixiJS | 轻量 2D 动画，支持精灵动画 |
| **桌宠打包** | Electron | 系统托盘驻留、窗口置顶/穿透、开机启动、跨平台 |
| **实时通信** | WebSocket (FastAPI WS + Socket.IO) | 桌宠 ↔ 服务端双向实时通信 |
| **LLM SDK** | 直接调用 DeepSeek API (httpx/aiohttp) | 已有 API 接入，轻量无多余依赖 |
| **向量数据库** | pgvector (PostgreSQL 扩展) | 与主库统一，零额外运维，支持 HNSW 索引、混合搜索 |
| **Embedding 模型** | DeepSeek Embedding / BGE-M3 (本地) | 首选 API，支持本地离线部署作为 fallback |
| **文档解析** | unstructured / PyMuPDF / python-docx | 多格式文档解析（PDF/Word/Markdown/TXT/HTML） |
| **文本分块** | LangChain TextSplitter / 自研 | 语义分块 + 滑动窗口重叠，可配置策略 |
| **Reranker** | BGE-Reranker / Cohere Rerank | 检索后重排序，提升召回精度 |
| **认证** | JWT (python-jose) + OAuth2 | 无状态认证，适合分布式部署 |
| **部署** | Docker + Nginx + 腾讯云/阿里云 | 容器化部署，一键启动 |
| **监控** | Prometheus + Grafana（可选） | 服务监控与告警 |

---

## 五、项目目录结构（规划）

```
WithU/
├── backend/                        # FastAPI 后端
│   ├── app/
│   │   ├── api/                    # API 路由层
│   │   │   ├── v1/
│   │   │   │   ├── auth.py         # 认证接口
│   │   │   │   ├── chat.py         # 对话接口
│   │   │   │   ├── config.py       # 配置管理接口
│   │   │   │   ├── knowledge.py     # 知识库管理接口
│   │   │   │   ├── memory.py       # 记忆管理接口
│   │   │   │   ├── plugin.py       # 插件管理接口
│   │   │   │   ├── task.py         # 任务管理接口
│   │   │   │   └── user.py         # 用户管理接口
│   │   │   └── deps.py             # 依赖注入
│   │   ├── core/
│   │   │   ├── config.py           # 全局配置（环境变量）
│   │   │   ├── security.py         # 安全模块（JWT/加密/限流）
│   │   │   └── database.py         # 数据库连接
│   │   ├── models/                 # SQLAlchemy 数据模型
│   │   │   ├── user.py
│   │   │   ├── memory.py
│   │   │   ├── knowledge.py        # 知识库文档/chunk 模型
│   │   │   ├── conversation.py
│   │   │   ├── plugin.py
│   │   │   └── task.py
│   │   ├── schemas/                # Pydantic 请求/响应 Schema
│   │   ├── services/               # 业务逻辑层
│   │   │   ├── llm/                # LLM 调用封装
│   │   │   │   ├── base.py         # Provider 抽象基类
│   │   │   │   ├── deepseek.py     # DeepSeek Provider
│   │   │   │   ├── router.py       # 多模型路由（轻量/重量模型智能调度）
│   │   │   │   └── context.py      # 上下文构建
│   │   │   ├── memory/             # 记忆系统
│   │   │   │   ├── manager.py      # 记忆 CRUD
│   │   │   │   ├── extractor.py    # 自动记忆提取
│   │   │   │   └── recall.py       # 记忆召回
│   │   │   ├── compact/            # 上下文压缩
│   │   │   │   └── compressor.py   # 对话摘要与窗口管理
│   │   │   ├── plugin/             # 插件系统
│   │   │   │   ├── loader.py       # 插件加载器
│   │   │   │   └── registry.py     # 插件注册中心
│   │   │   ├── task/               # 任务调度
│   │   │   │   ├── scheduler.py    # 定时任务（APScheduler/Celery Beat）
│   │   │   │   └── manager.py      # 任务生命周期管理
│   │   │   ├── rag/                # RAG 检索增强
│   │   │   │   ├── embedding.py     # 向量化服务
│   │   │   │   ├── chunker.py       # 文档切片策略
│   │   │   │   ├── retriever.py     # 混合检索器
│   │   │   │   ├── reranker.py      # 结果重排序
│   │   │   │   └── ingestion.py     # 文档摄取管道
│   │   │   ├── cost/                # 成本追踪
│   │   │   │   └── tracker.py       # Token 用量与费用统计
│   │   │   ├── tools/               # Agent 工具调用（预留扩展）
│   │   │   │   └── registry.py      # 工具注册中心
│   │   │   ├── emotion/             # 情感状态机
│   │   │   │   └── engine.py        # 多维情感模型（好感度/活跃度/好奇度）
│   │   │   └── voice/              # 语音服务（预留）
│   │   ├── ws/                     # WebSocket 处理
│   │   │   └── handler.py          # 桌宠 WS 消息路由
│   │   └── main.py                 # FastAPI 入口
│   ├── alembic/                    # 数据库迁移
│   ├── tests/                      # 单元测试
│   ├── requirements.txt
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── frontend/                       # Vue 3 管理后台
│   ├── src/
│   │   ├── views/                  # 页面
│   │   │   ├── Dashboard.vue
│   │   │   ├── ChatHistory.vue
│   │   │   ├── LLMConfig.vue
│   │   │   ├── PersonaEditor.vue
│   │   │   ├── AppearanceConfig.vue
│   │   │   ├── MemoryManager.vue
│   │   │   ├── PluginMarket.vue
│   │   │   ├── KnowledgeBase.vue
│   │   │   ├── TaskScheduler.vue
│   │   │   └── UserManagement.vue
│   │   ├── components/             # 通用组件
│   │   ├── stores/                 # Pinia 状态
│   │   ├── api/                    # API 调用
│   │   ├── router/                 # 路由
│   │   └── App.vue
│   ├── package.json
│   └── vite.config.ts
│
├── pet/                            # 桌面宠物端
│   ├── electron/                   # Electron 壳（系统托盘/窗口管理/开机启动）
│   │   ├── main.ts                 # Electron 主进程
│   │   ├── preload.ts              # 预加载脚本（安全隔离）
│   │   └── tray.ts                 # 系统托盘管理
│   ├── src/
│   │   ├── pet.ts                  # 宠物主逻辑
│   │   ├── renderer.ts             # Canvas/PixiJS 渲染
│   │   ├── animations/             # 动画资源与状态机
│   │   ├── ws-client.ts            # WebSocket 客户端（含断线重连）
│   │   └── config.ts               # 宠物端本地配置
│   ├── assets/                     # 图片/精灵图资源
│   └── package.json
│
├── docs/                           # 详细设计文档
│   ├── claude-code-analysis.md     # Claude Code 源码分析笔记
│   ├── memory-system-design.md     # 记忆系统详细设计
│   ├── rag-system-design.md        # RAG 系统详细设计
│   └── plugin-system-design.md     # 插件系统详细设计
│
├── deploy/                         # 部署配置
│   ├── nginx.conf
│   ├── docker-compose.prod.yml
│   └── deploy.sh
│
└── README.md
```

---

## 六、核心模块设计概要

### 6.1 记忆系统

参考 Claude Code `memdir/` 设计，适配为个人伴侣场景：

```
记忆类型映射：
  Claude Code          →    WithU
  ─────────────             ─────
  user (用户画像)      →    用户画像：年龄、职业、兴趣、交流风格偏好
  feedback (行为反馈)  →    偏好习惯：喜欢/不喜欢的回复风格、话题偏好
  project (项目上下文) →    会话摘要：上次聊到哪里、未完成的话题
  reference (参考信息) →    知识库：用户教给机器人的事实、约定
```

**核心机制**：
- **自动提取**：每次对话后自动分析，提取高价值记忆存入 PostgreSQL
- **时效衰减**：记忆按时间加权，过期记忆召回权重降低
- **冲突检测**：新增记忆与已有记忆做语义相似度比对，去重或更新
- **分层存储**：热记忆 → Redis（高频访问），全量 → PostgreSQL

### 6.2 上下文压缩

参考 Claude Code `services/compact/`，采用三层策略：

1. **滑动窗口**：保留最近 N 轮完整对话历史
2. **摘要压缩**：超出窗口的历史消息压缩为摘要（调用 LLM 生成）
3. **动态 Token 预算**：
   - 追踪每轮 token 消耗
   - 当用量超过阈值（90%）时触发压缩
   - 收益递减检测：连续 3 轮 token 增量 < 阈值时不继续扩展上下文
4. **Redis 缓存**：压缩后的摘要缓存在 Redis，避免重复计算

### 6.3 插件系统

参考 Claude Code `plugins/` 设计，支持能力热扩展：

- **插件规范**：每个插件为一个 Python 包，包含 `manifest.json` + `plugin.py`
- **生命周期**：`load() → enable() → disable() → unload()`
- **注册钩子**：对话前置/后置处理、定时触发、消息拦截、UI 扩展点
- **进程隔离**：插件通过  subprocess + pipe 与主进程通信，崩溃不影响主服务
- **市场分发**：内置插件 + 第三方安装（Git URL / 本地路径）

### 6.4 任务管理

参考 Claude Code `Task.ts` 设计：

- **任务状态机**：`pending → running → completed / failed / killed`
- **任务类型**：一次性提醒、循环提醒（cron 表达式）、LLM 自主任务
- **持久化**：任务状态写入 PostgreSQL，Redis 缓存待执行队列
- **可靠执行**：Celery Beat 驱动定时触发，失败重试 + 死信队列

### 6.5 安全设计

| 层面 | 措施 |
|------|------|
| 传输层 | HTTPS/TLS 1.3 强制，WebSocket over TLS |
| 认证层 | JWT Access + Refresh Token，OAuth2 可选 |
| 应用层 | 输入白名单校验、ORM 防注入、CORS 白名单 |
| 数据层 | API Key AES-256 加密存储（密钥从环境变量注入）、敏感日志脱敏 |
| 限流 | Redis 令牌桶：API 接口级 + 用户级双层限流 |
| 审计 | 所有配置变更、记忆修改记录审计日志 |
| 文档安全 | 知识库文档访问权限控制、敏感信息扫描（API Key/手机号/身份证正则脱敏） |

---

### 6.6 RAG 检索增强系统

#### 设计理念

RAG 是独立于 Claude Code 参考体系的新设计——Claude Code 面向代码仓库，WithU 面向知识学习场景，两者的知识获取路径完全不同。

当一个用户对 WithU 说"我想了解一下量子计算的基本原理"，RAG 引擎会：
1. 将用户问题向量化
2. 从预设知识库中检索最相关的文档片段
3. 把检索结果与用户问题一起送入 LLM，生成准确、有据可查的回答

#### 整体架构

```
用户提问 ──▶ 对话引擎 ──▶ RAG 路由决策
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         直接 LLM        RAG 增强        知识库检索
         (闲聊模式)      (学习模式)       (仅搜索)
              │               │
              │     ┌─────────▼─────────┐
              │     │   查询重写 (可选)   │  ← 优化检索意图
              │     └─────────┬─────────┘
              │               ▼
              │     ┌───────────────────┐
              │     │   向量化 (Embedding)│  ← DeepSeek / BGE-M3
              │     └─────────┬─────────┘
              │               ▼
              │     ┌───────────────────┐
              │     │  混合检索          │
              │     │  语义 (pgvector)   │
              │     │  + 关键词 (BM25)   │
              │     └─────────┬─────────┘
              │               ▼
              │     ┌───────────────────┐
              │     │  Reranker 重排序   │  ← BGE-Reranker
              │     └─────────┬─────────┘
              │               ▼
              │     ┌───────────────────┐
              │     │  上下文组装        │
              │     │  检索结果 + 提示词  │
              │     └─────────┬─────────┘
              │               │
              ▼               ▼
          ┌───────────────────────┐
          │   LLM (DeepSeek) 生成  │
          │   + 引用来源标注        │
          └───────────────────────┘
```

#### 文档摄取管道（Ingestion Pipeline）

```
用户上传文档 ──▶ 格式解析 ──▶ 文本提取 ──▶ 智能分块 ──▶ 向量化 ──▶ 存入 pgvector
   (Web后台)     (每种格式       (unstructured/   (语义/固定/   (Embedding    (HNSW索引
                 专用解析器)      PyMuPDF等)      滑动窗口)     API)           
```

**1. 支持的文档格式**

| 格式 | 解析器 | 说明 |
|------|--------|------|
| PDF | PyMuPDF / pdfplumber | 支持文字型与扫描型（OCR）PDF |
| Word (.docx) | python-docx | 保留段落/表格结构 |
| Markdown (.md) | markdown-it-py | 保留标题层级作为元数据 |
| 纯文本 (.txt) | 原生读取 | 按段落自然分块 |
| HTML | BeautifulSoup | 提取正文，剥离标签 |
| 代码文件 | 原生读取 | 保留语法结构信息 |
| CSV / Excel | pandas | 表格数据向量化 |

**2. 智能分块策略**

```python
# 分块策略可配置，不同文档类型使用不同策略
CHUNKING_STRATEGIES = {
    "general": {  # 通用文档
        "chunk_size": 512,        # tokens
        "chunk_overlap": 64,      # 重叠量
        "separators": ["\n\n", "\n", "。", ".", " "],
    },
    "code": {  # 代码文件
        "chunk_size": 1024,
        "chunk_overlap": 128,
        "separators": ["\n## ", "\nclass ", "\ndef ", "\n"],
    },
    "qa": {  # 问答对
        "chunk_size": 256,
        "chunk_overlap": 0,
        "separators": ["\nQ:", "\n问：", "\n\n"],
    },
}
```

**3. 分块元数据（增强检索精度）**

每个 chunk 携带的结构化元数据：

| 字段 | 类型 | 用途 |
|------|------|------|
| `document_id` | UUID | 关联源文档 |
| `chunk_index` | int | 片段在文档中的顺序 |
| `page_number` | int | 原始页码（PDF） |
| `heading_path` | list[str] | 标题层级路径（Markdown） |
| `content_hash` | str | 内容哈希，去重与增量更新 |
| `token_count` | int | token 数量 |
| `created_at` | datetime | 创建时间 |

#### 混合检索策略

| 层级 | 方法 | 技术 | 用途 |
|------|------|------|------|
| **稀疏检索** | BM25 / TF-IDF | PostgreSQL `tsvector` + GIN 索引 | 关键词精确匹配 |
| **稠密检索** | 向量相似度 | pgvector `<=>` cosine distance + HNSW 索引 | 语义相似匹配 |
| **混合融合** | RRF (Reciprocal Rank Fusion) | 应用层实现 | 合并两种检索结果，取交集排序 |
| **重排序** | Cross-Encoder | BGE-Reranker-v2-m3 | 对 Top-K 结果精排 |
| **查询重写** | LLM 改写 | 调用 DeepSeek | 将口语化问题改写为检索友好格式 |

#### 检索精度保障

| 机制 | 说明 |
|------|------|
| **元数据过滤** | 按知识库分类、文档类型、时间范围等过滤 |
| **相似度阈值** | 低于阈值（默认 0.7）的检索结果丢弃 |
| **来源标注** | 每个检索结果标注源文档名、页码、相关度分数 |
| **检索质量评估** | RAGAS 指标（Faithfulness/Context Recall/Context Precision） |
| **A/B 测试框架** | 不同分块/检索策略的效果对比 |

#### 性能优化

| 优化点 | 方案 |
|--------|------|
| **向量索引** | pgvector HNSW 索引（`ef_construction=128, m=16`），百万级向量毫秒检索 |
| **Embedding 缓存** | Redis 缓存已向量化的 chunk，相同内容不重复调用 API |
| **批量摄入** | Celery 异步任务批量处理文档，不阻塞主服务 |
| **增量更新** | 基于 `content_hash` 比对，只更新变化的 chunk |
| **查询缓存** | Redis 缓存高频查询的检索结果（TTL 5 分钟） |
| **连接池** | pgvector 连接复用，避免频繁建连 |

#### RAG 与对话引擎的集成

```
用户消息到达
    │
    ▼
┌──────────────┐
│ 意图识别       │  ← 判断是否需要 RAG
│ (LLM 轻量分类) │
└──────┬───────┘
       │
   ┌───┴───┐
   │       │
 闲聊   知识类
   │       │
   ▼       ▼
常规LLM   触发RAG
          │
          ▼
     ┌──────────────┐
     │  查询重写      │
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │  混合检索      │
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │  结果注入      │
     │  System Prompt │
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │  LLM 生成      │
     │  + 来源引用     │
     └──────────────┘
```

System Prompt 注入格式：

```
## 参考知识（来自知识库）

以下内容来自用户的知识库，请基于这些内容回答用户问题。
如果参考内容不足以回答问题，请明确说明并基于你的知识补充。

[来源: 《量子计算入门》第3章, 相关度: 0.94]
量子计算是利用量子力学原理进行信息处理的计算方式...

[来源: 《现代物理导论》第7章, 相关度: 0.87]
量子比特（qubit）是量子计算的基本单位，与经典比特不同...

---
## 用户问题
{用户原始问题}
```

#### pgvector 数据模型

```sql
-- 知识库文档表
CREATE TABLE knowledge_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(500) NOT NULL,
    category VARCHAR(200),           -- 分类标签
    file_type VARCHAR(50),           -- pdf/docx/md/txt等
    file_size BIGINT,
    status VARCHAR(20) DEFAULT 'processing',  -- processing/ready/error
    chunk_count INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 向量 chunk 表
CREATE TABLE knowledge_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES knowledge_documents(id) ON DELETE CASCADE,
    chunk_index INT NOT NULL,
    content TEXT NOT NULL,
    content_hash VARCHAR(64),
    embedding VECTOR(1024),          -- pgvector 向量字段
    metadata JSONB DEFAULT '{}',     -- page_number/heading_path等
    token_count INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW 向量索引（加速检索）
CREATE INDEX idx_chunks_embedding 
ON knowledge_chunks 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 128);

-- 全文搜索索引（BM25 关键词检索）
CREATE INDEX idx_chunks_content_fts 
ON knowledge_chunks 
USING gin (to_tsvector('simple', content));

-- 元数据索引
CREATE INDEX idx_chunks_document_id ON knowledge_chunks(document_id);
CREATE INDEX idx_chunks_hash ON knowledge_chunks(content_hash);
```



### 6.7 LLM Provider 与多模型路由

为避免只依赖单一 LLM 供应商，对话引擎使用 Provider 抽象模式：

```python
# Provider 抽象基类
class BaseLLMProvider(ABC):
    @abstractmethod
    async def chat(self, messages: list, **kwargs) -> str: ...
    @abstractmethod
    async def chat_stream(self, messages: list, **kwargs) -> AsyncIterator[str]: ...
    @abstractmethod
    def count_tokens(self, text: str) -> int: ...

# 具体 Provider
class DeepSeekProvider(BaseLLMProvider): ...
class OpenAIProvider(BaseLLMProvider): ...
class QianwenProvider(BaseLLMProvider): ...
```

**多模型路由**：不是所有对话都需要最强的模型，通过路由降本增效：

```
用户消息 → 复杂度评估（快速小模型判断）→ 路由决策
                                          │
                  ┌───────────────────────┼───────────────────────┐
                  ▼                       ▼                       ▼
          闲聊/问候 → 便宜模型    知识问答 → 强模型+RAG     复杂推理 → 最强模型
          (deepseek-chat)      (deepseek-reasoner)    (deepseek-reasoner)
```

| 路由规则 | 触发条件 | 目标模型 |
|---------|---------|---------|
| 轻量级 | 问候/闲聊/短指令（< 50 token） | 便宜模型 |
| 标准级 | 一般问题回复 | 默认模型 |
| 重量级 | 知识问答/长篇生成/复杂推理 | 强模型 |

---

### 6.8 成本追踪

参考 Claude Code `cost-tracker.ts`，每轮 LLM 调用实时统计用量和费用：

- **Token 计数**：输入/输出 token 分开统计，支持不同模型不同单价
- **费用计算**：按 DeepSeek 官方定价实时计算（`input_price * tokens + output_price * tokens`）
- **存储**：每次调用的用量写入 PostgreSQL，Redis 缓存当日汇总
- **Dashboard 展示**：今日/本周/本月费用曲线，按模型/会话拆分
- **预算告警**：可设置日/月预算上限，接近时 WebSocket 推送警告

---

### 6.9 Agent 工具调用

除了聊天和知识检索，机器人还能通过工具调用执行具体动作（参考 Claude Code `tools/` 体系）：

```
用户："半小时后提醒我开会"
  → LLM 识别意图 → 调用 set_reminder(delta=30min, content="开会")
  → 机器人回复："好的，30分钟后我会提醒你开会"
```

**内置工具集（一期预留，Phase 2 实现）**：

| 工具 | 功能 | 示例 |
|------|------|------|
| `set_reminder` | 设置定时提醒 | "半小时后提醒我开会" |
| `get_weather` | 查询天气 | "明天需要带伞吗？" |
| `search_news` | 搜索新闻 | "今天有什么科技新闻？" |
| `play_music` | 播放音乐 | "放一首轻音乐" |
| `translate` | 翻译 | "把这段话翻译成英文" |
| `calculate` | 计算器 | "计算 12345 × 6789" |
| `search_files` | 本地文件搜索 | "帮我找上周那份PDF" |

---

### 6.10 情感状态机

作为"伴侣"而非纯工具，机器人维护一个简单的多维情感模型：

```
情感维度：
  好感度 (affection)    0 ────────── 100    (受对话质量影响)
  活跃度 (energy)       0 ────────── 100    (随时间自然衰减，互动恢复)
  好奇度 (curiosity)    0 ────────── 100    (新话题激发)

状态：
  开心                    平静                     疲惫
  affection > 70        30-70                  < 30
  回复活泼、表情丰富      正常对话                回复简短、建议休息
```

情感值影响：
- **回复风格**：好感度高时语气更亲密，低时保持礼貌距离
- **动画表现**：活跃度影响桌宠动作频率和幅度
- **主动行为**：好奇度高时更倾向发起新话题

---

### 6.11 多角色与对话场景

支持同一用户创建多个机器人角色，每个角色独立配置：

| 角色 | System Prompt 示例 | 特点 |
|------|-------------------|------|
| 工作助理 | "你是专业的工作助理，帮助用户规划日程..." | 禁止闲聊、回复简洁 |
| 生活伴侣 | "你是用户的知心朋友，可以聊任何话题..." | 情感丰富、主动关怀 |
| 学习导师 | "你是耐心博学的导师，善于解释复杂概念..." | 引用知识库、引导式提问 |

**对话场景模板**：一键切换模式，自动调整 system prompt 和知识库范围

| 场景 | 效果 |
|------|------|
| 学习模式 | 启用 RAG，引用知识库来源 |
| 闲聊模式 | 禁用 RAG，自由对话 |
| 专注模式 | 减少主动消息，降低打扰 |
| 夜聊模式 | 语气温柔，适合作息提醒 |

---

### 6.12 上下文压缩的成本考量

6.2 节提到的摘要压缩调用 LLM 本身也消耗 token。这一成本在以下条件下是可以接受的：

1. 摘要请求通常输入长但输出短（几百 token 摘要 vs 数千 token 原始对话），净节省显著
2. 使用便宜模型做摘要，进一步降低成本
3. 摘要结果缓存在 Redis，同一段对话不重复压缩
4. 触发条件严格——只有接近 token 上限（90%）时才执行

---

## 七、测试策略

| 层级 | 工具 | 范围 | 目标 |
|------|------|------|------|
| 单元测试 | pytest + pytest-asyncio | 所有 services 纯逻辑函数 | 覆盖率 > 80% |
| 集成测试 | pytest + httpx (TestClient) | API 端点 + 数据库交互 | 核心流程全覆盖 |
| E2E 测试 | Playwright | 管理后台关键用户路径 | 登录→配置→对话全链路 |
| 向量检索测试 | pytest | RAG 检索精度（RAGAS 指标） | Faithfulness > 0.8 |
| 性能测试 | Locust | WebSocket 并发 / API 吞吐 | 100 并发 WS + 500 QPS REST |

测试目录结构：
```
backend/tests/
├── unit/
│   ├── test_memory.py
│   ├── test_rag_retriever.py
│   ├── test_compact.py
│   └── test_cost_tracker.py
├── integration/
│   ├── test_api_auth.py
│   ├── test_api_chat.py
│   └── test_ws_handler.py
└── e2e/（frontend/tests/ 下）
```

---

## 八、待优化与扩展（非一期）

以下事项已识别但不在当前交付范围内，作为后续迭代的候选项：

| 优先级 | 事项 | 说明 |
|--------|------|------|
| P1 | WebSocket 流式中断支持 | 用户发送新消息时取消当前 LLM 流式输出（AbortController 模式） |
| P1 | 记忆提取频率控制 | 短对话（< 3 轮）跳过提取；合并为对话结束后批量提取 |
| P2 | 本地 Embedding 离线 fallback | 当 DeepSeek Embedding API 不可用时自动切换本地 BGE-M3 |
| P2 | 多语言国际化 | 管理后台中英双语（vue-i18n）；机器人回复跟随用户设置 |
| P3 | 图表可视化 | Dashboard 引入 ECharts，展示对话趋势/费用曲线/情感波动 |
| P3 | 移动端适配 | 管理后台响应式适配，桌宠端考虑 React Native 移植 |
| P3 | 联邦知识库 | 允许多用户共享知识库，权限隔离 |


## 九、关键参考文件索引

以下是 Claude Code 源码中需要深入分析的关键文件，按模块分类：

### 记忆系统
- [memdir/memoryTypes.ts](D:\Wangz\ClaudeCode\claude-code\src\memdir\memoryTypes.ts) — 记忆类型定义与使用指南
- [memdir/memdir.ts](D:\Wangz\ClaudeCode\claude-code\src\memdir\memdir.ts) — 记忆入口构建与截断逻辑
- [memdir/memoryScan.ts](D:\Wangz\ClaudeCode\claude-code\src\memdir\memoryScan.ts) — 记忆扫描
- [memdir/memoryAge.ts](D:\Wangz\ClaudeCode\claude-code\src\memdir\memoryAge.ts) — 记忆时效管理
- [services/extractMemories/](D:\Wangz\ClaudeCode\claude-code\src\services\extractMemories) — 自动记忆提取

### 上下文压缩
- [services/compact/autoCompact.ts](D:\Wangz\ClaudeCode\claude-code\src\services\compact\autoCompact.ts) — 自动压缩触发
- [services/compact/compact.ts](D:\Wangz\ClaudeCode\claude-code\src\services\compact\compact.ts) — 压缩后消息构建
- [services/compact/reactiveCompact.ts](D:\Wangz\ClaudeCode\claude-code\src\services\compact\reactiveCompact.ts) — 反应式压缩
- [query/tokenBudget.ts](D:\Wangz\ClaudeCode\claude-code\src\query\tokenBudget.ts) — Token 预算追踪
- [services/contextCollapse/](D:\Wangz\ClaudeCode\claude-code\src\services\contextCollapse) — 上下文折叠

### 任务管理
- [Task.ts](D:\Wangz\ClaudeCode\claude-code\src\Task.ts) — 任务状态机与类型定义
- [commands/tasks/](D:\Wangz\ClaudeCode\claude-code\src\commands\tasks) — 任务命令实现

### 插件系统
- [plugins/](D:\Wangz\ClaudeCode\claude-code\src\plugins) — 插件加载与管理
- [plugins/bundled/](D:\Wangz\ClaudeCode\claude-code\src\plugins\bundled) — 内置插件

### 语音交互
- [commands/voice/](D:\Wangz\ClaudeCode\claude-code\src\commands\voice) — 语音命令
- [hooks/useVoice.ts](D:\Wangz\ClaudeCode\claude-code\src\hooks\useVoice.ts) — 语音 Hook
- [hooks/useVoiceEnabled.ts](D:\Wangz\ClaudeCode\claude-code\src\hooks\useVoiceEnabled.ts) — 语音开关
- [context/voice.tsx](D:\Wangz\ClaudeCode\claude-code\src\context\voice.tsx) — 语音上下文

### 整体架构与优化
- [main.tsx](D:\Wangz\ClaudeCode\claude-code\src\main.tsx) — 入口与启动优化（并行预取、懒加载）
- [QueryEngine.ts](D:\Wangz\ClaudeCode\claude-code\src\QueryEngine.ts) — LLM 查询引擎
- [query.ts](D:\Wangz\ClaudeCode\claude-code\src\query.ts) — 查询编排
- [history.ts](D:\Wangz\ClaudeCode\claude-code\src\history.ts) — 历史记录管理
- [context.ts](D:\Wangz\ClaudeCode\claude-code\src\context.ts) — 系统/用户上下文收集

---

## 十、部署方案

### 云服务器配置建议
| 阶段 | 配置 | 预估成本 |
|------|------|---------|
| 开发/测试 | 2C4G + 50G SSD | ~50元/月 |
| 一期上线 | 4C8G + 100G SSD | ~150-300元/月 |
| 二期（+语音） | 8C16G + 200G SSD | ~400-600元/月 |

### 部署架构
```
用户浏览器 ──HTTPS──▶ Nginx (反向代理)
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          FastAPI    Vue静态文件   WebSocket
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
PostgreSQL   Redis    Celery
+pgvector             Worker
```

### 一键部署
```bash
# 开发环境
docker-compose up -d

# 生产环境（腾讯云/阿里云）
docker-compose -f docker-compose.prod.yml up -d

# 首次部署需先启用 pgvector 扩展
# docker exec -it withu-postgres psql -U withu -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

---

## 十一、开发路线图

### Phase 1（当前）
- [ ] 项目脚手架搭建（FastAPI + Vue 3 + PostgreSQL + Redis）
- [ ] 用户认证系统（JWT + RBAC）
- [ ] LLM 接入层（DeepSeek API 封装 + 流式响应）
- [ ] 基础对话功能 + WebSocket 实时通信
- [ ] Web 管理后台基础框架
- [ ] LLM 配置管理页面
- [ ] 机器人人设编辑器
- [ ] 桌宠 2D 渲染引擎（Canvas/PixiJS）
- [ ] 记忆系统（参考 Claude Code memdir 设计）
- [ ] 上下文压缩（对话摘要 + 滑动窗口）
- [ ] RAG 基础管道（pgvector + 文档摄取 + 向量化 + 基础检索）
- [ ] LLM Provider 抽象层 + 多模型路由
- [ ] 成本追踪（Token 用量 + 费用统计 + Dashboard 面板）
- [ ] 安全防护加固

### Phase 2
- [ ] RAG 高级特性（混合检索 + Reranker + 查询重写 + 检索质量评估）
- [ ] 知识库管理后台（文档上传/分类/检索测试页面）
- [ ] Agent 工具调用（提醒/天气/翻译等）
- [ ] 情感状态机（好感度/活跃度/好奇度）
- [ ] 多角色与对话场景模板
- [ ] 插件系统
- [ ] 任务调度（定时提醒）
- [ ] 外观配置（皮肤/主题/动画）
- [ ] 语音配置（TTS 集成）
- [ ] 对话历史搜索与导出
- [ ] 管理后台 Dashboard 完善
- [ ] 单元测试覆盖 > 80%
- [ ] 部署到腾讯云/阿里云

### Phase 3（第二期）
- [ ] 语音交互（ASR + TTS 全双工）
- [ ] 视频交互（人脸检测/表情识别）
- [ ] 物理机器人硬件适配（树莓派）

### Phase 4（第三期）
- [ ] 移动底盘控制
- [ ] 人体追踪与自动跟随
- [ ] 自主导航与避障
