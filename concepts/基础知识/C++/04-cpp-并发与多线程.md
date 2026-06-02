# 并发与多线程

> std::thread、互斥锁、条件变量、原子操作、内存序、线程池、Linux 线程调度。

---

## 一、线程管理

### 1.1 std::thread

```cpp
#include <thread>

void task(int id) { /* ... */ }

std::thread t1(task, 1);          // 函数+参数
std::thread t2([] { /* ... */ }); // lambda

t1.join();   // 等待线程结束
t2.detach(); // 分离，独立运行

// 检查状态
if (t1.joinable()) t1.join();
```

### 1.2 线程局部存储（thread_local）

```cpp
thread_local int data = 0;  // 每线程独立副本
```

### 1.3 std::jthread（C++20）

```cpp
std::jthread jt(task);  // 析构时自动 join，支持中断请求
```

---

## 二、同步原语

### 2.1 互斥锁

| 类型 | 说明 |
|------|------|
| `std::mutex` | 标准互斥锁 |
| `std::recursive_mutex` | 同一线程可重复加锁 |
| `std::timed_mutex` | 支持超时加锁 `try_lock_for` |
| `std::shared_mutex` (C++17) | 读写锁，多读单写 |

**RAII 锁包装**：

```cpp
std::mutex mtx;
{
    std::lock_guard<std::mutex> lock(mtx);    // 构造加锁，析构解锁
    std::unique_lock<std::mutex> ulock(mtx);  // 更灵活，可手动 lock/unlock
    std::shared_lock<std::shared_mutex> slock(smtx); // 共享锁（读锁）
}

// 同时锁多个 mutex，避免死锁
std::scoped_lock lock(mtx1, mtx2);  // C++17
std::lock(mtx1, mtx2);              // C++11 等效
```

### 2.2 条件变量

```cpp
std::mutex mtx;
std::condition_variable cv;
std::queue<int> q;

// 生产者
void produce(int val) {
    {
        std::lock_guard<std::mutex> lock(mtx);
        q.push(val);
    }
    cv.notify_one();
}

// 消费者
void consume() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [this]{ return !q.empty(); });  // 唤醒后检查谓词
    int val = q.front(); q.pop();
}
```

### 2.3 信号量（C++20）

```cpp
std::counting_semaphore<10> sem(3);  // 最大 10，初始 3
sem.acquire();  // 减一，无资源时阻塞
sem.release();  // 加一
```

---

## 三、原子操作与内存序

### 3.1 std::atomic

```cpp
std::atomic<int> counter{0};

counter++;                // 原子自增
int x = counter.load();   // 原子读
counter.store(42);        // 原子写
int old = counter.exchange(100);  // 原子交换
bool ok = counter.compare_exchange_strong(expected, desired);  // CAS
```

### 3.2 内存序

| 内存序 | 说明 |
|--------|------|
| `memory_order_relaxed` | 仅保证原子性，不保证顺序 |
| `memory_order_acquire` | 之后的读写不能重排到之前 |
| `memory_order_release` | 之前的读写不能重排到之后 |
| `memory_order_acq_rel` | acquire + release |
| `memory_order_seq_cst` | 全局顺序一致（默认，最重） |

```cpp
std::atomic<bool> flag{false};  // xmemory_order_release
int data = 0;

// 线程1
data = 42;
flag.store(true, std::memory_order_release);

// 线程2
if (flag.load(std::memory_order_acquire)) {
    // 保证看到 data == 42
}
```

### 3.3 CAS 与无锁编程

```cpp
std::atomic<int> value{0};
int expected = value.load();
while (!value.compare_exchange_weak(expected, expected + 1)) {
    // expected 已被更新为当前值，重试
}
```

**ABA 问题**：CAS 中值从 A→B→A，无法检测到变化。可用 ` tagged_ptr` 或 `std::atomic<std::shared_ptr>` 解决。

---

## 四、异步任务

