# Asio / Cobalt 协程场景示例

> 参考 [[C++20协程]]，涵盖 Boost.Asio 与 Boost.Cobalt 协程相关的全部典型场景。

---

## 一、环境与头文件

```cpp
// 编译需 C++20，链接 Boost
// g++ -std=c++20 -I/path/to/boost example.cpp -lpthread

#include <boost/asio.hpp>
#include <boost/cobalt.hpp>
#include <boost/cobalt/main.hpp>
#include <boost/cobalt/promise.hpp>
#include <boost/cobalt/task.hpp>
#include <boost/cobalt/generator.hpp>
#include <boost/cobalt/channel.hpp>
#include <boost/cobalt/join.hpp>
#include <boost/cobalt/race.hpp>
#include <boost/cobalt/gather.hpp>
#include <boost/cobalt/with.hpp>
#include <boost/cobalt/wait_group.hpp>
#include <iostream>
#include <chrono>

namespace asio = boost::asio;
namespace cobalt = boost::cobalt;
```

---

## 二、入口方式

### 2.1 co_main（推荐入门）

**底层实现**：`cobalt::main` 由 Cobalt 提供入口，内部创建 `io_context`、设置默认 executor，并注册 SIGINT/SIGTERM 到取消。协程在单线程事件循环中执行，I/O 由 Asio reactor 驱动：Linux 使用 `epoll`，macOS/BSD 使用 `kqueue`。

```cpp
// 自动配置 io_context、内存、SIGINT/SIGTERM 取消
cobalt::main co_main(int argc, char* argv[])
{
    co_await do_something();
    co_return 0;
}
```

### 2.2 cobalt::thread

**底层实现**：本质是 `cobalt::promise<void>` 协程 + `join()` 同步等待。协程在独立线程内运行自己的 `io_context::run()`，`t.join()` 阻塞直到协程完成。不涉及 epoll/kqueue，由独立线程的事件循环驱动。

```cpp
cobalt::promise<void> my_thread()
{
    co_await do_something();
    co_return;
}

int main()
{
    //my_thread 本身是协程；join() 是协程库的同步等待接口；底层实现里会用到真正的 OS 线程来跑那条协程所在的事件循环。
    auto t = my_thread();
    t.join();
    return 0;
}
```

### 2.3 task + run（同步等待）

**底层实现**：`cobalt::task<T>` 是 C++20 无栈协程的返回类型，内部持有 `std::coroutine_handle<promise_type>`，协程帧在堆上分配。`task` 为**惰性**：构造时不执行，`cobalt::run()` 内部创建或复用 `io_context`，在调用线程中 `post` 启动协程并 `run()` 直到完成。Asio 侧在 Linux 使用 `epoll`（epoll_create/epoll_wait/epoll_ctl），macOS/BSD 使用 `kqueue`，定时器等通过 reactor 的事件循环驱动。

```cpp
cobalt::task<int> my_task()
{
    co_await do_something();
    co_return 42;
}

int main()
{
    int res = cobalt::run(my_task());
    return res;
}
```

### 2.4 task + spawn（异步启动）

**底层实现**：`cobalt::spawn(ctx, task, detached)` 将 task 的首次执行 `post` 到 `io_context`，由 `ctx.run()` 的线程驱动。底层与 Asio `co_spawn` 类似，复用同一 reactor（epoll/kqueue），多个协程共享同一事件循环。

```cpp
cobalt::task<void> background_work()
{
    co_await do_something();
    co_return;
}

int main()
{
    asio::io_context ctx;
    cobalt::spawn(ctx, background_work(), asio::detached);
    ctx.run();
    return 0;
}
```

---

## 三、协程类型

### 3.1 promise\<T\>（急切，默认推荐）

**底层实现**：`cobalt::promise<T>` 也是 C++20 协程返回类型，与 `task` 不同之处是**急切**：函数返回时协程已在当前 executor 上 `post` 启动。内部通过 `coroutine_handle` 引用协程帧，`co_await p` 时若未完成则挂起当前协程，由 Asio 的 completion handler 在 reactor 就绪后恢复。I/O 复用层：Linux 用 `epoll`，BSD/macOS 用 `kqueue`，定时器基于 `timerfd`（Linux）或 kqueue 的 EVFILT_TIMER。

