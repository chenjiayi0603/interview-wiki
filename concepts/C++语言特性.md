# C++语言特性

See also: [[C++高频面试问题]], [[C++引用与引用折叠]]

## 类型推导与类型系统

### auto 与 decltype（C++11）
**auto**：由初始化表达式推导变量类型。
```cpp
auto i = 42;                    // int
auto d = 3.14;                  // double
auto s = "hello";               // const char*
auto vec = std::vector<int>();  // std::vector<int>
```

**decltype**：推导表达式的类型，表达式不会被运算。
```cpp
int x = 10;
decltype(x) y = 20;       // y 为 int
decltype((x)) z = x;      // z 为 int&（因 (x) 是左值表达式）
```

**auto 与 decltype 的区别**：auto 必须显式初始化才能推导；decltype 不需要初始化，根据表达式推导类型。

**尾置返回类型**（配合 decltype）：
```cpp
template<typename T, typename U>
auto add(T t, U u) -> decltype(t + u) {
    return t + u;
}
```

### 函数返回类型推导（C++14）
```cpp
auto func(int i) { return i; }            // 推导为 int
decltype(auto) get_ref(int& x) { return x; }  // 返回 int&
```

### 类型别名 using（C++11）
比 typedef 更直观，支持模板别名。
```cpp
using IntPtr = int*;
template<typename T>
using Vec = std::vector<T>;
```

### 强类型枚举 enum class（C++11）
```cpp
enum class AColor { kRed, kGreen, kBlue };
AColor c = AColor::kRed;
// if (AColor::kRed == BColor::kWhite) {}  // 编译失败
```

### 类型特征 type_traits（C++11）
```cpp
#include <type_traits>
static_assert(std::is_integral<int>::value, "");
using NoRef = std::remove_reference<int&>::type;   // int
using T = std::conditional<true, int, double>::type;  // int
```

### 结构化绑定（C++17）
```cpp
auto [id, name] = std::pair(42, "hello");
std::map<int, std::string> m = {{0, "a"}, {1, "b"}};
for (const auto& [k, v] : m) { /* ... */ }
```

## 控制流与循环

### 范围 for 循环（C++11）
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
for (int val : vec) { /* 值拷贝 */ }
for (int& val : vec) { val *= 2; }         // 引用，可修改
```

### if / switch 带初始化（C++17）
```cpp
if (int a = GetValue(); a < 101) { std::cout << a; }
if (auto it = m.find(key); it != m.end()) { return it->second; }
```

### if constexpr 编译期分支（C++17）
编译时分支，不会生成未使用分支的代码，用于模板元编程。
```cpp
template<typename T>
auto process(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;
    } else if constexpr (std::is_floating_point_v<T>) {
        return value * 2.0;
    } else {
        return value;
    }
}
```

## Lambda 表达式

### 基本 Lambda（C++11）
**基本语法：** `[捕获列表](参数列表) mutable -> 返回类型 { 函数体 }`
```cpp
auto add = [](int x, int y) { return x + y; };
int x = 100;
auto val_cap = [x]() { return x + 1; };         // 值捕获
auto ref_cap = [&x]() { x += 2; };              // 引用捕获
auto all_val = [=]() { return a + b + x; };     // 全部值捕获
auto all_ref = [&]() { return a + b + x; };     // 全部引用捕获
auto mut = [x]() mutable { x++; return x; };    // mutable 允许修改值捕获副本
```

**Lambda 特性一览表：**
| 特点 | 说明 |
|------|------|
| 本质 | 生成匿名闭包类型对象，重载 `operator()` |
| 捕获方式 | 值捕获、引用捕获、表达式初始化捕获(C++14) |
| 默认 const | 捕获的值默认为 const，加 mutable 可修改 |
| 用途 | STL 算法、函数参数、容器存储、立即调用 |
| 大小 | 无捕获时通常1字节，有捕获则等于捕获内容大小 |
| 注意 | 引用捕获须确保变量生命周期有效，避免悬挂引用 |

### 泛型 Lambda（C++14）
```cpp
auto add = [](auto a, auto b) { return a + b; };
```

### Lambda 初始化捕获（C++14）
```cpp
auto ptr = std::make_unique<int>(42);
auto f = [p = std::move(ptr)]() { return *p; };  // 移动捕获
```

### constexpr Lambda（C++17）
```cpp
constexpr auto lamb = [](int n) { return n * n; };
static_assert(lamb(3) == 9, "");
```

### *this 捕获对象副本（C++17）
`[*this]` 捕获对象的拷贝（而非 this 指针），生命周期与对象无关，多线程场景更安全。

### Lambda 模板参数（C++20）
```cpp
auto lambda = []<typename T>(T x) { std::cout << x << std::endl; };
```

## 内存、所有权与智能指针

### 右值引用与移动语义（C++11）
**核心概念：**
- **左值**：有名字、可取地址（如变量 `a`）
- **右值**：临时的、不可取地址（如字面量 `42`、表达式 `a+b`）
- **右值引用 `T&&`**：绑定到右值，为移动语义和完美转发提供基础

```cpp
int a = 1;
int&& rref = 42;             // 绑定到右值
int&& moved = std::move(a);  // std::move 将左值转为右值引用
```

**移动构造 / 移动赋值**：避免不必要的深拷贝，直接"窃取"资源。
```cpp
class MyString {
    char* data;
    size_t size;
public:
    MyString(MyString&& other) noexcept
        : data(other.data), size(other.size) {
        other.data = nullptr;
        other.size = 0;
    }
    MyString& operator=(MyString&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }
};
```

**std::move 原理：** 本质就是 `static_cast<remove_reference_t<T>&&>(arg)`。

### 完美转发 std::forward（C++11）
**问题**：模板函数中，右值参数传到下一层时会变成左值（因为有了变量名）。
**解决**：`std::forward<T>(arg)` 保持参数原本的值类别。

```cpp
void process(int& x)  { std::cout << "lvalue" << std::endl; }
void process(int&& x) { std::cout << "rvalue" << std::endl; }

