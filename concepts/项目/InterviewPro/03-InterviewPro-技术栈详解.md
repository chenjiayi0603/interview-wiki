# InterviewPro 技术栈详解

> 更新时间：2026-06-06  
> 代码路径：`/home/tommychen/english-learner/backend`

---

## 一、后端技术栈总览

| 分类 | 技术 | 用途 | 版本 |
|------|------|------|------|
| Web 框架 | Gin | HTTP 路由 + 中间件 | v1.12 |
| AI 编排 | CloudWeGo Eino | LLM Agent + Graph 编排 | v0.8.13 |
| 数据库 ORM | GORM | PostgreSQL / SQLite ORM | v1.31 |
| 缓存 | go-redis | Redis 客户端 | v9 |
| 实时通信 | gorilla/websocket | WebSocket | v1.5 |
| 配置管理 | Viper | YAML + 环境变量 | v2 |
| 日志 | Zap | 结构化日志 | v1.27 |
| 序列化 | Sonic | 高性能 JSON | v1.15 |
| JWT | golang-jwt | 令牌认证 | v5 |
| 参数校验 | go-playground/validator | 请求校验 | v10 |
| 向量数据库 | Qdrant go-client | 向量检索 | v1.17 |
| 指标 | Prometheus | 性能监控 | v1.23 |
| 语音 SDK | 阿里云 NLS Go SDK | STT/TTS | v1.1.1 |
| 雪花 ID | 自实现 | 分布式 ID | - |

---

## 二、Gin Web 框架

### 2.1 路由注册

```go
// internal/app/app.go
func setupRoutes(r *gin.Engine) {
    r.GET("/healthz", handler.HealthCheck)
    r.GET("/ws/interview", wsHandler)
    
    api := r.Group("/api")
    api.Use(middleware.Auth())
    {
        api.POST("/interview/session", interviewHandler.CreateSession)
        api.POST("/interview/answer", interviewHandler.SubmitAnswer)
        // ...
    }
    
    admin := r.Group("/api/admin")
    admin.Use(middleware.AdminAuth())
    {
        admin.GET("/questions", questionAdminHandler.List)
        admin.PUT("/prompts", promptAdminHandler.Update)
        // ...
    }
}
```

### 2.2 中间件链

| 中间件 | 文件 | 功能 |
|--------|------|------|
| JWT Auth | `middleware/auth.go` | Bearer Token 验证 |
| Admin Auth | `middleware/admin_auth.go` | 管理端独立 JWT |
| Logger | `middleware/logger.go` | 请求日志（方法/路径/状态码/延迟） |
| CORS | Gin 内置 | 跨域配置 |

---

## 三、CloudWeGo Eino

### 3.1 核心概念

| 概念 | 说明 |
|------|------|
| ChatModel | LLM 统一接口，支持 streaming |
| compose.Chain | 串行编排 |
| compose.Parallel | 并行编排 |
| compose.Runnable | 编译后的可执行 Graph |
| Lambda | 函数式节点 |
| Message | 多轮对话消息（System/User/Assistant） |

### 3.2 在本项目中的使用

```go
// 三路并行
parallel := compose.NewParallel()
parallel.AddLambda("scorer", scorerLambda)
parallel.AddLambda("question", questionLambda)
parallel.AddLambda("pronunciation", pronLambda)

chain := compose.NewChain[GraphInput, *MergedOutput]()
chain.AppendParallel(parallel)
chain.AppendLambda(mergeLambda)
```

### 3.3 为什么用 Lambda 而非 Graph

当前实现使用 `compose.Parallel` + Lambda 而不是 `compose.Graph`，原因：

| 方式 | 适用场景 | 本项目选择 |
|------|----------|-----------|
| Graph | 复杂拓扑（分支/合并/循环） | 当前三路并行结构固定，无需动态拓扑 |
| Parallel | 固定并行分支 | 评分/出题/发音评测互不依赖，天然并行 |

### 3.4 Eino 扩展

```go
import "github.com/cloudwego/eino-ext/components/model/deepseek"
```

当前主要使用 `ChatModel` 接口，未来可扩展：
- `Retriever`：对接 Qdrant 向量检索
- `Tool`：STT/TTS 作为 Agent Tool
- `Callback`：Token 计数、延迟追踪

---

## 四、AI 模型服务

### 4.1 模型提供者

