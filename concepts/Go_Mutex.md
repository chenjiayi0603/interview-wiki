# Go Mutex（互斥锁）

## 3.1 Mutex 基本用法

**Mutex 用于保护共享资源，确保同一时刻只有一个 Goroutine 访问。**

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock() // 确保解锁
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

func main() {
    counter := &Counter{}
    
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Inc()
        }()
    }
    
    wg.Wait()
    fmt.Printf("最终值: %d\n", counter.Value()) // 输出: 1000
}
```

## 3.2 Mutex 的常见陷阱

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// 错误示例：忘记解锁
func badExample() {
    var mu sync.Mutex
    mu.Lock()
    // 忘记调用 mu.Unlock()
    // 导致死锁
}

// 错误示例：重复解锁
func badExample2() {
    var mu sync.Mutex
    mu.Lock()
    mu.Unlock()
    mu.Unlock() // panic: unlock of unlocked mutex
}

// 正确示例：使用 defer
func goodExample() {
    var mu sync.Mutex
    mu.Lock()
    defer mu.Unlock() // 确保函数退出时解锁
    // 临界区代码
}

// 错误示例：锁的范围过大
func badExample3(mu *sync.Mutex, data []int) {
    mu.Lock()
    defer mu.Unlock()
    // 不需要锁保护的操作
    time.Sleep(1 * time.Second) // 阻塞操作，持有锁过久
    
    // 需要锁保护的操作
    data[0] = 42
}

// 正确示例：缩小锁的范围
func goodExample2(mu *sync.Mutex, data []int) {
    // 不需要锁保护的操作
    time.Sleep(1 * time.Second)
    
    mu.Lock()
    // 只保护需要保护的操作
    data[0] = 42
    mu.Unlock()
}
```

## 3.3 Mutex 的零值可用性

```go
package main

import (
    "sync"
    "fmt"
)

// type 是用来定义新的结构体或类型的关键字。例如：
// type SafeCounter struct { ... }
// 表示声明一个名为 SafeCounter 的结构体类型
type SafeCounter struct {
    // sync.Mutex 的零值是未锁定的互斥锁
    // 可以直接使用，无需初始化
    mu sync.Mutex
    count int
}

// 符号 * 为啥放前面？因为 Go 语法规定方法接收者写法为 func (接收者类型) 方法名，
// 如果方法接收者是指针，就要在类型名前加 *，表示该方法可以修改接收者内容
// c 是接收者（receiver），代表调用该方法的 SafeCounter 实例。
// Go 方法语法 func (接收者类型) 方法名。
// 这里 (c *SafeCounter) 表示方法作用于 SafeCounter 的指针接收者。
func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func main() {
    // var 是 Go 语言中用于声明变量的关键字，此处声明了一个名为 counter 的 SafeCounter 类型变量
    var counter SafeCounter
    // 零值 Mutex 可以直接使用
    counter.Inc()
    fmt.Println("零值 Mutex 可用")
}
```

[src: raw/ingested/2技术/go/go同步-三、Mutex（互斥锁）.md]