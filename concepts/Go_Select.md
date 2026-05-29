# Go Select 多路复用

## 9.1 Select 基本用法

**Select 用于同时等待多个 Channel 操作，类似于 switch 语句。**
Go 语言有 switch 语句。select 和 switch 语句类似，但 select 用于同时等待多个 Channel 操作，而 switch 用于条件分支选择。

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "消息 1"
    }()
    
    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "消息 2"
    }()
    
    // select 会阻塞直到有一个 case 准备好
    select {
    case msg1 := <-ch1:
        fmt.Printf("从 ch1 接收: %s\n", msg1)
    case msg2 := <-ch2:
        fmt.Printf("从 ch2 接收: %s\n", msg2)
    }
    
    fmt.Println("select 完成")
}
```

## 9.2 Select 的 default 分支（非阻塞）

```go
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int)
    
    // 非阻塞发送
    select {
    case ch <- 42:
        fmt.Println("发送成功")
    default:
        fmt.Println("发送失败（Channel 已满或未准备好）")
    }
    
    // 非阻塞接收
    select {
    case value := <-ch:
        fmt.Printf("接收成功: %d\n", value)
    default:
        fmt.Println("接收失败（Channel 为空）")
    }
}
```

## 9.3 Select 超时控制

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string)
    
    // 模拟一个可能长时间运行的 Goroutine
    go func() {
        time.Sleep(2 * time.Second)
        ch <- "结果"
    }()
    
    // 超时控制
    timeout := time.After(1 * time.Second)
    
    select {
    case result := <-ch:
        fmt.Printf("收到结果: %s\n", result)
    case <-timeout:
        fmt.Println("操作超时")
    }
}
```

## 9.4 Select 的随机性

**当多个 case 同时准备好时，select 会随机选择一个执行。**

```go
package main

import (
    "fmt"
)

func main() {
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    
    // 同时准备好两个 Channel
    ch1 <- 1
    ch2 <- 2
    
    select {
    case v1 := <-ch1:
        fmt.Printf("选择了 ch1: %d\n", v1)
    case v2 := <-ch2:
        fmt.Printf("选择了 ch2: %d\n", v2)
    }
    
    // 多次运行，结果可能不同（随机选择）
}
```

[src: raw/ingested/2技术/go/go同步-九、Select-多路复用.md]