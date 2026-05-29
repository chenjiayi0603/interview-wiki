# libgo Hook 机制

## 概述

libgo 的 Hook（钩子）机制是其核心特性之一，它能够让原本会阻塞整个线程的同步系统调用，变成只阻塞当前协程而不阻塞线程的"协程友好"操作。

**核心思想：**
```
用户代码调用 read()
    │
    ▼
libgo Hook 层拦截（替换原始 read 函数）
    │
    ├── 1. 检测当前是否在协程环境中
    ├── 2. 将 fd 设为非阻塞模式
    ├── 3. 尝试执行真实 read
    ├── 4. 若返回 EAGAIN/EWOULDBLOCK：
    │      - 注册 fd 到 epoll/kqueue 监听可读事件
    │      - 挂起当前协程（yield），切换到其他协程
    │      - 事件就绪后唤醒协程继续执行
    └── 5. 返回结果给用户代码
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## Hook 实现技术

libgo 在 Linux 上主要通过 **动态链接器符号覆盖** 和 **LD_PRELOAD** 机制实现 Hook。

### 动态链接器符号覆盖

```cpp
// 原理：利用动态链接器的符号解析顺序
// 当程序调用 read() 时，动态链接器按以下顺序查找符号：
// 1. 可执行文件本身
// 2. LD_PRELOAD 指定的库
// 3. 依赖库（按链接顺序）
// 4. 标准库 libc.so

