## P1：InterviewPro 简历知识点论证

**简历原文**：独立开发 InterviewPro AI 面试练习 App（Go + React Native + DeepSeek AI + 流式 ASR，全栈独立完成）

### 数据支撑

| 指标 | 值 |
|------|----|
| 首字响应延迟 | <200ms |
| 流式 ASR | 阿里云实时（WebSocket）|
| ASR 中间结果间隔 | 200ms/次 |
| DeepSeek 超时阈值 | 3s |
| context window | 8K token |

### 核心技术点

**流式架构延迟优化**：ASR 中间结果稳定后立即触发 AI（不等终态），AI 流式输出（不等完整响应），首字延迟降到 <200ms

**DeepSeek 双模型容错降级**：请求 DeepSeek 同时启动定时器（3s），超时降级到本地小模型，前端无感切换

**多轮上下文管理**：8K token 限制，按"轮数衰减 + 关键词权重"截断，避免 FIFO 丢失早期重要信息

### 追问预案

**Q：流式 ASR 中间结果不稳定，怎么避免重复触发 AI？**
连续 N（=3）次相同后才触发，实测重复率 <5%

**Q：DeepSeek 和小模型质量差异？**
前端提示用户，短问短答场景影响小，后续加 RAG 补足

[src: raw/ingested/3项目/面试准备/简历知识点论证手册.md]
