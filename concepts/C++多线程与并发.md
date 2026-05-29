# C++ 多线程与并发

> 本文涵盖 C++ 多线程编程核心概念：线程管理、同步原语、原子操作、线程池、异步任务等。

See also: [[POSIX线程管理]], [[Linux线程调度]], [[C++手写代码模板]], [[packaged_task]], [[futex]]

## 一、线程管理

### 1.1 std::thread

```cpp
#include <thread>
#include <iostream>

void hello() {
    std::cout << "Hello from thread " << std::this_thread::get_id() << std::endl;
}

int main() {
    std::thread t(hello);
    t.join();  // 等待线程结束
    return 0;
}
```

### 1.2 线程生命周期

- **join()**：阻塞等待线程结束
- **detach()**：分离线程，使其独立运行
- **joinable()**：检查线程是否可 join

### 1.3 std::jthread (C++20)

```cpp
#include <thread>
#include <iostream>

int main() {
    std::jthread t([](std::stop_token st) {
        while (!st.stop_requested()) {
            std::cout << "working..." << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    });
    
    std::this_thread::sleep_for(std::chrono::seconds(1));
    t.request_stop();  // 请求停止
    // jthread 析构时自动 join
    return 0;
}
```

## 二、同步原语

### 2.1 std::mutex 与锁

```cpp
#include <mutex>
#include <thread>

std::mutex mtx;
int shared_data = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);
    ++shared_data;
}

void increment_unique() {
    std::unique_lock<std::mutex> lock(mtx);
    ++shared_data;
    // unique_lock 可手动 unlock/lock
}
```

### 2.2 死锁避免

```cpp
std::mutex m1, m2;

// 使用 std::lock 同时锁定多个 mutex
void safe_operation() {
    std::lock(m1, m2);
    std::lock_guard<std::mutex> lk1(m1, std::adopt_lock);
    std::lock_guard<std::mutex> lk2(m2, std::adopt_lock);
    // 操作共享数据
}

// C++17: std::scoped_lock
void safe_operation_cpp17() {
    std::scoped_lock lock(m1, m2);
    // 操作共享数据
}
```

### 2.3 std::shared_mutex (C++17)

```cpp
#include <shared_mutex>

std::shared_mutex rw_mutex;
int data = 0;

void reader() {
    std::shared_lock lock(rw_mutex);  // 共享锁，多个读者可同时持有
    int value = data;
}

void writer() {
    std::unique_lock lock(rw_mutex);  // 独占锁
    ++data;
}
```

### 2.4 std::condition_variable

```cpp
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;

void producer() {
    for (int i = 0; i < 10; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        queue.push(i);
        cv.notify_one();
    }
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [] { return !queue.empty(); });
        int value = queue.front();
        queue.pop();
        lock.unlock();
        // 处理 value
    }
}
```

### 2.5 C++20 新增同步原语

```cpp
#include <latch>
#include <barrier>
#include <semaphore>

// latch: 一次性同步点
std::latch done(3);
void worker_latch() {
    // 工作...
    done.count_down();  // 计数减一
}
// done.wait();  // 等待计数归零

// barrier: 可重用同步点
std::barrier sync_point(3);
void worker_barrier() {
    // 阶段1
    sync_point.arrive_and_wait();
    // 阶段2
    sync_point.arrive_and_wait();
}

// semaphore: 信号量
std::counting_semaphore<5> sem(5);  // 最大计数5
void worker_sem() {
    sem.acquire();  // 获取信号量
    // 工作...
    sem.release();  // 释放信号量
}
```

## 三、原子操作

### 3.1 std::atomic

```cpp
#include <atomic>

std::atomic<int> counter{0};
std::atomic<bool> flag{false};

void increment() {
    counter.fetch_add(1, std::memory_order_relaxed);
}

void set_flag() {
    flag.store(true, std::memory_order_release);
}

bool check_flag() {
    return flag.load(std::memory_order_acquire);
}
```

### 3.2 内存序

| 内存序 | 说明 |
|--------|------|
| `memory_order_relaxed` | 仅保证原子性，无同步/顺序约束 |
| `memory_order_acquire` | 后续读写不能重排到此操作之前 |
| `memory_order_release` | 之前读写不能重排到此操作之后 |
| `memory_order_acq_rel` | 同时具有 acquire 和 release 语义 |
| `memory_order_seq_cst` | 顺序一致性（默认，最强保证） |

