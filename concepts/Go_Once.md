# Go Once（只执行一次）

## 6.1 Once 基本用法

**Once 确保某个函数只执行一次，常用于单例模式、初始化等场景。**

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var once sync.Once
var config map[string]string

func initConfig() {
    fmt.Println("初始化配置...")
    config = make(map[string]string)
    config["database"] = "mysql"
    config["cache"] = "redis"
    // 模拟耗时操作
    time.Sleep(100 * time.Millisecond)
    fmt.Println("配置初始化完成")
}

func getConfig() map[string]string {
    // Do 方法会调用传入的函数，但只会执行一次（即使多个 Goroutine 并发调用也只执行一次 initConfig）
    // once.Do 内部实现已经加锁，并且通过原子操作（原子标记）保证只执行一次
    once.Do(initConfig)
    return config
}

func main() {
    var wg sync.WaitGroup
    
    // 多个 Goroutine 同时调用，但 initConfig 只会执行一次
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            cfg := getConfig()
            fmt.Printf("Goroutine %d 获取配置: %v\n", id, cfg)
        }(i)
    }
    
    wg.Wait()
}
```

## 6.2 Once 的实现原理

```go
// sync.Once 的简化实现原理
type Once struct {
    done uint32 // 标记是否已执行
    m    Mutex  // 互斥锁，保护 done
}

// // func(int) string 有参数有返回值的函数类型
// 指针接收者方法 (o *Once) Do 既可以通过 Once 结构体指针调用（如 once.Do(...)），
// 也可以通过结构体值调用（如 var o Once; o.Do(...)），Go 会自动取地址。
// 但如果方法接收者是值类型，则只能通过值调用（不能自动取址到赋值变量）。
// 选择指针接收者的原因是方法内部需要修改对象（如 Once 的字段）。
// 所以，调用者可以是指针也可以是值，Go 会自动处理。

func (o *Once) Do(f func()) {// 这里的 f 参数类型是 func()，即“无参数、无返回值”的函数类型
    // 快速路径：如果已经执行过，直接返回
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    // 这里还需要加锁（虽然 done 是原子变量），因为要保证 f() 只被一个 goroutine 执行，
    // 防止多个 goroutine 同时通过 done 检查后的并发执行 f()，加锁后只有一个 goroutine 能进入此区间
    o.m.Lock()
    defer o.m.Unlock()
    
    // 双重检查：再次确认是否已执行
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

[src: raw/ingested/2技术/go/go同步-六、Once（只执行一次）.md]