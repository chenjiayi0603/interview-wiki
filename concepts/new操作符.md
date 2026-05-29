# new 操作符

See also: [[C++内存模型]], [[智能指针]], [[C++高频面试问题]]

`new` 操作符是 C++ 中用于动态内存分配的核心机制，有多种用法和形式。

## 1. new 操作符的基本形式

| 形式 | 语法 | 用途 | 说明 |
|------|------|------|------|
| **普通 new** | `new T` | 分配单个对象 | 分配内存并调用构造函数 |
| **数组 new** | `new T[n]` | 分配对象数组 | 分配 n 个对象的连续内存 |
| **定位 new** | `new(ptr) T` | 在指定位置构造对象 | 不分配内存，只在指定位置构造。常配合 `alignas`（C++11）使用确保对齐 |
| **不抛出异常 new** | `new(nothrow) T` | 分配失败返回 nullptr | 不抛出异常，返回空指针 |
| **对齐 new** | `new(align_val_t, ...)` | 指定对齐要求 | C++17，指定内存对齐。注意：`alignas` 关键字是 C++11 特性 |

## 2. 普通 new（new T）

```cpp
int* p = new int;           // 分配一个 int，未初始化
int* q = new int(42);        // 分配一个 int，初始化为 42
MyClass* obj = new MyClass;              // 调用默认构造函数
MyClass* obj2 = new MyClass(arg1, arg2); // 调用带参数的构造函数
```

**特点**：
- 分配内存并调用构造函数
- 分配失败时抛出 `std::bad_alloc` 异常
- 必须使用对应的 `delete` 释放内存

### 2.1 new 操作符的内部实现机制

`new` 操作符的内部实现分为两个阶段：

**阶段 1：内存分配**
- 调用 `operator new(size_t)` 函数分配原始内存
- `operator new` 是 C++ 标准库提供的全局函数，负责从堆中分配指定大小的内存
- 如果分配失败，`operator new` 会抛出 `std::bad_alloc` 异常

**阶段 2：对象构造**
- 在分配的内存上调用对象的构造函数
- 构造函数完成对象的初始化

**实际的编译器展开过程**：

```cpp
// 用户代码：MyClass* obj = new MyClass(arg1, arg2);
// 编译器实际展开为：
void* raw_memory = operator new(sizeof(MyClass));  // 分配内存
MyClass* obj = static_cast<MyClass*>(raw_memory);
try {
    new(obj) MyClass(arg1, arg2);  // placement new，调用构造函数
} catch (...) {
    operator delete(raw_memory);  // 如果构造失败，释放内存
    throw;  // 重新抛出异常
}
```

### 2.2 operator new 内部详细实现机制

`operator new` 是 C++ 内存分配的核心函数，其内部执行了多个关键步骤：

1. **参数验证**：处理 0 字节请求（C++ 标准要求至少分配 1 字节）
2. **内存分配**：调用底层内存分配函数（通常是 `malloc`）
3. **失败处理**：如果分配失败，调用 `new_handler` 尝试恢复
4. **异常抛出**：如果最终失败，抛出 `std::bad_alloc`
5. **对齐保证**：返回的指针满足对齐要求
6. **线程安全**：保证多线程环境下的安全性

**底层调用链（Linux 系统示例）**：

```
用户代码：new int
    ↓
operator new(sizeof(int))
    ↓
std::malloc(4)
    ↓
glibc 的 malloc 实现
    ↓
系统调用：brk() 或 mmap()
    ↓
Linux 内核：分配虚拟内存页
```

### 2.3 new_handler 的设置与管理

`new_handler` 是 C++ 提供的内存分配失败处理机制，允许用户在内存不足时执行自定义操作。

```cpp
#include <new>

void my_new_handler() {
    std::cerr << "内存分配失败，执行恢复操作...\n";
    throw std::bad_alloc();
}

int main() {
    std::new_handler old_handler = std::set_new_handler(my_new_handler);
    // ...
    std::set_new_handler(old_handler);  // 恢复
    return 0;
}
```

**new_handler 设计要点**：
- handler 应该要么释放内存，要么抛出异常，避免无限循环
- 可以使用 RAII 风格管理 handler 的生命周期
- C++11 起支持线程局部的 new_handler

## 3. 数组 new（new T[n]）

