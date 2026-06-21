# 现代 C++ 特性

> C++11/14/17/20 新特性，按版本组织，配合重点示例。

---

## 一、C++11：现代 C++ 起点

### 1.1 类型推导 — auto / decltype / using

**核心用途**：简化类型声明、模板编程中推导返回类型、节省冗长的迭代器类型书写。

```cpp
// auto：编译器推导类型
auto i = 42;                       // int
auto d = 3.14;                     // double
auto f = [](int x) { return x; };  // lambda 类型（不可写）

// decltype：获取表达式的类型
int x = 0;
decltype(x) y = 1;                 // int
decltype((x)) ref = x;             // int& —— (x) 是左值表达式，推导为引用

// 返回类型后置（trailing return type）—— 模板中最常用
template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b) { return a + b; }

// using 别名（替代 typedef，更清晰）
using Vec = std::vector<int>;
using Cb = void(*)(int, double);   // 函数指针别名

// decltype(auto) —— C++14，完美转发返回类型
decltype(auto) get_ref(T& t) { return t; }  // 保持引用性
```

**常见陷阱**：
- `auto` 会忽略引用和 cv 限定符 → 显式 `const auto&`
- `decltype((x))` 和 `decltype(x)` 不同——多加一层括号变引用
- 避免过度使用 `auto` 让代码难以阅读（如 `auto p = sth` 不如 `MyService* p` 清晰）

### 1.2 右值引用与移动语义

**解决的问题**：临时对象拷贝造成性能浪费，移动语义将资源"偷"过来避免深拷贝；完美转发解决参数值类别丢失的问题。

---

#### 1.2.1 右值引用 (`&&`) — 区分值类别

**解决的问题**：C++98 无法区分左值和右值，导致临时对象也发生深拷贝。右值引用 `&&` 绑定到即将销毁的临时对象，允许"偷取"资源。

```cpp
#include <iostream>
#include <string>
#include <vector>

// 重载区分左值/右值
void process(const std::string& s) {
    std::cout << "左值版本: " << s << "\n";
}

void process(std::string&& s) {
    std::cout << "右值版本: " << s << "\n";
}

int main() {
    std::string name = "hello";
    process(name);              // 调用左值版本（const&）
    process("world");           // 调用右值版本（&&）—— 临时对象
    process(std::move(name));   // 调用右值版本（&&）—— static_cast<string&&>
    // std::move 本质：不做任何移动，只是将参数转为右值引用类型
}
// 输出：
//   左值版本: hello
//   右值版本: world
//   右值版本: hello

// 右值引用变量本身是左值（有名字、可取地址）
std::string&& ref = std::string("temp");
// ref 是左值（尽管类型是右值引用），可以 &ref 取地址
process(ref);                   // ❌ 调用左值版本！不会触发移动
process(std::move(ref));        // ✅ 显式转回右值
```

**关键理解**：`std::move` 不移动任何东西，只是 `static_cast<T&&>()`。移动发生在移动构造函数/赋值运算符中。

---

#### 1.2.2 移动语义 — 移动构造/赋值

**解决的问题**：深拷贝临时对象浪费 CPU 和内存。移动语义将堆上资源指针"偷"过来，源对象置空，性能从 O(n) 降到 O(1)。

```cpp
#include <string>
#include <vector>

// 自定义 Buffer 实现移动语义
class Buffer {
    char* data_;
    size_t size_;
public:
    // 构造函数
    Buffer(size_t size) : data_(new char[size]), size_(size) {
        std::cout << "构造 " << size_ << "\n";
    }
    
    // 析构函数
    ~Buffer() { delete[] data_; std::cout << "析构\n"; }
    
    // 拷贝构造 —— 深拷贝 O(n)
    Buffer(const Buffer& other)
        : data_(new char[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_);
        std::cout << "拷贝构造 O(n)\n";
    }
    
    // 移动构造 —— 偷指针 O(1)
    Buffer(Buffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;    // 源对象不再持有资源
        other.size_ = 0;
        std::cout << "移动构造 O(1)\n";
    }
    
    // 移动赋值
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;       // 释放自己的旧资源
            data_ = other.data_;  // 偷取对方的资源
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
};

// 标准库容器自动利用移动语义
std::vector<int> v1 = {1, 2, 3, 4, 5};
std::vector<int> v2 = std::move(v1);   // v1 变为空，v2 获得数据 O(1)
std::cout << v1.size() << "\n";         // 0（合法但未指定状态）
std::cout << v2.size() << "\n";         // 5

// push_back 移动版本 —— 类内插入避免拷贝
std::vector<std::string> words;
std::string s = "hello";
words.push_back(s);                     // 拷贝（保留 s）
words.push_back(std::move(s));          // 移动（s 变空）
// 移动后 s 处于合法但未指定状态，s = "new" 可以，但 s[0] 未定义

// noexcept 的重要性
std::vector<Buffer> bufs;
bufs.reserve(1);                        // 预分配 1 个元素空间
Buffer b(100);
bufs.push_back(std::move(b));           // 移动构造 —— noexcept 保证
// 如果移动构造没有 noexcept，vector 扩容时会选拷贝（保证强异常安全）
// 因为拷贝失败可以回滚，移动失败无法撤销
```

**性能对比**：

| 操作 | 拷贝 | 移动 |
|:----:|:----:|:----:|
| `std::string` | O(n) 内存分配+拷贝 | O(1) 指针交换 |
| `std::vector<int>(1000)` | O(n) 复制 | O(1) 指针交换（控制块） |
| `std::array<int,1000>` | O(n) 复制 | O(n) 复制（无动态资源） |

**注意**：移动后源对象处于"合法但未指定"状态，只能赋值或析构，不能假设其值。对于基本类型（int、double 等），`std::move` 退化为拷贝。

---

#### 1.2.3 完美转发 — `std::forward`

**解决的问题**：模板参数推导会丢失值类别信息（引用折叠导致左值/右值不分）。`std::forward` 根据模板参数类型条件式转发为左值或右值。

```cpp
#include <iostream>
#include <memory>
#include <utility>

// 引用折叠规则（模板参数 T 推导时）：
//   T& + &  → T&        T& + && → T&
//   T&& + & → T&        T&& + && → T&&
// 简记：只要出现左值引用，结果就是左值引用

// 完美转发函数
template<typename T>
void wrapper(T&& arg) {
    // arg 是左值（有名字），直接传递永远走左值版本
    // process(arg);               // ❌ 始终调用左值版本
    
    // std::forward 根据 T 推导结果决定转发方式
    process(std::forward<T>(arg));  // ✅ 保持原始值类别
}

void process(int& i)  { std::cout << "左值\n"; }
void process(int&& i) { std::cout << "右值\n"; }

int main() {
    int x = 42;
    wrapper(x);                     // T = int&  → forward<int&>  → 左值引用
    wrapper(42);                    // T = int   → forward<int>   → 右值引用
    wrapper(std::move(x));          // T = int   → forward<int>   → 右值引用
}
// 输出：
//   左值
//   右值
//   右值

// 实用案例：make_unique 的完美转发实现
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(
        new T(std::forward<Args>(args)...)  // 保持每个参数的原始值类别
    );
}

struct Person {
    Person(std::string name, int age) 
        : name_(std::move(name)), age_(age) {}
private:
    std::string name_;
    int age_;
};

auto p = make_unique<Person>("Alice", 30);
// "Alice" 是 const char[6] → 右值 → 转发为右值引用
// 30 是 int → 按值传递

// emplace_back 内部完美转发 —— 直接在容器内构造，减少一次移动
std::vector<std::pair<int, std::string>> v;
v.push_back(std::make_pair(1, "hello"));  // 构造 pair + 移动进 vector
v.emplace_back(1, "hello");               // ✅ 直接在 vector 内构造，零拷贝
// emplace_back 将参数完美转发给 pair 的构造函数
```

### 1.3 Lambda 表达式

**解决的问题**：就地定义匿名函数对象，配合 STL 算法使用极其方便。

```cpp
// 基本形式：[捕获列表](参数列表) mutable(可选) noexcept(可选) -> 返回类型 { 函数体 }

// 基础用法
auto add = [](int a, int b) { return a + b; };      // 编译器推导返回类型
std::cout << add(3, 4) << std::endl;                 // 7

// 与 STL 算法配合 —— 最常用场景
std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });  // 降序排列
auto it = std::find_if(v.begin(), v.end(), [](int x) { return x > 5; });

// 捕获方式详解
int x = 1, y = 2;
auto f1 = [x, &y]() { /* x 只读, y 可修改 */ };     // 按值捕获 x，按引用捕获 y
auto f2 = [=, &y]() { /* 默认按值，y 按引用 */ };   // 捕获所有外部变量（按值）
auto f3 = [&, x]() { /* 默认按引用，x 按值 */ };     // 捕获所有（按引用）

// mutable —— 按值捕获的变量可修改（但外部不受影响）
int count = 0;
auto gen = [count]() mutable { return ++count; };    // 每次调用递增
std::cout << gen() << gen();                          // 12（count 外部仍是 0）

// 捕获 this / *this
class Handler {
    int value_ = 42;
public:
    std::function<int()> getFunc() {
        return [this]() { return value_; };           // 捕获 this 指针（注意生命周期）
        // return [*this]() { return value_; };       // C++17，捕获 *this 副本（安全）
    }
};

// 泛型 Lambda（C++14）
auto print = [](const auto& x) { std::cout << x << std::endl; };
print(42);      // int
print(3.14);    // double
print("hi");    // const char*
```

**性能注意**：无捕获的 lambda 可退化为函数指针，零开销。按引用捕获无开销。按值捕获大的对象有开销。

### 1.4 智能指针

**解决的问题**：手动 `new`/`delete` 容易泄漏，智能指针将资源管理绑定到对象生命周期（RAII）。

---

#### 1.4.1 std::unique_ptr — 独占所有权

**解决的问题**：裸指针需手动释放，`unique_ptr` 保证同一时间只有一个所有者，离开作用域自动释放。

```cpp
#include <memory>
#include <cstdio>

// 基本用法：独占所有权，零开销
std::unique_ptr<int> p1 = std::make_unique<int>(42);
// auto p2 = p1;                  // ❌ 禁止拷贝（编译错误）
auto p2 = std::move(p1);          // ✅ 移动转移所有权，p1 变空
// p1 此刻为 nullptr，再次解引用是 UB

// 自定义删除器 —— 管理非内存资源（文件、socket 等）
auto file_deleter = [](FILE* fp) {
    if (fp) {
        std::cout << "Closing file\n";
        fclose(fp);
    }
};
std::unique_ptr<FILE, decltype(file_deleter)> fp(fopen("a.txt", "r"), file_deleter);
// 离开作用域时自动 fclose，无需手动调用

// 数组特化 —— 自动调用 delete[]
std::unique_ptr<int[]> arr = std::make_unique<int[]>(10);
arr[0] = 42;                     // operator[] 可用，普通 unique_ptr 不行

// 工厂模式返回值 —— 最常用场景
std::unique_ptr<Base> create() {
    return std::make_unique<Derived>();  // 多态：返回派生类指针
}
// auto p = create();  // 使用者不需要手动 delete
```

**性能**：零开销抽象，与裸指针大小相同（无自定义删除器时），无运行时额外成本。

---

#### 1.4.2 std::shared_ptr — 引用计数共享所有权

