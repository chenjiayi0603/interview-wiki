# InterviewPro 改用 Eino 编排 Agent

> 分析时间：2026-05-07  
> 仓库地址：https://gitee.com/chenjiayi/interview-quicker  
> 分析目的：了解现有架构和AI逻辑实现，为Eino改造方案做准备

### Eino 关键地址

| 名称 | 地址 | 说明 |
|------|------|------|
| Eino 核心库 | https://github.com/cloudwego/eino | 组件接口定义、Graph/Chain编排、流处理、Callback系统 |
| Eino 扩展库 | https://github.com/cloudwego/eino-ext | ChatModel/Retriever/Embedding等组件实现（OpenAI/DeepSeek/Ollama/Redis...） |
| Eino 示例 | https://github.com/cloudwego/eino-examples | 可运行的示例应用和最佳实践 |
| Eino 官方文档 | https://www.cloudwego.io/docs/eino/ | 用户手册、API文档、快速开始 |
| Eino API文档 | https://pkg.go.dev/github.com/cloudwego/eino | GoDoc API参考 |
| CloudWeGo 官网 | https://www.cloudwego.io | CloudWeGo生态总入口（Kitex/Hertz/Eino） |

#### eino vs eino-ext 的关系

```
eino（核心库）              eino-ext（扩展库）
├── 组件接口定义             ├── ChatModel实现（OpenAI/Claude/DeepSeek/Ollama...）
├── Graph/Chain/Workflow编排 ├── Retriever实现（ElasticSearch/Redis/向量库...）
├── 流处理机制               ├── Embedding实现（OpenAI/Ark...）
├── Callback系统             ├── Tool实现（Google Search/DuckDuckGo...）
└── 类型定义                 ├── Loader/Parser实现（Web/S3/File...）
                             └── 评估器、Prompt优化器等工具
```

类比：eino = Go标准库的接口，eino-ext = 第三方库的具体实现。开发时两个都要import。

#### Eino vs Gin 定位区别

- **Gin** = HTTP框架，管路由、中间件、请求响应
- **Eino** = LLM应用框架，管AI组件编排、流式数据、状态管理
- Eino调模型底层就是HTTP client，不管路由和端口
- 改造后架构：**Gin（接请求）→ Eino（编排AI流程）→ DeepSeek API**

---

## 一、项目概述

**InterviewPro** 是一款 AI 面试练习应用，支持语音对话、实时转写、发音评测和智能评分。项目代码位于 `english-learner/` 子目录下。

| 维度 | 描述 |
|------|------|
| 项目名称 | InterviewPro（仓库名 interview-quicker） |
| 项目定位 | AI 面试练习 App，支持英语口语面试训练 |
| 核心功能 | 语音对话、实时转写、发音评测、AI评分、面试报告 |
| Gitee | https://gitee.com/chenjiayi/interview-quicker |
| 联系邮箱 | jia_yi_chen@126.com |

---

## 二、项目目录结构树