```cpp
cobalt::promise<int> fetch_value()
{
    co_await asio::steady_timer{co_await cobalt::this_coro::executor,
                                std::chrono::milliseconds(10)}.async_wait(cobalt::use_op);
    co_return 42;
}

cobalt::main co_main(int argc, char* argv[])
{
    auto p = fetch_value();  // 立即开始执行
    co_await do_other_thing();
    int res = co_await p;    // 等待完成
    std::cout << res << std::endl;
    co_return 0;
}
```

### 3.2 task\<T\>（惰性，可换执行器）

**底层实现**：`task<T>` 惰性执行，首次 `co_await` 时才从 awaiter 继承 executor 并 `post` 启动协程。相比 `promise` 可跨 executor 使用；默认用 `pmr::get_default_resource()` 分配协程帧，也可通过 `std::allocator_arg` 指定自定义分配器。

```cpp
cobalt::task<std::string> lazy_work()
{
    co_await asio::steady_timer{co_await cobalt::this_coro::executor,
                                std::chrono::milliseconds(50)}.async_wait(cobalt::use_op);
    co_return "done";
}

cobalt::main co_main(int argc, char* argv[])
{
    auto t = lazy_work();    // 不执行
    co_await do_other_thing();
    auto res = co_await t;   // 此时才启动并等待
    co_return 0;
}
```

### 3.3 generator\<T\>（多值产出）

**底层实现**：`cobalt::generator<T>` 基于 C++20 协程的 `co_yield`，内部保存 `coroutine_handle`，每次 `co_await g` 恢复协程至下一个 `co_yield` 并取回值。无 POSIX 系统调用，纯用户态状态机；若协程内有 `co_await` I/O，则挂起时由 Asio reactor（epoll/kqueue）驱动。

```cpp
cobalt::generator<int> iota(int max)
{
    for (int i = 0; i < max; ++i)
        co_yield i;
    co_return max;
}

cobalt::main co_main(int argc, char* argv[])
{
    auto g = iota(5);
    while (g)
        std::cout << co_await g << " ";
    co_return 0;
}
```

### 3.4 detached（fire-and-forget）

**底层实现**：`cobalt::detached` 是无返回值的协程类型，调用时立即 `post` 到当前 executor，不持有句柄、不等待。协程帧在完成后由框架释放；若协程内做 I/O，底层仍通过 Asio reactor 复用（epoll/kqueue/select）。

```cpp
cobalt::detached log_async()
{
    co_await asio::steady_timer{co_await cobalt::this_coro::executor,
                                std::chrono::seconds(1)}.async_wait(cobalt::use_op);
    std::cout << "log done" << std::endl;
    co_return;
}

cobalt::main co_main(int argc, char* argv[])
{
    log_async();  // 启动后不等待
    co_await do_something();
    co_return 0;
}
```

### 3.5 generator 推值（push 模式）

```cpp
// generator<T, PushType>：co_yield 可接收调用方传入的值
cobalt::generator<double, int> scale(int init)
{
    int v = init;
    while (v != 0) {
        v = co_yield v * 0.1;  // 返回 v*0.1，并接收下一个 push 值
    }
    co_return 0;
}

cobalt::main co_main(int argc, char* argv[])
{
    auto g = scale(5);
    assert(0.5 == co_await g(4));  // push 4，得 0.5
    assert(0.4 == co_await g(3));  // push 3，得 0.4
    co_return 0;
}
```

---

## 四、同步组合：join / race / gather

### 4.1 join（等全部完成，任一异常则抛出）

**底层实现**：`cobalt::join` 并发启动多个 `promise`/`task`，内部用 Asio 的 `experimental::concurrent_wait` 或类似机制等待所有完成；任一异常即传播。无额外 POSIX 调用，纯协程组合 + Asio post/run。

```cpp
cobalt::promise<int> work_a() { /* ... */ co_return 1; }
cobalt::promise<double> work_b() { /* ... */ co_return 2.0; }

cobalt::main co_main(int argc, char* argv[])
{
    auto [a, b] = co_await cobalt::join(work_a(), work_b());
    std::cout << a << ", " << b << std::endl;
    co_return 0;
}
```

### 4.2 race（任一完成即返回，伪随机防饥饿）

**底层实现**：`cobalt::race` 并发等待多个协程，任一完成即返回 variant，其余取消。底层通过 Asio cancellation_slot 传播取消，协程挂起在 epoll/kqueue 等待时收到取消即提前返回。

