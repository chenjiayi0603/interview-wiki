# InterviewPro-AI面试 核心类组织关系

## 4. 核心类组织关系

### 4.1 App — 组合根

`internal/app/App` 是整个应用的**组合根（Composition Root）**，持有所有顶层依赖的引用：

```
App
├── *http.Server          ← Gin 引擎包装
├── *gorm.DB              ← 数据库连接
├── *zap.Logger           ← 结构化日志
├── *config.Config        ← 全局配置
├── *service.AuthService  ← 认证业务
├── *jwt.Generator        ← Token 生成/验证
├── *service.InterviewService ← 面试会话管理
├── *service.QuestionService  ← 题库查询
├── *ws.Hub               ← WebSocket 连接中心
├── service.STTProvider   ← 语音识别（接口）
└── service.TTSProvider   ← 语音合成（接口）
```

`App.New()` 按以下顺序构建，严格遵循依赖拓扑：

```
Logger → DB → JWTGen → AuthSvc → ScenarioConfigSvc → InterviewSvc
→ ModelProvider (工厂) → DeepSeekService → STT/TTS Provider
→ PronunciationSvc → QuestionPool → Hub (+ go Run())
→ Handlers → Routes → http.Server
```

---

### 4.2 Service 层

Service 层是最复杂的一层，包含多个子系统：

#### AI 服务体系

```
AIService (interface)
├── GenerateQuestion()
├── EvaluateAnswer()
├── EvaluateFiveDimensions()
├── GenerateFeedback()
├── ApplyObviousPenalties()
└── TmplCacheExists() / TmplCacheGet()

         ┌─────────────────────┐
         │   DeepSeekService   │  ← 主要实现
         │  (implements AIService)│
         │  ┌───────────────┐  │
         │  │ ModelProvider │  │  ← 策略：可切换底层模型
         │  │ (interface)   │  │
         │  └──────┬────────┘  │
         │  sem chan struct{}   │  ← 信号量限流
         │  TemplateFeedbackCache│← 模板缓存
         └──────────┬──────────┘
                    │ 委托调用
         ┌──────────▼──────────┐
         │   OptimizedAIService│  ← 装饰器：性能优化包装
         │  ┌──────────────┐   │
         │  │ base AIService│  │
         │  │ ParallelExecutor│ │  ← 并行执行器
         │  │ BatchProcessor│  │  ← 批处理器
         │  │ TemplateFeedbackCache│
         │  └──────────────┘   │
         └─────────────────────┘
```

#### ModelProvider 体系（策略模式）

```
ModelProvider (interface)
├── Generate(ctx, systemPrompt, userPrompt) (string, error)
├── GenerateWithMaxTokens(...)
├── StreamGenerate(..., onChunk func(string))
└── GetName() string

实现类：
├── DeepSeekModel   ← 云端 API（OpenAI 兼容协议）
│   ├── http.Client（带超时）
│   ├── sem chan struct{}（并发限制 10）
│   └── 指数退避重试（maxRetries=1）
│
└── QwenLocalModel  ← 本地 llama.cpp（/v1/chat/completions）
    ├── http.Client（自定义 Transport，高并发连接复用）
    ├── sem chan struct{}（并发限制 = maxParallel）
    └── <think>...</think> 标签过滤（Qwen3 推理模式处理）

工厂函数：
GetModelProviderFromConfig(cfg AIConfig) ModelProvider
  ├── cfg.Provider == "qwen_local" → NewQwenLocalModel(...)
  └── default                      → NewDeepSeekModel(...)
```

#### STT/TTS Provider 体系

```
STTProvider (interface)
└── Transcribe(ctx, audio []byte, format string) (*Transcription, error)

实现：
├── AliyunSTT    ← 阿里云实时语音识别
├── BailianSTT   ← 百炼语音识别
└── WhisperSTT   ← OpenAI Whisper

TTSProvider (interface)
└── Synthesize(ctx, text, voice string) ([]byte, error)

实现：
├── AliyunTTS     ← 阿里云语音合成
└── ElevenLabsTTS ← ElevenLabs

工厂函数：
NewSTTProvider(cfg) STTProvider
NewTTSProvider(cfg) TTSProvider
```

#### 性能加速组件

```
ParallelExecutor               ← 受控并发任务执行器
├── maxWorkers int
├── sem chan struct{}           ← 信号量
└── Execute(ctx, []Task) []TaskResult

Pipeline                       ← 流水线（串行阶段组合）
└── Run(ctx, input) (output, error)

BatchProcessor                 ← 批量请求聚合（GPU 批推理优化）
├── maxBatchSize int
├── maxWaitTime  time.Duration
├── queue chan BatchRequest
└── processFn func(ctx, []BatchRequest) []BatchResult

HTTPClientPool / PooledHTTPClient ← HTTP 连接池复用
RequestBatcher                 ← 请求合并（减少 API 调用次数）
```

**落地情况（代码审查结论）**：上述列表描述的是**已实现类型与能力**，不等于**主流程已全部接入**。