**解决的问题**：多个对象需要共享同一资源的所有权，最后一个使用者负责释放。

```cpp
#include <memory>

// 优先用 make_shared：一次分配（控制块+对象合并），异常安全更好
auto sp1 = std::make_shared<int>(42);     // ✅ 1 次堆分配
std::shared_ptr<int> sp2(new int(42));    // ❌ 2 次堆分配（对象 + 控制块分开）
// ↑ 优先用 make_shared，节省一次分配，异常安全更好

// 引用计数工作原理
auto sp3 = sp1;                           // use_count == 2
std::cout << sp1.use_count() << "\n";     // 2
{
    auto sp4 = sp1;                       // use_count == 3
}                                         // sp4 析构，use_count == 2
sp3.reset();                              // use_count == 1
// sp1 离开作用域 → use_count == 0 → 自动 delete 对象

// 自定义删除器（类型擦除，unique_ptr 不支持）
std::shared_ptr<FILE> sp_fp(fopen("b.txt", "r"), [](FILE* fp) {
    if (fp) { fclose(fp); }
});
// 注意：shared_ptr 删除器不是模板参数（类型擦除），比 unique_ptr 灵活

// enable_shared_from_this —— 类内安全获取自身 shared_ptr
struct Task : std::enable_shared_from_this<Task> {
    void doWork() {
        auto self = shared_from_this();  // ✅ 安全获取当前对象的 shared_ptr
        // 不允许从普通指针（this）构造 shared_ptr（会重复释放）
    }
};
auto task = std::make_shared<Task>();
task->doWork();

// 循环引用警告 —— 导致内存泄漏！
struct Node {
    std::shared_ptr<Node> next;     // ❌ 环形引用，无法释放
};
// 解决方案：使用 weak_ptr（见 1.4.3）
```

**性能**：`shared_ptr` 大小是裸指针的 2 倍（指向对象的指针 + 指向控制块的指针），引用计数原子操作有同步开销。

---

#### 1.4.3 std::weak_ptr — 观察者，打破循环

**解决的问题**：`shared_ptr` 循环引用导致内存泄漏；需要"观察"资源是否存活而不延长生命周期。

```cpp
#include <memory>

// 基本行为：从 shared_ptr 构造，不增加引用计数
std::weak_ptr<int> wp;
{
    auto sp = std::make_shared<int>(42);
    wp = sp;                             // weak_count+1，use_count 不变（仍为 1）
    std::cout << wp.use_count() << "\n"; // 1
}                                        // sp 析构，对象被释放
// 对象已释放，weak_ptr 自动感知

// expired() + lock() —— 线程安全的访问模式
if (auto sp = wp.lock()) {               // ✅ lock() 返回 shared_ptr 或 nullptr
    std::cout << "Still alive: " << *sp << "\n";
} else {
    std::cout << "Object destroyed\n";
}
// lock() 是原子操作：检查 + 增加引用计数，线程安全

// 打破循环引用 —— 树形结构
struct TreeNode {
    int value;
    std::shared_ptr<TreeNode> children[10];
    std::weak_ptr<TreeNode> parent;      // ✅ 父节点用 weak_ptr，避免循环
    ~TreeNode() { std::cout << "~" << value << "\n"; }
};
{
    auto root = std::make_shared<TreeNode>(1);
    auto child = std::make_shared<TreeNode>(2);
    root->children[0] = child;
    child->parent = root;                // weak_ptr，不增加引用计数
}                                        // ✅ 两个节点都正确释放

// 缓存模式 —— 不持有资源，资源释放时缓存自动失效
std::map<int, std::weak_ptr<ExpensiveObj>> cache;
auto getOrCreate(int key) -> std::shared_ptr<ExpensiveObj> {
    if (auto obj = cache[key].lock()) {
        return obj;                      // 缓存命中
    }
    auto obj = std::make_shared<ExpensiveObj>(key);
    cache[key] = obj;
    return obj;                          // 缓存未命中，创建
}
// 当外部所有 shared_ptr 释放后，缓存弱引用自动失效，不造成泄漏
```

**性能**：`weak_ptr` 大小与 `shared_ptr` 相同（2 倍指针），`lock()` 涉及原子操作但开销小。

---

**选型原则**：

| 场景 | 使用 |
|:----|:----|
| 明确独占所有权 | `unique_ptr`（零开销优先） |
| 多个拥有者 | `shared_ptr`（需共享） |
| 观察/打破循环 | `weak_ptr` |
| 自定义删除器 | `unique_ptr` 删除器是类型一部分（有额外空间），`shared_ptr` 类型擦除（灵活） |

### 1.5 类型安全与枚举增强

#### 1.5.1 nullptr — 类型安全空指针

**解决的问题**：`NULL` 本质是整数 `0`，在重载解析中会意外匹配 `int` 版本。

```cpp
void foo(int);      void foo(char*);
foo(NULL);          // ❌ 调用 foo(int) —— 危险！NULL 是整数 0
foo(nullptr);       // ✅ 调用 foo(char*) —— nullptr 类型是 nullptr_t
foo(0);             // ❌ 也调用 foo(int)

// nullptr_t 可以隐式转指针和 bool，但不能转整数
int* p = nullptr;   // ✅
bool b = nullptr;   // ✅（false）
int n = nullptr;    // ❌ 编译错误
```

**性能**：零开销抽象，`nullptr` 在运行时就是 0，没有额外成本。

---

#### 1.5.2 enum class — 强类型枚举

**解决的问题**：C++98 枚举无作用域限制，会污染命名空间，且能隐式转整数。

```cpp
// C++98 枚举的问题
enum Color { RED, GREEN, BLUE };
enum Fruit { APPLE, BANANA, RED };   // ❌ 编译错误：RED 重定义
int c = RED;                         // ❌ 隐式转 int，类型不安全

// C++11 强类型枚举
enum class Color { RED, GREEN, BLUE };
enum class Fruit { APPLE, BANANA, RED };  // ✅ 各自独立作用域
Color c = Color::RED;                      // 必须作用域访问
// int n = c;                              // ❌ 不能隐式转 int
if (c == Color::RED) {}                    // ✅

// 可指定底层类型（默认 int）
enum class Flag : uint8_t { READ = 1, WRITE = 2, EXEC = 4 };
Flag f = static_cast<Flag>(static_cast<uint8_t>(Flag::READ) | static_cast<uint8_t>(Flag::WRITE));
```

**适用场景**：所有枚举类型都应该用 `enum class`，除非有特殊理由（如与 C 代码交互）。

---

#### 1.5.3 constexpr — 编译期计算

**解决的问题**：需要在编译期计算的值（数组大小、模板参数、static_assert）。

```cpp
// C++11 constexpr 函数：只能包含一条 return 语句（无局部变量、无循环）
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// 编译期使用
int arr[factorial(5)];          // 数组大小 120，编译期确定
static_assert(factorial(5) == 120, "factorial(5) should be 120");

// 非类型模板参数
template<int N>
struct CompileTimeBuffer {
    char data[N];
};
CompileTimeBuffer<factorial(3)> buf;  // N = 6

// 运行时也能调用（双重语义）
int n = 5;
int r = factorial(n);               // 运行时计算，没问题

// 和 const 的区别
const int x = 5;                    // 运行时常量（不一定编译期）
constexpr int y = 5;                // 编译期常量
// constexpr int z = rand();        // ❌ 编译错误：rand() 不是 constexpr
```

**注意**：C++11 的 constexpr 限制严格（只能单 return + 递归），C++14 大幅放宽（允许循环/局部变量）。

---

#### 1.5.4 Range-based for — 简化遍历

**解决的问题**：传统 `for (auto it = v.begin(); it != v.end(); ++it)` 繁琐易错。

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// 只读遍历（推荐：const auto& 避免拷贝）
for (const auto& x : v) {
    std::cout << x << " ";        // 1 2 3 4 5
}

// 修改遍历
for (auto& x : v) {
    x *= 2;                       // 每个元素翻倍
}

// 值拷贝（适合小类型如 int/char，或需要修改副本不影响原数据）
for (auto x : v) {
    // x 是副本
}

// 底层原理：编译器展开为
// for (auto _begin = v.begin(), _end = v.end(); _begin != _end; ++_begin) {
//     auto x = *_begin;
// }

// 自定义类型支持 range-for：只需提供 begin()/end()
class MyRange {
    int* data_;
public:
    int* begin() { return data_; }
    int* end()   { return data_ + size_; }
};
```

---

#### 1.5.5 特殊成员控制 — override / final / default / delete

**解决的问题**：显式表达设计意图，让编译器帮检查错误。

```cpp
// override —— 显式覆盖基类虚函数（编译器检查拼写和签名）
struct Base {
    virtual void foo() const;
    virtual void bar();
};
struct Derived : Base {
    void foo() const override;    // ✅ 覆盖 Base::foo
    void foo() override;          // ❌ 编译错误：缺少 const，不匹配
    void bar() override;          // ✅ 覆盖
};

// final —— 禁止继承或禁止覆盖
struct Sealed final : Base {      // final 类：不能被继承
    void bar() final;             // final 虚函数：不能被覆盖
};
// struct Fail : Sealed {};       // ❌ 编译错误：Sealed 是 final 的

// default —— 让编译器生成默认实现（= 0 是纯虚，= default 是默认实现）
struct Base {
    virtual ~Base() = default;    // ✅ 让编译器生成默认析构
    Base() = default;
    Base(const Base&) = default;  // 默认拷贝构造
    Base& operator=(const Base&) = default;
};

// delete —— 禁止编译器生成或调用某函数
struct NonCopyable {
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;     // 禁止拷贝构造
    NonCopyable& operator=(const NonCopyable&) = delete; // 禁止拷贝赋值
};

// delete 还能用于禁止特定重载（更巧妙）
void foo(int x) { /* ... */ }
void foo(double) = delete;        // ❌ 调用 foo(3.14) 编译错误，避免隐式转换
// foo(3.14f);                    // 会优先匹配 int（float 可转 int 和 double）
// foo(3.14);                     // 编译错误：调用已删除的 double 版本
```

---

#### 1.5.6 初始化列表 — initializer_list

**解决的问题**：统一初始化语法，支持为容器传入任意数量的初始化值。

```cpp
#include <initializer_list>

// 容器初始化（最常用）
std::vector<int> v = {1, 2, 3, 4, 5};
std::map<int, std::string> m = {{1, "one"}, {2, "two"}};
std::set<int> s = {3, 1, 4, 1, 5, 9};  // 重复自动忽略

// 自定义类支持 initializer_list
class MyVector {
    int* data_;
    size_t size_;
public:
    MyVector(std::initializer_list<int> list)
        : data_(new int[list.size()]), size_(list.size()) {
        std::copy(list.begin(), list.end(), data_);
    }
    ~MyVector() { delete[] data_; }
};
MyVector mv = {1, 2, 3, 4, 5};       // 调用 initializer_list 构造

// 注意：initializer_list 构造函数的优先级高于普通构造函数
std::vector<int> v1(5, 10);          // 5 个 10 → [10, 10, 10, 10, 10]
std::vector<int> v2{5, 10};          // initializer_list → [5, 10]（不是 5 个 10！）

