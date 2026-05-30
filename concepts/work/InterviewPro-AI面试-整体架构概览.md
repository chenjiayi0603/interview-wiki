# InterviewPro-AI面试 - 整体架构概览

本项目是一个**实时英语面试辅助平台**的后端服务，支持：

- HTTP REST API（注册/登录/面试管理）
- WebSocket 实时对话（语音识别 STT → AI 出题/评测 → 语音合成 TTS）
- 多 AI 模型切换（DeepSeek 云端 / Qwen 本地；Qwen 服务架构与并发见 [`../docs/qwen-llm-architecture-and-concurrency.md`](../docs/qwen-llm-architecture-and-concurrency.md)）
- Prometheus 可观测性指标

整体采用**经典分层架构**，通过**依赖注入**在应用启动时手动组装所有依赖，无第三方 IoC 容器。

```
┌──────────────────────────────────────────────────────┐
│                    Client (Browser/App)               │
└──────────────┬───────────────────────┬───────────────┘
               │ HTTP REST             │ WebSocket
               ▼                       ▼
┌──────────────────────────────────────────────────────┐
│              Gin Router + Middleware Chain             │
│   (Recovery · CORS · RequestID · Metrics · JWT)       │
└──────────────┬───────────────────────┬───────────────┘
               │                       │
               ▼                       ▼
┌─────────────────────┐   ┌────────────────────────────┐
│     HTTP Handlers   │   │    WebSocket Hub + Client   │
│  (Auth/Interview/   │   │  (ReadPump/WritePump/        │
│   Question/User)    │   │   saveLoop/evaluateAndResp) │
└─────────┬───────────┘   └──────────────┬─────────────┘
          │                              │
          ▼                              ▼
┌──────────────────────────────────────────────────────┐
│                    Service Layer                       │
│  AuthSvc · InterviewSvc · AIService · QuestionPool    │
│  STTProvider · TTSProvider · PronunciationSvc         │
│  ScenarioConfigSvc · OptimizedAIService               │
└──────────────────────────┬───────────────────────────┘
                           │
          ┌────────────────┼───────────────────┐
          ▼                ▼                   ▼
   ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
   │  GORM / DB  │  │  AI Model    │  │ 阿里云/百炼   │
   │ (Postgres/  │  │  Provider    │  │ STT/TTS API  │
   │  SQLite)    │  │ (DeepSeek /  │  └──────────────┘
   └─────────────┘  │  Qwen Local) │
                    └──────────────┘
```

[src: raw/ingested/3项目/InterviewPro-AI面试/ai_interview-1.-整体架构概览.md]