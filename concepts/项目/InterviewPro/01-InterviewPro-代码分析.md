# InterviewPro — Eino Agent 架构实现分析

> 分析时间：2026-06-06  
> 代码路径：`/home/tommychen/english-learner/backend`  
> 分析目的：了解当前基于 Eino 的 Agent 架构实现

---

## 一、项目概述

**InterviewPro** 是一款 AI 面试练习应用，支持语音对话、实时转写、发音评测和智能评分。后端已全面采用 **CloudWeGo Eino** 框架编排 AI 流程。

| 维度 | 描述 |
|------|------|
| 项目名称 | InterviewPro |
| 后端语言 | Go 1.25 |
| AI 编排框架 | CloudWeGo Eino v0.8.13 |
| 核心功能 | 语音对话、实时转写、发音评测、AI 5 维评分、面试报告、题库管理、计费系统 |
| 代码路径 | `/home/tommychen/english-learner/backend` |

---

## 二、项目目录结构

```
backend/
├── cmd/
│   ├── server/main.go           # 应用入口（Gin + Eino + 所有服务初始化）
│   ├── seed/main.go             # 种子数据导入（题库/scenario）
│   └── stt-check/main.go        # STT 连通性检测工具
│
├── internal/
│   ├── agent/                   # ★ Eino Agent 核心
│   │   ├── graph.go             # InterviewGraph: Eino compose.Parallel 编排
│   │   ├── graph_runner.go      # GraphRunner: 图执行 + 题库兜底 + fallback
│   │   ├── state.go             # GraphInput/GraphState/MergedOutput 定义
│   │   ├── interviewer.go       # InterviewerAgent: 面试官 Agent（出题）
│   │   ├── scorer.go            # ScorerAgent: 评分 Agent（5 维评估）
│   │   ├── memory.go            # LongTermMemory: 跨会话长期记忆（Qdrant）
│   │   ├── reranker.go          # Reranker 接口 + NoopReranker
│   │   ├── reranker_siliconflow.go # SiliconFlow 重排序实现
│   │   └── adapters.go          # 适配器（避免循环依赖）
│   │
│   ├── app/
│   │   └── app.go               # 应用引导（DB、路由、服务依赖注入）
│   │
│   ├── config/
│   │   ├── config.go            # Viper 配置结构体（全量配置）
│   │   └── credentials.go       # 凭据安全管理
│   │
│   ├── crypto/
│   │   └── aes.go               # AES 加解密工具
│   │
│   ├── dto/
│   │   ├── request/             # 请求 DTO（auth/interview/admin）
│   │   └── response/            # 响应 DTO（auth/interview/admin）
│   │
│   ├── handler/                 # ★ HTTP 处理器层
│   │   ├── auth.go              # 认证（JWT + 短信验证码）
│   │   ├── auth_github.go       # GitHub OAuth 登录
│   │   ├── interview.go         # 面试 API（创建会话/获取题目/提交回答）
│   │   ├── ws.go                # WebSocket 升级端点
│   │   ├── health.go            # 健康检查 /healthz
│   │   ├── speech_health_handler.go # 语音服务健康状态
│   │   ├── voice_test_handler.go    # 语音调试端点
│   │   ├── user.go              # 用户信息
│   │   ├── admin.go             # 管理端接口
│   │   ├── ai_model.go          # AI 模型管理 API
│   │   ├── billing.go           # 计费/积分 API
│   │   ├── position.go          # 岗位管理
│   │   ├── question.go          # 题目 API
│   │   ├── question_admin.go    # 题目管理端 API
│   │   ├── question_extended.go # 扩展题目操作
│   │   ├── prepare.go           # 面试准备 API
│   │   ├── resume.go            # 简历上传/解析 API
│   │   ├── prompt_admin.go      # 提示词管理 API
│   │   ├── statistics.go        # 统计 API
│   │   ├── trace_admin.go       # 追踪管理
│   │   ├── rag_trace_admin.go   # RAG 检索追踪
│   │   ├── vector_admin.go      # 向量管理 API
│   │   └── api_provider.go      # API Provider 管理
│   │
│   ├── middleware/
│   │   ├── auth.go              # JWT 认证中间件
│   │   ├── admin_auth.go        # 管理端认证中间件
│   │   ├── logger.go            # 请求日志中间件
│   │   └── middleware.go        # 中间件注册
│   │
│   ├── model/                   # GORM 数据模型
│   │   ├── model.go             # 核心表模型
│   │   ├── admin.go             # 管理员模型
│   │   ├── billing.go           # 计费模型
│   │   ├── api_provider.go      # API Provider 模型
│   │   ├── trace.go             # 调用追踪模型
│   │   └── rag_trace.go         # RAG 检索追踪模型
│   │
│   ├── service/                 # ★ 业务逻辑层
│   │   ├── ai/                  # ★ AI 服务（多 Provider）
│   │   │   ├── types.go         # FiveDimensionResult 等核心类型
│   │   │   ├── factory.go       # 模型工厂（DeepSeek/Qwen/Eino）
│   │   │   ├── provider.go      # ModelProvider 接口
│   │   │   ├── llm.go           # AIService 主接口
│   │   │   ├── deepseek_model.go    # DeepSeek HTTP 客户端
│   │   │   ├── deepseek_service.go  # DeepSeek AI 服务
│   │   │   ├── qwen_model.go        # Qwen Local HTTP 客户端
│   │   │   ├── qwen_service.go      # Qwen Local AI 服务
│   │   │   ├── eino_model.go        # Eino ChatModel 封装
│   │   │   ├── prompts.go           # 系统提示词
│   │   │   ├── question_prompts.go  # 出题提示词
│   │   │   ├── optimized.go         # 优化版评估逻辑
│   │   │   ├── parallel.go          # 并行调用
│   │   │   └── gpu_batch.go         # GPU 批处理
│   │   │
│   │   ├── speech/              # ★ 语音服务（多 Provider 工厂）
│   │   │   ├── types.go         # STTProvider/TTSProvider 接口定义
│   │   │   ├── factory.go       # Fallback 链式工厂
│   │   │   ├── health.go        # SpeechHealth 健康监控
│   │   │   ├── aliyun.go        # 阿里云 STT/TTS
│   │   │   ├── bailian.go       # 百炼 paraformer-realtime-v2 STT
│   │   │   ├── bailian_stream.go    # 百炼流式 STT
│   │   │   ├── qwen_asr.go          # Qwen3-ASR-Flash
│   │   │   ├── whisper.go           # OpenAI Whisper STT
│   │   │   ├── sensevoice_stt.go    # SenseVoiceSmall STT
│   │   │   ├── volcengine_asr.go    # 火山引擎 ASR
│   │   │   ├── xunfei_iat.go        # 讯飞 IAT
│   │   │   ├── azure_stt.go         # Azure Speech STT
│   │   │   ├── elevenlabs.go        # ElevenLabs TTS
│   │   │   ├── volcano_tts.go       # 火山引擎 TTS
│   │   │   ├── siliconflow_tts.go   # 硅基流动 CosyVoice2 TTS
│   │   │   ├── azure_tts.go         # Azure TTS
│   │   │   ├── edge_tts.go          # Edge TTS（免费）
│   │   │   ├── xunfei_tts.go        # 讯飞 TTS
│   │   │   └── analyzer.go          # 音频分析工具
│   │   │
│   │   ├── pronunciation/        # ★ 发音评测（多后端）
│   │   │   ├── manager.go       # PronunciationManager 统一入口
│   │   │   ├── model.go         # 评测结果模型
│   │   │   ├── aliyun_asr.go    # 阿里云 ASR 词匹配
│   │   │   └── xunfei_ise.go    # 科大讯飞 ISE
│   │   │
│   │   ├── auth.go              # 短信验证码认证
│   │   ├── auth_github.go       # GitHub OAuth
│   │   ├── billing.go           # 计费系统
│   │   ├── billing_catalog.go   # 计费目录
│   │   ├── billing_provider.go  # 计费 Provider
│   │   ├── contribution.go      # 用户贡献
│   │   ├── embedding.go         # Embedding 服务（SiliconFlow/Ollama）
│   │   ├── hybrid_search.go     # 混合检索（RRF）
│   │   ├── vector_factory.go    # 向量数据库工厂
│   │   ├── vector_provider.go   # 向量 Provider 接口
│   │   ├── vector_service.go    # 向量检索服务
│   │   ├── pgvector_provider.go # pgvector 实现
│   │   ├── interview.go         # 面试服务
│   │   ├── question.go          # 题目服务
│   │   ├── question_pool.go     # 题目池（按场景/岗位出题）
│   │   ├── question_admin.go    # 题目管理服务
│   │   ├── question_extended.go # 扩展题目服务
│   │   ├── question_migration.go    # 题目迁移
│   │   ├── question_fill_answers.go # 题目答案填充
│   │   ├── review_service.go    # 复习/回顾服务
│   │   ├── scoring_feedback.go  # 评分反馈
│   │   ├── resume.go            # 简历解析
│   │   ├── resume_question.go   # 简历出题
│   │   ├── position.go          # 岗位服务
│   │   ├── position_seed.go     # 岗位种子数据
│   │   ├── scenario_config.go   # 场景配置
│   │   ├── scenario_cleanup.go  # 场景清理
│   │   ├── skill_loader.go      # 技能配置加载
│   │   ├── prompt_manager.go    # 提示词管理
│   │   ├── trace_service.go     # 追踪服务
│   │   ├── api_provider.go      # API Provider 服务
│   │   ├── api_provider_seed.go # API Provider 种子
│   │   ├── starter_index.go     # 首页索引
│   │   ├── statistics.go        # 统计服务
│   │   ├── function_provider.go # 功能 Provider
│   │   ├── email.go             # 邮件服务
│   │   └── admin.go             # 管理端服务
│   │
│   ├── ws/                      # ★ WebSocket 层
│   │   ├── client.go            # WebSocket 连接管理（消息收发/序列化/DB持久化）
│   │   ├── session_flow.go      # ★ 面试业务逻辑（会话启停/消息处理/评估/出题）
│   │   ├── interview_session.go # 面试会话状态
│   │   ├── hub.go               # 连接注册/注销/广播/服务注入
│   │   └── streaming_stt.go     # 流式 STT 集成
│   │
│   ├── sms/                     # 短信服务
│   │   ├── sms.go               # SMS 接口
│   │   ├── aliyun.go            # 阿里云短信（号码认证/传统短信）
│   │   └── mock.go              # Mock 实现
│   │
│   └── model/                   # 数据模型
│
├── pkg/                         # 公共工具包
│   ├── apperror/                # 应用错误定义
│   ├── jwt/                     # JWT 令牌生成/验证
│   ├── response/                # 统一响应格式（基于 sonic）
│   ├── snowflake/               # 雪花 ID 生成器
│   ├── validator/               # 参数校验器
│   └── observability/           # 可观测性（Prometheus metrics）
│
├── config/
│   ├── config.yaml              # 应用配置模板
│   ├── data/                    # 预置数据
│   ├── prompts/                 # 提示词模板
│   └── skills/                  # 技能配置
│
├── migrations/                  # 数据库迁移
├── tests/                       # 集成/单元测试
├── scripts/                     # 部署脚本
├── server/                      # 部署配置
├── uploads/                     # 上传文件
├── Dockerfile                   # 多阶段构建
├── docker-compose.yml           # 全栈编排（PostgreSQL + Redis + Qdrant + API）
├── Makefile                     # 构建命令
├── go.mod / go.sum              # Go 依赖
└── .env                         # 环境变量
---

## 三、Eino Agent 核心架构

### 3.1 InterviewGraph

使用 Eino `compose.Parallel` 实现三路并行评估：

```
GraphInput
    │
    └──→ compose.Parallel ────────────────────┐
          │              │              │       │
     [scorer_eval] [question_gen] [pronunciation]
          │              │              │       │
     *EvalNodeOutput  string    *PronNodeOutput
          │              │              │       │
          └──────────────┴──────────────┘       │
                         │                      │
                    [merge] ← map[string]any    │
                         │
                    *MergedOutput
