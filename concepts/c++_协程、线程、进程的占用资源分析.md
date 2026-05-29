# C++ 协程、线程、进程的占用资源分析

## 1. 资源占用对比总览

| 维度 | 进程 (Process) | 线程 (Thread) | 协程 (Coroutine) |
|------|---------------|---------------|------------------|
| **内存开销** | 数MB~数十MB | 1MB~8MB（栈）| 几KB~几十KB |
| **创建时间** | ~10ms | ~1ms | ~1μs |
| **切换开销** | ~1ms（需内核态） | ~10μs（内核态） | ~100ns（用户态） |
| **调度方式** | 操作系统内核 | 操作系统内核 | 用户态调度器 |
| **并发数上限** | 数百~数千 | 数千~数万 | 数十万~数百万 |
| **隔离性** | 完全隔离 | 共享地址空间 | 共享线程栈 |

## 2. 进程（Process）资源分析

### 2.1 内存布局

```
┌─────────────────────────────────────┐  高地址
│           内核空间                   │  (用户不可访问)
├─────────────────────────────────────┤  0xFFFFFFFF (32位)
│              栈区                    │  ↓ 向下增长
│         (Stack)                     │
├─────────────────────────────────────┤
│              ↓                      │
│           (空闲)                    │
│              ↑                      │
├─────────────────────────────────────┤
│              堆区                    │  ↑ 向上增长
│         (Heap)                      │
├─────────────────────────────────────┤
│            BSS段                    │  未初始化全局变量
├─────────────────────────────────────┤
│         Data段（只读/可写）           │  • 只读数据段（.rodata）：常量、字符串字面量
│                                     │  • 数据段（.data）：已初始化全局/静态变量
├─────────────────────────────────────┤
│           Text段                    │  代码段（只读）
├─────────────────────────────────────┤
│        共享库映射区                  │
└─────────────────────────────────────┘  低地址 0x00000000
```

### 2.2 进程资源清单

```cpp
// 进程独占的资源
struct ProcessResources {
    // 1. 独立地址空间（页表）
    PageTable* page_table;              // 每进程一套，占用数KB~数MB
    
    // 2. 内核数据结构
    task_struct* task;                  // Linux进程描述符 ~1.7KB
    mm_struct* mm;                      // 内存管理结构
    files_struct* files;                // 文件描述符表
    fs_struct* fs;                      // 文件系统信息
    signal_struct* signal;              // 信号处理
    
    // 3. 内存区域
    void* text_segment;                 // 代码段
    void* data_segment;                 // 数据段
    void* bss_segment;                  // BSS段
    void* heap;                         // 堆（初始132KB，brk管理）
    void* stack;                        // 栈（默认8MB限制）
    void* mmap_region;                  // 内存映射区
    
    // 4. 其他资源
    int* fd_table;                      // 文件描述符（默认1024个）
    pid_t pid;                          // 进程ID
    uid_t uid, gid;                     // 用户/组ID
};
```

### 2.3 进程创建开销

```cpp
#include <unistd.h>
#include <sys/wait.h>
#include <chrono>

void benchmark_fork() {
    const int N = 1000;
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < N; i++) {
        pid_t pid = fork();
        if (pid == 0) {
            _exit(0);
        } else {
            waitpid(pid, nullptr, 0);
        }
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    
    std::cout << "Average fork time: " << duration.count() / N << " μs\n";
}
```

## 3. 线程（Thread）资源分析

### 3.1 线程内存模型

```
进程地址空间（所有线程共享）
┌─────────────────────────────────────┐
│           内核空间                   │
├─────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐           │
│  │线程1栈  │  │线程2栈  │  ...      │  每个线程独立栈
│  │ 1~8MB   │  │ 1~8MB   │           │
│  └─────────┘  └─────────┘           │
├─────────────────────────────────────┤
│         共享堆区 (Heap)              │  所有线程共享
├─────────────────────────────────────┤
│         共享数据段                   │  所有线程共享
├─────────────────────────────────────┤
│         共享代码段                   │  所有线程共享
└─────────────────────────────────────┘
```

### 3.2 线程资源清单

```cpp
// 线程独占的资源
struct ThreadResources {
    // 1. 线程栈（最大开销来源）
    void* stack;                        // 默认8MB（可配置）
    size_t stack_guard;                 // 保护页 4KB
    
    // 2. 线程局部存储（TLS）
    void* tls_data;                     // 线程私有数据，通常几KB
    
    // 3. 内核数据结构
    task_struct* task;                  // Linux下每线程一个task_struct
    
    // 4. 寄存器上下文
    struct {
        uint64_t rax, rbx, rcx, rdx;
        uint64_t rsi, rdi, rbp, rsp;
        uint64_t r8, r9, r10, r11;
        uint64_t r12, r13, r14, r15;
        uint64_t rip, rflags;
        uint8_t fpu_state[512];
    } registers;                        // ~600 bytes
    
    // 5. 调度信息
    int priority;
    int cpu_affinity;
    uint64_t cpu_time;
};
```

