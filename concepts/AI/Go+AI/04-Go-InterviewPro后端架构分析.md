# InterviewPro 后端架构分析

> 基于 english-learner/backend 真实 Go 项目源码的深度架构分析。涵盖依赖注入、配置管理、Eino AI 编排、语音服务、WebSocket 会话流、题库与混合搜索等核心子系统。
> 源码路径：/home/tommychen/english-learner/backend

---

## 一、项目概览

### 1.1 定位

InterviewPro 是一个**英语面试模拟与评价平台**。用户与 AI 面试官进行实时语音/文本对话，回答面试问题，获得多维度评分和反馈。

### 1.2 技术栈

| 层级 | 技术 | 用途 |
|------|------|------|
| 语言 | Go 1.25 | 后端开发语言 |
| Web 框架 | Gin v1.12 | HTTP 路由 |
| WebSocket | gorilla/websocket | 实时面试会话 |
| ORM | GORM v1.31 | 数据库操作 |
| AI 编排 | CloudWeGo Eino v0.8.13 | LLM Agent 流水线 |
| 数据库 | PostgreSQL 16 / SQLite 3 | 业务数据存储 |
| 向量库 | Qdrant v1.16 / pgvector | 向量检索 + 混合搜索 |
| 缓存 | Redis 7 | 会话缓存 |
| 配置 | Viper | YAML + 环境变量 |
| 日志 | zap | 结构化日志 |
| 序列化 | sonic | 高性能 JSON |
| 监控 | Prometheus | 指标采集 |
| 部署 | Docker + Compose | 容器化部署 |

### 1.3 核心业务流

`
┌──────────┐    ┌─────────────┐    ┌──────────────────────┐    ┌──────────────┐
│  前端     │───→│  HTTP API   │───→│  WebSocket 会话层    │───→│  Eino Graph  │
│ (React)  │    │ (Gin路由)   │    │ (session_flow.go)   │    │  编排引擎    │
└──────────┘    └─────────────┘    └──────────────────────┘    └──────────────┘
                                          │                           │
                                          ├── STT/TTS 语音服务        ├── ScorerAgent（评分）
                                          ├── QuestionPool 题库       ├── InterviewerAgent（出题）
                                          ├── LongTermMemory         └── PronunciationEval（发音）
                                          └── Pronunciation 评测
`

---

## 二、项目结构