```

| 分支 | 节点名 | 功能 | 超时 |
|------|--------|------|------|
| scorer_eval | ScorerAgent | 5 维评分 | 30s |
| question_gen | InterviewerAgent | 生成下一题 | 30s |
| pronunciation | PronunciationEval | 发音评测（语音模式） | 30s |

### 3.2 双 Agent

**ScorerAgent** — 评分 Agent，输出 4 维评分（Fluency/Grammar/Vocabulary/Content）+ overall，每维度带 issues/suggestions。

**InterviewerAgent** — 出题 Agent，使用完整 Q&A 历史做上下文出题，支持 LTM 快照注入。

### 3.3 长期记忆（LTM）

```
Recall(userID) → 搜索 Qdrant 过去会话摘要 → LTMSnapshot
Store(userID, sessionID, transcript) → LLM 总结 → bge-m3 向量化 → 存入 Qdrant
```

---

## 四、语音服务

### 4.1 STT（语音转文字）

支持 Fallback 链：`STT_PROVIDER=bailian,whisper,xunfei`

bailian(百炼) / qwen3_asr / whisper / volcengine / aliyun / azure / sensevoice / xunfei

### 4.2 TTS（文字转语音）

aliyun / elevenlabs / volcengine / siliconflow / azure / edge / xunfei

### 4.3 发音评测

xunfei_ise（默认）/ aliyun_asr，通过 `pronunciation.default_model` 配置

---

## 五、向量检索

| 后端 | 配置 | 说明 |
|------|------|------|
| pgvector（默认） | VECTOR_PROVIDER=pgvector | PostgreSQL 内置 |
| Qdrant | VECTOR_PROVIDER=qdrant | 专用向量数据库 |

混合检索（RRF）+ 可选 cross-encoder 精排（BAAI/bge-reranker-v2-m3）。

---

## 六、技术栈

| 分类 | 技术 |
|------|------|
| Web 框架 | Gin |
| AI 编排 | CloudWeGo Eino v0.8.13 |
| 数据库 | PostgreSQL / SQLite / Redis / Qdrant |
| 语音 SDK | 阿里云 NLS / 百炼 / 火山引擎 / 讯飞 / Azure / ElevenLabs / SiliconFlow |
| 配置 | Viper（YAML + 环境变量） |
| 日志 | Zap |
| 可观测性 | Prometheus |
| 序列化 | Sonic |
| 认证 | JWT + GitHub OAuth + 短信验证码 |
| 部署 | Docker Compose（PostgreSQL + Redis + Qdrant + API） |

---

## 七、关键变更对比

| 维度 | 旧架构 | 当前架构 |
|------|--------|----------|
| AI 编排 | 硬编码 if-else | Eino compose.Parallel |
| 评分出题 | 串行调用 | 独立双 Agent（评分 + 出题） |
| 题库 | 无 | QuestionPool + reference_answer |
| 发音评测 | 阿里云轮询 | 讯飞 ISE / 阿里云 ASR 双后端 |
| STT/TTS | 单一阿里云 | 8 STT + 7 TTS，Fallback 链 |
| 向量存储 | 无 | Qdrant / pgvector + bge-m3 |
| 长期记忆 | 无 | LTM 跨会话记忆 |
| 数据库 | SQLite | PostgreSQL 生产 + SQLite 开发 |
| 部署 | 单容器 | Docker Compose 全栈 |
