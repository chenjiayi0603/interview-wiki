# C++类型转换

See also: [[C++语言特性]], [[C++面向对象]], [[现代C++特性按版本划分]]

## 1. static_cast — 编译期静态类型转换

**特点：**
- 编译时确定，不进行运行时检查
- 常用于：基本类型转换、void* 与指针转换、类层次上行/下行转换

```cpp
int a = 10, b = 3;
double result = static_cast<double>(a) / b;

struct Base { int m_a; };
struct Sub : Base { int m_b; };
Sub s;
Base *b = static_cast<Base*>(&s);  // ✅ 向上转换安全
Sub *s2 = static_cast<Sub*>(b);    // ⚠️ 向下转换不安全
```

## 2. const_cast — 常量性移除转换

**特点：** 唯一能去掉 const / volatile 修饰符的转换

```cpp
const int a = 10;
const int* p = &a;
int* q = const_cast<int*>(p);
*q = 20;  // ⚠️ 未定义行为
```

## 3. reinterpret_cast — 位级重解释转换（最危险）

**特点：** 仅仅重新解释比特位，不做类型安全检查

```cpp
int *a = new int(10);
double *d = reinterpret_cast<double*>(a);
long long addr = reinterpret_cast<long long>(a);
```

## 4. dynamic_cast — 运行期安全的类层次转换

**特点：**
- 运行时类型检查（RTTI）
- 仅适用于包含虚函数的类层次
- 转换失败返回 `nullptr`

```cpp
class Animal { public: virtual void speak() {} };
class Dog : public Animal {};

Animal* a = new Dog;
Dog* d1 = dynamic_cast<Dog*>(a);  // ✅ OK
Animal* b = new Animal;
Dog* d2 = dynamic_cast<Dog*>(b);  // ❌ 返回 nullptr
```

## 5. 四种 cast 对比表

| 转换符                | 转换时机 | 主要用途              | 安全性   | 可去除 const | 是否依赖虚函数 |
| ------------------ | ---- | ----------------- | ----- | --------- | ------- |
| `static_cast`      | 编译期  | 基本类型 / 类层次        | 中等    | ❌         | ❌       |
| `const_cast`       | 编译期  | 去除 const/volatile | 低（易错） | ✅         | ❌       |
| `reinterpret_cast` | 编译期  | 指针/整数/函数指针        | 极低    | ❌         | ❌       |
| `dynamic_cast`     | 运行期  | 类层次上/下转型          | 高     | ❌         | ✅       |

## 6. C++ 反射与 RTTI

### 6.1 基于 RTTI 的"有限反射"（typeid、dynamic_cast）

> 标准 C++（C++20 及之前）没有 Java/C# 那种完整反射机制，常见做法是使用 RTTI + 注册表模拟"运行时反射"能力。

- `typeid` 是 C++ 标准库中提供的一种利用 RTTI（运行时类型信息）机制的操作，可以在运行时获取对象的实际类型信息（如类型名）。
- RTTI（Run-Time Type Information，运行时类型信息）是 C++ 支持的一种机制，让程序在运行期间能够获知对象的真实类型。RTTI 主要包括两个关键特性：`typeid` 和 `dynamic_cast`。
- 反射指的是"运行时检查与操作类型信息"的能力，标准 C++（到 C++20）并不具备完整的反射，但可以通过 RTTI（即 `typeid`、`dynamic_cast` 等）实现有限的"运行时类型识别"功能。
- 总结关系：`typeid` 是使用 RTTI 的具体工具，用于实现 C++ 中有限的"类型反射"（如获取类型名、类型比对等）。完整的反射则需要更丰富的语言支持，C++ 目前主要依赖 RTTI 相关功能做基础。

```cpp
#include <functional>
#include <iostream>
#include <memory>
#include <string>
#include <typeinfo>
#include <unordered_map>

class Animal {
public:
    virtual ~Animal() = default;
    virtual void speak() const = 0;
};

class Dog : public Animal {
public:
    void speak() const override { std::cout << "Woof\n"; }
};

class Cat : public Animal {
public:
    void speak() const override { std::cout << "Meow\n"; }
};

class AnimalFactory {
public:
    using Creator = std::function<std::unique_ptr<Animal>()>;

    static void registerType(const std::string& name, Creator c) {
        registry()[name] = std::move(c);
    }

    static std::unique_ptr<Animal> create(const std::string& name) {
        auto it = registry().find(name);
        return (it == registry().end()) ? nullptr : it->second();
    }

private:
    static std::unordered_map<std::string, Creator>& registry() {
        static std::unordered_map<std::string, Creator> map;
        return map;
    }
};

int main() {
    AnimalFactory::registerType("Dog", [] { return std::make_unique<Dog>(); });
    AnimalFactory::registerType("Cat", [] { return std::make_unique<Cat>(); });

    auto a = AnimalFactory::create("Cat");
    if (a) {
        std::cout << "RTTI dynamic type: " << typeid(*a).name() << '\n';

        if (auto* dog = dynamic_cast<Dog*>(a.get())) {
            std::cout << "dynamic_cast result: this is Dog\n";
        } else if (auto* cat = dynamic_cast<Cat*>(a.get())) {
            std::cout << "dynamic_cast result: this is Cat\n";
        } else {
            std::cout << "dynamic_cast result: unknown Animal subtype\n";
        }

        a->speak();
    }
}
```

