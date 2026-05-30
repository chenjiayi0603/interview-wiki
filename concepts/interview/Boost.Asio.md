# Boost.Asio

> Boost.Asio 是跨平台 C++ 异步 I/O 库，用于网络和低层 I/O 编程，提供统一的异步模型。本文档覆盖其核心功能、常用 API 与项目中的示例代码。

See also: [[C++网络编程]], [[C++多线程与并发]], [[C++20协程]], [[Asio-Cobalt协程场景示例]]

---

# 一、概述与设计模型

## 1.1 核心定位

- **跨平台**：Linux（epoll）、macOS（kqueue）、Windows（IOCP）、BSD 等
- **异步模型**：Reactor 模式，基于完成回调（completion handler）
- **现代 C++**：配合 C++11/14/17/20 协程、executor 等特性

## 1.2 同步 vs 异步

| 模式 | 行为 | 典型 API |
|------|------|----------|
| 同步 | 调用阻塞直到操作完成 | `read()`, `write()`, `connect()` |
| 异步 | 发起后立即返回，完成时回调 | `async_read()`, `async_write()`, `async_connect()` |

异步操作在后台执行，不阻塞调用线程，完成时由 `io_context::run()` 调度的 handler 处理结果。

## 1.3 设计原则

- **Proactor 模式**：异步操作完成后通过 completion handler 通知
- **资源释放**：操作完成且 handler 被调用前，临时资源（fd、内存）已释放，便于高效链式组合
- **Executor 模型**：控制 handler 在哪个执行上下文运行（单线程、strand、线程池等）

---

# 二、核心功能分类

## 2.1 网络能力

| 协议/功能 | 说明 | 典型类 |
|-----------|------|--------|
| TCP | 流式传输，客户端/服务端 | `ip::tcp::socket`, `ip::tcp::acceptor`, `ip::tcp::resolver` |
| UDP | 无连接数据报 | `ip::udp::socket` |
| ICMP | 网络诊断（如 ping） | `ip::icmp::socket` |
| SSL/TLS | 加密通信 | `asio::ssl::stream<socket>` |
| Unix Domain | 本地进程间通信 | `local::stream_protocol::socket` |

## 2.2 低层 I/O

| 类型 | 说明 |
|------|------|
| 文件 | 异步文件读写 |
| 管道 | 进程间通信 |
| 串口 | 串口通信 |
| 信号 | 信号处理（如 SIGINT、SIGTERM） |
| POSIX 流 | 任意流式文件描述符 |

## 2.3 任务调度与时序

| 功能 | 说明 |
|------|------|
| `io_context` | 事件循环，调度 handler |
| `post` / `dispatch` | 投递任务到执行器 |
| `steady_timer` | 单调时钟定时器（推荐） |
| `deadline_timer` | 系统时钟定时器 |
| `high_resolution_timer` | 高精度定时器 |

## 2.4 并发与线程安全

| 功能 | 说明 |
|------|------|
| `strand` | 同一 strand 上的 handler 串行执行，避免锁 |
| `thread_pool` | 多线程执行器 |
| `executor` | 抽象执行上下文 |
| 多线程 `run()` | 多线程共同驱动同一 `io_context` |

## 2.5 协程支持

| 类型 | 说明 |
|------|------|
| Stackless 协程 | `yield_context` |
| Stackful 协程 | `spawn()` |
| C++20 协程 | `co_await`, `co_spawn`, `use_awaitable` |

## 2.6 高级机制

| 功能 | 说明 |
|------|------|
| Completion Tokens | `use_future`, `use_awaitable`, `deferred` |
| Cancellation | 操作取消 |
| Handler 追踪 | 调试 handler 调用链 |
| 自定义内存分配 | 为 handler 等提供 allocator |

---

# 三、常用 API 速查

## 3.1 核心类

`io_context` = 任务队列（post/dispatch）+ I/O 多路复用（Linux 下为 epoll）

