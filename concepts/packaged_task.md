# packaged_task

`std::packaged_task` 是 C++11 引入的异步任务包装器，它将一个可调用对象（函数、lambda、函数对象等）包装成一个异步任务，并关联一个 `std::future` 用于获取任务结果。

## 基本用法

```cpp
#include <future>
#include <iostream>
#include <thread>

int main() {
    std::packaged_task<int(int)> task([](int x){ return x * x; });
    std::future<int> result = task.get_future();
    std::thread t(std::move(task), 5);   // 5 作为参数传入，task 在新线程执行
    std::cout << result.get() << std::endl;  // 25
    t.join();
}
```

## 核心特点

- **任务与结果分离**：`packaged_task` 封装可调用对象，通过 `get_future()` 获取关联的 `std::future`，在任务执行完成后通过 `future.get()` 获取结果。
- **可移动不可复制**：`packaged_task` 只能移动（`std::move`），不能复制，因此可以安全地转移到线程中执行。
- **延迟执行**：任务在显式调用 `operator()` 或转移到线程中时才执行。
- **异常传播**：如果任务抛出异常，异常会被捕获并存储在 `future` 中，调用 `future.get()` 时会重新抛出。

## 与 std::async 的对比

| 特性 | `std::packaged_task` | `std::async` |
|------|---------------------|-------------|
| 执行控制 | 手动控制何时、在哪个线程执行 | 由系统决定（可能异步或延迟） |
| 线程管理 | 需手动创建线程或在线程池中执行 | 自动管理线程 |
| 灵活性 | 高，可放入任务队列、线程池 | 低，执行策略有限 |
| 适用场景 | 需要精确控制任务执行时机和线程 | 简单的异步任务 |

## 使用场景

- **线程池**：将 `packaged_task` 放入任务队列，由工作线程取出执行。
- **GUI 线程通信**：将耗时任务包装为 `packaged_task`，在后台线程执行，通过 `future` 获取结果更新 UI。
- **任务调度**：需要精确控制任务在特定线程或特定时机执行的场景。

## 相关面试要点

- `packaged_task` 与 `std::async`、`std::promise` 的区别和适用场景。
- `packaged_task` 的移动语义和线程安全。
- 如何基于 `packaged_task` 实现简单的线程池。

[src: raw/ingested/2技术/cpp/C++多线程完整手册-5.3-packaged_task.md]

## Related Pages
- [[C++多线程与并发]]
- [[现代C++特性按版本划分]]
- [[C++手写代码模板]]