```
interview-quicker/
├── english-learner/                    # ★ 主项目目录
│   ├── frontend/                       # React Native 前端
│   │   ├── src/                        # 源码
│   │   ├── app.json                    # Expo 配置
│   │   ├── package.json                # 依赖管理
│   │   └── ...
│   ├── backend/                        # ★ Go 后端
│   │   ├── cmd/
│   │   │   └── server/
│   │   │       └── main.go             # 应用入口
│   │   ├── internal/
│   │   │   ├── config/                 # 配置管理
│   │   │   │   └── config.go           # Viper 配置加载
│   │   │   ├── handler/                # HTTP 处理器
│   │   │   │   └── interview.go        # 面试相关 API
│   │   │   ├── model/                  # 数据模型
│   │   │   │   └── model.go            # GORM 模型定义
│   │   │   ├── service/                # ★ 业务逻辑层（核心）
│   │   │   │   ├── ai.go               # AI 服务主逻辑
│   │   │   │   ├── model_factory.go    # ★ 模型工厂（DeepSeek + Qwen）
│   │   │   │   ├── aliyun_speech.go    # 阿里云 STT/TTS
│   │   │   │   ├── pronunciation.go    # ★ 发音评测服务
│   │   │   │   ├── bailian_stt.go      # 百炼 STT（WebSocket 实时识别）
│   │   │   │   ├── speech.go           # 语音 Provider 工厂
│   │   │   │   └── ...
│   │   │   ├── ws/                     # ★ WebSocket 模块
│   │   │   │   ├── hub.go              # WebSocket Hub（连接管理）
│   │   │   │   └── client.go           # WebSocket Client（消息处理）
│   │   │   └── middleware/             # 中间件
│   │   │       └── auth.go             # JWT 认证中间件
│   │   ├── config/
│   │   │   └── config.yaml             # 应用配置文件
│   │   ├── .env                        # 环境变量（敏感配置）
│   │   ├── .env.example                # 环境变量模板
│   │   ├── go.mod                      # Go 模块依赖
│   │   └── go.sum                      # 依赖校验
│   ├── logs/                           # 运行日志
│   │   └── backend.log
│   └── start.sh                        # 启动脚本
│
├── 2技术/                              # 技术学习资料
├── storys/                             # 面试故事模板
├── routines/                           # 技术追踪文件
├── todo/                               # 待办事项
└── README.md
```

---

## 三、技术栈总结

### 3.1 后端技术栈

| 技术 | 包名 | 版本 | 用途 |
|------|------|------|------|
| **Gin** | `gin-gonic/gin` | - | HTTP Web 框架 |
| **GORM** | `gorm.io/gorm` | - | ORM 数据库操作 |
| **go-redis** | `go-redis/v9` | v9 | Redis 缓存/会话存储 |
| **gorilla/websocket** | `gorilla/websocket` | - | WebSocket 实时通信 |
| **阿里云 NLS SDK** | `alibabacloud-nls-go-sdk` | v1.1.1 | 语音识别/合成 |
| **Viper** | `spf13/viper` | - | 配置管理（YAML） |
| **Zap** | `go.uber.org/zap` | - | 结构化日志 |
| **JWT** | `golang-jwt/jwt` | - | 用户认证 |
| **net/http** | 标准库 | - | HTTP 客户端（调用 AI API） |

### 3.2 前端技术栈

| 技术 | 用途 |
|------|------|
| **React Native** | 跨平台移动应用框架 |
| **Expo** | React Native 开发工具链 |
| **NativeBase** | UI 组件库 |

### 3.3 外部服务

| 服务 | 用途 | 调用方式 |
|------|------|----------|
| **DeepSeek API** | AI 对话/评估 | HTTP REST API |
| **Qwen Local (llama.cpp)** | 本地 AI 模型 | HTTP REST API（localhost:8080） |
| **阿里云 STT** | 语音转文字 | 阿里云 NLS SDK |
| **阿里云 TTS** | 文字转语音 | 阿里云 NLS SDK |
| **阿里云发音评测** | 发音评分 | HTTP REST API |
| **百炼 STT** | 备用语音识别 | WebSocket 实时流 |
| **MySQL** | 主数据存储 | GORM |
| **Redis** | 缓存/会话 | go-redis |

---

## 四、AI 相关代码清单

### 4.1 核心AI文件

