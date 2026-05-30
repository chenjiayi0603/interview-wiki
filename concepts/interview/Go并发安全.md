# Go并发安全

## Sync包

### Mutex (互斥锁)

```go
// Mutex用于保护共享资源，同一时刻只允许一个goroutine访问
var mu sync.Mutex

mu.Lock()         // 获取锁，如果锁已被占用则阻塞
defer mu.Unlock() // 确保函数返回时释放锁
// 临界区代码
```

**运行示例**：
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()         // 获取锁，如果锁已被占用则阻塞等待
    defer mu.Unlock() // 确保函数返回时释放锁(即使发生panic也会释放)
    counter++         // 临界区代码：受锁保护的共享资源访问
    fmt.Printf("Counter: %d\n", counter)
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
            time.Sleep(10 * time.Millisecond)
        }()
    }
    wg.Wait()
    fmt.Println("最终Counter:", counter)
}
```

**运行结果**：
```
Counter: 1
Counter: 2
Counter: 3
Counter: 4
Counter: 5
最终Counter: 5
```

**注意**：Mutex不可重入，同一goroutine重复Lock会导致死锁
- **完整示例**（以下程序会死锁，仅作说明用，勿直接运行）：
```go
package main
import "sync"
func main() {
    var mu sync.Mutex
    mu.Lock()
    mu.Lock() // 同一 goroutine 再次 Lock，死锁
}
```
- 正确用法：`mu.Lock(); defer mu.Unlock()` 保护临界区

### RWMutex (读写锁)

- 多个读锁可并发，写锁互斥
  - **性能优势**：读操作不互斥，适合读多写少的场景，性能优于Mutex
- 适合读多写少场景
  - **注意**：如果写操作频繁，RWMutex的性能可能不如Mutex，因为RWMutex内部有更复杂的逻辑

**运行示例**：
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var (
	cache = make(map[string]string)
	rw    sync.RWMutex
)

func Get(k string) string {
	rw.RLock()
	defer rw.Unlock()
	return cache[k]
}

func Set(k, v string) {
	rw.Lock()
	defer rw.Unlock()
	cache[k] = v
}

func main() {
	Set("name", "Go")
	Set("version", "1.21")

	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			// 多个goroutine可同时Get，Set时独占
			fmt.Printf("G%d Get name=%s\n", id, Get("name"))
			time.Sleep(10 * time.Millisecond)
			fmt.Printf("G%d Get version=%s\n", id, Get("version"))
		}(i)
	}
	wg.Wait()
	fmt.Println("最终 cache:", Get("name"), Get("version"))
}
```

**运行结果**：
```
G0 Get name=Go
G2 Get name=Go
G1 Get name=Go
G4 Get name=Go
G3 Get name=Go
G2 Get version=1.21
G0 Get version=1.21
...
最终 cache: Go 1.21
```

### WaitGroup (等待组)

```go
// WaitGroup用于等待一组goroutine完成
var wg sync.WaitGroup

wg.Add(1)  // 增加等待计数
go func() {
    defer wg.Done()  // 完成时减少计数
    // 工作
}()
wg.Wait()  // 阻塞直到计数为0
```

**运行示例**：
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()  // 完成时通知WaitGroup，减少计数(即使发生panic也会执行)
    fmt.Printf("Worker %d 开始工作\n", id)
    time.Sleep(100 * time.Millisecond)  // 模拟工作
    fmt.Printf("Worker %d 完成工作\n", id)
}

func main() {
    var wg sync.WaitGroup
    
    // 启动3个worker
    for i := 1; i <= 3; i++ {
        wg.Add(1)  // 增加等待计数
        go worker(i, &wg)
    }
    
    fmt.Println("等待所有worker完成...")
    wg.Wait()  // 阻塞直到所有worker完成
    fmt.Println("所有worker已完成")
}
```

**运行结果**：
```
等待所有worker完成...
Worker 1 开始工作
Worker 2 开始工作
Worker 3 开始工作
Worker 2 完成工作
Worker 1 完成工作
Worker 3 完成工作
所有worker已完成
```

### Once (单例)

```go
// Once保证某个操作只执行一次，即使多个goroutine同时调用
var once sync.Once

once.Do(func() {
    // 这个函数只会执行一次，即使多次调用once.Do()
})
```

**运行示例**：
```go
package main

import (
    "fmt"
    "sync"
)

var (
    instance string
    once     sync.Once
)

func initInstance() {
    fmt.Println("初始化实例（只执行一次）")
    instance = "单例实例"
}

func getInstance() string {
    once.Do(initInstance)  // 只执行一次，即使多个goroutine同时调用也是线程安全的
    return instance
}

func main() {
    var wg sync.WaitGroup
    
    // 多个goroutine同时获取实例
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            inst := getInstance()
            fmt.Printf("Goroutine %d 获取到: %s\n", id, inst)
        }(i)
    }
    
    wg.Wait()
}
```

**运行结果**：
```
初始化实例（只执行一次）
Goroutine 0 获取到: 单例实例
Goroutine 1 获取到: 单例实例
Goroutine 2 获取到: 单例实例
Goroutine 3 获取到: 单例实例
Goroutine 4 获取到: 单例实例
```

## 原子操作

**atomic包**：
- 保证操作的原子性，无需加锁
- 适用于简单计数器场景

```go
// atomic包提供原子操作，无需加锁即可保证操作的原子性
var count int32

atomic.AddInt32(&count, 1)        // 原子性地增加count
val := atomic.LoadInt32(&count)   // 原子性地读取count
```

**运行示例**：
```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

var count int32

func increment() {
    atomic.AddInt32(&count, 1)  // 原子操作，线程安全，无需加锁
    // 注意：count++ 不是原子操作，多核环境下需要使用atomic保证
}

func main() {
    var wg sync.WaitGroup
    
    // 启动100个goroutine并发增加计数
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
        }()
    }
    
    wg.Wait()
    finalCount := atomic.LoadInt32(&count)
    fmt.Printf("最终计数: %d\n", finalCount)
}
```

**运行结果**：
```
最终计数: 100
```

**注意**：对地址的赋值操作不一定是原子的，多核环境下需要atomic保证
- **重要提醒**：即使看起来是简单的赋值操作(如`count++`)，在多核环境下也不是原子的，必须使用atomic保证原子性
- **适用场景**：简单计数器、标志位等，复杂逻辑仍需要锁保护

[src: raw/ingested/2技术/go/Go大厂考点复习文档.md]

## Related Pages
- [[Go_Atomic]]
- [[Goroutine调度]]
- [[Channel]]
- [[Go语言基础]]
- [[Go高频面试问题]]