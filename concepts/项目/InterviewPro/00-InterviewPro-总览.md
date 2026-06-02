# InterviewPro AI 面试 App 总览

> 跨平台 AI 面试练习 App（Go 后端 + React Native 前端），2015-2019。

---

## 一、文件地图

| 文件 | 内容 |
|------|------|
| **01-InterviewPro-架构设计.md** | 后端分层架构、核心类组织、设计模式、并发模型 |
| **02-InterviewPro-面试考点速查.md** | STAR 故事、性能数据、高频 Q&A |

## 二、项目速览

| 项目 | 数值 |
|------|------|
| 产品 | AI 面试练习 App（iOS/Android/H5） |
| 技术栈 | Go + Gin + WebSocket + React Native + DeepSeek + 阿里云语音 |
| 部署 | Docker + K3s 单节点 |
| 单机连接 | 10,000 并发（2 核 4GB） |
| 端到端延迟 | 510ms（语音识别）、4.5s（问题生成 + TTS） |

## 三、核心特性

- 流式 WebSocket 实时语音对话
- DeepSeek / Qwen 双模型切换（策略模式）
- 阿里云 STT/TTS 语音服务集成
- Prometheus 可观测性埋点
- 手工依赖注入（无框架），分层架构纯净
