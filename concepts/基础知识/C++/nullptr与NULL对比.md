# nullptr 与 NULL 对比

See also: [[C++语言特性]], [[C++类型转换]], [[C++指针体系]]

## 1. NULL 的问题

在绝大多数实现中，`NULL` 在 C++ 里通常是这样定义的：

```cpp
#define NULL 0   // 或 0L 等整型常量
```

因此在 C++ 中，`NULL` 本质上是一个**整型常量**，而不是"指针类型"。这会带来几个问题：

- **不是指针类型，而是整数 0**：只是"约定俗成"拿来当空指针用。
- **重载易产生歧义：**

```cpp
void f(int);
void f(char*);

f(NULL);   // 更倾向于匹配 f(int)，而不是你以为的 f(char*)
```

- **模板/泛型代码里语义不清晰：**

```cpp
template<typename T>
void foo(T p);

foo(NULL);   // T 会被推导成 int，而不是某种指针类型
```

## 2. nullptr 带来的改进

C++11 引入了关键字 `nullptr` 及其类型 `std::nullptr_t`：

- **`nullptr` 的类型是 `std::nullptr_t`**：这是专门表示"空指针常量"的独立类型；
- 它可以**隐式转换为任意指针类型（包括成员指针）**，但**不会当作整数**参与重载。

这带来了更好的类型安全和重载行为：

```cpp
void f(int);
void f(char*);

f(nullptr);  // 必然匹配 f(char*)
```

在模板环境中也更符合语义：

```cpp
template<typename T>
void foo(T p);

foo(nullptr);  // T 被推导为 std::nullptr_t，明确就是"空指针"
```

## 3. 小结与实践建议

- **`NULL`**
  - 本质：整型常量（通常是 0 或 0L）
  - 类型：`int` / `long` 等
  - 问题：与整数混淆，在重载/模板中容易产生歧义。

- **`nullptr`**
  - 本质：专用"空指针常量"
  - 类型：`std::nullptr_t`
  - 特点：可以安全转换为任意指针类型，但不会当作整数，用于重载决议和模板推导时语义清晰。

> **实践建议：现代 C++ 代码统一使用 `nullptr`，不再使用 `NULL`。**

[src: raw/ingested/2技术/cpp/C++基础语法手册-补充：为什么需要-`nullptr`（相对-`NULL`-的优势）.md]

## Related Pages
- [[C++语言特性]]
- [[C++类型转换]]
- [[C++指针体系]]
- [[现代C++特性按版本划分]]