template<typename T>
void forwarder(T&& arg) {
    process(std::forward<T>(arg));  // 完美转发
}
```

**引用折叠规则：**
- `T& &` → `T&`
- `T&& &` → `T&`
- `T& &&` → `T&`
- `T&& &&` → `T&&`

> 详细引用折叠规则与万能引用推导，参见 [[C++引用与引用折叠]]。

## 初始化

### 列表初始化与 std::initializer_list（C++11）
```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
std::map<std::string, int> m = {{"a", 1}, {"b", 2}};

void print(std::initializer_list<int> list) {
    for (auto val : list) std::cout << val << " ";
}
print({1, 2, 3, 4, 5});
```

### 统一初始化语法（C++11）
```cpp
int x{42};
int arr[]{1, 2, 3, 4, 5};
std::vector<int> vec{1, 2, 3, 4, 5};
// int n{3.14};  // 错误！防止窄化转换
```

### 指定初始化器（C++20）
```cpp
struct Config {
    int width;
    int height;
    std::string title;
};
Config c = {.width = 1920, .height = 1080, .title = "App"};
```

## 类与继承

### 委托构造函数与继承构造函数（C++11）
**委托构造函数：** 同一类中一个构造函数调用另一个。
```cpp
struct A {
    A() {}
    A(int a) { a_ = a; }
    A(int a, int b) : A(a) { b_ = b; }            // 委托给 A(int)
};
```

**继承构造函数：** 派生类直接使用基类构造函数。
```cpp
struct Base {
    Base(int a) { a_ = a; }
};
struct Derived : Base {
    using Base::Base;  // 继承所有基类构造函数
};
```

### 关键字 final / override / default / delete / explicit（C++11）
```cpp
struct Derived : public Base {
    void func() override {}      // 确保是重写
};
struct Final final { };          // 禁止继承

