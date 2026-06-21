# InterviewPro 爬虫系统设计

> 面试题库数据采集 —— 轻量面经爬虫，按岗位定向抓取牛客/掘金面经 → 调 Go 后端 API 出题入库。
> 参考实现：`/home/tommychen/english-learner/crawler`（SSH: ubuntu26@192.168.3.61）

---

## 一、爬虫定位

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  外部数据源    │ ──→ │  Python 爬虫      │ ──→ │  InterviewPro │
│ (牛客/掘金)   │     │  (搜索+抓取+去重)  │     │  Go 后端      │
└──────────────┘     └────────┬─────────┘     └──────────────┘
                              │ ① POST /api/v1/admin/questions/smart-parse
                              │ ② POST /api/v1/admin/questions/batch-insert
                              ▼
                       ┌──────────────┐
                       │  state.json  │
                       │  (去重持久化)  │
                       └──────────────┘
```

- **目标**：按岗位关键词定向采集面经文章，经 LLM 解析生成面试题，批量入库
- **架构**：**Python 爬虫**（搜索 + 抓取 + 去重）→ 调用 **Go 后端 API**（LLM 解析 + 入库）
- **输出**：结构化面试题 → 直接写入 InterviewPro 题库
- **频率**：每日凌晨 3 点（CST），cron 触发 + 随机 0-30min jitter

---

## 二、数据源清单

| 数据源 | 类型 | 采集方式 | 是否需要登录 | 备注 |
|--------|------|---------|-------------|------|
| 牛客网 `nowcoder.com` | 面经帖子 | HTML 解析 + `__INITIAL_STATE__` 提取 | **否**（游客可用） | 搜索页 + 面经板块 |
| 掘金 `juejin.cn` | 技术文章 | REST API 调用 | **否**（游客可用） | 搜索 API + 推荐流 |

> 两个源均**无需 Cookie** 即可正常抓取。Cookie 为可选项，配置后可提升搜索精准度。

---

## 三、架构设计

### 3.1 整体架构

```
┌──────────────────────────────────────────────────────────┐
│                    main.py (入口 + 调度)                    │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│   │ argparse │  │ 配置加载  │  │ 日志设置  │              │
│   │ CLI      │  │ (YAML)   │  │          │              │
│   └──────────┘  └──────────┘  └──────────┘              │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│                    run() 主循环                             │
│  遍历 jobs[] × sources[] → 逐 keyword 搜索 → 逐文章处理      │
└────────┬───────────┬───────────┬──────────────────────────┘
         │           │           │
         ▼           ▼           ▼
┌────────────┐ ┌────────────┐ ┌──────────────────────┐
│ spiders/   │ │ dedup.py   │ │ importer.py           │
│ 反爬+搜索+  │ │ SHA256     │ │ Go 后端 API 客户端     │
│ 详情抓取   │ │ 去重       │ │ auth + smart-parse   │
└────────────┘ └────────────┘ │ + batch-insert       │
                              └──────────────────────┘
```

### 3.2 模块职责

| 模块 | 职责 | 关键实现 |
|------|------|---------|
| **main.py** | CLI 入口、调度循环、cron jitter | argparse + YAML 配置驱动 |
| **spiders/base.py** | Spider 基类、反爬策略封装 | 9 UA 池、70/20/10 延迟分布、Session 轮换、指数退避重试 |
| **spiders/juejin.py** | 掘金 API 爬虫 | 搜索 API + 推荐流回退 + 文章详情 API |
| **spiders/nowcoder.py** | 牛客 HTML 爬虫 | 搜索页解析 + 面经板块 + `__INITIAL_STATE__` 提取 |
| **models.py** | 数据模型定义 | RawArticle / ParsedQuestion / CrawlResult |
| **dedup.py** | 内容去重 | SHA256(title + content[:500]) + state.json 持久化 |
| **importer.py** | Go 后端 API 客户端 | 认证 (JWT) + 岗位列表 + smart-parse + batch-insert |

### 3.3 数据流

```
main.py run()
  │
  ├─ 读取 config.yaml + 环境变量
  ├─ importer.authenticate() → JWT token
  ├─ importer.fetch_jobs() → 从数据库加载岗位列表
  ├─ 合并 extra_jobs（定向主题搜索）
  │
  └─ 遍历 jobs[]
       └─ 遍历 sources[]
            ├─ create_spider() → 工厂方法
            ├─ spider.search(keyword) → list[RawArticle]
            │    ├─ juejin: POST search API / 推荐流
            │    └─ nowcoder: 搜索页 HTML 解析 / 面经板块
            │
            └─ 遍历 articles[]
                 ├─ spider.get_article_detail() → 正文
                 ├─ dedup.is_duplicate() → 跳过已处理
                 ├─ importer.smart_parse() → LLM 解析出题
                 │    └─ POST /api/v1/admin/questions/smart-parse
                 ├─ 闸2 过滤 relevance < 3
                 └─ importer.batch_insert() → 批量入库
                      └─ POST /api/v1/admin/questions/batch-insert
