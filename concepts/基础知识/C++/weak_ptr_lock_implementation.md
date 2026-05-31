# weak_ptr::lock() 的实现原理

## 6.1 内部数据结构

shared_ptr 和 weak_ptr 共享一个控制块（control block）：

```
┌─────────────────────────────────┐
│      Control Block              │
├─────────────────────────────────┤
│  use_count (shared_ptr 计数)   │  ← 当为 0 时，对象被销毁
│  weak_count (weak_ptr 计数)    │  ← 当为 0 时，控制块被销毁
│  ptr (指向对象的指针)           │
└─────────────────────────────────┘
          ↑              ↑
          │              │
     shared_ptr      weak_ptr
```

## 6.2 lock() 的伪代码实现

```cpp
template<typename T>
std::shared_ptr<T> weak_ptr<T>::lock() const noexcept {
    // 1. 检查控制块是否存在
    if (!control_block_) {
        return std::shared_ptr<T>();  // 返回空指针
    }
    
    // 2. 原子地增加 use_count（如果对象还存在）
    // 这是一个原子操作，保证线程安全
    if (control_block_->try_increment_use_count()) {
        // 3. 如果成功增加引用计数，创建并返回 shared_ptr
        return std::shared_ptr<T>(control_block_);
    } else {
        // 4. 如果对象已被销毁（use_count 已经是 0），返回空指针
        return std::shared_ptr<T>();  // 返回空指针
    }
}
```

## 6.3 什么时候返回空指针？

**情况1**：weak_ptr 是默认构造的（从未关联任何对象）
```cpp
std::weak_ptr<Observer> wp;  // 默认构造
auto sp = wp.lock();  // 返回空指针
```

**情况2**：weak_ptr 观察的对象已被销毁
```cpp
{
    auto observer = std::make_shared<Observer>(...);
    std::weak_ptr<Observer> wp = observer;
    // observer 离开作用域，引用计数变为 0，对象被销毁
}
auto sp = wp.lock();  // 返回空指针，因为对象已被销毁
```

**情况3**：weak_ptr 被重置
```cpp
std::weak_ptr<Observer> wp = observer;
wp.reset();  // 重置 weak_ptr
auto sp = wp.lock();  // 返回空指针
```

## 6.4 shared_ptr 的默认构造函数

```cpp
template<typename T>
class shared_ptr {
private:
    T* ptr_;                    // 指向对象的指针
    ControlBlock* control_block_;  // 指向控制块的指针

public:
    // 默认构造函数：创建一个空的 shared_ptr
    shared_ptr() noexcept 
        : ptr_(nullptr), control_block_(nullptr) {}
};
```

**关键点**：
- 默认构造函数将 ptr_ 设置为 nullptr
- 将 control_block_ 设置为 nullptr
- 这是一个"空"的 shared_ptr，不管理任何对象

## 6.5 如何判断 shared_ptr 是否为空

**方法1**：使用 operator bool()
```cpp
if (sp) {
    // 不为空
}
```

**方法2**：使用 get()
```cpp
if (sp.get() != nullptr) {
    // 不为空
}
```

**方法3**：与 nullptr 比较
```cpp
if (sp != nullptr) {
    // 不为空
}
```

[src: raw/ingested/weak_ptr_lock_implementation.md]