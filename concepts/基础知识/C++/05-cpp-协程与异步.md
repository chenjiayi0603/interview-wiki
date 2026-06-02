# 协程与异步

> C++20 协程、Boost.Asio、Cobalt、libgo 对比与实践。

---

## 一、C++20 协程

### 1.1 核心概念

| 概念 | 说明 |
|------|------|
| **协程帧（frame）** | 编译器分配的堆内存，含 promise 对象、局部变量、挂起点状态 |
| **coroutine_handle** | 指向帧的句柄（类似指针），`resume()` / `destroy()` |
| **promise_type** | 定义协程行为的类型（返回值、异常处理、挂起策略） |
| **Awaitable / Awaiter** | `await_ready` / `await_suspend` / `await_resume` 三接口 |

### 1.2 三个新关键字

```cpp
co_await some_awaitable;  // 挂起等待异步操作
co_yield value;           // 生成一个值
co_return result;         // 返回结果
```

### 1.3 最小协程示例

```cpp
struct LazyTask {
    struct promise_type {
        LazyTask get_return_object() {
            return LazyTask{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    void resume() { if (!handle.done()) handle.resume(); }
    ~LazyTask() { if (handle) handle.destroy(); }
};
```

### 1.4 自定义 Awaitable

```cpp
struct SimpleDelayAwaitable {
    int ms;
    bool await_ready() const noexcept { return ms <= 0; }
    void await_suspend(std::coroutine_handle<> h) const {
        std::this_thread::sleep_for(std::chrono::milliseconds(ms));
        h.resume();
    }
    void await_resume() const noexcept {}
};
```

---

## 二、协程对比：libgo vs C++20 vs libcopp

| 维度 | libgo | C++20 协程 | libcopp |
|------|-------|------------|---------|
| 栈模型 | 有栈（共享栈/独立栈） | 无栈（stackless） | 有栈 |
| 调度器 | 自带 M:N 调度器 | 无，需自行构建 | 自带 |
| 易用性 | 开箱即用，API 类 Go | 需自定义 Promise/Awaitable | 中等 |
| 性能 | ~124 ns/切换 | 生成器比迭代器慢 3.5-4x | ~91 ns/切换 |
| 适用场景 | 高并发网络 I/O | 简单生成器、惰性序列 | 极致性能 |

**选型建议**：
- 高并发网络服务 → **libgo**
- 简单生成器/异步流 → **C++20 协程**
- 需要标准、少依赖 → **C++20 协程**

---

## 三、Boost.Asio

### 3.1 核心模型

**Proactor 模式**：异步操作完成后通过 completion handler 通知。

```
应用程序 → 发起异步操作 → 立即返回 → io_context::run() 事件循环 → 完成时回调 handler
```

### 3.2 核心类

| 类 | 功能 |
|------|------|
| `io_context` | 事件循环，调度 handler |
| `ip::tcp::socket` | TCP socket |
| `ip::tcp::acceptor` | TCP 监听器 |
| `steady_timer` | 单调时钟定时器 |
| `asio::ssl::stream` | SSL/TLS 加密流 |

### 3.3 基本用法

```cpp
asio::io_context io;

// 定时器示例
asio::steady_timer timer(io, std::chrono::seconds(3));
timer.async_wait([](std::error_code ec) {
    std::cout << "Timer expired\n";
});

io.run();  // 事件循环
```

### 3.4 Asio 与协程结合（C++20）

```cpp
asio::awaitable<void> session(asio::ip::tcp::socket socket) {
    char data[1024];
    auto [ec, n] = co_await socket.async_read_some(
        asio::buffer(data), asio::as_tuple(asio::use_awaitable));
    // ...
}

asio::awaitable<void> server() {
    auto executor = co_await asio::this_coro::executor;
    asio::ip::tcp::acceptor acceptor(executor, {asio::ip::tcp::v4(), 8080});
    while (true) {
        auto socket = co_await acceptor.async_accept(asio::use_awaitable);
        co_spawn(executor, session(std::move(socket)), asio::detached);
    }
}
```

---

## 五、实战建议与常见误区

### 5.1 协程 vs 线程性能对比

| 操作 | 协程切换 | 线程切换（内核） | 进程切换 |
|------|:--------:|:----------------:|:--------:|
| 延迟 | ~10-100ns | ~1-10μs | ~10-100μs |
| 栈开销 | ~KB级（无栈） | ~8MB（有栈） | ~GB级（地址空间） |

### 5.2 常见陷阱

- **协程帧堆分配**：C++20 协程帧默认堆分配，频繁创建销毁有开销（可用 `promise_type::operator new` 定制分配器）
- **永远不挂起的协程**：如果 `await_ready()` 永远返回 true，协程变成了普通函数，失去了异步价值
- **协程句柄生命周期**：`coroutine_handle` 必须确保协程帧未被销毁时调用，否则 undefined behavior
- **Asio 多线程 io_context**：`io_context::run()` 多线程调用时 handler 执行顺序不确定，需注意线程安全

```cpp
#include <boost/cobalt.hpp>
namespace cobalt = boost::cobalt;

cobalt::main co_main(int argc, char* argv[]) {
    co_await do_something();
    co_return 0;
}

// 生成器
cobalt::generator<int> range(int n) {
    for (int i = 0; i < n; ++i)
        co_yield i;
}
```

---

## 六、面试高频追问

### Q1: C++20 协程是无栈（stackless）的，和有栈协程比有什么优劣？

