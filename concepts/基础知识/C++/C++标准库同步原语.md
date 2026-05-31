# C++标准库同步原语

## 锁类型对比

| 类型 | 特点 | 使用场景 |
|------|------|----------|
| `std::mutex` | 基本互斥锁 | 一般场景 |
| `std::timed_mutex` | 支持超时 | 需要超时控制 |
| `std::recursive_mutex` | 可递归加锁 | 递归函数 |
| `std::recursive_timed_mutex` | 递归+超时 | 递归+超时 |
| `std::shared_mutex` (C++17) | 读写锁 | 读多写少 |
| `std::shared_timed_mutex` (C++14) | 读写锁+超时 | 读多写少+超时 |

## RAII锁管理

```cpp
#include <mutex>
#include <shared_mutex>

std::mutex mtx;
std::shared_mutex rw_mtx;

// lock_guard: 简单RAII，不可移动
void example_lock_guard() {
    std::lock_guard<std::mutex> lock(mtx);
    // 临界区
}

// unique_lock: 灵活RAII，可移动，支持延迟加锁
void example_unique_lock() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
    // ... 做一些不需要锁的工作
    lock.lock();
    // 临界区
    lock.unlock();
    // ... 不需要锁的工作
    lock.lock();
    // 临界区
} // 自动解锁

// shared_lock: 共享锁RAII
void example_shared_lock() {
    std::shared_lock<std::shared_mutex> lock(rw_mtx);
    // 读操作
}

// scoped_lock (C++17): 同时锁定多个互斥量，避免死锁
void example_scoped_lock() {
    std::mutex mtx1, mtx2;
    std::scoped_lock lock(mtx1, mtx2);  // 使用死锁避免算法
    // 临界区
}
```

## std::lock 避免死锁

```cpp
std::mutex mtx1, mtx2;

void safe_swap() {
    // 使用std::lock同时锁定多个互斥量
    std::lock(mtx1, mtx2);
    
    // adopt_lock表示已经获得锁
    std::lock_guard<std::mutex> lock1(mtx1, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(mtx2, std::adopt_lock);
    
    // 交换操作
}

// C++17更简洁
void safe_swap_cpp17() {
    std::scoped_lock lock(mtx1, mtx2);
    // 交换操作
}
```

## std::call_once 单次调用

```cpp
#include <mutex>

std::once_flag flag;
std::shared_ptr<Resource> resource;

void init_resource() {
    resource = std::make_shared<Resource>();
}

std::shared_ptr<Resource> get_resource() {
    std::call_once(flag, init_resource);
    return resource;
}

// 线程安全的单例模式
class Singleton {
public:
    static Singleton& instance() {
        std::call_once(init_flag, []{ 
            instance_.reset(new Singleton()); 
        });
        return *instance_;
    }

private:
    Singleton() = default;
    static std::once_flag init_flag;
    static std::unique_ptr<Singleton> instance_;
};

// C++11局部静态变量也是线程安全的
class Singleton2 {
public:
    static Singleton2& instance() {
        static Singleton2 inst;  // 线程安全
        return inst;
    }
private:
    Singleton2() = default;
};
```

## std::future 和 std::promise

```cpp
#include <future>

// promise-future通信
void example_promise() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();
    
    std::thread t([&prom]() {
        // 计算结果
        prom.set_value(42);
    });
    
    int result = fut.get();  // 阻塞等待结果
    t.join();
}

// async异步执行
void example_async() {
    // launch::async: 新线程执行
    // launch::deferred: 延迟到get()时执行
    auto fut = std::async(std::launch::async, []() {
        return compute_something();
    });
    
    // 可以做其他工作
    
    auto result = fut.get();  // 获取结果
}

// packaged_task封装可调用对象
void example_packaged_task() {
    std::packaged_task<int(int, int)> task([](int a, int b) {
        return a + b;
    });
    
    std::future<int> fut = task.get_future();
    
    std::thread t(std::move(task), 2, 3);
    
    int result = fut.get();  // 5
    t.join();
}
```

## std::shared_future

```cpp
#include <future>

void example_shared_future() {
    std::promise<int> prom;
    std::shared_future<int> sf = prom.get_future().share();
    
    // 多个线程可以等待同一个结果
    auto consumer = [sf]() {
        int value = sf.get();  // 每个线程都能获取
        // 使用value
    };
    
    std::thread t1(consumer);
    std::thread t2(consumer);
    std::thread t3(consumer);
    
    prom.set_value(42);
    
    t1.join(); t2.join(); t3.join();
}
```

[src: raw/ingested/2技术/cpp/c++线程进程同步分析-C++标准库同步原语.md]