```
io_context        事件循环/调度器
strand            串行化 handler，保证顺序
executor          执行器抽象
steady_timer      定时器（单调时钟）
thread_pool       多线程池
```

## 3.2 TCP 相关

```
ip::tcp::socket      流式 socket
ip::tcp::acceptor    监听与 accept
ip::tcp::resolver    域名解析
ip::tcp::endpoint    端点（地址+端口）
```

## 3.3 UDP 相关

```
ip::udp::socket      数据报 socket
ip::udp::endpoint    端点
```

## 3.4 缓冲区

```
mutable_buffer       可写缓冲区
const_buffer         只读缓冲区
streambuf            流式缓冲区（可自动扩展）
dynamic_buffer       动态缓冲区概念
```

---

# 四、常用功能与技巧（附小例子）

## 4.1 最小同步 TCP 客户端

适合快速验证服务是否可连、收发是否正常。

```cpp
boost::asio::io_context ioc;
boost::asio::ip::tcp::resolver resolver(ioc);
boost::asio::ip::tcp::socket socket(ioc);

auto endpoints = resolver.resolve("127.0.0.1", "8080");
boost::asio::connect(socket, endpoints);

std::string msg = "hello\n";
boost::asio::write(socket, boost::asio::buffer(msg));

char buf[1024];
size_t n = socket.read_some(boost::asio::buffer(buf));
std::cout.write(buf, n);
```

## 4.2 最小异步 TCP Echo 服务器

单连接版本，核心是 `async_read_some` + `async_write` 链式回调。

```cpp
boost::asio::io_context ioc;
boost::asio::ip::tcp::acceptor acc(ioc, {{}, 8080});

boost::asio::ip::tcp::socket sock(ioc);
acc.async_accept(sock, [&](auto ec) {
    if (ec) return;
    auto buf = std::make_shared<std::array<char, 1024>>();
    auto do_read = [&, buf](auto&& self) mutable {
        sock.async_read_some(boost::asio::buffer(*buf),
            [&, buf, self](auto ec2, std::size_t n) mutable {
                if (ec2) return;
                boost::asio::async_write(sock, boost::asio::buffer(*buf, n),
                    [&, buf, self](auto ec3, std::size_t) mutable {
                        if (!ec3) self(self);  // 继续读
                    });
            });
    };
    do_read(do_read);
});
ioc.run();
```

## 4.3 steady_timer 定时任务（周期心跳）

使用 `expires_after` + 递归 `async_wait` 实现周期性任务。

```cpp
boost::asio::io_context ioc;
boost::asio::steady_timer timer(ioc);

std::function<void()> tick;
tick = [&] {
    timer.expires_after(std::chrono::seconds(1));
    timer.async_wait([&](auto ec) {
        if (ec) return;               // 可能被取消
        std::cout << "tick\n";
        tick();                       // 再次安排下一次
    });
};
tick();
ioc.run();
```

## 4.4 strand 串行化共享状态

在多线程 `run()` 或 `thread_pool` 中，保护共享容器不用显式加锁。

```cpp
boost::asio::io_context ioc;
auto strand = boost::asio::make_strand(ioc);
std::vector<int> data;

for (int i = 0; i < 100; ++i) {
    boost::asio::post(strand, [&, i] {
        data.push_back(i);  // 始终在 strand 上执行，线程安全
    });
}

std::jthread t1([&]{ ioc.run(); });
std::jthread t2([&]{ ioc.run(); });
```

## 4.5 async_read + 动态缓冲区读一行

利用 `async_read_until` + `streambuf` 读取以 `\n` 结尾的一行。

```cpp
boost::asio::ip::tcp::socket socket(ioc);
boost::asio::streambuf buf;

boost::asio::async_read_until(socket, buf, '\n',
    [&](auto ec, std::size_t n) {
        if (ec) return;
        std::istream is(&buf);
        std::string line;
        std::getline(is, line);
        std::cout << "line: " << line << "\n";
    });
```