// auto 推导的陷阱
auto il = {1, 2, 3};                 // il 的类型是 std::initializer_list<int>，不是 vector！
```

---

#### 1.5.7 委托构造 — 避免重复代码

**解决的问题**：多个构造函数有共同初始化逻辑，避免代码重复。

```cpp
class Foo {
    int a, b;
    std::string name;
public:
    // 最通用的构造函数
    Foo(int x, int y, std::string n) : a(x), b(y), name(std::move(n)) {
        std::cout << "init: " << name << "\n";
    }

    // 委托到上面的构造函数 —— 避免重复初始化逻辑
    Foo() : Foo(0, 0, "default") {}                  // 默认构造 → 委托
    Foo(int x) : Foo(x, 0, "half") {}                // 单参数 → 委托
    Foo(int x, int y) : Foo(x, y, "full") {}         // 双参数 → 委托

    // 注意：不能委托后再初始化其他成员（所有成员初始化交给被委托的构造）
    // Foo(int x) : a(x), Foo(x, 0) {}    ❌ 编译错误：委托构造必须唯一
};

// 如果有共同逻辑不在初始化列表中的，提取为私有函数
class Bar {
    void init() { /* 通用初始化逻辑 */ }
public:
    Bar() { init(); }
    Bar(int) { init(); /* 额外逻辑 */ }
};
```

**限制**：一旦使用了委托构造，就不能再初始化当前类的其他成员，所有成员初始化都交给被委托的构造函数。

---

#### 1.5.8 std::function — 类型擦除的可调用包装器

**解决的问题**：需要统一存储和传递不同类型的可调用对象（函数指针、lambda、仿函数、bind 表达式）。

```cpp
#include <functional>

// 定义：std::function<返回类型(参数类型...)>
std::function<int(int, int)> f;

// 可以赋值各种可调用对象
f = [](int a, int b) { return a + b; };           // lambda
std::cout << f(3, 4) << std::endl;                // 7

int mul(int a, int b) { return a * b; }
f = mul;                                           // 函数指针
std::cout << f(3, 4) << std::endl;                // 12

struct Functor {
    int operator()(int a, int b) { return a - b; }
};
f = Functor{};                                     // 仿函数
std::cout << f(3, 4) << std::endl;                // -1

// std::bind —— 绑定参数（C++11 常用，C++14 后 lambda 更优）
f = std::bind(mul, 10, std::placeholders::_1);    // 固定第一个参数为 10
std::cout << f(5, 0) << std::endl;                // 50（第二个参数被忽略）

// 实际应用：回调注册
class Button {
    std::function<void()> onClick_;
public:
    void setOnClick(std::function<void()> cb) { onClick_ = std::move(cb); }
    void click() { if (onClick_) onClick_(); }
};

Button btn;
btn.setOnClick([] { std::cout << "clicked!\n"; });
btn.click();   // "clicked!"

// 性能注意：std::function 有类型擦除开销（小对象优化 + 虚函数调用），
// 对于高频回调路径，考虑直接模板参数或函数指针
template<typename F>
void registerCallback(F&& f) { /* 模板避免了类型擦除 */ }
```

---

## 二、C++14：完善增强

**背景**：C++14 是对 C++11 的修修补补，没有新增重大特性，主要完善了类型推导和 constexpr。

### 2.1 泛型 Lambda

```cpp
// C++11：必须指定参数类型
auto add = [](int a, int b) { return a + b; };

// C++14：参数可用 auto，编译器生成模板化的 operator()
auto generic_add = [](auto a, auto b) { return a + b; };
std::cout << generic_add(3, 4) << std::endl;        // int
std::cout << generic_add(3.14, 2.0) << std::endl;   // double

// 实际应用 —— 任意容器打印
auto print = [](const auto& container) {
    for (const auto& x : container) std::cout << x << " ";
};
print(std::vector<int>{1,2,3});       // 1 2 3
print(std::set<std::string>{"a","b"});// a b
```

### 2.2 返回类型推导

```cpp
// 编译器从函数体推导返回类型（之前只能用 auto + trailing return type）
auto make_pair(int a, double b) { return std::pair(a, b); }

// constexpr 函数增强 —— 可以包含局部变量/循环/分支
constexpr int factorial(int n) {       // C++11：只能一条 return 语句
    int result = 1;                    // C++14：允许局部变量
    for (int i = 2; i <= n; ++i)       // C++14：允许循环
        result *= i;
    return result;
}
static_assert(factorial(5) == 120);    // 编译期断言
```

### 2.3 make_unique

```cpp
// C++11 只有 make_shared，C++14 补齐 make_unique
auto p = std::make_unique<int>(42);    // ✅ 推荐写法
std::unique_ptr<int> p2(new int(42));  // ❌ 不推荐（异常安全略差）
```

### 2.4 实用工具增强

#### 2.4.1 std::exchange — 原子替换并返回旧值

**解决的问题**：需要同时读旧值和写新值，传统写法需要临时变量，在多线程语境下需要原子性。

```cpp
#include <utility>
#include <atomic>
#include <vector>
#include <iostream>

// ─── 典型应用1：移动赋值/移动构造（最常用） ───
// 传统写法需要临时变量 + 手动置空，容易漏掉
class Buffer {
    int* data_;
    size_t size_;
public:
    // 用 std::exchange 一行搞定：交换指针 + 源对象置空
    Buffer(Buffer&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))
        , size_(std::exchange(other.size_, 0)) {}

    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;                         // 释放旧资源
            data_ = std::exchange(other.data_, nullptr);
            size_ = std::exchange(other.size_, 0);
        }
        return *this;
    }
};
// 对比手写：std::exchange 确保不会遗漏置空操作，且语义更清晰

// ─── 典型应用2：自旋锁的 try_lock ───
class SpinLock {
    std::atomic<bool> locked_{false};
public:
    bool try_lock() {
        // 尝试把 locked_ 从 false 改为 true，返回旧值
        // 如果旧值是 false → 获取锁成功
        return !std::exchange(locked_, true);
        // 等价于 atomic 的 CAS：
        // bool expected = false;
        // return locked_.compare_exchange_strong(expected, true);
    }
    void unlock() {
        locked_.store(false, std::memory_order_release);
    }
};

SpinLock spin;
if (spin.try_lock()) {
    // 临界区
    spin.unlock();
}

// ─── 典型应用3：工作窃取队列的 pop ───
// 任务队列中，线程从自己的队列尾部取任务
// 使用 std::exchange 安全地取出任务并清空槽位
template<typename T>
class WorkStealQueue {
    std::vector<T> items_;
public:
    // 尝试从队列尾部取一个任务
    bool try_pop(T& result) {
        if (items_.empty()) return false;
        // 取出最后一个元素，同时将其置为默认值（避免重复释放）
        result = std::exchange(items_.back(), T{});
        items_.pop_back();
        return true;
    }
};

// ─── 典型应用4：状态机转移 ───
enum class State { IDLE, RUNNING, STOPPED };

class Machine {
    State state_ = State::IDLE;
public:
    State transition(State new_state) {
        // 原子地：记录旧状态 → 设置新状态
        State old = std::exchange(state_, new_state);
        std::cout << "State: " << static_cast<int>(old)
                  << " → " << static_cast<int>(new_state) << "\n";
        return old;  // 返回旧状态，调用者可决定后续逻辑
    }
};

Machine m;
State prev = m.transition(State::RUNNING);  // 返回 IDLE

// ─── 对比 std::atomic::exchange（线程安全版本） ───
std::atomic<int> atomic_val{10};
int old_atomic = atomic_val.exchange(20);   // 原子操作（线程安全）
// std::exchange 是普通函数，atomic::exchange 是原子操作
// 非原子场景用 std::exchange，多线程场景用 atomic::exchange
```

**性能**：对于基本类型，`std::exchange` 编译后就是一条 `xchg` 指令（或等效的 mov+mov），零开销。相比手写三行赋值，编译器可能无法优化掉临时变量，`std::exchange` 明确表达了语义，编译器更容易生成最优代码。

---

#### 2.4.2 变量模板 — 模板化的变量常量

**解决的问题**：需要为不同类型定义不同的常量值，传统方法用函数模板或类模板特化比较繁琐。

```cpp
#include <limits>
#include <iostream>
#include <type_traits>

// ─── 基础用法：数学常量 ───
// 传统方式：用函数模板（每次调用生成函数，无法直接当常量用）
template<typename T>
constexpr T pi_func() { return T(3.1415926535897932385); }

// C++14 变量模板：直接定义模板化变量（语法更自然，是真正的变量）
template<typename T>
constexpr T pi = T(3.1415926535897932385);

// 使用
auto f = pi<float>;      // 3.1415927（float 精度）
auto d = pi<double>;     // 3.14159265358979（double 精度）
auto i = pi<int>;        // 3（整数截断！注意精度丢失）
static_assert(pi<double> > 3.14);  // 编译期比较

// ─── 典型应用1：物理/数学常量库 ───
template<typename T>
constexpr T speed_of_light = T(299792458);        // m/s

template<typename T>
constexpr T gravitational_constant = T(6.67430e-11);  // N⋅m²/kg²

template<typename T>
constexpr T planck_constant = T(6.62607015e-34);      // J⋅Hz⁻¹

// 单位转换常量
template<typename T>
constexpr T inch_to_mm = T(25.4);                     // 1 英寸 = 25.4 毫米

template<typename T>
constexpr T celsius_to_kelvin = T(273.15);            // 0°C = 273.15K

// 使用：自动匹配精度
double dist_inch = 10.0;
double dist_mm = dist_inch * inch_to_mm<double>;      // 254.0 mm

// ─── 典型应用2：类型萃取别名（简化 traits 使用） ───
template<typename T>
constexpr bool is_integral_v = std::is_integral<T>::value;
// C++17 标准库直接用 std::is_integral_v<T>
// 这就是变量模板的典型标准化应用

// 自定义 trait 变量模板
template<typename T>
constexpr bool is_string_like = std::is_same_v<T, std::string>
                              || std::is_same_v<T, std::string_view>
                              || std::is_same_v<T, const char*>;

static_assert(is_string_like<std::string>);            // ✅
static_assert(!is_string_like<int>);                    // ❌

// ─── 典型应用3：变量模板 + 特化 ───
template<typename T>
constexpr T max_value = std::numeric_limits<T>::max();

// 特化：为特定类型自定义值
template<>
constexpr int max_value<int> = 2147483647;

template<>
constexpr char max_value<char> = 127;

// 使用特化
std::cout << max_value<unsigned long long> << "\n";    // 18446744073709551615
std::cout << max_value<int> << "\n";                    // 2147483647

// ─── 典型应用4：编译期单位转换表 ───
template<typename T>
struct UnitTraits {
    // 角度转弧度
    static constexpr T deg_to_rad = pi<T> / T(180);
    // 弧度转角度
    static constexpr T rad_to_deg = T(180) / pi<T>;
};

double angle_rad = 45.0 * UnitTraits<double>::deg_to_rad;  // π/4

// ─── 变量模板 vs 函数模板对比 ───
// 函数模板：每次调用有函数调用语义（哪怕被内联）
template<typename T>
constexpr T square_func(T x) { return x * x; }
int arr1[square_func(5)];           // 需要 constexpr 函数，OK

