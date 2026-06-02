# new 与 delete 详解

> 各种 new/delete 形式、operator new 内部实现、new_handler、placement new、malloc/free。

---

## 一、new/delete 的各种形式

### 1.1 形式速查

| 形式 | 语法 | 说明 | 释放方式 |
|------|------|------|---------|
| 普通 new | `new T` | 分配 + 构造，失败抛 `bad_alloc` | `delete p` |
| 数组 new | `new T[n]` | 分配 n 个对象连续内存 | `delete[] p` |
| 定位 new | `new(ptr) T` | 不分配，在指定位置构造 | 手动 `p->~T()` |
| 不抛异常 new | `new(nothrow) T` | 失败返回 `nullptr` | `delete p` |
| 对齐 new | `new(align_val_t) T` | C++17，指定对齐要求 | `operator delete(p, align_val_t)` |

### 1.2 普通 new 的内部实现

```cpp
// 编译器将 new T 展开为：
void* raw = operator new(sizeof(T));   // 分配原始内存
if (!raw) throw std::bad_alloc();
T* obj = static_cast<T*>(raw);
try {
    new(obj) T(args...);               // placement new 调用构造函数
} catch (...) {
    operator delete(raw);              // 构造失败 → 释放内存
    throw;
}
```

### 1.3 operator new 的调用链

```
new T  →  operator new(size_t)  →  malloc(size)  →  brk (<128KB) / mmap (≥128KB)
  ↑             ↑                     ↑
用户代码    标准库函数             glibc 实现
```

### 1.4 数组 new 的特殊性

```cpp
int* arr = new int[10];   // new[] 会在头部写入数组长度
// 内存布局：[长度头(4字节)] [arr[0]] [arr[1]] ... [arr[9]]

delete[] arr;             // delete[] 读取长度头，逐个析构
// ❌ delete arr;         // 只析构第一个元素，内存泄漏！
```

---

## 二、new_handler 机制

### 2.1 工作原理

`operator new` 分配失败时，在抛出 `std::bad_alloc` 之前，会先调用 `new_handler`：

```cpp
void* operator new(size_t size) {
    void* ptr = malloc(size);
    if (!ptr) {
        std::new_handler h = std::get_new_handler();
        while (h) {
            h();                     // handler 可能释放一些内存
            ptr = malloc(size);      // 再次尝试
            if (ptr) return ptr;
            // handler 什么都不做 → 无限循环！
        }
        throw std::bad_alloc();
    }
    return ptr;
}
```

### 2.2 设置 new_handler

```cpp
#include <new>

void my_handler() {
    std::cerr << "内存不足，释放缓存...\n";
    release_cache();
    // 必须：要么释放内存，要么抛异常
    // 什么都不做会导致无限循环
}

int main() {
    std::set_new_handler(my_handler);     // 设置
    // ...
    std::set_new_handler(nullptr);        // 清除
}
```

### 2.3 最佳实践

- handler 必须：**释放内存** 或 **抛出 `std::bad_alloc`** 或 **终止程序**
- 好的 handler 应限制重试次数，避免无限循环
- 每个线程可设置自己的 handler（C++11）
- 类也可定制自己的 operator new 和 new_handler

---

## 三、placement new

在指定内存地址构造对象（不分配内存）：

```cpp
char buffer[sizeof(MyClass)];          // 栈上或池中预分配
MyClass* p = new(buffer) MyClass();    // 在 buffer 位置构造
// 使用 p
p->~MyClass();                         // 手动析构（不能用 delete！）
```

**使用场景**：
- 内存池/对象池：预分配大块，原地构造
- 共享内存：在共享内存区构造对象
- 寄存器映射：在固定地址构造

**注意事项**：
- ❌ 不能使用 `delete p`（内存不是 new 分配的）
- ✅ 必须手动调⽤析构函数 `p->~T()`
- 确保地址满足对齐要求（C++17 `std::align_val_t`）
- 搭配 `alignas` 确保对齐：

```cpp
alignas(MyClass) char buffer[sizeof(MyClass)];
MyClass* p = new(buffer) MyClass();
```

---

## 四、malloc/free

### 4.1 与 new/delete 对比

| 对比 | new/delete | malloc/free |
|------|-----------|-------------|
| 语言 | C++ 操作符 | C 标准库函数 |
| 构造/析构 | 调⽤构造/析构 | 不调⽤ |
| 返回类型 | 类型安全 | `void*` |
| 失败行为 | 抛异常 / nothrow 返 nullptr | 返回 nullptr |
| 内存大小 | 编译器推算 | 手动指定 |
| 重载 | 可全局/类特定重载 | 可被 `malloc` 钩子替代 |

### 4.2 malloc 底层实现

```cpp
// 简化版内部策略
if (size >= 128KB) {
    return mmap(0, size, ...);     // 大块：mmap 匿名映射
} else {
    return sbrk(size);              // 小块：扩展 brk 堆
}
```

---

## 五、常见陷阱

| 陷阱 | 错误 | 正确 |
|------|------|------|
| new[]/delete 不匹配 | `new int[10]` + `delete p` | `delete[] p` |
| placement new 用 delete | `new(buf) T` + `delete p` | `p->~T()` |
| 基类非虚析构 | `Base* p = new Derived; delete p;` | 基类析构声明 `virtual` |
| 构造失败内存泄漏 | `T* p = new T;` 构造抛异常 | operator new 会自动释放 |
| this 构造 shared_ptr | `shared_ptr<T>(this)` | 继承 `enable_shared_from_this` |
