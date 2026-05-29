# 低延迟高频交易C++核心编程

> 本文整理自数字货币高频交易系统C++开发工程师面试题中关于C++核心编程的部分，涵盖现代C++特性、内存管理、多线程与并发等高频考点。

See also: [[C++语言特性]], [[C++多线程与并发]], [[无锁编程]], [[智能指针]], [[现代C++特性按版本划分]]

## 现代C++特性

### 移动语义与右值引用

右值引用使用 `&&` 声明，移动语义避免深拷贝，可将数据传递开销从O(n)降到O(1)，减少微秒级延迟。

```cpp
// 右值引用使用 && 声明
void processOrder(std::vector<Order>&& orders);  // 接受右值

// 移动语义避免深拷贝
class OrderBook {
    std::vector<Order> orders;
public:
    // 移动构造函数
    OrderBook(OrderBook&& other) noexcept 
        : orders(std::move(other.orders)) {}
    
    // 移动赋值运算符
    OrderBook& operator=(OrderBook&& other) noexcept {
        orders = std::move(other.orders);
        return *this;
    }
};

// 高频交易中的应用：订单数据传递
OrderBook generateOrderBook() {
    OrderBook book;
    // 生成大量订单数据...
    return book;  // 自动使用移动语义，避免拷贝大量订单数据
}
```

**为什么重要：** 高频交易系统处理大量订单数据，移动语义可将数据传递开销从O(n)降到O(1)，减少微秒级延迟。

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]

### 智能指针的使用场景和内存管理策略

| 类型 | 所有权 | 使用场景 |
|------|--------|----------|
| `unique_ptr` | 独占 | 订单对象、连接句柄 |
| `shared_ptr` | 共享 | 配置对象、共享数据 |
| `weak_ptr` | 无所有权 | 打破循环引用、观察者模式 |

```cpp
// 高频交易系统中的典型用法
class TradingEngine {
    // 独占所有权：每个连接只有一个管理者
    std::vector<std::unique_ptr<ExchangeConnection>> connections;
    
    // 共享所有权：多个模块共享市场数据
    std::shared_ptr<MarketData> marketData;
    
public:
    void addConnection(std::unique_ptr<ExchangeConnection> conn) {
        connections.push_back(std::move(conn));
    }
};

// 避免循环引用
class OrderManager;
class OrderHandler {
    std::weak_ptr<OrderManager> manager;  // 使用weak_ptr避免循环
public:
    void checkManager() {
        if (auto sp = manager.lock()) {
            // Manager仍然存在，安全使用
        }
    }
};
```

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]

### 完美转发

```cpp
// 完美转发保持参数的值类别（左值/右值）
template<typename T>
void wrapper(T&& arg) {
    // std::forward保持arg的原始值类别
    processOrder(std::forward<T>(arg));
}

// 高频交易中的应用：通用消息分发器
template<typename Msg, typename... Args>
void dispatchOrder(Args&&... args) {
    auto msg = std::make_unique<Msg>(std::forward<Args>(args)...);
    orderQueue.push(std::move(msg));
}

// 使用
dispatchOrder<LimitOrder>(price, quantity, timestamp);  // 完美转发所有参数
```

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]

### constexpr和const的区别

```cpp
// const: 运行时常量
const int runtime_val = getConfigValue();  // 运行时确定

// constexpr: 编译时常量
constexpr double TICK_SIZE = 0.01;
constexpr int MAX_ORDER_SIZE = 10000;

// constexpr函数：编译时计算
constexpr double calculateFee(double amount) {
    return amount * 0.001;  // 手续费率
}

// 高频交易中的应用：编译时计算交易参数
constexpr double MIN_ORDER_VALUE = calculateFee(1000.0);  // 编译时计算
constexpr int ORDER_BUFFER_SIZE = 1024 * 1024;  // 编译时确定数组大小

// 编译时确定大小的数组
std::array<Order, ORDER_BUFFER_SIZE> orderBuffer;
```

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]

## 内存管理

### RAII

RAII (Resource Acquisition Is Initialization) 是通过对象生命周期管理资源的技术。