## 四、异步任务

### 4.1 std::async

```cpp
#include <future>

int compute() {
    return 42;
}

int main() {
    // 异步执行
    std::future<int> result = std::async(std::launch::async, compute);
    
    // 做其他事情...
    
    int value = result.get();  // 获取结果，阻塞直到完成
    return 0;
}
```

### 4.2 std::packaged_task

`std::packaged_task` 将可调用对象包装为异步任务，与 `std::future` 关联，可精确控制任务在哪个线程执行。详见 [[packaged_task]]。

```cpp
#include <future>
#include <thread>

int main() {
    std::packaged_task<int(int)> task([](int x) { return x * x; });
    std::future<int> result = task.get_future();
    
    std::thread t(std::move(task), 5);
    std::cout << result.get() << std::endl;  // 25
    t.join();
}
```

[src: raw/ingested/2技术/cpp/C++多线程完整手册-5.3-packaged_task.md]

### 4.3 std::promise

```cpp
#include <future>

void set_value(std::promise<int> p) {
    p.set_value(42);
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();
    
    std::thread t(set_value, std::move(p));
    int value = f.get();  // 42
    t.join();
}
```

## 五、线程池

### 5.1 线程池基本组成

- **任务队列**：生产者投递任务，工作线程消费。
- **工作线程集合**：固定或可伸缩的线程，循环取任务→执行。
- **同步机制**：互斥锁 + 条件变量（或无锁队列）。

[src: raw/ingested/2技术/cpp/C++多线程完整手册-8.2-线程池基本组成.md]

### 5.2 简单线程池实现

```cpp
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <future>

class ThreadPool {
public:
    ThreadPool(size_t threads) : stop(false) {
        for (size_t i = 0; i < threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex);
                        condition.wait(lock, [this] {
                            return stop || !tasks.empty();
                        });
                        if (stop && tasks.empty()) return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
            });
        }
    }
    
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::result_of<F(Args...)>::type> {
        using return_type = typename std::result_of<F(Args...)>::type;
        
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        std::future<return_type> res = task->get_future();
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            if (stop) throw std::runtime_error("enqueue on stopped ThreadPool");
            tasks.emplace([task]() { (*task)(); });
        }
        condition.notify_one();
        return res;
    }
    
    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread& worker : workers) {
            worker.join();
        }
    }
    
private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};
```

### 5.3 线程池示例（使用 packaged_task）

```cpp
#include <future>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <vector>
#include <functional>
#include <iostream>

class ThreadPool {
public:
    explicit ThreadPool(size_t n) : stop_(false) {
        for (size_t i = 0; i < n; ++i)
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mutex_);
                        cv_.wait(lock, [this]{ return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task();
                }
            });
    }
    ~ThreadPool() { shutdown(); }

    template<typename F>
    auto submit(F&& f) -> std::future<decltype(f())> {
        using R = decltype(f());
        auto task = std::make_shared<std::packaged_task<R()>>(std::forward<F>(f));
        auto fut = task->get_future();
        {
            std::lock_guard<std::mutex> lock(mutex_);
            if (stop_) throw std::runtime_error("pool stopped");
            tasks_.emplace([task]{ (*task)(); });
        }
        cv_.notify_one();
        return fut;
    }

    void shutdown() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stop_ = true;
        }
        cv_.notify_all();
        for (auto& w : workers_) if (w.joinable()) w.join();
    }
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mutex_;
    std::condition_variable cv_;
    bool stop_;
};

int main() {
    ThreadPool pool(4);
    // 提交有返回值任务，通过 future 获取结果
    auto f1 = pool.submit([] { return 42; });
    auto f2 = pool.submit([]{ return 10 + 20; });
    std::cout << f1.get() << std::endl;  // 42
    std::cout << f2.get() << std::endl;  // 30
    // 提交无返回值任务
    pool.submit([]{ std::cout << "task done\n"; });
    // 析构时 shutdown 会等待所有任务完成
}
```