struct A {
    A() = default;                          // 显式声明默认构造
    A(const A&) = delete;                   // 禁止拷贝构造
    explicit A(int value) {}                // 禁止隐式转换
};
```

### 内联变量（C++17）
头文件中可定义变量，多个源文件包含不会重复定义。
```cpp
inline int global_counter = 0;
struct A {
    inline static const int value = 10;
};
```

## 编译期计算与断言

### constexpr（C++11/14/20）
- `const`：read only，运行时不可修改，但仍可能是动态变量
- `constexpr`：真正的编译期常量，编译时就会计算出结果

C++14 允许 constexpr 函数包含循环、局部变量、多条语句。
C++20 支持 constexpr 虚函数和 constexpr 容器（vector、string）。

### consteval 与 constinit（C++20）
**consteval**：必须在编译期求值（立即函数），不能在运行期调用。
**constinit**：保证变量在编译期初始化，防止静态初始化顺序问题。

### static_assert（C++11）
```cpp
static_assert(sizeof(int) == 4, "int must be 4 bytes");
```

## 模板

### 变参模板 Variadic Templates（C++11）
```cpp
template<class T>
void ShowList(const T& t) { std::cout << t << std::endl; }

template<class T, class... Args>
void ShowList(T value, Args... args) {
    std::cout << value << " ";
    ShowList(args...);
}
```

### 变量模板（C++14）
```cpp
template<typename T>
constexpr T pi = T(3.1415926535897932385L);
```

### 折叠表达式（C++17）
简化变参模板的展开。
```cpp
template<typename... Args>
auto sum(Args... args) { return (args + ...); }          // 一元右折叠
```

### 类模板参数推导 CTAD（C++17）
```cpp
std::pair p(1, 2.0);         // std::pair<int, double>
std::vector v = {1, 2, 3};   // std::vector<int>
```

### 概念 Concepts（C++20）
约束模板参数，替代 SFINAE，更清晰的编译错误信息。
```cpp
#include <concepts>
template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
};
template<Addable T>
T sum(T a, T b) { return a + b; }
```

## 函数与可调用对象

### std::function 与 std::bind（C++11）
```cpp
#include <functional>
std::function<int(int, int)> func;
func = [](int a, int b) { return a * b; };
auto bound = std::bind(add, 10, std::placeholders::_1);
```

### std::ref 与 std::cref（C++11）
`std::bind` 默认值传递参数。如需引用传递，用 `std::ref` / `std::cref`。

### std::apply 与 std::make_from_tuple（C++17）
```cpp
int add(int a, int b) { return a + b; }
std::cout << std::apply(add, std::pair(1, 2)) << std::endl;  // 3
```

## 并发与线程

### 线程库（C++11）
```cpp
#include <thread>
#include <mutex>
std::thread t1([]() { /* ... */ });
if (t1.joinable()) t1.join();

