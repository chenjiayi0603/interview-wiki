# Go 同步常见问题和陷阱

> 本文总结了 Go 并发编程中常见的三大问题和陷阱：Goroutine 泄漏、数据竞争（Data Race）和死锁，并给出正确的解决方案。

See also: [[Goroutine调度]], [[Go并发安全]], [[Go_Mutex]], [[Go_Atomic]], [[Channel]]

## 1. Goroutine 泄漏

### 问题描述

当一个 Goroutine 向无缓冲 Channel 发送数据，但主 Goroutine 不再接收时，该 Goroutine 会一直阻塞，导致泄漏。

```go
// 错误示例：Goroutine 泄漏
func leakExample() {
    ch := make(chan int)
    
    go func() {
        ch <- 42
        // 如果主 Goroutine 不接收，这个 Goroutine 会一直阻塞
    }()
    
    // 主 Goroutine 退出，子 Goroutine 泄漏
}
```

### 解决方案

#### 使用缓冲 Channel

```go
func noLeakExample() {
    ch := make(chan int, 1) // 缓冲 Channel
    
    go func() {
        ch <- 42
    }()
    
    // 或者确保接收
    // value := <-ch
}
```

#### 使用 Context 控制 Goroutine 生命周期

```go
func contextExample() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    go func() {
        for {
            select {
            case <-ctx.Done():
                return // 正常退出
            default:
                // 工作代码
            }
        }
    }()
}
```

## 2. 数据竞争（Data Race）

### 问题描述

多个 Goroutine 同时读写同一变量，且至少有一个是写操作，导致结果不确定。

```go
// 错误示例：数据竞争
func dataRaceExample() {
    var counter int
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // 数据竞争！
        }()
    }
    
    wg.Wait()
    fmt.Println(counter) // 结果不确定
}
```

### 解决方案

#### 使用 Mutex

```go
func mutexExample() {
    var counter int
    var mu sync.Mutex
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }
    
    wg.Wait()
    fmt.Println(counter) // 输出: 1000
}
```

#### 使用 Atomic

```go
func atomicExample() {
    var counter int64
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1)
        }()
    }
    
    wg.Wait()
    fmt.Println(counter) // 输出: 1000
}
```

## 3. 死锁

### 问题描述

两个或多个 Goroutine 互相等待对方持有的锁，导致所有 Goroutine 永久阻塞。

```go
// 错误示例：死锁
func deadlockExample() {
    var mu1, mu2 sync.Mutex
    
    go func() {
        mu1.Lock()
        defer mu1.Unlock()
        
        mu2.Lock()
        defer mu2.Unlock()
        
        fmt.Println("Goroutine 1 完成")
    }()
    
    go func() {
        mu2.Lock()
        defer mu2.Unlock()
        
        mu1.Lock()
        defer mu1.Unlock()
        
        fmt.Println("Goroutine 2 完成")
    }()
    
    // 死锁：两个 Goroutine 互相等待对方持有的锁
}
```

### 解决方案

保持锁的获取顺序一致。

```go
func noDeadlockExample() {
    var mu1, mu2 sync.Mutex
    
    go func() {
        mu1.Lock()
        defer mu1.Unlock()
        
        mu2.Lock()
        defer mu2.Unlock()
        
        fmt.Println("Goroutine 1 完成")
    }()
    
    go func() {
        mu1.Lock() // 保持相同的顺序
        defer mu1.Unlock()
        
        mu2.Lock()
        defer mu2.Unlock()
        
        fmt.Println("Goroutine 2 完成")
    }()
}
```

[src: raw/ingested/2技术/go/go同步-十三、常见问题和陷阱.md]