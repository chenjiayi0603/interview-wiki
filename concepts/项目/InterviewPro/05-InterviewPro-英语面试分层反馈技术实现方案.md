# InterviewPro 英语面试分层反馈技术实现方案

> 更新时间：2026-06-06  
> 对应代码：`internal/agent/` + `internal/ws/session_flow.go`

---

## 概述

InterviewPro 通过三层级反馈架构，实现"及时反馈"与"不打断用户思路"的平衡。后端使用 Eino 编排的 ScorerAgent + Pronunciation 双通道评估，在前端配合实现分层展示。

---

## 一、整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户对话层                                    │
│     语音输入 → STT 转文字 → 用户回答文本                               │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                       Eino 并行评估引擎                                │
│                                                                    │
│   ┌──────────────────┐    ┌──────────────────┐                      │
│   │   ScorerAgent     │    │  Pronunciation    │                      │
│   │   (LLM 评分)      │    │  (发音评测 API)   │                      │
│   │                   │    │                   │                      │
│   │ • Fluency 1-10   │    │ • Pron 0-100      │                      │
│   │ • Grammar 1-10   │    │ • Accuracy 0-100  │                      │
│   │ • Vocabulary 1-10│    │ • Integrity 0-100 │                      │
│   │ • Content 1-10   │    │ • Fluency 0-100   │                      │
│   │ • Issues/Suggest │    └──────────────────┘                      │
│   └──────────────────┘                                              │
│                                                                    │
│   ┌──────────────────────────────────────────────────────────┐      │
│   │              整体分算术均值计算                             │      │
│   │  OverallScore = avg(Fluency, Grammar, Vocabulary,         │      │
│   │                   Content, Pronunciation(voice mode))     │      │
│   └──────────────────────────────────────────────────────────┘      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                      反馈分层输出                                    │
│                                                                    │
│   Level 1: WebSocket 实时推送分数                                    │
│   └── evaluation_result: {overall, fluency, grammar, ...}           │
│                                                                    │
│   Level 2: 文本反馈 + 语音合成                                        │
│   └── interviewer 语气给出下一问题或简短点评                           │
│                                                                    │
│   Level 3: 会话结束完整报告                                           │
│   └── 5 维雷达图 + 具体 Issue/Suggestion + 改进方向                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、评分体系

### 2.1 五维评分

| 维度 | 范围 | 评分方式 | 说明 |
|------|------|----------|------|
| **Fluency** | 1-10 | ScorerAgent (LLM) | 流利度、自然度 |
| **Grammar** | 1-10 | ScorerAgent (LLM) | 语法准确性 |
| **Vocabulary** | 1-10 | ScorerAgent (LLM) | 词汇丰富度和准确性 |
| **Content** | 1-10 | ScorerAgent (LLM) | 内容相关性、深度、结构 |
| **Pronunciation** | 0-100 | 发音评测 API | 语音模式时由独立评测给出 |

### 2.2 整体分计算

```go
// internal/service/ai/types.go
func ArithmeticMeanOverall(hasVoice bool, pron, fluency, grammar, vocabulary, content float64) float64 {
    // 语音模式：5 维平均
    // 文本模式：4 维平均（不含 Pronunciation）
}
```

> 使用算术均值而非 LLM 自返的 overall_score，因为 LLM 自返值偏低。

### 2.3 评分校准

| 分数段 | 评级 | 说明 |
|--------|------|------|
| 8-10 | 优秀 | 明显的竞争优势 |
| 6-7 | 良好 | 扎实表现，有小差距 |
| 4-5 | 一般 | 需要提升，但看到了努力 |
| 1-3 | 待提高 | 需要显著改进 |

---

## 三、实时反馈流程

### 3.1 文本回答流程

```
用户提交文本回答
    │
    ▼
GraphRunner.Run()
    │
    ├── 并行执行:
    │   ├── ScorerAgent.Evaluate()
    │   │     → 4 维评分 + 优缺点 + 改进建议
    │   │
    │   └── InterviewerAgent.GenerateNextQuestion()
    │         → 基于对话历史生成下一题
    │
    ├── 结果组装:
    │     → FiveDimensionResult + next_question
    │
    └── WebSocket 推送:
          ├── evaluation_result (分数 + 反馈)
          └── audio_response (TTS 合成)
```

### 3.2 语音回答流程

```
用户语音输入
    │
    ▼
STT 转文字 (Bailian/Whisper/...)
    │
    ▼
GraphRunner.Run()
    │
    ├── 并行执行:
    │   ├── ScorerAgent.Evaluate()     ← 基于转写文本
    │   ├── InterviewerAgent.GenerateNextQuestion()
    │   └── PronunciationEval()        ← 基于原始音频
    │
    ├── 结果组装:
    │     → FiveDimensionResult (含发音分) + next_question
    │
    └── WebSocket 推送:
          ├── evaluation_result (5 维分数含发音)
          └── audio_response (TTS 合成下一题)
```

### 3.3 会话启动流程

```
handleStartSession()
    │
    ├── 1. 解析场景参数
    ├── 2. 异步召回 LTM（跨会话记忆）
    ├── 3. 预热 STT 引擎
    ├── 4. 重连检测
    ├── 5. 生成首题（题库优先）
    └── 6. TTS 合成推送
```

---

## 四、分层反馈实现

### 4.1 Level 1: 实时分数