### 3.3 线程栈大小配置

```cpp
#include <pthread.h>
#include <iostream>

void configure_thread_stack() {
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    
    size_t default_stack_size;
    pthread_attr_getstacksize(&attr, &default_stack_size);
    std::cout << "Default stack size: " << default_stack_size / 1024 << " KB\n";
    
    size_t new_stack_size = 64 * 1024;
    pthread_attr_setstacksize(&attr, new_stack_size);
    pthread_attr_setguardsize(&attr, 4096);
    
    pthread_t thread;
    pthread_create(&thread, &attr, thread_func, nullptr);
    pthread_attr_destroy(&attr);
}
```

## 4. 协程（Coroutine）资源分析

### 4.1 C++20 协程内存模型

```
线程栈
┌─────────────────────────────────────┐
│         线程栈帧                     │
│    ┌─────────────────────┐          │
│    │  普通函数栈帧        │          │
│    └─────────────────────┘          │
│    ┌─────────────────────┐          │
│    │  协程A (挂起点)      │          │
│    │    ↓                │          │
│    │  [协程帧在堆上]      │──────────┼──┐
│    └─────────────────────┘          │  │
│    ┌─────────────────────┐          │  │
│    │  协程B (运行中)      │          │  │
│    └─────────────────────┘          │  │
└─────────────────────────────────────┘  │
                                         │
堆内存                                   │
┌─────────────────────────────────────┐  │
│  ┌──────────────────┐               │  │
│  │  协程帧A          │◄──────────────┼──┘
│  │  - promise对象    │   几十~几百bytes
│  │  - 参数副本       │               │
│  │  - 局部变量       │               │
│  │  - 挂起点状态     │               │
│  └──────────────────┘               │
│  ┌──────────────────┐               │
│  │  协程帧B          │               │
│  └──────────────────┘               │
└─────────────────────────────────────┘
```

### 4.2 协程帧结构

```cpp
struct CoroutineFrame {
    void (*resume_fn)(CoroutineFrame*);
    void (*destroy_fn)(CoroutineFrame*);
    Promise promise;
    uint16_t suspend_index;
    Args... args;
    Locals... locals;
};
```

### 4.3 有栈协程 vs 无栈协程

| 特性 | 无栈协程（C++20） | 有栈协程（Boost.Context） |
|------|------------------|--------------------------|
| 内存占用 | 几十~几百字节 | 64KB~1MB |
| 栈深度 | 只能在协程函数顶层挂起 | 可在任意调用深度挂起 |
| 分配位置 | 堆（可自定义） | 预分配栈空间 |
| 编译器支持 | 需要C++20 | C++11即可 |
| 切换开销 | ~10-50ns | ~50-200ns |

### 4.4 协程内存优化

```cpp
// 1. 自定义协程帧分配器
template<typename T>
struct PoolAllocatorPromise {
    static inline std::pmr::synchronized_pool_resource pool;
    
    void* operator new(size_t size) {
        return pool.allocate(size);
    }
    
    void operator delete(void* ptr, size_t size) {
        pool.deallocate(ptr, size);
    }
};

// 2. 协程帧大小优化
struct OptimizedTask {
    struct promise_type {
        OptimizedTask get_return_object() { 
            return OptimizedTask{std::coroutine_handle<promise_type>::from_promise(*this)}; 
        }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
    };
    
    std::coroutine_handle<promise_type> handle;
};

// 优化前：大量局部变量
OptimizedTask bad_coroutine() {
    char buffer[4096];
    std::vector<int> vec(1000);
    co_await std::suspend_always{};
    co_return;
}

// 优化后：使用堆分配或引用
OptimizedTask good_coroutine() {
    auto buffer = std::make_unique<char[]>(4096);
    auto vec = std::make_shared<std::vector<int>>(1000);
    co_await std::suspend_always{};
    co_return;
}
```

## 5. 切换开销详细对比

### 5.1 进程切换

```
进程A → 进程B 切换步骤：
┌─────────────────────────────────────────────────────────┐
│ 1. 保存进程A的CPU寄存器到内核栈                          │  ~100 cycles
├─────────────────────────────────────────────────────────┤
│ 2. 保存进程A的内核栈指针到task_struct                    │  ~10 cycles
├─────────────────────────────────────────────────────────┤
│ 3. 切换页表（CR3寄存器）                                 │  ~1000+ cycles
│    - TLB全部失效                                        │  (最大开销)
├─────────────────────────────────────────────────────────┤
│ 4. 切换内核栈                                           │  ~10 cycles
├─────────────────────────────────────────────────────────┤
│ 5. 恢复进程B的寄存器                                    │  ~100 cycles
├─────────────────────────────────────────────────────────┤
│ 6. 返回用户态                                           │  ~100 cycles
└─────────────────────────────────────────────────────────┘

总开销：~1000-5000 cycles + TLB miss惩罚
      ≈ 1-10 微秒
```