// 变量模板：直接是变量，不生成调用
template<typename T>
constexpr T square_v = T(25);       // 固定值，不是表达式
int arr2[square_v<int>];            // 常量值，直接可用
// 注意：变量模板适合常量定义，函数模板适合计算表达式
```

**优势**：
- 相比函数模板：变量模板是真正的变量，可直接当常量使用，不生成函数调用
- 相比类模板 traits：变量模板语法更简洁（`pi<float>` vs `pi<float>::value`）
- C++17 标准库大量采用此模式（`_v` 后缀 traits）

**适用场景**：数学/物理常量库、类型 trait 别名、编译期配置参数。

---

#### 2.4.3 std::integer_sequence — 编译期整数序列

**解决的问题**：需要在编译期生成一个整数序列（如 0,1,2,3,...），用于展开参数包或实现编译期数组。

```cpp
#include <utility>

// 基本概念
// std::integer_sequence<T, Is...>       —— 任意整数类型序列
// std::index_sequence<Is...>             —— 即 integer_sequence<size_t, Is...>
// std::make_index_sequence<N>            —— 生成 0..N-1 序列
// std::index_sequence_for<Ts...>         —— 生成 0..sizeof...(Ts)-1 序列

// 典型应用1：展开 std::tuple
template<typename Tuple, size_t... Is>
void print_tuple_impl(const Tuple& t, std::index_sequence<Is...>) {
    // 折叠表达式展开
    ((std::cout << (Is == 0 ? "" : ", ") << std::get<Is>(t)), ...);
}

template<typename... Args>
void print_tuple(const std::tuple<Args...>& t) {
    std::cout << "(";
    print_tuple_impl(t, std::index_sequence_for<Args...>{});
    std::cout << ")\n";
}
// 使用
auto t = std::make_tuple(42, 3.14, "hello");
print_tuple(t);   // 输出: (42, 3.14, hello)

// 典型应用2：编译期数组初始化
template<typename T, size_t N>
struct CompileTimeArray {
    T data[N];

    template<size_t... Is>
    constexpr CompileTimeArray(std::index_sequence<Is...>, T(*gen)(size_t))
        : data{gen(Is)...} {}   // 展开为 data{gen(0), gen(1), ..., gen(N-1)}
};

// 生成编译期平方数数组
constexpr size_t square(size_t i) { return i * i; }
constexpr auto squares = CompileTimeArray<size_t, 5>{
    std::make_index_sequence<5>{}, square
};
// squares.data = {0, 1, 4, 9, 16}
static_assert(squares.data[3] == 9);

// 典型应用3：参数包索引展开
template<typename... Args>
void process_all(Args&&... args) {
    // 对每个元素加上索引处理
    [&]<size_t... Is>(std::index_sequence<Is...>) {
        ((std::cout << "[" << Is << "]=" << args << "\n"), ...);
    }(std::index_sequence_for<Args...>{});
}
process_all(10, 20, 30);
// 输出:
// [0]=10
// [1]=20
// [2]=30
```

**性能**：所有操作在编译期完成，运行期零开销。

---

## 三、C++17：实用大升级

**背景**：C++17 大幅增强了实用性和表达能力，新增了多个"缺了会不方便"的类型和语法。

### 3.1 结构化绑定（Structured Binding）

**解决的问题**：从 pair/tuple/struct 中拆出元素时不需要显式写类型。

```cpp
// 1. 从 pair 中拆解（map insert 返回值）
std::map<int, std::string> m;
auto [it, inserted] = m.insert({1, "one"});
if (inserted) {
    std::cout << "inserted: " << it->second << std::endl;
}

// 2. 从 tuple 拆解
std::tuple<int, double, char> t = {1, 3.14, 'a'};
auto [a, b, c] = t;    // a=1, b=3.14, c='a'

// 3. 从结构体拆解（成员按声明顺序绑定）
struct Point { double x, y, z; };
Point p = {1.0, 2.0, 3.0};
auto [x, y, z] = p;    // x=1.0, y=2.0, z=3.0

// 4. 配合引用修改
auto& [key, value] = *m.begin();   // key/value 绑定到 map 中的实际元素
value = "modified";                // 直接修改 map 中的值
```

**注意**：结构化绑定不要求类型是 POD，但所有成员必须为 public（或通过 tuple_size 适配）。

### 3.2 if constexpr（编译期条件）

**解决的问题**：模板编程中根据类型不同走不同分支，之前用 SFINAE 或特化很复杂。

```cpp
// 基本用法 —— 编译期分支，未选中分支不实例化
template<typename T>
auto get_value(T t) {
    if constexpr (std::is_pointer_v<T>) {
        return *t;                    // 只有 T 是指针时才编译
    } else if constexpr (std::is_same_v<T, std::string>) {
        return t.size();              // 只有 T 是 string 时才编译
    } else {
        return t;                     // 否则原样返回
    }
}

// 应用：类型安全转换
template<typename To, typename From>
To safe_cast(From&& from) {
    if constexpr (std::is_same_v<std::decay_t<From>, To>) {
        return std::forward<From>(from);     // 同类型直接转发
    } else {
        return static_cast<To>(std::forward<From>(from));  // 否则转型
    }
}
```

**对比 SFINAE**：

| 方案 | 可读性 | 维护性 | 错误信息 |
|:----:|:------:|:------:|:--------:|
| SFINAE + enable_if | ❌ 差 | ❌ 差 | ❌ 难懂 |
| if constexpr | ✅ 清晰 | ✅ 简单 | ✅ 友好 |

### 3.3 std::optional / std::variant / std::any

这三个类型分别解决"可能有值"、"多种类型之一"、"任意类型"的问题。

```cpp
// std::optional —— 可能为空的返回值（替代 out 参数或 nullptr 哨兵）
std::optional<int> parse_int(const std::string& s) {
    try {
        return std::stoi(s);
    } catch (...) {
        return std::nullopt;           // 返回空值
    }
}
auto result = parse_int("42");
if (result) {
    std::cout << *result << std::endl;  // 解引用访问
}
// value_or —— 提供默认值
int val = result.value_or(0);

// std::variant —— 类型安全的联合体（替代 union）
std::variant<int, double, std::string> v;
v = 42;                              // 当前是 int
v = 3.14;                            // 当前是 double
v = "hello"s;                        // 当前是 string

// 访问 variant（编译期检查类型安全）
std::visit([](const auto& x) {
    std::cout << x << std::endl;     // 自动推导当前类型
}, v);

// std::any —— 可持有任意类型（类型擦除，运行时检查）
std::any a = 42;
if (a.type() == typeid(int)) {
    std::cout << std::any_cast<int>(a) << std::endl;
}
a = std::string("hello");
std::cout << std::any_cast<std::string>(a) << std::endl;
```

**选型建议**：

| 类型 | 适用场景 | 性能 |
|:----|:---------|:----:|
| `optional` | 可能为空的返回值 | 小对象栈分配 |
| `variant` | 固定 N 种类型之一 | 栈分配，空间=最大类型 |
| `any` | 任意类型（类型擦除） | 堆分配，有性能开销 |

### 3.4 std::string_view（零拷贝字符串视图）

**解决的问题**：函数接收字符串参数时避免不必要的拷贝和分配。

```cpp
#include <string_view>

// 优化前：每次传入字符串字面量都会构造 std::string
void process1(const std::string& s);       // 隐含一次分配
process1("hello");                          // 临时 string 构造+析构

// 优化后：零拷贝，兼容 char* / string / string_view
void process2(std::string_view sv);         // 不分配内存
process2("hello");                          // 直接指向字面量
process2(std::string("world"));             // 指向 string 内部缓冲区

// 适用场景：函数不需要修改字符串、不需要保证生命周期
// 危险：string_view 不拥有数据，原字符串销毁后访问未定义行为
std::string_view sv;
{
    std::string s = "temporary";
    sv = s;                                 // sv 指向 s 的内部存储
}                                           // s 销毁，sv 悬空！
// std::cout << sv;                         // ❌ 未定义行为
```

### 3.5 std::filesystem（跨平台文件系统）

```cpp
#include <filesystem>
namespace fs = std::filesystem;

// 路径操作
fs::path p = "/home/user/data.txt";
p.filename();           // "data.txt"
p.extension();          // ".txt"
p.parent_path();        // "/home/user"

// 遍历目录
for (const auto& entry : fs::directory_iterator("/home/user")) {
    if (entry.is_regular_file()) {
        std::cout << entry.path() << " size=" << entry.file_size() << std::endl;
    }
}

// 增删改查
fs::create_directories("/tmp/a/b/c");       // 创建多级目录
fs::copy("src.txt", "dst.txt", fs::copy_options::overwrite_existing);
fs::remove_all("/tmp/trash");               // 递归删除
```

### 3.6 语言与算法增强

#### 3.6.1 折叠表达式 — 变参包展开的优雅写法

**解决的问题**：C++11 变参模板的参数包展开方式有限（只能逐个展开或用递归），无法对包做聚合运算。

```cpp
// C++17 引入了四种折叠表达式

// 1. 右折叠（最常用）：(args + ...) → arg0 + (arg1 + (arg2 + ...))
template<typename... Args>
auto sum_right(Args... args) {
    return (args + ...);
}

// 2. 左折叠：(... + args) → ((arg0 + arg1) + arg2) + ...
template<typename... Args>
auto sum_left(Args... args) {
    return (... + args);
}

// 对于加法，左右结果相同；对于减法则不同
// sum_right(1, 2, 3) = 1 + (2 + 3) = 6
// sum_left(1, 2, 3)  = (1 + 2) + 3 = 6（加法交换）
// sub_right(1, 2, 3) = 1 - (2 - 3) = 2
// sub_left(1, 2, 3)  = (1 - 2) - 3 = -4

// 3. 一元右折叠：(args op ...) → arg0 op (arg1 op (...))
template<typename... Args>
bool all_true(Args... args) {
    return (args && ...);         // arg0 && arg1 && arg2 && ...
}

// 4. 一元左折叠：(... op args) → ((arg0 op arg1) op ...) op argN
template<typename... Args>
bool any_true(Args... args) {
    return (... || args);         // arg0 || arg1 || arg2 || ...
}

// 5. 逗号折叠 —— 对每个元素执行操作
template<typename... Args>
void print_all(Args... args) {
    (std::cout << ... << args) << std::endl;   // 折叠插入
}

template<typename... Args>
void print_all_lines(Args... args) {
    ((std::cout << args << "\n"), ...);        // 逗号折叠：逐行打印
}

// 实际应用：编译期遍历 tuple
template<typename Tuple, size_t... Is>
void for_each_impl(Tuple&& t, std::index_sequence<Is...>) {
    (/* 对每个元素执行操作 */, ...);
}

// 检查类型是否全部相同
template<typename T, typename... Rest>
concept all_same = (std::same_as<T, Rest> && ...);
static_assert(all_same<int, int, int>);   // ✅
static_assert(!all_same<int, double>);    // ❌
```

**性能**：折叠表达式纯编译期机制，运行期零开销，和手写展开效果相同。

---

#### 3.6.2 内联变量 — 头文件中安全定义全局变量

**解决的问题**：C++17 之前，在头文件中定义全局变量会导致多重定义链接错误，只能用 `extern` 声明 + 在某个 .cpp 中定义，非常不便。

```cpp
#include <string>
#include <map>

// ─── 典型应用1：头文件库中的全局配置 ───
// C++17 之前：必须在头文件声明，在 .cpp 定义
// header.h
extern int global_counter;
extern const std::string version;
// impl.cpp
int global_counter = 0;
const std::string version = "1.0.0";

