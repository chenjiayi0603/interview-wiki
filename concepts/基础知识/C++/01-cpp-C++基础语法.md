# C++ 基础语法

> 基本语法、类型系统、面向对象、模板、异常处理、RAII。

---

## 一、核心语言特性

### 1.1 关键字与修饰符

| 关键字 | 作用 |
|--------|------|
| `static` | 静态局部变量（函数内）、静态全局/函数（文件内 linkage）、类静态成员、类静态方法 |
| `const` | 常量、常量指针、常量引用、常量成员函数 |
| `constexpr` (C++11) | 编译期常量表达式 |
| `volatile` | 禁止编译器优化，每次从内存读取 |
| `explicit` | 禁止隐式转换/拷贝初始化 |
| `mutable` | 在 const 成员函数中可修改 |
| `virtual` | 虚函数，运行时多态 |
| `override` (C++11) | 显式覆盖基类虚函数 |
| `final` (C++11) | 禁止继承或覆盖 |

### 1.2 static 关键字的四种用法

| 用法 | 说明 |
|------|------|
| **静态局部变量** | 函数内声明，只初始化一次，程序结束时销毁 |
| **静态全局变量/函数** | 限制作用域在本文件内（internal linkage） |
| **类的静态成员** | 所有对象共享，需类外初始化 |
| **类的静态方法** | 无 this 指针，只能访问静态成员 |

### 1.3 指针与引用

| 特性 | 指针 | 引用 |
|------|------|------|
| 可为空 | ✅ 可 nullptr | ❌ 必须绑定对象 |
| 重新赋值 | ✅ 可改变指向 | ❌ 绑定后不可改 |
| 访问方式 | 需解引用 `*p` | 直接使用 |
| sizeof | 8 字节（64位） | 对象大小 |
| 多级 | ✅ `int**` | ❌ 只有一级 |

### 1.4 右值引用与移动语义（C++11）

```cpp
// 左值：可取地址、有名字
// 右值：临时对象、字面量
int&& rref = 42;           // 右值引用绑定右值

// 移动语义：避免不必要的拷贝
std::vector<int> v1 = {1,2,3};
std::vector<int> v2 = std::move(v1);  // 转移所有权

// 完美转发
template<typename T>
void wrapper(T&& arg) {
    func(std::forward<T>(arg));  // 保持左值/右值性
}
```

---

## 二、面向对象

### 2.1 四大特性

| 特性 | 说明 |
|------|------|
| **封装** | 访问控制（public/protected/private）、友元 |
| **继承** | 公有/私有/保护继承、多重继承、虚继承（解决菱形继承） |
| **多态** | 运行时多态（虚函数 vtable）、静态多态（模板/重载） |
| **抽象** | 抽象类（纯虚函数）、接口类 |

### 2.2 虚函数实现原理

- 每个包含虚函数的类有一个**虚表（vtable）**——函数指针数组
- 每个对象有一个**虚表指针（vptr）**指向类的虚表
- 虚函数调用通过 `vptr[vtable_index]` 间接调用
- 多继承时可能有多个 vptr
- 虚表在**编译时**确定，vptr 在**构造函数初始化列表**中设置

### 2.3 拷贝控制（Rule of Five）

```cpp
class Resource {
public:
    Resource();                          // 构造函数
    Resource(const Resource&);           // 拷贝构造
    Resource(Resource&&) noexcept;       // 移动构造（C++11）
    Resource& operator=(const Resource&);// 拷贝赋值
    Resource& operator=(Resource&&);     // 移动赋值（C++11）
    ~Resource();                         // 析构函数
};
```

**Rule of Zero**：若类不管理资源，应让编译器自动生成所有特殊成员函数。

---

## 三、模板

### 3.1 基本模板

模板是 C++ **泛型编程**的核心，让代码在保持类型安全的同时复用于不同类型。

```cpp
// 函数模板 —— 编译器根据实参推导类型
template<typename T>
T max(T a, T b) { return a > b ? a : b; }
int x = max(3, 7);          // T = int
double y = max(3.14, 2.7);  // T = double

// 类模板 —— 参数化类型/非类型
template<typename T, int N>
class Array {
    T data[N];
public:
    int size() const { return N; }
    T& operator[](int i) { return data[i]; }
};
Array<int, 10> arr;  // 编译期确定大小

// 模板参数可以是类型、整型、指针、枚举
template<typename T, size_t N, typename Alloc = std::allocator<T>>
class MyVector { /* ... */ };

// 变参模板（C++11）—— 可变数量的模板参数
template<typename... Args>
void print(Args... args) {
    (std::cout << ... << args) << std::endl;  // 折叠表达式（C++17）
}
print(1, 3.14, "hello");  // 输出 "13.14hello"
```

### 3.2 模板特化与偏特化

**解决的问题**：通用模板不适用于某些特定类型，需要定制实现。

```cpp
// 全特化 —— 对特定类型完全重新实现
template<>
class Array<bool, 8> {
    unsigned char data_;  // 8 个 bool 压缩到 1 字节
public:
    bool operator[](int i) const { return (data_ >> i) & 1; }
};

// 偏特化 —— 对部分参数特化（保留其他参数的泛型性）
template<typename T>
class Array<T*, 0> {  // 指针类型 + 长度为 0 的特化
    // 空数组的优化实现
};

// 函数模板不能偏特化，只能用重载
template<typename T>
bool is_null(T* ptr) { return ptr == nullptr; }

// 模板特化的应用：类型萃取
template<typename T>
struct is_void { static constexpr bool value = false; };
template<>
struct is_void<void> { static constexpr bool value = true; };
```

### 3.3 SFINAE 与 type_traits