```cpp
int* arr = new int[10];        // 分配 10 个 int，未初始化
int* arr2 = new int[10]();      // 分配 10 个 int，全部初始化为 0
MyClass* objs = new MyClass[5];  // 调用 5 次默认构造函数
```

**重要注意事项**：
1. **必须使用 `delete[]` 释放**：`new[]` 会分配额外的元数据记录数组大小，`delete[]` 需要这些信息来正确调用每个元素的析构函数
2. **数组大小可以是变量**：`int n = 10; int* arr = new int[n];` ✅ 合法
3. **对象数组会调用构造函数**：每个元素都会调用默认构造函数

## 4. 定位 new（Placement new）

定位 new 在已分配的内存上构造对象，不分配新内存：

```cpp
#include <new>

// 使用对齐内存（alignas 是 C++11 特性）
alignas(MyClass) char buffer[sizeof(MyClass)];
MyClass* obj = new(buffer) MyClass(arg1, arg2);

// 手动调用析构函数
obj->~MyClass();  // 必须手动调用析构函数
// 注意：不需要 delete，因为内存不是 new 分配的
```

**使用场景**：
- 内存池实现
- 共享内存
- 性能优化（避免额外的内存分配）

**重要注意事项**：
- 定位 new 不分配内存，只在指定位置构造对象
- 必须手动调用析构函数
- 不能使用 `delete`，因为内存不是 `new` 分配的

## 5. 不抛出异常 new（new(nothrow)）

```cpp
#include <new>

int* p = new(std::nothrow) int;
if (p == nullptr) {
    // 分配失败，但不抛出异常
    std::cerr << "内存分配失败\n";
    return;
}
```

**使用场景**：
- 需要检查分配是否成功，但不希望使用异常处理
- 在异常处理受限的环境中（如某些嵌入式系统）

## 6. 对齐 new（C++17）

```cpp
#include <new>

struct alignas(64) AlignedStruct {
    int data[16];
};

// 使用对齐 new（C++17 特性）
AlignedStruct* p = new(std::align_val_t(64)) AlignedStruct;

// 释放时也需要指定对齐
operator delete(p, std::align_val_t(64));
```

**注意**：
- `alignas` 关键字是 **C++11** 引入的对齐说明符
- 对齐 new（`new(std::align_val_t)`）是 **C++17** 引入的

## 7. new 与 malloc 的区别

| 特性 | new | malloc |
|------|-----|--------|
| **语言** | C++ 操作符 | C 函数 |
| **返回类型** | 类型化指针（如 `int*`） | `void*`（需要转换） |
| **构造函数** | ✅ 自动调用 | ❌ 不调用 |
| **析构函数** | ✅ 自动调用（delete） | ❌ 不调用 |
| **失败处理** | 抛出异常（默认） | 返回 nullptr |
| **大小计算** | 自动（根据类型） | 手动指定字节数 |
| **重载** | ✅ 可以重载 | ❌ 不能重载 |
| **对齐** | 自动满足类型对齐 | 需要手动处理 |

## 8. new 操作符的常见陷阱

| 陷阱 | 错误示例 | 正确做法 |
|------|---------|---------|
| **new[]/delete不匹配** | `int* arr = new int[10]; delete arr;` | `delete[] arr;` |
| **placement new忘记析构** | `new(buffer) T; // 忘记析构` | `obj->~T();` |
| **基类非虚析构** | `Base* p = new Derived; delete p;` | 基类析构函数声明为`virtual` |
| **循环引用** | `shared_ptr`相互持有 | 使用`weak_ptr`打破循环 |
| **异常导致泄漏** | `new`后异常，未释放 | 使用RAII或智能指针 |

## 9. 面试考点

| 考点 | 一句话回答 |
|------|-----------|
| **new vs malloc** | new=分配+构造+异常，malloc=字节+void* |
| **new[]/delete[]配对** | new[]写入长度头，必须delete[]逐个析构 |
| **placement new** | 不分配内存，需手动析构，不能用delete |
| **new_handler机制** | operator new失败先调handler，可释放内存或抛异常 |

[src: raw/ingested/2技术/内存/C++内存管理复习文档.md]

## Related Pages
- [[C++内存模型]]
- [[智能指针]]
- [[C++高频面试问题]]
- [[内存管理]]
- [[C++对象内存布局]]