`
backend/
├── cmd/
│   ├── server/main.go          # 应用入口
│   ├── seed/main.go            # 数据播种
│   └── stt-check/main.go       # STT 连通性检查
├── config/
│   ├── config.yaml             # 默认配置文件
│   ├── data/                   # 预设数据
│   ├── prompts/                # 提示词模板
│   └── skills/                 # 技能定义
├── internal/
│   ├── agent/                  # Eino Agent 系统
│   │   ├── graph.go            # InterviewGraph 编排
│   │   ├── graph_runner.go     # Graph 执行器
│   │   ├── scorer.go           # 评分 Agent
│   │   ├── interviewer.go      # 面试官 Agent
│   │   ├── memory.go           # 长期记忆
│   │   ├── reranker.go         # 精排接口
│   │   ├── reranker_siliconflow.go # SiliconFlow 精排实现
│   │   ├── state.go            # Graph 状态定义
│   │   └── adapters.go         # 适配层（解耦循环依赖）
│   ├── app/
│   │   └── app.go              # 应用引导（DI 容器）
│   ├── config/
│   │   ├── config.go           # Viper 配置定义
│   │   └── credentials.go      # 运行时凭据管理
│   ├── crypto/
│   │   └── aes.go              # AES 加密
│   ├── dto/                    # 数据传输对象
│   │   ├── request/            # 请求 DTO
│   │   └── response/           # 响应 DTO
│   ├── handler/                # HTTP 处理器
│   │   ├── ws.go               # WebSocket 升级
│   │   ├── auth.go             # 认证
│   │   ├── interview.go        # 面试 CRUD
│   │   ├── question.go         # 题目 CRUD
│   │   └── ...
│   ├── middleware/              # Gin 中间件
│   │   ├── auth.go             # JWT 鉴权
│   │   ├── admin_auth.go       # 管理端鉴权
│   │   └── logger.go           # 请求日志
│   ├── model/                  # GORM 数据模型
│   │   ├── model.go            # 核心模型（User, Session, Message...）
│   │   ├── billing.go          # 计费模型
│   │   ├── admin.go            # 管理端模型
│   │   └── ...
│   ├── service/                # 业务逻辑层
│   │   ├── ai/                 # AI 服务
│   │   │   ├── types.go        # 接口 & 类型定义
│   │   │   ├── factory.go      # 模型工厂
│   │   │   ├── deepseek_model.go # DeepSeek 实现
│   │   │   ├── qwen_model.go   # Qwen 本地模型实现
│   │   │   └── ...
│   │   ├── speech/             # 语音服务
│   │   │   ├── types.go        # STT/TTS 接口定义
│   │   │   ├── factory.go      # 工厂 + 降级链
│   │   │   ├── bailian.go      # 百炼 STT
│   │   │   ├── whisper.go      # OpenAI Whisper
│   │   │   ├── elevenlabs.go   # ElevenLabs TTS
│   │   │   └── ...
│   │   ├── pronunciation/      # 发音评测
│   │   ├── question_pool.go    # 题库
│   │   ├── hybrid_search.go    # 混合搜索
│   │   ├── embedding.go        # 向量化
│   │   └── ...
│   ├── sms/                    # 短信服务
│   ├── ws/                     # WebSocket 层
│   │   ├── hub.go              # Hub（连接管理）
│   │   ├── client.go           # 客户端（收发消息）
│   │   ├── session_flow.go     # 会话业务流程
│   │   ├── interview_session.go# 会话状态
│   │   └── streaming_stt.go    # 流式 STT
│   └── ...
├── pkg/                        # 可复用工具包
│   ├── jwt/jwt.go              # JWT 生成与验证
│   ├── response/response.go    # 统一响应
│   ├── snowflake/snowflake.go  # 雪花 ID 生成
│   ├── validator/validator.go  # 参数校验
│   └── observability/metrics.go# Prometheus 指标
├── migrations/                 # 数据库迁移脚本
├── tests/                      # 测试
├── Dockerfile                  # 多阶段构建
├── docker-compose.yml          # 完整编排
└── go.mod                      # Go 模块定义
`

---

## 三、依赖注入与 App 引导

### 3.1 New() — 大型 DI 容器

`app.go 的 New(cfg *config.Config) 是整个应用的**依赖注入**入口。它按序初始化所有子系统：

`
New(cfg)
├── Logger（zap）
├── GORM DB（PostgreSQL / SQLite）
├── JWT Generator
├── Redis（go-redis）
├── Snowflake ID 生成器
├── AI Model Provider（DeepSeek / Qwen）
├── AI Service（AIService 接口）
├── Embedding Service（bge-m3）
├── Vector Provider（pgvector / Qdrant）
├── Speech Services（STT + TTS）
├── Pronunciation Service
├── Eino Agents
│   ├── ScorerAgent
│   ├── InterviewerAgent
│   ├── LongTermMemory
│   ├── InterviewGraph → GraphRunner
│   └── Reranker（SiliconFlow）
├── Business Services
│   ├── AuthService
│   ├── InterviewService
│   ├── QuestionService
│   ├── QuestionPool
│   └── ...
├── WebSocket Hub
└── Gin Router（注册所有路由 + 中间件）
`

**设计要点**：
- 所有依赖在 New() 中一次性构建，之后通过结构体字段传递
- 使用**适配器模式**（internal/agent/adapters.go）解决 `gent → service → agent 循环依赖
- AIService 是**门面接口**，对外暴露 GenerateQuestion / EvaluateFiveDimensions / GenerateFeedback / ParseResumeText 等方法

### 3.2 关键代码片段

