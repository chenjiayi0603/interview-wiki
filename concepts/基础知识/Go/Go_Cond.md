# Go Cond（条件变量）

> Cond 用于在满足特定条件时唤醒等待的 Goroutine。

## 7.1 Cond 基本用法

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Queue struct {
    items []int
    mu    sync.Mutex
    cond  *sync.Cond
}

func NewQueue() *Queue {
    q := &Queue{
        items: make([]int, 0),
    }
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Enqueue(item int) {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    q.items = append(q.items, item)
    fmt.Printf("入队: %d\n", item)
    q.cond.Signal() // 唤醒一个等待的 Goroutine
}

func (q *Queue) Dequeue() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    // 如果队列为空，等待
    for len(q.items) == 0 {
        fmt.Println("队列为空，等待...")
        q.cond.Wait() // 释放锁并阻塞，被唤醒后重新获取锁
    }
    
    item := q.items[0]
    q.items = q.items[1:]
    fmt.Printf("出队: %d\n", item)
    return item
}

func main() {
    queue := NewQueue()
    
    // 消费者 Goroutine
    go func() {
        for i := 0; i < 5; i++ {
            queue.Dequeue()
        }
    }()
    
    time.Sleep(100 * time.Millisecond)
    
    // 生产者 Goroutine
    for i := 1; i <= 5; i++ {
        queue.Enqueue(i)
        time.Sleep(50 * time.Millisecond)
    }
    
    time.Sleep(200 * time.Millisecond)
}
```

## 7.2 Cond 的 Broadcast 和 Signal

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var mu sync.Mutex
    cond := sync.NewCond(&mu)
    ready := false
    
    // 等待者 Goroutines
    for i := 0; i < 5; i++ {
        go func(id int) {
            mu.Lock()
            defer mu.Unlock()
            
            for !ready {
                fmt.Printf("Goroutine %d 等待条件满足...\n", id)
                // cond.Wait() 可能会发生虚假唤醒，因此要用 for 检查条件
                // cond.Wait() 本身并不检查条件，只是让当前 Goroutine 阻塞并释放锁
                // 需要配合 for 循环显式检查条件，否则可能因为虚假唤醒导致错误
                cond.Wait()
            }
            
            fmt.Printf("Goroutine %d 被唤醒，条件已满足\n", id)
        }(i)
    }
    
    time.Sleep(500 * time.Millisecond)
    
    // 发送信号
    mu.Lock()
    ready = true
    fmt.Println("条件已满足，唤醒所有等待者")
    cond.Broadcast() // 唤醒所有等待的 Goroutine
    mu.Unlock()
    
    time.Sleep(200 * time.Millisecond)
}
```

[src: raw/ingested/2技术/go/go同步-七、Cond（条件变量）.md]