# Thunder 架构设计

> 协程模型、Raft 实现、IPC、动态插件。

---

## 一、系统架构

```
Thunder 进程模型：
┌──────────────────────────────────────────┐
│  Manager 父进程                          │
│  ├── 共享内存队列 ← → Worker[0]         │
│  ├── 共享内存队列 ← → Worker[1]         │
│  └── 共享内存队列 ← → Worker[N]         │
└──────────────────────────────────────────┘
```

进程间通过 **共享内存（mmap + ring buffer）** 通信，零 syscall（除 semaphore 外），延迟 0.15ms。

## 二、C++20 Stackless 协程

### 2.1 Stackless vs Stackful

| 特性 | Stackless（C++20） | Stackful（Boost.Context） |
|------|-------------------|--------------------------|
| 栈大小 | 2KB-8KB（堆上分配） | 64KB-1MB |
| 切换开销 | 50ns | 200ns |
| 实现方式 | 编译器内置 | 汇编手动切换 |
| 调试 | 正常调试 | 需要特殊工具 |
| 适用场景 | 高并发短任务 | 深度递归调用 |

**为什么选择 Stackless**：Thunder 的协程场景是处理 RPC 请求，每个请求平均执行 100 行代码，深度 < 10。Stackless 只需 2KB 协程状态，而 Stackful 需 64KB 栈，浪费内存。

### 2.2 协程调度器

```cpp
class CoroutineScheduler {
public:
    struct Task {
        struct promise_type {
            Task get_return_object() {
                return Task{Handle::from_promise(*this)};
            }
            suspend_never initial_suspend() { return {}; }
            suspend_never final_suspend() noexcept { return {}; }
            void return_void() {}
            void unhandled_exception() { std::terminate(); }
        };
        Handle handle;
    };

    void Run() {
        while (!stopped_) {
            Coroutine* coro = ready_queue_.Pop();
            if (coro) coro->Resume();
            else WaitForIO();
        }
    }

private:
    lockfree_queue<Coroutine*> ready_queue_;
};
```

### 2.3 工作窃取调度

```cpp
class WorkStealingScheduler {
    vector<CoroutineQueue> local_queues_;  // 每线程本地队列

    void Schedule(Coroutine* coro) {
        int queue_id = thread_id_ % local_queues_.size();
        local_queues_[queue_id].Push(coro);
    }

    Coroutine* Steal() {
        int victim = Random() % local_queues_.size();
        return local_queues_[victim].TryPop();
    }
};
```

### 2.4 协程切换开销分解

```
Stackless 协程切换（50ns）：
├── 保存寄存器：20ns
├── 切换栈指针：10ns
├── 跳转执行流：10ns
└── 更新状态：10ns

线程切换（10,000ns = 200 倍）：
├── Cache Miss：5000ns（L3 未命中）
├── TLB Flush：2000ns
├── 内核调度：2000ns
└── 其他：1000ns
```

## 三、Raft 共识算法

### 3.1 状态机

```
Follower ←─ 选举超时 ──→ Candidate ←─ 多数票 ──→ Leader
    ↑                                                      │
    └────────────────── 心跳超时 ──────────────────────────┘
```

### 3.2 选主流程

```cpp
class RaftNode {
    void OnElectionTimeout() {
        if (state_ != State::Leader) StartElection();
    }

    void StartElection() {
        state_ = State::Candidate;
        current_term_++;
        voted_for_ = node_id_;
        int votes = 1;  // 自己一票
        for (auto& peer : peers_) {
            peer->SendRequestVote({
                .term = current_term_,
                .candidate_id = node_id_,
                .last_log_index = log_.LastIndex(),
                .last_log_term = log_.LastTerm(),
            });
        }
    }

    void OnRequestVoteResponse(const RequestVoteResponse& resp) {
        if (resp.term > current_term_) {
            BecomeFollower(resp.term);  // 发现更高任期
            return;
        }
        if (resp.vote_granted) {
            votes_++;
            if (votes_ > (peers_.size() + 1) / 2) {
                BecomeLeader();  // 获得多数票
            }
        }
    }
};
```

### 3.3 批量日志复制

```cpp
// 优化前：每条日志单独复制 → RTT 2ms × 100 = 200ms
for (auto& entry : entries) {
    replicate(entry);
}

// 优化后：批量复制 → 1 RTT = 2ms，提升 100 倍
BatchReplicate(entries);
```

