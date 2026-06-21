# InterviewPro 项目总览

> 跨平台 AI 面试练习 App — 技术详解与架构设计全貌。
> 后端 Go + React Native 前端，集成 DeepSeek AI + 阿里云语音。

---

## 一、项目速览

| 维度 | 信息 |
|------|------|
| **产品定位** | AI 面试练习 App（iOS / Android / H5） |
| **技术栈** | Go + Gin + WebSocket + React Native + DeepSeek + 阿里云语音 |
| **部署** | Docker + K3s 单节点 |
| **单机并发** | 10,000 WebSocket 连接（2 核 4GB） |
| **端到端延迟** | ~510ms（语音识别）、~4.5s（问题生成 + TTS） |
| **开发周期** | 独立全栈开发 |

## 二、文件地图

| 序号 | 文件 | 内容 |
|------|------|------|
| 01 | **01-InterviewPro-代码分析.md** | 后端代码结构分析 |
| 02 | **02-InterviewPro-四Agent提示词系统设计.md** | 四 Agent 提示词系统设计 |
| 03 | **03-InterviewPro-技术栈详解.md** | 技术栈与架构总览 |
| 04 | **04-InterviewPro-核心技术选型与落地.md** | 核心选型与实现 |
| 05 | **05-InterviewPro-英语面试分层反馈技术实现方案.md** | 英语面试分层反馈方案 |
| 06 | **06-InterviewPro-架构设计.md** | 后端分层架构、设计模式、并发模型 |
| 07 | **07-InterviewPro-面试考点速查.md** | STAR 故事、性能数据、高频 Q&A |
| 08 | **08-InterviewPro-前后端协议.md** | REST + WebSocket 完整协议定义 |
| 09 | **09-InterviewPro-登录认证流程.md** | JWT 双 Token 认证、注册登录、会话恢复 |
| 10 | **10-InterviewPro-前端架构详解.md** | 前端组件架构、状态管理、服务层、类型系统 |
| 11 | **11-InterviewPro-语音录制系统设计.md** | 音频处理链、预暖机制、E2E 测试策略 |
| 12 | **12-InterviewPro-评分模型选型分析.md** | Qwen 4B vs DeepSeek 准确率对比 |
| 13 | **13-InterviewPro-LLM服务参数配置.md** | llama.cpp 参数调优与推荐值 |
| 14 | **14-InterviewPro-Ollama模型生命周期.md** | K8s 部署、GPU 显存管理、模型清单 |
| 15 | **15-InterviewPro-出题流程优化.md** | 混合检索、去重、AI 补充出题 |
| 16 | **16-InterviewPro-QA测试结果记录.md** | 全功能测试、Bug 修复记录 |
| 17 | **17-InterviewPro-并发模型分析.md** | Go/Frontend 线程模型、Goroutine 深度分析 |
| 18 | **18-InterviewPro-爬虫系统设计.md** (→ `../8-InterviewPro爬虫系统/`) | 爬虫架构、反爬策略、数据流水线（Python + Go 后端 API） |

## 三、核心特性

| 特性 | 描述 |
|------|------|
| **流式实时对话** | WebSocket 实时双向通信，音频帧流式上传 + AI 流式输出 |
| **双 AI 模型切换** | DeepSeek（云端）/ Qwen（本地）双模型，策略模式+工厂方法 |
| **语音服务集成** | 阿里云 STT/TTS，支持实时流识别与语音合成 |
| **可观测性** | Prometheus 埋点，关键指标（并发数、延迟、错误率）可视化 |
| **纯净依赖管理** | 手工依赖注入（无 IoC 框架），分层架构严格单向依赖 |
| **幂等保护** | CAS 原子操作防止重复 AI 评估，saveLoop 串行化 DB 写入 |

## 四、分层架构