```

---

## 四、反爬策略

### 4.1 反爬矩阵

| 策略 | 实现 | 代码位置 |
|------|------|---------|
| **UA 池** | 9 种真实浏览器 UA（Chrome/Firefox/Safari x Win/Mac/Linux） | `base.py` `USER_AGENTS` |
| **请求间隔** | 70% 正常延迟 / 20% 长延迟(×1.5-2.5) / 10% 超长延迟(×3-5) | `base.py` `_human_delay()` |
| **Session 轮换** | 每 10-15 个请求重建 Session（换浏览器指纹） | `base.py` `_request()` |
| **Referer 模拟** | 详情页请求带上 referer，模拟从搜索页点击进入 | `base.py` `_request()` |
| **指数退避重试** | 429/网络异常 → 2ⁿ × random(3-8)s，最多 3 次 | `base.py` `_request()` |
| **验证码检测** | 403/503 响应体检测 "captcha"/"verify"，抛 `AntiCrawlerException` | `base.py` `_request()` |
| **Cookie 可选** | 配置后附加到请求头，无 Cookie 也能正常抓取 | `juejin.py`/`nowcoder.py` |

### 4.2 延迟分布

```python
def _human_delay(self):
    low, high = self.delay_range  # 默认 [3, 8] 秒
    roll = random.random()
    if roll < 0.7:        # 70%：正常阅读节奏
        delay = random.uniform(low, high)
    elif roll < 0.9:      # 20%：像在仔细读长文
        delay = random.uniform(high * 1.5, high * 2.5)
    else:                 # 10%：像离开去喝水
        delay = random.uniform(high * 3, high * 5)
    time.sleep(delay)
```

### 4.3 去重

#### 4.3.1 精确去重（当前方案）

```
SHA256(title + content[:500])
  → state.json（持久化到磁盘）
  → 内存 set 加速判断
```

- **算法**：SHA256 全文哈希（字段 = title + content 前 500 字）
- **存储**：`state.json`（JSON 数组，支持 Docker 持久卷挂载）
- **路径**：`CRAWLER_STATE_DIR` 环境变量可配置，默认项目根目录

#### 4.3.2 语义去重（向量方案 — Go 后端已实现）

精确哈希无法处理"同一面经被改写/转载/翻译"的场景。Go 后端通过 **bge-m3 embedding + pgvector** 实现了两道向量去重：

- **批量入库后**：`VectorizeAllPending()` 逐条 embedding → pgvector 查重（阈值 0.95）→ 重复则标记 `duplicate`
- **单条插入时**：`generateEmbedding()` 直接 upsert 向量（新题不太可能重复）

##### 完整去重流水线

```
┌─────────────────────────────────────────────────────────────────┐
│                   三道去重防线                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  第一道：Python 爬虫 SHA256 精确去重                               │
│    dedup.py content_hash(title + content[:500])                   │
│    → 完全相同的文章跳过，省掉后续的 HTTP 请求                         │
│                                                                  │
│  第二道：Go 后端 batch-insert 文本去重                              │
│    question_admin.go:589                                          │
│    → WHERE question = ? 精确匹配，跳过已存在的完全相同问题           │
│                                                                  │
│  第三道：向量语义去重（核心）                                       │
│    question_extended.go:379 VectorizeAllPending                    │
│    → bge-m3 embedding → pgvector cosine 搜索(阈值0.95)             │
│    → 相似度 > 0.95 标 vector_status=duplicate                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