| 组件 | 主路径是否使用 | 说明 |
|------|----------------|------|
| `ParallelExecutor` | **否**（生产路径） | 仅在 `NewOptimizedAIService` 内构造并保存于字段 `executor`，**任何方法均未调用** `Execute` / `StreamExecute` 等；`internal/app` 使用的是 `NewDeepSeekServiceWithProvider`，**未**使用 `OptimizedAIService`。 |
| `Pipeline`（`parallel_executor.go`） | **否** | 仅 `parallel_executor_test.go` 单测使用 `NewPipeline`，业务代码无引用。 |
| `BatchProcessor`（`gpu_batch_processor.go`） | **否**（作为加速路径） | 仅在 `NewOptimizedAIService` 内创建；`OptimizedAIService` 未接入应用组装；且包装类内**从未调用** `batchProc.Submit`，`processBatch` 为占位实现（返回空 `Result`）。 |
| `PooledHTTPClient` / `HTTPClientPool` / `RequestBatcher` | **否** | 定义于 `connection_pool.go`，仓库内**无**其它包调用 `NewPooledHTTPClient` / `NewHTTPClientPool` / `NewRequestBatcher`，属预留实现。 |
| `ParallelExecutor`（作为 API） | **是**（仅测试） | `parallel_executor_test.go` 覆盖 `Execute`、依赖图执行、`StreamExecute` 等。 |

**补充（WebSocket 侧同名/并行实现）**：`internal/ws/optimized_client.go` 中的 `OptimizedEvaluator`、`PipelineEvaluator`、本地 `BatchProcessor` **未被** `client.go` 引用，当前实时面试仍走 `Client` 主路径直接调 `AIService`，上述 WS 优化层亦为预留。

**若要真正启用「优化包装」**：在 `internal/app/app.go` 中将 `aiSvc` 改为 `service.NewOptimizedAIService(baseDeepSeek, logger)`（并补齐 `executor` / `batchProc.Submit` 调用或删除死字段），再按需把连接池接入 `ModelProvider` 的 HTTP 客户端。

#### 缓存组件

```
TemplateFeedbackCache          ← 反馈模板缓存（TTL=24h，容量=500）
├── cache map[string]*CacheEntry
└── Get/Set/Exists/Evict

（五维评估结果**不做**应用层 EvalCache；`OptimizedAIService.EvaluateFiveDimensions` 仅委托 `base`。）
```

---

### 4.3 WebSocket 层

#### Hub — 连接中心（Actor 模型）

```
Hub
├── clients  map[string]*Client   ← 活跃连接表（RWMutex 保护）
├── register   chan *Client        ← 注册通道
├── unregister chan *Client        ← 注销通道（缓冲 64）
├── shutdown   atomic.Bool        ← 关闭标志
└── wg         sync.WaitGroup     ← 等待所有 Goroutine 退出

服务依赖（延迟注入，通过 SetServices 方法）：
├── aiService        AIService
├── sttProvider      STTProvider
├── ttsProvider      TTSProvider
├── questionPool     *QuestionPool
├── interviewSvc     *InterviewService
└── pronunciationSvc *PronunciationService
```

Hub 采用**单线程事件循环**模式处理注册/注销，避免对 `clients` map 的并发写竞争。读操作（Broadcast/SendToClient）通过 `RWMutex` 保护。

#### Client — 单会话状态机

```
Client
├── conn       *websocket.Conn    ← 底层连接
├── hub        *Hub
├── send       chan []byte         ← 发送缓冲（256条）
├── saveCh     chan saveRequest    ← DB 写入序列化（64条）
│
├── 原子状态标志：
│   ├── isEvaluating  atomic.Bool  ← 防止并发评估（CAS 锁）
│   ├── closed        atomic.Bool  ← 防止写已关闭通道
│   ├── feedbackGenerated atomic.Bool ← 确保 Feedback 幂等生成
│   └── lastEventID   atomic.Int32 ← 单调递增事件序号
│
└── writeMu    sync.Mutex          ← 仅保护 ws.Conn 写操作

Goroutine 模型（每个 Client 4~N 个 Goroutine）：
├── ReadPump()           ← 读取 WS 消息，分发事件
├── WritePump()          ← 从 send 通道写入 WS
├── saveLoop()           ← 序列化 DB 写入（避免并发 INSERT）
└── evaluateAndRespond() ← 临时 Goroutine：AI 评估 + TTS 合成
```

---

### 4.4 Handler 层

所有 Handler 均为**薄层**，只负责：请求绑定 → 参数校验 → 调用 Service → 统一响应格式。

```
InterviewHandler
├── NewInterviewHandler(svc *InterviewService, aiSvc AIService)
└── 方法：GetScenarios / GetSubScenarios / CreateSession /
         EndSession / ListSessions / GetFeedback / GetJobTypes

AuthHandler
└── Register / Login / Refresh / Logout

QuestionHandler
└── List / Random / Categories

UserHandler
└── GetProfile / ChangePassword

HealthHandler
└── Liveness(/healthz) / Readiness(/readyz)

WSHandler
└── HandleWS / GetActiveSessions / VerifyWSAuth(middleware)

AIModelHandler
└── GetModelStatus
```

---

### 4.5 公共包（pkg）

| 包 | 核心类型 | 说明 |
|----|---------|------|
| `pkg/jwt` | `Generator`, `Claims`, `TokenPair` | Access/Refresh Token 双令牌 |
| `pkg/apperror` | `Kind`, `E` | 结构化错误（Kind 枚举 + HTTP 状态映射） |
| `pkg/response` | `Response` | 统一 JSON 响应：`{code, message, data}` |
| `pkg/validator` | `*validator.Validate` | 基于 `go-playground/validator` 封装 |
| `pkg/observability` | Prometheus 指标变量 | AI 并发数、Token 用量、请求耗时 |

[src: raw/ingested/3项目/InterviewPro-AI面试/ai_interview-4.-核心类组织关系.md]