| Provider | 实现文件 | 底层调用 |
|----------|----------|----------|
| DeepSeek | `deepseek_model.go` | HTTP POST → api.deepseek.com |
| DeepSeek Eino | `eino_model.go` | Eino ChatModel → deepseek |
| Qwen Local | `qwen_model.go` | HTTP POST → Ollama API |

### 4.2 统一接口

```go
type ModelProvider interface {
    Generate(ctx, systemPrompt, userPrompt string) (string, error)
    GenerateWithMessages(ctx, []Message) (string, error)
    GetName() string
}
```

### 4.3 AI Service 接口

```go
type AIService interface {
    EvaluateAnswer(ctx, question, answer, scenarioType, difficulty string) (*FiveDimensionResult, error)
    GenerateQuestion(ctx, *QuestionGenInput) (string, error)
    GenerateSummary(ctx, transcript string) (string, error)
    GenerateFeedback(ctx, string) (string, error)
}
```

### 4.4 评分输出结构

```go
type FiveDimensionResult struct {
    OverallScore    float64
    Dimensions      Dimensions     // Fluency/Grammar/Vocabulary/Content/Pronunciation
    OverallFeedback OverallFeedback // Strengths/AreasForImprovement/SampleImprovedAnswer
}
```

---

## 五、语音服务

### 5.1 架构设计

```
STT/TTS Provider 使用工厂 + Fallback 链模式：

factory.go
  ├── NewSTTProvider("bailian,whisper", cfg) → FallbackSTTProvider
  │     ├── BailianSTT (首选)
  │     └── WhisperSTT (备选)
  │
  └── NewTTSProvider("elevenlabs,edge", cfg) → FallbackTTSProvider
        ├── ElevenLabsTTS (首选)
        └── EdgeTTS (备选)
```

### 5.2 STT Provider 接口

```go
type STTProvider interface {
    Transcribe(ctx, audio []byte, format string) (*Transcription, error)
    Prewarm(ctx) error
    Engine() string
}

type StreamingSTTProvider interface {
    OpenStream(ctx) (STTStream, error)
}

type STTStream interface {
    Feed(pcm []byte) error
    Finish() (*Transcription, error)
    Close() error
}
```

### 5.3 TTS Provider 接口

```go
type TTSProvider interface {
    Synthesize(ctx, text, voice string) ([]byte, error)
    Engine() string
}
```

### 5.4 发音评测

双后端：
- **xunfei_ise**：科大讯飞 ISE，标准发音评测 API
- **aliyun_asr**：百炼 ASR 词匹配，基于 ASR 对齐评分

```go
type PronunciationManager struct {
    backends map[string]PronunciationBackend
}

func (m *PronunciationManager) Evaluate(ctx, audio []byte, refText string) (*PronResult, error)
```

---

## 六、向量检索

### 6.1 双后端架构

```go
// vector_factory.go
type VectorProvider interface {
    SearchInterviewRecords(ctx, userID string, limit int) ([]RecordResult, error)
    UpsertInterviewRecord(ctx, id string, vector []float32, payload *RecordPayload) error
}
```

- **pgvector**：PostgreSQL 扩展，零额外组件
- **Qdrant**：专用向量数据库，gRPC 高性能接口

### 6.2 Embedding 服务

```go
type EmbeddingService struct {
    provider string  // "siliconflow" | "ollama"
    model    string  // "BAAI/bge-m3"
    dims     int     // 1024
}
```

### 6.3 混合检索

```go
// hybrid_search.go
func HybridSearch(ctx, query string) []SearchResult {
    denseResults := VectorSearch(query)       // ANN 向量检索
    sparseResults := KeywordSearch(query)      // BM25 全文检索
    return RRFMerge(denseResults, sparseResults) // RRF 融合
}
```

---

## 七、WebSocket 实时通信

### 7.1 分层架构

| 层 | 文件 | 职责 |
|----|------|------|
| 传输层 | `client.go` | 连接生命周期、消息收发、序列化 |
| 业务层 | `session_flow.go` | 面试流程控制 |
| 状态层 | `interview_session.go` | 会话状态存储 |
| 管理层 | `hub.go` | 注册/注销/广播 |

### 7.2 消息协议

```json
// 客户端 → 服务端
{"event": "start_session", "data": {"scenarioType": "behavior", "difficulty": "mid"}}
{"event": "text_answer", "data": {"text": "My experience..."}}
{"event": "audio_chunk", "data": {"audio": "<base64>"}}

// 服务端 → 客户端
{"event": "evaluation_result", "data": {"scores": {...}, "feedback": {...}}}
{"event": "audio_response", "data": {"audio": "<base64>"}}
{"event": "session_ended", "data": {"report": {...}}}
```

