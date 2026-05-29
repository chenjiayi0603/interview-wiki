# Libev C++ 异步网络框架（事件驱动 + 多进程 Worker）

**简历原文**：基于 Libev 自研 C++ 异步网络框架（事件驱动 + 多进程 Worker）

## 数据支撑

| 组件 | 设计 | 效果 |
|------|------|------|
| 事件循环 | Libev（epoll LT 模式） | 空闲连接近零 CPU 唤醒成本 |
| 多进程 Worker | SO_REUSEPORT | 内核负载均衡，避免 accept 锁竞争 |
| 进程数 | CPU 核数（8 核 → 8 Worker） | 利用多核，无线程上下文切换开销 |
| 单跳内部转发延迟 | <5ms | gRPC 同 DC 1–3ms |
| 服务端处理 P99 | <50ms | 不含客户端网络 RTT |

## 理论支撑

Libev 基于 epoll 的 Level-Triggered（LT）模式：只要 socket buffer 有数据就持续触发，编程模型简单，不需要 while-EAGAIN 循环也不会丢事件（相比 ET 模式更安全，代价是多一次系统调用）。

多进程 + SO_REUSEPORT：Linux 3.9+ 内核支持多个进程绑定同一端口，内核按 4 元组哈希均匀分发新连接到各 Worker 进程。相比单进程 accept + 线程池，避免了 accept 队列锁竞争和惊群问题（参考 Nginx 多进程模型）。

## 业界对标

- **Nginx**：同样多进程 + SO_REUSEPORT + epoll LT，是百万连接网络框架的最直接业界参考
- **Memcached**：多线程 epoll 模型，思路类似（多进程改为多线程）
- **WhatsApp Ejabberd**：Erlang BEAM 多进程调度，单机 2M 连接的标杆案例（2012 年）

## 追问预案

**Q：Libev 用的是 ET 还是 LT？**
> Libev 默认使用 LT（Level-Triggered）。LT 模式编程模型简单，socket buffer 有数据就持续触发，不需要 while-EAGAIN 循环。我们选 LT 是因为 IM 消息有严格的保序要求，ET 模式必须在单次触发时读完所有数据，实现复杂度更高，bug 风险更大。LT 的代价是多一次 epoll_wait 返回，对比节省的编程复杂度完全值得。

[src: raw/ingested/3项目/面试准备/简历知识点论证手册-L1：Libev-C++-异步网络框架（事件驱动-+-多进程-Worker）.md]