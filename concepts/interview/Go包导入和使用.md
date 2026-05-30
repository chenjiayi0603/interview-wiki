# Go 包导入和使用

## 二、包的导入和使用

### 2.1 标准库导入

**Go 标准库无需额外安装，直接导入即可使用。**

```go
package main

import (
    "fmt"      // 格式化输出
    "net/http" // HTTP 服务器和客户端
    "os"       // 操作系统接口
    "time"     // 时间处理
    "sync"     // 同步原语
    "context"  // 上下文控制
)

func main() {
    fmt.Println("Hello, Go!")
}
```

### 2.2 第三方库导入

**第三方库需要先通过 `go get` 安装，然后导入使用。**

```go
package main

import (
    "fmt"
    "github.com/gin-gonic/gin"           // Web 框架
    "github.com/go-redis/redis/v8"       // Redis 客户端
    "gorm.io/gorm"                       // ORM 框架
    "gorm.io/driver/mysql"
)

func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "pong"})
    })
    r.Run()
}
```

**安装命令：**
```bash
go get github.com/gin-gonic/gin
go get github.com/go-redis/redis/v8
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

### 2.3 导入方式

#### 2.3.1 标准导入

```go
import "fmt"
import "net/http"
```

#### 2.3.2 分组导入（推荐）

```go
import (
    // 标准库
    "fmt"
    "net/http"
    "time"
    
    // 第三方库
    "github.com/gin-gonic/gin"
    "github.com/go-redis/redis/v8"
    
    // 本地包
    "./utils"
    "myapp/models"
)
```

#### 2.3.3 别名导入

```go
import (
    f "fmt"                    // 使用 f 代替 fmt
    redis "github.com/go-redis/redis/v8"  // 使用 redis 代替完整路径
    _ "github.com/lib/pq"      // 匿名导入（只执行 init 函数）
)
```

#### 2.3.4 点导入（不推荐，仅特殊场景）

```go
import . "fmt"  // 可以直接使用 Println 而不需要 fmt.Println

func main() {
    Println("Hello")  // 不需要 fmt 前缀
}
```

#### 2.3.5 匿名导入

```go
import _ "database/sql/driver"  // 只执行包的 init 函数，**不会导入变量/函数，不能直接用包的其它内容**。这样做通常用于注册驱动、插件等副作用性操作，比如数据库驱动注册。只执行 init 有什么用？就是为了让包自动初始化（如自动注册到全局列表），但你不能直接访问这个包的函数、变量。

// 常用于：
// 1. 数据库驱动注册
import _ "github.com/lib/pq"           // PostgreSQL
import _ "github.com/go-sql-driver/mysql"  // MySQL

// 2. 插件系统
import _ "myapp/plugins/plugin1"
```

[src: raw/ingested/2技术/go/第三方库-go库使用-二、包的导入和使用.md]