`go
// app.go 核心结构
type App struct {
    Server       *http.Server
    DB           *gorm.DB
    Logger       *zap.Logger
    Config       *config.Config
    AuthSvc      *service.AuthService
    JWTGen       *jwt.Generator
    InterviewSvc *service.InterviewService
    QuestionSvc  *service.QuestionService
    WSHub        *ws.Hub
    STTProvider  svcspeech.STTProvider
    TTSProvider  svcspeech.TTSProvider
    VectorSvc    service.VectorProvider       // nil = 降级运行
    EmbeddingSvc *service.EmbeddingService
}
`

### 3.3 优雅关闭

`go
func (a *App) Shutdown() {
    // 1. 停止接受新请求
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    a.Server.Shutdown(ctx)
    // 2. 关闭 WebSocket Hub
    a.WSHub.GracefulShutdown()
    // 3. 等待 WebSocket 协程退出
    a.WSHub.WaitPendingOperations()
    // 4. 关闭数据库连接
    sqlDB, _ := a.DB.DB()
    sqlDB.Close()
}
`

---

## 四、配置系统

### 4.1 Viper 配置优先级

internal/config/config.go 使用 Viper 实现配置覆盖链：

`
硬编码默认值 < config.yaml < 环境变量（.env）
`

**关键特性**：
- mapstructure tag 映射 YAML 到 Go 结构体
- 短名环境变量绑定（如 AI_PROVIDER → `i.provider）
- k3s 部署兼容（DB_HOST, DB_PORT 等）
- 配置按模块分组：server, database, `i, speech, embedding, qdrant, ector 等

### 4.2 配置结构

`go
type Config struct {
    Server        ServerConfig        // port, mode, allowed_origins
    Database      DatabaseConfig      // driver(postgres/sqlite), dsn
    Redis         RedisConfig         // addr, password, db
    JWT           JWTConfig           // access_secret, refresh_secret
    AI            AIConfig            // provider, model, api_key
    Speech        SpeechConfig        // STT/TTS provider 组合, 各厂商凭据
    Pronunciation PronunciationConfig // xunfei_ise / aliyun_asr
    Embedding     EmbeddingConfig     // bge-m3 配置
    Qdrant        QdrantConfig        // host, port
    Vector        VectorProviderConfig // pgvector / qdrant
    Admin         AdminConfig         // 管理端 JWT
    SMS           SMSConfig           // 短信服务商
    OAuth         OAuthConfig         // GitHub OAuth
}
`

### 4.3 运行时凭据管理

internal/config/credentials.go 实现了**动态凭据管理**：

- 启动时从 `pi_credentials 表加载加密凭据到内存缓存
- 优先使用 DB 凭据，.env 值作为兜底
- 管理员在后台更新凭据后可通过回调热重载
- 凭据用 AES 加密存储

`go
// GetCredential 读取凭据：DB 优先，.env 兜底
func GetCredential(key string) string {
    // 先查 credCache
    if val, ok := credCache[key]; ok && val != "" {
        return val
    }
    // 兜底 os.Getenv
    return os.Getenv(key)
}
`

---

## 五、Eino Agent 系统（核心亮点）

### 5.1 架构演进

| 阶段 | 架构 | 延迟 | 问题 |
|------|------|------|------|
| v1 (旧) | 顺序执行：评分→出题→发音 | ~18s | 串行阻塞，1+1+1=3倍慢 |
| v2 (Eino) | **compose.Parallel** 三路并行 | ~8s | 3 个分支独立运行，取最大值 |

### 5.2 图拓扑（internal/agent/graph.go）

`
GraphInput
    │
    └──→ compose.Parallel ───────────────────────┐
          │             │             │            │
     [scorer_eval] [question_gen] [pronunciation]  │
          │             │             │            │
     *parallelResult  *parallelResult *parallelResult
          │             │             │            │
          └─────────────┴─────────────┘            │
                         │                         │
                    [merge] ← map[string]any       │
                         │                         │
                    *MergedOutput
`

**每个分支独立 30s 超时**，一个分支失败不影响其他分支（非致命）。

### 5.3 ScorerAgent（评分）

`go
type ScorerAgent struct {
    chatModel    model.ChatModel   // Eino ChatModel 接口
    systemPrompt string
    logger       *zap.Logger
}