### 5.2 线程切换

```
线程A → 线程B 切换步骤（同进程内）：
┌─────────────────────────────────────────────────────────┐
│ 1. 系统调用进入内核态                                    │  ~100 cycles
├─────────────────────────────────────────────────────────┤
│ 2. 保存线程A的寄存器                                    │  ~100 cycles
├─────────────────────────────────────────────────────────┤
│ 3. 切换内核栈                                           │  ~10 cycles
├─────────────────────────────────────────────────────────┤
│ 4. 【无需切换页表】                                      │  节省最大开销
├─────────────────────────────────────────────────────────┤
│ 5. 恢复线程B的寄存器                                    │  ~100 cycles
├─────────────────────────────────────────────────────────┤
│ 6. 返回用户态                                           │  ~100 cycles
└─────────────────────────────────────────────────────────┘

总开销：~500-2000 cycles
      ≈ 0.2-1 微秒
```

### 5.3 协程切换

```
协程A → 协程B 切换步骤（用户态）：
┌─────────────────────────────────────────────────────────┐
│ 1. 保存协程A的少量寄存器到协程帧                         │  ~10 cycles
│    （仅callee-saved寄存器）                             │
├─────────────────────────────────────────────────────────┤
│ 2. 保存挂起点索引                                       │  ~5 cycles
├─────────────────────────────────────────────────────────┤
│ 3. 恢复协程B的寄存器和挂起点                             │  ~10 cycles
├─────────────────────────────────────────────────────────┤
│ 4. 跳转到协程B的恢复地址                                 │  ~5 cycles
└─────────────────────────────────────────────────────────┘

总开销：~30-100 cycles
      ≈ 10-50 纳秒
```

## 6. 实际应用场景选择

```
┌─────────────────────────────────────────────────────────────┐
│                     选型决策树                               │
└─────────────────────────────────────────────────────────────┘
                            │
                   需要隔离性吗？
                    ┌───────┴───────┐
                   Yes              No
                    │                │
              选择【进程】      需要真正并行？
                    │           ┌────┴────┐
            多进程架构         Yes        No
         (Nginx/Chrome)         │          │
                          选择【线程】  选择【协程】
                                │          │
                         CPU密集型     I/O密集型
                         多核利用      高并发连接
                                       事件循环
```

## 7. 代码示例：百万级并发对比

```cpp
// ==================== 协程方案 ====================
struct LazyTask {
    struct promise_type {
        LazyTask get_return_object() { 
            return LazyTask{std::coroutine_handle<promise_type>::from_promise(*this)}; 
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
    };
    std::coroutine_handle<promise_type> handle;
};

LazyTask simple_task() {
    co_return;
}

void coroutine_approach() {
    const int TARGET = 1'000'000;
    std::vector<LazyTask> tasks;
    tasks.reserve(TARGET);
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < TARGET; i++) {
        tasks.push_back(simple_task());
    }
    
    auto created = std::chrono::high_resolution_clock::now();
    std::cout << "Created 1M coroutines in " 
              << std::chrono::duration_cast<std::chrono::milliseconds>(created - start).count()
              << " ms\n";
    
    for (auto& task : tasks) {
        task.handle.destroy();
    }
}
```

## 8. 总结

| 维度 | 进程 | 线程 | 协程 |
|------|------|------|------|
| **内存模型** | 独立地址空间 | 共享地址空间，独立栈 | 共享栈，独立小帧 |
| **栈空间** | 8MB（主栈） | 1-8MB | 几十~几百字节 |
| **切换成本** | 最高（TLB刷新） | 中等（系统调用） | 最低（用户态） |
| **创建成本** | ~10ms | ~100μs | ~1μs |
| **适用场景** | 隔离、安全 | CPU密集并行 | I/O密集高并发 |
| **调试难度** | 简单 | 中等 | 较难 |
| **最大数量** | 数百 | 数千~万 | 数十万~百万 |

**选择原则：**
1. **需要进程隔离** → 进程
2. **CPU密集 + 多核利用** → 线程池
3. **I/O密集 + 高并发** → 协程
4. **混合场景** → 线程 + 协程（每线程一个协程调度器）

[src: raw/ingested/2技术/性能优化/内存优化-cpp内存优化-c++-协程、线程、进程的占用资源分析-c++-协程、线程、进程的占用资源分析.md]