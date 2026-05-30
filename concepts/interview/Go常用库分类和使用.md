# Go 常用库分类和使用

> 本页整理 Go 语言常用第三方库的分类、安装与代码示例，涵盖 Web 框架、数据库 ORM、Redis 客户端、配置管理、日志库、测试框架和工具库。

See also: [[Go第三方库手册]], [[Go框架与工具]], [[Go网络编程]]

---

## 4.1 Web 框架

### Gin（高性能 HTTP Web 框架）

```go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

func main() {
    r := gin.Default()
    
    // 路由
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "pong"})
    })
    
    // 路由组
    v1 := r.Group("/v1")
    {
        v1.GET("/users", getUsers)
        v1.POST("/users", createUser)
    }
    
    r.Run(":8080")
}

func getUsers(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"users": []string{"Alice", "Bob"}})
}

func createUser(c *gin.Context) {
    var user struct {
        Name string `json:"name" binding:"required"`
    }
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusCreated, gin.H{"user": user})
}
```

**安装：**
```bash
go get github.com/gin-gonic/gin
```

### Echo（极简 Web 框架）

```go
package main

import (
    "github.com/labstack/echo/v4"
    "net/http"
)

func main() {
    e := echo.New()
    
    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, Echo!")
    })
    
    e.Logger.Fatal(e.Start(":8080"))
}
```

---

## 4.2 数据库 ORM

### GORM（最流行的 Go ORM）

```go
package main

import (
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "log"
)

type User struct {
    ID   uint   `gorm:"primaryKey"`
    Name string
    Age  int
}

func main() {
    // 连接数据库
    dsn := "user:password@tcp(localhost:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    
    // 自动迁移
    db.AutoMigrate(&User{})

    // 插入数据（Create）
    user := User{Name: "Alice", Age: 25}
    db.Create(&user)

    // 查询（Read）
    var foundUser User
    db.First(&foundUser, user.ID)

    // 更新（Update）
    db.Model(&user).Update("Age", 26)

    // 删除（Delete）
    db.Delete(&user)
}
```

**安装：**
```bash
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

### SQLX（扩展 database/sql）

```go
package main

import (
    "github.com/jmoiron/sqlx"
    _ "github.com/go-sql-driver/mysql"
)

