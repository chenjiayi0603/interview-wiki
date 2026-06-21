# 出题流程与优化

> 进度参考文档，记录出题全链路和已完成的 6 项优化。

## 整体流程

```
CreateSessionDeck → GetQuestion → (deck 耗尽)AI Supplement → GetQuestion → ...
       ↑                                    ↑
  混合检索 + 加权洗牌               generateAISupplement (5 题)
```

### 1. CreateSessionDeck（会话创建时）

```
用户信息 + 场景参数
  ↓
getUserAskedTextsRecent(userID, 30day) → 跨会话去重黑名单
  ↓
query = keywords + role (无 keywords 时用 role+subScene)
  ↓
┌─ canHybrid? ─────────────────────────────────────┐
│  (query非空 && vectorSvc存在 && positionID有效)    │
│  YES → HybridSearch(query, positionID, scene, 60) │
│        Qdrant 向量 + DB LIKE → RRF k=60           │
│        结果 < 10 → ListByScene fallback 补齐       │
│  NO  → ListByScene(scene, 60)                     │
└──────────────────────────────────────────────────┘
  ↓
去重 → 构建 Question 列表（带 QualityScore、Source、CreatedAt）
  ↓
weightedShuffle → 冷启动 boost (7天内的新题权重×2)
  ↓
SessionDeck 返回
```

### 2. GetQuestion（每轮取题时）

```
Deck 为空？→ 返回 nil → 触发 AI 实时生成
  ↓
previousQuestions → case-insensitive dedup set
  ↓
attempt loop (最多 len(deck) 次):
  index 越界？→ AI supplement (5 题) → reshuffle → index 归零
  ↓
  q = deck[index]; index++
  q 在 previousQuestions 中？→ skip, continue
  ↓
  返回 QuestionResult{Text, Source, DBID}
```

### 3. 会话结束

```
endSessionAndGenerateFeedback
  ↓
UpdateQuestionUsageStats(askedDBIDs)
  → UseCount++, QualityScore = (UseCount+5)/(UseCount+10)
```

---

## 6 项优化详情

| # | 优化 | 位置 | 说明 |
|---|------|------|------|
| 1 | **题目来源追踪** | `question_pool.go:27` | Question.Source 标记 "hybrid_search" / "list_by_scene" / "ai_generated"；GetQuestion 返回值升级为 `QuestionResult` 结构体 |
| 2 | **30 天跨会话去重** | `question_pool.go:377` | `getUserAskedTextsRecent` 查询 `user_asked_questions` 表，WHERE `created_at > now()-30d`；旧的全量去重方法保留为 legacy |
| 3 | **无关键词混合检索降级** | `question_pool.go:236` | query 从 role+subScene 构造，即使 keywords 为空也能走 hybrid search；Qdrant 不可用时自动降级 ListByScene |
| 4 | **题库耗尽 AI 补充** | `question_pool.go:143` | `generateAISupplement` 批量生成 5 题追加到 deck，失败不中断流程，继续 reshuffle |
| 5 | **冷启动权重提升** | `question_pool.go:345` | `weightedShuffle` 内 time decay：新题（<7天）权重从 2x 线性降至 1x；QualityScore≤0 默认 1.0 |
| 6 | **QualityScore 动态更新** | `question_pool.go:410` | `UpdateQuestionUsageStats`：会话结束时 UseCount 递增，QualityScore 用递减收益公式更新，频繁使用的题目自然排序靠前 |

---

## 关键数据结构

```go
// Question — 题库中的单题（内部表示）
type Question struct {
    ID           string    // 内存 ID
    DBID         string    // DB UUID → 用于 usage 统计
    Category     string
    Question     string
    Difficulty   string
    QualityScore float64   // 0.5~1.0，影响洗牌权重
    Source       string    // hybrid_search / list_by_scene / ai_generated
    CreatedAt    time.Time // 用于冷启动 boost
}

// QuestionResult — GetQuestion 返回值（精简）
type QuestionResult struct {
    Text   string // 问题文本
    Source string // 来源标记
    DBID   string // 统计用
}

// SessionDeck — 每会话独立，无锁
type SessionDeck struct {
    questions   []Question
    used        map[string]bool
    index       int
    askedDBIDs  []string   // 会话结束时批量更新 usage
    // ... 场景参数、AI service 引用
}
```

---

## RAG 管线追踪（可观测性）

在出题全链路埋点，1 个 Session 对应 1 条 RAGTrace 记录，覆盖 Embedding → Qdrant → DB → RRF → Deck → AI 补充 全部阶段。