func (s *ScorerAgent) Evaluate(ctx context.Context, input *ScorerInput) (*ScorerOutput, error)
`

**评分维度**（四个维度，每题评分）：

| 维度 | 范围 | 说明 |
|------|------|------|
| Fluency（流利度） | 1-10 | 流畅度和自然度 |
| Grammar（语法） | 1-10 | 语法准确性 |
| Vocabulary（词汇） | 1-10 | 词汇选择和多样性 |
| Content（内容） | 1-10 | 相关性、深度和结构 |

**评分校准**：8-10 优秀 / 6-7 良好 / 4-5 一般 / 1-3 需大幅改进

**解析**：parseScorerJSON() 处理各种 LLM 输出格式（Markdown fence、前缀文本、截断等），只提取 JSON。

### 5.4 InterviewerAgent（出题）

`go
type InterviewerAgent struct {
    chatModel model.ChatModel
    skills    svcai.SkillProvider   // 技能配置
    prompts   svcai.PromptProvider  // 提示词模板
    logger    *zap.Logger
}

func (a *InterviewerAgent) GenerateNextQuestion(ctx context.Context, input *InterviewInput) (string, error)
`

**关键区别**（对比旧方案）：
- 旧方案：AI 只看到 PreviousQuestions []string（仅问题文本）
- 新方案：AI 看到**完整对话历史**（assistant/user 交替），支持真正的追问

**出题上下文**：
`
System: AICharacter + ScenarioType + Difficulty
History: Q1 → A1 → Q2 → A2 → ... (interleaved)
User: Role, Company, SubScenario, Keywords, ResumeContext, LTMSummary
`

### 5.5 LongTermMemory（长期记忆）

`go
type InterviewMemory struct {
    store    VectorStore      // Qdrant / pgvector
    embedder Embedder         // bge-m3 embedding
    aiSvc    svcai.ModelProvider
    logger   *zap.Logger
}
`

**流程**：
1. **Recall**：会话启动时，从向量数据库召回该用户最近 5 次面试摘要
2. **Store**：会话结束时，LLM 生成 2-3 句摘要，embedding 后 upsert 到向量库

**摘要内容**：做得好的地方、困难的地方、一个待改进方向

### 5.6 GraphRunner（执行器）

graph_runner.go 封装调用流程：

`
Run(input)
├── QuestionPool 提前获取题目（如命中则绕过 LLM）
├── 构建 GraphInput → RunWithTiming（Eino 并行执行）
├── 失败处理 → 降级为默认评分 + 题库题目
├── 成功 → MergedOutput.ToFiveDimensionResult()
├── 发音分数提取（语音模式）
└── Timing 采集（scorer_eval / question_gen / pronunciation 各分支耗时）
`

**ToFiveDimensionResult()** 使用 ArithmeticMeanOverall 计算总分（算术均值），而非 LLM 自评的 overall_score（LLM 自评偏低，某维度低会拖累整体）。

---

## 六、语音服务

### 6.1 双接口体系

`go
// STT: Speech-to-Text
type STTProvider interface {
    Transcribe(ctx context.Context, audio []byte, format string) (*Transcription, error)
    Prewarm(ctx context.Context) error
    Engine() string
}

// TTS: Text-to-Speech
type TTSProvider interface {
    Synthesize(ctx context.Context, text, voice string) ([]byte, error)
    Engine() string
}

// 流式 STT（部分 Provider 支持）
type StreamingSTTProvider interface {
    OpenStream(ctx context.Context) (STTStream, error)
}

type STTStream interface {
    Feed(pcm []byte) error
    Finish() (*Transcription, error)
    Close() error
    Engine() string
}
`

### 6.2 降级链（Fallback Chain）

STT_PROVIDER=bailian,whisper,xunfei 和 TTS_PROVIDER=elevenlabs,edge 支持**逗号分隔**的降级链：

`
NewSTTProvider("bailian,whisper,xunfei", ...)
  ├── 尝试 bailian → 成功则返回
  ├── 失败 → 尝试 whisper → 成功则返回
  └── 全部失败 → 返回错误