type User struct {
    ID   int    `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}

func main() {
    db, err := sqlx.Connect("mysql", "user:password@tcp(localhost:3306)/dbname")
    if err != nil {
        panic(err)
    }
    defer db.Close()
    
    // 查询到结构体
    var users []User
    db.Select(&users, "SELECT * FROM users WHERE age > ?", 18)
    
    // 命名查询
    namedQuery := `SELECT * FROM users WHERE name=:name AND age=:age`
    rows, _ := db.NamedQuery(namedQuery, map[string]interface{}{
        "name": "Alice",
        "age":  25,
    })
}
```

---

## 4.3 Redis 客户端

### go-redis

```go
package main

import (
    "context"
    "github.com/go-redis/redis/v8"
)

func main() {
    ctx := context.Background()
    
    // 创建客户端
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,
    })
    
    // 基本操作
    rdb.Set(ctx, "key", "value", 0)
    val, _ := rdb.Get(ctx, "key").Result()
    
    // Hash 操作
    rdb.HSet(ctx, "user:1", "name", "Alice", "age", "25")
    name, _ := rdb.HGet(ctx, "user:1", "name").Result()
    
    // List 操作
    rdb.LPush(ctx, "list", "item1", "item2")
    items, _ := rdb.LRange(ctx, "list", 0, -1).Result()
    
    // 发布订阅
    pubsub := rdb.Subscribe(ctx, "channel")
    msg, _ := pubsub.ReceiveMessage(ctx)
}
```

**安装：**
```bash
go get github.com/go-redis/redis/v8
```

---

## 4.4 配置管理

### Viper（强大的配置管理）

```go
package main

import (
    "fmt"
    "github.com/spf13/viper"
)

func main() {
    // 设置配置文件名和路径
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    
    // 读取配置文件
    if err := viper.ReadInConfig(); err != nil {
        panic(fmt.Errorf("读取配置失败: %w", err))
    }
    
    // 读取配置值
    dbHost := viper.GetString("database.host")
    dbPort := viper.GetInt("database.port")
    
    // 支持环境变量
    viper.AutomaticEnv()
    dbHost = viper.GetString("DB_HOST")
    
    // 设置默认值
    viper.SetDefault("database.port", 3306)
}
```

**配置文件示例（config.yaml）：**
```yaml
database:
  host: localhost
  port: 3306
  username: root
  password: password

server:
  port: 8080
  timeout: 30s
```

**安装：**
```bash
go get github.com/spf13/viper
```

---

## 4.5 日志库

### Zap（Uber 开发的高性能日志库）

```go
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func main() {
    // 开发环境
    logger, _ := zap.NewDevelopment()
    defer logger.Sync()

    // 自定义配置
    config := zap.NewProductionConfig()
    config.Level = zap.NewAtomicLevelAt(zapcore.DebugLevel)
    logger, _ := config.Build()
    
    // 使用
    logger.Info("用户登录",
        zap.String("username", "alice"),
        zap.Int("user_id", 123),
    )
    
    logger.Error("数据库连接失败",
        zap.String("error", "connection timeout"),
        zap.String("host", "localhost:3306"),
    )
    
    // 使用 Sugar
    sugar := logger.Sugar()
    sugar.Infof("用户 %s 登录成功，ID: %d", "alice", 123)
}
```

**安装：**
```bash
go get go.uber.org/zap
```

### Logrus（经典日志库）

```go
package main

import (
    "github.com/sirupsen/logrus"
)

func main() {
    // 设置日志级别
    logrus.SetLevel(logrus.DebugLevel)
    
    // 设置格式
    logrus.SetFormatter(&logrus.JSONFormatter{})
    
    // 使用
    logrus.WithFields(logrus.Fields{
        "username": "alice",
        "user_id":  123,
    }).Info("用户登录")
    
    logrus.Error("数据库连接失败")
}
```

---

## 4.6 测试框架

### Testify（断言和 Mock 工具）

```go
package main

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/suite"
)

// 基本断言
func TestBasic(t *testing.T) {
    assert.Equal(t, 123, 123, "应该相等")
    assert.NotNil(t, []int{1, 2, 3}, "不应该为 nil")
    assert.Contains(t, "Hello World", "World")
}

// Require（失败立即停止）
func TestRequire(t *testing.T) {
    require.Equal(t, 123, 123)
}

// Suite（测试套件）
type MyTestSuite struct {
    suite.Suite
    data []int
}

func (suite *MyTestSuite) SetupTest() {
    suite.data = []int{1, 2, 3}
}

func (suite *MyTestSuite) TestExample() {
    suite.Equal(3, len(suite.data))
}

func TestMyTestSuite(t *testing.T) {
    suite.Run(t, new(MyTestSuite))
}
```

**安装：**
```bash
go get github.com/stretchr/testify
```

---

## 4.7 工具库

### UUID 生成

```go
package main

import (
    "fmt"
    "github.com/google/uuid"
)

func main() {
    // 生成 UUID v4
    id := uuid.New()
    fmt.Println(id.String())
    
    // 解析 UUID
    parsed, _ := uuid.Parse("550e8400-e29b-41d4-a716-446655440000")
    fmt.Println(parsed)
}
```

### 验证器

```go
package main

import (
    "fmt"
    "github.com/go-playground/validator/v10"
)

type User struct {
    Name  string `validate:"required,min=3,max=20"`
    Email string `validate:"required,email"`
    Age   int    `validate:"gte=18,lte=100"`
}

func main() {
    validate := validator.New()
    
    user := User{
        Name:  "Alice",
        Email: "alice@example.com",
        Age:   25,
    }

    if err := validate.Struct(user); err != nil {
        fmt.Println("验证失败:", err)
    } else {
        fmt.Println("验证通过")
    }
}
```

[src: raw/ingested/2技术/go/第三方库-go库使用-四、常用库分类和使用.md]