**说明：**
- `typeid(*a).name()` 可拿到运行时类型信息（不同编译器输出格式可能不同）。
- 这不是真正"完整反射"（不能自动枚举成员变量/方法），但在工程里非常实用。
- 若希望"字段级反射"，通常依赖：宏注册、代码生成、第三方库（如 `RTTR`、`Boost.PFR` 等）。

### 6.2 type_name 和 typeid 的区别

本质是 "编译期、手写枚举的类型分支" 和 "由语言/实现提供的类型查询（多为运行时动态类型）" 的差别。

1. **type_name<T>()**：`constexpr` + `if constexpr` + `is_same_v`，在编译期就把分支选死。对象针对模板参数 T，名字是自己手写的表。
2. **typeid**：`typeid(T)` 基本是编译期常量性质；`typeid(*ptr)` 且基类有虚函数时，用于拿到对象的动态类型。`type_info::name()` 通常是实现定义的名字/修饰名，不一定可读、不可移植。

总结：`type_name<T>()` 管"编译期已知的 T"；`typeid(*p)` 管"运行期对象实际类型"（在有多态的前提下）。

### 6.3 C++20 反射现状（type traits、concepts、constexpr）

**结论先说：**
- **C++20 没有进入标准的"原生反射"语法**（不能像 Java/C# 一样直接枚举类型成员）。
- C++20 里常用的是"**静态反射替代思路**"：`type traits`、`concepts`、`constexpr`、模板元编程。

```cpp
#include <concepts>
#include <iostream>
#include <string_view>
#include <type_traits>

struct User {
    int id;
    double score;
};

template <typename T>
concept Numeric = std::is_arithmetic_v<T>;

template <typename T>
constexpr std::string_view type_name() {
    if constexpr (std::is_same_v<T, int>)
        return "int";
    else if constexpr (std::is_same_v<T, double>)
        return "double";
    else if constexpr (std::is_same_v<T, User>)
        return "User";
    else
        return "unknown";
}

template <typename T>
void print_type_info() {
    std::cout << "type = " << type_name<T>()
              << ", is_class = " << std::boolalpha << std::is_class_v<T>
              << ", is_trivial = " << std::is_trivial_v<T> << '\n';
}

int main() {
    print_type_info<int>();
    print_type_info<User>();
    static_assert(Numeric<int>);
    static_assert(!Numeric<User>);
}
```

**怎么理解这段代码：**
- 通过 `std::is_xxx_v` + `if constexpr`，在编译期做"类型检查与分支"，这就是 C++20 常见的"反射替代"。
- 它能回答"这个类型是不是类/算术类型"等问题，但**不能自动遍历 `User` 的字段列表**。
- 真正的标准静态反射仍在提案推进阶段；工程上通常配合库或代码生成来实现字段级反射。

### 6.4 type traits 常用举例（面试高频）

> C++ 的 `type traits`（类型萃取/类型性质工具）是实现"静态反射"（compile-time reflection）的核心基础。通过 type traits，编译器能够在编译期判断、查询并变换类型相关信息。

`type traits` 是 C++ 标准库中"在编译期查询/改造类型"的工具集合，头文件是 `<type_traits>`。

```cpp
#include <iostream>
#include <string>
#include <type_traits>

struct Base {};
struct Derived : Base {};
struct NotDerived {};

template <typename T>
void classify_type() {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "integral type\n";
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "floating-point type\n";
    } else if constexpr (std::is_class_v<T>) {
        std::cout << "class type\n";
    } else {
        std::cout << "other type\n";
    }
}

int main() {
    // 1) 类型判断
    std::cout << std::boolalpha;
    std::cout << "is_pointer<int*> = " << std::is_pointer_v<int*> << '\n';
    std::cout << "is_pointer<int> = " << std::is_pointer_v<int> << '\n';
    std::cout << "is_same<int, long> = " << std::is_same_v<int, long> << '\n';

    // 2) 类型关系
    std::cout << "is_base_of<Base, Derived> = "
              << std::is_base_of_v<Base, Derived> << '\n';
    std::cout << "is_base_of<Base, NotDerived> = "
              << std::is_base_of_v<Base, NotDerived> << '\n';

    // 3) 类型变换（remove / add）
    using A = const int&;
    using B = std::remove_cvref_t<A>;
    using C = std::add_pointer_t<B>;
    std::cout << "B is int? " << std::is_same_v<B, int> << '\n';
    std::cout << "C is int*? " << std::is_same_v<C, int*> << '\n';

    // 4) 编译期分支
    classify_type<int>();
    classify_type<double>();
    classify_type<std::string>();
}
```

**记忆方式：**
- `is_xxx_v<T>`：问"是不是某种类型/关系"（判断）。
- `remove_xxx_t<T>` / `add_xxx_t<T>`：对类型做"变换"（改造）。
- `if constexpr` + traits：把"类型判断"变成"编译期分支"，零运行时开销。

[src: raw/ingested/2技术/cpp/C++基础语法手册-第四章-类型转换-第四章-类型转换.md]

## Related Pages
- [[C++语言特性]]
- [[C++面向对象]]
- [[现代C++特性按版本划分]]
- [[C++特有面向对象特性]]
- [[设计模式]]