## 4.6 使用 C++20 协程（co_spawn + use_awaitable）

让异步逻辑看起来像同步代码，易读易维护。

```cpp
using boost::asio::awaitable;
using boost::asio::use_awaitable;

awaitable<void> echo_session(boost::asio::ip::tcp::socket socket) {
    char data[1024];
    for (;;) {
        std::size_t n = co_await socket.async_read_some(
            boost::asio::buffer(data), use_awaitable);
        co_await boost::asio::async_write(
            socket, boost::asio::buffer(data, n), use_awaitable);
    }
}

// 在 main 中：
// co_spawn(ioc, echo_session(std::move(sock)), detached);
```

## 4.7 超时控制 + 取消操作

使用定时器 + `cancel()` 给某个异步操作加超时。

```cpp
boost::asio::io_context ioc;
boost::asio::ip::tcp::socket socket(ioc);
boost::asio::steady_timer timer(ioc);

// 连接
socket.async_connect({boost::asio::ip::make_address("1.2.3.4"), 80},
    [&](auto ec) { /* 连接结果 */ });

// 3 秒超时
timer.expires_after(std::chrono::seconds(3));
timer.async_wait([&](auto ec) {
    if (!ec) {
        socket.cancel();  // 触发 connect 的 handler，返回 operation_aborted
    }
});

ioc.run();
```

## 4.8 最小 UDP 收发示例

UDP 适合简单的请求/响应或广播场景。

```cpp
boost::asio::io_context ioc;
boost::asio::ip::udp::socket sock(ioc, {{}, 9000});

boost::asio::ip::udp::endpoint sender;
std::array<char, 1024> buf{};

sock.async_receive_from(boost::asio::buffer(buf), sender,
    [&](auto ec, std::size_t n) {
        if (ec) return;
        sock.async_send_to(boost::asio::buffer(buf, n), sender,
            [](auto, std::size_t) {});
    });

ioc.run();
```

---

# 五、项目示例解析

以下示例来自 `algorithm/performance-code-testing/modules/boost_asio_examples.cpp`。

## 5.1 io_context + post：异步任务投递

单线程事件循环，任务按投递顺序执行。源码：`boost_asio_examples.cpp:38-48`。

```cpp
static void example_boost_asio_post(size_t n) {
    measure("boost::asio::post", [n] {
        boost::asio::io_context ioc;
        std::atomic<size_t> cnt{0};
        for (size_t i = 0; i < n; ++i) {
            boost::asio::post(ioc, [&cnt] { ++cnt; });
        }
        ioc.run();
        assert(cnt.load() == n);
    });
}
```

- `measure(name, fn)`：项目中的耗时测量包装，用于 benchmark
- 调用时 `n` 通常为 1e5 量级，用于测试 post 吞吐

- `post(ioc, handler)`：将 handler 投递到 `ioc` 的队列
- `ioc.run()`：阻塞直到所有任务完成且无更多工作

**注意**：post 体现的是「延后执行」，而非 I/O 重叠。`post()` 立即返回、handler 延后到 `run()` 里执行，但 `ioc.run()` 会阻塞到所有任务完成，整体仍是同步等待。真正的异步是 `async_read` / `async_accept` 等 I/O 操作：发起后立即返回，多连接在单线程内交错处理。见下文 5.5。

## 5.2 steady_timer：异步定时器

定时器到时后，handler 在 `io_context::run()` 中被调用。

```cpp
void example_boost_asio_timer() {
    boost::asio::io_context ioc;
    boost::asio::steady_timer timer(ioc, std::chrono::milliseconds(10));
    std::atomic<bool> done{false};
    timer.async_wait([&](const boost::system::error_code& ec) {
        (void)ec;
        done.store(true);
    });
    ioc.run();
    assert(done.load());
}
```