### 7.3 面试会话流程

```
start_session → 预热 STT → 首题生成 → TTS 合成 → 推送
      ↓
text_answer → GraphRunner.Run() → 评分 + 出题 + 发音评测
      ↓
evaluation_result → TTS 合成 → audio_response
      ↓
end_session → 保存记录 → LTM Store
```

---

## 八、数据库设计

### 8.1 核心表

| 表 | 模型 | 用途 |
|----|------|------|
| users | `model.User` | 用户信息 |
| interview_sessions | `model.InterviewSession` | 面试会话 |
| interview_messages | `model.InterviewMessage` | 对话消息 |
| questions | `model.Question` | 题库 |
| scenarios | `model.Scenario` | 面试场景配置 |
| positions | `model.Position` | 岗位定义 |
| billing_records | `model.BillingRecord` | 计费记录 |
| api_providers | `model.APIProvider` | API Provider 配置 |
| traces | `model.Trace` | AI 调用追踪 |
| rag_traces | `model.RAGTrace` | RAG 检索追踪 |

### 8.2 数据库迁移

通过 GORM AutoMigrate 自动管理表结构：

```go
db.AutoMigrate(
    &model.User{},
    &model.InterviewSession{},
    &model.InterviewMessage{},
    &model.Question{},
    // ...
)
```

---

## 九、认证与安全

### 9.1 认证方式

| 方式 | 实现 | 用途 |
|------|------|------|
| JWT | `pkg/jwt/jwt.go` | API 认证 |
| 短信验证码 | `internal/sms/` | 手机号登录 |
| GitHub OAuth | `internal/handler/auth_github.go` | 第三方登录 |
| 管理端 JWT | `middleware/admin_auth.go` | 管理后台 |

### 9.2 JWT 令牌

```go
type Generator struct {
    accessSecret  string
    refreshSecret string
    accessExpiry  time.Duration  // 15 分钟
    refreshExpiry time.Duration  // 7 天
}
```

---

## 十、可观测性

| 维度 | 实现 | 端点 |
|------|------|------|
| 结构化日志 | Zap（debug/release 模式） | stdout / 文件 |
| 业务指标 | Prometheus + 自定义 metrics | `/metrics` |
| 健康检查 | Gin handler | `/healthz` |
| 语音健康 | SpeechHealth 聚合 | `/api/speech/health` |
| AI 追踪 | TraceService（DB 存储） | 管理端查询 |

---

## 十一、部署

### 11.1 Docker 多阶段构建

```dockerfile
FROM golang:1.25-alpine AS builder
  → go build -o /server cmd/server/main.go

FROM node:22-alpine AS web-builder
  → npm run build (admin-web)

FROM alpine:3.20
  COPY --from=builder /server .
  COPY --from=web-builder /dist ./web/admin/
  CMD ["./server"]
```

### 11.2 Docker Compose

```yaml
services:
  postgres:   # PostgreSQL 16
  redis:      # Redis 7
  qdrant:     # Qdrant v1.16.0
  api:        # Go 后端 (port 8080)
```

### 11.3 健康探针

```yaml
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/healthz || exit 1
```

---

## 十二、关键技术决策

| 决策点 | 选择 | 替代方案 | 原因 |
|--------|------|----------|------|
| AI 编排 | Eino (Go) | LangGraph (Python) | 统一 Go 技术栈，避免跨语言调用 |
| 向量数据库 | pgvector / Qdrant | Elasticsearch | 性能更好、资源消耗更小 |
| LLM 接口 | 统一 ModelProvider | 直接 HTTP 调用 | 方便切换 Provider、统一监控 |
| 语音 Provider | Fallback 链 | 单一 Provider | 高可用，任一 Provider 故障自动切换 |
| WS 分层 | client + session_flow | 单文件 | 职责分离，可测试性提升 |
| 评分出题 | 双独立 Agent | 单一 Agent | 专注性更好，独立测试/调优 |
| 评分计算 | 算术均值替代 LLM 自返 | LLM overall_score | 自返值偏低，算术均值与会话末汇总一致 |
| 参考回答 | 题库预存，LLM 不生成 | LLM 生成 | 避免 LLM 幻觉，保证答案质量 |
