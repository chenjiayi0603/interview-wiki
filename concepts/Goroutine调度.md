# Goroutine 调度

## 一、Goroutine（协程）

### 1.1 Goroutine 基本概念

**Goroutine 是 Go 语言的轻量级线程，由 Go 运行时（runtime）管理。**

| 特性 | Goroutine | 系统线程（Thread） |
|------|-----------|-------------------|
| 内存占用 | 初始 2KB，最大 1GB（64位） | 约 1MB + guard page |
| 创建/销毁 | 用户态，直接申请 | 系统调用 |
| 切换开销 | 只需保存 3 个寄存器（PC、SP、BP） | 需要保存所有寄存器 |
| 调度方式 | Go 运行时调度（GMP 模型） | 操作系统内核调度 |
| 抢占式 | 是（Go 1.14+） | 是 |

### 1.2 Goroutine 创建

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 使用 go 关键字启动 Goroutine
    go func() {
        fmt.Println("Goroutine 1 执行")
    }()
    
    // 带参数的 Goroutine
    go func(name string) {
        fmt.Printf("Goroutine %s 执行\n", name)
    }("goroutine-2")
    
    // 主 Goroutine 等待，避免程序立即退出
    time.Sleep(100 * time.Millisecond)
    fmt.Println("主函数执行完成")
}
```

**go的Goroutine 为啥说是抢占式的？为啥c++的协程是协作式的？**
Go 的 Goroutine 被称为“抢占式”，是因为 Go 从 1.14 开始在运行时引入了抢占调度机制：当某个 Goroutine 执行时间太长或者陷入死循环时，Go 的调度器可以在一些安全点自动“打断”这个协程，将执行机会分配给其他 Goroutine，而不是完全依赖协程自己主动让出（比如通过 IO、channel 阻塞或调用 `runtime.Gosched()`）。这样可以保证程序中的所有 Goroutine 都有执行的机会，不会因为某个协程长时间占用 CPU 而“饿死”其他协程。

而 C++ 的协程属于“协作式调度”，它的切换完全依赖开发者在代码中明确写出挂起（如 `co_await`, `co_yield`, `co_return`）的位置。协程只有在这些挂起点才会让出控制权，如果协程内部没有挂起点（比如遇到死循环），调度器就无法自动将 CPU 分给其他协程，导致其他协程无法运行。这要求开发人员合理设计协程的挂起逻辑，否则很容易发生“饿死”问题。

总结：Go 的抢占式调度更像操作系统的线程，切换可以由系统自动决定；C++ 的协作式调度完全依赖开发者设计“让路”的时机。

**go的抢占调度机制 是怎么实现的？安全点又是啥？**
**Go 的抢占调度机制实现原理：**

1. **定期轮询（Preemptive Polling）**  
   Go 1.14 及之后，编译器在函数调用和循环等“安全点”内插入了轮询逻辑。每当 Goroutine 执行到这些安全点，就会检测调度器是否要求当前协程让出 CPU。如果发现自己已被抢占，当前执行会自动让出，调度器有机会切走这个 Goroutine。

2. **系统信号触发（利用抢占信号）**  
   运行时会向运行太久的线程注入信号（如 UNIX 下的 `SIGURG`），使其在适当时刻打断当前 Goroutine，检查是否需要被挂起，让其他 Goroutine 有机会运行。

3. **调度器“抢占标记”（runtime stack preemption）**  
   Go 调度器会标记某些需要被抢占的 Goroutine（如执行超过 10ms 的），并通过安全点机制，在最早遇到的安全点暂停其执行，切回调度逻辑实现抢占。

4. **协作与被动混合**  
   除了内置抢占，Goroutine 在阻塞（比如 IO、channel 阻塞、sleep 等）时也会触发调度。抢占机制则保证即使完全计算型的 Goroutine，长时间不阻塞也能被及时调度出去。

**什么是“安全点”？**

- 安全点（Safe Point）指的是编译器和 runtime 保证内存状态一致、可以安全切换线程或执行 GC 的位置。
- 只有在安全点，Goroutine 才能被安全地切换、挂起或中断（抢占）。
- 典型的安全点有：函数调用处、循环跳转处、部分内置语句（如 `return`），有 GC 的地方等。
- Go 编译器会自动在这些位置插入检测抢占的代码，无需用户干预。

---

**举例说明：**

```go
func busyLoop() {
    for {
        // Go 编译器通常在循环“开始处”插入“抢占检测”（也可能在某些场景下插入在跳转语句后），以便每次进入循环体时都能及时检测是否需要被抢占
        // 如果被标记为抢占，runtime 会切到别的 Goroutine
    }
}
```

**总结：**  
- Go 的“抢占调度”是利用安全点插桩和系统信号，确保即便没有主动让出 CPU，长时间执行的 Goroutine 也会被强制挂起。
- 安全点是实现抢占和垃圾回收的关键机制。


### 1.3 GMP 调度模型

**GMP 模型是 Go 语言并发调度的核心：**

- **G（Goroutine）**：协程，包含执行栈、状态和任务函数
- **M（Machine）**：操作系统线程，真正执行计算的资源
- **P（Processor）**：逻辑处理器，维护本地 Goroutine 队列，提供执行环境

**调度流程：**
1. 每个 P 维护一个本地 G 队列（Local Run Queue）
2. 新创建的 G 优先放入当前 P 的本地队列，满了则放入全局队列
3. M 从 P 的本地队列获取 G 执行
4. 如果 P 的本地队列为空，M 会从全局队列或其他 P 的本地队列窃取 G（work-stealing）

**调度策略：**
- **主动调度**：通过 `runtime.Gosched()` 主动让出执行权
- **被动调度**：发生阻塞操作（如 channel 阻塞、sleep、垃圾回收）
- **抢占调度**：执行时间过长的 G 或被系统调用阻塞的 G 会被抢占

### 1.4 Goroutine 与线程的区别

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // 查看当前 Goroutine 数量
    fmt.Printf("当前 Goroutine 数量: %d\n", runtime.NumGoroutine())
    
    // 查看 CPU 核心数
    fmt.Printf("CPU 核心数: %d\n", runtime.NumCPU())
    
    // 设置最大可同时执行的 Goroutine 数量（P 的数量）
    // 默认为 CPU 核心数
    runtime.GOMAXPROCS(4)
    
    // 创建多个 Goroutine
    for i := 0; i < 10; i++ {
        go func(id int) {
            fmt.Printf("Goroutine %d 执行\n", id)
            time.Sleep(100 * time.Millisecond)
        }(i)
    }
    
    time.Sleep(200 * time.Millisecond)
    fmt.Printf("最终 Goroutine 数量: %d\n", runtime.NumGoroutine())
}
```

[src: raw/ingested/2技术/go/go同步-一、Goroutine（协程）.md]