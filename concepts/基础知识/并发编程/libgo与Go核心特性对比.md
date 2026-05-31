# libgo 与 Go 核心特性对比

> 本文对比 libgo（C++ 协程库）与 Go 语言在协程创建、Channel 通信、同步原语、定时器等方面的核心特性。

## 一、协程创建

**Go:**
```go
// 使用 go 关键字创建协程
go func() {
    fmt.Println("Hello from goroutine")
}()

// 等待协程结束需要使用 sync.WaitGroup 或 channel
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // 工作代码
}()
wg.Wait()
```

**libgo:**
```cpp
#include <libgo/libgo.h>

// 使用 go 宏创建协程
go []{
    printf("Hello from coroutine\n");
};

// 启动调度器
co_sched.Start();

// 或者指定线程数
co_sched.Start(4); // 4个线程
```

## 二、Channel 通信

**Go:**
```go
// 创建 channel
ch := make(chan int, 10)  // 带缓冲
ch := make(chan int)       // 无缓冲

// 发送和接收
ch <- 42      // 发送
val := <-ch   // 接收

// select 多路复用
select {
case v := <-ch1:
    fmt.Println(v)
case ch2 <- x:
    fmt.Println("sent")
default:
    fmt.Println("no communication")
}
```

**libgo:**
```cpp
#include <libgo/libgo.h>

// 创建 channel
co_chan<int> ch(10);  // 带缓冲
co_chan<int> ch;      // 无缓冲

// 发送和接收
ch << 42;     // 发送
int val;
ch >> val;    // 接收

// 也支持 TryPush 和 TryPop
bool success = ch.TryPush(42);
bool success = ch.TryPop(val);

// 带超时的操作
ch.TimedPush(42, std::chrono::seconds(1));
ch.TimedPop(val, std::chrono::seconds(1));
```

## 三、同步原语

**Go:**
```go
// 互斥锁
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()

// 读写锁
var rwmu sync.RWMutex
rwmu.RLock()
defer rwmu.RUnlock()

// 条件变量
cond := sync.NewCond(&mu)
cond.Wait()
cond.Signal()
cond.Broadcast()
```

**libgo:**
```cpp
// 协程互斥锁 (不会阻塞线程，只阻塞协程)
co_mutex mtx;
{
    std::unique_lock<co_mutex> lock(mtx);
    // 临界区
}

// 协程读写锁
co_rwmutex rwmtx;
{
    // 读锁
    rwmtx.reader().lock();
    rwmtx.reader().unlock();
    
    // 写锁  
    rwmtx.writer().lock();
    rwmtx.writer().unlock();
}

// 协程条件变量
co_condition_variable cv;
cv.wait(lock);
cv.notify_one();
cv.notify_all();
```

## 四、定时器

**Go:**
```go
// 定时器
timer := time.NewTimer(time.Second)
<-timer.C  // 等待定时器触发

// Ticker 周期触发
ticker := time.NewTicker(time.Second)
for t := range ticker.C {
    fmt.Println("Tick at", t)
}

// 简单延时
time.Sleep(time.Second)
```

**libgo:**
```cpp
// 协程 sleep (不阻塞线程)
co_sleep(1000);  // 毫秒
co_sleep(std::chrono::seconds(1));

// 定时器
co_timer_add(std::chrono::seconds(1), []{
    printf("Timer fired!\n");
});
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-三、核心特性对比-三、核心特性对比.md]