| 文件路径 | 功能描述 | 关键函数/结构 |
|----------|----------|---------------|
| `internal/service/ai.go` | AI 服务主逻辑 | `AIService.EvaluateAnswer()` — 调用模型工厂，评估回答 |
| `internal/service/model_factory.go` | ★ 模型工厂（策略模式） | `ModelProvider` 接口、`DeepSeekModel`、`QwenLocalModel`、`GetModelProvider()` |
| `internal/service/aliyun_speech.go` | 阿里云语音服务 | `AliyunSTT.Transcribe()`、`AliyunTTS.Synthesize()` |
| `internal/service/pronunciation.go` | ★ 发音评测服务 | `PronunciationService.Evaluate()`、`submitTask()`、`getResult()` |
| `internal/service/bailian_stt.go` | 百炼 STT 服务 | WebSocket 实时流式语音识别，支持中/英/日/韩/德/法/俄 |
| `internal/service/speech.go` | 语音 Provider 工厂 | `NewSTTProvider()` — 根据配置返回 STT 实现 |
| `internal/ws/client.go` | ★ WebSocket 客户端 | `handleAudioChunk()`、`evaluateAndRespond()`、`synthesizeAndSend()` |
| `internal/ws/hub.go` | WebSocket Hub | 连接管理、广播消息 |

### 4.2 AI 调用链路详解

#### 面试对话流程

```
用户录音 ──WebSocket──→ client.handleAudioChunk()
                          │
                          ├──→ aliyun_speech.go: STT 转文字
                          │     └── ffmpeg 转码 (webm → wav)
                          │
                          ├──→ pronunciation.go: 发音评测
                          │     └── 阿里云 SpeechEvaluation API
                          │     └── refText = STT 文字
                          │
                          ├──→ ai.go + model_factory.go: AI 评估
                          │     └── DeepSeek 或 Qwen Local
                          │     └── 5维评分: Fluency/Grammar/Vocabulary/Content/Pronunciation
                          │
                          └──→ aliyun_speech.go: TTS 合成回复
                                └── 阿里云 NLS TTS
```

#### AI 模型调用流程

```
ai.go: EvaluateAnswer()
  │
  ├── GetModelProvider()        ← model_factory.go
  │     ├── AI_PROVIDER=deepseek → DeepSeekModel
  │     │     └── HTTP POST → DeepSeek ChatCompletion API
  │     └── AI_PROVIDER=qwen_local → QwenLocalModel
  │           └── HTTP POST → localhost:8080/completion (llama.cpp)
  │
  ├── model.Generate(systemPrompt, userPrompt)
  │     └── 返回 AI 响应文本
  │
  └── 解析评估结果（5维评分 JSON）
```

#### 模型工厂接口设计

```go
type ModelProvider interface {
    Generate(systemPrompt, userPrompt string) (string, error)
    GetName() string
}

// 实现：
type DeepSeekModel struct { apiKey, baseURL string }
type QwenLocalModel struct { apiURL string; temperature float64 }
```

### 4.3 发音评测架构

```go
// 核心流程
PronunciationService.Evaluate(ctx, audioData, refText)
  │
  ├── submitTask()              // 提交评测任务
  │     └── POST https://nls-gateway-inner.aliyuncs.com/rest/v1/general/SpeechEvaluation
  │     └── params: appkey, model_id=en.pred.score, refText
  │
  └── getResult(taskID)         // 轮询获取结果（最多20次，500ms间隔）
        └── 返回 PronunciationResult
              ├── Overall    // 总分
              ├── Pron       // 发音分
              ├── Accuracy   // 准确度
              ├── Integrity  // 完整度
              ├── Fluency    // 流畅度
              └── Words      // 单词级详情
```

### 4.4 WebSocket 通信架构

```
Client (React Native)           Server (Go)
     │                              │
     │── WS Connect ──────────────→│  hub.go: Register
     │                              │
     │── audio_chunk ─────────────→│  client.go: handleAudioChunk()
     │                              │  ├── STT 转文字
     │                              │  ├── 发音评测
     │                              │  └── AI 评估
     │                              │
     │←── evaluation_result ──────│  client.go: evaluateAndRespond()
     │                              │
     │←── audio_response ─────────│  client.go: synthesizeAndSend()
     │                              │  └── TTS 合成语音
     │                              │
     │── heartbeat ──────────────→│  保活
     │←── heartbeat ──────────────│
```

---

