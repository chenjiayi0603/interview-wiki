# Interview Pro Agent 提示词系统设计

> 更新时间：2026-06-06  
> 对应代码：`internal/agent/` 下 ScorerAgent + InterviewerAgent

---

## 1. 架构设计思路

### 1.1 当前架构：双 Agent + Eino Graph

与文档标题"四Agent"不同，当前代码实现的是**双 Agent + Eino Graph 编排**架构：

```
┌─────────────────────────────────────────────────────────┐
│                     用户输入 (回答/音频)                   │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Eino compose.Parallel（并行编排）              │
│                                                         │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│   │ ScorerAgent  │  │Interviewer  │  │Pronunciation │      │
│   │ (评分)       │  │Agent (出题)  │  │评测 (语音)   │      │
│   │ 温度: 0.1   │  │ 温度: 0.7   │  │ 外部 API     │      │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │
│          │                │                │              │
│          └────────────────┼────────────────┘              │
│                           │                               │
│                      [merge]                              │
│                    合并评分 + 问题                          │
└─────────────────────────────────────────────────────────┘
```

| Agent | 文件 | 职责 | 温度 |
|-------|------|------|------|
| **ScorerAgent** | `internal/agent/scorer.go` | 5 维评分 + 优缺点分析 | 0.1 |
| **InterviewerAgent** | `internal/agent/interviewer.go` | 上下文感知出题 | 0.7 |
| **Pronunciation** | `internal/service/pronunciation/` | 发音评测（外部 API） | - |

### 1.2 为什么不需要 4 个独立 Agent

原始设计文档提出的"4 Agent"方案（流程管家 + 简历分析师 + 面试官 + 面试教练），在实际实现中被简化为 2 个核心 Agent + Eino Graph 编排，原因：

| 原始设计 | 实际实现 | 原因 |
|----------|----------|------|
| 流程管家 (Orchestrator) | GraphRunner + Eino Parallel | Eino 框架自带编排能力，不需要 LLM 做路由决策 |
| 简历分析师 | ResumeContext → InterviewerAgent | 简历信息作为 prompt 上下文注入，不需要独立 Agent |
| 面试官 (Interviewer) | InterviewerAgent | 保留，但使用 Eino ChatModel 统一接口 |
| 面试教练 (Coach) | ScorerAgent | 评分 + 反馈合并为一个 Agent，避免角色冲突 |

---

## 2. ScorerAgent 提示词设计

### 2.1 System Prompt

文件：`config/prompts/system/five_dim_eval.md`（可外部配置）

```
You are an honest interview evaluator. Score the candidate's answer on four dimensions (1-10):
- Fluency: smoothness and natural flow
- Grammar: grammatical accuracy
- Vocabulary: word choice and range
- Content: relevance, depth, and structure

Scoring calibration:
- 8-10: Excellent, clear competitive advantage
- 6-7: Good, solid performance with minor gaps
- 4-5: Average, needs development but shows effort
- 1-3: Significant improvement needed

Feedback rules:
- "strengths": Be honest. List 1-3 genuine, specific positives.
- "areas_for_improvement": Be specific and actionable. List 1-3 concrete improvements.

Output a JSON object with overall_score, dimensions (fluency/grammar/vocabulary/content
each with score/issues/suggestions), and feedback (strengths/areas_for_improvement).
Return ONLY the JSON object, no markdown.
```

### 2.2 User Prompt 构建

```go
func buildScorerUserPrompt(input *ScorerInput) string {
    if input.ReferenceAnswer != "" {
        return fmt.Sprintf("Reference answer: %s\n\nQuestion: %s\n\nCandidate's answer: %s",
            input.ReferenceAnswer, input.Question, input.Answer)
    }
    return fmt.Sprintf("Question: %s\n\nCandidate's answer: %s", input.Question, input.Answer)
}
```

### 2.3 输出解析

```go
type ScorerOutput struct {
    Overall    float64          `json:"overall_score"`
    Dimensions ScorerDimensions `json:"dimensions"`
    Feedback   ScorerFeedback   `json:"feedback"`
}

type ScorerDimensions struct {
    Fluency    ScorerDimScore `json:"fluency"`
    Grammar    ScorerDimScore `json:"grammar"`
    Vocabulary ScorerDimScore `json:"vocabulary"`
    Content    ScorerDimScore `json:"content"`
}

type ScorerDimScore struct {
    Score       float64  `json:"score"`
    Issues      []string `json:"issues"`
    Suggestions []string `json:"suggestions"`
}

type ScorerFeedback struct {
    Strengths    []string `json:"strengths"`
    Improvements []string `json:"areas_for_improvement"`
}
```