std::mutex mtx;
std::lock_guard<std::mutex> lock(mtx);   // 轻量级，不可手动解锁
std::unique_lock<std::mutex> ulock(mtx); // 更灵活，可手动 unlock
```

### 原子操作 std::atomic（C++11）
无需互斥锁即可保证线程安全的整型操作。
```cpp
#include <atomic>
std::atomic<int> count;
void add() { ++count; }
```

### 条件变量 std::condition_variable（C++11）
条件变量配合 `std::unique_lock` 使用。
```cpp
std::condition_variable cv;
std::mutex cv_mtx;
bool ready = false;
cv.wait(lock, [] { return ready; });
cv.notify_one();
```

### std::future / std::promise / std::async（C++11）
- `std::future`：异步结果的传输通道，通过 `get()` 获取结果
- `std::promise`：包装一个值，与 future 绑定
- `std::packaged_task`：包装一个函数，与 future 绑定
- `std::async`：标准库内置的自动化异步执行工具，简单高效，日常首选

```cpp
std::future<int> res = std::async(std::launch::async, [](int x) { return x + 1; }, 5);
std::cout << res.get() << std::endl;
```

### std::call_once（C++11）
保证函数在多线程环境中只调用一次。

### thread_local（C++11）
每个线程拥有独立实例，类似于 static 但线程隔离。

### std::shared_mutex 读写锁（C++17）
多读单写锁，比 C++14 的 `shared_timed_mutex` 更轻量。
```cpp
std::shared_mutex mtx;
void reader() { std::shared_lock lock(mtx); }
void writer() { std::unique_lock lock(mtx); }
```

### std::scoped_lock（C++17）
多互斥锁 RAII，避免死锁。
```cpp
std::mutex m1, m2;
std::scoped_lock lock(m1, m2);
```

### std::jthread 与 std::stop_token（C++20）
`std::jthread`：析构时自动 join，支持协作式停止。

### 同步原语 latch / barrier / semaphore（C++20）
```cpp
std::latch latch(3);
std::barrier barrier(3);
std::counting_semaphore<10> sem(5);
```

## 文件与系统

### std::filesystem（C++17）
```cpp
#include <filesystem>
namespace fs = std::filesystem;
fs::exists("file.txt");
fs::create_directories("path/to/dir");
```

### std::source_location（C++20）
替代 `__FILE__` / `__LINE__` 宏，用于日志和调试。

## 字符串与文本

### 正则表达式（C++11）
```cpp
#include <regex>
std::regex pattern(R"([\w\.-]+@[\w\.-]+\.\w+)");
std::smatch matches;
if (std::regex_search(text, matches, pattern)) { /* ... */ }
```

### std::string_view（C++17）
非拥有的只读字符串视图，避免不必要的字符串拷贝。
```cpp
#include <string_view>
void print(std::string_view sv) { std::cout << sv << std::endl; }
```

### 字符串转换 from_chars / to_chars（C++17）
高性能的字符串与数值转换（不涉及内存分配）。

### std::format（C++20）
类似 Python 的 format，替代 printf/sprintf。
```cpp
#include <format>
std::cout << std::format("Hello, {}! x={}, y={:.2f}", "world", 42, 3.14159);
```

## 容器与算法

### 新容器（C++11）
- **std::array**：固定大小数组，替代 C 风格数组，支持边界检查、迭代器、STL 算法。
- **std::forward_list**：单向链表，只支持前向遍历，比 `std::list` 省一个指针。
- **std::unordered_set / unordered_map**：哈希容器，O(1) 平均查找。
- **std::tuple**：元组，比 `pair` 更灵活，可存任意数量、任意类型。

### 新算法（C++11）
- **条件判断**：`all_of` / `any_of` / `none_of`
- **查找与复制**：`find_if_not` / `copy_if`
- **填充与极值**：`iota` / `minmax_element` / `is_sorted`

### 并行算法（C++17）
| 策略 | 含义 |
|------|------|
| `std::execution::seq` | 串行，与原算法行为一致 |
| `std::execution::par` | 多线程并行（不同线程处理不同区间） |
| `std::execution::par_unseq` | 多线程 + 允许更激进重排/向量化（SIMD） |

```cpp
#include <execution>
std::sort(std::execution::par, v.begin(), v.end());
```

### std::clamp（C++17）
将值限制在指定区间 `[lo, hi]`，等价于 `max(lo, min(hi, x))`。

### 范围 Ranges（C++20）
函数式风格的数据处理管道，惰性求值，可组合。
```cpp
#include <ranges>
auto result = v
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::views::transform([](int n) { return n * n; })
    | std::views::take(3);
```

## 时间与随机

### chrono 时间库（C++11）
**三个核心概念：**
- `duration`：时间段（如 100ms）
- `time_point`：时间点
- `clocks`：时钟（steady_clock / system_clock / high_resolution_clock）

```cpp
#include <chrono>
auto start = std::chrono::high_resolution_clock::now();
auto end = std::chrono::high_resolution_clock::now();
auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
```

### 随机数库（C++11）
```cpp
#include <random>
std::default_random_engine gen(time(nullptr));
std::uniform_int_distribution<int> int_dis(0, 100);
int r = int_dis(gen);
```

## 内存布局

### 内存对齐（C++11）
**为什么要内存对齐：**
1. 硬件平台限制，保证处理器正确存取数据
2. 提高 CPU 访问速度（对齐后一次就能读完）

**对齐规则：**
- 数据成员对齐：按 `#pragma pack` 和成员自身大小的较小值对齐
- 结构体整体对齐：按最大成员大小和 `#pragma pack` 的较小值，总大小为其整数倍

## 字面量与语法糖

### 自定义字面量（C++11）
```cpp
constexpr mytype operator"" _mytype(unsigned long long n) { return mytype{n}; }
mytype mm = 123_mytype;
```

### 二进制字面量与数字分隔符（C++14）
```cpp
int binary = 0b0001'0011'1010;
int million = 1'000'000;
```

## 命名空间与组织

### 嵌套命名空间（C++17）
```cpp
namespace A::B::C { void func(); }
```