```cpp
cobalt::promise<int> slow() {
    co_await asio::steady_timer{co_await cobalt::this_coro::executor,
                                std::chrono::seconds(10)}.async_wait(cobalt::use_op);
    co_return 1;
}
cobalt::promise<int> fast() {
    co_await asio::steady_timer{co_await cobalt::this_coro::executor,
                                std::chrono::milliseconds(10)}.async_wait(cobalt::use_op);
    co_return 2;
}

cobalt::main co_main(int argc, char* argv[])
{
    auto v = co_await cobalt::race(slow(), fast());
    // v 为 variant，根据 index() 取值
    std::cout << "first: " << (v.index() == 0 ? std::get<0>(v) : std::get<1>(v)) << std::endl;
    co_return 0;
}
```

### 4.3 left_race（确定性，从左到右）

```cpp
auto v = co_await cobalt::left_race(slow(), fast());
// 同 race，但选择顺序确定
```

### 4.4 gather（等全部完成，各捕获异常）

```cpp
cobalt::promise<int> may_throw(bool ok) {
    if (!ok) throw std::runtime_error("fail");
    co_return 42;
}

cobalt::main co_main(int argc, char* argv[])
{
    auto [r1, r2] = co_await cobalt::gather(may_throw(true), may_throw(false));
    // r1, r2 为 boost::outcome::result<T,E>
    if (r1) std::cout << *r1 << std::endl;
    if (!r2) std::cout << r2.error().message() << std::endl;
    co_return 0;
}
```

---

## 五、channel（协程间通信）

### 5.1 生产者-消费者

**底层实现**：`cobalt::channel<T>` 是协程间有界队列，`write`/`read` 在队列满/空时挂起，由 Asio 的 completion handler 在对方 `write`/`read` 时恢复。底层通常用锁 + 条件变量或 Asio 的 `post` 链实现，不直接使用 epoll；与 reactor 集成在同一线程中串行执行。

```cpp
cobalt::promise<void> producer(cobalt::channel<int>& ch)
{
    for (int i = 0; i < 5; ++i) {
        co_await ch.write(i);
    }
    ch.close();
    co_return;
}

cobalt::promise<void> consumer(cobalt::channel<int>& ch)
{
    while (ch.is_open()) {
        try {
            int v = co_await ch.read();
            std::cout << "got " << v << std::endl;
        } catch (...) { break; }
    }
    co_return;
}

cobalt::main co_main(int argc, char* argv[])
{
    cobalt::channel<int> ch(1);  // 容量 1
    co_await cobalt::join(producer(ch), consumer(ch));
    co_return 0;
}
```

### 5.1.1 多线程生产者-消费者

**说明**：与 5.1 不同，此处多个线程共享 `io_context`，producer 和 consumer 由不同线程派发，需用 **Asio 的 `experimental::concurrent_channel`**（线程安全）。`cobalt::channel` 仅适用于同一 strand，不能跨线程共享。

```cpp
#include <boost/asio/experimental/concurrent_channel.hpp>

using channel_type = asio::experimental::concurrent_channel<void(asio::error_code, int)>;

cobalt::promise<void> producer(std::shared_ptr<channel_type> ch)
{
    for (int i = 0; i < 5; ++i) {
        auto ec = co_await ch->async_send(asio::error_code{}, i, cobalt::use_op);
        if (ec) break;
    }
    ch->close();
}

cobalt::promise<void> consumer(std::shared_ptr<channel_type> ch)
{
    while (ch->is_open()) {
        auto [ec, v] = co_await ch->async_receive(cobalt::use_op);
        if (ec) break;
        std::cout << "got " << v << std::endl;
    }
}

int main()
{
    asio::io_context ctx;
    auto ch = std::make_shared<channel_type>(ctx, 1);

    cobalt::spawn(ctx, producer(ch), asio::detached);
    cobalt::spawn(ctx, consumer(ch), asio::detached);

    std::vector<std::thread> threads;
    for (int i = 0; i < 2; ++i)
        threads.emplace_back([&] { ctx.run(); });
    for (auto& t : threads)
        t.join();
    return 0;
}
```

**底层实现**：`concurrent_channel` 内部用锁 + 条件变量实现线程安全，可从任意线程调用 `async_send`/`async_receive`。多个线程 `ctx.run()` 时，completion handler 会被不同线程派发，producer 和 consumer 可运行在不同线程。

### 5.2 channel 与 race 组合（多路选择）

