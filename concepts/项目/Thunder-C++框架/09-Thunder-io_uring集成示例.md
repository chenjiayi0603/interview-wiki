# Thunder IO 原理 — io_uring + libev + Asio

> 代码可编译运行，注释用大白话。

---

```cpp
// asio_uring_libev_demo.cpp
// 编译：g++ -std=c++20 -o demo asio_uring_libev_demo.cpp -lev -lpthread
// 运行：echo "hello" | ./demo
//
// 这个例子做什么：
//   从标准输入读数据，读到后打印出来，3 秒后退出。
//
// 用了三个东西：
//   1. io_uring  — 内核的"帮我读完叫我"机制，不用自己 read()
//   2. Asio      — 把 io_uring 封装成好用的 C++ 接口
//   3. libev     — 事件循环，负责"等"（等数据、等定时器、等信号）
//
// 核心思路：
//   Asio 内部有个 io_uring，io_uring 有一个 ring_fd。
//   这个 ring_fd 在 I/O 完成时会变成"可读"。
//   libev 监视这个 ring_fd，可读时就去 Asio 里拿结果。
//   libev 也监视定时器、信号等，所有事情一个线程搞定。

#include <ev.h>
#include <asio.hpp>
#include <asio/posix/stream_descriptor.hpp>
#include <cstdio>
#include <cstring>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <cstdlib>

// ─── 找 ring_fd ───
// 先搞清楚三个东西：
//
//   SQE（Submission Queue Entry）：
//     你写给 io_uring 的"订单"。上面写着：
//     "帮我从 fd=X 读数据，读到 buf 里，读 len 字节"
//     填 SQE = 下单，io_uring_submit = 把订单给到内核。
//
//   CQE（Completion Queue Entry）：
//     你让 io_uring 帮你读数据，它读完了给你一张"回执"，
//     上面写着"读了多少字节"或"出错了"。这张回执就叫 CQE。
//
//   ring_fd：
//     io_uring 实例有一个文件描述符。当有 CQE（回执）可以领取时，
//     这个 fd 会变成"可读"。它就像一个门铃——有结果了，叮咚。
//
// 我们要找到这个 ring_fd，让 libev 监视它。
// ring_fd 一响（可读），libev 就去 Asio 里拿 CQE（读结果）。
// 找法：扫描 /proc/self/fd/，看哪个 fd 的名字是 "[io_uring]"。
int FindIoUringRingFd() {
    DIR* dir = opendir("/proc/self/fd");
    if (!dir) return -1;
    struct dirent* e;
    int found = -1;
    while ((e = readdir(dir)) != nullptr) {
        char src[64], dst[256];
        snprintf(src, sizeof(src), "/proc/self/fd/%s", e->d_name);
        ssize_t n = readlink(src, dst, sizeof(dst) - 1);
        if (n > 0) {
            dst[n] = '\0';
            if (strstr(dst, "[io_uring]")) {
                found = atoi(e->d_name);
                break;
            }
        }
    }
    closedir(dir);
    return found;
}

// ─── 全局数据 ───
struct DemoContext {
    asio::io_context io_ctx{1};                            // Asio（内部用 io_uring）
    asio::posix::stream_descriptor sock{io_ctx, STDIN_FILENO};  // 包装标准输入
    char buf[4096];
    int total_read = 0;
    int ring_fd = -1;

    // libev 的四个监视器
    struct ev_prepare prepare_w;    // epoll_wait 前调用
    struct ev_check   check_w;      // epoll_wait 后调用
    struct ev_io      ring_watcher; // ring_fd 可读时调用
    struct ev_timer   timer_w;      // 超时调用
};

// ─── 三个钩子函数（每次循环都会跑一遍） ──────
// io_context.poll() 的意思：去 io_uring 看看有没有完成的 I/O，
//   有就读出来，传给对应的回调。没有就立即返回，不等。
//   libev 负责等（epoll_wait），poll 只负责读结果。

// 1. epoll_wait 前：先看看有没有已经完成的
static void OnPrepare(struct ev_loop*, struct ev_prepare* w, int) {
    auto* ctx = (DemoContext*)w->data;
    ctx->io_ctx.poll();   // 读已完成的事件
}

// 2. epoll_wait 后：看看等待期间有没有新完成的
static void OnCheck(struct ev_loop*, struct ev_check* w, int) {
    auto* ctx = (DemoContext*)w->data;
    ctx->io_ctx.poll();   // 读新完成的事件
}

// 3. ring_fd 可读：io_uring 有结果了
static void OnRingReady(struct ev_loop*, struct ev_io* w, int) {
    auto* ctx = (DemoContext*)w->data;
    ctx->io_ctx.poll();   // 读完成事件
}

// 4. 超时：3 秒到了，退出
static void OnTimeout(struct ev_loop* loop, struct ev_timer* w, int) {
    auto* ctx = (DemoContext*)w->data;
    printf("--- 3秒到，共读了 %d 字节 ---\n", ctx->total_read);
    ev_break(loop, EVBREAK_ALL);
}

// ─── 入口 ──────────────────────────────────
int main() {
    DemoContext ctx;

    // ---- 准备 ----

    // 让 Asio 内部创建 io_uring（通过一次 pipe 操作触发）
    int pfd[2];
    pipe2(pfd, O_NONBLOCK | O_CLOEXEC);
    {
        asio::posix::stream_descriptor tmp(ctx.io_ctx, pfd[0]);
        (void)tmp.release();
    }
    ::close(pfd[0]); ::close(pfd[1]);
    ctx.io_ctx.poll();    // 触发 io_uring 创建

    // 找到 ring_fd
    ctx.ring_fd = FindIoUringRingFd();
    if (ctx.ring_fd < 0) {
        fprintf(stderr, "没找到 io_uring（内核版本低于 5.1？）\n");
        return 1;
    }
    printf("找到 ring_fd = %d\n", ctx.ring_fd);

    // 标准输入设为非阻塞
    int flags = fcntl(STDIN_FILENO, F_GETFL, 0);
    fcntl(STDIN_FILENO, F_SETFL, flags | O_NONBLOCK);

    // ---- 注册 libev 监视器 ----
    struct ev_loop* loop = EV_DEFAULT;

    // epoll_wait 前调用
    ev_prepare_init(&ctx.prepare_w, OnPrepare);
    ctx.prepare_w.data = &ctx;
    ev_prepare_start(loop, &ctx.prepare_w);

    // epoll_wait 后调用
    ev_check_init(&ctx.check_w, OnCheck);
    ctx.check_w.data = &ctx;
    ev_check_start(loop, &ctx.check_w);

    // ring_fd 可读时调用
    ev_io_init(&ctx.ring_watcher, OnRingReady, ctx.ring_fd, EV_READ);
    ctx.ring_watcher.data = &ctx;
    ev_io_start(loop, &ctx.ring_watcher);

    // 3 秒超时
    ev_timer_init(&ctx.timer_w, OnTimeout, 3.0, 0.0);
    ctx.timer_w.data = &ctx;
    ev_timer_start(loop, &ctx.timer_w);

    // ---- 发起异步读 ----
    // 告诉 Asio：帮我从标准输入读数据，读完调用 do_read
    std::function<void(asio::error_code, size_t)> do_read;
    do_read = [&ctx, &do_read](asio::error_code ec, size_t n) {
        if (ec) { printf("读出错: %d\n", ec.value()); return; }
        if (n == 0) { printf("读完了\n"); return; }
        ctx.total_read += (int)n;
        ctx.buf[n] = '\0';
        printf("读到 %zu 字节: %s", n, ctx.buf);
        // 继续读（keep-alive 语义，像 HTTP 长连接一样）
        ctx.sock.async_read_some(
            asio::buffer(ctx.buf, sizeof(ctx.buf)), do_read);
    };
    ctx.sock.async_read_some(asio::buffer(ctx.buf, sizeof(ctx.buf)), do_read);

    // ---- 启动 libev 事件循环（不是 io.run()！） ----
    printf("进入 libev 循环...\n");
    ev_run(loop, 0);

    // ---- 清理 ----
    ev_prepare_stop(loop, &ctx.prepare_w);
    ev_check_stop(loop, &ctx.check_w);
    ev_io_stop(loop, &ctx.ring_watcher);
    ev_timer_stop(loop, &ctx.timer_w);

    return 0;
}
```

