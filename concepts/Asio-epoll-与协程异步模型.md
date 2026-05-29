# Asio、epoll 与协程异步模型

> 本文档涵盖 Asio 的 Strand、多 io_context 池、epoll 与协程的协作机制、阻塞类型处理、co_await 生产者/消费者模型等核心概念。

See also: [[Boost.Asio]], [[C++20协程]], [[Asio-Cobalt协程场景示例]], [[C++多线程与并发]]

---

## 一、Strand

| 项目 | 说明 |
|------|------|
| **定义** | Asio 的串行执行器，保证同一 strand 上的 handler 不并发 |
| **调度** | FIFO 队列，按入队顺序依次执行 |
| **用途** | 替代显式锁，保护共享状态 |

---

## 二、多 io_context 池

| 项目 | 说明 |
|------|------|
| **结构** | 每线程一个 io_context + 独立 epoll，连接按策略分配 |
| **效率高** | 无共享队列，无跨线程竞争，无锁 |
| **多核好** | 可绑核，负载分散 |
| **代价** | 跨连接共享状态需额外同步 |

---

## 三、多线程四种方式对比

| 方式 | 并发 | 效率 | 多核 | 延迟 | 适用 |
|------|------|------|------|------|------|
| ① 单 io_context 多线程 | 高 | 中（队列竞争） | 好 | 中 | 通用 |
| ② 多 io_context 池 | 高 | 高（无竞争） | 好 | 低 | 高并发连接 |
| ③ strand 串行化 | 低 | 低（串行） | 差 | 高 | 共享状态多 |
| ④ 每线程独立+消息 | 中 | 高 | 好 | 低 | 分区明确 |

---

## 四、io_context、epoll 与协程

| 组件 | 关系 |
|------|------|
| **io_context** | 每实例一个 epoll fd，独立事件循环 |
| **epoll** | 监听 fd 就绪，I/O 完成时触发 handler |
| **协程** | `co_await` 挂起，handler 执行时 `handle.resume()` 恢复 |
| **流程** | epoll 就绪 → handler 入队 → 执行 handler → 恢复协程 |

---

## 五、阻塞类型与 epoll

| 类型 | epoll 适用 | 处理方式 |
|------|------------|----------|
| socket/pipe | ✅ | epoll |
| 普通文件（磁盘） | ❌ | 线程池 |
| sleep | ❌（可用 timerfd） | timerfd + epoll |
| mutex | ❌ | 异步 mutex，挂起+post 恢复 |

---

## 六、epoll 无法解决的阻塞：协程让出

| 阻塞类型 | 让出方式 | 恢复方式 |
|----------|----------|----------|
| 磁盘 I/O | `co_await` 文件读 awaitable | 线程池完成后 post 恢复 |
| sleep | `co_await` timer.async_wait | timerfd 到时，epoll 触发 |
| 锁 | `co_await` mutex.async_lock | unlock 时 post 恢复 |

---

## 七、co_await 与 API 职责

| 问题 | 答案 |
|------|------|
| co_await 会注册 epoll？ | 否，是 async API（如 async_read_some）内部注册 |
| 非 I/O 会阻塞？ | 设计正确则否，阻塞在 worker 线程 |
| C++20 协程作用？ | 提供挂起/恢复机制，不负责 epoll、线程池 |
| epoll vs 线程池谁决定？ | 各 async API 的实现决定，co_await 不判断 |

---

## 八、co_await 生产者/消费者模型

| 角色 | 谁 | 职责 |
|------|-----|------|
| **生产者** | 协程（调用 co_await） | 发起异步操作，挂起，产生「等待请求」 |
| **消费者** | epoll 处理 或 worker 线程 | 完成操作，触发 completion handler，恢复协程 |

| 场景 | 消费者 | 说明 |
|------|--------|------|
| 网络 I/O、定时器 | epoll 处理 | epoll 检测就绪 → 事件循环执行 handler |
| 磁盘 I/O、CPU 密集 | worker 线程 | worker 完成 → post(handler) → 事件循环执行 |

---

## 九、异步 API 提供方

| 类型 | 提供方 |
|------|--------|
| 网络 I/O、定时器 | Boost.Asio |
| HTTP、WebSocket | Boost.Beast |
| MySQL、Redis | Boost.MySQL、Boost.Redis |
| 文件异步读 | 应用用 post+线程池封装 |

[src: raw/ingested/2技术/cpp/C++协程技术总结-十一、Asio、epoll-与协程异步模型.md]

## Related Pages
- [[Boost.Asio]]
- [[C++20协程]]
- [[Asio-Cobalt协程场景示例]]
- [[C++多线程与并发]]
- [[C++网络编程]]