- `steady_timer`：基于 `std::chrono::steady_clock`，不受系统时间调整影响
- `async_wait(handler)`：异步等待，到时调用 handler

## 5.3 strand：串行化访问

同一 strand 上的 handler 串行执行，共享数据无需加锁。

```cpp
void example_boost_asio_strand(size_t n) {
    boost::asio::io_context ioc;
    auto strand = boost::asio::make_strand(ioc);
    std::vector<int> seq;
    seq.reserve(n);
    for (size_t i = 0; i < n; ++i) {
        boost::asio::post(strand, [&seq, i] { seq.push_back(static_cast<int>(i)); });
    }
    ioc.run();
    assert(seq.size() == n);
}
```

- `make_strand(ioc)`：创建与 `ioc` 关联的 strand
- `post(strand, handler)`：保证 handler 在 strand 上串行执行

## 5.4 thread_pool：多线程并行

利用多核并行执行任务。

```cpp
void example_boost_asio_thread_pool(size_t n) {
    boost::asio::thread_pool pool(std::thread::hardware_concurrency());
    std::atomic<size_t> cnt{0};
    for (size_t i = 0; i < n; ++i) {
        boost::asio::post(pool, [&cnt] { ++cnt; });
    }
    pool.join();
    assert(cnt.load() == n);
}
```

- `thread_pool(N)`：创建 N 个 worker 线程
- `post(pool, handler)`：任务会被任意 worker 执行
- 若访问共享数据需同步（如用 `strand` 或 `mutex`）

## 5.5 完整异步示例：TCP Echo 服务器

单线程同时维护多个连接，`async_accept` / `async_read` / `async_write` 发起后立即返回，I/O 等待时由 epoll 通知，体现真正的异步重叠。

```cpp
#include <boost/asio.hpp>
#include <memory>
#include <functional>

void do_read(std::shared_ptr<boost::asio::ip::tcp::socket> sock);

void run_async_echo_server(uint16_t port) {
    boost::asio::io_context ioc;
    boost::asio::ip::tcp::acceptor acceptor(ioc, {{}, port});

    std::function<void()> do_accept;
    do_accept = [&] {
        auto sock = std::make_shared<boost::asio::ip::tcp::socket>(ioc);
        acceptor.async_accept(*sock, [&, sock](auto ec) {
            if (ec) return;
            do_read(sock);
            do_accept();  // 立即继续 accept 下一个，不等待
        });
    };
    do_accept();

    ioc.run();  // 单线程：accept、read、write 交错完成
}

void do_read(std::shared_ptr<boost::asio::ip::tcp::socket> sock) {
    auto buf = std::make_shared<std::array<char, 4096>>();
    sock->async_read_some(boost::asio::buffer(*buf), [sock, buf](auto ec, size_t n) {
        if (ec || n == 0) return;
        boost::asio::async_write(*sock, boost::asio::buffer(*buf, n),
            [sock](auto ec, size_t) { if (!ec) do_read(sock); });
    });
}
```

**异步体现**：`async_accept` / `async_read_some` / `async_write` 调用后立即返回，线程不阻塞；等待 I/O 时处理其他连接；epoll 通知就绪后执行对应 handler。

**同步对比**：若用 `acceptor.accept()`、`read_some()`、`write()`，每个操作都会阻塞，单线程同时只能服务 1 个连接。异步模式下 100 个连接单线程即可处理。

---

# 六、典型用法模式

## 6.1 单线程异步服务器

```cpp
boost::asio::io_context ioc;
boost::asio::ip::tcp::acceptor acceptor(ioc, endpoint);
// async_accept -> async_read -> async_write 链式调用
// 所有 handler 在同一线程执行，无需锁
```

## 6.2 多线程 io_context

```cpp
boost::asio::io_context ioc;
// 多线程 run
std::vector<std::thread> threads;
for (int i = 0; i < N; ++i)
    threads.emplace_back([&ioc] { ioc.run(); });
```

