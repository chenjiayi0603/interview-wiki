# C++协程库对比：Folly / Asio+Cobalt / libgo

> 综合对比 Folly::coro、Boost.Asio + Boost.Cobalt、libgo 三套 C++ 协程方案，涵盖性能、同步原语、线程模型、调度算法、I/O 挂起/恢复等维度。

See also: [[C++20协程]], [[Boost.Asio]], [[Asio-Cobalt协程场景示例]], [[C++多线程与并发]]

---

## 一、性能与场景对比

| 维度 | Folly::coro | Asio / Cobalt | libgo |
|------|-------------|---------------|-------|
| 协程模型 | C++20 无栈 | C++20 无栈 | 有栈 |
| 单协程内存 | 几百 B ~ 几 KB | 几百 B ~ 几 KB | 4 KB ~ 128 KB |
| 切换开销 | ~几十 ns（无栈） | ~几十 ns（无栈） | ~197 ns (boost) |
| 创建开销 | 堆分配 | 堆分配 | 栈分配，可栈池优化 |
| 网络延迟 | 通用 | 微秒级优化 | 通用 |
| 最佳场景 | 通用异步、大数据 | 网络服务、RPC | 高并发 I/O、类 Go 开发 |

---

## 二、同步原语：内部结构与效率

| 原语 | Folly | Asio/Cobalt | libgo |
|------|-------|-------------|-------|
| **内部结构** | Baton：条件变量+锁；Mutex：锁+等待队列 | channel：单线程队列，无锁；race：多 awaitable | Channel：deque+mutex+3 个 CV；WaitGroup：原子+sema |
| **锁竞争** | 有锁 | channel 无锁 | Channel 有锁 |
| **单次唤醒** | ~100 ns–几 μs | ~几十 ns | ~100 ns–几 μs |

---

## 三、单线程 vs 多线程支持

| 维度 | Folly::coro | Asio / Cobalt | libgo |
|------|-------------|---------------|-------|
| 默认模型 | 多线程（Executor） | 单线程或多线程可选 | 多线程（M:N） |
| 单线程支持 | ✅ 可配置 | ✅ 单线程 run() | ⚠️ 可配置 |
| 多线程支持 | ✅ 原生 | ✅ 多线程 run + strand | ✅ 原生 |
| 同步原语 | Baton/Mutex 跨线程 | channel 单线程内 | Channel/WaitGroup 跨线程 |

### Asio/Cobalt 单多线程配置

- **单线程**：仅一个线程调用 `io_context::run()`
- **多线程**：多线程同时调用 `io_context::run()`
- **多线程 + strand**：关键操作绑定 strand，保证串行
- **channel 限制**：Cobalt channel 为线程局部，跨线程使用需在构造时指定 strand

---

## 四、M:N 与调度算法

| 项目 | 含义 |
|------|------|
| **M** | 用户态线程（协程）数量 |
| **N** | 内核态线程（OS 线程）数量 |

| 库 | 调度算法 | 说明 |
|------|----------|------|
| **libgo** | 工作窃取（Work Stealing） | 每 P 本地队列+全局队列，空时从其他 P 窃取 |
| **Asio/Cobalt** | 事件驱动（Proactor） | epoll/kqueue 等待 I/O，完成 handler 入队，线程取任务执行 |
| **Folly** | Executor 线程池 | 共享任务队列（UMPMCQueue），支持优先级，无窃取 |

---

## 五、任务获取/派发规则详解

### libgo（工作窃取）

**任务获取顺序**（M 从队列取可执行协程 G 时）：
1. **runnext**：优先检查当前 P 的 runnext（单个高优先级 G）
2. **本地队列 LRQ**：从当前 P 的本地就绪队列取
3. **全局队列 GRQ**：每 61 次调度循环检查一次全局队列
4. **窃取**：从其他 P 的 LRQ 尾部窃取约一半的 G
5. **Netpoller**：检查是否有 I/O 就绪的 G 可唤醒

**任务派发**：新创建的 G 优先放入当前 P 的 LRQ；若 LRQ 满则放入全局队列。

### Asio/Cobalt（事件驱动）

**任务获取**：
1. **epoll_wait / kqueue / GetQueuedCompletionStatus**：阻塞等待 I/O 完成事件
2. **完成事件**：I/O 完成 → 将对应 completion handler 入队到 `io_context` 内部队列
3. **取任务**：调用 `run()` 的线程从队列头部取 handler 执行
4. **多线程**：多线程共享同一队列，竞争取任务（多消费者）

**任务派发**：`co_spawn`、`post`、`defer` 等将 handler 入队；`strand` 保证同一 strand 上的 handler 串行执行。

### Folly（Executor 线程池）

**任务获取**：
1. **CPUThreadPoolExecutor**：工作线程从共享 UMPMCQueue 取任务；支持多优先级时先查高优先级队列
2. **LifoSem**：LIFO 唤醒，提高缓存局部性
3. **IOThreadPoolExecutor**：每线程独立 EventBase，粘性分配（同一调用线程倾向同一工作线程）

**任务派发**：`add()` 将任务入队；`scheduleOn(executor)` 将协程投递到指定 Executor 的队列；无窃取，负载均衡依赖共享队列。

---

## 六、协程作用与 I/O 挂起/恢复

### 协程 vs 任务队列

| 层面 | 任务队列（Executor） | 协程 |
|------|----------------------|------|
| **作用** | 调度：任务在哪个线程、何时执行 | 编程模型：如何写异步逻辑 |
| **关系** | 协程产生的任务入队 | 协程是任务的来源之一 |

协程解决**怎么写**异步代码，任务队列解决**在哪跑**。无窃取只影响调度策略，不影响是否用协程。

### 协程的核心作用

- **挂起与恢复**：`co_await` 时挂起不占线程，条件满足后恢复
- **顺序式写法**：异步流程写成顺序代码，替代回调嵌套
- **状态保存**：挂起时自动保存局部变量和恢复点

### 协程 vs 线程优势

| 维度 | 线程 | 协程 |
|------|------|------|
| 切换开销 | ~1–10 μs（内核态） | ~几十–几百 ns（用户态） |
| 内存占用 | ~1–8 MB/线程 | 几百 B–几 KB（无栈）或 4–128 KB（有栈） |
| 并发数量 | 几百–几千 | 可达百万级 |
| 适用场景 | CPU 密集 | I/O 密集 |

### I/O 挂起/恢复：三库均支持

| 库 | 实现方式 | 挂起时机 | 恢复时机 |
|------|----------|----------|----------|
| **Folly::coro** | 异步 API + 事件循环 | `co_await` 异步操作时 | I/O 完成，handler 触发 |
| **Asio/Cobalt** | Proactor 异步 I/O | `co_await async_read` 时 | epoll 返回，completion handler 执行 |
| **libgo** | Hook 阻塞系统调用 | 调用 `read`/`write` 等被 Hook 时 | epoll 就绪，调度器恢复 |

**Asio/Cobalt**：显式异步，必须用 `co_await async_xxx`。**libgo**：可写同步式 `read`/`write`，由 Hook 自动挂起/恢复。

[src: raw/ingested/2技术/cpp/C++协程技术总结-八、Folly---Asio+Cobalt---libgo-综合对比.md]

## Related Pages
- [[C++20协程]]
- [[Boost.Asio]]
- [[Asio-Cobalt协程场景示例]]
- [[C++多线程与并发]]
- [[C++网络编程]]
