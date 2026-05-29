# C++进阶知识点

See also: [[C++语言特性]], [[C++高频面试问题]], [[C++可变参数函数]]

## 模板元编程

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

## 异常处理

### 异常安全保证
- **基本保证**：不泄漏资源，对象处于有效状态
- **强保证**：操作要么成功，要么回滚到操作前状态
- **不抛出保证**：操作保证不抛出异常（`noexcept`）

### noexcept 关键字（C++11）
```cpp
void func() noexcept;                    // 保证不抛出异常
void func() noexcept(condition);         // 条件性 noexcept
```

### RAII 与异常安全
使用 RAII 管理资源，确保异常发生时资源被正确释放。
```cpp
class FileGuard {
    FILE* f;
public:
    FileGuard(const char* path) : f(fopen(path, "r")) {}
    ~FileGuard() { if (f) fclose(f); }
};
```

## 编译器优化

### RVO / NRVO（返回值优化）
编译器优化，避免临时对象的拷贝/移动。
```cpp
MyClass create() {
    return MyClass();  // RVO
}
```

### 零开销原则
C++ 设计哲学：你不用的东西，不需要为之付出代价。

### PIMPL 惯用法
隐藏实现细节，减少编译依赖。
```cpp
// header
class Widget {
    struct Impl;
    std::unique_ptr<Impl> pImpl;
public:
    Widget();
    ~Widget();
    void doSomething();
};

// source
struct Widget::Impl { /* 实际数据 */ };
```

## ABI 兼容性

### ABI（Application Binary Interface，应用二进制接口）
指"二进制层面如何协作的规则"，包括：函数如何传参/返回值、对象的内存布局（成员偏移、对齐、虚表位置）、符号命名方式、异常和栈展开等。

一句话：**ABI = 编译好之后，库与程序之间约定的"二进制协议"**。

### 模板多态的缺点与 ABI 兼容性

#### 模板多态的典型缺点

- **编译时间长、二进制体积大（code bloat）**：
  每一组不同模板参数都会实例化出一份独立代码；项目中模板参数类型组合一多，编译时间和最终可执行文件体积都会明显增大。
- **错误信息晦涩难读**：
  模板嵌套、SFINAE、`enable_if` 等一起使用时，一旦出错，编译器报错信息极长且层层展开，调试成本高。
- **接口契约不够显式**：
  模板本质是"鸭子类型"，只要满足用到的表达式就能通过编译，没有统一的抽象基类约束，很多约束只能靠文档和约定，维护难度偏大。
- **编译期逻辑复杂，可读性差**：
  大量使用元编程、`constexpr`、类型萃取等把逻辑推到编译期，虽高效，但对阅读者要求高，不利于新同事快速上手。
- **ABI 稳定性差**：
  模板接口通常在头文件中实现，内部实现细节一旦调整，很容易影响到已经编译好的代码与新库之间的二进制兼容性。

#### ABI 与模板为什么容易不兼容

- **模板实例是"按实现生成代码"的**：
  模板函数/类在每个使用点根据"当时看到的头文件实现"生成具体实例代码；
  如果仅升级动态库（或静态库），而**应用程序没有重新用新头文件一起编译**：
  - 应用程序中，某些模板类型（如 `Wrapper<A>`）的布局、成员偏移等是"旧版本理解"的；
  - 新库中，这些模板的实现可能因为模板内部或依赖类型成员调整而具有"新布局"；
  - 运行时双方对同一内存的"解释"不一致，就会出现读写错位、未定义行为甚至崩溃。

- **典型风险场景总结**：
  - **只重新编译库，不重编应用**：
    模板实现或依赖类型的布局发生变化 → 库里的布局是新的，应用里按旧布局访问 → **ABI 不兼容，高风险**。
  - **库和应用一起、统一使用新头文件重新编译**：
    所有翻译单元对模板类型与函数的理解完全一致 → 布局一致，一般不会因为模板变化产生 ABI 问题。

- **实践建议**：
  - 模板接口适合作为"源码级接口"（提供头文件给调用方一起编译），**不适合作为长期稳定的二进制 ABI 接口**；
  - 升级使用大量模板的库版本时，通常应**整体重编所有依赖该库的代码**，而不仅仅是替换 `.so / .dll` 文件。

[src: raw/ingested/2技术/cpp/C++基础语法手册-第八章-可变参数函数.md]

## Related Pages
- [[C++语言特性]]
- [[C++高频面试问题]]
- [[C++可变参数函数]]
- [[静多态与动态多态]]
- [[C++面向对象]]
- [[C++特有面向对象特性]]