## 五、现有架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                       InterviewPro 现有架构                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│   │  iOS     │  │ Android  │  │  H5      │  │  小程序   │             │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│        │            │            │            │                     │
│        └────────────┴────────────┴────────────┘                     │
│                          │ WebSocket + HTTP                         │
│                          ▼                                           │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    API Gateway (Nginx)                      │   │
│   │                    负载均衡 + SSL + 限流                       │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                  Go Backend (Gin + GORM)                     │   │
│   │                                                              │   │
│   │   ┌────────────┐  ┌────────────┐                             │   │
│   │   │ HTTP路由   │  │ WebSocket  │                             │   │
│   │   │ /api/xxx   │  │ /ws/interview │                          │   │
│   │   └────────────┘  └────────────┘                             │   │
│   │                                                              │   │
│   │   ┌─────────────────────────────────────────────────────┐     │   │
│   │   │                  Middleware                          │     │   │
│   │   │  JWT认证 │ 日志(Zap) │ 限流 │ CORS │ 错误处理        │     │   │
│   │   └─────────────────────────────────────────────────────┘     │   │
│   │                                                              │   │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐             │   │
│   │   │ 面试服务    │  │ 用户服务    │  │ 评分服务    │             │   │
│   │   │ Interview  │  │   User     │  │  Scoring   │             │   │
│   │   └────────────┘  └────────────┘  └────────────┘             │   │
│   │                                                              │   │
│   │   ┌────────────┐  ┌────────────┐  ┌────────────┐             │   │
│   │   │ AI服务      │  │ 语音服务    │  │ 发音评测    │             │   │
│   │   │ (Factory)  │  │ (STT/TTS)  │  │ (Pron.)    │             │   │
│   │   └────────────┘  └────────────┘  └────────────┘             │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│   ┌─────────────┬─────────────┬─────────────┬─────────────┐         │
│   │   MySQL     │    Redis    │ DeepSeek API│ 阿里云 NLS   │         │
│   │  (主数据)   │  (会话/缓存) │  (AI对话)   │ (STT/TTS/评测)│        │
│   └─────────────┴─────────────┴─────────────┴─────────────┘         │
│                                                                      │
│   ┌─────────────┐                                                    │
│   │  llama.cpp  │  本地 Qwen 模型 (可选)                              │
│   │ localhost:8080 │                                                  │
│   └─────────────┘                                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### AI 评估数据流

```
                          音频输入
                             │
              ┌──────────────┴──────────────┐
              │                              │
              ▼                              ▼
     ┌──────────────┐              ┌──────────────┐
     │  STT 转文字   │              │  发音评测      │
     │  (阿里云/百炼) │              │  (阿里云 API)  │
     └──────┬───────┘              └──────┬───────┘
            │                              │
            │  refText                     │  Pronunciation Score
            ▼                              │
     ┌──────────────┐                     │
     │  AI 评估      │                     │
     │  (DeepSeek/   │                     │
     │   Qwen Local) │                     │
     └──────┬───────┘                     │
            │                              │
            │  Fluency/Grammar/Vocab/      │
            │  Content Scores              │
            ▼                              ▼
     ┌──────────────────────────────────────┐
     │           合并 5 维评分                 │
     │  Fluency | Grammar | Vocabulary      │
     │  Content | Pronunciation             │
     └──────────────────┬───────────────────┘
                        │
                        ▼
                 WebSocket 推送
                 + TTS 语音回复
```

---

## 六、痛点识别

### 6.1 AI 编排层面

| # | 痛点 | 现状 | 影响 |
|---|------|------|------|
| 1 | **无流式响应** | AI 评估结果一次性返回，非 streaming | 用户体验差，等待时间长，无法逐字显示 |
| 2 | **硬编码编排逻辑** | STT → AI → TTS 的链路在 `client.go` 中硬编码 if-else | 难以扩展新功能（如追问、多轮记忆、RAG） |
| 3 | **模型切换手动管理** | `model_factory.go` 手动实现策略模式 | 无法动态切换、A/B测试、fallback |
| 4 | **无 Agent 架构** | 面试官角色是硬编码 prompt，无状态管理 | 无法实现追问、上下文记忆、多轮对话编排 |
| 5 | **发音评测与 AI 评估割裂** | 两个服务独立调用，结果在 `client.go` 中合并 | 评测维度不一致，缺少综合评估逻辑 |