---

## 编译与运行

```bash
sudo apt install libev-dev libasio-dev
g++ -std=c++20 -o demo asio_uring_libev_demo.cpp -lev -lpthread
echo "hello thunder" | ./demo
```

---

## 到底在干什么

```
一次完整的读流程：

①  async_read_some(fd, buf, callback)
    告诉 Asio："帮我读这个 fd"
    → Asio 跟 io_uring 说："帮我读这个 fd"
    → io_uring 在本子上记一笔："要读这个 fd"（记的叫 SQE）

②  内核自己读数据到 buf（不占用户 CPU）

③  读完了，io_uring 写一张回执："读完了，读了 XX 字节"
    （这张回执叫 CQE），然后按门铃（ring_fd 变成可读）

④  libev 听到门铃响（epoll_wait 发现 ring_fd 可读）
    → 调 OnRingReady → io_context.poll()
    → poll 去拿回执（CQE），看看读了多少
    → 调 do_read 回调
    → 打印数据

⑤  do_read 里又调 async_read_some，继续读下一个
```

**libev 一次循环**：

```
① ev_prepare  → io_context.poll()   看看有没有已经完成的结果
② epoll_wait  → 阻塞等待（等 ring_fd 或定时器）
③ ev_io(ring_fd) → io_context.poll()  ring_fd 可读，拿结果
④ ev_check    → io_context.poll()   再看看有没有漏的
```

**io_uring 三件套**：

| 概念 | 类比 | 一句话 |
|------|------|--------|
| **SQE** (Submission Queue Entry) | 订单 | 告诉内核"帮我干啥"（填在提交队列里） |
| **CQE** (Completion Queue Entry) | 回执 | 内核告诉你"干完了，结果在这"（写在完成队列里） |
| **ring_fd** | 门铃 | 有回执了，叮咚（CQE 可取时变成可读） |

流程对应：`async_read_some` = 填 SQE + submit（下单），`poll` = 读 CQE（取回执）。

**三个东西各管什么**：

| 东西 | 管什么 | 大白话 |
|------|--------|--------|
| **io_uring** | 真正的读写 | 内核里的"帮我干完叫我" |
| **Asio** | 封装 io_uring | 让你写 `async_read_some` 就行，不用管 SQE/CQE |
| **libev** | 事件循环 | 负责"等"，等 ring_fd、等定时器、等信号 |

**没有 `io.run()`，没有独立线程，没有锁**。