解析兼容 markdown fence、前缀文本等 LLM 输出格式问题。

### 2.4 参考回答（Reference Answer）校准

当题目有预存参考回答时，注入 user prompt 用于校准 Content 分数：
- Content 维度参考参考回答的深度和结构
- 其他维度仍基于候选人实际回答
- `SampleImprovedAnswer` 直接使用预存回答，不再由 LLM 生成

---

## 3. InterviewerAgent 提示词设计

### 3.1 System Prompt

```go
func buildMessages(input *InterviewInput) []*schema.Message {
    systemPrompt := svcai.GetSystemPrompt(
        input.AICharacter,
        input.ScenarioType,
        input.Difficulty,
        a.skills,
        a.prompts,
    )
    // ...
}
```

System Prompt 由 `skills` 和 `prompts` Provider 动态构建，包含：
- **角色设定**：HR / Tech Lead / Boss，不同角色有不同提问风格
- **场景类型**：行为面试 / 技术面试 / 情景面试
- **难度级别**：Junior / Mid / Senior，决定问题深度
- **技能要求**：从技能配置中加载目标岗位的技能列表

### 3.2 历史消息构建

InterviewerAgent 使用完整的对话历史构建消息数组：

```
[System] {system prompt}
[Assistant] {上一轮问题}
[User] {上一轮回答}
[Assistant] {上上轮问题}
[User] {上上轮回答}
...
[User] {当前请求：Role=X, Company=Y, 生成下一个问题}
```

### 3.3 首题 vs 追问策略

```go
if len(input.History) == 0 {
    // 首题：要求具体、有区分度
    sb.WriteString("This is the FIRST question. Ask a specific, varied opening question " +
        "-- avoid generic 'tell me about a recent project' openers.")
} else {
    // 追问：基于对话历史，避免重复主题
    sb.WriteString("Based on the conversation above, ask a contextually appropriate " +
        "follow-up or NEW question. Do NOT ask about the same topic or theme already covered.")
}
```

### 3.4 LTM 快照注入

当用户有历史面试记录时，LTM Snapshot 注入出题 prompt：

```go
if input.LTMSnapshot != nil && input.LTMSnapshot.HasHistory {
    fmt.Fprintf(&sb, "Candidate's past performance summary:\n%s\n", input.LTMSnapshot.Summary)
    if len(input.LTMSnapshot.Weaknesses) > 0 {
        fmt.Fprintf(&sb, "Areas to probe: %v\n", input.LTMSnapshot.Weaknesses)
    }
}
```

---

## 4. 提示词管理

### 4.1 外部配置文件

提示词存储在 `config/prompts/` 目录下，支持运行时修改后重新加载：

```
config/prompts/
├── system/
│   ├── five_dim_eval.md         # ScorerAgent system prompt（可外部编辑）
│   └── ...
└── interview/
    ├── hr.md                    # HR 角色
    ├── tech-lead.md             # 技术主管角色
    └── boss.md                  # Boss 角色
```

### 4.2 PromptManager

`internal/service/prompt_manager.go` 管理所有提示词的加载和缓存。

### 4.3 管理员 API

通过管理端 API `/api/admin/prompts` 可在线编辑提示词，无需重启服务。

---

## 5. 与原始四Agent方案对比

| 原始设计 | 当前实现 | 差异说明 |
|----------|----------|----------|
| 4 个独立 Agent | 2 个 Agent + Eino Graph | Eino 编排取代了 Orchestrator，发音评测用外部 API |
| 流程管家 (LLM) | GraphRunner (代码) | 路由逻辑由代码控制，更可靠、可测试 |
| 简历分析师 (LLM) | ResumeContext (数据) | 简历信息作为结构化数据注入 prompt |
| 面试官 | InterviewerAgent | 保留并增强，使用完整对话历史 |
| 面试教练 | ScorerAgent | 合并评分 + 反馈，输出结构化 JSON |
| 对话记忆 | LTM (Qdrant) | 向量数据库存储跨会话记忆 |
| 题库支撑 | QuestionPool | 预存题目 + 参考回答，减少 LLM 调用 |