// C++17 内联变量：直接在头文件中定义，链接器自动合并
// header.h（多份包含也只生成一份实例）
inline int global_counter = 0;                  // 多个编译单元共享一个变量
inline const std::string version = "1.0.0";     // const 变量默认 inline
inline constexpr double pi = 3.14159;           // constexpr 变量默认 inline

// ─── 典型应用2：类的静态成员 inline 定义 ───
struct Config {
    // C++17 之前：需要类外定义（在某个 .cpp 文件中）
    // static std::string cache_dir;
    // static int timeout_ms;
    
    // C++17：直接在类内定义，不需额外 .cpp
    static inline std::string cache_dir = "/tmp/cache";  // ✅
    static inline int timeout_ms = 5000;                  // ✅
    static inline bool debug_mode = false;
    
    // 静态 constexpr 成员默认 inline（C++17 之前就是 inline 行为）
    static constexpr int max_connections = 100;
};
// C++17 之前需要：
// // config.cpp
// std::string Config::cache_dir = "/tmp/cache";
// int Config::timeout_ms = 5000;

// ─── 典型应用3：模板类中的静态成员 ───
// 模板类静态成员在 C++17 前需要在头文件中小心处理
template<typename T>
struct Registry {
    static inline std::map<std::string, T> items;  // ✅ 每个特化独立实例
    static inline int count = 0;
};
// 使用
Registry<int>::items["answer"] = 42;
Registry<double>::items["pi"] = 3.14;
// int Registry<int>::count = 0;     // C++17 前需要这行，现在不需要

// ─── 典型应用4：头文件唯一标识符（防止 ODR 违反） ───
// 内联函数中的静态局部变量在多编译单元中只有一份
inline int get_unique_id() {
    static int counter = 0;           // 所有编译单元共享同一个 counter
    return counter++;
}
// 如果不用 inline 函数，而是直接在多个 .cpp 中定义非内联变量，会链接错误

// ─── 典型应用5：编译期注册（magic static + inline） ───
struct Plugin {
    const char* name;
    void (*init)();
};

inline auto& get_plugins() {
    static std::vector<Plugin> plugins;  // 所有 TU 共享同一个 vector
    return plugins;
}