[src: raw/ingested/2技术/cpp/C++多线程完整手册-8.2-线程池基本组成.md]

### 5.4 按线程编号隔离的线程池（SPSC 无锁队列 + 路由线程）

很多场景希望**同一业务 key 永远在同一工作线程上执行**，实现"按线程编号（或分片编号）隔离"的线程池，例如：

- **按用户ID / 会话ID 分片**：同一用户的请求在同一线程上串行执行，避免额外锁。
- **按分区/分片分工**：每个线程只负责自己负责的资源分区，提高缓存命中率。

这里给出一个更"分片化"的实现：**每个分片一个 SPSC 无锁队列（单生产者单消费者环形队列）**，且**假设只有一个生产者线程**：

- 整个线程池只有一个生产者线程负责调用 `submit`；
- 每个工作线程消费一个专属的 SPSC 队列；
- **每个工作线程分配一个从 1 开始的整数 id**，作为"硬索引"直接对应队列数组下标 `id-1`，不再依赖业务 key；
- 这样可以完全满足 SPSC 场景（单生产者 + 单消费者），实现纯无锁的分片队列。

```cpp
#include <atomic>
#include <functional>
#include <future>
#include <iostream>
#include <string>
#include <thread>
#include <vector>

// 按"分片编号/线程编号"隔离的线程池：每个线程一个 SPSC 无锁队列
template<size_t ShardCount, size_t ShardQueueCapacity = 1024>
class ShardedSPSCThreadPool {
public:
    using Task = std::function<void()>;

    ShardedSPSCThreadPool()
        : stop_(false) {
        // 启动工作线程，线程 id 从 1 开始，作为分片索引（id-1）
        for (size_t i = 0; i < ShardCount; ++i) {
            size_t threadId = i + 1;
            workers_.emplace_back([this, i, threadId] {
                workerLoop(i, threadId);
            });
        }
    }

    ~ShardedSPSCThreadPool() {
        shutdown();
    }

    // 单生产者：按"线程 id"提交任务，直接写入对应分片的 SPSC 队列
    template<typename F>
    auto submit(std::size_t threadId, F&& f) -> std::future<decltype(f())> {
        using R = decltype(f());
        auto task = std::make_shared<std::packaged_task<R()>>(std::forward<F>(f));
        auto fut = task->get_future();

        if (threadId == 0 || threadId > ShardCount) {
            throw std::out_of_range("invalid threadId");
        }
        size_t idx = threadId - 1;

        // 单生产者直接写入对应分片的 SPSC 队列
        Task wrapper = [task] { (*task)(); };
        // 简单自旋直到成功（实践中可加入退避/丢弃策略）
        while (!stop_.load(std::memory_order_acquire)) {
            if (shardQueues_[idx].push(wrapper)) {
                break;
            }
            std::this_thread::yield();
        }
        return fut;
    }

    void shutdown() {
        bool expected = false;
        if (!stop_.compare_exchange_strong(expected, true)) {
            return; // 已经 shutdown
        }

        // 等待所有工作线程自然退出（轮询 stop_ + 队列空）
        for (auto& w : workers_) {
            if (w.joinable()) w.join();
        }
    }

private:
    // 每个分片只有一个消费线程，对应一个 SPSC 队列
    void workerLoop(size_t shardIdx, size_t threadId) {
        for (;;) {
            if (stop_.load(std::memory_order_acquire) &&
                shardQueues_[shardIdx].empty()) {
                break;
            }
            Task task;
            if (shardQueues_[shardIdx].pop(task)) {
                // 这里可以使用 threadId 做业务上的分片标识
                task();
            } else {
                std::this_thread::yield();
            }
        }
    }

    std::atomic<bool> stop_;

    // 分片 SPSC 队列：每个只允许 1 生产者（调用 submit 的线程）+ 1 消费者（worker）
    SPSCQueue<Task, ShardQueueCapacity> shardQueues_[ShardCount];

    std::vector<std::thread> workers_;
};

// 示例：调用方显式指定线程 id（1..ShardCount），每个线程有固定 id（1..ShardCount）
int main() {
    constexpr size_t Shards = 4;
    ShardedSPSCThreadPool<Shards> pool;

    auto submitToThread = [&](std::size_t threadId, int value) {
        return pool.submit(threadId, [threadId, value] {
            std::cout << "threadId=" << threadId
                      << " value=" << value
                      << " run in thread " << std::this_thread::get_id()
                      << std::endl;
            return value * 2;
        });
    };

    // 显式将任务投递到指定线程（1..Shards），相同 threadId 的任务在同一工作线程上串行执行
    auto f1 = submitToThread(1, 1);
    auto f2 = submitToThread(2, 2);
    auto f3 = submitToThread(1, 3); // 与 threadId=1 相关的任务会落在同一分片线程上
    auto f4 = submitToThread(3, 4);

    // get() 会阻塞直到任务完成；若任务抛异常，get() 会在当前线程重新抛出
    try {
        std::cout << "result1=" << f1.get() << std::endl;
        std::cout << "result2=" << f2.get() << std::endl;
        std::cout << "result3=" << f3.get() << std::endl;
        std::cout << "result4=" << f4.get() << std::endl;
    } catch (const std::exception& e) {
        std::cerr << "task exception: " << e.what() << std::endl;
    }

    pool.shutdown();
    return 0;
}
```