### 6.2 RAG/知识库层面

| # | 痛点 | 现状 | 影响 |
|---|------|------|------|
| 6 | **无 RAG 检索** | 面试题目完全依赖 AI 生成，无知识库支撑 | 题目质量不稳定，缺乏行业/岗位针对性 |
| 7 | **无向量存储** | 无 embedding 和向量检索能力 | 无法实现语义搜索、相似题推荐 |

### 6.3 工程架构层面

| # | 痛点 | 现状 | 影响 |
|---|------|------|------|
| 8 | **WebSocket 状态管理复杂** | `client.go` 文件庞大（900+ 行），手动管理连接/消息/超时 | 难以维护和测试 |
| 9 | **无重试/容错机制** | AI 调用失败直接返回错误 | 服务可靠性低 |
| 10 | **配置分散** | `.env` + `config.yaml` + 硬编码混合 | 配置管理混乱 |
| 11 | **无观测性** | 只有 Zap 日志，无 metrics/trace | 问题排查困难 |

### 6.4 语音处理层面

| # | 痛点 | 现状 | 影响 |
|---|------|------|------|
| 12 | **音频格式依赖 ffmpeg** | webm → wav 转码依赖系统 ffmpeg | 部署环境要求高 |
| 13 | **STT/TTS Provider 切换硬编码** | `speech.go` 工厂手动判断 | 新增 Provider 需修改工厂代码 |
| 14 | **发音评测轮询机制** | 最多轮询 20 次，500ms 间隔 | 延迟不可控，可能超时 |

---

## 七、Eino 改造的关键切入点

### 7.1 优先级排序

| 优先级 | 改造点 | 对应痛点 | Eino 组件 | 预期收益 |
|--------|--------|----------|-----------|----------|
| **P0** | AI 模型调用统一 | #2, #3 | `ChatModel` | 替代 model_factory，统一接口，自动 streaming |
| **P0** | 流式响应改造 | #1 | `StreamReader` | 逐字输出，体验提升 5x |
| **P1** | 面试流程 Graph 编排 | #2, #4 | `Graph` | 替代硬编码 if-else，可视化、可调试 |
| **P1** | 面试官 Agent | #4 | `Agent (ReAct)` | 多轮追问、上下文记忆、智能评分 |
| **P2** | RAG 知识库集成 | #6, #7 | `Retriever` + `Indexer` | 面试题库检索、行业/岗位针对性 |
| **P2** | 语音服务工具化 | #12, #13 | `Tool` | STT/TTS 作为 Agent Tool，灵活调用 |
| **P3** | Multi-Agent 协作 | #5 | `Multi-Agent` | 面试官 Agent + 评分 Agent + 纠音 Agent |
| **P3** | 可观测性 | #11 | `Callback` | Token 计数、延迟追踪、成本分析 |

### 7.2 具体改造方案

#### 切入点 1：AI 模型层 → Eino ChatModel

**改造前**（model_factory.go）：
```go
type ModelProvider interface {
    Generate(systemPrompt, userPrompt string) (string, error)
    GetName() string
}
// 手动实现 DeepSeekModel, QwenLocalModel
// 手动 HTTP 调用，无 streaming
```

