# InterviewPro 核心技术选型与落地

> 更新时间：2026-06-06  
> 对应代码：`/home/tommychen/english-learner/backend`

---

## 一、向量检索系统

### 1.1 技术选型：Qdrant / pgvector

当前系统支持两种向量数据库后端，通过 `VECTOR_PROVIDER` 环境变量切换：

| 后端 | 配置值 | 优势 | 适用场景 |
|------|--------|------|----------|
| **pgvector** | `pgvector`（默认） | 无需额外组件，PostgreSQL 原生扩展 | 开发环境、简单部署 |
| **Qdrant** | `qdrant` | 专用向量数据库，性能更高 | 生产环境、大规模检索 |

**为什么不用 Elasticsearch？**
- Elasticsearch 的向量检索（dense_vector）性能不如专用向量库
- ES 资源消耗大，不适合单机/小集群部署
- Qdrant 原生支持 gRPC，延迟更低
- pgvector 对已有 PostgreSQL 用户零成本接入

### 1.2 向量化服务

| 配置 | 默认值 | 说明 |
|------|--------|------|
| Provider | SiliconFlow / Ollama | 远程 API 或本地模型 |
| 模型 | BAAI/bge-m3 | 中英双语，1024 维 |
| 维度 | 1024 | bge-m3 固定维度 |
| 超时 | 30s | 请求超时时间 |
| 并发 | 2 | 并发 embedding 上限 |

### 1.3 混合检索（Hybrid Search）

```go
// internal/service/hybrid_search.go
query → Embedding(bge-m3) → 向量检索 (ANN)
     → Keyword/BM25    → 全文检索
     → RRF Fusion      → 精排（可选 reranker）
     → Top-K 结果
```

**RRF（Reciprocal Rank Fusion）** 混合检索策略：
- 向量检索和全文检索各自返回 Top-N
- RRF 公式融合排序：`score = 1/(k + rank)`，默认 k=60
- 融合结果可选 cross-encoder 精排

### 1.4 精排（Reranker）

| 组件 | 模型 | 说明 |
|------|------|------|
| SiliconFlow Reranker | BAAI/bge-reranker-v2-m3 | cross-encoder 精排，提升 P@10 |
| NoopReranker | - | 关闭精排时直接按 RRF 分数排序 |

通过 `EMBEDDING_RERANK_ENABLED` 控制是否启用精排（默认关闭）。

---

## 二、AI 模型层

### 2.1 技术选型：DeepSeek + Qwen Local

| Provider | 配置 | 适用场景 |
|----------|------|----------|
| **DeepSeek**（默认） | `AI_PROVIDER=deepseek` | 开发、需要高质量评分 |
| **DeepSeek Eino** | `AI_PROVIDER=deepseek_eino` | 需要流式输出时 |
| **Qwen Local** | `AI_PROVIDER=qwen_local` | 生产环境，通过 Ollama 部署本地模型 |

### 2.2 模型工厂

```go
// internal/service/ai/factory.go
func GetModelProviderFromConfig(cfg AIConfig, logger *zap.Logger) ModelProvider
```

统一接口：
```go
type ModelProvider interface {
    Generate(ctx, systemPrompt, userPrompt string) (string, error)
    GenerateWithMessages(ctx, []Message) (string, error)
    GetName() string
}
```

### 2.3 Eino ChatModel

当使用 `deepseek_eino` Provider 时，通过 Eino 的 ChatModel 接口调用：

```go
import "github.com/cloudwego/eino-ext/components/model/deepseek"

chatModel, _ := deepseek.NewChatModel(ctx, &deepseek.ChatModelConfig{
    APIKey:  apiKey,
    BaseURL: baseURL,
    Model:   model,
})
```

Eino 封装提供原生 streaming 支持、自动重试、Token 计数等能力。

### 2.4 AIService 业务接口

```go
// internal/service/ai/llm.go
type AIService interface {
    EvaluateAnswer(ctx, question, answer, scenarioType, difficulty string) (*FiveDimensionResult, error)
    GenerateQuestion(ctx, *QuestionGenInput) (string, error)
    GenerateSummary(ctx, transcript string) (string, error)
    GenerateFeedback(ctx, string) (string, error)
}
```

---

## 三、语音服务

### 3.1 STT（语音转文字）

支持 **8 个 Provider**，通过 Fallback 链实现高可用：

```go
STT_PROVIDER=bailian,whisper,xunfei  // 逗号分隔，按序尝试
```

| Provider | 引擎 | 接入方式 | 延迟 | 成本 |
|----------|------|----------|------|------|
| **bailian** | paraformer-realtime-v2 | HTTP | 低 | 按量计费 |
| **qwen3_asr** | Qwen3-ASR-Flash | HTTP | 低 | 按量计费 |
| **whisper** | OpenAI Whisper API | HTTP | 中 | 按量计费 |
| **volcengine** | 火山引擎 ASR | WebSocket | 低 | 按量计费 |
| **aliyun** | 阿里云 NLS SDK | SDK | 中 | 按量计费 |
| **azure** | Azure Speech | HTTP | 中 | 按量计费 |
| **sensevoice** | SenseVoiceSmall | HTTP | 中 | 按量计费 |
| **xunfei** | 讯飞 IAT | HTTP | 中 | 按量计费 |

### 3.2 TTS（文字转语音）

同样支持 Fallback 链：