### using enum（C++20）
```cpp
enum class Color { red, green, blue };
using enum Color;
Color c1 = red;
```

## 标准库工具函数

### std::exchange / std::integer_sequence / std::quoted（C++14）
**std::exchange：** 将 obj 设为 new_val，返回旧值。
**std::integer_sequence：** 编译时整数序列。
**std::quoted：** 给字符串添加双引号。

## 属性与编译提示

### 属性（C++11/14/17）
```cpp
[[noreturn]] void fatal_error();
[[deprecated("use new_func instead")]] void old_func() {}
[[nodiscard]] int compute();
[[maybe_unused]] void debug_func();
[[fallthrough]];
```

### [[likely]] / [[unlikely]]（C++20）
分支预测提示，仅作为编译器优化建议。

### __has_include 预处理（C++17）
```cpp
#if __has_include(<charconv>)
    #include <charconv>
#endif
```

## 运算符与比较

### 三路比较运算符 <=>（C++20）
自动生成 `==`、`!=`、`<`、`<=`、`>`、`>=`。
```cpp
#include <compare>
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;
};
```

## 协程与模块

### 协程 Coroutines（C++20）
通过 `co_await`、`co_yield`、`co_return` 实现挂起/恢复的异步逻辑。

### 模块 Modules（C++20）
替代头文件系统，减少编译时间，避免宏污染。
```cpp
// mymodule.cppm
export module mymodule;
export int add(int a, int b) { return a + b; }

// main.cpp
import mymodule;
```

## 编译器与标准支持

| GCC 版本 | C++11 | C++14 | C++17 | C++20 | 说明 |
|----------|-------|-------|-------|-------|------|
| GCC 4.8+ | ✅ | 部分 | ✗ | ✗ | |
| GCC 5+ | ✅ | ✅ | 部分 | ✗ | |
| GCC 7+ | ✅ | ✅ | ✅ | 部分 | filesystem 建议 GCC 8+ |
| GCC 10 | ✅ | ✅ | ✅ | ~90% | Coroutine/Ranges 部分缺失 |
| **GCC 11** | ✅ | ✅ | ✅ | **✅** | **推荐的 C++20 最低版本** |
| GCC 12/13/14 | ✅ | ✅ | ✅ | ✅ | 更稳定，部分 C++23 支持 |

## 面试重点速查

### C++11 必考
| 知识点 | 核心要点 |
|--------|---------|
| **右值引用/移动语义** | `T&&`、`std::move`、移动构造/赋值、避免深拷贝 |
| **智能指针** | `shared_ptr`（引用计数）、`unique_ptr`（独占）、`weak_ptr`（打破循环引用） |
| **Lambda** | 捕获列表、值/引用捕获、mutable、与 STL 配合 |
| **完美转发** | `std::forward`、引用折叠、万能引用 `T&&` |
| **auto/decltype** | 类型推导、区别 |
| **线程库** | `thread`、`mutex`、`lock_guard/unique_lock`、`condition_variable`、`atomic` |
| **变参模板** | 参数包展开、递归/逗号表达式方式 |

### C++14/17 常考
| 知识点 | 核心要点 |
|--------|---------|
| **结构化绑定** | 解构 pair/tuple/struct/array |
| **std::optional** | 表达可能无值的返回 |
| **std::variant** | 类型安全联合体，`std::visit` |
| **string_view** | 避免字符串拷贝 |
| **if constexpr** | 编译期分支 |
| **折叠表达式** | 简化变参模板 |

### C++20 了解
| 知识点 | 核心要点 |
|--------|---------|
| **Concepts** | 约束模板参数，清晰错误信息 |
| **Coroutines** | `co_yield`、`co_await`、`co_return` |
| **Ranges** | 管道式数据处理 |
| **`<=>`** | 自动生成所有比较运算符 |
| **std::format** | 现代格式化 |
| **std::jthread** | 自动 join + 协作式停止 |

[src: raw/ingested/2技术/cpp/现代C++特性完整复习手册.md]

## Related Pages
- [[C++高频面试问题]]
- [[C++进阶知识点]]
- [[STL容器与算法]]
- [[智能指针]]
- [[C++引用与引用折叠]]
