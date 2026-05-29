# Go Context（上下文）

**Context 用于在 Goroutine 树中传递取消信号、超时和值。**
**Goroutine 树指的是由父 goroutine 派生出子 goroutine，形成的类似树状的调用关系结构。在这种结构中，通常需要在整个 Goroutine 树中传递取消信号、超时或携带数据。Context 就是用于解决这个需求的工具。**

## Context 类型

| Context 类型 | 用途 |
|-------------|------|
| context.Background() | 根 Context，用于主函数、初始化 |
| context.TODO() | 不确定使用什么 Context 时使用 |
| context.WithCancel() | 可取消的 Context |
| context.WithTimeout() | 可超时的 Context |
| context.WithDeadline() | 可设置截止时间的 Context |
| context.WithValue() | 可携带值的 Context |

## Context 取消和超时

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func worker(ctx context.Context, name string, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("%s 收到取消信号: %v\n", name, ctx.Err())
            return
        default:
            fmt.Printf("%s 工作中...\n", name)
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    // 可取消的 Context
    ctx1, cancel1 := context.WithCancel(context.Background())
    var wg sync.WaitGroup
    
    wg.Add(1)
    go worker(ctx1, "Worker1", &wg)
    
    time.Sleep(2 * time.Second)
    cancel1()
    wg.Wait()
    
    fmt.Println("---")
    
    // 带超时的 Context
    ctx2, cancel2 := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel2()
    
    wg.Add(1)
    go worker(ctx2, "Worker2", &wg)
    wg.Wait()
    
    fmt.Println("---")
    
    // 带截止时间的 Context
    deadline := time.Now().Add(2 * time.Second)
    ctx3, cancel3 := context.WithDeadline(context.Background(), deadline)
    defer cancel3()
    
    wg.Add(1)
    go worker(ctx3, "Worker3", &wg)
    wg.Wait()
}
```

## Context 传值

```go
package main

import (
    "context"
    "fmt"
)

func process(ctx context.Context) {
    userID := ctx.Value("userID")
    role := ctx.Value("role")
    
    fmt.Printf("处理请求 - UserID: %v, Role: %v\n", userID, role)
}

func main() {
    ctx := context.WithValue(context.Background(), "userID", 12345)
    ctx = context.WithValue(ctx, "role", "admin")
    
    process(ctx)
}
```

## Context 使用原则

1. **不要将 Context 放在结构体中，应该作为第一个参数传递**
2. **使用 context.Background() 作为根 Context**
3. **Context 是线程安全的，可以在多个 Goroutine 中使用**
4. **只传递必要的值，不要传递可选参数**
5. **Context 应该显式传递，不要存储在全局变量中**

> **Q：Context 应该作为局部变量不该作为全局变量吗？**

**A：对，Context 绝不能作为全局变量使用。**

Context 的设计目标在于随请求链路显式传递，用于控制超时、取消信号、携带链路信息等。如果将 Context 存为全局变量：
- 失去了上下文的隔离性和请求唯一性，容易在多个并发请求间交叉污染，导致数据错乱和协程协作混乱。
- 不能正确表达携带"单个请求"的元数据（如 trace id/user id 等）。
- 违背 Context 的不可变性和临时性原则。

**最佳实践：**
- Context 应始终作为"函数的第一个参数"依次传递。
- 只在需要协同/链路透传的地方派生子 context。
- 不应该作为字段存放在结构体，也绝不能定义为包/全局变量。

**错误示例**：
```go
var globalCtx = context.Background() // 错误！不要这样用
```

**正确用法**：
```go
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    doSomething(ctx)
}
```

简言之：**Context 始终作为参数明示传递，绝不作为全局变量持有。**

## sync 与 context 对比

- **sync 包**：Go 标准库中的"同步原语"，解决并发安全问题（如数据竞争）。典型的有：sync.Mutex、sync.RWMutex、sync.WaitGroup、sync.Once、sync.Cond。
- **context 包**：跨 Goroutine 的控制信号传递，解决"协作式取消"、超时、截止时间和元数据传递。

**总结**：sync 用于 goroutine 之间对共享数据的互斥与同步；context 通常用于任务的协作取消、异常退出和跨 API 的请求信息传递，两者功能互补。

## struct{} 零字节类型

struct{} 是 Go 语言中的一种"空结构体类型"，它不包含任何字段，也不占用空间，是零字节的。常见用途包括：

1. 作为信号的占位符（如 chan struct{}）
2. 做 map 的 value，表示"集合"（set）
3. 作为约定无需保存具体数据但需要标记类型的时候

struct{} 之所以能做到 0 字节，是因为 Go 编译器对空结构体有特殊优化：空结构体 struct{} 没有任何字段，因此它不需要占用任何存储空间。在内存布局上，编译器会让所有 struct{} 的实例指向同一个地址，并且其大小为 0 字节。

[src: raw/ingested/2技术/go/go同步-十、Context（上下文）-十、Context（上下文）.md]