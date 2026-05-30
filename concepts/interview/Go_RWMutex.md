# Go RWMutex（读写锁）

## 基本用法

**RWMutex 允许多个读操作同时进行，但写操作是独占的。**

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type DataStore struct {
    mu   sync.RWMutex
    data map[string]string
}

// Go 规范中，通常以 New 类型名 的方式（比如 NewDataStore）来命名构造函数，表示“构造并返回类型的新实例”。这样命名有助于读者一眼识别出这是一个工厂函数或构造器。
// 以 New 开头可以直观表达“创建一个新的 DataStore”语义，符合 Go 生态的通用习惯与社区规范。
// 可以不用 New，直接声明变量或使用字面量
// 例1：直接变量声明
// var store DataStore
// store.data = make(map[string]string)

// 例2：用字面量初始化
// store := DataStore{data: make(map[string]string)}

// 或者保留工厂函数（仅为方便），名字不用 New 也行，例如：
// func CreateDataStore() *DataStore {
func NewDataStore() *DataStore {
    return &DataStore{
        data: make(map[string]string),
    }
}

func (ds *DataStore) Read(key string) string {
    // 只有访问共享资源（如 map）时才需要加锁。
    // 如果你要读写的变量可能被多个 goroutine 并发访问（如 map、slice、计数器等），则必须加锁；
    // 如果变量只在单个 goroutine 内部使用、不会被其他 goroutine 访问，则是 goroutine 私有的，无需加锁。
    // 例子：
    // func example() {
    //     var count int
    //     go func() {
    //         // 这里虽然是在 goroutine 中访问外部的 count 变量，
    //         // 但因为 count 是 example 函数的局部变量，理论上 example 返回后 count 就不再可用，
    //         // 但实际上，由于闭包被 goroutine 捕获且 goroutine 尚未结束，count 的生命周期会被延长直到 goroutine 结束，
    //         // Go 的 GC 会保证只要 goroutine 用到，count 就不会被回收。即：只要协程没结束，count 仍然存活。
    //         count++
    //         fmt.Println(count)
    //     }()
    // }
    // 此处访问 ds.data，是共享资源，必须加读锁。
    ds.mu.RLock()         // 读锁
    defer ds.mu.RUnlock() // 释放读锁
    return ds.data[key]
}
//     // C++20 协程的闭包（如 lambda 捕获、coroutine promise 持有）是否能延长外部变量生命周期要视具体捕获方式：
//     // （1）如果 lambda 按引用捕获外部变量，则该变量必须在 lambda 或协程体有效期间活着，否则会悬垂引用（UB）。
//     // （2）如果 lambda 按值捕获，则捕获的是变量的一份拷贝，生命周期与 lambda/协程 promise 绑定，被延长。
//     //     比如 co_yield/capture 到 promise 存储中，则引用计数等会延长数据活期。
//     //     若 promise_type 中主动保存了某对象，确实能延长。
//     // 总结：C++ 闭包本身不会自动延长局部变量生命周期。引用捕获无延长，值捕获（拷贝）在协程对象存活期内有效。
//     // 详见 C++ lambda/coroutine 对象生命周期分析。


func (ds *DataStore) Write(key, value string) {
    ds.mu.Lock()         // 写锁
    defer ds.mu.Unlock() // 释放写锁
    ds.data[key] = value
}

func main() {
    store := NewDataStore()
    store.Write("key1", "value1")
    
    // 问：WaitGroup 底层是 C++ 信号量的机制吗？
    // 答：不是。Go 的 sync.WaitGroup 是用 Go 自己实现的，并没有直接依赖 C++、也没有使用 C++ 的信号量机制。
    // Go 运行时（runtime）是 Go 语言程序在运行时依赖的底层支撑系统，负责协程调度、垃圾回收、内存管理、并发原语等核心功能。Go runtime 的绝大部分是用 Go 和汇编实现的，底层的部分原语（如 mutex、信号量等）可能用 C 实现，但并非直接复用 C 标准库，而是高度定制化，专门为 Go 的并发模型服务。Go runtime 与 C 语言有一定底层联系（如启动流程、系统调用等），但并不依赖 C++ 也和 C++ 的 STL/同步机制没有关系。
    // 在 Go 1.8+，WaitGroup 内部主要依赖 runtime 包里的信号量和原子操作（如 atomic increment/decrement、同步唤醒 goroutine 等），
    // 是操作系统级别和 Go runtime 层的互斥/信号机制，而不是 C++ 层的技术。
var wg sync.WaitGroup
    
    // 多个读操作可以并发执行
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            value := store.Read("key1")
            fmt.Printf("Goroutine %d 读取到: %s\n", id, value)
        }(i)
    }
    
    // 写操作会阻塞所有读操作
    time.Sleep(50 * time.Millisecond)
    wg.Add(1)
    go func() {
        defer wg.Done()
        store.Write("key1", "value2")
        fmt.Println("写操作完成")
    }()
    
    wg.Wait()
}
```

## 使用场景

| 场景 | 推荐使用 |
|------|---------|
| 读多写少 | RWMutex（性能更好） |
| 读写相当 | Mutex（实现简单） |
| 写多读少 | Mutex（RWMutex 开销更大） |

[src: raw/ingested/2技术/go/go同步-四、RWMutex（读写锁）.md]