// libgo 定义同名函数覆盖标准库函数
extern "C" {
    typedef ssize_t (*read_func_t)(int fd, void* buf, size_t count);
    static read_func_t g_sys_read = nullptr;
    
    ssize_t read(int fd, void* buf, size_t count) {
        return libgo_read(fd, buf, count);
    }
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 获取原始函数指针

```cpp
#include <dlfcn.h>

class HookInitializer {
public:
    HookInitializer() {
        g_sys_read = (read_func_t)dlsym(RTLD_NEXT, "read");
        g_sys_write = (write_func_t)dlsym(RTLD_NEXT, "write");
        g_sys_connect = (connect_func_t)dlsym(RTLD_NEXT, "connect");
        g_sys_accept = (accept_func_t)dlsym(RTLD_NEXT, "accept");
        g_sys_recv = (recv_func_t)dlsym(RTLD_NEXT, "recv");
        g_sys_send = (send_func_t)dlsym(RTLD_NEXT, "send");
        g_sys_poll = (poll_func_t)dlsym(RTLD_NEXT, "poll");
        g_sys_select = (select_func_t)dlsym(RTLD_NEXT, "select");
        g_sys_sleep = (sleep_func_t)dlsym(RTLD_NEXT, "sleep");
        g_sys_usleep = (usleep_func_t)dlsym(RTLD_NEXT, "usleep");
        g_sys_nanosleep = (nanosleep_func_t)dlsym(RTLD_NEXT, "nanosleep");
    }
};

static HookInitializer g_hook_initializer;
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## 核心 Hook 函数实现

### read Hook 实现

```cpp
ssize_t libgo_read(int fd, void* buf, size_t count) {
    if (!is_in_coroutine()) {
        return g_sys_read(fd, buf, count);
    }
    
    FdContext* fd_ctx = get_fd_context(fd);
    if (!fd_ctx || !fd_ctx->is_socket()) {
        return g_sys_read(fd, buf, count);
    }
    
    if (fd_ctx->user_nonblock()) {
        return g_sys_read(fd, buf, count);
    }
    
    if (!fd_ctx->sys_nonblock()) {
        int flags = fcntl(fd, F_GETFL);
        fcntl(fd, F_SETFL, flags | O_NONBLOCK);
        fd_ctx->set_sys_nonblock(true);
    }
    
    while (true) {
        ssize_t n = g_sys_read(fd, buf, count);
        if (n >= 0) return n;
        if (errno == EINTR) continue;
        if (errno != EAGAIN && errno != EWOULDBLOCK) return -1;
        
        if (!wait_fd_event(fd, EPOLLIN, fd_ctx->read_timeout())) {
            errno = ETIMEDOUT;
            return -1;
        }
    }
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### connect Hook 实现

```cpp
int libgo_connect(int fd, const struct sockaddr* addr, socklen_t addrlen) {
    if (!is_in_coroutine()) {
        return g_sys_connect(fd, addr, addrlen);
    }
    
    FdContext* fd_ctx = get_fd_context(fd);
    if (!fd_ctx || fd_ctx->user_nonblock()) {
        return g_sys_connect(fd, addr, addrlen);
    }
    
    if (!fd_ctx->sys_nonblock()) {
        int flags = fcntl(fd, F_GETFL);
        fcntl(fd, F_SETFL, flags | O_NONBLOCK);
        fd_ctx->set_sys_nonblock(true);
    }
    
    int ret = g_sys_connect(fd, addr, addrlen);
    if (ret == 0) return 0;
    if (errno != EINPROGRESS) return -1;
    
    if (!wait_fd_event(fd, EPOLLOUT, fd_ctx->connect_timeout())) {
        errno = ETIMEDOUT;
        return -1;
    }
    
    int error = 0;
    socklen_t len = sizeof(error);
    getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len);
    if (error) {
        errno = error;
        return -1;
    }
    return 0;
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### accept Hook 实现

```cpp
int libgo_accept(int fd, struct sockaddr* addr, socklen_t* addrlen) {
    if (!is_in_coroutine()) {
        return g_sys_accept(fd, addr, addrlen);
    }
    
    FdContext* fd_ctx = get_fd_context(fd);
    if (!fd_ctx || fd_ctx->user_nonblock()) {
        return g_sys_accept(fd, addr, addrlen);
    }
    
    if (!fd_ctx->sys_nonblock()) {
        int flags = fcntl(fd, F_GETFL);
        fcntl(fd, F_SETFL, flags | O_NONBLOCK);
        fd_ctx->set_sys_nonblock(true);
    }
    
    while (true) {
        int client_fd = g_sys_accept(fd, addr, addrlen);
        if (client_fd >= 0) {
            create_fd_context(client_fd);
            return client_fd;
        }
        if (errno == EINTR) continue;
        if (errno != EAGAIN && errno != EWOULDBLOCK) return -1;
        if (!wait_fd_event(fd, EPOLLIN, -1)) return -1;
    }
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### sleep 系列 Hook 实现

```cpp
unsigned int libgo_sleep(unsigned int seconds) {
    if (!is_in_coroutine()) {
        return g_sys_sleep(seconds);
    }
    Coroutine* co = get_current_coroutine();
    add_timer(seconds * 1000, [co]() {
        scheduler->resume(co);
    });
    co->yield();
    return 0;
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### poll/select Hook 实现

```cpp
int libgo_poll(struct pollfd* fds, nfds_t nfds, int timeout) {
    if (!is_in_coroutine()) {
        return g_sys_poll(fds, nfds, timeout);
    }
    // ... 复杂实现，涉及 epoll 事件注册和协程挂起
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## FdContext - 文件描述符上下文管理

每个被 Hook 的 fd 都有一个上下文结构，用于跟踪 fd 类型、非阻塞状态、超时设置等。

```cpp
class FdContext {
public:
    enum FdType { FD_TYPE_UNKNOWN, FD_TYPE_SOCKET, FD_TYPE_PIPE, FD_TYPE_FILE };
private:
    int fd_;
    FdType type_;
    bool user_nonblock_;
    bool sys_nonblock_;
    int connect_timeout_;
    int read_timeout_;
    int write_timeout_;
public:
    FdContext(int fd);
    bool is_socket() const;
    bool user_nonblock() const;
    bool sys_nonblock() const;
    void set_sys_nonblock(bool v);
    int read_timeout() const;
    int write_timeout() const;
    int connect_timeout() const;
    void set_timeout(int type, int ms);
};
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## socket/close Hook - 跟踪 fd 生命周期

```cpp
int libgo_socket(int domain, int type, int protocol) {
    int fd = g_sys_socket(domain, type, protocol);
    if (fd >= 0) g_fd_manager.create(fd);
    return fd;
}

int libgo_close(int fd) {
    g_fd_manager.remove(fd);
    return g_sys_close(fd);
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## DNS 相关 Hook

```cpp
struct hostent* libgo_gethostbyname(const char* name) {
    if (!is_in_coroutine()) return g_sys_gethostbyname(name);
    // 使用异步 DNS 库（如 c-ares）实现
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## Windows 平台 Hook 实现

Windows 上使用 Detours 或 MinHook 库实现 Hook。

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## Hook 机制的优缺点

| 优点 | 说明 |
|------|------|
| **零侵入** | 不需要修改现有代码，同步代码自动变异步 |
| **兼容性好** | 第三方同步库（如 mysqlclient、hiredis）可直接使用 |
| **开发效率高** | 用同步思维写代码，获得异步性能 |
| **调试友好** | 调用栈清晰，不像回调地狱 |

| 缺点 | 说明 |
|------|------|
| **平台依赖** | 不同 OS 需要不同实现 |
| **潜在兼容问题** | 某些库可能与 Hook 冲突 |
| **调试复杂** | Hook 层可能影响 gdb 调试 |
| **性能开销** | 每次系统调用多一层间接调用 |

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## Hook 开关控制

```cpp
void disable_hook();
void disable_hook_in_coroutine();
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## Hook 与 Go 运行时的对比

| 特性 | libgo Hook | Go Runtime |
|------|-----------|------------|
| 实现层 | 用户态 Hook | 运行时内置 |
| 系统调用处理 | 动态拦截替换 | 编译器+运行时协作 |
| netpoller | 手动实现 epoll 封装 | runtime/netpoll 内置 |
| 线程管理 | 调度器管理线程池 | M:P:G 自动管理 |
| 阻塞检测 | Hook 点检测 | sysmon 监控 |
| 抢占 | 无（协作式） | 有（信号抢占） |

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## libgo 钩子函数调用的底层 epoll

### epoll 基础回顾

```cpp
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### libgo 的 epoll 封装层

```cpp
class Reactor {
private:
    int epoll_fd_;
    std::vector<epoll_event> ready_events_;
    struct FdWaitInfo {
        Coroutine* read_co;
        Coroutine* write_co;
        uint32_t registered_events;
    };
    std::unordered_map<int, FdWaitInfo> fd_wait_map_;
    std::mutex mutex_;
    int wakeup_fd_[2];
public:
    Reactor();
    void add_event(int fd, uint32_t events, Coroutine* co);
    void del_event(int fd, uint32_t events);
    void poll(int timeout_ms);
    void wakeup();
};
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 事件注册实现

```cpp
void Reactor::add_event(int fd, uint32_t events, Coroutine* co) {
    std::lock_guard<std::mutex> lock(mutex_);
    auto& info = fd_wait_map_[fd];
    if (events & EPOLLIN) info.read_co = co;
    if (events & EPOLLOUT) info.write_co = co;
    uint32_t new_events = info.registered_events | events;
    struct epoll_event ev;
    ev.events = new_events | EPOLLET;
    ev.data.fd = fd;
    int op = (info.registered_events == 0) ? EPOLL_CTL_ADD : EPOLL_CTL_MOD;
    epoll_ctl(epoll_fd_, op, fd, &ev);
    info.registered_events = new_events;
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 核心事件循环 (Event Loop)

```cpp
void Reactor::poll(int timeout_ms) {
    int n = epoll_wait(epoll_fd_, ready_events_.data(), ready_events_.size(), timeout_ms);
    if (n < 0) {
        if (errno == EINTR) return;
        perror("epoll_wait");
        return;
    }
    std::vector<Coroutine*> to_resume;
    {
        std::lock_guard<std::mutex> lock(mutex_);
        for (int i = 0; i < n; i++) {
            int fd = ready_events_[i].data.fd;
            uint32_t events = ready_events_[i].events;
            if (fd == wakeup_fd_[0]) {
                char buf[64];
                while (read(wakeup_fd_[0], buf, sizeof(buf)) > 0);
                continue;
            }
            auto it = fd_wait_map_.find(fd);
            if (it == fd_wait_map_.end()) continue;
            auto& info = it->second;
            if ((events & EPOLLIN) && info.read_co) {
                to_resume.push_back(info.read_co);
                info.read_co = nullptr;
            }
            if ((events & EPOLLOUT) && info.write_co) {
                to_resume.push_back(info.write_co);
                info.write_co = nullptr;
            }
            if (events & (EPOLLERR | EPOLLHUP)) {
                if (info.read_co) { to_resume.push_back(info.read_co); info.read_co = nullptr; }
                if (info.write_co) { to_resume.push_back(info.write_co); info.write_co = nullptr; }
            }
        }
    }
    for (Coroutine* co : to_resume) {
        scheduler->resume(co);
    }
}
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 调度器主循环

```cpp
class Scheduler {
    Reactor reactor_;
    std::deque<Coroutine*> ready_queue_;
    std::mutex queue_mutex_;
public:
    void run_loop() {
        while (!stopped_) {
            Coroutine* co = nullptr;
            {
                std::lock_guard<std::mutex> lock(queue_mutex_);
                if (!ready_queue_.empty()) {
                    co = ready_queue_.front();
                    ready_queue_.pop_front();
                }
            }
            if (co) {
                current_co_ = co;
                co->resume();
                current_co_ = nullptr;
                if (co->is_finished()) delete co;
            } else {
                int timeout = get_next_timer_timeout();
                reactor_.poll(timeout);
                process_expired_timers();
            }
        }
    }
    void resume(Coroutine* co) {
        {
            std::lock_guard<std::mutex> lock(queue_mutex_);
            ready_queue_.push_back(co);
        }
        reactor_.wakeup();
    }
};
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### Hook 函数与 epoll 的交互流程

```
用户代码调用 read(fd, buf, n)
    │
    ▼
libgo_read Hook
    │
    ├── 1. 设置 fd 非阻塞 (fcntl O_NONBLOCK)
    ├── 2. 调用真实 read() 尝试读取
    │      ├── 成功 ──► 返回数据
    │      └── EAGAIN
    │           ├── 3. reactor.add_event(fd, EPOLLIN, co)
    │           ├── 4. co->yield()
    │           └── 5. 调度器继续运行其他协程或 epoll_wait
    │
    │  (fd 变为可读)
    ▼
6. epoll_wait 返回，检测到 fd 可读
7. scheduler->resume(co)
8. 调度器从队列取出协程，恢复执行
9. 协程从 yield 点恢复，再次调用真实 read() 读取数据
    │
    ▼
返回数据给用户代码
```

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 边缘触发 vs 水平触发

libgo 通常使用边缘触发 (EPOLLET)，原因：减少 epoll_wait 返回次数，提高性能。

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 多线程 epoll 模型

libgo 支持多种多线程 epoll 模型：
- 每个线程独立 epoll（推荐）
- 共享 epoll，多线程 epoll_wait（利用 EPOLLONESHOT 避免惊群）

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 定时器与 epoll 的协作

基于最小堆的定时器管理，与 epoll_wait 的 timeout 配合使用。

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### epoll 数据结构优化

使用 epoll_event.data.ptr 存储协程相关信息，避免每次都查 map。

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

### 性能优化要点

| 优化点 | 说明 |
|--------|------|
| **边缘触发** | 减少 epoll_wait 返回次数 |
| **EPOLLONESHOT** | 多线程共享 epoll 时避免惊群 |
| **批量处理** | 一次 epoll_wait 处理多个事件 |
| **data.ptr** | 避免 fd->协程 的 map 查找 |
| **wakeup 管道** | 使用 eventfd 替代 pipe（更高效） |
| **就绪队列** | 无锁队列减少锁竞争 |

[src: raw/ingested/2技术/go/原理-libgo和go的对比-libgo的钩子函数-libgo的钩子函数.md]

## 相关页面

- [[C++协程库对比-Folly-Asio-Cobalt-libgo]]
- [[Asio-epoll-与协程异步模型]]
- [[Goroutine调度]]
- [[Go语言基础]]
- [[Linux_IO]]
- [[零拷贝与异步IO]]