```cpp
// std::async
std::future<int> fut = std::async(std::launch::async, []{ return 42; });
int result = fut.get();  // 阻塞等待结果

// std::promise + std::future
std::promise<int> prom;
std::future<int> fut = prom.get_future();
std::thread([&prom]{ prom.set_value(42); }).detach();
int val = fut.get();
```

---

## 五、线程池实现要点

```cpp
class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop = false;
public:
    template<typename F>
    void enqueue(F&& f) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            tasks.emplace(std::forward<F>(f));
        }
        cv.notify_one();
    }
    // ... 工作线程循环取任务执行
};
```

---

## 六、Linux 线程调度策略

| 策略 | 类型 | 优先级 | 说明 |
|------|------|--------|------|
| `SCHED_OTHER` | 非实时（默认） | 静态 0 + nice | 公平调度，通用场景 |
| `SCHED_FIFO` | 实时 | 1~99 | 先到先服务，高优先级可抢占 |
| `SCHED_RR` | 实时 | 1~99 | FIFO + 时间片轮转 |

```cpp
pthread_attr_t attr;
pthread_attr_init(&attr);
pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
param.sched_priority = 50;
pthread_attr_setschedparam(&attr, &param);
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
```


## 七、快速决策表

| 需求 | 推荐工具 |
|------|----------|
| 临界区保护 | `std::mutex` + `lock_guard` |
| 多锁防死锁 | `std::scoped_lock` |
| 读多写少 | `std::shared_mutex` + `shared_lock` |
| 等待通知 | `std::condition_variable` |
| 计数器/标志位 | `std::atomic` |
| 限流/资源计数 | `std::counting_semaphore` |
| 异步结果 | `std::future`/`std::promise` |
| 并行循环 | `std::execution::par` / OpenMP / TBB |

---

## 八、面试高频追问

### Q1: `std::async` 与直接创建 `std::thread` 的选型区别？

| 维度 | `std::async` | `std::thread` |
|:----|:------------|:-------------|
| 返回值 | 返回 `future`，可获取结果 | 无返回值，需通过 promise/引用传回 |
| 异常处理 | 异常存储在 future，`get()` 时重抛 | 线程内异常需自行捕获，否则 `terminate` |
| 线程管理 | 系统决定是否新建线程或复用（延迟启动） | 必须显式 join/detach |
| 资源开销 | 可能使用线程池（实现相关） | 每次都新建线程 |
| 适用场景 | 异步任务 + 需要返回值 | 长期运行的后台线程、线程池管理 |

**建议**：一次性异步任务用 `std::async`；需要精细控制线程生命周期用 `std::thread`。

### Q2: `unique_lock` 比 `lock_guard` 灵活在哪？什么场景必须用？

| 功能 | `lock_guard` | `unique_lock` |
|:----|:-----------:|:-------------:|
| 构造加锁 | ✅ | ✅ |
| 析构解锁 | ✅ | ✅ |
| 手动 lock/unlock | ❌ | ✅ |
| 转移所有权 | ❌ | ✅（move） |
| 条件变量 wait | ❌ | ✅（必须） |
| 超时加锁 | ❌ | ✅（try_lock_for/until） |
| 开销 | 零额外 | 略大（维护状态标志） |

**必须用 unique_lock 的场景**：需要配合 `condition_variable::wait()`、需要提前 unlock、需要超时锁。

### Q3: 什么是虚假唤醒（spurious wakeup）？如何避免？

```cpp
// ❌ 错误：可能被虚假唤醒
cv.wait(lock);  // 醒来时不检查条件，可能条件仍然不满足

// ✅ 正确：带谓词的 wait
cv.wait(lock, []{ return !queue.empty(); });
// 等价于：
while (!queue.empty()) {
    cv.wait(lock);
}
```

**原因**：操作系统可能出于调度原因唤醒等待线程，即使没有 notify。**始终使用带谓词的 wait**。

### Q4: 自旋锁 vs 互斥锁的选择？