`

每个分支有独立 context.WithCancel，避免 provider 间互相干扰。

### 6.3 支持的 Provider

| 类型 | Provider | 说明 |
|------|----------|------|
| STT | bailian | 百炼 paraformer-realtime-v2（推荐） |
| STT | whisper | OpenAI Whisper（免费档） |
| STT | volcengine | 火山引擎 ASR（付费档） |
| STT | aliyun | 阿里云 NLS |
| STT | qwen3_asr | Qwen3-ASR-Flash |
| STT | xunfei_iat | 讯飞语音听写 |
| STT | sensevoice | 硅基流 SenseVoiceSmall |
| STT | azure | Azure Speech |
| TTS | elevenlabs | ElevenLabs（推荐） |
| TTS | aliyun | 阿里云 TTS |
| TTS | volcengine | 火山引擎 TTS |
| TTS | edge | edge-tts（免费） |
| TTS | siliconflow | CosyVoice2 |
| TTS | xunfei | 讯飞 TTS |
| TTS | azure | Azure Speech |

### 6.4 健康监控（SpeechHealth）

每个降级链记录成功/失败到 SpeechHealth（Prometheus 指标），支持：
- speech_stt_success_total
- speech_stt_failure_total
- speech_tts_success_total
- speech_tts_failure_total

---

## 七、WebSocket 会话流

### 7.1 架构分层

`
hub.go           → 连接管理（注册/注销/广播/优雅关闭）
client.go        → WS 连接生命周期 + 消息收发 + 事件序列化
session_flow.go  → 面试业务逻辑（本文档）
interview_session.go → 会话状态
`

**职责边界**：
- client.go：只负责"怎么收发消息"，不关心"消息内容"
- session_flow.go：只负责"面试业务怎么做"，不关心消息如何序列化

### 7.2 Hub 并发模型

使用 **Actor 模型（单线程事件循环）**：

- hub.Run() 在单个 goroutine 中处理 register/unregister 事件
- sync.RWMutex 保护 clients map，供其他 goroutine（Broadcast/SendToClient）读
- unregister 使用缓冲 channel（64），减少批量断连时的栈压力
- `tomic.Bool 标记 shutdown 状态

### 7.3 会话启动流程

`
handleStartSession
├── 步骤1: 解析客户端参数（scenarioType, subScenario, company, role, difficulty, aiCharacter）
├── 步骤2: 从 DB 加载会话上下文（兜底 + positionID）
├── 步骤3: 加载简历上下文（个性化模式）
├── 步骤4: 校验 AI 角色（hr/tech-lead/boss，拒绝旧人名值）
├── 步骤5: 异步召回 LTM（10s 超时，不阻塞首问）
├── 步骤6: 发送 session_started 事件
├── 步骤7: 预热 STT 引擎（20s 超时，context.Background 避免被会话取消）
├── 步骤8: 重连检测（有历史消息则回放，不重新出题）
├── 步骤9: 生成首问（预生成缓存 > 题库 > AI 兜底）
└── 步骤10: 异步 TTS 语音合成
`

### 7.4 消息处理

所有 WebSocket 事件通过 client.handleMessage() 分发（client.go）：

| 事件类型 | 处理函数 | 说明 |
|----------|----------|------|
| start_session | handleStartSession | 启动会话 |
| user_message | handleUserMessage | 处理用户回答 |
| end_session | handleEndSession | 结束会话并生成反馈 |
| ping | handlePing | 心跳 |

**handleUserMessage** 是核心流程：
`
handleUserMessage
├── 语音输入 → STT 转写（流式/非流式）
├── 文本输入 → 直接使用
├── 保存消息到 DB
├── GraphRunner.Run（Eino 并行：评分+出题+发音）
├── 发送评分结果（five_dimension_evaluation）
├── 保存下一轮题目
├── TTS 合成下一题语音
└── 发送下一题（ai_response_final）
`

---

## 八、题库与混合搜索

### 8.1 问题来源优先级

