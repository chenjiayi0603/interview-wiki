# C++特有面向对象特性

See also: [[C++面向对象]], [[静多态与动态多态]], [[C++语言特性]]

## 多继承

C++ 支持多继承，一个派生类可以同时继承多个基类。

```cpp
class A {
public:
    void funcA() {}
};

class B {
public:
    void funcB() {}
};

class C : public A, public B {
    // 同时拥有 funcA 和 funcB
};
```

### 菱形继承与虚继承

多继承可能导致菱形继承问题：派生类通过多条路径继承同一个基类，导致基类成员有多份副本。

```cpp
class Base {
public:
    int data;
};

class A : public Base {};
class B : public Base {};
class C : public A, public B {};  // C 中有两份 Base::data
```

**虚继承**解决菱形继承问题，保证基类只有一份副本：

```cpp
class A : virtual public Base {};
class B : virtual public Base {};
class C : public A, public B {};  // C 中只有一份 Base::data
```

## 运算符重载

C++ 允许重载大部分运算符，使自定义类型支持原生运算符语法。

```cpp
class Complex {
    double real, imag;
public:
    Complex(double r, double i) : real(r), imag(i) {}
    
    Complex operator+(const Complex& other) const {
        return Complex(real + other.real, imag + other.imag);
    }
    
    bool operator==(const Complex& other) const {
        return real == other.real && imag == other.imag;
    }
    
    friend std::ostream& operator<<(std::ostream& os, const Complex& c);
};
```

### 可重载与不可重载的运算符

| 可重载 | 不可重载 |
|--------|----------|
| `+ - * / %` | `.` (成员访问) |
| `== != < > <= >=` | `.*` (成员指针访问) |
| `= += -= *= /=` | `::` (作用域解析) |
| `++ --` | `?:` (三元条件) |
| `[] () ->` | `sizeof` |
| `new delete` | `typeid` |
| `, & | ^ ~ !` | `const_cast` 等 |

## 模板（编译期多态）

C++ 模板提供编译期多态能力，详见 [[静多态与动态多态]]。

### 函数模板

```cpp
template<typename T>
T max(T a, T b) {
    return a > b ? a : b;
}
```

### 类模板

```cpp
template<typename T>
class Stack {
    std::vector<T> elements;
public:
    void push(const T& elem) { elements.push_back(elem); }
    T pop() {
        T elem = elements.back();
        elements.pop_back();
        return elem;
    }
};
```

### 模板特化

```cpp
// 全特化
template<>
class Stack<bool> {
    // 针对 bool 的优化实现
};

// 偏特化（仅类模板）
template<typename T>
class Stack<T*> {
    // 针对指针类型的特化
};
```

## RAII 资源管理

RAII（Resource Acquisition Is Initialization）是 C++ 核心资源管理理念：
- 在构造函数中获取资源
- 在析构函数中释放资源
- 利用栈对象生命周期自动管理资源

```cpp
class FileHandle {
    FILE* file;
public:
    FileHandle(const char* path, const char* mode) 
        : file(fopen(path, mode)) {
        if (!file) throw std::runtime_error("Failed to open file");
    }
    
    ~FileHandle() {
        if (file) fclose(file);
    }
    
    // 禁止拷贝，允许移动
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept : file(other.file) {
        other.file = nullptr;
    }
};
```

### RAII 典型应用

- `std::lock_guard` / `std::unique_lock`：自动释放互斥锁
- `std::unique_ptr` / `std::shared_ptr`：自动释放堆内存
- `std::fstream`：自动关闭文件

[src: raw/ingested/2技术/cpp/C++基础语法手册-C++-特有的面向对象编程特性.md]

## Related Pages
- [[C++面向对象]]
- [[静多态与动态多态]]
- [[C++语言特性]]
- [[智能指针]]
- [[设计模式]]