##### 第三道详解：VectorizeAllPending 向量去重

**文件**：`backend/internal/service/question_extended.go:379`

```go
func (s *QuestionService) VectorizeAllPending(ctx context.Context,
    embeddingSvc *EmbeddingService, vectorSvc VectorProvider) (int, error) {

    // 只处理 review_status=approved 且 vector_status=pending 的题目
    var questions []model.InterviewQuestion
    s.db.WithContext(ctx).Model(&model.InterviewQuestion{}).
        Where("review_status = ? AND vector_status = ?", "approved", "pending").
        Find(&questions)

    for _, q := range questions {
        // 拼接 question + question_zh 作为 embedding 输入
        text := q.Question
        if q.QuestionZh != "" {
            text += " " + q.QuestionZh
        }

        // ① 调用 bge-m3 生成 1024 维向量
        vec, err := embeddingSvc.Embed(ctx, text)
        if err != nil {
            continue  // embedding 失败静默跳过，不阻塞
        }

        // ② 查重：在 pgvector 中搜索语义相似度 > 0.95 的题目
        results, _ := vectorSvc.SearchQuestions(ctx, vec,
            posIDStr(q.PositionID), q.Scene, 3, 0.95)
        if len(results) > 0 {
            // 标记为重复，跳过 upsert
            s.db.Model(&q).Updates(map[string]interface{}{
                "vector_status": "duplicate",
                "review_status": "rejected",
            })
            continue
        }

        // ③ 无重复 → upsert 向量
        vectorSvc.UpsertQuestion(ctx, q.ID.String(), vec, &VectorPayload{
            QuestionID: q.ID.String(),
            PositionID: posIDStr(q.PositionID),
            Scene:      q.Scene,
            Difficulty: q.Difficulty,
            Source:     q.Source,
        })

        // ④ 标记向量化完成
        s.db.Model(&q).Update("vector_status", "vectorized")
        count++
    }
    return count, nil
}
```

##### EmbeddingService 详解

**文件**：`backend/internal/service/embedding.go:19`

```go
type EmbeddingService struct {
    baseURL    string     // https://api.siliconflow.cn/v1
    model      string     // BAAI/bge-m3
    apiKey     string     // 从 SILICONFLOW_API_KEY 读取
    dimensions int        // 1024
    timeout    time.Duration
    client     *http.Client
    logger     *zap.Logger
    sem        chan struct{}  // 并发控制信号量
}
```

**调用硅基流 API**：

```
POST https://api.siliconflow.cn/v1/embeddings
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "model": "BAAI/bge-m3",
  "input": ["题目文本..."],
  "encoding_format": "float"
}

← 响应
{
  "data": [{ "embedding": [0.012, 0.034, ...], "index": 0 }],
  "model": "BAAI/bge-m3"
}
```

**关键设计**：
- **凭证热加载**：`resolved()` 方法每次调用时从 DB 配置表读取最新的 API Key，修改后立即生效，无需重启
- **并发控制**：通过 `sem chan struct{}` 信号量限制最大并发数（`embedding.max_concurrent`），避免打爆 API
- **批量处理**：`EmbedBatch()` 支持一次请求多条文本，分批（每批 20 条）调用硅基流 API
- **超时控制**：`http.Client` 设置 30 秒超时，`EmbedBatch` 内部每批独立超时
- **不需要 query 前缀**：bge-m3 v3 不需要加 `"为这个句子生成向量"` 前缀，实测加前缀反而降低相似度

##### VectorProvider 接口与 pgvector

**文件**：`backend/internal/service/vector_provider.go:8`

```go
type VectorProvider interface {
    Name() string
    SearchQuestions(ctx context.Context, vector []float32,
        positionID, scene string, limit uint64,
        scoreThreshold float32) ([]ScoredPoint, error)
    UpsertQuestion(ctx context.Context, id string,
        vector []float32, payload *VectorPayload) error
    DeleteQuestion(ctx context.Context, id string) error
    // ... 其他方法
}
```