```cpp
cobalt::promise<void> select_example(cobalt::channel<int>& ch)
{
    auto timer = asio::steady_timer{co_await cobalt::this_coro::executor,
                                    std::chrono::seconds(1)};
    auto wait_timer = timer.async_wait(cobalt::use_op);

    switch (auto v = co_await cobalt::race(ch.read(), wait_timer); v.index()) {
    case 0:
        std::cout << "channel: " << std::get<0>(v) << std::endl;
        break;
    case 1:
        std::cout << "timeout" << std::endl;
        break;
    }
    co_return;
}
```

---

## 六、with 与 wait_group（资源与作用域）

### 6.1 with（异步 RAII）

异步 RAII是指在协程/异步上下文中，像RAII（Resource Acquisition Is Initialization）一样，自动管理资源的申请和释放，确保无论协程如何挂起、恢复、提前退出，都可以正确地初始化和清理资源。
在 Cobalt 的 with 实现里，它会在进入 .with() 的 lambda 或回调前自动创建资源对象，并在作用域结束（即 lambda 或协程退出时）自动等待并清理（通过 co_await 等待资源释放/相关协程全部完成）。
这非常适合需要异步清理的场景（如连接池、并发组、文件句柄等），避免手动管理协程间的资源释放和同步逻辑。


**底层实现**：`cobalt::with(wait_group, fn)` 在进入 fn 前创建 `wait_group`，退出时 `co_await` 等待组内所有协程完成。`wait_group` 内部用 Asio 的 completion handler 链或条件变量跟踪协程数量，无额外 POSIX 调用。

```cpp
cobalt::promise<void> run_with_resource(cobalt::wait_group& wg)
{
    wg.push_back(worker1());
    wg.push_back(worker2());
    co_await wg.wait_one();  // 等任意一个完成
    co_return;
}

cobalt::main co_main(int argc, char* argv[])
{
    co_await cobalt::with(
        cobalt::wait_group(asio::cancellation_type::all, asio::cancellation_type::all),
        &run_with_resource);
    co_return 0;
}
```

### 6.2 wait_group 限制并发数

```cpp
cobalt::promise<void> run_server(cobalt::wait_group& workers)
{
    auto listen = listen_generator();
    while (true) {
        if (workers.size() >= 10)
            co_await workers.wait_one();
        workers.push_back(handle_client(co_await listen));
    }
}
```

---

## 七、网络 I/O

### 7.1 Echo Server（Cobalt + use_op）

**底层实现**：`tcp::socket` / `tcp::acceptor` 的 `async_read_some`、`async_accept` 等将操作注册到 Asio reactor。Linux 上 reactor 使用 `epoll`（epoll_ctl 注册 fd，epoll_wait 等待），就绪后通过 completion handler 恢复协程。`cobalt::use_op` 将 Asio 的 async 操作适配为 C++20 awaitable。

```cpp
namespace cobalt = boost::cobalt;
using tcp = asio::ip::tcp;
using tcp_socket = cobalt::use_op_t::as_default_on_t<tcp::socket>;
using tcp_acceptor = cobalt::use_op_t::as_default_on_t<tcp::acceptor>;

cobalt::promise<void> echo(tcp_socket socket)
{
    try {
        char buf[4096];
        while (socket.is_open()) {
            std::size_t n = co_await socket.async_read_some(asio::buffer(buf));
            co_await asio::async_write(socket, asio::buffer(buf, n));
        }
    } catch (std::exception& e) {
        std::printf("echo: %s\n", e.what());
    }
}

cobalt::generator<tcp_socket> listen()
{
    tcp_acceptor acc{co_await cobalt::this_coro::executor, {tcp::v4(), 55555}};
    for (;;) {
        tcp_socket sock = co_await acc.async_accept();
        co_yield std::move(sock);
    }
}

cobalt::main co_main(int argc, char* argv[])
{
    co_await cobalt::with(cobalt::wait_group(), [](cobalt::wait_group& wg) -> cobalt::promise<void> {
        auto l = listen();
        while (true) {
            if (wg.size() >= 10)
                co_await wg.wait_one();
            wg.push_back(echo(co_await l));
        }
    });
    co_return 0;
}
```

### 7.2 纯 Asio awaitable + co_spawn

**底层实现**：`asio::awaitable<T>` 是 Asio 自带的协程类型，`co_spawn` 将协程 `post` 到 `io_context`。I/O 与 Cobalt 相同，均通过 reactor：Linux epoll、BSD/macOS kqueue、Windows IOCP。`asio::use_awaitable` 是 Asio 的 completion token，将 async 操作转为 awaitable。