**SFINAE（Substitution Failure Is Not An Error）**：模板替换失败时不报错，只是将该模板从重载集合中移除。

```cpp
// SFINAE 典型用法：enable_if 控制函数重载
template<typename T>
typename std::enable_if<std::is_integral<T>::value, double>::type
convert(T val) {
    return static_cast<double>(val);  // 整数类型用 static_cast
}

template<typename T>
typename std::enable_if<std::is_class<T>::value, std::string>::type
convert(const T& obj) {
    return obj.toString();  // 类类型调用 toString()
}

// decltype SFINAE —— 检测是否存在某个成员函数
template<typename T>
auto get_size(const T& t) -> decltype(t.size(), size_t()) {
    return t.size();  // 只有 T 有 size() 时才启用
}

// C++17 简化：if constexpr 替代大部分 SFINAE
template<typename T>
auto to_string(const T& val) {
    if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(val);          // 数字
    } else if constexpr (std::is_same_v<T, std::string>) {
        return val;                          // 字符串直接返回
    } else {
        return std::string(val);             // 其他（需有 const char* 转换）
    }
}

// 常用 type_traits
static_assert(std::is_integral<int>::value);     // 整数类型
static_assert(std::is_pointer<int*>::value);      // 指针
static_assert(std::is_class<std::string>::value); // 类类型
static_assert(std::is_same<int, int32_t>::value); // 类型相同
using T = std::remove_const<const int>::type;     // 移除 const → int
using T2 = std::decay_t<const int&>;              // 退化 → int
```

---

## 四、异常处理

**C++ 异常机制**：`try` 块中抛出异常 → 逐层回退（stack unwinding）→ 析构局部对象 → 匹配 `catch`。

```cpp
#include <stdexcept>
#include <vector>

// 自定义异常 —— 继承 std::exception 体系
class NetworkException : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
    int error_code() const { return code_; }
private:
    int code_ = 0;
};

// 异常安全示例 —— 保证资源不泄漏
void process_data() {
    std::vector<int> v(100);        // vector 析构自动释放堆内存
    FILE* f = fopen("data.txt", "r");
    if (!f) throw std::runtime_error("cannot open file");

    try {
        read_file(f, v);            // 可能抛出异常的操作
    } catch (...) {
        fclose(f);                  // ✅ 确保资源释放
        throw;                      // 重新抛出
    }
    fclose(f);
}

// noexcept —— 承诺不抛异常
// 好处：① 调用方知道不需处理异常 ② 编译器可更好优化
void fast_func() noexcept {
    // 如果 noexcept 函数内抛出异常 → 直接 std::terminate()
}

// 移动操作通常应为 noexcept —— 否则 STL 容器扩容时会用拷贝代替移动
void move_assign(T& a, T& b) noexcept {
    a = std::move(b);
}
```

**异常安全保证级别**：

| 级别 | 说明 | 典型场景 |
|:----:|:-----|:---------|
| **不抛保证** | 绝不抛出异常 | 析构函数、`noexcept` 标记的函数 |
| **强保证** | 完全成功或回滚到操作前状态 | `vector::push_back`（复制元素失败则恢复） |
| **基本保证** | 资源不泄漏，对象状态有效但不确定 | 大部分 STL 操作 |
| **无保证** | 异常后对象不可用 | 极少见 |

**注意**：析构函数、`operator delete`、`swap` 函数**必须为 noexcept**（默认 noexcept）。

---

## 五、RAII 惯用法

**核心思想**：Resource Acquisition Is Initialization —— 构造函数获取资源，析构函数释放资源。
RAII 是 C++ 独有且强大的资源管理范式，也是异常安全的基石。

```cpp
// RAII 管理互斥锁 —— lock_guard 的原理
class LockGuard {
    std::mutex& mtx_;
public:
    explicit LockGuard(std::mutex& m) : mtx_(m) { mtx_.lock(); }   // 构造加锁
    ~LockGuard() { mtx_.unlock(); }                                // 析构解锁
    LockGuard(const LockGuard&) = delete;
    LockGuard& operator=(const LockGuard&) = delete;
};

// 使用：函数正常返回或抛出异常，锁都能正确释放
void thread_safe_func() {
    static std::mutex m;
    LockGuard guard(m);              // 加锁
    /* 临界区代码... */
}   // 离开作用域自动解锁 —— 无需显式写 unlock

// RAII 管理堆内存 —— 手动实现三/五法则
class Buffer {
    int* data_;
    size_t size_;
public:
    Buffer(size_t n) : data_(new int[n]), size_(n) {}
    ~Buffer() { delete[] data_; }

    // 拷贝构造（深拷贝）
    Buffer(const Buffer& other)
        : data_(new int[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + other.size_, data_);
    }
    // 移动构造（接管资源）
    Buffer(Buffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
};

// 标准库中的 RAII 应用
// std::unique_ptr<T>     独占所有权，超出作用域自动 delete
// std::shared_ptr<T>     共享所有权，引用计数归零时 delete
// std::lock_guard        自动加锁/解锁 mutex
// std::unique_lock       灵活加锁/解锁 mutex（支持提前 unlock）
// std::fstream           RAII 管理文件打开/关闭
// std::vector<T>         自动管理动态数组
// std::thread            RAII 管理线程（C++20 jthread 自动 join）

// 日常使用 —— 优先用标准库 RAII 包装器
void process() {
    auto p = std::make_unique<int>(42);            // ✅ 自动 delete
    std::lock_guard lk(mtx);                       // ✅ 自动 unlock
    std::ifstream file("data.txt");                 // ✅ 自动 close
}   // 离开作用域，所有资源自动释放 —— 即使有异常也安全