### 管线流程图

```
┌──────────────────┐     ┌──────────────────┐
│ ① 输入上下文       │     │ ② Embedding      │
│ Scene / Role      │ ──▶ │ Ollama bge-m3    │
│ Keywords(简历+职位)│     │ 文本→1024维向量   │
└──────────────────┘     └────────┬─────────┘
      │                          │
      ▼                          ▼
┌──────────────────┐     ┌──────────────────┐
│ ③ 向量检索 Qdrant │     │ ④ 关键词检索 DB  │
│ cosine 相似度     │     │ ILIKE 模糊匹配   │
│ limit=30, thr=0.6│     │ limit=40         │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
         └───────────┬────────────┘
                     ▼
          ┌──────────────────┐
          │ ⑤ RRF 融合 k=60  │
          │ 合并两路 → 去重   │
          │ → RRF 分数排序    │
          └────────┬─────────┘
                   ▼
          ┌──────────────────┐
          │ ⑥ 题库 SessionDeck│
          │ Deck = Hybrid     │
          │     + Fallback    │
          │     + ColdStart   │
          └────────┬─────────┘
                   ▼
          ┌──────────────────┐
          │ ⑦ AI 补充 (按需)  │
          │ Deck 耗尽时触发   │
          │ LLM 5 题/次       │
          └──────────────────┘
```

### RAGTrace 数据模型

```go
// backend/internal/model/rag_trace.go
type RAGTrace struct {
    ID        uuid.UUID // 主键
    SessionID uuid.UUID // 关联面试会话
    UserID    uuid.UUID

    // ① 输入上下文
    Keywords   string // JSON []string
    PositionID string
    Scene      string
    Role       string

    // ② Embedding
    EmbeddingMs int
    EmbeddingOK bool

    // ③ 向量检索
    VectorSearchMs  int
    VectorResultCnt int
    VectorTopScore  float64 // 最高余弦相似度
    VectorAvgScore  float64 // 平均相似度

    // ④ 关键词检索
    KeywordSearchMs  int
    KeywordResultCnt int

    // ⑤ RRF 融合
    FusionMs      int
    FusionFinalCnt int

    // ⑥ 题库组成
    DeckTotal      int
    HybridCnt      int // 混合检索贡献
    FallbackCnt    int // ListByScene 降级补充
    ColdStartBoost int // 7天内新题加权曝光

    // ⑦ AI 补充
    AISupplementCallCnt int // 调用次数
    AIQuestionServedCnt int // 实际使用题数

    TotalMs   int    // 全流程墙钟耗时
    ErrorMsg  string
    CreatedAt time.Time
}
```

### 采集点

| 阶段 | 位置 | 采集方式 |
|------|------|---------|
| ①②③④⑤ | `question_pool.go CreateSessionDeck` | 同步采集 → async goroutine 写 DB |
| ⑥⑦ | `question_pool.go GetQuestion` | 计数器累加（aiSupplementCalls / aiQuestionsServed） |
| ⑦ 回填 | `ws/client.go endSessionAndGenerateFeedback` | 调用 `FinalizeRAGTrace` 更新 AI 统计 |

**设计要点：**
- trace 记录用 `go func()` 异步写入，不阻塞出题路径
- `SessionDeck` 上挂 `ragTraceID`，session 生命周期内累计 AI 补充统计
- 1 Session = 1 Trace，不按每次 search 记录（避免数据爆炸）

### 关键词效果评估

后端 Stats API 自动从 `keywords` JSON 字段提取单个关键词，聚合统计：

```go
type KeywordEffect struct {
    Keyword          string  // 单个关键词
    UseCount         int     // 出现次数
    AvgVectorScore   float64 // 该关键词的平均向量相似度
    AvgDeckSize      int     // 平均产出 Deck 大小
    AISupplementRate float64 // AI 补充率（越低说明题库覆盖越好）
}
```

AI 补充率 > 30% 在管理面板标红，表示该关键词题库覆盖不足。

### 管理 API

| Method | Path | 说明 |
|--------|------|------|
| GET | `/api/v1/admin/rag-traces` | 分页列表，?keyword=&sessionId=&scene=&positionId= |
| GET | `/api/v1/admin/rag-traces/:id` | 单条详情 |
| GET | `/api/v1/admin/rag-traces/stats` | 聚合统计 + 关键词效果 |

### 前端管理页面

