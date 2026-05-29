# Go Atomic（原子操作）

## 8.1 Atomic 基本用法

**Atomic 提供底层的原子操作，适用于简单的计数器、标志位等场景，性能高于 Mutex。**

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

func main() {
    var counter int64
    var wg sync.WaitGroup
    
    // 使用 atomic 进行原子操作
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1)
        }()
    }
    
    wg.Wait()
    fmt.Printf("最终计数器值: %d\n", atomic.LoadInt64(&counter))
    
    // Compare-And-Swap (CAS) 操作
    oldValue := atomic.LoadInt64(&counter)
    newValue := oldValue + 100
    // CompareAndSwapInt64 用于比较并在相等时交换值：
    // 如果 *counter 等于 oldValue，则将 newValue 写入 *counter，并返回 true；否则返回 false。
    swapped := atomic.CompareAndSwapInt64(&counter, oldValue, newValue)
    fmt.Printf("CAS 操作结果: %v, 新值: %d\n", swapped, atomic.LoadInt64(&counter))
}
```

## 8.2 Atomic 常用操作

```go
package main

import (
    "fmt"
    "sync/atomic"
)

func main() {
    var value int64
    
    // 原子加法
    atomic.AddInt64(&value, 10)
    fmt.Printf("AddInt64: %d\n", value)
    
    // 原子加载
    loaded := atomic.LoadInt64(&value)
    fmt.Printf("LoadInt64: %d\n", loaded)
    
    // 原子存储
    atomic.StoreInt64(&value, 100)
    // atomic.LoadInt64(&value) 返回当前 value 的值，并且这个读取是原子的（线程安全的）。
    fmt.Printf("StoreInt64: %d\n", atomic.LoadInt64(&value))
    
    // 原子交换
    // 原子交换操作，将 value 的当前值存入 old，同时把 200 赋给 value
    old := atomic.SwapInt64(&value, 200)
    fmt.Printf("SwapInt64: 旧值=%d, 新值=%d\n", old, atomic.LoadInt64(&value))
    
    // Compare-And-Swap
    swapped := atomic.CompareAndSwapInt64(&value, 200, 300)
    fmt.Printf("CAS 成功: %v, 当前值: %d\n", swapped, atomic.LoadInt64(&value))
    
    swapped = atomic.CompareAndSwapInt64(&value, 100, 400)
    fmt.Printf("CAS 失败: %v, 当前值: %d\n", swapped, atomic.LoadInt64(&value))
}
```

## 8.3 Atomic.Value 用于任意类型

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

type Config struct {
    Host string
    Port int
}

func main() {
    var config atomic.Value
    
    // 初始配置
    // Store 方法将给定的值存进 atomic.Value 中，使其可被并发安全地读取。
    // Store 方法是线程安全的，可以安全地在多个 goroutine 中写入。
    config.Store(&Config{Host: "localhost", Port: 8080}) // 存储（设置）初始配置
    
    var wg sync.WaitGroup
    
    // 多个 Goroutine 读取配置
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // (*Config) 是类型断言，将 interface{} 类型转换为 *Config 指针类型
            // config 是 atomic.Value 类型，其底层通过 interface{} 存储任意类型的数据。
            // config.Load() 返回的是 interface{} 类型的数据，需要类型断言为 *Config 才能使用具体字段。
            // config.Load() 读取操作本身是线程安全的
            cfg := config.Load().(*Config)
            fmt.Printf("Goroutine %d 读取配置: %+v\n", id, cfg)
        }(i)
    }
    
    // 更新配置
    time.Sleep(50 * time.Millisecond)
    config.Store(&Config{Host: "example.com", Port: 9090})
    fmt.Println("配置已更新")
    
    wg.Wait()
}
```

[src: raw/ingested/2技术/go/go同步-八、Atomic（原子操作）.md]