// 宏辅助：各 .cpp 文件中自动注册
#define REGISTER_PLUGIN(name, init_fn)                        \
    inline int register_##name() {                             \
        get_plugins().push_back({#name, init_fn});             \
        return 0;                                              \
    }                                                          \
    static int plugin_registered_##name = register_##name();

// config_plugin.cpp
// REGISTER_PLUGIN(config, load_config);
// network_plugin.cpp
// REGISTER_PLUGIN(network, init_network);
// 程序启动时自动注册，无需手动调用初始化函数

// ─── 注意事项 ───
// 1. inline 变量必须在所有编译单元中是相同的定义（否则 UB）
// 2. inline 变量可以有不同的地址吗？不可以——链接器保证唯一实例
// 3. const 全局变量默认内部链接（static），除非用 extern
// 4. constexpr 全局变量默认 inline（C++17 起）
// 5. 不要滥用：真正需要跨编译单元共享时才用 inline
```

**性能**：内联变量在最终可执行文件中只有一份实例，链接器负责合并重复定义，没有运行时开销。对比 `extern + .cpp` 定义，唯一的区别是编译期链接器做了更多工作。

---

#### 3.6.3 并行算法 — STL 算法的多核加速

**解决的问题**：单线程排序/遍历在多核 CPU 上浪费算力，需要简单的方式利用多核，而不必手动管理线程。

##### 一、编译需求与后端线程池

C++17 的 `std::execution::par` 只是一个**执行策略标记**，真正实现并行需要标准库后端线程池的支持。不同实现差异巨大：

| 标准库 | 后端线程池 | 编译需求 | 状态 |
|:-------|:----------|:---------|:-----|
| **MSVC** (VS 2019+) | 内置 Windows 线程池 | `/std:c++17`，无需额外链接 | ✅ 生产就绪 |
| **libstdc++** (GCC) | 依赖 **Intel TBB** | `-ltbb` 链接 TBB，否则退化为串行 | ⚠️ 需安装 TBB |
| **libc++** (Clang) | 依赖 **Intel TBB** | `-ltbb` 链接 TBB，或 `-stdlib=libc++` | ⚠️ 需安装 TBB |
| **Intel Parallel STL** | Intel TBB | 独立库，在 `std::execution` 标准化前的前身 | 已合并入 GCC/Clang |

```bash
# === 编译命令示例 ===

# MSVC —— 直接编译，无需额外库
cl /std:c++17 /EHsc /O2 parallel.cpp

# GCC —— 必须链接 TBB，否则 par 降级为 seq
g++ -std=c++17 -O2 parallel.cpp -ltbb

# Clang —— 同样需要 TBB
clang++ -std=c++17 -O2 parallel.cpp -ltbb

# 安装 TBB（各平台）
# Ubuntu/Debian:  sudo apt install libtbb-dev
# macOS:          brew install tbb
# Windows (vcpkg): ./vcpkg install tbb
# Conan:          conan install tbb/2021.8.0
```

**CMake 配置示例**：

```cmake
cmake_minimum_required(VERSION 3.10)
project(ParallelDemo CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(demo main.cpp)

# 查找并链接 TBB（libstdc++/libc++ 需要）
find_package(TBB REQUIRED)
target_link_libraries(demo PRIVATE TBB::tbb)

# MSVC 不需要 TBB，可以条件判断
if(MSVC)
    # MSVC 内置并行支持
    target_compile_options(demo PRIVATE "/EHsc")
endif()
```

##### 二、TBB 在并行算法中的角色

`std::execution::par` 本身**不创建线程**，它只告诉标准库"可以并行"。标准库将工作提交给 TBB 的 `parallel_for`/`parallel_sort` 等算法，TBB 再通过 **work-stealing 调度器**分配到线程：

```
用户代码                           标准库内部                          TBB 线程池
─────────                        ──────────                        ───────────
std::sort(par, v)  →  libstdc++  →  tbb::parallel_sort()  →  Thread 0 (worker)
                                                          →  Thread 1 (worker)
                                                          →  Thread 2 (worker)
                                                          →  Thread 3 (worker)
                                                              ↑ work-stealing 自动负载均衡
```

```cpp
// 标准库内部（libstdc++ 简化示意）
namespace std {
    template<typename RanIt, typename Cmp>
    void sort(execution::parallel_policy, RanIt first, RanIt last, Cmp comp) {
        if (/* 数据量太小 */) {
            serial_sort(first, last, comp);        // 小数据退化到串行
        } else {
            // 委托给 TBB 的并行排序
            tbb::parallel_sort(first, last, comp);  // 使用 TBB 线程池
        }
    }
}
```

**TBB 线程池控制**：

###### 默认线程数

TBB 默认创建 **CPU 核心数个 worker 线程**（含超线程，即 `std::thread::hardware_concurrency()` 的返回值）：

| 硬件 | `hardware_concurrency()` | TBB 默认 worker 数 |
|:----|:------------------------:|:------------------:|
| 4 核 8 线程（Intel i7） | 8 | 8 |
| 8 核 16 线程（AMD Ryzen 7） | 16 | 16 |
| 2 核 4 线程（低功耗） | 4 | 4 |
| 云端容器（限制 4 vCPU） | 4 | 4 |

> **注意**：超线程（HT）下，TBB 会将逻辑核心视为独立 worker。但 CPU 密集计算时，超线程带来的加速通常只有 10-30%，反而可能因缓存竞争降低性能。**建议 CPU 密集型场景手动限制为物理核心数**。

###### 设置线程数的三种方法

```cpp
// === 方法1：global_control（推荐，运行时动态设置） ===
#include <tbb/global_control.h>

// 限制 TBB 全局并行度（作用域内生效）
{
    tbb::global_control control(
        tbb::global_control::max_allowed_parallelism,
        4  // 限制最多 4 个 worker 线程
    );
    // 在此作用域内，所有 TBB 并行算法（包括 std::execution::par）最多使用 4 线程
    std::sort(std::execution::par, v.begin(), v.end());  // 4 线程
}
// 离开作用域，恢复默认（如 8 线程）
std::sort(std::execution::par, v.begin(), v.end());  // 8 线程

// 可嵌套：内层作用域取最小值
tbb::global_control limit_global(  // 全局限制 max 4
    tbb::global_control::max_allowed_parallelism, 4);
{
    tbb::global_control limit_local(  // 局部限制 max 2（取 min）
        tbb::global_control::max_allowed_parallelism, 2);
    // 实际并行度：min(4, 2) = 2
}

// === 方法2：task_arena（限制特定代码块的线程数） ===
#include <tbb/task_arena.h>

tbb::task_arena arena(2);  // 创建只有 2 个线程的 arena
arena.execute([&] {
    // 在这块区域内，所有 TBB 算法用 2 线程
    std::sort(std::execution::par, v.begin(), v.end());
});
// 其他代码仍用默认线程数

// === 方法3：环境变量（全局固定，最常用） ===
// $ export TBB_NUM_WORKERS=4   # 限制 TBB 为 4 线程
// $ ./my_program               # 所有 TBB 操作只用 4 线程
//
// 其他常用 TBB 环境变量：
// TBB_NUM_WORKERS=4            线程数
// TBB_VERSION=1                打印版本信息
// TBB_VERBOSE=1                打印运行时诊断
```

###### 查询当前线程数

```cpp
#include <tbb/global_control.h>
#include <tbb/info.h>
#include <iostream>

// 方法1：查询当前活跃线程数（动态）
int active_threads = tbb::info::default_concurrency();
std::cout << "TBB active threads: " << active_threads << "\n";

// 方法2：查询硬件并发度
int hw_threads = std::thread::hardware_concurrency();
std::cout << "HW threads: " << hw_threads << "\n";

// 方法3：获取当前限制（如果有）
// 注意：global_control 不提供 getter，需自行记录
```

###### Work-Stealing 自动负载均衡原理

TBB 的核心调度机制是 **work-stealing（工作窃取）**，这是其相比 C++11 原生线程能自动负载均衡的关键：

```
每个 worker 线程维护一个双端队列（deque）：

Thread 0 的队列        Thread 1 的队列        Thread 2 的队列
┌───┬───┬───┬───┐    ┌───┬───┬───┐         ┌───┬───┬───┬───┬───┐
│ T │ T │ T │ T │    │ T │ T │ T │         │ T │ T │ T │ T │ T │
│ 0 │ 1 │ 2 │ 3 │    │ 4 │ 5 │ 6 │         │ 7 │ 8 │ 9 │10 │11 │
└───┴───┴───┴───┘    └───┴───┴───┘         └───┴───┴───┴───┴───┘
  ↑                    ↑                     ↑
  steal from tail     steal from tail       线程 2 忙，队列变长
  (FIFO)              (FIFO)
  
Thread 0 空闲了 → 从 Thread 2 队列尾部偷取任务 T11, T10...
                 （偷旧任务，让 Thread 2 继续处理新任务 T8, T9）

本线程从自己的队列头部取任务（LIFO）→ 缓存局部性好
其他线程从队列尾部偷任务（FIFO）→ 减少竞争
```

**详细流程**：

```
1. 任务拆分（分治）：
   parallel_for(0, 1000, body)
   → TBB 递归拆分：[0,500),[500,1000)
   → 继续拆到粒度足够小（通常 ~256 元素/块）

2. 任务入队：
   每个线程新建的任务 push 到自己队列的尾部（LIFO 端）
   → 线程自己从尾部 pop 任务执行（最近添加的任务，数据还在缓存中）

3. 工作窃取：
   空闲线程随机选择一个 victim（目标线程）
   → 从 victim 队列的头部（FIFO 端）偷取任务
   → 偷取的是最旧的任务（粒度最大，值得偷）
   → 如果 victim 队列为空，换一个 victim 重试

4. 递归窃取：
   被偷取的任务执行时可能继续拆分
   → 新拆的子任务进入偷取者的队列
   → 其他空闲线程可继续偷取这些子任务
   → 形成高效的级联负载均衡
```

**关键优势**：

| 特性 | 效果 |
|:----|:-----|
| **自动负载均衡** | 无需开发者手动分配任务，空闲线程自动偷取繁忙线程的任务 |
| **缓存局部性** | 本线程 LIFO 取任务 → 数据大概率还在 L1/L2 缓存中 |
| **低竞争** | 偷取从队列头部（FIFO），本线程从尾部（LIFO），头尾不冲突 |
| **递归拆分** | 任务粒度自动适配，大任务拆到合适大小，避免过细的调度开销 |
| **无锁** | 队列操作使用 CAS 无锁实现，偷取不阻塞 victim 线程 |

**对比：无 work-stealing 的线程池**：

```cpp
// 静态任务分配（没有 work-stealing）
// 问题：如果某个线程的任务比其他线程慢，其他线程只能空闲等待
Thread 0: ████████████████░░░░░░░░░░░░░░░░░░░░  ← 任务重，还没做完
Thread 1: ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ← 做完了，空闲！
Thread 2: ████████████████░░░░░░░░░░░░░░░░░░░░  ← 任务重
Thread 3: ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ← 空闲！

// TBB work-stealing
Thread 0: ████████████████████████████████████  ← Thread 1 偷走了部分任务
Thread 1: ████████████████████████████████████  ← 偷来任务继续做
Thread 2: ████████████████████████████████████
Thread 3: ████████████████████████████████████  ← 所有线程同时完成
```

**性能影响**：work-stealing 的调度开销约 **每任务 ~50-100ns**（纯用户态，无系统调用），远低于线程上下文切换的 ~1-10μs。

**代码示例①：模拟负载不均 → work-stealing 自动平衡**

```cpp
#include <tbb/parallel_for.h>
#include <tbb/task_arena.h>
#include <iostream>
#include <chrono>

void demo_auto_balance() {
    constexpr int N = 32;
    volatile int dummy[N];
    tbb::parallel_for(0, N, [&](int i) {
        // 故意让后半部分任务更重（模拟负载不均）
        int heavy = (i >= N / 2) ? (i - N / 2 + 1) * 100'000 : 100;
        for (int j = 0; j < heavy; j++) dummy[i] += j;
    });
    // 如果没有 work-stealing，前 16 个轻任务先做完的线程只能空闲
    // TBB work-stealing → 空闲线程自动偷取重任务，所有线程几乎同时完成
}
// 对比：注掉 tbb::parallel_for，用 std::for_each(seq) 执行，耗时差数倍
```

**代码示例②：无 stealing vs 有 stealing 实测对比**

```cpp
#include <tbb/parallel_for.h>
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

void compare_steal_vs_nosteal() {
    const int N = 8;         // 任务数 = 线程数
    const int HEAVY = 50;    // 只有后半部分任务重
    std::vector<long long> workload(N, 1000);
    for (int i = N/2; i < N; i++) workload[i] = HEAVY * 1000;

    // ---- 无 work-stealing：静态划分，每人一个任务 ----
    std::atomic<int> done{0};
    auto t1 = std::chrono::steady_clock::now();
    std::vector<std::thread> threads;
    for (int i = 0; i < N; i++)
        threads.emplace_back([i, &workload, &done] {
            volatile long long s = 0;
            for (long long j = 0; j < workload[i]; j++) s += j;
            done++;
        });
    for (auto& t : threads) t.join();                   // 重任务的线程拖慢整体
    auto t2 = std::chrono::steady_clock::now();
    auto ms_nosteal = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    std::cout << "no-steal: " << ms_nosteal << "ms\n";  // ≈ HEAVY × 1000 耗时

    // ---- TBB work-stealing：空闲线程偷取重任务 ----
    t1 = std::chrono::steady_clock::now();
    tbb::parallel_for(0, N, [&](int i) {
        volatile long long s = 0;
        for (long long j = 0; j < workload[i]; j++) s += j;
    });
    t2 = std::chrono::steady_clock::now();
    auto ms_steal = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    std::cout << "TBB-steal: " << ms_steal << "ms\n";   // 快几倍，重任务被分摊
    // 输出: no-steal: ~500ms, TBB-steal: ~80ms（8 核）
}
```

**代码示例③：任务执行跟踪 → 可视化偷取过程**

```cpp
#include <tbb/task_arena.h>
#include <tbb/parallel_for.h>
#include <iostream>
#include <thread>
#include <vector>

void trace_stealing() {
    tbb::task_arena arena(4);  // 限制 4 线程便于观察
    std::vector<int> who_does_what(16, -1);
    arena.execute([&] {
        tbb::parallel_for(0, 16, [&](int i) {
            who_does_what[i] = tbb::this_task_arena::current_thread_index();
            // 模拟负载不均：任务 8-15 比 0-7 重 10 倍
            volatile long long s = 0;
            int heavy = (i >= 8) ? 500'000 : 50'000;
            for (int j = 0; j < heavy; j++) s += j;
        });
    });
    for (int t = 0; t < 4; t++) {
        std::cout << "Thread " << t << ": ";
        for (int i = 0; i < 16; i++)
            if (who_does_what[i] == t) std::cout << i << " ";
        std::cout << "\n";
    }
    // 输出示例（每次可能不同，取决于调度时机）：
    // Thread 0: 0 1 2 3 8               ← 空闲后偷了重任务 8
    // Thread 1: 4 5 6 9 10              ← 偷了 9 10
    // Thread 2: 7 11 12                 ← 偷了 11 12
    // Thread 3: 13 14 15                ← 初始分配少，偷了 3 个重任务
    // 总效果：重任务 8-15 被分摊到所有线程，而非集中在后 4 个线程
}
```

##### 三、完整示例与性能对比

```cpp
#include <execution>
#include <vector>
#include <algorithm>
#include <iostream>
#include <chrono>
#include <random>

// 生成随机数据
std::vector<int> make_data(size_t n) {
    std::vector<int> v(n);
    std::mt19937 rng(42);
    std::generate(v.begin(), v.end(), rng);
    return v;
}

// 计时模板
template<typename Func>
void bench(const char* name, Func f) {
    auto start = std::chrono::steady_clock::now();
    f();
    auto end = std::chrono::steady_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    std::cout << name << ": " << ms << "ms\n";
}

int main() {
    const size_t N = 50'000'000;  // 5000 万元素
    auto data = make_data(N);

    // 顺序排序
    bench("seq", [&] {
        auto v = data;
        std::sort(std::execution::seq, v.begin(), v.end());
    });

    // 并行排序（需要 TBB 后端支持）
    bench("par", [&] {
        auto v = data;
        std::sort(std::execution::par, v.begin(), v.end());
    });

    // 并行 + 向量化（用于简单变换，排序不支持 par_unseq）
    bench("par_unseq", [&] {
        auto v = data;
        std::transform(std::execution::par_unseq, v.begin(), v.end(), v.begin(),
                       [](int x) { return x * 2; });
    });

    return 0;
}
```

**实测性能参考（8 核 CPU，5000 万整数，GCC 12 + TBB）**：

| 算法 | seq | par | 加速比 |
|:----|:---:|:---:|:------:|
| `sort` | 3850ms | 910ms | **4.2x** |
| `transform` | 320ms | 85ms | **3.8x** |
| `for_each` | 310ms | 80ms | **3.9x** |
| `reduce` | 280ms | 70ms | **4.0x** |

##### 四、执行策略的选择原则

| 执行策略 | 含义 | 线程安全要求 | 适用场景 |
|:--------|:----|:-----------|:---------|
| `seq` | 顺序执行（默认） | 无要求 | 小数据、有数据依赖 |
| `par` | 多线程并行 | 元素间无数据竞争 | 大数据量、元素独立操作 |
| `par_unseq` | 并行 + CPU 向量化 | 元素间无数据竞争 + 无线程局部状态 | 简单算术运算、连续内存访问 |

**注意**：`par_unseq` 要求函数不能调用 `thread_local` 变量和同步原语（因为向量化可能在同一线程交错执行多个迭代），违反导致 UB。

##### 五、常见问题

**Q：链接 TBB 后程序启动变慢？**
TBB 线程池初始化有 ~10-50ms 开销（创建线程、内存分配预热）。对于短生命周期程序影响较大，建议复用进程或使用线程池预热。

**Q：并行算法内部加锁吗？**
标准不要求，但大多数实现无锁（TBB 工作窃取是无锁的）。但你的 lambda 内加锁会降低并行效率。

**Q：如何检测当前实现是否真正并行？**
```cpp
#include <execution>
#include <iostream>

// 检测策略类型
constexpr bool has_parallel =
    std::is_same_v<std::execution::parallel_policy,
                   decltype(std::execution::par)>;

// 运行时检测：如果串行和并行时间一样，说明退化了
// 最简单：strace 看是否创建了多个线程
// $ strace -f ./a.out 2>&1 | grep clone | wc -l
```

**Q：多个并行算法同时执行会导致 CPU 过载吗？**
会。每个 `std::sort(par, ...)` 默认使用全部核心。如果多个线程各自调用并行算法，可考虑：
1. 用 TBB 的 `global_control` 限制全局线程数
2. 用 `seq` 执行非热路径
3. 用自定义线程池 + 任务队列手动调度

##### 六、适用场景速查

| 场景 | 推荐策略 | 预期加速（8核） | 备注 |
|:----|:--------|:---------------|:-----|
| 排序 10^7+ 元素 | `par` | 3~5x | 内存带宽可能成为瓶颈 |
| 元素级变换（*2） | `par_unseq` | 4~6x | 简单算术，向量化收益大 |
| 归约（求和） | `par` + `reduce` | 3~5x | TBB 分块归约再合并 |
| 小数组 < 1000 | `seq` | 无加速 | 线程开销 > 计算时间 |
| 有锁操作 | `seq` | 可能更慢 | 锁竞争让并行不如串行 |

---

#### 3.6.4 数值工具增强 — gcd / lcm / clamp / midpoint

**解决的问题**：常用数学运算之前需自己实现（手写欧几里得算法）或依赖 Boost，标准库补齐了这些基础工具。

```cpp
#include <numeric>
#include <algorithm>
#include <vector>
#include <iostream>
#include <cassert>

// ─── 典型应用1：std::gcd / std::lcm 基础 ───
// 最大公约数 / 最小公倍数
int g = std::gcd(12, 18);        // 6
int l = std::lcm(12, 18);        // 36（12 * 18 / 6）

// 编译期计算
static_assert(std::gcd(12, 18) == 6);
static_assert(std::lcm(12, 18) == 36);

// 支持任意整数类型
long long g2 = std::gcd(10000000000LL, 5000000000LL);  // 5000000000

// ─── 典型应用2：简化分数 + 分数运算 ───
struct Fraction {
    int num, den;
    void simplify() {
        int g = std::gcd(num, den);
        num /= g;
        den /= g;
    }
    Fraction operator+(const Fraction& other) const {
        Fraction result{
            num * other.den + other.num * den,
            den * other.den
        };
        result.simplify();
        return result;
    }
};

Fraction f1{1, 6}, f2{1, 3};
Fraction sum = f1 + f2;           // {3, 6} → simplify → {1, 2}
assert(sum.num == 1 && sum.den == 2);

// ─── 典型应用3：std::clamp 约束值范围 ───
// 将值限制在 [min, max] 区间内
int clamped = std::clamp(value, min_val, max_val);
// 等价于：value < min ? min : (value > max ? max : value)

// 实际应用：进度条、音量控制
float volume = 1.5f;
float safe_volume = std::clamp(volume, 0.0f, 1.0f);    // 1.0（截断到合法范围）

int brightness = -10;
int valid = std::clamp(brightness, 0, 255);             // 0（截断到 0-255）

// clamp 与 std::min/max 组合的对比
int v = 5;
// 传统写法：std::max(min_val, std::min(v, max_val))  ← 难读
// C++17： std::clamp(v, min_val, max_val)              ← 直观

// ─── 典型应用4：std::midpoint 安全中点计算 ───
// (a+b)/2 可能溢出（如 a=20亿, b=20亿, a+b 超出 int 范围）
int mid = std::midpoint(10, 20);               // 15（安全计算）

// 大数安全
long long big = std::midpoint(1LL, 10000000000LL);  // 5000000005
// 如果手写 (a+b)/2 会溢出，midpoint 用 a + (b-a)/2 避免溢出

// 指针/迭代器的中点（C++20）
std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8};
auto mid_it = std::midpoint(vec.begin(), vec.end());  // 指向第 4 个元素
std::cout << *mid_it << "\n";   // 4（vec[3]）

// ─── 典型应用5：综合——日期计算 ───
// 计算两个日期之间有多少个完整的周期
struct Date { int year, month, day; };

// 辅助：判断闰年
constexpr bool is_leap(int year) {
    return (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0);
}

// 计算两个年份之间的闰年个数
int count_leap_years(int y1, int y2) {
    if (y1 > y2) std::swap(y1, y2);
    int leaps = 0;
    for (int y = y1; y <= y2; ++y) {
        if (is_leap(y)) ++leaps;
    }
    return leaps;
}

// 两个日期的天数差（简化版）
int days_between(Date d1, Date d2) {
    int days_per_month[] = {31,28,31,30,31,30,31,31,30,31,30,31};
    int total = 0;
    // 年份差的天数
    for (int y = d1.year; y < d2.year; ++y) {
        total += is_leap(y) ? 366 : 365;
    }
    // 减去 d1 已过的天数
    for (int m = 1; m < d1.month; ++m) {
        total -= days_per_month[m-1] + (m == 2 && is_leap(d1.year) ? 1 : 0);
    }
    total -= d1.day;
    // 加上 d2 已过的天数
    for (int m = 1; m < d2.month; ++m) {
        total += days_per_month[m-1] + (m == 2 && is_leap(d2.year) ? 1 : 0);
    }
    total += d2.day;
    return total;
}
// 使用 std::gcd 简化周期计算
int whole_cycles(Date d1, Date d2, int cycle_days) {
    int total = days_between(d1, d2);
    int g = std::gcd(total, cycle_days);
    return total / cycle_days;
    // 如果 total 和 cycle_days 有公约数，简化后更精确
}
```

**版本注意**：
- `std::gcd` / `std::lcm`：C++17 (`<numeric>`)
- `std::clamp`：C++17 (`<algorithm>`)
- `std::midpoint`：C++20 (`<numeric>`)

**性能**：
- gcd/lcm 使用辗转相除法（欧几里得算法），O(log min(a,b))，编译期可计算
- clamp 编译为两条 cmp + cmov 指令，零开销抽象
- midpoint 使用 `a + (b - a) / 2` 避免溢出，对整数和指针都安全

---

## 四、C++20：革命性更新

**背景**：C++20 是 C++11 之后最大的版本更新，引入了 Concepts/Ranges/Coroutines/Modules 四大重磅特性。

### 4.1 Concepts（概念）—— 模板的约束与检查

**解决的问题**：模板错误信息难读懂、无法对模板参数做语义约束。

```cpp
#include <concepts>

// 基本用法：定义概念
template<typename T>
concept Integral = std::is_integral_v<T>;

// 用法1：template 方式
template<Integral T>
T add(T a, T b) { return a + b; }

// 用法2：auto 方式（更简洁）
auto add(Integral auto a, Integral auto b) { return a + b; }

// 用法3：requires 子句（灵活约束）
template<typename T>
    requires std::integral<T> || std::floating_point<T>
T multiply(T a, T b) { return a * b; }

// 常见标准概念
std::integral<T>       // 整数类型
std::floating_point<T> // 浮点类型
std::same_as<T, U>     // T 和 U 相同类型
std::derived_from<T, Base>  // T 继承自 Base
std::invocable<T, Args...>  // T 可用 Args 调用
std::predicate<T, Args...>  // T 返回 bool

// 复杂概念定义
template<typename T>
concept Container = requires(T t) {
    typename T::value_type;              // 必须有 value_type 类型
    t.begin();                           // 必须有 begin() 方法
    t.end();                             // 必须有 end() 方法
    t.size();                            // 必须有 size() 方法
    requires std::same_as<typename T::value_type, int>;  // value_type 必须是 int
};

// 使用概念的优势：错误信息从 "模板实例化失败" 变为 "约束未满足"，更清晰易懂
```

**对比之前**：

| 方案 | 代码 | 错误信息 |
|:----:|:----|:--------|
| C++98/11 SFINAE | `enable_if<is_integral<T>>` | 不可读 |
| C++20 Concepts | `requires Integral<T>` | `constraints not satisfied` + 原因 |

### 4.2 Ranges（范围）—— 更优雅的集合操作

**解决的问题**：STL 算法（`std::find_if`/`std::transform`）用迭代器对操作不够直观。

```cpp
#include <ranges>
#include <vector>

std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// 管道操作符 —— 链式组合，像函数式编程一样
auto result = v
    | std::views::filter([](int x) { return x % 2 == 0; })   // 过滤偶数
    | std::views::transform([](int x) { return x * x; })     // 平方
    | std::views::take(3);                                    // 取前 3 个
// result 延迟求值，遍历时才计算

// 传统写法 vs Ranges
// 传统：需要临时容器或嵌套
std::vector<int> filtered;
std::copy_if(v.begin(), v.end(), std::back_inserter(filtered),
             [](int x) { return x > 5; });
std::vector<int> squared;
std::transform(filtered.begin(), filtered.end(), std::back_inserter(squared),
               [](int x) { return x * x; });

// Ranges：一行搞定
auto result2 = v | std::views::filter([](int x) { return x > 5; })
                  | std::views::transform([](int x) { return x * x; });

// ranges::sort —— 省去 begin/end 对
std::ranges::sort(v);                    // 等价于 sort(v.begin(), v.end())

// views 是惰性求值 —— 不分配新容器，遍历时实时计算
for (int x : v | std::views::reverse) {  // 反向遍历，无拷贝
    std::cout << x << " ";
}
```

**性能优势**：Ranges 是**零开销抽象**——编译器会将管道展开为等效的循环，没有运行时开销。

### 4.3 三路比较（`<=>` 宇宙飞船运算符）

**解决的问题**：实现所有比较运算符（`<` `>` `<=` `>=` `==` `!=`）需要至少 6 个函数，繁琐且易错。

```cpp
#include <compare>

// 自动生成所有比较运算符
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;   // 一次性生成所有比较
    // 编译器自动生成：== != < > <= >=
};