[src: raw/ingested/2技术/cpp/C++多线程完整手册-8.6-根据线程编号隔离的线程池完整示例（SPSC-无锁队列-+-路由线程）.md]

### 5.5 无锁、无条件变量版本（Fire-and-Forget，纯 SPSC）

若要**完全避免锁和条件变量**，可放弃 `future`/`promise`（内部含 mutex + condition_variable），改为 **Fire-and-Forget**：`submit` 仅投递任务，不返回 future，调用方拿不到返回值。

- **任务队列**：每线程一个 SPSC 无锁队列（同 5.4）
- **同步**：仅用 `std::atomic<bool> stop_` 控制退出，worker 自旋 `pop`，无 mutex、无 condition_variable
- **结果传递**：无，任务为 `void()` 或返回值被忽略，适合日志、副作用、回调等

```cpp
#include <atomic>
#include <cstddef>
#include <functional>
#include <iostream>
#include <thread>
#include <vector>

// SPSC 无锁环形队列（同 5.4）
template<typename T, size_t Capacity>
class SPSCQueue {
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be power of two");
public:
    SPSCQueue() : head_(0), tail_(0) {}
    bool push(const T& value) {
        size_t head = head_.load(std::memory_order_relaxed);
        size_t next = (head + 1) & (Capacity - 1);
        if (next == tail_.load(std::memory_order_acquire)) return false;
        buffer_[head] = value;
        head_.store(next, std::memory_order_release);
        return true;
    }
    bool pop(T& result) {
        size_t tail = tail_.load(std::memory_order_relaxed);
        if (tail == head_.load(std::memory_order_acquire)) return false;
        result = std::move(buffer_[tail]);
        tail_.store((tail + 1) & (Capacity - 1), std::memory_order_release);
        return true;
    }
    bool empty() const {
        return head_.load(std::memory_order_acquire) == tail_.load(std::memory_order_acquire);
    }
private:
    T buffer_[Capacity];
    std::atomic<size_t> head_;
    std::atomic<size_t> tail_;
};

// 无锁、无条件变量：仅 atomic + SPSC，submit 无返回值
template<size_t ShardCount, size_t ShardQueueCapacity = 1024>
class ShardedSPSCLockFreePool {
public:
    using Task = std::function<void()>;

    ShardedSPSCLockFreePool() : stop_(false) {
        for (size_t i = 0; i < ShardCount; ++i) {
            size_t threadId = i + 1;
            workers_.emplace_back([this, i, threadId] { workerLoop(i, threadId); });
        }
    }
    ~ShardedSPSCLockFreePool() { shutdown(); }

    // Fire-and-forget：无 future，无锁，直接 push 到 SPSC
    template<typename F>
    void submit(std::size_t threadId, F&& f) {
        if (stop_.load(std::memory_order_acquire)) return;
        if (threadId == 0 || threadId > ShardCount) return;
        size_t idx = threadId - 1;
        Task task = std::forward<F>(f);
        while (!stop_.load(std::memory_order_acquire)) {
            if (shardQueues_[idx].push(task)) break;
            std::this_thread::yield();
        }
    }

    void shutdown() {
        bool expected = false;
        if (!stop_.compare_exchange_strong(expected, true)) return;
        for (auto& w : workers_) if (w.joinable()) w.join();
    }

private:
    void workerLoop(size_t shardIdx, size_t threadId) {
        for (;;) {
            if (stop_.load(std::memory_order_acquire) && shardQueues_[shardIdx].empty()) break;
            Task task;
            if (shardQueues_[shardIdx].pop(task)) {
                task();  // 执行，无 future，异常需任务内部处理
            } else {
                std::this_thread::yield();
            }
        }
    }

    std::atomic<bool> stop_;
    SPSCQueue<Task, ShardQueueCapacity> shardQueues_[ShardCount];
    std::vector<std::thread> workers_;
};

int main() {
    ShardedSPSCLockFreePool<4> pool;
    for (int i = 1; i <= 4; ++i) {
        std::string msg = "task-" + std::to_string(i);
        pool.submit(static_cast<size_t>(i), [i, msg] {
            std::cout << "threadId=" << i << " " << msg << " running\n";
        });
    }
    pool.shutdown();
}
```