```cpp
// 交易所连接的RAII封装
class ExchangeConnection {
    int socket_fd;
    bool connected;
public:
    ExchangeConnection(const std::string& host, int port) {
        socket_fd = socket(AF_INET, SOCK_STREAM, 0);
        // 连接逻辑...
        if (socket_fd < 0) throw std::runtime_error("Failed to connect");
        connected = true;
    }
    
    ~ExchangeConnection() {
        if (connected) {
            close(socket_fd);  // 自动释放资源
        }
    }
    
    // 禁止拷贝
    ExchangeConnection(const ExchangeConnection&) = delete;
    ExchangeConnection& operator=(const ExchangeConnection&) = delete;
    
    // 允许移动
    ExchangeConnection(ExchangeConnection&& other) noexcept 
        : socket_fd(other.socket_fd), connected(other.connected) {
        other.connected = false;
    }
};

// 使用：无论正常返回还是异常，都会自动释放
void processOrders() {
    ExchangeConnection conn("exchange.com", 443);  // 获取资源
    // 处理订单...
    // 函数结束时自动释放连接
}
```

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]

### 内存泄漏检测与避免

**检测工具：**
```bash
# Valgrind检测
valgrind --leak-check=full --show-leak-kinds=all ./trading_app

# AddressSanitizer (编译时)
g++ -fsanitize=address -g -o trading_app main.cpp
```

**避免方法：**
```cpp
// 1. 使用智能指针
auto order = std::make_unique<Order>();

// 2. RAII管理资源
class OrderBuffer {
    void* data;
public:
    OrderBuffer(size_t size) : data(malloc(size)) {}
    ~OrderBuffer() { free(data); }
};

// 3. 容器代替原始数组
std::vector<Order> orders;  // 自动管理内存

// 4. make_shared/make_unique 避免裸new
auto config = std::make_shared<Config>();  // 异常安全
```

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]

## 多线程与并发

### C++内存顺序（memory_order）

```cpp
// memory_order枚举
// memory_order_relaxed: 最弱，只保证原子性
// memory_order_acquire: 读操作，之前的写操作不能重排到之后
// memory_order_release: 写操作，之前的读写不能重排到之后
// memory_order_acq_rel: 同时具有acquire和release语义
// memory_order_seq_cst: 顺序一致性，最强保证

// 高频交易中的应用：订单状态更新
class OrderStatus {
    std::atomic<uint64_t> orderId;
    std::atomic<bool> filled;
    
public:
    // 生产者：发布订单状态
    void setFilled(uint64_t id) {
        orderId.store(id, std::memory_order_relaxed);
        filled.store(true, std::memory_order_release);  // 发布
    }
    
    // 消费者：获取订单状态
    bool isFilled(uint64_t& id) {
        if (filled.load(std::memory_order_acquire)) {  // 获取
            id = orderId.load(std::memory_order_relaxed);
            return true;
        }
        return false;
    }
};
```

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]

### 无锁队列（单生产者单消费者）

```cpp
#include <atomic>

// 简单的无锁队列（单生产者单消费者）
template<typename T, size_t Size>
class SPSCQueue {
    std::array<T, Size> buffer;
    std::atomic<size_t> head{0};
    std::atomic<size_t> tail{0};
    
public:
    bool push(const T& item) {
        size_t current_tail = tail.load(std::memory_order_relaxed);
        size_t next_tail = (current_tail + 1) % Size;
        
        if (next_tail == head.load(std::memory_order_acquire)) {
            return false;  // 队列满
        }
        
        buffer[current_tail] = item;
        tail.store(next_tail, std::memory_order_release);
        return true;
    }
    
    bool pop(T& item) {
        size_t current_head = head.load(std::memory_order_relaxed);
        
        if (current_head == tail.load(std::memory_order_acquire)) {
            return false;  // 队列空
        }
        
        item = buffer[current_head];
        head.store((current_head + 1) % Size, std::memory_order_release);
        return true;
    }
};

// 高频交易中的应用：订单队列
SPSCQueue<Order, 1024> orderQueue;
```

[src: raw/ingested/2技术/性能优化/低延迟-数字货币高频交易系统-C++开发工程师-面试题-2.-C++核心编程.md]