**改造后**（Eino ChatModel）：
```go
import (
    "github.com/cloudwego/eino-ext/components/model/openai"
    "github.com/cloudwego/eino-ext/components/model/ollama"  // 本地模型
)

// Eino ChatModel 统一接口，自动支持 streaming
chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
    APIKey:  os.Getenv("DEEPSEEK_API_KEY"),
    BaseURL: "https://api.deepseek.com",
    Model:   "deepseek-chat",
})

// 本地 Qwen 模型通过 Ollama 兼容接口
localModel, _ := ollama.NewChatModel(ctx, &ollama.ChatModelConfig{
    BaseURL: "http://localhost:8080",
    Model:   "qwen3-4b",
})
```

**收益**：
- ✅ 原生 streaming 支持
- ✅ 统一接口，无需手写 HTTP 调用
- ✅ 自动 fallback、retry、timeout
- ✅ Token 计数、成本追踪

---

#### 切入点 2：面试流程 → Eino Graph 编排

**改造前**（client.go 硬编码）：
```go
// 硬编码的评估流程
func (c *Client) evaluateAndRespond() {
    answer := c.sttResult
    // 1. AI 评估
    evalResult := c.aiService.EvaluateAnswer(question, answer)
    // 2. 发音评测（并发）
    go c.pronunciationSvc.Evaluate(ctx, audioData, answer)
    // 3. 合并评分
    // 4. TTS 合成
    // 5. WebSocket 推送
}
```

**改造后**（Eino Graph）：
```go
import "github.com/cloudwego/eino/compose"

// 定义面试评估 Graph
graph := compose.NewGraph[Input, Output]()

// 添加节点
graph.AddNode("stt", sttNode)           // 语音转文字
graph.AddNode("ai_eval", aiEvalNode)     // AI 评估
graph.AddNode("pron_eval", pronEvalNode) // 发音评测
graph.AddNode("merge", mergeNode)        // 合并评分
graph.AddNode("tts", ttsNode)            // 语音合成

// 编排流程
graph.AddEdge(compose.START, "stt")
graph.AddEdge("stt", "ai_eval")
graph.AddEdge("stt", "pron_eval")        // 并发执行
graph.AddEdge("ai_eval", "merge")
graph.AddEdge("pron_eval", "merge")
graph.AddEdge("merge", "tts")
graph.AddEdge("tts", compose.END)

// 运行
runnable, _ := graph.Compile(ctx)
result, _ := runnable.Invoke(ctx, input)
```

**收益**：
- ✅ 流程可视化、可调试
- ✅ 并发节点自动调度（ai_eval || pron_eval）
- ✅ 状态管理统一
- ✅ 新增节点无需改主流程

---

#### 切入点 3：面试官 Agent

**改造后**：
```go
import (
    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/components/tool"
)

// 面试官 Agent 工具
tools := []tool.BaseTool{
    questionBankTool,     // 题库查询
    pronunciationTool,    // 发音评测
    scoringTool,          // 评分计算
    followUpTool,         // 追问生成
}

// 创建面试官 Agent
agent := adk.NewReActAgent(ctx, &adk.ReActAgentConfig{
    ChatModel: chatModel,
    Tools:     tools,
    SystemPrompt: `你是一位专业的英语面试官...`,
})

// 流式对话
stream, _ := agent.Stream(ctx, messages)
for chunk := range stream {
    // 逐字推送到前端
    c.sendToClient(chunk)
}
```

**收益**：
- ✅ 面试官有记忆，可追问
- ✅ 动态决策：根据回答质量决定追问还是换题
- ✅ 工具调用标准化
- ✅ 流式输出

---

#### 切入点 4：RAG 知识库

**改造后**：
```go
import (
    "github.com/cloudwego/eino/components/retriever"
    "github.com/cloudwego/eino-ext/components/indexer/redis"  // 向量存储
)

// 构建知识库检索器
ret, _ := redis.NewRetriever(ctx, &redis.RetrieverConfig{
    Addrs:    []string{"localhost:6379"},
    IndexKey: "interview_questions",
})

// 在 Graph 中集成 RAG
graph.AddNode("rag_retrieve", func(ctx context.Context, input *Input) (*Retrieved, error) {
    docs, _ := ret.Retrieve(ctx, &retriever.Query{Query: input.Topic})
    return &Retrieved{Documents: docs}, nil
})
graph.AddEdge("rag_retrieve", "ai_eval")  // RAG 检索结果注入 AI 评估
```