**与 5.4 对比**：5.4 用 `packaged_task` 返回 future，内部有锁+条件变量；本版完全无锁+无条件变量，但 `submit` 不返回 future，适用 Fire-and-Forget 场景。

[src: raw/ingested/2技术/cpp/C++多线程完整手册-8.7-无锁、无条件变量版本（Fire-and-Forget，纯-SPSC）.md]

## 六、OpenMP vs 线程池

> 对比 OpenMP 与线程池在不同场景下的适用性。详见 [[OpenMP-vs-线程池]]。

| 场景 | OpenMP | 线程池 |
| ---- | ------ | ------ |
| CPU 密集型（矩阵乘等） | ⭐⭐⭐⭐⭐ 最快 | ⭐⭐⭐ 较慢 |
| I/O 密集型 | ⭐⭐ 不合适 | ⭐⭐⭐⭐ 较好 |
| 不规则并行（树/图遍历） | ⭐⭐ 难并行 | ⭐⭐⭐⭐⭐ 灵活 |

- **选 OpenMP**：循环/数据并行、CPU 密集、追求性能。
- **选线程池**：I/O、复杂任务图、不规则并行。

[src: raw/ingested/2技术/cpp/C++多线程完整手册-9.8-OpenMP-vs-线程池.md]

## 七、futex 与底层锁

Linux 底层高效锁原语 futex 是 `std::mutex`、`pthread_mutex_t`、`sem_t`、`std::condition_variable` 等同步原语的底层实现核心。详见 [[futex]]。

## 八、面试高频问题

### Q1: mutex 和 atomic 的区别？
- **mutex**：操作系统级同步，阻塞等待，适合临界区较长的场景
- **atomic**：CPU 指令级原子操作，无锁，适合单个变量的原子更新

### Q2: lock_guard 和 unique_lock 的区别？
- **lock_guard**：RAII 风格，构造时加锁，析构时解锁，不可手动控制
- **unique_lock**：更灵活，可手动 lock/unlock，可配合 condition_variable

### Q3: 如何避免死锁？
- 按固定顺序加锁
- 使用 `std::lock` 或 `std::scoped_lock` 同时锁定多个 mutex
- 使用超时锁 `try_lock_for`
- 避免嵌套锁

### Q4: condition_variable 为什么需要 while 循环？
- 防止**虚假唤醒**（spurious wakeup）
- 确保条件真正满足后再继续执行

### Q5: std::async 的 launch 策略？
- `std::launch::async`：强制异步执行
- `std::launch::deferred`：延迟执行，调用 get/wait 时才执行
- 默认（两者组合）：由实现决定

[src: raw/ingested/2技术/cpp/C++多线程完整手册-5.3-packaged_task.md]

## Related Pages
- [[POSIX线程管理]]
- [[Linux线程调度]]
- [[C++手写代码模板]]
- [[MPMC环形无锁队列-Vyukov]]
- [[现代C++特性按版本划分]]
- [[packaged_task]]
- [[futex]]
- [[OpenMP-vs-线程池]]
- [[C++17并行算法]]