| 对比 | 自旋锁 | 互斥锁 |
|:----|:------|:-------|
| 等待方式 | 忙等待（CPU 空转） | 线程休眠，上下文切换 |
| 适用临界区 | 极短（< 几微秒） | 较长 |
| CPU 消耗 | 高（一直跑） | 低（休眠不占 CPU） |
| 适用场景 | 内核态、实时系统 | 大部分用户态场景 |

```cpp
// 用户态自旋锁（极少用，通常用 mutex 更好）
std::atomic_flag spinlock = ATOMIC_FLAG_INIT;
while (spinlock.test_and_set(std::memory_order_acquire)) {}  // 自旋
// 临界区
spinlock.clear(std::memory_order_release);
```

### Q5: 内存序中的 Release-Acquire 如何保证可见性？

```
线程1:                   线程2:
data = 42;              while (!flag.load(acquire)) {}
flag.store(true, release);  assert(data == 42);  // ✅ 保证看到
```

**核心原理**：
- **Release**：之前的所有写操作不能重排到 release 之后
- **Acquire**：之后的所有读操作不能重排到 acquire 之前
- 构成 **synchronizes-with** 关系：线程1的 release 与线程2的 acquire 配对，确保线程2能看到线程1 release 之前的所有写入

**不是所有原子操作都保证顺序**：`memory_order_relaxed` 只保证原子性，不保证顺序。

### Q6: 线程池的核心参数如何设置？

| 参数 | 依据 | 推荐值 |
|:----|:-----|:-------|
| 线程数 | CPU 核心数 × (1 + I/O等待时间/计算时间) | CPU 密集型 = 核心数；I/O 密集型 = 2×核心数或更高 |
| 任务队列容量 | 最大并发请求数 | 有界（防 OOM），配拒绝策略 |
| 空闲线程回收时间 | 任务到达间隔 | 60s（典型） |

**经验公式**（CPU 密集型）：
```
线程数 = CPU 核心数 + 1
```
**经验公式**（I/O 密集型）：
```
线程数 = CPU 核心数 × (1 + I/O等待时间 / CPU计算时间)
```

### Q7：CAS 的 ABA 问题及解决方案？

```cpp
// ABA 问题：值从 A→B→A，CAS 误认为未被修改
std::atomic<int> val{100};
// 线程1：期望 100 → 改为 200
// 线程2：100 → 200 → 100（中间被改过）
// 线程1 的 CAS 成功，但实际已被修改

// 解决：使用 tagged_ptr（版本号指针）
std::atomic<uint64_t> tagged;  // 低 48 位指针，高 16 位版本号
// 或：std::atomic<std::shared_ptr<T>>（C++20）
```

### Q8：进程间同步与线程间同步的区别？

| 维度 | 线程同步 | 进程同步 |
|:----|:---------|:---------|
| 共享方式 | 共享内存（同一地址空间） | 共享内存 + 同步原语 |
| mutex | `std::mutex`（线程） | `pthread_mutex_t` with `PTHREAD_PROCESS_SHARED` |
| 原子操作 | `std::atomic`（线程） | 需放在共享内存中 |
| 条件变量 | `std::condition_variable` | `pthread_cond_t` with process-shared |
| 信号量 | `std::counting_semaphore`（C++20） | `sem_t` with `pshared=1` |
| 性能 | 最快（同一进程） | 稍慢（跨进程上下文切换） |

### Q9：std::call_once 与双重检查锁定的关系？

```cpp
// ✅ C++11 起：std::call_once 更安全
static std::once_flag flag;
std::call_once(flag, []{ /* 初始化一次 */ });

// ❌ 双重检查锁定（DCLP）在 C++ 中不安全
// 问题：指令重排序导致另一个线程看到未完全构造的对象
static Singleton* instance = nullptr;
if (!instance) {          // 第一次检查
    lock_guard lock(mtx);
    if (!instance) {      // 第二次检查
        instance = new Singleton();  // 可能重排：先赋值后构造
    }
}
```

**C++11 保证**：`static` 局部变量初始化是线程安全的，直接使用即可：
```cpp
Singleton& getInstance() {
    static Singleton instance;  // C++11 起线程安全
    return instance;
}
```