`
1. starterCache（预生成缓存）—— <1ms，无 embedding 调用
2. HybridSearch（向量+关键词 RRF 融合）—— 500ms~2s
3. ListByScene（基础 DB 查询）—— 兜底
4. AI Supplement（AI 现场补题，5 题/批）—— 题库耗尽时
5. 硬编码默认问题—— 最后兜底
`

### 8.2 混合搜索（Hybrid Search）

`go
// hybrid_search.go
HybridSearch(ctx, query, positionID, scene, limit, vectorSvc, embeddingSvc, reranker)
├── Vector Search（Qdrant/pgvector）：query → bge-m3 embedding → 向量相似度
├── Keyword Search（PostgreSQL full-text search）：tsquery 匹配
├── RRF Fusion（Reciprocal Rank Fusion）：融合两路结果
└── Optional Reranker（SiliconFlow cross-encoder）：精排（默认关闭，待基线验证）
`

### 8.3 题库去重

- **UUID 去重（主）**：user_question_history 表，按 question_bank_id
- **文本去重（兜底）**：user_asked_questions 表，按 question text
- 时间窗口：最近 30 天

### 8.4 权重洗牌（Weighted Shuffle）

`
weightedShuffle(questions)
├── 基础权重：QualityScore（use_count 计算）
├── 冷启动加成：7天内的新题权重 1x~2x（越新权重越大）
├── 随机因子：rand.ExpFloat64() / weight
└── 排序：高权重/高随机的题优先出现
`

---

## 九、AI 模型提供者

### 9.1 工厂模式

`go
// service/ai/factory.go
func GetModelProviderFromConfig(cfg AIConfig, logger *zap.Logger) ModelProvider {
    switch cfg.Provider {
    case "deepseek_eino":
        return NewEinoDeepSeekModel(apiKey, baseURL, model, timeout, logger)
    case "qwen_local":
        return NewQwenLocalModel(url, modelID, temperature, maxTokens, maxParallel, timeout, logger)
    default:  // "deepseek"
        return NewDeepSeekModel(apiKey, baseURL, model, timeout, logger)
    }
}
`

| Provider | 说明 | 适用场景 |
|----------|------|----------|
| deepseek | 原生 DeepSeek API | 开发/生产 |
| deepseek_eino | 通过 Eino sdk 调用 DeepSeek | Eino 链路追踪 |
| qwen_local | Ollama 本地模型（如 qwen3-scoring） | 离线/自建 |

### 9.2 ModelProvider 接口

`go
type ModelProvider interface {
    Generate(ctx context.Context, system, user string) (string, error)
    GenerateWithJSON(ctx context.Context, system, user string, target interface{}) error
}
`

### 9.3 AIService 门面

`go
type AIService interface {
    GenerateQuestion(ctx context.Context, req *GenerateQuestionRequest) (string, error)
    EvaluateAnswer(ctx context.Context, req *EvaluateAnswerRequest) (*EvaluationResult, error)
    EvaluateFiveDimensions(ctx context.Context, req *EvaluateAnswerRequest) (*FiveDimensionResult, error)
    GenerateFeedback(ctx context.Context, session *model.InterviewSession, 
        messages []model.Message, voiceSessionHint bool) (*FeedbackResult, error)
    ApplyObviousPenalties(answer string, scores *FeedbackResult, hasVoiceInput bool) *FeedbackResult
    ParseResumeText(ctx context.Context, rawText string) (string, error)
}
`

---

## 十、数据模型

### 10.1 核心表结构

| 表 | 主键 | 说明 |
|----|------|------|
| users | UUID | 用户（支持 GitHub OAuth + 手机号注册） |
| interview_sessions | UUID | 面试会话 |
| messages | UUID | 消息（user/ai，含评分 JSON） |
| feedbacks | UUID | 会话反馈 |
| resumes | UUID | 简历解析结果 |
| positions | UUID | 预置岗位定义 |
| user_positions | UUID | 用户选择的岗位 |
| interview_questions | UUID | 题库题目 |
| question_bank | UUID | 预置题目（旧表） |
| user_question_history | auto-inc | 跨会话去重记录 |

### 10.2 特色设计