| Provider | 引擎 | 音质 | 延迟 | 成本 |
|----------|------|------|------|------|
| **aliyun** | 阿里云 NLS TTS | 中 | 低 | 按量计费 |
| **elevenlabs** | ElevenLabs API | 高 | 中 | 按量计费 |
| **volcengine** | 火山引擎 TTS (seed-tts-1.0) | 高 | 低 | 按量计费 |
| **siliconflow** | CosyVoice2 | 高 | 中 | 按量计费 |
| **azure** | Azure Speech | 高 | 低 | 按量计费 |
| **edge** | Edge TTS | 中 | 低 | 免费 |
| **xunfei** | 讯飞 TTS | 中 | 低 | 按量计费 |

### 3.3 发音评测

双后端可选：

| 后端 | 配置值 | 说明 |
|------|--------|------|
| **xunfei_ise**（默认） | `xunfei_ise` | 科大讯飞 ISE，标准发音评测 |
| **aliyun_asr** | `aliyun_asr` | 百炼 ASR 词匹配，基于识别结果评分 |

通过 `PRONUNCIATION_DEFAULT_MODEL` 或 `PRONUNCIATION_MODEL` 环境变量选择。

### 3.4 健康监控

`SpeechHealth` 模块追踪所有 STT/TTS 调用：

| 指标 | 端点 |
|------|------|
| STT 成功率 | `/api/speech/health` |
| TTS 成功率 | `/api/speech/health` |
| 各 Provider 状态 | `/api/speech/health` |

---

## 四、Eino Agent 编排

### 4.1 技术选型：CloudWeGo Eino

**为什么选 Eino 而非 LangGraph：**

| 维度 | Eino (Go) | LangGraph (Python) |
|------|-----------|---------------------|
| 语言栈 | 纯 Go，与后端一致 | Python，需额外运行时 |
| 部署 | 单二进制 | Python 运行时或容器 |
| 性能 | 进程内调用 ~0ms | 跨进程 ~1-10ms |
| 学习成本 | Go 团队零门槛 | 需 Python 技能 |
| 社区 | CloudWeGo 生态 | 成熟但 Python 生态 |

### 4.2 核心组件

| 组件 | 用法 |
|------|------|
| `compose.Parallel` | 三路并行：评分 / 出题 / 发音评测 |
| `compose.Chain` | 串联 Parallel → Merge |
| `model.ChatModel` | 统一 AI 模型接口 |
| `compose.Runnable` | Graph 编译后执行接口 |

### 4.3 执行流程

```go
// 图定义（graph.go）
parallel := compose.NewParallel()
parallel.AddLambda("scorer", scorerLambda)
parallel.AddLambda("question", questionLambda)
parallel.AddLambda("pronunciation", pronLambda)

chain := compose.NewChain[GraphInput, *MergedOutput]()
chain.AppendParallel(parallel)
chain.AppendLambda(mergeLambda)

runnable, _ := chain.Compile(ctx)
result, _ := runnable.Invoke(ctx, input)
```

### 4.4 性能数据

| 指标 | 旧架构（串行） | 当前架构（并行） |
|------|---------------|-----------------|
| 单次评估延迟 | ~18s | ~8s（取三个分支最大值） |
| 题库命中时出题 | N/A | ~0ms（跳过 LLM 调用） |
| 失败率 | 单点故障 | 分支隔离，非致命错误降级 |

---

## 五、数据库选型

### 5.1 主数据库

| 环境 | 数据库 | 驱动 | 说明 |
|------|--------|------|------|
| 生产 | PostgreSQL 16 | `gorm.io/driver/postgres` | 高可用、性能好 |
| 开发 | SQLite | `gorm.io/driver/sqlite` | 零配置、快速迭代 |

### 5.2 缓存

- **Redis 7**：会话缓存、JWT 黑名单、临时数据
- **go-redis/v9**：Redis 客户端

### 5.3 向量数据库

- **Qdrant v1.16.0**：生产环境向量检索（gRPC 接口）
- **pgvector**：开发环境零成本向量存储

---

## 六、可观测性

| 维度 | 工具 | 端点/方法 |
|------|------|-----------|
| 结构化日志 | Zap | 文件/stdout，debug/release 模式 |
| 性能指标 | Prometheus | `/metrics` |
| 健康检查 | Gin handler | `/healthz`（K8s 探针） |
| 语音健康 | SpeechHealth | `/api/speech/health` |
| AI 调用追踪 | TraceService | 存储到数据库 |

---

## 七、部署架构

### 7.1 Docker Compose 全栈编排

```yaml
services:
  postgres:    # PostgreSQL 16 数据库
  redis:       # Redis 7 缓存
  qdrant:      # Qdrant v1.16.0 向量数据库
  api:         # Go 后端（多阶段构建）
```

### 7.2 多阶段构建

```
Stage 1: golang:1.25-alpine → 编译 Go 二进制
Stage 2: node:22-alpine → 构建 admin-web 前端
Stage 3: alpine:3.20 → 合并运行（含 edge-tts python 依赖）
```

### 7.3 生产环境关键配置

```
AI_PROVIDER=qwen_local
QWEN_LOCAL_MODEL=qwen3-scoring
QWEN_TEMPERATURE=0.1
DATABASE_DRIVER=postgres
STT_PROVIDER=bailian
TTS_PROVIDER=elevenlabs
VECTOR_PROVIDER=qdrant
EMBEDDING_BASE_URL=http://host.docker.internal:11434/api
```