**pgvector 实现**（`pgvector_provider.go`）：

SearchQuestions 使用 PostgreSQL 原生向量索引：

```sql
SELECT id, 1 - (embedding <=> $1::vector) AS score,
       question_id, position_id, scene, difficulty
FROM interview_embeddings
WHERE position_id = $2 AND scene = $3
ORDER BY embedding <=> $1::vector  -- cosine 距离
LIMIT $4
```

- `<=>` 是 pgvector 的 cosine 距离运算符
- `1 - distance` 得到 cosine 相似度（0~1）
- HNSW 索引加速（`SET LOCAL hnsw.ef_search = 100`）
- 支持按 `position_id` 和 `scene` 过滤

##### 配置

```yaml
# backend/config/config.yaml
embedding:
  base_url: "https://api.siliconflow.cn/v1"   # 硅基流 API
  model: "BAAI/bge-m3"                         # 1024 维中英双语
  dimensions: 1024
  timeout: 30
  max_concurrent: 5                            # 可选，默认无限制

# 环境变量覆盖（热加载，无需重启）：
# EMBEDDING_API_KEY  /  SILICONFLOW_API_KEY
# EMBEDDING_BASE_URL /  SILICONFLOW_BASE_URL
# EMBEDDING_MODEL
```

##### 单条插入时的向量化

**文件**：`backend/internal/service/question_admin.go:163`

```go
func (s *QuestionAdminService) generateEmbedding(questionID, question string,
    posID uuid.UUID, scene string, difficulty int, tags []string, source string) {

    vec, err := s.embeddingSvc.Embed(ctx, question)  // bge-m3 → 1024维
    if err != nil {
        return  // 失败不阻塞入库
    }

    // 直接 upsert，不做查重（新题不太可能重复，批量去重由 VectorizeAllPending 负责）
    s.vectorSvc.UpsertQuestion(ctx, questionID, vec, &VectorPayload{...})

    // 标记向量化完成
    s.db.Model(&model.InterviewQuestion{}).
        Where("id = ?", questionID).
        Update("vector_status", "vectorized")
}
```

**触发时机**：`BatchInsert` 调用 `Insert` 后，`Insert` 内部异步调用 `generateEmbedding`。

##### 向量去重效果

| 场景 | SHA256 去重（爬虫） | 文本去重（batch-insert） | pgvector 语义去重 |
|------|-------------------|------------------------|-----------------|
| 完全相同的文章 | ✅ | ✅ | ✅ |
| 改了几个字/格式 | ❌ | ❌ | ✅（阈值0.95可捕获） |
| 转载换标题 | ❌ | ❌ | ✅ |
| 同一面经不同人整理 | ❌ | ❌ | ❌（阈值0.95太高） |
| 完全不同但主题相似 | ✅（不重复） | ✅（不重复） | ✅（不重复） |

> **阈值 0.95 说明**：当前设得较高，仅过滤几乎完全相同的题目（如同一篇文章被两个爬虫 session 重复抓取）。如需召回"同一面经不同人改写"这类重复，需调低至 0.85-0.90。但调低也有误杀风险（两道不同的题也可能 cosine > 0.85）。

##### 历史问题

参见 `issus-list.md` 问题6：
- 2026-06-02 发现 `vector_status` 大量 `pending`（1474 题 pending，仅 46 题 vectorized）
- 根因：`embedding.go` 的 `EMBEDDING_BASE_URL` 环境变量未设，回退到默认 `localhost:11434`（旧 Ollama），但 Ollama 未运行
- 修复：配硅基流 API，`VectorizeAllPending` 回填 1365 题，覆盖率从 3% 提升至 96.1%
- 后续：问题17 进一步修复 bge-m3 对 database 领域向量质量差的问题（cosine 从 0.39→0.60）

---

## 五、内容质量门禁

### 5.1 闸1：标题相关性过滤

```
岗位关键词 → 技术词映射 (TECH_TOKEN_MAP)
  ├─ "C++"  → ["c++", "cpp"]
  ├─ "Go"   → ["golang", "go"]
  ├─ "前端"  → ["前端", "frontend", "vue", "react", ...]
  └─ 通用岗位（无技术词匹配）→ 兜底要求含"面经"/"面试"
```

