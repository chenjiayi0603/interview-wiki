# InterviewPro 技术栈详解

> 本文档系统讲解 InterviewPro 项目所用技术栈的底层原理、项目用法和面试关联，帮助开发者深入理解每个技术组件，建立完整的知识体系。

**项目地址**: https://gitee.com/chenjiayi/interview-quicker  
**代码目录**: `english-learner/`  
**核心技术**: Go后端 + React Native前端 + AI语音服务

---

## 目录

1. [项目架构总览](#1-项目架构总览)
2. [Go后端技术栈详解](#2-go后端技术栈详解)
3. [AI/语音技术栈详解](#3-ai语音技术栈详解)
4. [前端技术栈详解](#4-前端技术栈详解)
5. [数据库设计](#5-数据库设计)
6. [部署架构](#6-部署架构)
7. [技术栈面试映射表](#7-技术栈面试映射表)
8. [学习路径建议](#8-学习路径建议)

---

## 1. 项目架构总览

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           InterviewPro 系统架构                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                         客户端层 (Client)                             │  │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │  │
│   │   │   iOS    │  │ Android  │  │   H5     │  │  小程序   │            │  │
│   │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘            │  │
│   │        │             │             │             │                   │  │
│   │        └─────────────┴─────────────┴─────────────┘                   │  │
│   │                          React Native + Expo                          │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                         WebSocket (实时) + HTTP (REST)                      │
│                                      │                                      │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                        API Gateway (Nginx)                           │  │
│   │                     负载均衡 + SSL终端 + 限流                           │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                     Go Backend (Gin + GORM)                         │  │
│   │                                                                       │  │
│   │   ┌─────────────────┐     ┌─────────────────┐                         │  │
│   │   │   HTTP Server   │     │  WebSocket Hub  │                         │  │
│   │   │   /api/* 路由   │     │  /ws/interview   │                         │  │
│   │   └────────┬────────┘     └────────┬────────┘                         │  │
│   │            │                       │                                  │  │
│   │            └───────────┬───────────┘                                  │  │
│   │                        │                                              │  │
│   │   ┌────────────────────────────────────────────────────────────────┐  │  │
│   │   │                     Middleware Stack                            │  │  │
│   │   │   JWT认证 → CORS → 日志(Zap) → 限流 → 错误恢复                    │  │  │
│   │   └────────────────────────────────────────────────────────────────┘  │  │
│   │                        │                                              │  │
│   │   ┌────────────────────────────────────────────────────────────────┐  │  │
│   │   │                    Service Layer                                │  │  │
│   │   │                                                                │  │  │
│   │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │  │  │
│   │   │   │ Interview    │  │     User     │  │   Scoring    │       │  │  │
│   │   │   │   Service    │  │   Service    │  │   Service    │       │  │  │
│   │   │   └──────────────┘  └──────────────┘  └──────────────┘       │  │  │
│   │   │                                                                │  │  │
│   │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │  │  │
│   │   │   │  AI Service  │  │Speech Service│  │Pronunciation │       │  │  │
│   │   │   │(ModelFactory)│  │ (STT/TTS)    │  │   Service    │       │  │  │
│   │   │   └──────────────┘  └──────────────┘  └──────────────┘       │  │  │
│   │   └────────────────────────────────────────────────────────────────┘  │  │
│   │                                                                       │  │
│   └───────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│   ┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐ │
│   │     MySQL       │      Redis      │   DeepSeek API  │   阿里云 NLS     │ │
│   │   (持久化)       │   (会话/缓存)    │    (AI对话)      │ (STT/TTS/评测)   │ │
│   └─────────────────┴─────────────────┴─────────────────┴─────────────────┘ │
│                                                                             │
│   ┌─────────────────┐                                                       │
│   │   llama.cpp     │  本地 Qwen 模型 (可选)                                │
│   │  localhost:8080 │                                                       │
│   └─────────────────┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 请求生命周期

一个完整的面试会话从开始到结束的流程如下：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        面试会话完整生命周期                                   │
└─────────────────────────────────────────────────────────────────────────────┘

阶段1: 连接建立
─────────────────────────────────────────────────────────────────────────────
用户打开App → WebSocket连接 → JWT验证 → Hub注册Client → 会话建立

    React Native                          Go Backend
         │                                     │
         │──── WS Connect (w/ JWT Token) ────→│
         │                                     │ → Validate JWT
         │                                     │ → Create Client
         │                                     │ → Register to Hub
         │←──── Connection ACK ───────────────│
         │                                     │

阶段2: 面试开始
─────────────────────────────────────────────────────────────────────────────
客户端发送开始请求 → 创建面试会话 → 生成面试问题 → TTS合成语音 → WebSocket推送

         │──── Start Interview ──────────────→│
         │                                     │ → Create Session (MySQL)
         │                                     │ → Generate Question (AI)
         │                                     │ → TTS Synthesize
         │←──── Question (Text + Audio) ──────│
         │                                     │

阶段3: 实时对话循环 (每个问答轮次)
─────────────────────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   用户录音 → 发送音频Chunk → STT转文字 → 发音评测    → 并发执行              │
│                                       ↓                                    │
│                                  AI评估回答                                  │
│                                       ↓                                    │
│                              生成反馈 + 新问题                               │
│                                       ↓                                    │
│                              TTS合成回复语音                                 │
│                                       ↓                                    │
│                              WebSocket推送结果                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

         │──── audio_chunk (webm) ───────────→│
         │                                     │
         │                                     │ → ffmpeg转码 (webm→wav)
         │                                     │ → STT Transcribe
         │                                     │ → Pronunciation Evaluate (并发)
         │                                     │ → AI Evaluate (并发)
         │                                     │ → Merge Scores
         │                                     │ → TTS Synthesize
         │                                     │
         │←──── evaluation_result (JSON) ─────│
         │←──── audio_response (wav) ─────────│

阶段4: 面试结束
─────────────────────────────────────────────────────────────────────────────
用户结束 → 保存会话记录 → 生成报告 → 清理资源

         │──── End Interview ───────────────→│
         │                                     │ → Save to MySQL
         │                                     │ → Generate Report
         │                                     │ → Close WebSocket
         │←──── Final Report ─────────────────│
```

### 1.3 音频数据流

音频数据在系统中的完整流转过程：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          音频数据流转路径                                     │
└─────────────────────────────────────────────────────────────────────────────┘

客户端                              服务端                                 外部服务
   │                                 │                                        │
   │  ┌──────────────────────────────┴──────────────────────────────┐       │
   │  │                        音频采集与发送                          │       │
   │  └──────────────────────────────────────────────────────────────┘       │
   │  │                                                                   │   │
   │  ▼                                                                   ▼   │
   │ ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐            │
   │ │ 麦克风   │────→│ 编码器   │────→│ 分片器   │────→│WebSocket│            │
   │ │(Recorder│     │(Opus/   │     │(Chunk)  │     │ Sender  │            │
   │ │ PCM)    │     │AAC)     │     │         │     │         │            │
   │ └─────────┘     └─────────┘     └─────────┘     └────┬────┘            │
   │                                                         │                │
   │                                                         ▼                │
   │  ┌──────────────────────────────────────────────────────────────────────┐ │
   │  │                          Go Backend                                  │ │
   │  │                                                                       │ │
   │  │   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │ │
   │  │   │  ffmpeg转码   │     │  STT 服务     │     │  评测服务    │        │ │
   │  │   │webm → pcm/wav│────→│  阿里云/百炼   │────→│  阿里云API   │        │ │
   │  │   └──────────────┘     └──────────────┘     └──────────────┘        │ │
   │  │          │                   │                    │                 │ │
   │  │          │ refText           │ transcript         │ audioScore      │ │
   │  │          ▼                   ▼                    ▼                 │ │
   │  │   ┌──────────────────────────────────────────────────────────────┐  │ │
   │  │   │                    AI 评估服务                                 │  │ │
   │  │   │  DeepSeek / Qwen Local → 5维评分 → JSON结果                  │  │ │
   │  │   └──────────────────────────────────────────────────────────────┘  │ │
   │  │                              │                                    │  │ │
   │  │                              │ evalResult                         │  │ │
   │  │                              ▼                                    │  │ │
   │  │   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     │  │ │
   │  │   │ TTS 合成     │     │ 响应组装      │     │ WebSocket    │     │  │ │
   │  │   │ 文字→语音    │────→│ 合并数据      │────→│ 推送结果      │     │  │ │
   │  │   └──────────────┘     └──────────────┘     └──────┬───────┘     │  │ │
   │  │                                                       │              │  │ │
   │  └───────────────────────────────────────────────────────┼──────────────┘  │
   │                                                          │                 │
   │                                                          ▼                 │
   │                                                   ┌──────────────┐          │
   │                                                   │   客户端     │          │
   │                                                   │ Audio Player │          │
   │                                                   └──────────────┘          │
   │                                                        │                     │
   └────────────────────────────────────────────────────────┘                     │
                                                                                 │
                               阿里云NLS / 百炼 / DeepSeek                       │
```

---

## 2. Go后端技术栈详解

### 2.1 Gin框架

#### 2.1.1 Gin的核心原理

**路由树 (Radix Tree)**

Gin使用压缩前缀树（Radix Tree）实现路由匹配，这是其高性能的关键。

```go
// Radix Tree 的基本原理
// 传统前缀树: 每个字符一个节点
// 压缩前缀树: 合并单字符路径，减少节点数

// 例如路由: /user/:id, /user/list, /user/:id/profile
// 压缩后:
// /user
//   ├── /:id (叶子或继续分叉)
//   │   └── /profile
//   └── /list

// Gin 内部路由节点结构 (简化)
type node struct {
    path      string      // 当前节点的路径片段
    indices   string      // 快速索引子节点的字符
    children  []*node     // 子节点
    handlers  HandlersChain // 该路由的处理函数链
    priority  int         // 优先级（用于排序）
}
```

**Radix Tree 的查找复杂度**: O(k)，k是路径长度，与路由数量无关。

**中间件洋葱模型**

Gin采用经典的洋葱模型执行中间件：

```go
// 中间件执行顺序
func MiddlewareChain(c *Context) {
    // 1. 第一个中间件开始
    fmt.Println("Middleware 1 - Before")
    
    c.Next() // 调用下一个中间件/处理器
    
    // 4. 控制流返回第一个中间件
    fmt.Println("Middleware 1 - After")
}

// 最终处理器
func Handler(c *Context) {
    // 2. 进入业务逻辑
    fmt.Println("Handler")
    c.Next() // 调用后续处理器（如果有）
    // 3. 返回前
}

// 完整调用链: M1-Before → M2-Before → Handler → M2-After → M1-After
```

**Context对象池**

Gin使用`sync.Pool`复用Context对象，减少GC压力：

```go
// Gin的Context池机制
var contextPool = sync.Pool{
    New: func() interface{} {
        return &Context{}
    },
}

func AcquireContext() *Context {
    return contextPool.Get().(*Context)
}

func ReleaseContext(c *Context) {
    // 重置字段
    c.Reset()
    contextPool.Put(c)
}
```

#### 2.1.2 InterviewPro中的路由设计

在项目中的实现位于 `english-learner/backend/internal/handler/`：

```go
// interview.go - 面试相关路由
func SetupRouter(r *gin.Engine) {
    // REST API 路由组
    api := r.Group("/api")
    {
        // 面试管理
        api.POST("/interview/start", handler.StartInterview)
        api.POST("/interview/end", handler.EndInterview)
        api.GET("/interview/:id/report", handler.GetReport)
        api.GET("/interview/history", handler.GetHistory)
        
        // 用户管理
        api.POST("/user/register", handler.Register)
        api.POST("/user/login", handler.Login)
    }
    
    // WebSocket 路由
    r.GET("/ws/interview", wsHandler.HandleWebSocket)
}
```

**路由分组的好处**：
- 统一前缀管理（`/api`）
- 中间件作用域控制
- 代码组织清晰

#### 2.1.3 中间件链实现

在项目中的中间件实现位于 `english-learner/backend/internal/middleware/` 和 `main.go`：

```go
// main.go 中的中间件配置
func setupMiddleware(r *gin.Engine) {
    // 1. 恢复Panic，防止服务器崩溃
    r.Use(gin.Recovery())
    
    // 2. CORS中间件，允许跨域
    r.Use(corsMiddleware())
    
    // 3. JWT认证中间件（保护/api路由）
    r.Use("/api", jwtAuthMiddleware())
    
    // 4. Zap日志中间件，记录请求
    r.Use(zapLoggerMiddleware())
    
    // 5. 限流中间件
    r.Use(rateLimitMiddleware())
}

// JWT认证中间件实现
func jwtAuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.JSON(401, gin.H{"error": "missing token"})
            c.Abort()
            return
        }
        
        // 解析JWT
        claims, err := parseToken(token)
        if err != nil {
            c.JSON(401, gin.H{"error": "invalid token"})
            c.Abort()
            return
        }
        
        // 将用户信息存入Context
        c.Set("user_id", claims.UserID)
        c.Next()
    }
}
```

#### 2.1.4 与标准库net/http的关系

Gin是对`net/http`的封装，不是替代：

```
net/http                    Gin
─────────────────────────────────────
ServeHTTP(w, r)      →      HandlerFunc(c *Context)
ServeMux             →      Radix Tree 路由
ResponseWriter       →      Context.Writer
Request              →      Context.Request
手动读取Body          →      Context.Binding / Context.ShouldBind
```

**Gin的核心价值**：
1. 更快的路由匹配（Radix Tree vs 线性遍历）
2. 更优雅的API（Context vs 原始Request/Response）
3. 内置中间件系统
4. 参数绑定和验证

#### 2.1.5 面试角度：Gin vs Echo vs Fiber

| 维度 | Gin | Echo | Fiber |
|------|-----|------|-------|
| 底层 | net/http | net/http | Fasthttp |
| 路由 | Radix Tree | Radix Tree | Radix Tree |
| 性能 | ★★★★☆ | ★★★★☆ | ★★★★★ |
| 生态 | ★★★★★ | ★★★★☆ | ★★★☆☆ |
| 中间件 | 丰富 | 丰富 | 中等 |
| 团队熟悉度 | 高 | 中 | 中 |
| **适用场景** | **通用Web/API** | 高性能API | 超高并发 |

**面试标准回答**：
> "Gin是目前Go生态最成熟的Web框架，选择它主要考虑：1) 社区活跃，文档完善；2) 中间件生态丰富；3) 与项目技术栈一致，团队熟悉。Echo性能更好但生态稍弱，Fiber极致性能但学习曲线陡峭。"

---

### 2.2 GORM

#### 2.2.1 GORM的核心原理

**Session模式**

GORM的所有操作都基于Session，确保数据库连接的复用和事务的一致性：

```go
// Session 是 GORM 的核心概念
// 每次 DB 调用实际是创建/复用 Session

// 创建Session（带配置）
session := db.Session(&gorm.Session{
    FullSaveAssociations: false,
    Logger:               logger.Default.LogMode(logger.Info),
})

// 在Session中执行操作
session.Create(&user)

// Session的生命周期
// Session创建 → 配置生效 → 执行操作 → Session结束
// 不会影响原始 db 对象
```

**Callback链**

GORM使用Callback链在CRUD操作前后注入逻辑：

```go
// Callback 执行顺序
// Create: BeforeSave → BeforeCreate → Create → AfterCreate → AfterSave
// Update: BeforeSave → BeforeUpdate → Update → AfterUpdate → AfterSave
// Delete: BeforeDelete → Delete → AfterDelete
// Query:  BeforeQuery → Query → AfterQuery

// GORM内置Callback示例
db.Callback().Create().Before("gorm:create").Register("my:before:create", func(scope *gorm.Scope) {
    // 在Create前执行
    if scope.FieldByName("CreatedAt") != nil {
        scope.SetColumn("CreatedAt", time.Now())
    }
})
```

**SQL构建器**

GORM的链式调用最终生成SQL：

```go
// 链式调用
result := db.Where("age > ?", 18).
            Where("name LIKE ?", "%John%").
            Order("created_at DESC").
            Limit(10).
            Offset(0).
            Find(&users)

// 等价SQL:
// SELECT * FROM users WHERE age > 18 AND name LIKE '%John%' 
// ORDER BY created_at DESC LIMIT 10 OFFSET 0
```

#### 2.2.2 InterviewPro的模型定义

在项目中的实现位于 `english-learner/backend/internal/model/model.go`：

```go
// User 用户模型
type User struct {
    ID           uint      `gorm:"primaryKey"`
    Username     string    `gorm:"size:50;not null;uniqueIndex"`
    Email        string    `gorm:"size:100;not null;uniqueIndex"`
    PasswordHash string    `gorm:"size:255;not null"`
    CreatedAt    time.Time
    UpdatedAt    time.Time
    Interviews   []Interview `gorm:"foreignKey:UserID"`
}

// Interview 面试会话模型
type Interview struct {
    ID          uint      `gorm:"primaryKey"`
    UserID      uint      `gorm:"index;not null"`
    Topic       string    `gorm:"size:100"`        // 面试主题
    Status      string    `gorm:"size:20;default:'active'"` // active/completed
    StartedAt   time.Time
    EndedAt     *time.Time
    Score       float64   `gorm:"type:decimal(5,2)"` // 综合评分
    CreatedAt   time.Time
    UpdatedAt   time.Time
    Messages    []Message `gorm:"foreignKey:InterviewID"`
}

// Message 消息记录（问答对）
type Message struct {
    ID           uint      `gorm:"primaryKey"`
    InterviewID  uint      `gorm:"index;not null"`
    Role         string    `gorm:"size:10"`         // "user" or "assistant"
    Content      string    `gorm:"type:text"`
    AudioURL     string    `gorm:"size:255"`
    Scores       string    `gorm:"type:json"`       // 5维评分JSON
    CreatedAt    time.Time
}
```

#### 2.2.3 查询优化

**预加载（Preload）解决N+1问题**：

```go
// N+1 问题示例
users, _ := db.Find(&users)
for _, user := range users {
    // 每个user都会触发一次查询获取interviews
    fmt.Println(user.Interviews)  // N+1!
}

// 预加载解决方案
db.Preload("Interviews").Find(&users)  // 2次查询: 1次用户 + 1次所有面试

// 多层预加载
db.Preload("Interviews.Messages").Find(&users)

// 条件预加载
db.Preload("Interviews", "status = ?", "completed").Find(&users)

// Jion预加载（更高效）
db.Joins("JOIN interviews ON interviews.user_id = users.id").
    Where("interviews.status = ?", "completed").
    Find(&users)
```

**批量操作**：

```go
// 批量创建
users := []User{{Name: "u1"}, {Name: "u2"}, {Name: "u3"}}
db.Create(&users)  // 一条 INSERT ... VALUES (...), (...), (...)

// 批量更新
db.Model(&User{}).Where("age > ?", 30).Update("status", "inactive")

// 事务批量操作
err := db.Transaction(func(tx *gorm.DB) error {
    for _, item := range items {
        if err := tx.Create(&item).Error; err != nil {
            return err
        }
    }
    return nil
})
```

#### 2.2.4 GORM的坑和最佳实践

```go
// 坑1: 指针类型默认值
type User struct {
    Age  int     `gorm:"default:0"`    // ⚠️ 0会被当作有效值
    Name *string `gorm:"default:null"`  // ✅ 推荐用指针
}

// 坑2: Update时不会自动更新updated_at
user.Name = "new name"
db.Save(&user)    // ✅ Save会更新
db.Model(&user).Update("name", "new name")  // ⚠️ 不会更新updated_at
// 正确做法:
db.Model(&user).Select("name", "updated_at").Updates(map[string]interface{}{
    "name":       "new name",
    "updated_at": time.Now(),
})

// 坑3: First/Find空结果
result := db.First(&user)
if result.Error == gorm.ErrRecordNotFound {
    // 处理未找到
}
// ⚠️ 不要这样写:
// if result.RowsAffected == 0 { ... } // 可能在Error之前检查

// 坑4: 全局禁用默认事务
db, _ := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
    SkipDefaultTransaction: true,  // ⚠️ 影响Write操作
})
```

#### 2.2.5 面试角度：ORM vs 原生SQL

**ORM优势**：
- 开发效率高，代码可读性好
- 跨数据库支持（MySQL/PostgreSQL/SQLite）
- 防止SQL注入
- 自动迁移和模型同步

**ORM劣势**：
- 复杂查询不如SQL直观
- 性能开销（大量数据时）
- 学习曲线（理解ORM语义）

**面试标准回答**：
> "GORM在项目中主要用于标准CRUD操作和简单查询。对于面试场景，5维评分生成、用户历史记录查询这类操作用GORM非常方便。涉及复杂报表或性能关键查询（如大量数据聚合）我会考虑原生SQL或者使用GORM的Raw方法混合编程。"

---

### 2.3 gorilla/websocket

#### 2.3.1 WebSocket协议原理

**握手过程（HTTP Upgrade）**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              WebSocket 握手                                  │
└─────────────────────────────────────────────────────────────────────────────┘

客户端 → 服务端 (HTTP GET 请求)
─────────────────────────────────────────────────────────────────────────────
GET /ws/interview HTTP/1.1
Host: api.interviewpro.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://interviewpro.com

服务端 → 客户端 (101 Switching Protocols)
─────────────────────────────────────────────────────────────────────────────
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**帧格式**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           WebSocket 帧结构                                   │
└─────────────────────────────────────────────────────────────────────────────┘

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┬─┬─┬─┬─┬─┬─┬─┼─┬─┬─┬─┬─┬─┬─┬─┬─┼─┬─┬─┬─┬─┬─┬─┬─┬─┼─┬─┬─┬─┬─┬─┬─┬─┤
│F|R|R|R|  Opcode   │M|     Payload len    │    Extended payload     │
│I|S|S|S|   (4)     │A|         (7)        │        length           │
│N│V│V│V│            │S│                     │          (16/64)       │
│ │1│2│3│            │K│                     │                          │
├─┴─┴─┴─┴────────────┴─┴─────────────────────┴──────────────────────────┤
│                               │         │              │                │
│                               │         │              │                │
├───────────────────────────────┼─────────┴──────────────┴────────────────┤
│                         Payload Data                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Opcode:
  0x0 = Continuation frame
  0x1 = Text frame
  0x2 = Binary frame
  0x8 = Close
  0x9 = Ping
  0xA = Pong
```

**控制帧（Ping/Pong）**：

```go
// Ping/Pong 用于心跳保活
// 服务端或客户端可以主动发送 Ping
// 对方必须回复 Pong

// gorilla/websocket 自动处理 Ping/Pong
// 但需要设置读超时
conn.SetReadDeadline(time.Now().Add(60 * time.Second))
conn.SetPongHandler(func(appData string) error {
    // 收到Ping后自动发送Pong，这里可以更新心跳时间
    return nil
})

// 定期发送Ping
go func() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    for {
        <-ticker.C
        if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
            return
        }
    }
}()
```

#### 2.3.2 InterviewPro的Hub+Client架构

在项目中的实现位于 `english-learner/backend/internal/ws/`：

```go
// hub.go - WebSocket Hub（中央协调器）
type Hub struct {
    clients    map[*Client]bool    // 注册的客户端
    broadcast  chan []byte         // 广播消息队列
    register   chan *Client        // 注册频道
    unregister chan *Client         // 注销频道
    mutex      sync.RWMutex         // 并发保护
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mutex.Lock()
            h.clients[client] = true
            h.mutex.Unlock()
            log.Info("Client registered", zap.Int("total", len(h.clients)))
            
        case client := <-h.unregister:
            h.mutex.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mutex.Unlock()
            
        case message := <-h.broadcast:
            h.mutex.RLock()
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mutex.RUnlock()
        }
    }
}
```

```go
// client.go - WebSocket Client（单个连接）
type Client struct {
    hub            *Hub
    conn           *websocket.Conn
    send           chan []byte    // 发送队列
    userID         uint
    interviewID    uint
    audioBuffer    []byte         // 音频数据缓冲
}

func (c *Client) handleAudioChunk(data []byte) error {
    // 1. 音频数据累积
    c.audioBuffer = append(c.audioBuffer, data...)
    
    // 2. 检查是否收到完整帧（根据业务逻辑判断）
    if isCompleteFrame(c.audioBuffer) {
        // 3. STT 转换
        text, err := c.sttService.Transcribe(c.audioBuffer)
        if err != nil {
            return err
        }
        
        // 4. 并发执行发音评测和AI评估
        var wg sync.WaitGroup
        wg.Add(2)
        
        go func() {
            defer wg.Done()
            c.evaluateAndRespond(text)  // AI评估 + TTS
        }()
        
        go func() {
            defer wg.Done()
            c.pronunciationSvc.Evaluate(c.audioBuffer, text)  // 发音评测
        }()
        
        wg.Wait()
        
        // 5. 清空缓冲区
        c.audioBuffer = nil
    }
    
    return nil
}

func (c *Client) evaluateAndRespond(text string) {
    // 1. 调用AI评估
    evalResult, err := c.aiService.EvaluateAnswer(c.question, text)
    if err != nil {
        c.sendError(err)
        return
    }
    
    // 2. 发送评估结果
    c.sendToClient([]byte(evalResult))
    
    // 3. TTS合成并发送
    c.synthesizeAndSend(evalResult)
}

func (c *Client) sendToClient(data []byte) {
    select {
    case c.send <- data:
    default:
        // 队列满，关闭连接
        log.Warn("Send buffer full, closing client")
        c.hub.unregister <- c
    }
}
```

#### 2.3.3 生产级WebSocket注意事项

**并发写保护**：

```go
// ⚠️ 错误：多个goroutine同时写会冲突
go func() { c.conn.WriteMessage(...) }()
go func() { c.conn.WriteMessage(...) }()

// ✅ 正确：使用互斥锁或通道序列化
type SafeWriter struct {
    conn   *websocket.Conn
    send   chan []byte
    mutex  sync.Mutex
}

func (w *SafeWriter) Write(msg []byte) error {
    w.mutex.Lock()
    defer w.mutex.Unlock()
    return w.conn.WriteMessage(websocket.TextMessage, msg)
}

// 或者用channel（推荐）
func (c *Client) writePump() {
    for message := range c.send {
        if err := c.conn.WriteMessage(websocket.TextMessage, message); err != nil {
            return
        }
    }
}
```

**心跳保活机制**：

```go
func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()
    
    c.conn.SetReadLimit(512 * 1024)  // 最大帧大小
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    
    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway) {
                log.Printf("WebSocket error: %v", err)
            }
            break
        }
        
        // 处理消息
        c.handleMessage(message)
        
        // 重置读超时
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    }
}
```

**消息缓冲和背压**：

```go
const (
    sendBufferSize = 256
    maxMessageSize = 512 * 1024  // 512KB
)

// 使用有缓冲channel作为消息队列
send := make(chan []byte, sendBufferSize)

// 背压处理
select {
case c.send <- message:
    // 正常发送
default:
    // 队列已满，丢弃或返回错误
    log.Warn("Send buffer full, dropping message")
    return ErrBufferFull
}
```

#### 2.3.4 面试角度：WebSocket vs SSE vs Long Polling

| 维度 | WebSocket | SSE | Long Polling |
|------|-----------|-----|--------------|
| 方向 | 全双工 | 单向（服务端→客户端） | 半双工 |
| 协议 | ws:// | text/event-stream | HTTP |
| 连接数 | 1 | 1 | 多个（轮询时） |
| 防火墙 | 可能有 | 友好 | 友好 |
| 重连 | 手动实现 | 自动 | 自动 |
| 兼容性 | 现代浏览器 | IE不支持 | 全兼容 |
| **适用场景** | **实时双向（聊天、游戏）** | **推送、监控** | **简单轮询** |

**InterviewPro选型理由**：
> "面试场景需要：1) 客户端实时发送音频流；2) 服务端实时推送评估结果；3) 服务端推送TTS语音。这是典型的双向实时通信，所以选择WebSocket。如果只是单向推送（如面试结果通知），SSE更简单。"

---

### 2.4 Redis (go-redis)

#### 2.4.1 Redis在InterviewPro中的用途

在项目中主要用于：

```go
// 1. 会话存储（面试进行中的状态）
sessionKey := fmt.Sprintf("interview:session:%d", sessionID)
redis.Set(ctx, sessionKey, sessionData, 30*time.Minute)

// 2. 用户Token缓存
tokenKey := fmt.Sprintf("user:token:%s", token)
redis.Set(ctx, tokenKey, userID, 24*time.Hour)

// 3. 限流计数器
rateLimitKey := fmt.Sprintf("ratelimit:%s:%d", ip, time.Now().Minute())
redis.Incr(ctx, rateLimitKey)
redis.Expire(ctx, rateLimitKey, time.Minute)

// 4. 分布式锁
lockKey := fmt.Sprintf("lock:interview:%d", interviewID)
locked, _ := redis.SetNX(ctx, lockKey, "1", 10*time.Second).Result()
```

#### 2.4.2 数据结构选型

| 场景 | 数据结构 | 示例 |
|------|----------|------|
| 会话状态 | String/JSON | `SET session:123 '{"status":"active","q":3}'` |
| 用户画像 | Hash | `HSET user:123 name "John" age "25"` |
| 排行榜 | ZSet | `ZADD leaderboard 100 "user:123"` |
| 标签集合 | Set | `SADD user:123:interests "tech" "ai"` |
| 最新消息 | List | `LPUSH chat:123 "msg1"`, `LTRIM chat:123 0 99` |
| 延迟队列 | ZSet | `ZADD delay:queue timestamp "task:1"` |

#### 2.4.3 go-redis的Pipeline和事务

**Pipeline（批量优化）**：

```go
// 普通方式：N次RTT
for _, key := range keys {
    val, _ := redis.Get(ctx, key).Result()
    // ...
}

// Pipeline：1次RTT
pipe := redis.Pipeline()
cmds := make([]*redis.StringCmd, len(keys))
for i, key := range keys {
    cmds[i] = pipe.Get(ctx, key)
}
_, err := pipe.Exec(ctx)  // 一次性执行所有命令
for _, cmd := range cmds {
    val, _ := cmd.Result()
    // ...
}
```

**TX事务**：

```go
// 事务模式（MULTI/EXEC）
pipe := redis.TxPipeline()
pipe.Incr(ctx, "counter")
pipe.Decr(ctx, "counter")
_, err := pipe.Exec(ctx)

// WATCH（乐观锁）
watchErr := redis.Watch(ctx, func(tx *redis.Tx) error {
    n, _ := tx.Get(ctx, "counter").Int()
    _, err := tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
        pipe.Set(ctx, "counter", n+1, 0)
        return nil
    })
    return err
}, "counter")
```

#### 2.4.4 面试角度：Redis缓存策略

**缓存三兄弟**：

```
缓存穿透：查询不存在的数据 → 布隆过滤器 / 缓存空值
缓存击穿：热点key过期瞬间 → 互斥锁 / 永不过期+异步更新
缓存雪崩：大量key同时过期 → 随机TTL / 永不过期+版本号
```

**面试标准回答**：
> "InterviewPro中Redis主要用于：1) 会话状态存储（String），面试中断后可以恢复；2) JWT token缓存，减少DB查询；3) 接口限流（计数器）。面试场景并发量可控，没有涉及特别复杂的缓存策略，主要保证key有合理的TTL防止内存膨胀。"

---

### 2.5 Viper配置管理

#### 2.5.1 配置层级

Viper遵循严格的配置优先级：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Viper 配置加载优先级                                 │
│                                                                             │
│  高 ────────────────────────────────→ 低                                   │
│                                                                             │
│  命令行Flag → 环境变量 → config.yaml → 默认值                               │
│                                                                             │
│  优先级高的会覆盖优先级低的配置                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.5.2 InterviewPro的配置结构

在项目中的实现位于 `english-learner/backend/internal/config/config.go`：

```go
// config.go
type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
    JWT      JWTConfig
    AI       AIConfig
    Speech   SpeechConfig
}

type ServerConfig struct {
    Host string
    Port int
}

type DatabaseConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
    MaxIdle  int
    MaxOpen  int
}

type AIConfig struct {
    Provider   string  // "deepseek" or "qwen_local"
    DeepSeek   DeepSeekConfig
    QwenLocal  QwenLocalConfig
}

type SpeechConfig struct {
    AliyunKey    string
    AliyunSecret string
    AppKey       string
    Bailian     BailianConfig
}
```

```go
// 配置加载
func LoadConfig(path string) (*Config, error) {
    viper.SetConfigFile(path)
    viper.SetConfigType("yaml")
    
    // 读取配置文件
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    
    // 绑定环境变量
    viper.AutomaticEnv()
    viper.SetEnvPrefix("INTERVIEW")  // 前缀：INTERVIEW_DB_HOST
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    
    var cfg Config
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, err
    }
    
    return &cfg, nil
}
```

#### 2.5.3 热更新原理

```go
// 配置文件热更新
viper.WatchConfig()
viper.OnConfigChange(func(e fsnotify.Event) {
    log.Info("Config file changed", zap.String("name", e.Name))
    
    // 重新读取配置
    var newCfg Config
    viper.Unmarshal(&newCfg)
    
    // 通知相关组件重新加载
    configChanged.Notify(newCfg)
})
```

#### 2.5.4 面试角度

**面试标准回答**：
> "Viper是Go生态最流行的配置管理库，核心价值在于：1) 支持多种配置格式（YAML/TOML/JSON等）；2) 环境变量自动绑定；3) 配置热更新。在项目中，所有敏感信息（API Key、数据库密码）都通过环境变量注入，配置文件只放非敏感配置。"

---

### 2.6 Zap日志

#### 2.6.1 Zap为什么快

Zap是Uber开源的高性能日志库，相比logrus快10倍：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Zap 性能优化点                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 零分配（Zero Allocation）                                                │
│     ───────────────────────────────                                        │
│     传统日志：fmt.Sprintf("%s+%d", msg, num)  →  每次分配字符串               │
│     Zap日志：logger.Info("msg", zap.Int("num", 10))  →  使用 sync.Pool     │
│                                                                             │
│  2. 类型安全（Compile-time Checks）                                          │
│     ───────────────────────────────                                        │
│     zap.Int() 在编译时检查类型，运行时零反射                                  │
│                                                                             │
│  3. 结构化日志（JSON格式）                                                   │
│     ───────────────────────────────                                        │
│     {"level":"info","ts":1234567890,"msg":"request","method":"GET"}         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.6.2 InterviewPro的日志规范

在项目中的实现：

```go
// main.go - 初始化Logger
func initLogger() *zap.Logger {
    config := zap.Config{
        Level:       zap.NewAtomicLevelAt(zap.InfoLevel),
        Development: false,
        Encoding:    "json",  // 生产环境用JSON，便于收集
        EncoderConfig: zap.EncoderConfig{
            TimeKey:        "ts",
            LevelKey:       "level",
            NameKey:        "logger",
            CallerKey:      "caller",
            MessageKey:     "msg",
            StacktraceKey:  "stacktrace",
            LineEnding:     zap.DefaultLineEnding,
            EncodeLevel:    zap.LowercaseLevelEncoder,
            EncodeTime:     zap.ISO8601TimeEncoder,
            EncodeDuration: zap.SecondsDurationEncoder,
        },
        OutputPaths:      []string{"logs/backend.log", "stdout"},
        ErrorOutputPaths: []string{"logs/error.log"},
    }
    
    logger, _ := config.Build()
    return logger
}

// 使用示例
logger.Info("Server started",
    zap.String("host", cfg.Server.Host),
    zap.Int("port", cfg.Server.Port),
    zap.String("ai_provider", cfg.AI.Provider),
)

// 记录请求日志中间件
func zapLoggerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        
        c.Next()
        
        logger.Info("request",
            zap.String("method", c.Request.Method),
            zap.String("path", path),
            zap.Int("status", c.Writer.Status()),
            zap.Duration("latency", time.Since(start)),
            zap.String("client_ip", c.ClientIP()),
        )
    }
}
```

#### 2.6.3 日志级别和采样策略

```go
// 日志级别
DebugLevel  // 调试信息，生产环境关闭
InfoLevel   // 一般信息
WarnLevel   // 警告
ErrorLevel  // 错误
DPanicLevel // 开发环境panic
PanicLevel  // 记录panic后panic
FatalLevel  // 记录后os.Exit(1)

// 采样策略（高频日志场景）
sampler := &zap.Sampler{
    Initial:    100,    // 前100条必定记录
    Thereafter: 10,     // 之后每10条记录1条
}

// 高频日志示例
highFreqLogger, _ := config.Build(
    zap.WrapCore(func(core zapcore.Core) zapcore.Core {
        return zapcore.NewSampler(core, time.Second, 100, 10)
    }),
)
```

#### 2.6.4 面试角度

**面试标准回答**：
> "选择Zap是因为它是目前Go生态性能最好的日志库。在InterviewPro中，所有日志都是结构化JSON格式，便于后续ELK/Graylog收集和查询。日志内容包含：请求ID、用户ID、接口路径、响应状态、耗时等关键信息。日志级别在生产环境设为Info，避免刷屏。"

---

## 3. AI/语音技术栈详解

### 3.1 模型工厂模式

#### 3.1.1 策略模式在model_factory.go中的实现

在项目中的实现位于 `english-learner/backend/internal/service/model_factory.go`：

```go
// 模型提供者接口（策略模式的策略）
type ModelProvider interface {
    Generate(systemPrompt, userPrompt string) (string, error)
    GetName() string
}

// DeepSeek模型实现
type DeepSeekModel struct {
    apiKey   string
    baseURL  string
    model    string
    client   *http.Client
}

func (d *DeepSeekModel) Generate(systemPrompt, userPrompt string) (string, error) {
    url := fmt.Sprintf("%s/chat/completions", d.baseURL)
    
    payload := map[string]interface{}{
        "model": d.model,
        "messages": []map[string]string{
            {"role": "system", "content": systemPrompt},
            {"role": "user", "content": userPrompt},
        },
        "temperature": 0.7,
    }
    
    body, _ := json.Marshal(payload)
    req, _ := http.NewRequest("POST", url, bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", fmt.Sprintf("Bearer %s", d.apiKey))
    
    resp, err := d.client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    
    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)
    
    choices := result["choices"].([]interface{})
    message := choices[0].(map[string]interface{})["message"].(map[string]interface{})
    return message["content"].(string), nil
}

func (d *DeepSeekModel) GetName() string {
    return "DeepSeek"
}

// Qwen本地模型实现
type QwenLocalModel struct {
    apiURL      string
    temperature float64
}

func (q *QwenLocalModel) Generate(systemPrompt, userPrompt string) (string, error) {
    url := fmt.Sprintf("%s/completion", q.apiURL)
    
    // llama.cpp兼容格式
    prompt := fmt.Sprintf("### System:\n%s\n\n### User:\n%s\n\n### Assistant:", 
        systemPrompt, userPrompt)
    
    payload := map[string]interface{}{
        "prompt": prompt,
        "temperature": q.temperature,
        "stop": []string{"###"},
    }
    
    // HTTP调用localhost:8080
    // ...
    return result, nil
}

func (q *QwenLocalModel) GetName() string {
    return "Qwen-Local"
}
```

```go
// 工厂函数（策略选择的上下文）
func GetModelProvider() (ModelProvider, error) {
    provider := os.Getenv("AI_PROVIDER")
    
    switch provider {
    case "deepseek":
        return NewDeepSeekModel(
            os.Getenv("DEEPSEEK_API_KEY"),
            os.Getenv("DEEPSEEK_BASE_URL"),
        ), nil
        
    case "qwen_local":
        return NewQwenLocalModel(
            os.Getenv("QWEN_LOCAL_URL"),
        ), nil
        
    default:
        return nil, fmt.Errorf("unknown AI provider: %s", provider)
    }
}
```

#### 3.1.2 5维评分体系

在项目中的实现位于 `english-learner/backend/internal/service/ai.go`：

```go
// 评估结果结构
type EvaluationResult struct {
    Fluency      float64 `json:"fluency"`      // 流畅度 0-100
    Grammar      float64 `json:"grammar"`      // 语法 0-100
    Vocabulary   float64 `json:"vocabulary"`   // 词汇 0-100
    Content      float64 `json:"content"`      // 内容 0-100
    Pronunciation float64 `json:"pronunciation"` // 发音 0-100（来自评测服务）
    Overall      float64 `json:"overall"`      // 综合评分
    Feedback     string  `json:"feedback"`     // 详细反馈
}

func (s *AIService) EvaluateAnswer(question, answer string) (*EvaluationResult, error) {
    // 获取模型提供者
    model, err := GetModelProvider()
    if err != nil {
        return nil, err
    }
    
    // 构建评分Prompt
    systemPrompt := `你是一位专业的英语面试评分官。请根据以下标准评估用户的回答：
1. Fluency (流畅度): 表达是否流畅自然
2. Grammar (语法): 语法是否正确
3. Vocabulary (词汇): 词汇是否丰富恰当
4. Content (内容): 内容是否切题、有深度

请以JSON格式返回评分结果，包含0-100的分数和简要反馈。`
    
    userPrompt := fmt.Sprintf("面试问题: %s\n\n用户回答: %s\n\n请评分：", question, answer)
    
    // 调用AI获取评分
    response, err := model.Generate(systemPrompt, userPrompt)
    if err != nil {
        return nil, err
    }
    
    // 解析JSON结果
    result := &EvaluationResult{}
    json.Unmarshal([]byte(response), result)
    
    // 计算综合评分（加权平均）
    result.Overall = result.Fluency*0.2 + result.Grammar*0.25 + 
                     result.Vocabulary*0.2 + result.Content*0.35
    
    return result, nil
}
```

#### 3.1.3 面试角度：设计模式在AI应用中的实践

**策略模式的价值**：
1. **运行时切换**：通过环境变量切换DeepSeek/Qwen，无需修改调用代码
2. **隔离变化**：模型API差异封装在各自实现中
3. **扩展性**：新增模型只需实现接口，无需修改现有代码

**工厂模式的价值**：
1. **统一入口**：调用方只需调用`GetModelProvider()`
2. **配置驱动**：通过环境变量决定创建哪个实例
3. **依赖注入**：便于测试时替换为Mock实现

**面试标准回答**：
> "模型工厂是策略模式的典型应用。在面试评分场景中，我支持DeepSeek API（云端能力强）和Qwen Local（隐私/离线）两种模式，通过AI_PROVIDER环境变量切换。后续规划用Eino框架重构，统一到ChatModel接口，同时获得流式响应能力。"

---

### 3.2 阿里云语音服务

#### 3.2.1 STT原理：实时语音识别

阿里云NLS（智能语音交互）提供实时语音识别服务：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        阿里云NLS STT 架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  客户端                          服务端                        阿里云NLS      │
│    │                              │                              │         │
│    │  1. 获取Token                │                              │         │
│    │──────────────────────────────┼──────────────────────────────┼───────→│ │
│    │                              │                              │         │
│    │  2. WebSocket连接            │                              │         │
│    │──────────────────────────────┼──────────────────────────────┼───────→│ │
│    │                              │  ←────── 握手成功 ────────────│         │
│    │                              │                              │         │
│    │  3. 发送音频流 (PCM/wav)     │                              │         │
│    │══════════════════════════════╪══════════════════════════════╪═══════→│ │
│    │                              │                              │         │
│    │  4. 接收实时识别结果          │                              │         │
│    │←─────────────────────────────╫──────────────────────────────╫────────│ │
│    │                              │                              │         │
│    │  5. 关闭连接                 │                              │         │
│    │──────────────────────────────┼──────────────────────────────┼───────→│ │
│    │                              │                              │         │
│    └──────────────────────────────┴──────────────────────────────┴─────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

在项目中的实现位于 `english-learner/backend/internal/service/aliyun_speech.go`：

```go
// Aliyun STT 配置
type AliyunSTT struct {
    accessKey  string
    secretKey  string
    appKey    string
    region    string
}

// Transcribe 实时语音转文字
func (a *AliyunSTT) Transcribe(ctx context.Context, audioData []byte) (string, error) {
    // 1. 创建NLS客户端
    client := nls.NewNlsClient(a.accessKey, a.secretKey)
    
    // 2. 创建实时识别请求
    request := &nls.TranscribeRequest{
        AppKey:    a.appKey,
        Format:    "pcm",
        SampleRate: 16000,
        EnablePunctuation: true,
    }
    
    // 3. 打开WebSocket连接
    conn, err := client.CreateTranscribeConn(request)
    if err != nil {
        return "", err
    }
    defer conn.Close()
    
    // 4. 发送音频数据
    err = conn.Send(audioData)
    if err != nil {
        return "", err
    }
    
    // 5. 接收识别结果
    result := &nls.TranscribeResult{}
    err = conn.Receive(result)
    if err != nil {
        return "", err
    }
    
    return result.Text, nil
}
```

#### 3.2.2 TTS原理：文本转语音

```go
// Aliyun TTS 配置
type AliyunTTS struct {
    accessKey  string
    secretKey  string
    appKey    string
    voice     string  // 音色选择
}

// Synthesize 文本转语音
func (a *AliyunTTS) Synthesize(ctx context.Context, text string) ([]byte, error) {
    // 1. 创建NLS客户端
    client := nls.NewNlsClient(a.accessKey, a.secretKey)
    
    // 2. 创建语音合成请求
    request := &nls.SynthesizeRequest{
        AppKey:    a.appKey,
        Text:      text,
        Voice:     a.voice,
        Format:    "wav",
        SampleRate: 16000,
    }
    
    // 3. 执行合成
    audioData, err := client.Synthesize(request)
    if err != nil {
        return nil, err
    }
    
    return audioData, nil
}
```

**SSML标记语言**（高级用法）：

```xml
<speak>
    您好，欢迎参加面试。
    <break time="500ms"/>
    <prosody rate="+10%">请做一个自我介绍。</prosody>
</speak>
```

#### 3.2.3 发音评测原理

在项目中的实现位于 `english-learner/backend/internal/service/pronunciation.go`：

```go
// 发音评测服务
type PronunciationService struct {
    accessKey  string
    secretKey  string
    appKey    string
}

// 评测参数
type EvaluateParams struct {
    AudioData []byte  // 音频数据
    RefText   string  // 参考文本（STT结果）
    Language   string // 语言：en.zh
}

// 评测结果
type PronunciationResult struct {
    Overall     float64          // 总分
    Pron        float64          // 发音分
    Accuracy    float64          // 准确度
    Integrity   float64          // 完整度
    Fluency     float64          // 流畅度
    Words       []WordDetail     // 单词级详情
}

func (s *PronunciationService) Evaluate(ctx context.Context, params EvaluateParams) (*PronunciationResult, error) {
    // 1. 提交评测任务
    taskID, err := s.submitTask(params)
    if err != nil {
        return nil, err
    }
    
    // 2. 轮询获取结果
    result, err := s.getResult(taskID)
    if err != nil {
        return nil, err
    }
    
    return result, nil
}

func (s *PronunciationService) submitTask(params EvaluateParams) (string, error) {
    url := "https://nls-gateway-inner.aliyuncs.com/rest/v1/general/SpeechEvaluation"
    
    payload := map[string]interface{}{
        "app_key": s.appKey,
        "model_id": "en.pred.score",  // 英文评测模型
        "ref_text": params.RefText,
        "source_text": "cn",
        "评测方案": "EVALUATE_PLAN_ALL",
    }
    
    // multipart上传音频
    // POST请求获取taskID
    // ...
    
    return taskID, nil
}

func (s *PronunciationService) getResult(taskID string) (*PronunciationResult, error) {
    maxRetries := 20
    retryInterval := 500 * time.Millisecond
    
    for i := 0; i < maxRetries; i++ {
        result, err := s.queryTaskResult(taskID)
        if err == nil && result.Status == "completed" {
            return result, nil
        }
        
        time.Sleep(retryInterval)
    }
    
    return nil, fmt.Errorf("评测超时")
}
```

**评测维度说明**：

| 维度 | 说明 | 评分依据 |
|------|------|----------|
| Pron (发音) | 发音是否标准 | 音素级对比 |
| Accuracy (准确度) | 音准、语调 | 与标准音对比 |
| Integrity (完整度) | 是否有遗漏 | 文本覆盖比例 |
| Fluency (流畅度) | 停顿、重复 | 语速和连续性 |

#### 3.2.4 ffmpeg在音频处理中的角色

```bash
# 客户端录制的webm转服务端需要的pcm/wav
ffmpeg -i input.webm -ar 16000 -ac 1 -f wav output.wav

# 参数说明
-i input.webm     # 输入文件
-ar 16000         # 采样率 16kHz（阿里云要求）
-ac 1             # 单声道
-f wav            # 输出格式
```

在项目中的Go调用：

```go
func ConvertWebmToWav(input []byte) ([]byte, error) {
    // 创建临时文件
    tmpInput, _ := os.CreateTemp("", "input.webm")
    tmpOutput, _ := os.CreateTemp("", "output.wav")
    defer os.Remove(tmpInput.Name())
    defer os.Remove(tmpOutput.Name())
    
    tmpInput.Write(input)
    
    // 执行ffmpeg转换
    cmd := exec.Command("ffmpeg", "-y", "-i", tmpInput.Name(),
        "-ar", "16000", "-ac", "1", "-f", "wav", tmpOutput.Name())
    
    if err := cmd.Run(); err != nil {
        return nil, fmt.Errorf("ffmpeg error: %w", err)
    }
    
    return os.ReadFile(tmpOutput.Name())
}
```

#### 3.2.5 面试角度：实时语音系统的架构设计

**面试标准回答**：

> "InterviewPro的语音处理链路是：音频采集→格式转换→STT→AI评估→TTS合成。每个环节都有容错：
> 1. STT失败：返回文字输入提示
> 2. 评测超时：最多等10秒后跳过
> 3. TTS失败：返回纯文字回复
> 
> 后续规划用Eino框架将STT/TTS封装为Tool，让AI Agent自主决定何时调用语音能力。"

---

### 3.3 百炼STT

#### 3.3.1 与阿里云NLS STT的差异

| 维度 | 阿里云NLS | 百炼STT |
|------|-----------|---------|
| 协议 | 阿里云SDK | WebSocket直连 |
| 语言支持 | 中/英/日/韩 | 中/英/日/韩/德/法/俄 |
| 模型 | 通用模型 | 通义千问底座 |
| 实时性 | 毫秒级 | 毫秒级 |
| 费用 | 按量计费 | API套餐 |

#### 3.3.2 WebSocket实时流式识别实现

在项目中的实现位于 `english-learner/backend/internal/service/bailian_stt.go`：

```go
type BailianSTT struct {
    apiKey   string
    baseURL  string
    model    string  // "paraformer-realtime-v2"
}

func (b *BailianSTT) TranscribeStream(ctx context.Context, 
    audioReader io.Reader, 
    resultChan chan string) error {
    
    // 1. 建立WebSocket连接
    wsURL := fmt.Sprintf("%s?token=%s", b.baseURL, b.apiKey)
    conn, _, err := websocket.DefaultDialer.DialContext(ctx, wsURL, nil)
    if err != nil {
        return err
    }
    defer conn.Close()
    
    // 2. 接收消息协程
    go func() {
        for {
            _, message, err := conn.ReadMessage()
            if err != nil {
                close(resultChan)
                return
            }
            
            // 解析响应
            var resp BailianResponse
            json.Unmarshal(message, &resp)
            
            if resp.Text != "" {
                select {
                case resultChan <- resp.Text:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    
    // 3. 发送音频数据
    buf := make([]byte, 1024)
    for {
        n, err := audioReader.Read(buf)
        if n > 0 {
            // 发送音频帧
            err := conn.WriteMessage(websocket.BinaryMessage, buf[:n])
            if err != nil {
                return err
            }
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

---

## 4. 前端技术栈详解

### 4.1 React Native + Expo

#### 4.1.1 RN核心原理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    React Native 架构原理                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                          JavaScript 层                                       │
│                    ┌─────────────────────┐                                  │
│                    │     React 代码       │                                  │
│                    │  (Component/State)  │                                  │
│                    └──────────┬──────────┘                                  │
│                               │                                              │
│                    ┌──────────▼──────────┐                                  │
│                    │    React DOM 类似    │                                  │
│                    │    (React Native)    │                                  │
│                    └──────────┬──────────┘                                  │
│                               │                                              │
│    ┌──────────────────────────┼──────────────────────────┐                  │
│    │                          │                          │                  │
│    ▼                          ▼                          ▼                  │
│ ┌─────────┐            ┌─────────────┐            ┌──────────┐             │
│ │iOS Bridge│            │ Android Bridge│          │ JSI层    │             │
│ │ (ObjC)   │            │    (Java)    │           │ (新架构) │             │
│ └────┬─────┘            └──────┬───────┘            └────┬─────┘             │
│      │                          │                          │                  │
│      ▼                          ▼                          ▼                  │
│ ┌─────────┐            ┌─────────────┐            ┌──────────────────┐        │
│ │  Native  │            │    Native   │            │   C++ Host      │        │
│ │  Modules │            │   Modules   │            │   (v8/JSCore)   │        │
│ └─────────┘            └─────────────┘            └──────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**JS Bridge（传统架构）**：
- JS线程和Native线程通过JSON消息通信
- 异步、序列化开销
- React Native 0.65之前

**JSI（New Architecture）**：
- C++直接绑定，无序列化
- 同步调用
- 支持多线程
- React Native 0.68+

#### 4.1.2 Expo的受限与eject策略

```bash
# Expo开发模式
expo start
# 优点：无需Xcode/Android Studio即可开发
# 限制：无法使用未在Expo SDK中的Native模块

# Expo eject（脱离Expo）
expo eject
# 生成ios/和android/原生项目
# 可以添加任意Native模块
# 失去Expo OTA更新能力

# Expo prebuild（推荐）
expo prebuild
# 生成原生项目但不删除App.json
# 保持managed workflow的便利性
```

#### 4.1.3 InterviewPro的页面结构和导航

```
english-learner/frontend/
├── src/
│   ├── screens/
│   │   ├── HomeScreen.tsx        # 首页
│   │   ├── InterviewScreen.tsx   # 面试界面
│   │   ├── HistoryScreen.tsx     # 历史记录
│   │   └── ProfileScreen.tsx      # 个人中心
│   │
│   ├── components/
│   │   ├── AudioRecorder.tsx     # 录音组件
│   │   ├── AudioPlayer.tsx       # 播放组件
│   │   ├── ScoreDisplay.tsx      # 评分展示
│   │   └── WebSocketManager.tsx  # WS连接管理
│   │
│   ├── services/
│   │   ├── api.ts                # HTTP API
│   │   └── websocket.ts          # WebSocket服务
│   │
│   └── navigation/
│       └── AppNavigator.tsx      # 导航配置
```

### 4.2 WebSocket客户端

#### 4.2.1 前端WebSocket连接管理

```typescript
// services/websocket.ts
class WebSocketManager {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  
  connect(token: string): Promise<void> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(
        `wss://api.interviewpro.com/ws/interview?token=${token}`
      );
      
      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.reconnectAttempts = 0;
        resolve();
      };
      
      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        this.handleMessage(data);
      };
      
      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        reject(error);
      };
      
      this.ws.onclose = () => {
        this.attemptReconnect();
      };
    });
  }
  
  private attemptReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      setTimeout(() => {
        this.connect(this.token);
      }, 1000 * this.reconnectAttempts);
    }
  }
  
  sendAudioChunk(audioData: ArrayBuffer) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(audioData);
    }
  }
}
```

#### 4.2.2 音频录制和发送

```typescript
// components/AudioRecorder.tsx
import { Audio } from 'expo-av';

class AudioRecorder {
  private recording: Audio.Recording | null = null;
  
  async startRecording() {
    // 请求权限
    const permission = await Audio.requestPermissionsAsync();
    if (!permission.granted) {
      throw new Error('Microphone permission denied');
    }
    
    // 配置录音
    await Audio.setAudioModeAsync({
      allowsRecordingIOS: true,
      playsInSilentModeIOS: true,
    });
    
    // 开始录音
    const { recording } = await Audio.Recording.createAsync(
      Audio.RecordingOptionsPresets.HIGH_QUALITY
    );
    
    this.recording = recording;
  }
  
  async stopRecording(): Promise<ArrayBuffer> {
    if (!this.recording) {
      throw new Error('No active recording');
    }
    
    await this.recording.stopAndUnloadAsync();
    const uri = this.recording.getURI();
    this.recording = null;
    
    // 读取为ArrayBuffer
    const response = await fetch(uri);
    return await response.arrayBuffer();
  }
  
  async getStatus(): Promise<RecordingStatus> {
    if (!this.recording) {
      throw new Error('No active recording');
    }
    return await this.recording.getStatusAsync();
  }
}
```

---

## 5. 数据库设计

### 5.1 MySQL表结构

```sql
-- 用户表
CREATE TABLE users (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 面试会话表
CREATE TABLE interviews (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    topic VARCHAR(100) DEFAULT 'general',
    status ENUM('active', 'paused', 'completed') DEFAULT 'active',
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP NULL,
    total_score DECIMAL(5,2) DEFAULT NULL,
    duration INT DEFAULT 0 COMMENT '面试时长(秒)',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_started_at (started_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 消息记录表
CREATE TABLE messages (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    interview_id BIGINT UNSIGNED NOT NULL,
    role ENUM('user', 'assistant') NOT NULL,
    content TEXT,
    audio_url VARCHAR(255) DEFAULT NULL,
    scores JSON DEFAULT NULL COMMENT '5维评分JSON',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (interview_id) REFERENCES interviews(id) ON DELETE CASCADE,
    INDEX idx_interview_id (interview_id),
    INDEX idx_role (role)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 评分详情表（可选，用于更细粒度存储）
CREATE TABLE score_records (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    message_id BIGINT UNSIGNED NOT NULL,
    dimension VARCHAR(20) NOT NULL COMMENT 'fluency/grammar/vocabulary/content/pronunciation',
    score DECIMAL(5,2) NOT NULL,
    feedback TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (message_id) REFERENCES messages(id) ON DELETE CASCADE,
    UNIQUE KEY uk_message_dimension (message_id, dimension)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 5.2 GORM模型定义

```go
// model.go - 完整模型定义
type User struct {
    ID           uint           `gorm:"primaryKey"`
    Username     string         `gorm:"size:50;not null;uniqueIndex"`
    Email        string         `gorm:"size:100;not null;uniqueIndex"`
    PasswordHash string         `gorm:"size:255;not null"`
    Interviews   []Interview    `gorm:"foreignKey:UserID"`
    CreatedAt    time.Time
    UpdatedAt    time.Time
}

type Interview struct {
    ID         uint       `gorm:"primaryKey"`
    UserID     uint       `gorm:"index;not null"`
    Topic      string     `gorm:"size:100;default:'general'"`
    Status     string     `gorm:"size:20;default:'active'"`
    StartedAt  time.Time  `gorm:"autoCreateTime"`
    EndedAt    *time.Time `gorm:"default:null"`
    TotalScore *float64   `gorm:"type:decimal(5,2)"`
    Duration   int        `gorm:"default:0"`
    Messages   []Message  `gorm:"foreignKey:InterviewID"`
    User       User       `gorm:"foreignKey:UserID"`
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

type Message struct {
    ID          uint       `gorm:"primaryKey"`
    InterviewID uint       `gorm:"index;not null"`
    Role        string     `gorm:"size:10;not null"`
    Content     string     `gorm:"type:text"`
    AudioURL    string     `gorm:"size:255"`
    Scores      string     `gorm:"type:json"`  // 存储5维评分JSON
    Interview   Interview  `gorm:"foreignKey:InterviewID"`
    CreatedAt   time.Time
}

// 评分结果（JSON序列化）
type Scores struct {
    Fluency       float64 `json:"fluency"`
    Grammar       float64 `json:"grammar"`
    Vocabulary    float64 `json:"vocabulary"`
    Content       float64 `json:"content"`
    Pronunciation float64 `json:"pronunciation"`
    Overall       float64 `json:"overall"`
    Feedback      string  `json:"feedback"`
}
```

### 5.3 索引设计

```sql
-- 覆盖索引优化常见查询

-- 查询用户最近的面试记录
CREATE INDEX idx_user_started ON interviews(user_id, started_at DESC);

-- 查询活跃面试
CREATE INDEX idx_status_started ON interviews(status, started_at);

-- 统计每用户面试次数
CREATE INDEX idx_user_status ON interviews(user_id, status);

-- 分页查询（延迟关联）
SELECT i.*, m.content 
FROM interviews i 
LEFT JOIN messages m ON i.id = m.interview_id AND m.role = 'user'
WHERE i.user_id = 123
ORDER BY i.started_at DESC 
LIMIT 20 OFFSET 0;
```

---

## 6. 部署架构

### 6.1 当前部署方式

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        InterviewPro 部署架构                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                            用户请求                                          │
│                               │                                              │
│                               ▼                                              │
│                    ┌──────────────────┐                                    │
│                    │   Nginx          │                                    │
│                    │  负载均衡/SSL     │                                    │
│                    └────────┬─────────┘                                    │
│                             │                                              │
│           ┌─────────────────┼─────────────────┐                            │
│           │                 │                 │                            │
│           ▼                 ▼                 ▼                            │
│    ┌────────────┐   ┌────────────┐   ┌────────────┐                       │
│    │ Go Backend │   │ Go Backend │   │ Go Backend │                       │
│    │   :8080    │   │   :8080    │   │   :8080    │                       │
│    └─────┬──────┘   └─────┬──────┘   └─────┬──────┘                       │
│          │                 │                 │                            │
│          └─────────────────┼─────────────────┘                            │
│                            │                                              │
│           ┌────────────────┼────────────────┐                            │
│           │                │                │                            │
│           ▼                ▼                ▼                            │
│    ┌────────────┐   ┌────────────┐   ┌────────────┐                       │
│    │   MySQL    │   │   Redis    │   │  外部API   │                       │
│    │  (主库)     │   │  (集群)     │   │DeepSeek等  │                       │
│    └────────────┘   └────────────┘   └────────────┘                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 K3s部署规划

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: interview-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: interview-backend
  template:
    metadata:
      labels:
        app: interview-backend
    spec:
      containers:
      - name: backend
        image: interview-pro/backend:latest
        ports:
        - containerPort: 8080
        env:
        - name: AI_PROVIDER
          valueFrom:
            secretKeyRef:
              name: interview-secrets
              key: ai-provider
        - name: DEEPSEEK_API_KEY
          valueFrom:
            secretKeyRef:
              name: interview-secrets
              key: deepseek-api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: interview-backend-svc
spec:
  selector:
    app: interview-backend
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: interview-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  rules:
  - host: api.interviewpro.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: interview-backend-svc
            port:
              number: 80
  tls:
  - hosts:
    - api.interviewpro.com
    secretName: interview-tls
```

### 6.3 Docker多阶段构建

```dockerfile
# backend/Dockerfile
# ============ 构建阶段 ============
FROM golang:1.22-alpine AS builder

WORKDIR /app

# 缓存依赖
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# CGO_ENABLED=0 静态编译，无依赖
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -o backend \
    ./cmd/server

# ============ 运行阶段 ============
FROM alpine:3.19

WORKDIR /app

# 安装CA证书（HTTPS调用）
RUN apk --no-cache add ca-certificates tzdata

# 复制二进制
COPY --from=builder /app/backend .
COPY --from=builder /app/config.yaml .

# 创建非root用户
RUN adduser -D -g '' appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["./backend"]
```

### 6.4 CI/CD流程

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
    paths: ['backend/**']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: registry.interviewpro.com
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_TOKEN }}
          
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: |
            registry.interviewpro.com/interview-backend:${{ github.sha }}
            registry.interviewpro.com/interview-backend:latest
          cache-from: type=registry,ref=registry.interviewpro.com/interview-backend:latest
          cache-to: type=inline
          
      - name: Deploy to K3s
        uses: k8s-deploy@v1
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG }}
          namespace: production
          manifests: k8s/
          images: |
            registry.interviewpro.com/interview-backend:${{ github.sha }}
```

---

## 7. 技术栈面试映射表

### 7.1 每个技术对应面试问题

| 技术 | 基础问题 | 进阶问题 | 深度问题 |
|------|----------|----------|----------|
| **Gin** | Gin中间件原理？ | Gin vs Echo区别？ | Radix Tree如何实现？ |
| **GORM** | GORM如何防SQL注入？ | N+1问题及解决？ | GORM Callback机制？ |
| **WebSocket** | 握手过程？ | 重连机制？ | 并发写保护？ |
| **Redis** | Redis数据类型？ | 缓存穿透/击穿/雪崩？ | 主从/集群原理？ |
| **Viper** | 配置优先级？ | 热更新原理？ | 多环境配置？ |
| **Zap** | 为什么快？ | 结构化日志优势？ | 采样策略？ |
| **模型工厂** | 策略模式应用？ | 如何扩展新模型？ | Eino改造方案？ |
| **语音服务** | STT/TTS原理？ | 实时性如何保证？ | 多服务商切换？ |

### 7.2 如何用InterviewPro佐证技术能力

#### 自我介绍模板（1分钟）

> "我叫XXX，有X年Go开发经验。最近在做一个AI面试练习项目 **InterviewPro**，核心技术栈是 Go + Gin + WebSocket + React Native。
> 
> 项目实现了完整的语音对话流程：用户录音→STT转文字→AI评估→TTS回复，支持DeepSeek API和本地Qwen模型两种模式。项目地址是 https://gitee.com/chenjiayi/interview-quicker，欢迎交流。"

#### 技术深度追问应对

**Q: WebSocket如何保证可靠性？**

> "InterviewPro中我实现了三层保障：
> 1. **心跳机制**：每30秒发送Ping，对方需在60秒内响应Pong，超时判定为断连
> 2. **消息队列**：发送使用带缓冲的Channel（256容量），队列满时主动关闭连接
> 3. **自动重连**：前端WebSocket管理器检测到断连后，指数退避重连，最多重试5次
> 
> ```go
> // 心跳实现
> go func() {
>     ticker := time.NewTicker(30 * time.Second)
>     for { select {
>     case <-ticker.C:
>         conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
>         if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
>             return
>         }
>     }}
> }()
> ```"

**Q: 如何设计高并发的语音处理系统？**

> "InterviewPro的语音处理做了这些优化：
> 1. **STT和评测并行**：收到音频后，STT转文字和发音评测并发执行，减少总延迟
> 2. **异步处理**：WebSocket发送和AI评估通过goroutine并发，不阻塞主流程
> 3. **音频缓冲**：客户端分片发送，服务端组装后再处理
> 4. **限流保护**：Redis计数器实现接口限流，防止突发流量冲击后端"

**Q: 模型工厂的扩展性如何保证？**

> "模型工厂采用策略模式，定义ModelProvider接口：
> 
> ```go
> type ModelProvider interface {
>     Generate(systemPrompt, userPrompt string) (string, error)
>     GetName() string
> }
> ```
> 
> 新增模型只需实现接口，在GetModelProvider()工厂函数中添加case即可。后续规划用Eino框架统一到ChatModel接口，同时获得流式响应和Agent编排能力。"

### 7.3 技术亮点总结

| 亮点 | 描述 | 佐证能力 |
|------|------|----------|
| **WebSocket实时通信** | 完整的双向实时语音对话 | 实时系统、高并发 |
| **AI模型工厂** | 支持DeepSeek/Qwen双模式 | 设计模式、扩展性 |
| **多语音服务集成** | 阿里云NLS + 百炼STT | 外部服务集成 |
| **5维评分体系** | Fluency/Grammar/Vocabulary/Content/Pronunciation | AI应用设计 |
| **结构化日志** | Zap JSON日志 + 请求追踪 | 可观测性 |
| **容器化部署** | Docker多阶段构建 + K3s部署 | DevOps |

---

## 8. 学习路径建议

### 8.1 按优先级排列的学习路线

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        InterviewPro 技术学习路径                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  第一阶段：Go核心（1-2周）                                                   │
│  ═══════════════════════                                                     │
│  ├─ Go基础语法和并发模型                                                      │
│  ├─ Context并发控制                                                          │
│  ├─ sync包（WaitGroup/Mutex/Channel）                                        │
│  └─ 练习：实现一个并发任务调度器                                              │
│                                                                             │
│  第二阶段：Web开发（2-3周）                                                  │
│  ═══════════════════════                                                     │
│  ├─ Gin框架（路由/中间件/Context）                                            │
│  ├─ GORM（CRUD/预加载/事务）                                                 │
│  ├─ MySQL（索引/优化）                                                       │
│  └─ 练习：重构InterviewPro的HTTP层                                            │
│                                                                             │
│  第三阶段：实时通信（1-2周）                                                  │
│  ═══════════════════════                                                     │
│  ├─ WebSocket协议原理                                                        │
│  ├─ gorilla/websocket用法                                                   │
│  ├─ 消息队列和Channel                                                         │
│  └─ 练习：实现聊天室功能                                                      │
│                                                                             │
│  第四阶段：缓存和配置（1周）                                                  │
│  ═══════════════════════                                                     │
│  ├─ Redis数据类型和应用场景                                                   │
│  ├─ go-redis用法                                                             │
│  ├─ Viper配置管理                                                            │
│  └─ 练习：实现接口限流                                                        │
│                                                                             │
│  第五阶段：AI集成（2-3周）                                                    │
│  ═══════════════════════                                                     │
│  ├─ OpenAI/DeepSeek API调用                                                  │
│  ├─ llama.cpp本地部署                                                        │
│  ├─ 阿里云语音服务集成                                                        │
│  └─ 练习：实现一个简单的AI对话机器人                                          │
│                                                                             │
│  第六阶段：部署运维（1周）                                                    │
│  ═══════════════════════                                                     │
│  ├─ Docker容器化                                                              │
│  ├─ K8s/K3s基础                                                              │
│  ├─ CI/CD流程                                                                │
│  └─ 日志和监控                                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 推荐学习资源

**Go语言**

| 资源 | 类型 | 链接 | 说明 |
|------|------|------|------|
| Go语言圣经 | 书籍 | golang.org | 官方入门书 |
| Go进阶之路 | 视频 | B站 | 视频教程 |
| Go语言设计与实现 | 书籍 | 异步社区 | 源码分析 |
| Go Quiz | 网站 | goquiz.github.io | 面试题库 |

**Web开发**

| 资源 | 类型 | 链接 | 说明 |
|------|------|------|------|
| Gin文档 | 官方 | gin-gonic.github.io | 框架文档 |
| GORM文档 | 官方 | gorm.io | ORM文档 |
| MySQL实战45讲 | 专栏 | 极客时间 | 进阶教程 |

**实时通信**

| 资源 | 类型 | 链接 | 说明 |
|------|------|------|------|
| WebSocket RFC | 规范 | rfc6455 | 协议规范 |
| gorilla/websocket | GitHub | github.com/gorilla/websocket | 源码 |
| 自己实现WebSocket | 博客 | 源码实现 | 学习原理 |

**AI应用**

| 资源 | 类型 | 链接 | 说明 |
|------|------|------|------|
| Eino框架 | GitHub | github.com/cloudwego/eino | AI编排 |
| 阿里云NLS | 官方 | 阿里云 | 语音服务 |
| LangChain教程 | 视频 | YouTube | AI应用 |

### 8.3 动手实践建议

**阶段一：本地运行项目**

```bash
# 1. 克隆代码
git clone https://gitee.com/chenjiayi/interview-quicker.git
cd interview-quicker/english-learner/backend

# 2. 配置环境变量
cp .env.example .env
# 编辑.env填入API Key

# 3. 启动服务
go run cmd/server/main.go

# 4. 测试API
curl http://localhost:8080/api/ping
```

**阶段二：添加新功能**

1. 在`model_factory.go`添加Claude模型支持
2. 在`bailian_stt.go`添加实时字幕功能
3. 实现面试题库随机抽取

**阶段三：优化性能**

1. 添加Redis缓存热门问题
2. 实现AI响应结果缓存
3. 添加Prometheus metrics

**阶段四：Eino改造**

参考 `../学习资料/InterviewPro代码分析.md` 的 Phase 1 方案：
1. 替换model_factory为Eino ChatModel
2. 验证流式输出
3. 重构评估流程为Graph编排

---

## 附录：代码索引

| 文件 | 功能 | 核心内容 |
|------|------|----------|
| `backend/cmd/server/main.go` | 入口 | Gin启动、中间件配置 |
| `backend/internal/config/config.go` | 配置 | Viper加载、环境变量 |
| `backend/internal/handler/interview.go` | HTTP处理 | REST API实现 |
| `backend/internal/model/model.go` | 数据模型 | GORM模型定义 |
| `backend/internal/service/ai.go` | AI服务 | 评估逻辑 |
| `backend/internal/service/model_factory.go` | 模型工厂 | 策略模式 |
| `backend/internal/service/aliyun_speech.go` | 阿里云语音 | STT/TTS |
| `backend/internal/service/pronunciation.go` | 发音评测 | 评测API |
| `backend/internal/service/bailian_stt.go` | 百炼STT | WebSocket识别 |
| `backend/internal/ws/hub.go` | WebSocket Hub | 连接管理 |
| `backend/internal/ws/client.go` | WebSocket Client | 消息处理 |
| `backend/internal/middleware/auth.go` | JWT认证 | Token验证 |

---

*文档版本：v1.0*  
*创建日期：2026年*  
*项目地址：https://gitee.com/chenjiayi/interview-quicker*
