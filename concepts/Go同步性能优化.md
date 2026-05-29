# Go 同步性能优化

> 本文涵盖 Go 中锁和 Channel 的性能优化建议，包括缩小锁范围、读写分离、使用 Atomic、缓冲 Channel 和批量处理等技巧。

See also: [[Go_Mutex]], [[Go并发安全]], [[Channel]]

## 14.1 锁的性能优化

```go
// 1. 缩小锁的范围
func optimizedLockExample() {
    var mu sync.Mutex
    var data []int
    
    // 不需要锁的操作
    result := expensiveComputation()
    
    mu.Lock()
    // 只保护必要的操作
    data = append(data, result)
    mu.Unlock()
}

// 2. 读写分离（RWMutex）
func rwLockExample() {
    var mu sync.RWMutex
    var data map[string]string
    
    // 读操作使用读锁
    func read(key string) string {
        mu.RLock()
        defer mu.RUnlock()
        return data[key]
    }
    
    // 写操作使用写锁
    func write(key, value string) {
        mu.Lock()
        defer mu.Unlock()
        data[key] = value
    }
}

// 3. 使用 Atomic 替代 Mutex（简单场景）
func atomicOptimization() {
    var counter int64
    
    // 使用 Atomic 比 Mutex 性能更好
    atomic.AddInt64(&counter, 1)
}
```

## 14.2 Channel 性能优化

```go
// 1. 使用缓冲 Channel 减少阻塞
func bufferedChannelExample() {
    // 无缓冲：发送和接收必须同时准备好
    ch1 := make(chan int)
    
    // 有缓冲：可以提前发送，减少阻塞
    ch2 := make(chan int, 100)
}

// 2. 批量处理减少 Channel 操作
func batchProcessing() {
    ch := make(chan []int, 10)
    
    // 批量发送
    batch := make([]int, 0, 100)
    for i := 0; i < 1000; i++ {
        batch = append(batch, i)
        if len(batch) >= 100 {
            ch <- batch
            // 重新赋值的原因：
            // 如果直接写 batch = nil 或 batch = batch[:0]，则底层数组还是同一个，会导致多个 goroutine 之间复用同一块内存（即 ch <- batch 后被修改），引发数据竞争或脏数据。
            // 用 make 分配新切片，则 batch 拥有独立底层数组，避免被并发读写。
            batch = make([]int, 0, 100)
        }
    }
    if len(batch) > 0 {
        ch <- batch
    }
}
```

[src: raw/ingested/2技术/go/go同步-十四、性能优化建议.md]