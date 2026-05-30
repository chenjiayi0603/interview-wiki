# C++ 编程优化

> 高频交易系统中 C++ 编程优化技术，涵盖无锁编程、内存分配、分支预测、SIMD 向量化、编译器优化等关键领域。

See also: [[C++多线程与并发]], [[原子操作与内存模型]], [[MPMC环形无锁队列-Vyukov]], [[C++并发性能优化]], [[现代C++特性按版本划分]]

## 5.1 无锁编程 (Lock-Free Programming)

### 原子操作与内存序

```cpp
#include <atomic>

// 无锁队列节点
template<typename T>
struct Node {
    T data;
    std::atomic<Node*> next{nullptr};
};

// 无锁单生产者单消费者队列
class LockFreeSPSCQueue {
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};
    
    static constexpr size_t SIZE = 1024;
    Order buffer_[SIZE];
    
public:
    // 生产者写入
    bool push(const Order& order) {
        size_t tail = tail_.load(std::memory_order_relaxed);
        size_t next = (tail + 1) % SIZE;
        
        // 检查队列是否已满
        if (next == head_.load(std::memory_order_acquire)) {
            return false;  // 队列满
        }
        
        buffer_[tail] = order;
        tail_.store(next, std::memory_order_release);
        return true;
    }
    
    // 消费者读取
    bool pop(Order& order) {
        size_t head = head_.load(std::memory_order_relaxed);
        
        // 检查队列是否为空
        if (head == tail_.load(std::memory_order_acquire)) {
            return false;  // 队列空
        }
        
        order = buffer_[head];
        head_.store((head + 1) % SIZE, std::memory_order_release);
        return true;
    }
};
```

#### 内存序详解

```cpp
// std::memory_order_relaxed - 最宽松，仅保证原子性
// std::memory_order_consume - 数据依赖顺序
// std::memory_order_acquire - 获取语义（读屏障）
// std::memory_order_release - 释放语义（写屏障）
// std::memory_order_acq_rel - 获取+释放
// std::memory_order_seq_cst - 顺序一致性（最严格，默认）

// 典型使用模式：发布-订阅
class MessageChannel {
    std::atomic<Message*> msg_{nullptr};
    
public:
    void publish(Message* msg) {
        // 先写入数据，再发布指针
        msg->prepare();  // 准备数据
        msg_.store(msg, std::memory_order_release);  // 发布
    }
    
    Message* consume() {
        // 先获取指针，再读取数据
        Message* msg = msg_.load(std::memory_order_acquire);  // 获取
        if (msg) {
            msg->process();  // 安全读取数据
        }
        return msg;
    }
};
```

[src: raw/ingested/2技术/性能优化/低延迟-高频交易系统优化技术指南-五、C++-编程优化.md]

## 5.2 避免动态内存分配

```cpp
// ❌ 错误：运行时动态分配
void process_order_bad(const OrderRequest& req) {
    auto order = std::make_unique<Order>();  // 堆分配，~100ns
    // ...
}

// ✅ 正确：使用对象池和栈分配
class OrderPool {
    static constexpr size_t POOL_SIZE = 10000;
    
    alignas(64) Order pool_[POOL_SIZE];
    std::atomic<size_t> index_{0};
    
public:
    Order* acquire() {
        size_t idx = index_.fetch_add(1, std::memory_order_relaxed);
        return &pool_[idx % POOL_SIZE];
    }
    
    void release(Order* order) {
        // 标记为可重用
        order->reset();
    }
};

// ✅ 正确：栈分配小对象
void process_order_good(const OrderRequest& req) {
    Order order;  // 栈分配，~1ns
    // ...
}
```

[src: raw/ingested/2技术/性能优化/低延迟-高频交易系统优化技术指南-五、C++-编程优化.md]

## 5.3 分支预测优化