**逻辑**：标题必须命中岗位的技术词，否则判为无关。这样可滤掉"前端offer帮选""练面试一步到位"等标题党/广告帖。单词边界处理：`_token_in_title()` 对纯 ASCII 词（go/java/ai）用正则词边界，避免 "go" 误中 "good/django"。

### 5.2 闸2：岗位相关性评分

```
LLM smart-parse 返回 relevance 1-5
  → relevance >= 3  → 入库
  → relevance < 3   → 丢弃（判为跑题）
```

### 5.3 正文质量

| 检查项 | 规则 | 处理 |
|--------|------|------|
| 正文长度 | < 50 字 | 跳过（无实质内容） |
| 去重 | SHA256 命中 | 跳过（已处理过） |
| smart-parse 出题数 | == 0 | 跳过（LLM 解析失败） |
| relevance | < 3 | 丢弃（闸2过滤） |
| 服务端质量闸 | Go 后端二次过滤 | 返回 skipped 计数 |

---

## 六、数据模型

### 6.1 RawArticle（原始文章）

```python
@dataclass
class RawArticle:
    source: str              # "nowcoder" | "juejin"
    source_id: str           # 源站文章 ID
    title: str
    content: str             # 纯文本/markdown 正文
    url: str                 # 原文链接
    author: str = ""
    publish_time: str = ""
    job_keyword: str = ""    # 匹配的搜索关键词
    crawled_at: str          # ISO 时间戳
```

### 6.2 ParsedQuestion（解析后的题目）

```python
@dataclass
class ParsedQuestion:
    question_text: str
    answer: str = ""                   # LLM 生成的规范参考答案
    key_points: List[str]             # 要点列表
    difficulty: str = "medium"        # easy/medium/hard
    scene: str = "technical_interview" # 面试场景
    category: str = ""                # 岗位分类（LLM 自动划分）
    relevance: int = 3                # 相关性 1-5（闸2过滤用）
    quality: float = 0.0              # 5 维质量均值 0.0-1.0
    quality_dimensions: dict          # 5 维明细
    tags: List[str]                  # 标签
    source_article_url: str = ""     # 原文链接
    source_article_title: str = ""   # 原文标题
```

### 6.3 CrawlResult（统计）

```python
@dataclass
class CrawlResult:
    source: str
    job: str
    articles_found: int = 0
    articles_deduped: int = 0
    questions_parsed: int = 0
    questions_inserted: int = 0
    questions_skipped: int = 0
    errors: List[str]
```


---

## 七、Spider 实现详解

### 7.1 掘金 Spider (`spiders/juejin.py`)

**策略**：优先搜索 API → 搜索空时自动切换到推荐流 + 客户端关键词过滤

```
搜索 API (JuejinSpider)
  ├─ 有 Cookie → POST /search_api/v1/search (key_word, id_type=0)
  │   解析: data[].result_model.article_info
  │   字段名必须是 "key_word"（不是 "query"），早期 bug 根源
  │
  └─ 无 Cookie / 搜索为空 → 推荐流回退
       ├─ POST /recommend_api/v1/article/recommend_all_feed
       ├─ 翻页最多 3 页，每页 20 条
       └─ 客户端标题关键词过滤 (title_matches_keyword)

文章详情 API
  ├─ POST /content_api/v1/article/detail (article_id)
  ├─ 优先取 mark_content (markdown)
  └─ 回退取 content (HTML) → BeautifulSoup 提取纯文本
```

**关键细节**：
- 搜索结果 `result_type != 2` 跳过（非文章类型）
- 推荐流 API 返回的 `brief_content` 也作为 content 暂存
- `_api_post()` 方法使用独立 requests（不继承 Session 的浏览器头），模拟掘金 Web 客户端请求头

### 7.2 牛客 Spider (`spiders/nowcoder.py`)

**策略**：搜索页（游客可用）→ 搜索无结果时退回公开面经板块

