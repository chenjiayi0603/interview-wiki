# Channel

## Channel类型

```go
package main

import "fmt"

// 无缓冲channel，同步模式：发送和接收必须同时准备好
// 有缓冲channel，容量10：缓冲区未满可发送，未空可接收，异步模式
// 只读/只写channel 用于函数参数约束方向，避免误用

func producer(ch chan<- int) { // 只写channel：只能发送数据
	ch <- 1
	ch <- 2
}

func consumer(ch <-chan int) { // 只读channel：只能接收数据
	fmt.Println(<-ch)
	fmt.Println(<-ch)
}

func main() {
	ch := make(chan int, 10)
	go producer(ch)
	consumer(ch)
	// 输出: 1 \n 2
}
```

**运行示例**：
```go
package main

import "fmt"

func main() {
    // 无缓冲channel示例：同步模式
    ch1 := make(chan int)
    go func() {
        ch1 <- 42  // 发送数据，会阻塞直到有接收者
    }()
    val := <-ch1  // 接收数据，会阻塞直到有数据(同步交接)
    fmt.Println("无缓冲channel接收:", val)  // 输出: 无缓冲channel接收: 42
    
    // 有缓冲channel示例：异步模式
    ch2 := make(chan int, 2)  // 容量为2，可以缓存2个值
    ch2 <- 1  // 不会阻塞，缓冲区有空间
    ch2 <- 2  // 不会阻塞，缓冲区有空间
    // ch2 <- 3  // 如果发送第3个值会阻塞，因为缓冲区已满
    fmt.Println("有缓冲channel接收:", <-ch2)  // 输出: 有缓冲channel接收: 1
    fmt.Println("有缓冲channel接收:", <-ch2)  // 输出: 有缓冲channel接收: 2
}
```

**运行结果**：
```
无缓冲channel接收: 42
有缓冲channel接收: 1
有缓冲channel接收: 2
```

**无缓冲vs有缓冲**：
- **无缓冲**：发送和接收必须同时准备好，同步交接
  - **使用场景**：需要同步的场景，如goroutine之间的同步点
- **有缓冲**：缓冲区未满可发送，未空可接收，异步
  - **使用场景**：生产者消费者模式，提高吞吐量，平滑流量

## Channel关闭规则

- 关闭已关闭的channel会panic
  - **注意**：关闭前检查channel状态，或使用sync.Once保证只关闭一次
  - **示例**：`close(ch); close(ch)` 第二次panic
- 向已关闭的channel写入会panic
  - **重要**：必须先关闭channel再通知接收端，或使用额外的signal channel
  - **示例**：`close(ch); ch <- 1` panic
- 从已关闭的channel读取：无缓冲返回零值，有缓冲可继续读取剩余值
- 判断关闭：`val, ok := <-ch`，ok为false表示已关闭

**关闭后读取完整示例**：
```go
package main
import "fmt"
func main() {
	ch := make(chan int, 2)
	ch <- 1
	ch <- 2
	close(ch)
	fmt.Println(<-ch, <-ch) // 1 2
	v, ok := <-ch
	fmt.Println(v, ok)     // 0 false，已关闭且无剩余
}
```

**关闭原则**：
- 只在发送端关闭channel
  - **原因**：发送端知道何时不再发送数据，避免接收端关闭时发送端还在发送导致panic
  - **示例**：生产者goroutine在发送完所有数据后 `close(ch)`，消费者用 `for v := range ch` 接收
- 多个发送者时，使用额外的signal channel通知关闭
  - **最佳实践**：使用`done` channel通知所有发送者停止发送，然后关闭数据channel
  - **示例**：`done := make(chan struct{}); 各发送者 select { case ch<-v: case <-done: return }; close(done)`

## Select

**Select特性**：
- 多路复用，随机选择就绪的case执行
  - **随机性**：如果多个case同时就绪，会随机选择一个，避免某个case一直不被选中(避免饥饿)
- 所有case都不就绪时执行default(非阻塞)
  - **非阻塞模式**：使用default可以实现非阻塞的channel操作，常用于定时、轮询等场景
- 无default时阻塞等待
  - **阻塞模式**：所有case都不就绪时会阻塞，直到至少有一个case就绪

**典型用法**：
```go
package main

import (
	"context"
	"fmt"
	"time"
)

func process(data int) { fmt.Println("处理:", data) }

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	ch := make(chan int, 1)
	value := 100

	go func() {
		time.Sleep(50 * time.Millisecond)
		ch <- 42
	}()

	// select多路复用：随机选择就绪的case执行
	select {
	case <-ctx.Done():
		return
	case data := <-ch:
		process(data) // 输出: 处理: 42
	case ch <- value:
		fmt.Println("发送成功")
	default:
		fmt.Println("非阻塞，无就绪case")
	}
}
```

**运行示例**：
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    // 启动goroutine发送数据
    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "来自ch1"
    }()
    
    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "来自ch2"
    }()
    
    // select会随机选择就绪的case（这里ch1先就绪）
    select {
    case msg1 := <-ch1:
        fmt.Println("接收到:", msg1)
    case msg2 := <-ch2:
        fmt.Println("接收到:", msg2)
    case <-time.After(500 * time.Millisecond):
        fmt.Println("超时")
    }
    
    // 带default的非阻塞select
    select {
    case msg := <-ch1:
        fmt.Println("接收到:", msg)
    default:
        fmt.Println("没有数据，不阻塞")
    }
}
```

**运行结果**：
```
接收到: 来自ch1
没有数据，不阻塞
```

[src: raw/ingested/2技术/go/Go大厂考点复习文档.md]

## Related Pages
- [[Goroutine调度]]
- [[Go并发安全]]
- [[Go语言基础]]
- [[Go高频面试问题]]
