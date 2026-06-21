# Thunder 协程调度器设计

> 调度器架构、工作窃取、IO 事件集成、定时器系统。

---

## 一、调度器架构

### 1.1 整体结构

```
--------------------------------------------
           CoroutineScheduler
--------------------------------------------
  Worker 0      Worker 1      Worker N
  | epoll |     | epoll |     | epoll |
--------------------------------------------
   共享全局协程队列 (work-stealing)
--------------------------------------------
```

- 每个 Worker 线程绑定一个 CPU 核心
- 每个 Worker 有自己的 epoll 实例和本地协程队列
- 全局队列用于负载均衡和工作窃取

### 1.2 核心数据结构

```cpp
class CoroutineScheduler {
    struct Worker {
        std::thread thread;
        int cpu_id;
        lockfree_queue local_queue;
        int epoll_fd;
        TimerWheel timer_wheel;
    };
    vector workers_;
    lockfree_queue global_queue_;
};
```

## 二、协程生命周期

### 2.1 状态机

```
Created -> Ready -> Running -> Yield/co_await -> Ready -> ... -> Completed
```

### 2.2 调度循环

```cpp
void Worker::Run() {
    while (!stopped_) {
        Coroutine* coro = local_queue.TryPop();
        if (!coro) coro = global_queue_.TryPop();
        if (!coro) {
            for (int i = 0; i < n; i++) {
                coro = workers_[victim]->TrySteal();
                if (coro) break;
            }
        }
        if (coro) coro->Resume();
        else event_loop->WaitForEvents(10);
    }
}
```

## 三、IO 事件集成

### 3.1 协程化 epoll

```cpp
struct EpollAwaiter {
    int fd_;
    Coroutine* coro_;

    void await_suspend(Coroutine::handle h) {
        coro_ = h.address();
        epoll_event ev{.events=EPOLLIN, .data={.ptr=this}};
        epoll_ctl(worker->epoll_fd, EPOLL_CTL_ADD, fd_, &ev);
    }
    void await_resume() {
        epoll_ctl(worker->epoll_fd, EPOLL_CTL_DEL, fd_, nullptr);
    }
};

Task<size_t> AsyncRead(int fd, char* buf, size_t len) {
    co_await EpollAwaiter{fd};
    return ::read(fd, buf, len);
}
```

### 3.2 io_uring 支持

完整实现和示例见 [09-Thunder-io_uring集成示例.md](09-Thunder-io_uring集成示例.md)。

```cpp
#if defined(THUNDER_USE_IOURING)
struct UringAwaiter {
    io_uring*     ring_;
    io_uring_sqe* sqe_;
    int           result_ = 0;

    bool await_ready() const noexcept { return false; }

    void await_suspend(std::coroutine_handle<> h) noexcept {
        // sqe->user_data 存 this 指针，CQE 完成后写回 result_
        io_uring_sqe_set_data(sqe_, this);
        io_uring_submit(ring_);
    }

    int await_resume() const noexcept {
        return result_;  // 返回 read/write 字节数或错误码
    }
};

// Worker 的 CQ 收割 → 恢复协程
void Worker::ProcessCompletions() {
    io_uring_cqe* cqe;
    unsigned head;
    io_uring_for_each_cqe(&io_ring_, head, cqe) {
        auto* awaiter = (UringAwaiter*)io_uring_cqe_get_data(cqe);
        awaiter->result_ = cqe->res;           // 写回结果
        io_uring_cqe_seen(&io_ring_, cqe);
        // 从 awaiter 找到 coroutine → 入就绪队列
        auto* coro = container_of(awaiter, Coroutine, awaiter);
        local_queue_.Push(coro);
    }
}
#endif
```

## 四、定时器系统

### 4.1 层级时间轮

```cpp
class TimerWheel {
    static const int SLOTS = 64;
    static const int LEVELS = 3;
    vector<Timer*> wheels_[LEVELS][SLOTS];
    int tick_ = 0;

    void AddTimer(Timer* t, uint64_t delay_ms) {
        if (delay_ms < 64)
            wheels_[0][(tick_+delay_ms)%64].push_back(t);
        else if (delay_ms < 4096)
            wheels_[1][(tick_/64+delay_ms/64)%64].push_back(t);
        else
            wheels_[2][(tick_/4096+delay_ms/4096)%64].push_back(t);
    }

    void Tick() {
        tick_ = (tick_ + 1) % 64;
        if (tick_ == 0) Cascade(1);
        for (auto* t : wheels_[0][tick_]) t->OnExpired();
        wheels_[0][tick_].clear();
    }
};
```

### 4.2 协程化延时

```cpp
Task<> WaitForTest() {
    co_await SleepFor(100ms);
    co_await SleepUntil(now() + 1s);
}

Task<Response> CallWithTimeout(Request req, Duration timeout) {
    auto result = co_await (CallRpc(req) || TimeoutAfter(timeout));
    if (result.IsTimeout()) { /* handle */ }
}
```

## 五、调度性能调优

| 参数 | 默认值 | 说明 |
|------|--------|------|
| Worker 数 | CPU 核心数 | 每核心一个 Worker 线程 |
| 本地队列大小 | 4096 | 超过则放入全局队列 |
| epoll 事件数 | 1024 | 每次 epoll_wait 最大事件数 |
| 定时器精度 | 1ms | 时间轮最小刻度 |
