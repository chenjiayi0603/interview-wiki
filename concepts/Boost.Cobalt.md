# Boost.Cobalt

> Boost.Cobalt 是 Boost 1.84.0 引入的 C++20 协程库，由 Klemens Morgenstern 开发，提供 promise、task、generator、channel 等高层协程抽象，与 Boost.Asio 深度集成。

See also: [[C++20协程]], [[Boost.Asio]], [[Asio-Cobalt协程场景示例]], [[C++协程库对比-Folly-Asio-Cobalt-libgo]]

---

## 一、开发与维护

- **作者**：Klemens Morgenstern
- **维护**：Boost 组织（boostorg）
- **仓库**：https://github.com/boostorg/cobalt

[src: raw/ingested/2技术/cpp/C++协程技术总结-九、Boost.Cobalt-详情.md]

---

## 二、成熟度与生产可用性

| 项目 | 情况 |
|------|------|
| 纳入 Boost | 1.84.0（2023 年 12 月） |
| 版本 | 已到 1.90.0 |
| 状态 | 活跃维护，文档较完整 |
| 生产可用 | 可上线，但上线时间短，建议非核心或灰度验证 |

[src: raw/ingested/2技术/cpp/C++协程技术总结-九、Boost.Cobalt-详情.md]

---

## 三、应用场景

HTTP/WebSocket、RPC、代理、数据库客户端、游戏服务端、IoT、流式处理。知名项目多为 coral、coroutine_blog 等开源项目，Cobalt 商业案例较少。

[src: raw/ingested/2技术/cpp/C++协程技术总结-九、Boost.Cobalt-详情.md]

---

## 四、Cobalt 功能列表

| 类别 | 功能 | 说明 |
|------|------|------|
| 协程类型 | promise\<T\> | 急切执行，返回单个结果，推荐默认 |
| 协程类型 | task\<T\> | 惰性执行，可调度到不同执行器 |
| 协程类型 | generator\<T\> | 生成器，产生多个值 |
| 协程类型 | detached | 无句柄，fire-and-forget |
| 同步组合 | channel\<T\> | 协程间通信，类 Go channel |
| 同步组合 | race() | 等待多协程中任一完成，伪随机防饥饿 |
| 同步组合 | left_race() | 确定性版本，从左到右 |
| 同步组合 | join() | 等待所有协程并收集结果 |
| 同步组合 | gather() | 等待所有协程，分别处理异常 |
| 入口/取消 | co_main | 自动配置执行器、内存、信号处理 |
| 入口/取消 | 取消机制 | 结构化取消、Signal/Slot |
| 入口/取消 | with | 资源/作用域管理 |
| Asio 集成 | awaitable 互操作 | 可与 Asio awaitable 互相等待 |
| Asio 集成 | strand | 支持指定执行上下文 |
| Asio 集成 | Beast/MySQL/Redis | 与 Boost 网络库配合 |
| 内存/执行 | PMR | 多态内存资源，可自定义分配器 |
| 内存/执行 | 线程局部执行器 | 每线程默认执行器 |
| 内存/执行 | 信号处理 | SIGINT、SIGTERM 等 |

[src: raw/ingested/2技术/cpp/C++协程技术总结-九、Boost.Cobalt-详情.md]

---

## Related Pages
- [[C++20协程]]
- [[Boost.Asio]]
- [[Asio-Cobalt协程场景示例]]
- [[C++协程库对比-Folly-Asio-Cobalt-libgo]]
