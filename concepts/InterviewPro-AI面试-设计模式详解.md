# InterviewPro-AI面试 设计模式详解

## 5. 设计模式详解

### 5.1 工厂方法模式（Factory Method）

**位置**：`service/model_factory.go`、`service/speech.go`

```go
// 根据配置动态选择底层实现
func GetModelProviderFromConfig(cfg AIConfig, logger *zap.Logger) ModelProvider {
    switch cfg.Provider {
    case "qwen_local":
        return NewQwenLocalModel(...)
    default:
        return NewDeepSeekModel(...)
    }
}
```

意义：调用方（`DeepSeekService`、`App`）只依赖 `ModelProvider` 接口，运行时由工厂决定具体实现，支持**无侵入地切换 AI 后端**。

---

### 5.2 策略模式（Strategy）

**位置**：`AIService`、`ModelProvider`、`STTProvider`、`TTSProvider`

```
context.AIService ──────────────────► interface
                                        │
                              ┌─────────┴──────────┐
                    DeepSeekService          （可扩展）OtherAIService

context.ModelProvider ──────────────► interface
                                        │
                           ┌────────────┴────────────┐
                    DeepSeekModel               QwenLocalModel
```

各接口定义行为契约，`Hub` 和 `Handler` 只持有接口引用，具体实现由 `App` 装配时注入。

---

### 5.3 装饰器模式（Decorator）

**位置**：`service/optimized_ai.go`

```go
type OptimizedAIService struct {
    base        AIService
    executor    *ParallelExecutor // 当前生产路径未使用 Execute
    batchProc   *BatchProcessor   // 占位；主流程未 Submit
    // EvaluateFiveDimensions：委托 base，不做五维结果缓存
}

func (s *OptimizedAIService) EvaluateFiveDimensions(ctx context.Context, req *EvaluateAnswerRequest) (*FiveDimensionResult, error) {
    return s.base.EvaluateFiveDimensions(ctx, req)
}
```

---

### 5.4 依赖注入（Dependency Injection）— 手工组装

**位置**：`internal/app/app.go`

项目**不使用** DI 框架（如 Wire/Dig），而是在 `App.New()` 中按依赖拓扑顺序手动构造，优点是**依赖关系一目了然**，调试链路清晰。

```go
func New(cfg *config.Config) (*App, error) {
    logger, _  := buildLogger(cfg)
    db, _      := setupDatabase(cfg, logger)
    jwtGen     := jwt.NewGenerator(...)
    authSvc    := service.NewAuthService(db, jwtGen, nil)
    provider   := service.GetModelProviderFromConfig(cfg.AI, logger)
    aiSvc      := service.NewDeepSeekServiceWithProvider(provider, logger)
    sttProvider := service.NewSTTProvider(cfg.Speech, logger)
    hub        := ws.NewHub(logger)
    hub.SetServices(aiSvc, sttProvider, ttsProvider)
    // ...
}
```

---

### 5.5 Actor 模型 / 事件循环（Actor Model）

**位置**：`internal/ws/hub.go`

```go
// Hub.Run() 是唯一修改 clients map 的 goroutine
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client.sessionID] = client
            h.mu.Unlock()
        case client := <-h.unregister:
            h.mu.Lock()
            delete(h.clients, client.sessionID)
            h.mu.Unlock()
        }
    }
}
```

注册/注销通过 **channel 消息传递**而非直接加锁，符合 Go 的 "Don't communicate by sharing memory, share memory by communicating" 理念。

---

### 5.6 生产者-消费者 / Worker Pool

**位置**：`service/parallel_executor.go`、`service/gpu_batch_processor.go`

```
生产者（HTTP Handler / WS Client）
        │ 投入任务
        ▼
  ┌─────────────┐     信号量 sem（maxWorkers 个槽）
  │ ParallelExecutor │──────────────────────────────────►
  └─────────────┘          Worker Goroutine × N
        │
        ▼ 收集结果（按原始顺序）
  []TaskResult
```

`BatchProcessor` 在此基础上增加了**时间窗口聚合**：在 `maxWaitTime`（50ms）内收集最多 `maxBatchSize`（8）个请求，一次性提交，减少 GPU/API 调用次数。

---

### 5.7 流水线模式（Pipeline）

**位置**：`service/parallel_executor.go`（`Pipeline` 类型）、`ws/optimized_client.go`（`PipelineEvaluator`）

```
输入 ──► Stage1(STT) ──► Stage2(AI评估) ──► Stage3(TTS) ──► 输出
         ↑                    ↑                  ↑
     各阶段独立 goroutine，通过 channel 传递中间结果
```

---

### 5.8 资源池模式（Object Pool）

**位置**：`service/connection_pool.go`

```
HTTPClientPool
├── pool  sync.Pool         ← 复用 PooledHTTPClient 对象
├── Get() *PooledHTTPClient
└── Put(*PooledHTTPClient)
```

避免频繁创建/销毁 HTTP Client，提升高并发下的资源利用率。

---

### 5.9 中间件链模式（Chain of Responsibility）

**位置**：`internal/middleware/`、`internal/app/app.go`

```go
r.Use(middleware.Recovery(logger))
r.Use(middleware.RequestID())
r.Use(middleware.Logger(logger))
r.Use(observability.MetricsMiddleware())
r.Use(middleware.CORS(cfg))

apiAuth := api.Group("/", middleware.JWTAuth(jwtGen))
apiAuth.Use(middleware.RateLimiter(...))
```

每个中间件是独立函数，通过 `gin.Use` 串联，职责单一，组合灵活。

---

### 5.10 幂等性保护（CAS 原子操作）

**位置**：`internal/ws/client.go`

```go
// 确保 Feedback 只生成一次，无论 HTTP 和 WS 是否并发触发
if !c.feedbackGenerated.CompareAndSwap(false, true) {
    return // 已有其他 goroutine 在处理
}

// 防止评估并发执行
if !c.isEvaluating.CompareAndSwap(false, true) {
    c.sendError("evaluation in progress")
    return
}
defer c.isEvaluating.Store(false)
```

---

### 5.11 仓储风格（Repository-like Service）

**位置**：`service/interview.go`、`service/auth.go` 等

项目没有单独的 `repository` 层，Service 直接使用 GORM 进行数据操作。这是一种**务实的简化**：在业务逻辑不复杂的场景下，避免过度分层。

[src: raw/ingested/3项目/InterviewPro-AI面试/ai_interview-5.-设计模式详解.md]