| 维度 | 无栈（C++20 协程） | 有栈（libgo/libcopp） |
|:----|:-----------------|:--------------------|
| 内存开销 | ~KB（仅保存局部变量和挂起点） | ~MB（独立栈空间，可共享栈优化） |
| 切换速度 | ~10-100ns（纯用户态指针交换） | ~100-200ns（需切换栈指针） |
| 嵌套调用 | 仅顶层可挂起，普通函数不可含 `co_await` | 任意嵌套深度可挂起 |
| 调度器 | 无内置调度器，需自行构建 | 自带 M:N 调度器，开箱即用 |
| 适用场景 | 生成器、简单异步流 | 高并发网络 I/O、复杂业务逻辑 |

**核心区别**：无栈协程的挂起点只能在协程函数体内（不能在其调用的普通函数中），有栈协程可以像线程一样任意深度挂起。

### Q2: 为什么 C++20 协程帧在堆上分配？有什么优化手段？

- **原因**：协程帧大小在编译期未知（取决于局部变量和 promise_type），且生命周期与调用者解耦
- **堆分配开销**：频繁创建/销毁协程帧会产生内存碎片和分配延迟
- **优化手段**：

```cpp
// 1. 定制分配器（promise_type 中重载 operator new）
struct MyPromise {
    void* operator new(size_t sz) {
        return pooled_alloc(sz);  // 从内存池分配
    }
    void operator delete(void* ptr, size_t sz) {
        pooled_free(ptr, sz);
    }
};

// 2. 协程复用（协程池）
// 3. 避免过小的协程（普通函数比协程快）
```

### Q3: `co_await` 的三个接口分别控制什么？

| 接口 | 返回值含义 | 控制点 |
|:----|:----------|:-------|
| `await_ready()` | true = 不需挂起（同步完成） | 决定是否立即执行 |
| `await_suspend(handle)` | void/bool/coroutine_handle | 挂起后做什么（调度/切换/恢复其他） |
| `await_resume()` | 任意类型 | `co_await` 表达式的返回值 |

```cpp
// 典型完整实现
struct MyAwaitable {
    bool await_ready() { return false; }  // 总是挂起
    
    void await_suspend(std::coroutine_handle<> h) {
        // 可在此将 h 存起，稍后恢复
        executor.post([h] { h.resume(); });
    }
    
    int await_resume() { return 42; }  // co_await 表达式值为 42
};
```

### Q4: Boost.Asio 的 Proactor 模式与 Reactor 模式的区别？

| 模式 | 工作方式 | 事件通知 | 典型实现 |
|:----|:--------|:---------|:---------|
| **Reactor** | IO 就绪时通知，应用层负责读写 | "可以读/写了" | epoll（Linux）、kqueue（macOS） |
| **Proactor** | IO 完成时通知，异步引擎负责读写 | "已经读/写完了" | IOCP（Windows）、Asio 封装 |

**Asio 的跨平台实现**：
- Linux：epoll + 模拟 Proactor（发起操作后立即返回，完成时通过 epoll 通知）
- Windows：IOCP（原生 Proactor）
- 开发者接口始终是 Proactor 风格（`async_read` + handler），底层自动适配

### Q5: 什么时候应该用协程而不是线程？

```cpp
// ✅ 适合协程：大量并发连接，每个连接少量计算
asio::awaitable<void> handle_connection(tcp::socket sock) {
    char buf[1024];
    for (;;) {
        auto n = co_await sock.async_read_some(buffer(buf), use_awaitable);
        if (n == 0) co_return;
        co_await async_write(sock, buffer(buf, n), use_awaitable);
    }
}

// ❌ 适合线程：CPU 密集型计算
std::thread([&] { result = heavy_computation(data); }).detach();
```

**选型速查**：

| 场景 | 推荐 | 原因 |
|:----|:-----|:------|
| 大量网络连接（>10000） | 协程 | 线程数受限于系统，协程可百万级 |
| CPU 密集型计算 | 线程 | 多核并行，协程本质是协作式调度 |
| IO 密集型 + 高吞吐 | 协程 | 避免线程上下文切换 |
| 复杂业务逻辑 + 深调用栈 | 线程/有栈协程 | C++20 无栈协程不能深度嵌套 |
| 简单生成器/惰性序列 | C++20 协程 | 比迭代器实现更简洁 |

### Q6: `co_yield` 和 `co_return` 底层分别如何工作？

- **`co_yield value`** → 等价于 `co_await promise.yield_value(value)`
  - 生成器模式：挂起当前协程，将值传递给调用者
  - 调用者通过 `resume()` 获取下一个值

- **`co_return expr`** → 调用 `promise.return_value(expr)` 或 `promise.return_void()`
  - 存入结果后执行 final_suspend（通常 `suspend_always` 以便调用者获取结果）
  - 协程帧此时仍存在，直至 `coroutine_handle` 调用 `destroy()`

### Q7: C++20 协程 vs Golang goroutine 对比？

| 维度 | C++20 协程 | Go goroutine |
|:----|:----------|:-------------|
| 类型 | 无栈协程（stackless） | 有栈协程（stackful） |
| 栈大小 | ~KB（帧） | 初始 2-4KB，可增长（最高 1GB） |
| 调度 | 无内置调度器 | 内置 M:N 调度器（GMP 模型） |
| 阻塞 | 挂起当前协程 | 挂起 goroutine，M 绑定到其他 G |
| 通道 | 需自行实现 | 语言级 channel |
| 错误处理 | 返回值/异常 | panic/recover |
| 成熟度 | 较新（C++20），生态仍在发展 | 成熟，Go 语言核心特性 |

**核心差异**：Go 的 goroutine 背后有运行时的完整支持（GMP 调度器、可增长栈、channel），C++20 协程是语言层面的最小原语，调度器和基础设施需要自行构建或借助库（Asio、libgo）。
