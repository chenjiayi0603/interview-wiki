# Go 常用开源库手册

> 以下库在业界使用非常广泛，涵盖 **Web、并发、数据库、配置、日志、测试、工具** 等方向。

See also: [[Go框架与工具]], [[Go网络编程]], [[Go并发安全]], [[Go语言基础]]

---

## 🌐 Web 框架

| 库                                         | 描述                      |
| :---------------------------------------- | :---------------------- |
| [Gin](https://github.com/gin-gonic/gin)   | 高性能 HTTP Web 框架         |
| [Fiber](https://github.com/gofiber/fiber) | 受 Express.js 启发的 Web 框架 | 快速、轻量 |
| [Echo](https://github.com/labstack/echo)  | 极简高性能 Web 框架            | REST API 友好 |

### Gin

Gin 是 Go 语言中最流行的 Web 框架之一，基于 Go 标准库 `net/http`，易于集成各种中间件和工具，兼容性佳。生态成熟，社区庞大，文档丰富，性能优异，适合主流 Go 项目。

**Gin 小例子：**

```go
import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{"msg": "pong"})
    })
    r.Run(":8080")
}
```

### Fiber

Fiber 是受 Express.js 启发的 Go Web 框架，拥有限量级、高性能、易用等特点，适合构建 REST API、Web 服务等。其语法风格与 Node.js 的 Express 类似，学习曲线平滑。

Fiber 基于 [fasthttp](https://github.com/valyala/fasthttp) 实现，无锁设计，追求极致性能，但与标准库 `net/http` 不兼容（第三方中间件支持相对较少）。

**Fiber 和 Gin 的区别：**
- Fiber 受 Node.js 的 Express 启发，API 风格与 Express 非常相似，上手快，语法简洁，适合有 Express 背景的开发者。
- Gin 更贴近 Go 原生语法，生态成熟，社区庞大，文档丰富，性能优异，适合主流 Go 项目。
- Fiber 基于 fasthttp 实现，无锁设计，追求极致性能，但与标准库 `net/http` 不兼容（第三方中间件支持相对较少）。
- Gin 基于 Go 标准库 `net/http`，易于集成各种中间件和工具，兼容性更佳。
- Fiber 要用 fiber 专属 Context，Gin 用自己的 gin.Context，但能方便获得底层 http 请求对象。
- 选择建议：极致性能/FastAPI 风格/Express 习惯选 Fiber，稳定/生态/兼容性优先选 Gin。

**Fiber 小例子：**

```go
import "github.com/gofiber/fiber/v2"

func main() {
    app := fiber.New()
    app.Get("/hello", func(c *fiber.Ctx) error {
        return c.SendString("Hello, Fiber!")
    })
    app.Listen(":3000")
}
```

### Echo

Echo 是极简高性能 Web 框架，REST API 友好。

**Echo 小例子：**

```go
import "github.com/labstack/echo/v4"

func main() {
    e := echo.New()
    e.GET("/", func(c echo.Context) error {
        return c.String(200, "Hello Echo")
    })
    e.Start(":1323")
}
```

---

## 💾 数据库 ORM

| 库                                       | 作用              | 特点              |
| :-------------------------------------- | :-------------- | :-------------- |
| [GORM](https://gorm.io)                 | ORM 框架          | 类似 ActiveRecord |
| [SQLX](https://github.com/jmoiron/sqlx) | 扩展 database/sql | 支持 Struct 映射    |
| [Ent](https://entgo.io)                 | Schema 驱动 ORM   | 代码生成型 ORM       |

### GORM

GORM 是 Go 语言中最流行的 ORM（对象关系映射）框架，支持主流数据库（MySQL、PostgreSQL、SQLite、SQL Server 等），API 类似 ActiveRecord，提供自动迁移、CRUD、关联、预加载等功能，语法简洁且高度灵活，社区和文档都非常完善。常用于 Go 项目的数据库操作。

**GORM 小例子：**

```go
import (
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    ID   uint
    Name string
}

// 打开数据库（sqlite），文件名 test.db，返回 *gorm.DB 和错误（这里只简单忽略错误处理）
db, _ := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
// 自动迁移，根据 User 结构体自动建表/更新表结构
db.AutoMigrate(&User{})
// 新增一条记录，Name 字段为 "Alice"
db.Create(&User{Name: "Alice"})
// 定义接收查询结果的变量 u
var u User
// 查询主键 id=1 的用户，结果赋值给 u
db.First(&u, 1)
```

### SQLX

SQLX 是 Go 语言对标准库 `database/sql` 的增强库，扩展了更强大的 SQL 查询能力和结构体映射功能。相比原生的 `database/sql`，sqlx 通过标签简化了查询结果和结构体的自动绑定，支持命名参数、Struct/Map 批量操作、事务管理等，兼容所有支持 database/sql 的驱动。常用于需要复杂 SQL 和高效表数据映射的场景。

**SQLX 小例子：**

```go
import "github.com/jmoiron/sqlx"

db, _ := sqlx.Connect("sqlite3", "test.db")
var users []struct {
    ID   int    `db:"id"`
    Name string `db:"name"`
}
db.Select(&users, "SELECT id, name FROM users")
```

### Ent

Ent 是 Schema 驱动的代码生成型 ORM。

**Ent 小例子：**（需先 `go run entgo.io/ent/cmd/ent new User` 生成 schema 与代码）

```go
client, _ := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
ctx := context.Background()
u, _ := client.User.Create().SetName("Bob").Save(ctx)
```

---

## 📦 配置管理

| 库                                                         | 描述                  |
| :-------------------------------------------------------- | :------------------ |
| [Viper](https://github.com/spf13/viper)                   | 配置加载（JSON/YAML/env），支持热加载 |
| [Cobra](https://github.com/spf13/cobra)                   | 命令行解析框架，Go CLI 工具首选 |
| [envconfig](https://github.com/kelseyhightower/envconfig) | 环境变量绑定，云部署常用       |

### Viper

Viper 是 Go 语言中最流行的配置管理库之一。它支持读取多种格式的配置文件（如 JSON、YAML、TOML）、环境变量、命令行参数，支持热加载和远程配置，非常适合现代应用的多环境配置需求。Viper 常与 Cobra 命令行库配合使用，是 Go 项目配置首选。

```go
import "github.com/spf13/viper"

viper.SetConfigFile("config.yaml")
viper.ReadInConfig()
port := viper.GetInt("server.port")        // 读嵌套键
name := viper.GetString("app.name")
viper.Set("key", "value")                  // 也可程序内设置
```

### Cobra

Cobra 是 Go 语言流行的命令行应用（CLI）库，由 spf13 社区维护。它支持命令分组、命令嵌套、参数与 Flag 管理、自动生成命令帮助文档等，广泛应用于 Go 各类 CLI、工具工程，常与 Viper 配合进行配置管理。

**Cobra 小例子：**

```go
import "github.com/spf13/cobra"

// 定义根命令
var rootCmd = &cobra.Command{
    Use:   "myapp",              // 命令名称
    Short: "我的 CLI 工具",        // 简要描述
}

// 定义 serve 子命令
var serveCmd = &cobra.Command{
    Use:   "serve",              // 子命令名称
    Short: "启动服务并打印参数",    // 子命令描述
    Run: func(cmd *cobra.Command, args []string) {
        println("做啥")          // 执行子命令时打印
        if len(args) > 0 {
            println("参数:", args) // 如果有参数则打印
        }
    },
}

// 添加 serve 子命令到根命令
rootCmd.AddCommand(serveCmd)

// 执行根命令（解析命令行）
rootCmd.Execute()
```

### envconfig

envconfig 用于从环境变量填充结构体，云部署常用。

**envconfig 小例子：**

```go
import "github.com/kelseyhightower/envconfig"

type Config struct {
    Port     int    `envconfig:"PORT" default:"8080"`
    Database string `envconfig:"DATABASE_URL" required:"true"`
}
var cfg Config
envconfig.Process("", &cfg)  // 从环境变量填充
```

---

## 🧰 工具与辅助

| 库                                                                     | 作用           |
| :-------------------------------------------------------------------- | :----------- |
| [GoDotEnv](https://github.com/joho/godotenv)                          | 加载 `.env` 文件 |
| [uuid](https://github.com/google/uuid)                                | UUID 生成      |
| [go-playground/validator](https://github.com/go-playground/validator) | 表单/结构体验证     |
| [spf13/cast](https://github.com/spf13/cast)                           | 类型安全转换       |

**GoDotEnv 小例子：**

```go
import "github.com/joho/godotenv"

godotenv.Load()                    // 加载 .env 到环境变量
// 之后用 os.Getenv("KEY") 读取
```

**uuid 小例子：**

```go
import "github.com/google/uuid"

id := uuid.New().String()          // "550e8400-e29b-41d4-a716-446655440000"
id2 := uuid.NewSHA1(uuid.NameSpaceDNS, []byte("example.com"))
```

**validator 小例子：**

```go
import "github.com/go-playground/validator/v10"

type User struct {
    Email string `validate:"required,email"`
    Age   int    `validate:"gte=0,lte=120"`
}
validate := validator.New()
u := User{Email: "bad", Age: 150}
err := validate.Struct(u)          // 校验失败
```

**cast 小例子：**

```go
import "github.com/spf13/cast"

s := cast.ToString(42)             // "42"
i := cast.ToInt("100")             // 100
b := cast.ToBool("true")           // true
```

---

## 🧵 并发与任务

| 库                                                   | 作用               |
| :-------------------------------------------------- | :--------------- |
| [ants](https://github.com/panjf2000/ants)           | 高性能 goroutine 池  |
| [go-redis](https://github.com/redis/go-redis)       | Redis 客户端        |
| [asynq](https://github.com/hibiken/asynq)           | 基于 Redis 的异步任务队列 |
| [workpool](https://github.com/gammazero/workerpool) | 并发任务控制           |

### ants

ants 是高性能 goroutine 池。

**ants 小例子：**

```go
import "github.com/panjf2000/ants/v2"

pool, _ := ants.NewPool(10)
defer pool.Release()
pool.Submit(func() {
    println("task in pool")
})
```

### go-redis

go-redis 是 Go 语言 Redis 客户端。

**go-redis 小例子：**

```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
rdb.Set(ctx, "key", "value", 0)
val, _ := rdb.Get(ctx, "key").Result()
```

### asynq

asynq 是基于 Redis 的异步任务队列。

**asynq 小例子：**

```go
import "github.com/hibiken/asynq"

client := asynq.NewClient(asynq.RedisClientOpt{Addr: "localhost:6379"})
task := asynq.NewTask("email:send", []byte(`{"to":"a@b.com"}`))
client.Enqueue(task)
// 需单独起 Worker 消费队列
```

### workpool

workpool（github.com/gammazero/workerpool）是 Go 的一个轻量级高性能并发任务池库，用于限制并发 goroutine 数量（实现任务限流），方便批量任务的高效调度和收集结果。

**workpool 小例子：**

```go
import "github.com/gammazero/workerpool"

// 创建一个"最多同时3个任务"的 worker pool，批量提交10个任务。
wp := workerpool.New(3)
for i := 0; i < 10; i++ {
    i := i // 防止闭包变量引用问题，捕获当前 i
    wp.Submit(func() { println("job", i) })
}
wp.StopWait() // 等待所有任务执行完成
```

---

## 🪵 日志系统

| 库                                            | 特点               |
| :------------------------------------------- | :--------------- |
| [zap](https://github.com/uber-go/zap)        | Uber 开发，高性能结构化日志 |
| [logrus](https://github.com/sirupsen/logrus) | 经典日志框架           |
| [zerolog](https://github.com/rs/zerolog)     | 超低开销 JSON 日志     |

**zap 小例子：**

```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
logger.Info("hello", zap.String("user", "alice"), zap.Int("count", 1))
logger.Sync()
```

**logrus 小例子：**

```go
import "github.com/sirupsen/logrus"

logrus.SetFormatter(&logrus.JSONFormatter{})
logrus.WithFields(logrus.Fields{"user": "alice"}).Info("hello")
```

**zerolog 小例子：**

```go
import "github.com/rs/zerolog/log"

log.Info().Str("user", "alice").Int("count", 1).Msg("hello")
```

---

## 🧩 序列化 / 通信

| 库                                                       | 用途         |
| :------------------------------------------------------ | :--------- |
| [encoding/json](https://pkg.go.dev/encoding/json)       | 标准库 JSON   |
| [gopkg.in/yaml.v3](https://pkg.go.dev/gopkg.in/yaml.v3) | YAML 支持    |
| [protobuf / gRPC](https://grpc.io)                      | 高性能 RPC 通信 |
| [msgpack](https://github.com/vmihailenco/msgpack)       | 二进制序列化     |

**encoding/json 小例子：**

```go
import "encoding/json"

type T struct{ A int `json:"a"` }
b, _ := json.Marshal(T{A: 1})    // []byte(`{"a":1}`)
var t T
json.Unmarshal(b, &t)
```

**yaml.v3 小例子：**

```go
import "gopkg.in/yaml.v3"

var cfg map[string]interface{}
yaml.Unmarshal([]byte("port: 8080\nname: app"), &cfg)
out, _ := yaml.Marshal(cfg)
```

**msgpack 小例子：**

```go
import "github.com/vmihailenco/msgpack/v5"

b, _ := msgpack.Marshal(map[string]int{"a": 1, "b": 2})
var m map[string]int
msgpack.Unmarshal(b, &m)
```

---

## 🧪 测试框架

| 库                                              | 特点           |
| :--------------------------------------------- | :----------- |
| [testify](https://github.com/stretchr/testify) | 断言 + mock 工具 |
| [ginkgo](https://github.com/onsi/ginkgo)       | BDD 风格测试框架   |
| [gomock](https://github.com/golang/mock)       | 接口 Mock 工具   |

**testify 小例子：**

```go
import "github.com/stretchr/testify/assert"

func TestAdd(t *testing.T) {
    assert.Equal(t, 3, 1+2)
    assert.True(t, 1 < 2)
    assert.NoError(t, nil)
}
```

**ginkgo 小例子：**（BDD 风格，需 `ginkgo bootstrap` 生成套件）

```go
var _ = Describe("Calculator", func() {
    It("adds two numbers", func() {
        Expect(1 + 2).To(Equal(3))
    })
})
```

**gomock 小例子：**（需用 mockgen 生成 mock 代码）

```go
// mockgen -source=repo.go -destination=repo_mock.go
ctrl := gomock.NewController(t)
defer ctrl.Finish()
m := NewMockRepo(ctrl)
m.EXPECT().Get(1).Return("alice", nil)
```

---

## 🧠 其他常用库

| 领域   | 库                                                                                | 说明            |
| :--- | :------------------------------------------------------------------------------- | :------------ |
| 网络   | [fasthttp](https://github.com/valyala/fasthttp)                                  | 高性能 HTTP      |
| 缓存   | [bigcache](https://github.com/allegro/bigcache)                                  | 高并发缓存         |
| 消息队列 | [nsq](https://github.com/nsqio/nsq), [sarama](https://github.com/Shopify/sarama) | Kafka/NSQ 客户端 |
| 文件上传 | [minio-go](https://github.com/minio/minio-go)                                    | S3 兼容存储 SDK   |
| 安全加密 | [bcrypt](https://pkg.go.dev/golang.org/x/crypto/bcrypt)                          | 密码哈希          |

### fasthttp

fasthttp 是 Go 语言的一个高性能 HTTP 服务库，专为极致性能优化，适合高并发场景，被广泛用于性能敏感的 Web 服务。它比标准库 net/http 快很多，但 API 略有不同，也不完全兼容 net/http。

**fasthttp 小例子：**

```go
import "github.com/valyala/fasthttp"

// 启动 HTTP 服务，处理每个请求
fasthttp.ListenAndServe(":8080", func(ctx *fasthttp.RequestCtx) {
    ctx.WriteString("Hello fasthttp")
})
```

### bigcache

bigcache 是一个在 Go 语言中的高并发、大容量内存缓存库。它用于在多核环境下避免全局锁，提供极高的并发读写能力，非常适合做应用级热点数据的本地缓存，秒杀等对性能要求极高场景尤其适用。

bigcache 用的不是 LRU 算法，而是基于分段和循环数组实现，无全局锁。若想用 LRU，可以用 groupcache/lru、hashicorp/golang-lru 或 go-cache。

**bigcache 小例子：**

```go
import "github.com/allegro/bigcache/v3"

// 创建缓存（10分钟过期）
cache, _ := bigcache.New(context.Background(), bigcache.DefaultConfig(10*time.Minute))
// 设置键值
cache.Set("key", []byte("value"))
// 获取值
val, _ := cache.Get("key")
```

**golang-lru 示例：**

```go
import lru "github.com/hashicorp/golang-lru/v2/simplelru"

lruCache, _ := lru.NewLRU[string, []byte](128) // 128为容量
lruCache.Add("key", []byte("value"))
val, ok := lruCache.Get("key")
if ok {
    // 使用 val
}
```

### sarama (Kafka)

**sarama 小例子：**

```go
import "github.com/Shopify/sarama"

config := sarama.NewConfig()
config.Producer.Return.Successes = true
producer, _ := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
producer.SendMessage(&sarama.ProducerMessage{Topic: "test", Value: sarama.StringEncoder("hello")})
```

### minio-go

**minio-go 小例子：**

```go
import (
    "context"
    "strings"
    "github.com/minio/minio-go/v7"
    "github.com/minio/minio-go/v7/pkg/credentials"
)

ctx := context.Background()
client, _ := minio.New("localhost:9000", &minio.Options{Creds: credentials.NewStaticV4("key", "secret", "")})
client.PutObject(ctx, "mybucket", "object.txt", strings.NewReader("content"), -1, minio.PutObjectOptions{})
```

### bcrypt

**bcrypt 小例子：**

```go
import "golang.org/x/crypto/bcrypt"

hash, _ := bcrypt.GenerateFromPassword([]byte("mypassword"), bcrypt.DefaultCost)
err := bcrypt.CompareHashAndPassword(hash, []byte("mypassword"))  // nil 表示匹配
```

---

## ✅ 总结建议

| 场景     | 推荐组合                     |
| :----- | :----------------------- |
| Web 后端 | Gin + GORM + Zap + Viper |
| CLI 工具 | Cobra + Viper            |
| 分布式系统  | gRPC + etcd + ants       |
| 微服务通信  | protobuf + asynq + Redis |
| 数据分析   | go-csv + go-chart        |

[src: raw/ingested/2技术/go/第三方库-Go 常用开源库.md]

## Related Pages
- [[Go框架与工具]]
- [[Go网络编程]]
- [[Go并发安全]]
- [[Go语言基础]]
- [[C++第三方库手册]]