```
┌──────────────────────────────────────────────┐
│              Client (Browser/App)              │
└────────┬──────────────┬────────────────────────┘
         │ HTTP REST     │ WebSocket
         ▼               ▼
┌──────────────────────────────────────────────┐
│        Gin Router + Middleware Chain           │
│  (Recovery · CORS · RequestID · JWT)          │
└────────┬──────────────┬────────────────────────┘
         │               │
         ▼               ▼
┌──────────────┐  ┌────────────────────────────┐
│ HTTP Handlers│  │   WebSocket Hub + Client    │
│ (Auth/       │  │  (ReadPump/WritePump/       │
│  Interview/  │  │   saveLoop/evaluateResp)    │
│  Question)   │  └──────────┬─────────────────┘
└──────┬───────┘             │
       │                     │
       ▼                     ▼
┌──────────────────────────────────────────────┐
│              Service Layer                     │
│  AuthSvc · InterviewSvc · AIService ·         │
│  STTProvider · TTSProvider · QuestionPool     │
└────────────────────┬──────────────────────────┘
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

各层核心职责：

| 层次 | 核心职责 | 依赖方向 |
|------|----------|---------|
| **入口层** `cmd/server` | 读取配置，调用 App.New | `internal/app`, `internal/config` |
| **组合根** `internal/app` | 依赖装配、路由注册、优雅关闭 | 所有 internal + pkg |
| **Handler 层** `internal/handler` | HTTP 绑定/校验/委托 Service | Service, DTO, pkg |
| **WebSocket 层** `internal/ws` | 实时双向通信、会话状态机 | Service, Model |
| **Service 层** `internal/service` | 业务逻辑、外部 API 集成 | Model, pkg |
| **Model 层** `internal/model` | GORM 实体定义 | `gorm.io/gorm` |
| **公共包** `pkg/` | 通用工具（JWT/错误/校验/指标） | 仅标准库+第三方 |

> **依赖方向严格单向**：`pkg/` 不可引用 `internal/`，Service 不可引用 Handler。

## 五、WebSocket 并发模型

### 5.1 Hub — Actor 模型

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

### 5.2 Client 状态机

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

## 六、设计模式应用

| 模式 | 使用场景 | 实现 |
|------|---------|------|
| **工厂方法** | AI 模型选择 | `GetModelProviderFromConfig` 根据配置返回不同 ModelProvider |
| **策略模式** | 语音/AI Provider | AIService/STTProvider/TTSProvider 均为接口，多实现 |
| **信号量限流** | 并发 AI 请求控制 | `sem := make(chan struct{}, maxConcurrent)` |
| **模板缓存** | 反馈模板复用 | `TemplateFeedbackCache` 带 TTL 和容量限制 |
| **Actor 模型** | WebSocket 连接管理 | Hub + Client channel 通信 |

## 七、数据流

### 7.1 实时面试流程

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

### 7.2 AI 调用链

```
DeepSeekService.EvaluateAnswer
  → 构建 systemPrompt
  → ModelProvider.GenerateWithMaxTokens（信号量限流）
    → HTTP POST /v1/chat/completions（指数退避重试）
  → extractJSONFromMarkdown → json.Unmarshal → EvaluationResult
```

## 八、WebSocket 事件协议

### 8.1 客户端 → 服务端

| 事件类型 | 说明 | 数据字段 |
|---------|------|---------|
| `start_session` | 开始面试会话 | scenarioType, company, role, difficulty |
| `text_message` | 发送文字回答 | content, inputMode |
| `audio_chunk` | 发送语音数据 | audio (base64), format, pronunciation_only |
| `interrupt` | 打断 AI 回答 | — |
| `end_session` | 结束会话 | — |
| `heartbeat` | 心跳保活 | — |

### 8.2 服务端 → 客户端

| 事件类型 | 说明 | 数据字段 |
|---------|------|---------|
| `session_started` | 会话已启动 | status |
| `ai_response_final` | AI 完整回复 | content, type (question/answer/feedback) |
| `ai_response_partial` | AI 流式回复（部分） | content |
| `transcription` | 语音转文字结果 | turn_id, text |
| `five_dimension_evaluation` | 五维评分与反馈 | overall_score, dimensions, feedback |
| `audio_response` | AI 语音回复 | audio (base64), format |
| `typing_indicator` | AI 正在输入 | is_typing |
| `feedback_ready` | 会话总结反馈已写入 | Feedback 模型 JSON |
| `session_ended` | 会话结束确认 | status |
| `heartbeat_ack` | 心跳响应 | server_time |
| `interrupted` | 打断确认 | status |
| `error` | 错误信息 | code, message |

## 九、关键流程说明

### 9.1 文字面试流程

```
用户 → 前端 → WebSocket → 后端 → AI 出题 → 显示问题
     → 用户输入回答 → 发送 text_message
     → 后端 AI 评估 → 返回评分 + 下一题
     → 循环直到结束
```

### 9.2 语音面试流程（Web 端）

```
用户按住说话 → MediaRecorder 录音 + Web Speech API 实时转文字
     → 松开按钮 → 获取转写结果 → 发送 text_message
     → 后端 AI 评估 → 返回评分 + 下一题 + TTS 语音
```

### 9.3 语音面试流程（Native 端）

```
用户按住说话 → expo-audio 录音 → 松开按钮 → 发送 audio_chunk
     → 后端阿里云 STT 转文字 → 返回 transcription
     → 后端 AI 评估 → 返回评分 + 下一题 + TTS 语音
```

## 十、性能数据速查

| 指标 | 数值 | 说明 |
|------|------|------|
| 单机最大并发 | 10,000 | 2 核 4GB |
| 每连接内存 | ~50KB | 含缓冲和状态 |
| STT 端到端延迟 | ~510ms | 说话到看到文字 |
| AI 首字延迟 | ~0.5s | 流式响应 |
| 面试端到端 | ~4.5s/轮 | 问题生成 + TTS + 播放 |
| 总内存占用 | ~3GB | 4GB 机器够用 |

## 十一、阅读路径

| 目标 | 阅读顺序 |
|------|---------|
| **快速了解项目** | 01 代码分析 → 03 技术栈详解 |
| **架构设计深度** | 06 架构设计 → 04 核心技术选型 |
| **前后端协议** | 08 前后端协议 → 09 登录认证流程 |
| **AI 提示词系统** | 02 四Agent提示词设计 |
| **前端架构** | 10 前端架构详解 → 11 语音录制系统设计 |
| **AI 模型与评分** | 12 评分模型选型 → 13 LLM服务参数 → 14 Ollama生命周期 |
| **出题与 QA** | 15 出题流程优化 → 16 QA测试记录 |
| **面试准备** | 07 面试考点速查 |
| **英语面试实现** | 05 英语面试分层反馈 |
