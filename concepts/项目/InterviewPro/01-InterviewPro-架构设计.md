# InterviewPro 架构设计

> Go 后端分层架构、核心类组织、设计模式、并发模型。

---

## 一、整体架构

```
┌────────────────────────────────────────────┐
│              Client (Browser/App)           │
└────────┬──────────────┬─────────────────────┘
         │ HTTP REST     │ WebSocket
         ▼               ▼
┌────────────────────────────────────────────┐
│        Gin Router + Middleware Chain         │
│  (Recovery · CORS · RequestID · JWT)        │
└────────┬──────────────┬─────────────────────┘
         │               │
         ▼               ▼
┌──────────────┐  ┌──────────────────────────┐
│ HTTP Handlers│  │   WebSocket Hub + Client  │
│ (Auth/       │  │  (ReadPump/WritePump/     │
│  Interview/  │  │   saveLoop/evaluateResp)  │
│  Question)   │  └──────────┬───────────────┘
└──────┬───────┘             │
       │                     │
       ▼                     ▼
┌────────────────────────────────────────────┐
│              Service Layer                   │
│  AuthSvc · InterviewSvc · AIService ·       │
│  STTProvider · TTSProvider · QuestionPool   │
└────────────────────┬───────────────────────┘
                     │
          ┌──────────┼──────────────┐
          ▼          ▼              ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ GORM/DB  │ │ AI Model │ │ 阿里云   │
   │ (SQLite/ │ │ Provider │ │ STT/TTS  │
   │ Postgres)│ │(DeepSeek/│ │          │
   └──────────┘ │ Qwen)    │ └──────────┘
                └──────────┘
```

## 二、分层架构

| 层次 | 核心职责 | 依赖方向 |
|------|----------|---------|
| **入口层** `cmd/server` | 读取配置，调用 App.New | `internal/app`, `internal/config` |
| **组合根** `internal/app` | 依赖装配、路由注册、优雅关闭 | 所有 internal + pkg |
| **Handler 层** `internal/handler` | HTTP 绑定/校验/委托 Service | Service, DTO, pkg |
| **WebSocket 层** `internal/ws` | 实时双向通信、会话状态机 | Service, Model |
| **Service 层** `internal/service` | 业务逻辑、外部 API 集成 | Model, pkg |
| **Model 层** `internal/model` | GORM 实体定义 | `gorm.io/gorm` |
| **公共包** `pkg/` | 通用工具（JWT/错误/校验/指标） | 仅标准库+第三方 |

**依赖方向严格单向**：`pkg/` 不可引用 `internal/`，Service 不可引用 Handler。

### 2.1 组合根 — App

```go
func New(cfg *config.Config) (*App, error) {
    logger, _  := buildLogger(cfg)
    db, _      := setupDatabase(cfg, logger)
    jwtGen     := jwt.NewGenerator(...)
    authSvc    := service.NewAuthService(db, jwtGen, nil)
    provider   := service.GetModelProviderFromConfig(cfg.AI, logger)
    aiSvc      := service.NewDeepSeekServiceWithProvider(provider, logger)
    hub        := ws.NewHub(logger)
    hub.SetServices(aiSvc, sttProvider, ttsProvider, ...)
    // ...
}
```

## 三、WebSocket 层

### 3.1 Hub — Actor 模型

```go
type Hub struct {
    clients    map[string]*Client  // RWMutex 保护
    register   chan *Client        // 注册通道
    unregister chan *Client        // 注销通道
    shutdown   atomic.Bool
    wg         sync.WaitGroup
}
```

注册/注销通过 channel 串行处理，符合 **"Don't communicate by sharing memory"** 理念。

### 3.2 Client — 单会话状态机

每个 WebSocket 连接对应 4+N 个 Goroutine：

| Goroutine | 职责 |
|-----------|------|
| **ReadPump** | 读取 WS 消息，分发事件 |
| **WritePump** | 从 send channel 写入 WS |
| **saveLoop** | 异步 DB 写入（防并发 INSERT） |
| **evaluateAndRespond** | 临时：AI 评估 + TTS 合成 |

**幂等性保护**：
```go
if !c.feedbackGenerated.CompareAndSwap(false, true) {
    return // 已有其他 goroutine 在处理
}
```

## 四、Service 层设计模式

### 4.1 工厂方法 — AI 模型切换

```go
func GetModelProviderFromConfig(cfg AIConfig) ModelProvider {
    switch cfg.Provider {
    case "qwen_local":
        return NewQwenLocalModel(...)
    default:
        return NewDeepSeekModel(...)
    }
}
```

### 4.2 策略模式 — 语音/模型 Provider

```
AIService (interface)
├── DeepSeekService (主要实现)
└── OptimizedAIService (装饰器包装)

ModelProvider (interface)
├── DeepSeekModel   ← 云端 API
└── QwenLocalModel  ← 本地 llama.cpp

STTProvider (interface)
├── AliyunSTT
├── BailianSTT
└── WhisperSTT
```

### 4.3 信号量限流

```go
// 控制并发 AI 请求，防止打爆 API
sem := make(chan struct{}, maxConcurrent)  // 云端默认 10
sem <- struct{}{}
defer func() { <-sem }()
```

### 4.4 缓存 — 反馈模板缓存

```go
type TemplateFeedbackCache struct {
    cache  map[string]*CacheEntry  // TTL=24h, 容量=500
}
```

## 五、数据流

### 5.1 实时面试流程

```
WS /ws/interview/:sessionId
  → JWT 验证 → NewClient → ReadPump + WritePump + saveLoop

音频帧 "audio_chunk"：
  → ReadPump → handleAudioChunk（积累缓冲）

音频结束 "audio_end"：
  → go evaluateAndRespond()
     ├── STT.Transcribe(audio)       → 文本
     ├── AI.EvaluateAnswer(question)  → 评分
     ├── AI.GenerateQuestion(context) → 下一题
     ├── TTS.Synthesize(nextQuestion) → 音频
     └── send channel ← 推送 WSMessage

DB 写入（异步）：
  → saveCh ← saveRequest
  → saveLoop goroutine → GORM Insert
```

### 5.2 AI 调用链

```
DeepSeekService.EvaluateAnswer
  → 构建 systemPrompt
  → ModelProvider.GenerateWithMaxTokens（信号量限流）
    → HTTP POST /v1/chat/completions（指数退避重试）
  → extractJSONFromMarkdown → json.Unmarshal → EvaluationResult
```