```cpp
asio::awaitable<void> echo_asio(tcp::socket socket)
{
    try {
        char data[1024];
        for (;;) {
            std::size_t n = co_await socket.async_read_some(
                asio::buffer(data), asio::use_awaitable);
            co_await asio::async_write(socket, asio::buffer(data, n), asio::use_awaitable);
        }
    } catch (std::exception& e) {
        std::printf("echo: %s\n", e.what());
    }
}

int main()
{
    asio::io_context ctx;
    asio::ip::tcp::acceptor acc{ctx, {tcp::v4(), 55555}};

    asio::co_spawn(ctx, [&]() -> asio::awaitable<void> {
        for (;;) {
            auto sock = co_await acc.async_accept(asio::use_awaitable);
            asio::co_spawn(ctx, echo_asio(std::move(sock)), asio::detached);
        }
    }, asio::detached);

    ctx.run();
    return 0;
}
```

---

## 八、定时器

### 8.1 简单延迟

**底层实现**：`asio::steady_timer` 在 Linux 上通常用 `timerfd_create` + `timerfd_settime` 创建可等待的 fd，并注册到 epoll；BSD/macOS 用 kqueue 的 EVFILT_TIMER。`async_wait` 将 completion handler 挂到 reactor，超时后 reactor 唤醒并恢复协程。

```cpp
cobalt::main co_main(int argc, char* argv[])
{
    asio::steady_timer tim{co_await cobalt::this_coro::executor,
                           std::chrono::milliseconds(100)};
    co_await tim.async_wait(cobalt::use_op);
    std::cout << "done" << std::endl;
    co_return 0;
}
```

### 8.2 race 实现超时

```cpp
cobalt::promise<std::string> fetch_with_timeout()
{
    cobalt::promise<std::string> do_fetch();  // 实际请求

    asio::steady_timer tim{co_await cobalt::this_coro::executor,
                           std::chrono::seconds(5)};
    auto v = co_await cobalt::race(
        do_fetch(),
        tim.async_wait(cobalt::use_op));
    if (v.index() == 0)
        co_return std::get<0>(v);
    throw std::runtime_error("timeout");
}
```

---

## 九、取消机制

### 9.1 检查取消状态

```cpp
cobalt::promise<void> cancellable_loop()
{
    while (!co_await cobalt::this_coro::cancelled) {
        co_await do_work();
    }
    co_return;
}
```

### 9.2 co_main 自动信号处理

`co_main` 会自动连接 SIGINT/SIGTERM 到取消，Ctrl+C 会向协程发出取消信号。

---

## 十、多线程与 Strand

### 10.1 单 io_context 多线程

**底层实现**：多个线程共享同一 `io_context::run()`，reactor（Linux epoll、BSD kqueue）的 `epoll_wait`/`kevent` 在任意调用线程中执行，就绪的 fd 由该线程派发 completion handler。需注意数据竞争，可用 `asio::strand` 串行化。

```cpp
int main()
{
    asio::io_context ctx;
    asio::ip::tcp::acceptor acc{ctx, {tcp::v4(), 55555}};

    asio::co_spawn(ctx, accept_loop(acc), asio::detached);

    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i)
        threads.emplace_back([&] { ctx.run(); });
    for (auto& t : threads)
        t.join();
    return 0;
}
```

### 10.2 使用 Strand 串行化

```cpp
// 每个连接绑定到独立 strand，保证单连接内串行
asio::co_spawn(ctx, [&]() -> asio::awaitable<void> {
    for (;;) {
        auto sock = co_await acc.async_accept(asio::use_awaitable);
        auto strand = asio::make_strand(ctx);
        asio::co_spawn(strand, echo_asio(std::move(sock)), asio::detached);
    }
}, asio::detached);
```

### 10.3 Cobalt + 手动指定执行器（Strand）

```cpp
// channel、promise 等需在构造/调用时指定 executor
cobalt::promise<void> example(int x, asio::executor_arg_t, cobalt::executor ex);

// 调用时
auto strand = asio::make_strand(ctx);
example(42, asio::executor_arg, strand);
```

---

## 十一、Asio 与 Cobalt 互操作

### 11.1 Cobalt 中 co_await Asio 操作

```cpp
// 使用 cobalt::use_op 作为 completion token
co_await socket.async_read_some(asio::buffer(buf), cobalt::use_op);
co_await timer.async_wait(cobalt::use_op);
```