`/admin/rag-traces` — 三个层次：
1. **统计面板** — 追踪数、均 Deck、均向量分、Hybrid 占比、AI 补充率、均耗时
2. **关键词效果条** — 横向滚动，每个关键词显示使用次数/平均向量分/AI 补充率
3. **追踪列表** — 可分页，点入查看字符流程图 + 分阶段明细（每项带中文注释）

---

## 向量管理（去重、删除、同步修复）

### P0-1：启动时 ClearCollection 误删修复

**问题**：原启动流程中 `vectorSvc.ClearCollection(CollectionQuestions)` 会清空 Qdrant 中 `interview_questions` collection 的全部向量，但后续 `VectorizeAllPending` 只处理 `vector_status='pending'` 的题目。非 curated 来源（user_contribution / resume_keyword / ai_generated）的题目 `vector_status` 已是 `'vectorized'`，不会被重新向量化。**每次重启静默丢失非 curated 题目的 Qdrant 向量。**

**修复**：移除 `ClearCollection` 全集删除，改为精确删除。

```go
// app.go 启动流程（修复后）
// 1. CleanupAllCuratedQuestions(db, logger, vectorSvc)
//    → 收集 curated 题目 ID → DB 删除 → 精确 DeleteByQuestionIDs(ids)
// 2. CleanupLegacyScenes(db, logger, vectorSvc)
//    → 收集废弃场景题目 ID → DB 删除 → 精确调 DeleteQuestion(id)
// 3. MigrateJSONQuestions → 重新导入 curated（FirstOrCreate 幂等）
// 4. VectorizeAllPending → 只处理 vector_status='pending'
// 非 curated 向量不受影响 ✓
```

**涉及方法**：
- `VectorService.DeleteByQuestionIDs(ctx, collection, ids)` — 使用 `Should` filter 批量匹配 `question_id` payload
- `VectorService.DeleteBySource(ctx, collection, source)` — 使用 `MatchKeyword("source", source)` 删除指定来源
- `VectorService.DeleteQuestion(ctx, id)` — 单个删除（先查 Qdrant point ID 再删）

### P0-2：级联删除（DB 删除 → Qdrant 清理）

**问题**：3 个删除路径都不删除 Qdrant 向量，导致孤儿向量累积。

| 删除场景 | 修复前 | 修复后 |
|---------|--------|--------|
| `CleanupAllCuratedQuestions` | DB 删，Qdrant 残留 | DB 删 + `DeleteByQuestionIDs` 级联删 Qdrant |
| `CleanupLegacyScenes` | DB 删，Qdrant 残留 | DB 删 + `DeleteQuestion` 逐条清 Qdrant |
| Admin 拒绝已 vectorized 的题 | 改 status，向量残留 | `DeleteQuestion` + `vector_status` 重置为 `pending` |

**AdminService.ReviewQuestion 级联**（`backend/internal/service/admin.go`）：
```go
if !approved && q.VectorStatus == "vectorized" && s.vectorSvc != nil {
    if err := s.vectorSvc.DeleteQuestion(ctx, q.ID.String()); err != nil {
        s.logger.Warn("failed to delete Qdrant vector for rejected question", ...)
    } else {
        s.db.WithContext(ctx).Model(&q).Update("vector_status", "pending")
    }
}
```

### P0-3：向量去重（入库前语义相似度检查）

**问题**：向量去重只在 `ReviewService.ReviewQuestion`（AI 审核非信任用户贡献）一处生效，其余 3 条路径跳过去重：
- 信任用户自动通过（AutoApprove=true）→ 不去重
- 简历关键词生成题目 → 直接 approved → 不去重
- 启动批量 `VectorizeAllPending` → 不去重

**修复**：在 `VectorizeAllPending` 的 embedding 完成后、upsert 前加入语义去重检查。

```go
// backend/internal/service/question_extended.go
// VectorizeAllPending 内：
if vectorSvc != nil {
    results, _ := vectorSvc.SearchQuestions(ctx, vec, posIDStr(q.PositionID), q.Scene, 3, 0.95)
    if len(results) > 0 {
        s.db.WithContext(ctx).Model(&q).Updates(map[string]interface{}{
            "vector_status": "duplicate",
            "review_status": "rejected",
            "reject_reason": "向量去重：与已有题目高度相似",
        })
        continue  // 不写入 Qdrant
    }
}
```

阈值 0.95（比 `ReviewService` 0.9 更严格，因为是已 approved 题目间的去重），limit 3（只取 top-3 做比较）。

#### 去重原理

**bge-m3 为什么能判重**

bge-m3 是一个语义嵌入模型（1024 维），核心能力是：