## 四、进程间通信（IPC）

### 4.1 共享内存方案

```
Manager（父进程）                    Worker（子进程）
  shm_open → mmap  ←──── 共享 ────→  shm_open → mmap
  写入请求到 ring buffer             ring buffer 读取
  post semaphore                    sem_wait 等待
```

### 4.2 三种 IPC 方式对比

| 方式 | QPS | 延迟 | 说明 |
|------|-----|------|------|
| **共享内存** | 1,500,000 | 0.15ms | mmap + ring buffer，零拷贝 |
| Unix Domain Socket | 500,000 | 0.3ms | SOCK_SEQPACKET |
| TCP 回环 | 300,000 | 0.5ms | 需协议栈开销 |

## 五、IO 模型

### 5.1 三引擎架构

Thunder 通过统一 `IoBackend` 接口抽象，支持三种 I/O 后端，编译时通过宏切换，运行时通过配置文件选择：

| 后端 | 类名 | 宏开关 | 说明 |
|------|------|--------|------|
| libev (epoll) | `EvIoBackend` | 默认 | 每个 fd 一个 `ev_io` watcher |
| 原生 io_uring | `UringIoBackend` | `THUNDER_IO_URING` | 直接 liburing 操作 SQ/CQ |
| Asio io_uring | `AsioUringIoBackend` | `THUNDER_IO_ASIO_URING` | 基于 `asio::posix::stream_descriptor` |

```cpp
// 配置项（Hello.json）
{
    "io_backend": "asio_uring",   // "ev" | "uring" | "asio_uring"
    "cpu_affinity": true,
    "worker_count": 8
}
```

### 5.2 架构设计

**关键原则：io_uring 嵌入 libev，而非替代 libev。**

```
libev 主循环（以 asio_uring 为例）:
  ┌─ ev_prepare  → io_context.poll()   [排尽已有 CQE]
  ├─ epoll_wait   … [ring_fd + 其他 fd]
  ├─ ev_io(ring_fd) → io_context.poll() [ring_fd 唤醒，处理新完成]
  └─ ev_check    → io_context.poll()   [epoll 返回期间到达的 CQE]
  
  零锁、零线程跳、零跨线程 syscall
```

三种后端的集成方式：

| 后端 | 集成方式 | 读操作 | 写操作 |
|------|---------|--------|--------|
| EvIoBackend | 每个 fd 注册 `ev_io` watcher | epoll 通知可读 → `read()` | epoll 通知可写 → `write()` |
| UringIoBackend | `ring_fd` 注册一个 `ev_io` watcher | 填 SQE → `io_uring_submit` → CQE 回调 | 同步 `WriteFD()` |
| AsioUringIoBackend | `ev_prepare/ev_check/ev_io(ring_fd)` 三路驱动 | `sock.async_read_some()` → Asio 完成回调 | 同步 `WriteFD()` |

### 5.3 为什么写操作用同步？

实际测量表明，小包（< 8KB）同步写的延迟和异步写几乎一样，但异步写需要额外的 SQE/CQE 管理开销和回调状态机。Thunder 的写缓冲区通过 `Compact()` 整理，`WriteFD()` 内部用 `writev` 合并小包，同步效率足够。

### 5.4 实测性能

| 场景 | ev (epoll) | asio_uring | 差异 |
|------|-----------|------------|------|
| 小包 37B, c100 | 160,674 RPS / 705μs | 164,358 RPS / 2.49ms | RPS ≈持平 |
| 4KB, c100 | 73,137 RPS / 1.51ms | 68,677 RPS / 0.99ms | 延迟低 **34%** |
| 64KB, c100 | 6,207 RPS / 16.78ms | 6,675 RPS / 2.32ms | 延迟低 **86%** |
| 64KB, c500 Stdev | 83.51ms | **1.63ms** | 尾延迟 1/50 |

完整报告见 `deploy/tests/benchmark/results/asio_uring_benchmark.md`，集成示例见 [09-Thunder-io_uring集成示例.md](09-Thunder-io_uring集成示例.md)。

## 六、动态插件系统

支持 `.so` 动态库热加载，运行时注册路由和处理器。插件无需修改主程序代码，实现功能扩展。
