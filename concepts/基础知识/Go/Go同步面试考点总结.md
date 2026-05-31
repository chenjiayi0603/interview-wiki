# Go 同步面试考点总结

## 15.1 核心知识点

1. **Goroutine 和线程的区别**
   - 内存占用、创建销毁、切换开销
   - GMP 调度模型
   - 抢占式调度（Go 1.14+）

2. **Channel 的特性**
   - 有缓冲 vs 无缓冲
   - 关闭 Channel 的规则
   - Select 的随机性

3. **Mutex 和 RWMutex**
   - 使用场景
   - 性能差异
   - 常见陷阱（死锁、忘记解锁）

4. **WaitGroup 的使用**
   - Add/Done/Wait 的正确使用
   - 指针传递的重要性

5. **Context 的作用**
   - 取消信号传递
   - 超时控制
   - 值传递

6. **原子操作**
   - Atomic vs Mutex
   - CAS 操作
   - Atomic.Value

## 15.2 常见面试题

1. **Goroutine 泄漏的原因和如何避免？**
2. **Channel 关闭后会发生什么？**
3. **Mutex 和 RWMutex 的性能差异？**
4. **如何实现一个线程安全的单例模式？**
5. **Select 的 default 分支的作用？**
6. **Context 的使用原则是什么？**
7. **如何避免死锁？**
8. **Atomic 和 Mutex 的选择标准？**

---

> **总结：**
> - Go 的同步机制丰富且易用，但需要正确理解和使用
> - 优先使用 Channel 进行通信，Mutex 保护共享资源
> - 注意避免常见的陷阱：Goroutine 泄漏、数据竞争、死锁
> - 根据场景选择合适的同步原语，平衡性能和复杂度

## 15.3 Go 的锁机制简述

Go 语言内置了多种锁机制，主要包括 `sync.Mutex`、`sync.RWMutex` 读写锁，还有更先进的原子操作（`sync/atomic`），用于在并发环境下保护共享资源、防止数据竞争。

### 1. Mutex（互斥锁）
- `sync.Mutex` 是最基本的互斥锁，能够保证同一时刻只允许一个 Goroutine 进入临界区。
- 常见用法：
    ```go
    var mu sync.Mutex

    mu.Lock()
    // 临界区，保护共享资源
    mu.Unlock()
    ```
- 推荐使用 `defer mu.Unlock()`，保证即使遇到 panic 或 return 也能正常释放锁，防止死锁。

### 2. RWMutex（读写锁）
- `sync.RWMutex` 允许多个读操作并行，但写操作时是独占的。
- 典型应用场景：读多写少，比如缓存、配置信息。
    ```go
    var rw sync.RWMutex
    rw.RLock()    // 获取读锁
    // 读操作
    rw.RUnlock()  // 释放读锁

    rw.Lock()     // 获取写锁
    // 写操作
    rw.Unlock()   // 释放写锁
    ```
- 读锁可并发，写锁互斥所有操作。

### 3. 锁的特性
- **零值可用**：Mutex 和 RWMutex 的零值就是未加锁状态，可以直接嵌入结构体使用，无需显示初始化。
- **不可复制**：Mutex 和 RWMutex 不允许被复制，只能传指针。

### 4. 原理简介
- Go 的锁机制由 runtime 支持，底层采用自旋、信号量、操作系统互斥原语等混合实现，效率高。
- Mutex 采用 "饥饿" 算法（优先让等待最久的 Goroutine 获得锁，减少饿死和公平性问题）。
- RWMutex 内部用计数和信号调度，防止写锁饿死，Go 1.9+ 优化了写锁公平性。

### 5. 注意事项与陷阱
- 避免重复加锁、忘记解锁（推荐 `defer mu.Unlock()`）
- 锁的粒度合适，不要锁过大（影响并发），也不要锁过小（增加开销）
- 非公共字段，确保锁与被保护的数据强绑定，防止外部误操作

### 6. 进阶：原子操作
- `sync/atomic` 提供原子级变量操作，适合简单计数等场景，性能优于锁，但功能有限，用于无锁增减：
    ```go
    import "sync/atomic"

    var count int64
    atomic.AddInt64(&count, 1)    // 原子递增
    n := atomic.LoadInt64(&count) // 原子读取
    ```

### 7. 选择建议
- **推荐优先级**：无共享（不需要同步） > Channel > 原子操作 > Mutex/RWMutex
- Channel 注重通信同步，Mutex 注重状态同步
- Channel 用于 Goroutine 间数据流动，Mutex 用于保护并发共享状态

---

**简而言之：Go 的锁机制核心简单易用，适合状态共享、多线程并发保护。实际开发中，按需选择最适合的同步原语，并严格注意死锁与性能问题。**

[src: raw/ingested/2技术/go/go同步-十五、面试考点总结.md]