**WebSocket 事件：`evaluation_result`**

```json
{
  "event": "evaluation_result",
  "data": {
    "overall": 7.5,
    "dimensions": {
      "fluency": { "score": 8, "issues": ["偶尔停顿"], "suggestions": ["多用填充语过渡"] },
      "grammar": { "score": 7, "issues": ["时态混淆"], "suggestions": ["注意过去时与现在时区分"] },
      "vocabulary": { "score": 7.5, "issues": ["词汇重复"], "suggestions": ["使用同义替换"] },
      "content": { "score": 8, "issues": [], "suggestions": [] },
      "pronunciation": { "score": 85 }
    },
    "feedback": {
      "strengths": ["结构清晰", "有具体数据支撑"],
      "areas_for_improvement": ["注意时态一致性", "增加高级词汇"],
      "sample_improved_answer": "预存参考回答（来自题库）"
    }
  }
}
```

### 4.2 Level 2: 语音点评

AI 生成的下一问题通过 TTS 合成语音推送，模拟真人面试官的连续对话体验：

```json
{
  "event": "audio_response",
  "data": {
    "audio": "<base64 音频>",
    "text": "Good answer! Could you tell me about a time when you had to handle a difficult team situation?"
  }
}
```

### 4.3 Level 3: 会话报告

会话结束时生成完整报告：

```json
{
  "event": "session_ended",
  "data": {
    "session_id": "xxx",
    "overall_score": 7.2,
    "dimension_scores": { ... },
    "strengths_summary": ["沟通清晰", "有结构思维"],
    "improvement_areas": ["语法准确性", "词汇多样性"],
    "turn_count": 8,
    "duration_seconds": 720,
    "detailed_feedback": [
      { "turn": 1, "question": "...", "answer": "...", "score": 7.5, "feedback": "..." },
      { "turn": 2, ... }
    ]
  }
}
```

会话结束后异步触发 LTM Store，将本场总结存入 Qdrant。

---

## 五、题库集成

### 5.1 题目池（QuestionPool）

```
QuestionPool
  ├── 按场景 (ScenarioType) 分类
  ├── 按难度 (Difficulty) 分级
  ├── 预存 question + reference_answer
  └── 避免已问题目重复

在 GraphRunner 中：
  Step 1: 查找题库
    ├── 命中 → 跳过 question_gen LLM 调用，直接使用池中题目
    └── 未命中 → 走 InterviewerAgent 生成
```

### 5.2 参考回答（Reference Answer）

- 题库中每个题目预存参考回答
- ScorerAgent 使用参考回答校准 Content 维度评分
- `SampleImprovedAnswer` 直接使用参考回答，不由 LLM 生成
- 避免 LLM 幻觉，保证反馈质量

---

## 六、长期记忆（LTM）

### 6.1 跨会话记忆

```
Session 1:
  回答 → 评分 → 总结 → 向量化 → 存入 Qdrant
                                              ↓
Session 2:
  启动 → 搜索 Qdrant → 获取历史总结 → 注入出题 prompt
  → "候选人过去表现总结：... "
  → "需要重点考察的薄弱项：..."
```

### 6.2 记忆存储

```go
// 会话结束时异步调用
func (m *InterviewMemory) Store(ctx, userID, sessionID, transcript string) error
  ├── LLM 生成摘要（2-3 句）
  ├── bge-m3 向量化（1024 维）
  └── Upsert 到 Qdrant interview_records 集合
```

### 6.3 记忆召回

```go
func (m *InterviewMemory) Recall(ctx, userID string) (*LTMSnapshot, error)
  ├── 搜索 Qdrant（top-5 最近会话）
  ├── 提取摘要 → 合并 Summary
  ├── 关键词匹配 → Strengths / Weaknesses
  └── 返回 LTMSnapshot
```

---

## 七、与旧方案对比

| 维度 | 旧方案（原始设计） | 当前实现 |
|------|-------------------|----------|
| 评分方式 | 5 维 LLM 评分 | ScorerAgent + 发音评测 API 双通道 |
| 整体分 | LLM 自返 | 算术均值（解决 LLM 偏低问题） |
| 出题方式 | 无上下文生成 | InterviewerAgent + 题库双来源 |
| 参考回答 | LLM 生成 | 题库预存（避免幻觉） |
| 发音评测 | 阿里云轮询 | 讯飞 ISE / 阿里云 ASR 双后端 |
| 跨会话记忆 | 无 | Qdrant 向量存储 + LLM 摘要 |
| 题库 | 无 | QuestionPool + reference_answer |
| 并发 | 串行 | Eino Parallel 三路并行 |
| 反馈时机 | 回答后统一反馈 | 实时分数推送 + 语音合成点评 |

---

## 八、关键代码路径

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 图编排 | `internal/agent/graph.go` | `NewInterviewGraph()` |
| 执行器 | `internal/agent/graph_runner.go` | `Run()` |
| 评分 Agent | `internal/agent/scorer.go` | `Evaluate()` |
| 出题 Agent | `internal/agent/interviewer.go` | `GenerateNextQuestion()` |
| 记忆 | `internal/agent/memory.go` | `Recall()` / `Store()` |
| 会话流程 | `internal/ws/session_flow.go` | `handleTextAnswer()` |
| 5 维结果 | `internal/service/ai/types.go` | `FiveDimensionResult` |