### 11.2 Asio awaitable 与 Cobalt 互等

```cpp
// Cobalt promise 可 co_await asio::awaitable（需适配）
// Asio 侧可用 use_awaitable，Cobalt 侧用 use_op
```

---

## 十二、磁盘 I/O 与线程池

### 12.1 epoll 不适用于普通文件

socket/pipe 可用 epoll；普通文件需用线程池：

```cpp
asio::thread_pool pool(4);

// 思路：asio::post(pool, [&] {
//   完成阻塞读后，post(executor, handler) 恢复协程
// });
// 参考 Boost 示例 thread_pool.cpp、thread.cpp 中的 concurrent_channel 用法
```

---

## 十三、多 io_context 池（无竞争）

```cpp
// 每线程一个 io_context，无共享队列，适合高并发连接
std::vector<std::thread> threads;
std::vector<std::shared_ptr<asio::io_context>> contexts(4);

for (size_t i = 0; i < 4; ++i) {
    contexts[i] = std::make_shared<asio::io_context>();
    threads.emplace_back([ctx = contexts[i]] { ctx->run(); });
}

// 连接按策略分配到不同 io_context
asio::io_context& pick_context(size_t conn_id) {
    return *contexts[conn_id % contexts.size()];
}
```

---

## 十四、场景速查表

| 场景 | Asio | Cobalt |
|------|------|--------|
| 入口 | `co_spawn` + `ctx.run()` | `co_main` / `cobalt::run` / `spawn` |
| 单结果协程 | `awaitable<T>` | `promise<T>`（急切）/ `task<T>`（惰性） |
| 多值 | 无内置 | `generator<T>` |
| 并发等待全部 | 手动 `when_all` 等 | `join` |
| 并发等任一 | 手动 | `race` / `left_race` |
| 各捕获异常 | 手动 | `gather` |
| 协程间通信 | `experimental::channel` | `channel<T>` |
| 资源/作用域 | 手动 | `with` + `wait_group` |
| 取消 | `cancellation_signal` | `this_coro::cancelled`，`race` 超时 |
| 多线程 | `io_context::run` 多线程 / strand | 同 Asio，注意 executor 传递 |

---

## 十五、完整 Echo Server 示例（Cobalt）

```cpp
#include <boost/asio.hpp>
#include <boost/cobalt.hpp>
#include <boost/cobalt/main.hpp>
#include <boost/cobalt/promise.hpp>
#include <boost/cobalt/generator.hpp>
#include <boost/cobalt/with.hpp>
#include <boost/cobalt/wait_group.hpp>
#include <iostream>

namespace asio = boost::asio;
namespace cobalt = boost::cobalt;
using tcp = asio::ip::tcp;
using tcp_socket = cobalt::use_op_t::as_default_on_t<tcp::socket>;
using tcp_acceptor = cobalt::use_op_t::as_default_on_t<tcp::acceptor>;

cobalt::promise<void> echo(tcp_socket socket)
{
    try {
        char buf[4096];
        while (socket.is_open()) {
            auto n = co_await socket.async_read_some(asio::buffer(buf));
            co_await asio::async_write(socket, asio::buffer(buf, n));
        }
    } catch (std::exception& e) {
        std::printf("echo: %s\n", e.what());
    }
}

cobalt::generator<tcp_socket> listen()
{
    tcp_acceptor acc{co_await cobalt::this_coro::executor, {tcp::v4(), 55555}};
    for (;;) {
        tcp_socket sock = co_await acc.async_accept();
        co_yield std::move(sock);
    }
}

cobalt::promise<void> run_server(cobalt::wait_group& workers)
{
    auto l = listen();
    while (true) {
        if (workers.size() >= 10)
            co_await workers.wait_one();
        workers.push_back(echo(co_await l));
    }
}

cobalt::main co_main(int argc, char** argv)
{
    std::cout << "Echo server on :55555" << std::endl;
    co_await cobalt::with(
        cobalt::wait_group(asio::cancellation_type::all, asio::cancellation_type::all),
        &run_server);
    co_return 0;
}
```

---

*文档参考 C++协程技术总结.md 与 Boost.Cobalt 官方文档整理。*

[src: raw/ingested/2技术/cpp/并行库-Asio-Cobalt协程场景示例.md]

## Related Pages
- [[Boost.Asio]]
- [[C++20协程]]
- [[C++网络编程]]
- [[Linux_IO]]
- [[C++多线程与并发]]