Point a{1, 2}, b{1, 3};
if (a < b) { /* true, 因为 x 相等时比较 y */ }
if (a == b) { /* false */ }

// 手动控制比较逻辑
struct Person {
    std::string name;
    int age;
    
    // 只比较年龄（忽略名字）
    auto operator<=>(const Person& other) const {
        return age <=> other.age;
    }
    bool operator==(const Person& other) const {
        return age == other.age;      // == 需要单独定义
    }
};

// 返回类型
// std::strong_ordering: 可互换比较（less/equal/greater）
// std::weak_ordering:  不可互换（如不区分大小写的字符串比较）
// std::partial_ordering: 某些值不可比（如 NaN for float）
```

### 4.4 实用工具与语言增强

#### 4.4.1 std::span — 连续内存视图，替代 (T* data, size_t len)

**解决的问题**：C 风格函数用 `(T* data, size_t len)` 传递数组，不安全（长度需手动维护）且不能直接用容器。

```cpp
#include <span>

// 传统 C 风格：容易出错
void legacy_process(int* data, size_t len);  // 调用者需保证 data 至少 len 长度

// C++20 std::span：轻量视图（不拥有数据），包含指针和长度
void process_data(std::span<int> data) {
    for (int& x : data) {      // 支持 range-for
        x *= 2;
    }
    std::cout << "size=" << data.size() << "\n";  // 知道长度
    std::cout << data[0] << "\n";                 // 随机访问
}

// 传入各种连续内存
std::vector<int> v = {1, 2, 3};
process_data(v);                         // ✅ 传入 vector

int arr[] = {4, 5, 6};
process_data(arr);                       // ✅ 传入 C 数组（自动推导大小 3）

process_data({v.data() + 1, 2});         // ✅ 手动构造子范围 [2, 3]

// span 的维度
std::span<int> s1;                       // 动态长度（占用 16 字节：指针+长度）
std::span<int, 3> s2;                    // 固定长度（占用 8 字节：只有指针）

// 子视图
std::span<int> first = s1.first(2);      // 前 2 个元素
std::span<int> last = s1.last(2);        // 后 2 个元素
std::span<int> sub = s1.subspan(1, 2);   // 从索引 1 开始取 2 个

// 作为函数参数时的安全保证
void unsafe(int* p, size_t n);           // 可能越界
void safe(std::span<int> s);             // ✅ 自带边界信息，可用 at() 检查
```

**性能**：`std::span` 是指针+长度的简单封装，传参不拷贝元素，零开销抽象。

**与 string_view 的对比**：

| 特性 | `std::span<T>` | `std::string_view` |
|:----|:-------------:|:------------------:|
| 元素类型 | 任何连续类型 | 仅字符 |
| 可变性 | `span<const T>` 只读，`span<T>` 可变 | 始终只读 |
| 子视图 | `subspan()` / `first()` / `last()` | `substr()` |
| 空终止 | 不保证 | 不保证（C++17 起） |

---

#### 4.4.2 std::format — Python 风格的类型安全格式化

**解决的问题**：C++ 格式化长期缺乏现代方案——`printf` 类型不安全，`iostream` 冗长且慢。

```cpp
#include <format>

// 基本用法：{} 占位符
std::string s = std::format("Hello, {}! You are {} years old.", "Alice", 25);
// s = "Hello, Alice! You are 25 years old."

// 格式化说明
std::format("{:<10}", "left");       // 左对齐（默认右对齐）
std::format("{:>10}", "right");      // 右对齐
std::format("{:^10}", "center");     // 居中
std::format("{:.2f}", 3.14159);      // 保留两位小数 → "3.14"
std::format("{:#x}", 255);           // 十六进制带前缀 → "0xff"
std::format("{:06d}", 42);           // 零填充 → "000042"
std::format("{:b}", 255);            // 二进制 → "11111111"
std::format("{:>10.2f}", 3.14);      // 右对齐宽度10，两位小数 → "      3.14"

