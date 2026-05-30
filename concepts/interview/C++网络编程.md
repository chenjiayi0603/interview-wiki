# C++网络编程

See also: [[性能优化]]

## 阻塞/非阻塞 I/O
- 阻塞 I/O：`read` / `write` / `accept` / `connect` 阻塞等待
- 非阻塞 I/O：`O_NONBLOCK` 设置，返回 `EAGAIN` 表示无数据
- I/O 多路复用：`select` / `poll` / `epoll`

## Reactor 模型
- 核心组件：EventLoop、Channel、Poller
- 工作流程：注册事件 → 等待事件 → 分发处理
- 常见实现：libevent、muduo、evpp

## Libev 框架
- 事件循环：`ev_loop`
- 监视器：`ev_io`（I/O）、`ev_timer`（定时器）、`ev_signal`（信号）
- 回调机制

[src: raw/ingested/2技术/cpp/C++核心知识.md]

## Related Pages
- [[性能优化]]
- [[TCP协议]]
- [[Boost.Asio]]
- [[Linux_IO]]
- [[C++多线程与并发]]
