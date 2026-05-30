# 现代 C++ 特性按版本划分

> 本文按 `C++11 / C++14 / C++17 / C++20` 划分章节，方便按版本系统复习。

See also: [[C++语言特性]], [[C++高频面试问题]], [[C++进阶知识点]]

---

## 各版本标准的 GCC 支持情况和推出时间

| C++标准 | 支持的 GCC 版本 | 正式发布时间   |
|:-------:|:---------------:|:-------------:|
| C++11   |   GCC 4.8 及以上 <br> (部分特性4.7/4.4起就有，4.8更完善)  | 2011年  |
| C++14   |   GCC 5.0 及以上  | 2014年  |
| C++17   |   GCC 7.1 及以上  | 2017年  |
| C++20   |   GCC 11.1 及以上 | 2020年  |

- *注意：GCC 版本越新，对新标准的支持越全面，某些实验性/部分特性在更早 GCC 上通过 `-std=c++2a`（C++20 前称）等实验参数可用，但完整支持见上表。*

### 实用命令
- 查看默认C++标准（如GCC 9）：`g++ -v`
- 显式选择标准：`g++ -std=c++17 xxx.cpp`
- 推荐 GCC 版本：如需全面体验 C++20，建议 GCC 11 或更高；C++17 则 GCC 7 以上即可。

### 版本特性支持速查
- C++11 通常从 GCC 4.8 开始实用。
- C++14 推荐 GCC 5+。
- C++17 推荐从 GCC 7 起。
- C++20 建议 GCC 11 及以上正式场合使用。