**收益**：
- ✅ 面试题目有行业/岗位针对性
- ✅ 语义搜索，推荐相似题
- ✅ 题库可运营、可更新

---

### 7.3 改造路线图

```
Phase 1（1-2周）：基础替换
├── model_factory.go → Eino ChatModel
├── ai.go → 使用 Eino ChatModel 的 Generate/Stream
└── 验证流式输出

Phase 2（2-3周）：流程编排
├── client.go 评估流程 → Eino Graph
├── 发音评测 + AI 评估并发化
├── 合并评分逻辑迁移到 Graph 节点
└── 验证端到端流程

Phase 3（2-3周）：Agent 化
├── 面试官 Agent (ReAct)
├── 工具注册：STT/TTS/评测/题库
├── 多轮对话 + 上下文记忆
└── 验证追问/换题场景

Phase 4（1-2周）：RAG 增强
├── 面试题库向量化
├── Retriever 集成
├── Graph 中添加 RAG 节点
└── 验证行业/岗位针对性题目

Phase 5（持续优化）
├── Multi-Agent（面试官 + 评分 + 纠音）
├── 可观测性（Callback + Metrics）
├── K8s 部署优化
└── 性能调优
```

---

## 八、关键文件对照表

| 现有文件 | 功能 | Eino 改造对应 |
|----------|------|---------------|
| `model_factory.go` | 模型工厂 | → Eino `ChatModel` (openai/ollama) |
| `ai.go` | AI 评估 | → Eino `Graph` 节点 |
| `aliyun_speech.go` | 阿里云语音 | → Eino `Tool` (STT/TTS) |
| `pronunciation.go` | 发音评测 | → Eino `Tool` |
| `bailian_stt.go` | 百炼 STT | → Eino `Tool` |
| `speech.go` | 语音工厂 | → Eino `Tool` 注册机制 |
| `ws/client.go` | WebSocket + 流程编排 | → Eino `Graph` 编排 + WebSocket 推送 |
| `ws/hub.go` | 连接管理 | 保留（Eino 不涉及连接管理） |
| `.env` | 配置 | → Eino `ChatModelConfig` |
| `config.yaml` | 配置 | 保留 + 扩展 Eino 配置项 |

---

## 九、技术优势总结

### 为什么 Eino 适合 InterviewPro

| 维度 | Eino (Go) | LangGraph (Python) |
|------|-----------|---------------------|
| 语言栈 | 纯 Go，与 InterviewPro 一致 | Python，需要跨语言调用 |
| 部署方式 | 单二进制 | Python 运行时或容器 |
| 调用方式 | 直接函数调用 | gRPC/HTTP 跨进程 |
| 状态管理 | 进程内 | 跨进程序列化 |
| 依赖管理 | go.mod | requirements.txt + go.mod |
| 团队技能 | Go 团队直接上手 | 需 Python 技能 |
| 容器数量 | 1 | 2+（Go 服务 + Python 服务） |
| 网络延迟 | 进程内调用 ~0ms | 跨进程 ~1-10ms |

### 面试加分项

> "我用字节 Eino 框架重构了 InterviewPro 的 AI 编排层，用 Graph 编排替代了原来的硬编码 if-else 链路，实现了面试官 Agent + 评分 Agent 的 Multi-Agent 协作。支持流式响应、RAG 知识检索，部署在 K8s 上弹性扩缩容。"

这句话比 "我用了 LangChain + Python" 有差异化优势，因为 Go + Eino 在 AI 工程化方向是独特路线。

---

*报告完成。下一步建议：先执行 Phase 1（ChatModel 替换 model_factory），这是最小改动、最大收益的切入点。*