- **UUID 主键**：所有业务表使用 UUID（而非自增 ID），便于分布式和数据迁移
- **gorm.Model** 手动替代：自定义 ID、CreatedAt、UpdatedAt 字段，精确控制
- **JSONB 字段**：interview_questions.tags、key_points、scoring_rubric 等使用 GORM serializer 序列化到 JSONB
- **软删除 + 级联**：User 删除时级联删除 Sessions 和 Messages

### 10.3 用户订阅模型

`go
func (u *User) FeatureSet() *FeatureConfig {
    switch u.EffectiveTier(time.Now()) {
    case "premium", "pro":
        // TTS=premium, STT=volcengine, 无限轮次, AI=deepseek-v4-pro
    default:  // free
        // TTS=browser, STT=whisper, 每会话3轮, AI=deepseek-v4-flash
    }
}
`

---

## 十一、认证系统

### 11.1 三种认证方式

| 方式 | 入口 | 说明 |
|------|------|------|
| 手机号 + 验证码 | POST /api/auth/send-code + /api/auth/login | 阿里云短信（支持号码认证） |
| GitHub OAuth | GET /api/auth/github/login + /callback | 社交登录 |
| JWT 双令牌 | 登录后返回 access + refresh token | 身份凭证 |

### 11.2 JWT 双令牌

`go
// access_token: 15分钟过期，Bearer 认证
// refresh_token: 7天过期，用于无感续期
type JWTConfig struct {
    AccessSecret  string  // 签名密钥
    RefreshSecret string  // 刷新密钥
    AccessExpiry  int     // 分钟
    RefreshExpiry int     // 天
}
`

### 11.3 短信服务

支持两种阿里云短信 API：
- `liyun_dypns：号码认证服务（个人实名可用，赠送签名+模板）
- `liyun：传统短信服务（需企业资质）
- mock：开发环境模拟

---

## 十二、部署与运维

### 12.1 Docker 多阶段构建

`
Stage 1: golang:1.25-alpine → 编译 Go 二进制（CGO_ENABLED=1, sqlite）
Stage 2: node:22-alpine → 构建管理前端
Stage 3: alpine:3.20 → 运行（仅含 ca-certificates + sqlite + ffmpeg + edge-tts）
`

**关键配置**：
- GOMEMLIMIT=400MiB：Go 软内存限制
- GOGC=50：GC 更激进（降低峰值内存）
- USER nobody：非 root 运行
- HEALTHCHECK：每 30s 检测 /healthz

### 12.2 Docker Compose 编排

| 服务 | 镜像 | 端口 |
|------|------|------|
| postgres | postgres:16-alpine | 5432 |
| redis | redis:7-alpine | 6379 |
| qdrant | qdrant/qdrant:v1.16.0 | 6333(HTTP), 6334(gRPC) |
| api | 本地构建 | 8080 |

### 12.3 数据库双驱动

`go
// 开发环境：SQLite（零配置）
// 生产环境：PostgreSQL（k3s 部署）
switch cfg.Database.Driver {
case "postgres":
    db, err = gorm.Open(postgres.Open(cfg.Database.DSN))
default:  // sqlite
    db, err = gorm.Open(sqlite.Open(cfg.Database.DSN))
}
`

---

## 十三、关键设计模式总结

### 13.1 适配器模式

internal/agent/adapters.go 解决包循环依赖：

`go
// agent 包需要调用 service 包的接口，但 service 包也引用了 agent 包的 Reranker
// 解决方案：在 agent 包中定义接口，由 service 包实现

// adapters.go
type QuestionDeck interface {
    GetQuestion(ctx context.Context, previousQuestions []string) (text, answer string, ok bool)
}
type Reranker interface { ... }
`

### 13.2 工厂模式

- service/ai/factory.go：AI Model Provider 工厂
- service/speech/factory.go：STT/TTS Provider 工厂（支持降级链）
- service/vector_factory.go：向量数据库工厂（pgvector / Qdrant）

### 13.3 门面模式