```
搜索页 (NowcoderSpider)
  ├─ GET /search/all?query={keyword}&type=all
  ├─ BeautifulSoup 解析 <a href="/discuss/{id}"> 链接
  ├─ 标题命中技术词（闸1）才保留
  └─ 无需 Cookie，游客可访问

公开面经板块（回退）
  ├─ GET /discuss?type=2&order=0&page={1-5}
  ├─ type=2 = 面经板块，无需登录
  ├─ 翻页最多 5 页，连续 2 页无新命中则提前停止
  └─ 标题关键词过滤

文章详情
  ├─ 正文由 JS 异步渲染，藏在 HTML 的 __INITIAL_STATE__ JSON 里
  ├─ 正则提取字段 content/richContent/contentText/richText
  ├─ 逐个字段 json.loads 解码（避免整段 JSON 解析失败）
  └─ 兜底：CSS 选择器 / 全页提取
```

**`__INITIAL_STATE__` 提取**：
```python
_CONTENT_FIELDS = ("content", "richContent", "contentText", "richText")
for field in self._CONTENT_FIELDS:
    for m in re.finditer(r'"%s"\s*:\s*%s' % (field, _JSON_STR), html):
        v = json.loads('"' + m.group(1) + '"')
        if len(v) > len(best):
            best = v
```

---

## 八、Go 后端 API 集成

### 8.1 API 清单

| API | 方法 | 用途 | 超时 |
|-----|------|------|------|
| `/api/v1/admin/login` | POST | 管理员认证（内网直接返回 token，外网两步验证） | 10s |
| `/api/v1/admin/verify-code` | POST | 外网两步验证换 token | 10s |
| `/api/v1/admin/positions` | GET | 读取数据库真实岗位列表 | 10s |
| `/api/v1/admin/questions/smart-parse` | POST | LLM 解析面经文章出题 | 90s |
| `/api/v1/admin/questions/batch-insert` | POST | 批量入库 | 30s |

### 8.2 认证流程

```python
# 内网：直接登录
POST /api/v1/admin/login
  → 响应 { "data": { "token": "jwt..." } }
  → 写入 Authorization: Bearer <token>

# 外网：两步验证
POST /api/v1/admin/login
  → 响应 { "data": { "need_verification": true, "admin_id": "...", "dev_code": "..." } }
POST /api/v1/admin/verify-code?admin_id={id}
  → 响应 { "data": { "token": "jwt..." } }
```

### 8.3 关键词生成

`Importer._generate_keywords()` 按岗位名称自动生成搜索关键词：

```
"C++后端开发" → ["C++后端开发面经", "C++面试", "C++后端面试",
                   "数据库索引", "REST API", "系统设计"]
"Go后端开发"  → ["Go后端开发面经", "Go面试", "Golang面试",
                   "数据库优化", "RESTful设计", "微服务"]
"HR面试"     → ["HR面试面经", "HR面试", "离职原因怎么答",
                   "薪资谈判", "薪资期望", "职业规划"]
```

---

## 九、调度策略

### 9.1 Docker Cron

```dockerfile
# Dockerfile
RUN echo "0 3 * * * cd /app && python main.py >> /proc/1/fd/1 2>&1" | crontab -
CMD ["cron", "-f"]
```

- **定时**：每天北京时间凌晨 3:00
- **随机 jitter**：cron 触发后，main.py 内部 `random.randint(0, 1800)` 秒，避免整点扎堆
- **手动调用**：指定 `--job/--source/--count/--dry-run` 时跳过 jitter

### 9.2 CLI 参数

```
python main.py                          # cron 模式（每岗位每源 3 篇）
python main.py --count 5                # 所有岗位，每源 5 篇
python main.py --job "Go后端开发" --count 10  # 指定岗位
python main.py --source juejin --count 5     # 只跑掘金
python main.py --dry-run --count 3      # 干跑不入库，输出 preview.json
python main.py --config prod.yaml       # 指定配置文件
```

### 9.3 岗位来源

```
岗位优先级：
  1. 数据库真实岗位（importer.fetch_jobs()）— 正式运行的唯一来源
  2. config.yaml extra_jobs（定向主题搜索，如"数据库优化""薪资谈判"）
  3. config.yaml demo_jobs（仅 dry-run 模式且数据库不可达时回退）
```

