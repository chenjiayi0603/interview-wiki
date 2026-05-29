# C++ 并发性能优化

> 本文涵盖 C++ 多线程性能优化的核心技巧：减少锁竞争、避免 False Sharing、线程局部存储、工作窃取等。

See also: [[C++多线程与并发]], [[无锁编程]], [[原子操作与内存模型]], [[C++代码层性能调优最佳实践]]

## 6.1 减少锁竞争

### 6.1.1 细粒度锁

```cpp
// ❌ 粗粒度锁
std::mutex global_mutex;
void process_data() {
    std::lock_guard<std::mutex> lock(global_mutex);
    // 所有操作都在锁内
}

// ✅ 细粒度锁
std::mutex data_mutex;
std::mutex cache_mutex;
void process_data() {
    {
        std::lock_guard<std::mutex> lock(data_mutex);
        // 只保护数据操作
    }
    {
        std::lock_guard<std::mutex> lock(cache_mutex);
        // 只保护缓存操作
    }
}
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-6.-多线程性能优化.md]

### 6.1.2 无锁编程

```cpp
#include <atomic>

// 使用原子操作代替锁
std::atomic<int> counter(0);

void increment() {
    counter.fetch_add(1, std::memory_order_relaxed);
}

// 无锁队列（简化版）
template<typename T>
class LockFreeQueue {
private:
    struct Node {
        std::atomic<T*> data;
        std::atomic<Node*> next;
    };
    
    std::atomic<Node*> head_;
    std::atomic<Node*> tail_;
    
public:
    void enqueue(T item) {
        Node* node = new Node;
        node->data.store(new T(item));
        node->next.store(nullptr);
        
        Node* prev_tail = tail_.exchange(node);
        prev_tail->next.store(node);
    }
};
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-6.-多线程性能优化.md]

### 6.1.3 读写锁

```cpp
#include <shared_mutex>

std::shared_mutex rw_mutex;
int shared_data = 0;

// 读操作（多个线程可同时读）
int read_data() {
    std::shared_lock<std::shared_mutex> lock(rw_mutex);
    return shared_data;
}

// 写操作（独占）
void write_data(int value) {
    std::unique_lock<std::shared_mutex> lock(rw_mutex);
    shared_data = value;
}
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-6.-多线程性能优化.md]

## 6.2 避免 False Sharing

```cpp
// ❌ False Sharing
struct Counter {
    std::atomic<int> count[8];  // 可能在同一缓存行
};

// ✅ 解决：缓存行对齐
struct alignas(64) Counter {
    std::atomic<int> count;
    char padding[64 - sizeof(std::atomic<int>)];
};

Counter counters[8];  // 每个计数器在不同缓存行
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-6.-多线程性能优化.md]

## 6.3 线程局部存储

```cpp
// 使用 thread_local
thread_local int thread_id = 0;
thread_local std::vector<int> local_cache;

void process() {
    local_cache.push_back(thread_id);  // 每个线程有独立的缓存
}

// 使用线程局部存储避免竞争
thread_local int local_sum = 0;
void accumulate(int value) {
    local_sum += value;  // 无竞争
}
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-6.-多线程性能优化.md]

## 6.4 工作窃取（Work Stealing）

```cpp
#include <deque>
#include <mutex>

template<typename T>
class WorkStealingQueue {
private:
    // 为什么多线程 Work Stealing 队列选 std::deque<T> 优于 std::list<T>？
    // 1. deque 内存块连续，空间局部性好，Cache 友好，适合多线程高频 push/pop/steal 操作。
    // 2. deque 支持双端高效插入/删除；而 list 尽管插入/删除也是 O(1)，但遍历慢且每个节点分配单独内存，易造成碎片和 cache miss，降低性能。
    // 3. deque 无额外节点指针跳跃，减少跨核/跨线程 cache 失效风险。
    // 结论：多线程下，Work Stealing 队列用 deque 性能更优。
    std::deque<T> queue_;  // 使用 deque 替代 list 实现高性能双端队列
    // 为啥需要mutable？因为队列的 try_pop/try_steal 函数可能被声明为 const（不改变逻辑数据），
    // 但加锁本身要修改 mutex 状态。如果 mutex_ 不是 mutable，就无法在 const 函数中加锁。
    mutable std::mutex mutex_;  // 保护队列，便于const成员函数内加锁
    
public:
    // 向队列尾部推入任务
    void push(T item) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push_back(item);
    }
    
    // 尝试从队头弹出任务（本地线程出队）
    bool try_pop(T& item) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.empty()) return false;
        item = queue_.front();
        queue_.pop_front();
        return true;
    }
    
    // 尝试从队尾偷取任务（其他线程窃取）
    bool try_steal(T& item) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.empty()) return false;
        item = queue_.back();
        queue_.pop_back();
        return true;
    }
};
```

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-6.-多线程性能优化.md]