// 按位置引用参数
std::format("{1} {0}", "world", "hello");   // "hello world"
std::format("{0} {0} {1}", "a", "b");       // "a a b"

// 自定义类型格式化（通过 std::formatter 特化）
struct Point {
    int x, y;
};

template<>
struct std::formatter<Point> {
    constexpr auto parse(auto& ctx) { return ctx.begin(); }
    auto format(const Point& p, auto& ctx) const {
        return std::format_to(ctx.out(), "({}, {})", p.x, p.y);
    }
};
Point p{3, 4};
std::cout << std::format("point = {}", p) << std::endl;  // "point = (3, 4)"

// std::format_to —— 写入输出迭代器（避免分配临时 string）
std::string buf;
std::format_to(std::back_inserter(buf), "{}", 42);

// 性能对比（粗略，取决于实现）
// printf:    ~100ns
// iostream:  ~200ns
// format:    ~80ns（编译时解析格式字符串可更快）
```

**优势**：类型安全、可读性强、编译期格式字符串检查（C++20 部分编译器支持）、性能优秀。

---

#### 4.4.3 std::jthread — 自动 join 的线程

**解决的问题**：`std::thread` 必须显式 `join()` 或 `detach()`，忘记 join 会 `std::terminate()`，异常路径下容易遗漏。

```cpp
#include <thread>

// C++11 std::thread：必须小心管理生命周期
{
    std::thread t([] { work(); });
    // 如果忘记 join()，析构时 std::terminate()！
    // 如果先 t.join()，后面抛异常也会跳过 join
    t.join();  // 必须保证这行被执行
}

// C++20 std::jthread：析构时自动 join
{
    std::jthread t([] { work(); });
    // 离开作用域，自动调用 t.join()，即使有异常也安全
    // 不需要显式 join()
}

// 中断请求（jthread 独有的特性）
void worker(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        // 正常处理...
        std::this_thread::sleep_for(100ms);
    }
    // 收到中断请求后清理退出
}

std::jthread t(worker);
// ... 一段时间后
t.request_stop();    // 发送中断请求
// t 析构时自动 join

// 对比：std::thread 没有中断机制，需要自己用 atomic 标志
std::atomic<bool> stop{false};
std::thread t2([&] {
    while (!stop) { /* ... */ }
});
stop = true;
t2.join();
```

**注意**：`std::jthread` 的自动 join 在析构时发生，如果线程死循环或死锁，析构会阻塞。此时可用 `t.detach()` 避免阻塞，但需确保资源安全。

---

#### 4.4.4 [[likely]] / [[unlikely]] — 分支预测提示

**解决的问题**：编译器默认假设 if 条件为真，但某些场景异常路径极少触发，给出提示可优化代码布局。

```cpp
// 基本用法：放在语句或标签前
if ([[likely]] success) {
    // 正常路径 —— 编译器将这段代码布局到更可能被取到的位置
    process_normal();
} else [[unlikely]] {
    // 异常路径 —— 放到远离正常 I-Cache 的位置
    handle_error();
}

// 更精细的用法：switch 或循环
switch (code) {
    [[likely]] case 0:  // 最常见
        handle_zero();
        break;
    case 1:
        handle_one();
        break;
    [[unlikely]] default:  // 极少发生
        handle_unknown();
}

// 实际应用：边界检查
void process_value(int* data, size_t index) {
    if ([[unlikely]] !data) {
        return;  // 空指针极不可能
    }
    if ([[unlikely]] index >= MAX_SIZE) {
        return;  // 越界几乎不会发生
    }
    // 正常处理
    data[index] = do_compute();
}

// 性能影响（实际测试数据）
// 正确使用 [[likely]]：分支预测命中率从 95% → 99%+
// 滥用或标错：性能下降（比不用更差）
// 编译器会忽略无效提示，但不一定能识别你的意图
```

**适用时机**：只有在性能关键的热路径上，并且你确定某个分支的概率远大于另一个时，才使用。错误提示会导致性能退化。

---

#### 4.4.5 constexpr 增强 — vector / string 编译期可用

**解决的问题**：C++14 的 constexpr 虽然加强了，但无法使用动态分配（`new`/`vector`/`string`），限制了编译期计算的表达能力。

```cpp
// C++20：std::vector 和 std::string 可在 constexpr 中使用（只要不涉及动态分配运行时行为）

// 编译期排序
constexpr std::vector<int> make_sorted_vec() {
    std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};
    std::sort(v.begin(), v.end());       // ✅ C++20 允许
    return v;
}
constexpr auto sorted_vec = make_sorted_vec();
static_assert(sorted_vec[0] == 1);       // 编译期验证

// 编译期字符串处理
constexpr std::string make_greeting(std::string_view name) {
    std::string result = "Hello, ";
    result += name;
    result += "!";
    return result;
}
constexpr auto greeting = make_greeting("World");
// greeting = "Hello, World!" （编译期确定）

// 编译期算法
constexpr int sum_even(const std::vector<int>& nums) {
    int s = 0;
    for (int x : nums) {
        if (x % 2 == 0) s += x;
    }
    return s;
}
constexpr std::vector<int> numbers = {1, 2, 3, 4, 5, 6};
static_assert(sum_even(numbers) == 12);  // 2 + 4 + 6

// 限制：不能用于 I/O、动态多态（virtual）、new 的定制分配器
// constexpr std::vector 的析构也必须在编译期可执行
```

**实用价值**：将运行时计算移到编译期，减少二进制体积和运行时间。适合配置数据生成、查找表构造、正则表达式预处理等场景。

---

#### 4.4.6 std::source_location — 源码位置信息

**解决的问题**：日志、断言、调试中需要手动写 `__FILE__`、`__LINE__`，宏难以封装且不灵活。

```cpp
#include <source_location>

// 基本用法：作为默认参数，自动捕获调用处位置
void log(const std::string& msg,
         std::source_location loc = std::source_location::current()) {
    std::cout << loc.file_name() << ":" << loc.line()
              << " (" << loc.function_name() << ") "
              << msg << std::endl;
}

// 使用：自动带上调用点信息
void foo() {
    log("entering foo");
    // 输出: main.cpp:42 (foo) entering foo
    //       （42 是 log() 调用处的行号，不是 log 函数内的行号）
}

// 自定义断言
void my_assert(bool cond,
               std::source_location loc = std::source_location::current()) {
    if (!cond) {
        std::cerr << "Assertion failed at "
                  << loc.file_name() << ":" << loc.line()
                  << " in " << loc.function_name() << std::endl;
        std::abort();
    }
}

// 对比传统宏
#define LOG(msg) std::cout << __FILE__ << ":" << __LINE__ << " " << msg << std::endl
// 问题：宏难调试、不能嵌套、作用域污染

// 对比 C++20 source_location
// ✅ 类型安全，不是宏
// ✅ 可作为函数默认参数，调用处自动捕获
// ✅ 可封装在普通函数中
// ✅ 包含 function_name（宏做不到）
```

**注意**：`current()` 必须在参数默认值中使用，如果手动调用 `current()` 只会返回调用 `current()` 自身的位置，而不是上层调用者的位置。

---

#### 4.4.7 Modules（模块） — 告别头文件时代

**解决的问题**：头文件体系的问题——宏污染、重复解析、慢编译、难以封装。

```cpp
// maths.ixx（模块接口文件）
module;                    // 全局模块片段
#include <vector>         // 在模块外包含头文件（不导出给使用者）

export module maths;       // 导出模块 maths

export int add(int a, int b) {        // 导出函数
    return a + b;
}

export class Calculator {             // 导出类
    int base;
public:
    Calculator(int b) : base(b) {}
    int add(int x) const { return base + x; }
};

export namespace utils {              // 导出命名空间
    void helper() { }
}

// 未导出的内容对用户不可见（真正的封装）
void internal_helper() { }  // 不 export，模块外不可见

// main.cpp（使用者）
import maths;               // 导入模块（不需要 #include）

int main() {
    int r = add(1, 2);               // ✅
    Calculator calc(10);
    int r2 = calc.add(5);             // ✅
    // internal_helper();             // ❌ 编译错误：未导出
    return 0;
}
```

**优势**：
| 对比 | 传统头文件 | Modules |
|:----|:---------|:--------|
| 编译速度 | 每个 .cpp 重复解析头文件 | 模块只编译一次，二进制缓存 |
| 封装 | 用 detail 命名空间约定 | 不 export 的符号完全不可见 |
| 宏污染 | #define 会扩散到所有包含者 | 模块内部宏不影响使用者 |
| 依赖顺序 | 头文件包含顺序重要 | 模块导入顺序无关 |
| 构建 | 需处理头文件依赖图 | 模块图由编译器管理 |

**现状**：Modules 是 C++20 最重要的特性之一，但编译器支持仍在完善（GCC 13+ / MSVC 2022 支持较好，Clang 17 部分支持），生产环境建议谨慎评估。

---

#### 4.4.8 Coroutines（协程）— 异步代码的同步写法

C++20 引入了无栈协程原语，详见 [05-协程与异步.md](05-协程与异步.md) 完整专题。

```cpp
// 这里仅作快速速查
// 三个关键字
co_await expr;    // 挂起等待（表达式必须是 Awaitable）
co_yield value;   // 生成一个值（生成器模式）
co_return expr;   // 返回结果

// 协程 vs 函数的核心区别
// 普通函数：调用 → 执行 → 返回（一次性）
// 协程：    调用 → 执行 → 挂起 → 恢复 → ... → 最终返回（可多次挂起恢复）

// 需要实现的 5 个关键点（在 promise_type 中）
struct MyPromise {
    // 1. 返回协程控制对象
    MyTask get_return_object();
    // 2. 初始挂起策略（是否第一次就挂起）
    std::suspend_always initial_suspend();
    // 3. 最终挂起策略
    std::suspend_always final_suspend() noexcept;
    // 4. 返回值处理
    void return_value(T val);
    // 5. 异常处理
    void unhandled_exception();
};
```

**典型应用**：异步网络 I/O（配合 Asio）、生成器/惰性序列、事件驱动编程。

---

## 五、版本迁移速查

| 从版本 | 到版本 | 最重要的新特性 | 需要修改的代码 |
|:------:|:------:|:--------------|:--------------|
| C++98 | C++11 | auto, move, shared_ptr, lambda | 加移动构造/赋值，用 `=delete` 代替 private 拷贝构造 |
| C++11 | C++14 | make_unique, 泛型 lambda, 返回类型推导 | 基本兼容，可增量使用 |
| C++14 | C++17 | optional, variant, string_view, if constexpr, filesystem | 基本兼容 |
| C++17 | C++20 | Concepts, Ranges, Coroutines, format, span, jthread | 基本兼容，编译器需升级到 GCC 10+ / Clang 10+ |

**迁移建议**：
1. 更新编译器到最新版本（GCC 13 / Clang 17 / MSVC 2022 已完全支持 C++20）
2. 代码库从旧标准迁移时，先开 `-std=c++17` 过渡（破坏性变化最少）
3. C++20 的 Coroutines 和 Modules 编译器支持仍有差异，谨慎用于生产