- service/ai/types.go 的 AIService 接口：统一对外暴露 AI 能力
- `gent/graph_runner.go 的 GraphRunner：封装 Eino Graph 调用细节

### 13.4 策略模式

- PronunciationConfig.DefaultModel：xunfei_ise / aliyun_asr 二选一
- speech health.go：统一健康监控策略

### 13.5 观察者模式

- config/credentials.go 的 RegisterOnCredentialsChanged：凭据更新回调
- 语音 provider 注册回调，凭据更新后自动重新初始化客户端

---

## 十四、性能优化

### 14.1 首题零延迟

HTTP POST /session 时后台异步运行 StartPregen()：
- **快速路径**（starterCache）：从内存直接选 Q1，< 1ms，无 embedding 调用
- **兜底路径**：CreateSessionDeck + HybridSearch，500ms~2s
- 结果缓存 60s，WebSocket 连接时直接取用

### 14.2 Eino 并行执行

`go
// graph.go: compose.Parallel 三路并行
parallel := compose.NewParallel()
parallel.AddLambda("scorer", scorerLambda)      // LLM 评分 3-10s
parallel.AddLambda("question", questionLambda)  // LLM 出题 3-10s（题库命中则 0s）
parallel.AddLambda("pronunciation", pronLambda) // 发音评测 1-3s（语音模式）
// 总延迟 = max(评分, 出题, 发音) ≈ 8s，原来顺序执行 ≈ 18s
`

### 14.3 题库短路

> 90% 以上的轮次题库有候选，直接返回，跳过 LLM 出题调用（省 5-10s）。

`go
if input.PoolQuestion != "" {
    return &parallelResult{Q: input.PoolQuestion}, nil  // 零 LLM 调用
}
`

### 14.4 STT 预热

会话启动时后台预热 STT 引擎（20s 超时），消除首次冷启动延迟（5-15s）。

### 14.5 批处理

- Embedding 批处理：每批最多 20 条文本，减少 API 调用次数
- AI 补题批处理：一次生成 5 题

---

## 十五、测试体系

### 15.1 测试分类

| 测试类型 | 位置 | 说明 |
|----------|------|------|
| 单元测试 | *_test.go | 函数级别测试 |
| Agent 测试 | 	ests/agent/ | ScorerAgent + InterviewerAgent |
| 集成测试 | 	ests/integration/ | 发音评测 + 全流程 |
| Handler 测试 | 	ests/handler/ | HTTP API 测试 |
| Service 测试 | 	ests/service/ | 业务逻辑测试 |

### 15.2 测试工具

- httptest：Gin HTTP 测试
- gorm.DB Mock 替换
- 环境变量控制：SKIP_INTEGRATION=true 跳过集成测试

---

## 附：面试 Q&A

### Q1: 为什么用 Eino 而不是直接调 DeepSeek API？

Eino 提供**声明式编排**（compose.Parallel）、**Graph 拓扑可视化**、**回调机制**（timing 采集），且支持运行时切换模型（从 DeepSeek 到 Qwen 本地模型只需改配置）。

### Q2: 为什么用 parallel 而不是 goroutine？

compose.Parallel 与 Eino 的 Graph 执行引擎深度集成：自动处理超时、上下文传递、错误传播、Streaming。手写 goroutine + WaitGroup 需要重复实现这些能力。

### Q3: 向量库为什么支持 pgvector 和 Qdrant 双后端？

- **pgvector**：零额外运维成本，对 10 万级题目足够
- **Qdrant**：百万级 + 需要混合搜索（BM25 过滤）时切换，VECTOR_PROVIDER=qdrant

### Q4: 降级链如何保证高可用？

- STT/TTS 各配置 2-3 个 Provider，按优先级排列
- 每个请求独立超时，单 Provider 失败自动切换到下一个
- Prometheus 健康监控 + 凭据后台热更新
- 系统级降级：向量库不可用时走纯 DB 查询 & 默认评分

### Q5: WebSocket 如何处理大批量断连？

- unregister channel 缓冲 64，读写分离的上下文
- graceful shutdown 流程：停止接收新请求 → 关闭所有 WS 连接 → 等待 in-flight 完成（30s 超时）
- atomic.Bool + WaitGroup 确保资源安全释放