更多细节见 [GCC C++支持官方文档](https://gcc.gnu.org/projects/cxx-status.html)

---

## 1. C++11：现代 C++ 基础

### 1.1 类型系统与声明

- `auto`、`decltype`（如：`auto i = 10; decltype(i) j = 0;`）
- `using` 类型别名（含模板别名）（如：`using Vec = std::vector<int>;`）
- `enum class` 强类型枚举（如：`enum class Color { Red, Green };`）
- `nullptr`（如：`int* p = nullptr;`）

#### 示例：type_traits 判断整型
- `type_traits` 基础元编程（例子：判断一个类型是否为整型：
  ```cpp
  #include <type_traits>
  static_assert(std::is_integral<int>::value, "int 是整型");
  static_assert(!std::is_integral<float>::value, "float 不是整型");
  ```
  ）

#### 示例：C++11 类型与声明基础
```cpp
auto i = 42;
decltype(i) j = 0;
using Vec = std::vector<int>;
enum class Color { Red, Green };
int* p = nullptr;
```

### 1.2 语言语法与控制流

- 范围 `for`
- 列表初始化、统一初始化
- `std::initializer_list`
- `final / override / default / delete / explicit`
- 委托构造、继承构造

#### 示例：范围 for 与列表初始化
```cpp
for (auto& x : vec) { x *= 2; }
std::vector<int> v{1, 2, 3};
```

### 1.3 Lambda 与可调用对象

- 基本 Lambda（捕获、参数、返回值）
- `std::function`、`std::bind`
- `std::ref` / `std::cref`

#### 示例：Lambda 与 std::function
```cpp
auto add = [](int a, int b) { return a + b; };
std::function<int(int, int)> f = add;
```

### 1.4 移动语义与资源管理

- 右值引用 `T&&`
- 移动构造/移动赋值
- `std::move`、`std::forward`（完美转发）
- 智能指针：`std::unique_ptr`、`std::shared_ptr`、`std::weak_ptr`

#### 示例：移动语义与 unique_ptr
```cpp
std::vector<std::string> src{"a", "b"};
std::vector<std::string> dst = std::move(src);
auto up = std::make_unique<int>(42); // C++14 提供 make_unique
```

### 1.5 编译期能力

#### 示例：constexpr 函数
- `constexpr`（初版）例子：
  ```cpp
  constexpr int square(int x) { return x * x; }
  int arr[square(3)]; // 合法，编译期常量
  ```
#### 示例：static_assert 编译期断言
- `static_assert` 例子：
  ```cpp
  static_assert(sizeof(int) == 4, "int 必须为4字节");
  ```
#### 示例：变参模板与折叠表达式
- 变参模板（Variadic Templates）简单例子：
  ```cpp
  template<typename... Args>
  void printAll(Args... args) {
      (std::cout << ... << args) << std::endl; // C++17 折叠表达式
  }
  printAll(1, " + ", 2, " = ", 3);
  ```

#### 示例：constexpr + static_assert 组合
```cpp
constexpr int sq(int x) { return x * x; }
static_assert(sq(5) == 25, "check");
```

### 1.6 并发与内存模型

- 线程库：`std::thread`、`std::mutex`、`std::lock_guard`
#### 示例：std::thread / std::mutex / std::lock_guard
  ```cpp
  #include <iostream>
  #include <thread>
  #include <mutex>
  std::mutex mtx;
  void task(int id) {
      std::lock_guard<std::mutex> lock(mtx);
      std::cout << "Thread " << id << "\n";
  }
  int main() {
      std::thread t1(task, 1);
      std::thread t2(task, 2);
      t1.join(); t2.join();
  }
  ```

- `std::atomic`
#### 示例：atomic 原子操作
  ```cpp
  #include <atomic>
  std::atomic<int> counter{0};
  void inc() { ++counter; }
  ```

- `std::condition_variable`
#### 示例：condition_variable 条件变量
  ```cpp
  #include <condition_variable>
  #include <mutex>
  #include <thread>
  #include <iostream>
  std::mutex m;
  std::condition_variable cv;
  bool ready = false;
  void worker() {
      std::unique_lock<std::mutex> lk(m);
      cv.wait(lk, []{ return ready; });
      std::cout << "Worker started\n";
  }
  int main() {
      std::thread t(worker);
      {
          std::lock_guard<std::mutex> lk(m);
          ready = true;
      }
      cv.notify_one();
      t.join();
  }
  ```

- `std::future` / `std::promise` / `std::async`
#### 示例：future / async
  ```cpp
  #include <future>
  #include <iostream>
  int calc() { return 7; }
  int main() {
      std::future<int> f = std::async(calc);
      std::cout << f.get() << std::endl; // 输出 7
  }
  ```

- `std::call_once`、`thread_local`
#### 示例：call_once / thread_local
  ```cpp
  #include <mutex>
  #include <iostream>
  std::once_flag flag;
  void init() {
      std::call_once(flag, []{ std::cout << "init\n"; });
  }
  thread_local int cnt = 0; // 每个线程自己的变量
  ```

### 1.7 标准库扩展（常用）

#### 示例：常用容器
- 容器：例如
  ```cpp
  std::array<int, 3> arr = {1,2,3};
  std::forward_list<int> fl = {1,2,3};
  std::unordered_map<int, std::string> m = {{1,"a"},{2,"b"}};
  std::tuple<int, double> t{42, 3.14};
  ```
#### 示例：常用算法
- 算法（`all_of` / `any_of` / `none_of`、`copy_if`、`iota` 等）例子：
  ```cpp
  std::vector<int> v{1,2,3,4};
  bool all_pos = std::all_of(v.begin(), v.end(), [](int x){ return x > 0; });
  std::vector<int> even;
  std::copy_if(v.begin(), v.end(), std::back_inserter(even), [](int x){ return x % 2 == 0; });
  std::iota(v.begin(), v.end(), 10); // v={10,11,12,13}
  ```
#### 示例：regex 匹配
- 文本：`std::regex` 例子
  ```cpp
  std::regex re("[a-z]+");
  std::smatch result;
  std::regex_match("abc", result, re); // true
  ```
#### 示例：chrono 与随机数
- 时间与随机：`<chrono>`、`<random>` 例子
  ```cpp
  auto now = std::chrono::system_clock::now();
  std::mt19937 rng(std::random_device{}());
  std::uniform_int_distribution<> dist(1, 6);
  int dice = dist(rng);
  ```
#### 示例：alignas 与 alignof
- 对齐：`alignas` / `alignof` 例子
  ```cpp
  struct alignas(16) S { double x; };
  static_assert(alignof(S) == 16, "");
  ```
#### 示例：自定义字面量 _km
- 自定义字面量 例子（简单）：
  ```cpp
  constexpr long double operator"" _km(long double x) {
      return x * 1000; // 转换为米
  }

  long double d = 3.5_km; // d = 3500
  ```

---

## 2. C++14：对 C++11 的增强与补全

### 2.1 推导与 Lambda 增强

- 函数返回类型推导
#### 示例：decltype(auto) 返回推导
- `decltype(auto)` 例子：
  ```cpp
  decltype(auto) f(int x) { return x; }   // 返回类型自动推断，可以为值/引用
  ```

#### 示例：泛型 Lambda
- 泛型 Lambda（`auto` 参数）例子：
  ```cpp
  auto add = [](auto a, auto b) { return a + b; };
  int x = add(1, 2);     // 3
  double y = add(1.5, 2.3); // 3.8
  ```

- Lambda 初始化捕获

#### 示例：初始化捕获
```cpp
auto id(int x) { return x; }
auto f = [](auto x) { return x + x; };
auto g = [ptr = std::make_unique<int>(7)] { return *ptr; };
```

### 2.2 编译期能力增强

#### 示例：C++14 constexpr 增强
- `constexpr` 放宽（允许更多语句，例子如下）：（中文名：编译期常量表述能力增强/放宽）
  ```cpp
  constexpr int factorial(int n) {
      int res = 1;
      for (int i = 1; i <= n; ++i)
          res *= i;
      return res;
  }

  static_assert(factorial(5) == 120, "check");
  ```
- 变量模板（Variable Templates）

#### 示例：变量模板
```cpp
template <typename T>
constexpr T pi_v = T(3.1415926535897932385L);
```

### 2.3 标准库补全

- `std::make_unique`
- 读写锁：`std::shared_timed_mutex`
- 工具函数：`std::exchange`、`std::integer_sequence`、`std::quoted`
- 字面量增强：二进制字面量、数字分隔符

#### 示例：make_unique 与二进制字面量
```cpp
auto p = std::make_unique<std::string>("hello");
int mask = 0b1010'1100;
```

---

## 3. C++17：实用性大升级

### 3.1 语法与控制流

- 结构化绑定
#### 示例：if/switch 初始化语句
- `if` / `switch` 初始化语句  
  例子（简单）：
  ```cpp
  if (auto it = m.find(key); it != m.end()) {
      std::cout << "value = " << it->second << "\n";
  }

  switch (int x = foo(); x) {
      case 1: std::cout << "x == 1\n"; break;
      case 2: std::cout << "x == 2\n"; break;
      default: std::cout << "x == " << x << "\n"; break;
  }
  ```
#### 示例：if constexpr 编译期分支
- `if constexpr`（编译期分支。例如：
  ```cpp
  template<class T>
  void func(T t) {
      if constexpr (std::is_integral_v<T>)
          std::cout << "int\n";
      else
          std::cout << "not int\n";
  }
  ```
- 内联变量（`inline` 变量）
- 嵌套命名空间简写

#### 示例：结构化绑定与 if 初始化
```cpp
auto [id, name] = std::pair{1, std::string("tom")};
if (auto it = m.find(key); it != m.end()) { /* ... */ }
```

### 3.2 模板与元编程

- 折叠表达式  
  例子（简单）：
  ```cpp
  template<typename... Args>
  auto sum(Args... args) {
      return (args + ...);
  }

  int total = sum(1, 2, 3, 4); // total = 10
  ```
- 类模板参数推导（CTAD）

#### 示例：折叠表达式与 CTAD
```cpp
#include <iostream>
#include <utility>

template <typename... Ts>
auto sum(Ts... xs) { return (xs + ...); }

int main() {
    std::pair p{1, 2.5}; // CTAD: 推导为 std::pair<int, double>
    std::cout << sum(1, 2, 3, 4) << "\n";   // 10
    std::cout << p.first << ", " << p.second << "\n";
}
```

### 3.3 常用库类型

- `std::optional`
- `std::variant`
- `std::any`
- `std::string_view`
- `from_chars` / `to_chars`

### 3.4 并发与算法

- `std::shared_mutex`
#### 示例：shared_mutex 读写锁
    ```cpp
    #include <shared_mutex>
    #include <thread>
    #include <iostream>

    std::shared_mutex mtx;
    int shared_data = 0;

    void reader(int id) {
        std::shared_lock lock(mtx);
        std::cout << "Reader " << id << ": " << shared_data << std::endl;
    }

    void writer() {
        std::unique_lock lock(mtx);
        ++shared_data;
        std::cout << "Writer updated data to " << shared_data << std::endl;
    }
    ```

- `std::scoped_lock`
#### 示例：scoped_lock 同时加锁
    ```cpp
    #include <mutex>
    #include <iostream>

    std::mutex m1, m2;

    void foo() {
        std::scoped_lock lock(m1, m2);
        std::cout << "Both mutexes locked\n";
    }
    ```

- 并行算法（执行策略 `<execution>`）
#### 示例：execution::par 并行算法
    ```cpp
    #include <vector>
    #include <algorithm>
    #include <execution>
    #include <iostream>

    int main() {
        std::vector<int> v{1, 2, 3, 4, 5};
        std::for_each(std::execution::par, v.begin(), v.end(), [](int& x) { x *= 2; });
        for (int x : v) std::cout << x << " ";
    }
    // 输出: 2 4 6 8 10（顺序可不保证）
    ```

- `std::clamp`
#### 示例：clamp 区间裁剪
    ```cpp
    #include <algorithm>
    #include <iostream>

    int main() {
        int x = 15;
        std::cout << std::clamp(x, 0, 10) << std::endl; // 输出10
        std::cout << std::clamp(3, 0, 10) << std::endl; // 输出3
    }
    ```

### 3.5 系统与工程化

- `std::filesystem`
- `__has_include`
- `std::apply`、`std::make_from_tuple`

### 3.6 属性与 Lambda

- `[[nodiscard]]`（C++17 起）
#### 示例：nodiscard
    ```cpp
    [[nodiscard]] int foo() { return 42; }
    int main() {
        foo(); // 编译器可能警告: 忽略了返回值
    }
    ```
- `[[fallthrough]]`（C++17）
#### 示例：fallthrough
    ```cpp
    switch (n) {
        case 1:
            do_something();
            [[fallthrough]];
        case 2:
            do_another();
            break;
    }
    ```
- `constexpr` Lambda
#### 示例：constexpr Lambda
    ```cpp
    constexpr auto add = [](int a, int b) { return a + b; };
    static_assert(add(2, 3) == 5);
    ```
- `[*this]` 捕获对象副本
#### 示例：[*this] 捕获对象副本
    ```cpp
    struct A {
        int x = 1;
        auto getLambda() const {
            return [*this] { return x; };
        }
    };
    ```

---

## 4. C++20：新一代语言能力

### 4.1 泛型编程革命

#### Concepts（概念） 示例：requires 约束模板
- Concepts（概念）  
  > 用于约束泛型模板参数，使其只接受满足特定条件/性质的类型，相当于"类型的编译期筛选器"。类似于泛型的"断言"，提高泛型代码的类型安全和可读性。

- `requires` 子句  
> 作用：Concepts和requires约束都是**编译期**（compile-time）机制。条件判断和类型检查都发生在编译阶段，
> 只有在参数类型满足指定性质时，相关模板才会被实例化或参与重载；否则编译器在编译时报错。不会产生运行时代码，
> 也不会影响运行时性能，是一种纯粹的编译期类型约束工具。

```cpp
template <typename T>
requires requires(T a, T b) { a + b; }
T add(T a, T b) { return a + b; }

int main() {
    int x = 10, y = 20;
    double a = 1.5, b = 2.5;
    std::cout << add(x, y) << std::endl;     // 输出 30
    std::cout << add(a, b) << std::endl;     // 输出 4
    return 0;
}
```

你也可以写成更清晰的两步：

```cpp
#include <concepts>
#include <iostream>
#include <string>

template <typename T>
concept Addable = requires(T a, T b) { a + b; };

template <typename T>
requires Addable<T>
T add(T a, T b) { return a + b; }

struct NoPlus {};

int main() {
    std::cout << add(10, 20) << '\n';                       // 30
    std::cout << add(1.5, 2.5) << '\n';                     // 4
    std::cout << add(std::string("ab"), std::string("cd")) << '\n'; // abcd
}
```
    
#### Concepts（概念） 示例：requires 尾置写法

    ```cpp
    template <typename T>
    T add(T a, T b) requires requires(T x, T y) { x + y; } {
        return a + b;
    }
    ```

#### Concepts（概念） 示例：std::integral 概念约束
```cpp
#include <concepts>
template <std::integral T>
T add(T a, T b) { return a + b; }
```

### 4.2 编译期与初始化控制

- `consteval` 示例（强制编译期求值，仅能在编译期调用）
    ```cpp
    consteval int square(int x) { return x * x; }
    int val = square(5); // 编译期计算，val=25
    ```

- `constinit` 示例（保证静态变量以常量表达式初始化，防止静态初始化顺序问题）
    ```cpp
    constinit int global_var = 42; // 必须能编译期初始化
    ```

- `constexpr` 能力继续增强（可用于容器与更多场景）
    ```cpp
    constexpr int sum() {
        std::vector<int> v = {1,2,3};
        int s = 0;
        for (auto x : v) s += x;
        return s;
    }
    static_assert(sum() == 6); // 编译期检查
    ```


### 4.3 并发升级

- `std::jthread`
- `std::stop_token`
#### 示例：latch / barrier / semaphore
- 同步原语：`latch` / `barrier` / `semaphore` 例子（简单）
    ```cpp
    #include <iostream>
    #include <thread>
    #include <latch>
    #include <barrier>
    #include <semaphore>

    void latch_example() {
        std::latch latch(3);
        auto worker = [&latch](int id){
            std::cout << "worker " << id << " done\n";
            latch.count_down();
        };
        std::thread t1(worker, 1), t2(worker, 2), t3(worker, 3);
        latch.wait();
        std::cout << "All workers done (latch)\n";
        t1.join(); t2.join(); t3.join();
    }

    void barrier_example() {
        std::barrier barr(3);
        auto worker = [&barr](int id){
            std::cout << "worker " << id << " at barrier\n";
            barr.arrive_and_wait();
            std::cout << "worker " << id << " passed barrier\n";
        };
        std::thread t1(worker, 1), t2(worker, 2), t3(worker, 3);
        t1.join(); t2.join(); t3.join();
    }

    void semaphore_example() {
        std::counting_semaphore<2> sem(2);
        auto worker = [&sem](int id){
            sem.acquire();
            std::cout << "worker " << id << " acquired semaphore\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(500));
            sem.release();
        };
        std::thread t1(worker, 1), t2(worker, 2), t3(worker, 3);
        t1.join(); t2.join(); t3.join();
    }
    ```
- `std::span`

#### 示例：jthread + stop_token + span
```cpp
#include <iostream>
#include <thread>
#include <stop_token>
#include <span>
#include <vector>

void worker(std::stop_token st, std::span<const int> data) {
    int sum = 0;
    for (int x : data) {
        if (st.stop_requested()) return;
        sum += x;
    }
    std::cout << "sum = " << sum << "\n";
}

int main() {
    std::vector<int> v{1, 2, 3, 4, 5};
    std::jthread t(worker, std::span<const int>(v));
    t.request_stop();
}
```

### 4.4 范围与算法新范式

- Ranges（`<ranges>`）
- 管道式视图：`views::filter` / `transform` / `take` 等

#### 示例：ranges 管道
```cpp
auto even = vec
    | std::views::filter([](int x) { return x % 2 == 0; })
    | std::views::transform([](int x) { return x * x; });
```

### 4.5 语法与表达能力

#### 示例：<=> 三路比较
- 三路比较运算符 `<=>` 例子：
  ```cpp
  #include <compare>
  struct Point {
      int x, y;
      auto operator<=>(const Point&) const = default;
  };
  Point a{1,2}, b{1,3};
  bool less = (a < b);      // true
  bool equal = (a == b);    // false
  ```

#### 示例：指定初始化器
- 指定初始化器 例子：
  ```cpp
  struct S { int x; double y; };
  S s{ .y = 3.14, .x = 10 }; // 指定成员初始化
  ```

#### 示例：using enum
- `using enum` 例子：
  ```cpp
  enum class Color { Red, Green, Blue };
  void print_color(Color c) {
      using enum Color;
      switch (c) {
          case Red: std::cout << "Red\n"; break;
          case Green: std::cout << "Green\n"; break;
          case Blue: std::cout << "Blue\n"; break;
      }
  }
  ```

#### 示例：Lambda 模板参数
- Lambda 模板参数列表（`[]<typename T>(T x){...}`）例子：
  ```cpp
  auto f = []<typename T>(T x) { return x + x; };
  int n = f(3);        // n = 6
  std::string s = f(std::string("Hi")); // s = "HiHi"
  ```

#### 示例：likely / unlikely
- `[[likely]]` / `[[unlikely]]` 例子：
  ```cpp
  int guess(int n) {
      if (n == 42) [[likely]]
          return 1;
      else [[unlikely]]
          return 0;
  }
  ```


### 4.6 文本、调试与工程能力

#### 示例：std::format
- `std::format` 例子：
  ```cpp
  #include <format>
  std::string s = std::format("id={}, score={:.2f}", 7, 99.5);
  std::cout << s << std::endl; // 输出: id=7, score=99.50
  ```

#### 示例：std::source_location
- `std::source_location` 例子：
  ```cpp
  #include <iostream>
  #include <source_location>

  void log(const std::string& msg, const std::source_location& loc = std::source_location::current()) {
      std::cout << loc.file_name() << ":" << loc.line() << " " << msg << std::endl;
  }

  int main() {
      log("Something happened");
  }
  ```
- 协程（Coroutines）
- 模块（Modules）

#### 示例：format 简写
```cpp
#include <format>
std::string s = std::format("id={}, score={:.2f}", 7, 99.5);
```

---

## 5. 版本选型建议

- **只要"现代 C++ 入门"**：先打牢 `C++11`（移动语义 + 智能指针 + 并发基础）
- **工程稳定性优先**：`C++17` 是当前最稳妥主流
- **需要新范式与表达力**：升级到 `C++20`（Concepts / Ranges / 协程）
- **面试准备顺序**：`11 -> 14 -> 17 -> 20`，按"语言基础 -> 库能力 -> 新范式"逐层推进

---

## 附：与原手册的对应关系（快速索引）

- 原手册按"功能分类"，本手册按"标准版本"重排
- `C++11`：覆盖原文中绝大多数基础语法、并发、容器算法基础条目
- `C++14`：覆盖原文中推导增强、泛型 Lambda、`make_unique`、变量模板等
- `C++17`：覆盖原文中结构化绑定、`if constexpr`、`optional/variant/any`、`filesystem`、并行算法等
- `C++20`：覆盖原文中 Concepts、Ranges、`<=>`、`format`、`jthread`、协程、模块等

[src: raw/ingested/2技术/cpp/现代C++特性完整复习手册-按版本划分.md]

## Related Pages
- [[C++语言特性]]
- [[C++高频面试问题]]
- [[C++进阶知识点]]
- [[C++20协程]]
- [[C++多线程与并发]]
- [[STL容器与算法]]