**关键原则**：正式运行只使用数据库中的真实岗位，保证抓取目标与面试场景一一对应。`demo_jobs` 绝不入库。

---

## 十、部署方案

### 10.1 Docker 部署

```dockerfile
FROM python:3.12-slim

# 安装 cron + curl
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl cron tzdata && rm -rf /var/lib/apt/lists/*

ENV TZ=Asia/Shanghai
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

# 每天北京时间凌晨 3:00
RUN echo "0 3 * * * cd /app && python main.py >> /proc/1/fd/1 2>&1" | crontab -

# 健康检查
HEALTHCHECK --interval=60s --timeout=10s --retries=3 \
    CMD test -f /data/state.json || exit 0

CMD ["cron", "-f"]
```

### 10.2 环境变量

| 变量 | 用途 |
|------|------|
| `ADMIN_USERNAME` | 管理员账号（默认 admin） |
| `ADMIN_PASSWORD` | 管理员密码 |
| `BACKEND_BASE_URL` | Go 后端地址（默认 http://localhost:8080） |
| `JUEJIN_COOKIE` | 掘金 Cookie（可选） |
| `NOWCODER_COOKIE` | 牛客 Cookie（可选） |
| `CRAWLER_STATE_DIR` | state.json 目录（Docker 挂载卷用） |

### 10.3 生产运维

```bash
# 查看容器日志
docker logs interview-crawler

# 手动触发爬取
docker exec interview-crawler python main.py --dry-run --count 2  # 干跑验证
docker exec interview-crawler python main.py --count 3           # 正式运行

# 更新 Cookie（重新部署）
vim /opt/interviewpro/.env
docker compose -f docker-compose.prod.yml up -d crawler
```

### 10.4 监控

| 指标 | 来源 | 说明 |
|------|------|------|
| 汇总统计 | 每日日志末尾 | 找到/去重/解析/入库数量 |
| state.json | 文件存在性 | 健康检查用 |
| 抓取失败 | 日志 ERROR level | URL、源、错误原因 |
| 反爬拦截 | 日志 WARNING/ERROR | 429/403/503 + 验证码 |

---

## 十一、项目结构

```
/home/tommychen/english-learner/crawler/
├── main.py                  # 入口 + 调度循环
├── config.yaml              # 配置文件（含敏感信息，不提交）
├── config.example.yaml      # 示例配置（可提交）
├── models.py                # 数据模型
├── dedup.py                 # SHA256 去重 + state.json 持久化
├── importer.py              # Go 后端 API 客户端
├── spiders/
│   ├── __init__.py
│   ├── base.py              # Spider 基类 + 反爬策略
│   ├── juejin.py            # 掘金 Spider（API 爬虫）
│   └── nowcoder.py          # 牛客 Spider（HTML 解析）
├── requirements.txt         # requests, beautifulsoup4, pyyaml
├── Dockerfile               # python:3.12-slim + cron
├── README.md                # 完整使用文档
├── state.json               # 去重持久化（自动生成）
├── crawler.log              # 运行日志（自动生成）
└── preview.json             # dry-run 输出（自动生成）
---

## 十二、常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 掘金搜索返回 0 篇 | 搜索 API 字段名用错（query 应为 key_word） | 已修复，确认请求体字段 |
| 牛客正文为空 | __INITIAL_STATE__ 结构变化 | 检查 _CONTENT_FIELDS 字段名 |
| smart-parse 报 403 | 外网 IP 被限流 | 从服务器内网运行（docker exec） |
| 解析出 0 题 | 面经正文太短或 LLM 解析质量差 | 先用 --dry-run 看 preview.json |
| Cookie 过期 | 掘金/牛客 Cookie 有效期 7-30 天 | 浏览器重新获取，更新环境变量 |

---

> **参考实现**：`/home/tommychen/english-learner/crawler`（SSH: tommychen@192.168.3.61）
> **技术栈**：Python 3.12 + requests + BeautifulSoup4 + PyYAML
> **后端依赖**：InterviewPro Go 后端（提供 JWT 认证 + smart-parse + batch-insert API）
> **部署**：Docker (python:3.12-slim) + cron 定时触发
