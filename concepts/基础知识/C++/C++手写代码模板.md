# C++手写代码模板

See also: [[C++面向对象]]

## 单例模式（线程安全）
```cpp
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance; // C++11 线程安全
        return instance;
    }
private:
    Singleton() = default;
    ~Singleton() = default;
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};
```

## 线程安全队列
```cpp
template<typename T>
class ThreadSafeQueue {
private:
    std::queue<T> q;
    std::mutex m;
    std::condition_variable cv;
public:
    void push(T val) {
        std::lock_guard<std::mutex> lock(m);
        q.push(std::move(val));
        cv.notify_one();
    }
    bool pop(T& val) {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, [this]{ return !q.empty(); });
        val = std::move(q.front());
        q.pop();
        return true;
    }
};
```

## 读写锁实现
```cpp
class ReadWriteLock {
    std::mutex mtx;
    std::condition_variable cv;
    int readers = 0;
    int writers = 0;
public:
    void lockRead() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]{ return writers == 0; });
        ++readers;
    }
    void unlockRead() {
        std::lock_guard<std::mutex> lock(mtx);
        --readers;
        cv.notify_all();
    }
    void lockWrite() {
        std::unique_lock<std::mutex> lock(mtx);
        ++writers;
        cv.wait(lock, [this]{ return readers == 0; });
    }
    void unlockWrite() {
        std::lock_guard<std::mutex> lock(mtx);
        --writers;
        cv.notify_all();
    }
};
```

## 生产者-消费者模型
```cpp
template<typename T>
class ProducerConsumer {
    std::queue<T> q;
    std::mutex mtx;
    std::condition_variable cv;
    bool done = false;
public:
    void produce(T val) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            q.push(std::move(val));
        }
        cv.notify_one();
    }
    void consume() {
        while (true) {
            T val;
            {
                std::unique_lock<std::mutex> lock(mtx);
                cv.wait(lock, [this]{ return !q.empty() || done; });
                if (done && q.empty()) return;
                val = std::move(q.front());
                q.pop();
            }
            // 处理 val
        }
    }
    void finish() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            done = true;
        }
        cv.notify_all();
    }
};
```

[src: raw/ingested/2技术/cpp/C++核心知识.md]

## Related Pages
- [[C++面向对象]]
- [[设计模式]]
- [[Go手写代码模板]]