共享 `io_context` 时，访问共享状态需用 `strand` 或锁。

## 6.3 strand 保护共享状态

```cpp
auto strand = boost::asio::make_strand(ioc);
// 所有访问共享数据的 handler 都 post 到 strand
boost::asio::post(strand, [&shared_data] { /* 安全访问 */ });
```

## 6.4 TCP 客户端连接示例

```cpp
boost::asio::io_context ioc;
boost::asio::ip::tcp::socket socket(ioc);
boost::asio::ip::tcp::resolver resolver(ioc);

resolver.async_resolve(host, port,
    [&](auto ec, auto it) {
        boost::asio::async_connect(socket, it,
            [](auto ec, auto) { /* 连接完成 */ });
    });
ioc.run();
```

---

# 七、安装与编译

## 7.1 安装

```bash
# Ubuntu/Debian
sudo apt install libboost-dev

# 或仅 Asio（header-only，无 Boost 依赖）
# 使用 standalone Asio: https://think-async.com/Asio/
```

## 7.2 编译

```bash
g++ -std=c++17 -o app main.cpp -lpthread
# 若用 Boost 版，通常只需 -I 和 -lpthread
```

---

# 八、与其他方案对比

| 方案 | 特点 |
|------|------|
| Boost.Asio | 成熟、跨平台、功能全，头文件多 |
| standalone Asio | 与 Boost.Asio 接口兼容，无 Boost 依赖 |
| libuv | C 库，Node.js 底层，回调风格 |
| 标准库 net（C++23） | 基于 Asio 设计，尚未普及 |

**说明**：C++23 并未正式纳入 `std::net`，当前多为 **Networking TS**（如 `std::experimental::net`）或后续标准。其 API 设计基于 Asio，命名与用法与 Asio 高度相似。下面用"假设的 std::net 风格"举例（实际编译需 TS 实现或 Asio）。

**1）同步 TCP 服务端（风格类比 Asio）**

```cpp
// 假设的 std::net API（与 Asio 对应关系）
#include <net>
namespace net = std::net;  // 或 std::experimental::net

net::io_context ioc;
net::ip::tcp::endpoint ep(net::ip::tcp::v4(), 8080);
net::ip::tcp::acceptor acceptor(ioc, ep);

while (true) {
    net::ip::tcp::socket socket(ioc);
    acceptor.accept(socket);
    std::string msg = "Hello from std::net\n";
    std::error_code ec;
    net::write(socket, net::buffer(msg), ec);
}
```

**2）异步 TCP 客户端（风格类比 Asio）**

```cpp
// 异步连接 + 异步读，命名与 Asio 一致
net::io_context ioc;
net::ip::tcp::socket socket(ioc);
net::ip::tcp::resolver resolver(ioc);

auto endpoints = resolver.resolve("localhost", "8080");
net::async_connect(socket, endpoints, [&](std::error_code ec) {
    if (!ec)
        net::async_read_some(socket, net::buffer(buf), [](auto ec, size_t n) { /* ... */ });
});
ioc.run();
```

**与 Asio 的对应关系**：上面若把 `net::` 换成 `asio::`、`#include <net>` 换成 `#include <asio.hpp>`，即为可编译的 Asio 代码。当前工程中直接使用 **Boost.Asio** 或 **standalone Asio** 即可。

---

# 九、参考链接

- [Boost.Asio 官方文档](https://www.boost.org/doc/libs/latest/doc/html/boost_asio.html)
- [standalone Asio](https://think-async.com/Asio/)
- 项目示例：`algorithm/performance-code-testing/modules/boost_asio_examples.cpp`

[src: raw/ingested/2技术/cpp/并行库-Boost.Asio完整手册.md]

## Related Pages
- [[C++网络编程]]
- [[C++20协程]]
- [[Asio-Cobalt协程场景示例]]
- [[C++多线程与并发]]
- [[Linux_IO]]
- [[TCP协议]]