```cpp
#include <likely.h>  // 或自定义宏

#define LIKELY(x)   __builtin_expect(!!(x), 1)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)

// 分支预测提示
void process_market_data(const MarketData& data) {
    // 假设价格上涨是大概率事件
    if (LIKELY(data.price_change > 0)) {
        handle_price_up(data);  // 预测执行路径
    } else if (UNLIKELY(data.price_change < -0.1)) {
        handle_crash(data);     // 异常路径
    }
}

// 使用查找表替代分支
inline PriceLevel get_price_level(double change) {
    // 避免 if-else 链
    static const PriceLevel levels[] = {
        LEVEL_DOWN_3, LEVEL_DOWN_2, LEVEL_DOWN_1,
        LEVEL_FLAT,
        LEVEL_UP_1, LEVEL_UP_2, LEVEL_UP_3
    };
    
    int idx = static_cast<int>((change + 0.15) / 0.05);
    idx = std::clamp(idx, 0, 6);
    return levels[idx];
}
```

[src: raw/ingested/2技术/性能优化/低延迟-高频交易系统优化技术指南-五、C++-编程优化.md]

## 5.4 SIMD 向量化

```cpp
#include <immintrin.h>

// AVX2 批量计算订单价格
void batch_calculate_prices(
    const float* quantities,
    const float* unit_prices,
    float* results,
    size_t n) {
    
    size_t i = 0;
    // 每次处理 8 个 float（256-bit AVX）
    for (; i + 8 <= n; i += 8) {
        __m256 qty = _mm256_loadu_ps(&quantities[i]);
        __m256 price = _mm256_loadu_ps(&unit_prices[i]);
        __m256 result = _mm256_mul_ps(qty, price);
        _mm256_storeu_ps(&results[i], result);
    }
    
    // 处理剩余元素
    for (; i < n; ++i) {
        results[i] = quantities[i] * unit_prices[i];
    }
}

// SIMD 字符串比较（订单ID匹配）
bool fast_order_id_match(const char* a, const char* b) {
    __m128i va = _mm_loadu_si128(reinterpret_cast<const __m128i*>(a));
    __m128i vb = _mm_loadu_si128(reinterpret_cast<const __m128i*>(b));
    __m128i cmp = _mm_cmpeq_epi8(va, vb);
    int mask = _mm_movemask_epi8(cmp);
    return mask == 0xFFFF;  // 16 字节全部相等
}
```

[src: raw/ingested/2技术/性能优化/低延迟-高频交易系统优化技术指南-五、C++-编程优化.md]

## 5.5 编译器优化

```cpp
// 强制内联关键函数
inline __attribute__((always_inline)) void hot_path() {
    // ...
}

// 分支预测友好的循环展开
void process_orders_unrolled(Order* orders, size_t n) {
    size_t i = 0;
    // 展开 4 次减少循环开销
    for (; i + 4 <= n; i += 4) {
        process(orders[i]);
        process(orders[i + 1]);
        process(orders[i + 2]);
        process(orders[i + 3]);
    }
    // 处理剩余
    for (; i < n; ++i) {
        process(orders[i]);
    }
}

// 编译屏障（防止指令重排）
#define COMPILER_BARRIER() asm volatile("" ::: "memory")

// 使用 restrict 关键字
void transform(float* __restrict__ out,
               const float* __restrict__ in, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        out[i] = in[i] * 2.0f;  // 编译器知道无别名，可优化
    }
}
```

[src: raw/ingested/2技术/性能优化/低延迟-高频交易系统优化技术指南-五、C++-编程优化.md]

## 5.6 编译优化选项

```bash
# GCC/Clang 高频交易优化选项
-O3                    # 最高优化级别
-march=native          # 针对本地 CPU 指令集优化
-flto                  # 链接时优化
-fomit-frame-pointer   # 省略帧指针
-fno-exceptions        # 禁用异常（如果不需要）
-fno-rtti              # 禁用 RTTI（如果不需要）
-ffast-math            # 激进浮点优化（注意精度问题）
-funroll-loops         # 循环展开
-finline-functions     # 积极内联
-DNDEBUG               # 禁用断言
```

[src: raw/ingested/2技术/性能优化/低延迟-高频交易系统优化技术指南-五、C++-编程优化.md]