> 语义相近的文本 → 向量空间中夹角小 → 余弦相似度高

```
"请描述你解决过的最困难的技术问题"    → vec_A
"谈谈你遇到过最难的技术挑战是什么"    → vec_B   // 换说法，语义几乎一样
"你最喜欢的编程语言是什么"            → vec_C   // 不同语义

cos(vec_A, vec_B) ≈ 0.97  ← 判重
cos(vec_A, vec_C) ≈ 0.55  ← 不判重
```

Qdrant 的 `search` 做的是近似最近邻搜索（ANN）：在 1024 维空间中找与查询向量夹角最小的 k 个点，score 就是余弦相似度。

**向量删不删？**

当前实现：**新题不入库，旧向量不动**。

```
VectorizeAllPending 的流程：
                        新题 embedding 完成
                              │
                              ▼
              Qdrant.SearchQuestions(threshold=0.95)
                    找 top-3 最相似向量
                              │
                    ┌─────────┴─────────┐
                    │                   │
               len > 0              len == 0
               (有相似)              (无相似)
                    │                   │
                    ▼                   ▼
           DB: review_status    Upsert 到 Qdrant
             → "rejected"       DB: vector_status
           vector_status          → "vectorized"
             → "duplicate"
           skip upsert ← 向量从不写入
```

旧向量不删的理由：
- 旧题是合法题目，正在题库中正常使用
- 重复的是新题（语义雷同），应拦截新题，保留旧题
- 如果删旧向量留新题，题库中已有的题目会丢向量，混合检索对此题降级为纯关键词
- 新题内容和旧题几乎一样，没有替换的必要

### P1-2：DB ↔ Qdrant 一致性对账

新增 `VectorAdminHandler.CheckConsistency` 端点，比较 DB 中 `vector_status='vectorized'` 的题目与 Qdrant 中所有 point 的 `question_id` payload：

```go
// backend/internal/handler/vector_admin.go
type ConsistencyReport struct {
    DBVectorizedCount int      `json:"dbVectorizedCount"`
    QdrantPointCount  int      `json:"qdrantPointCount"`
    MissingVectors    []string `json:"missingVectors"`  // DB有 Qdrant无
    OrphanVectors     []string `json:"orphanVectors"`   // Qdrant有 DB无
    OK                bool     `json:"ok"`
}
```

**API**：`POST /api/v1/admin/vectors/check-consistency`
**前端**：`/admin/rag-traces` 页面右上角"向量对账"按钮，结果在按钮下方展示

---

## 文件索引

| 文件 | 内容 |
|------|------|
| `backend/internal/service/question_pool.go` | QuestionPool、SessionDeck、weightedShuffle、AI supplement、RAG 埋点 |
| `backend/internal/ws/client.go` | GetQuestion 调用点、endSession → UpdateQuestionUsageStats + FinalizeRAGTrace |
| `backend/internal/service/question.go` | QuestionService: ListByScene、HybridSearch |
| `backend/internal/service/question_extended.go` | HybridSearchMetrics 采集、向量+关键词+RRF 各阶段计时 |
| `backend/internal/service/hybrid_search.go` | RRF 融合 |
| `backend/internal/service/embedding.go` | Ollama bge-m3 embedding |
| `backend/internal/service/rag_trace.go` | RAGTraceService: Record/Update/List/Stats、关键词效果聚合 |
| `backend/internal/handler/rag_trace_admin.go` | 管理端 API: list/detail/stats |
| `backend/internal/model/rag_trace.go` | RAGTrace GORM 模型 |
| `backend/internal/model/model.go:248` | InterviewQuestion GORM 模型 |
| `frontend/services/admin.ts` | RAGTraceItem/RAGTraceStats/KeywordEffect 类型 + API |
| `frontend/app/admin/rag-traces.tsx` | RAG 追踪管理面板（统计+流程图+关键词效果+列表+向量对账） |
| `backend/internal/service/vector_service.go` | Qdrant gRPC 客户端 — DeleteBySource / DeleteByQuestionIDs / ScrollQuestionIDs / DeleteQuestion |
| `backend/internal/service/scenario_cleanup.go` | 启动清理 — 精确级联删向量（不再 ClearCollection 全集） |
| `backend/internal/service/admin.go` | AdminService — ReviewQuestion 拒绝时级联删 Qdrant |
| `backend/internal/app/app.go` | 启动流程：移除 ClearCollection → 精确删 + 向量对账路由注册 |
| `backend/internal/handler/vector_admin.go` | 向量对账 admin API — CheckConsistency |
