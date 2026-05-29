# vector 扩容与 noexcept 移动构造

> 本文详细分析 `std::vector` 扩容时对元素移动构造与拷贝构造的选择策略，以及 `noexcept` 关键字在其中的关键作用。

See also: [[STL容器与算法]], [[STL源码考点]], [[C++语言特性]], [[C++高频面试问题]]

---

## 一、核心问题

`std::vector` 在扩容时需要将旧元素搬运到新内存空间。搬运时有两种选择：
- **拷贝构造**：安全但可能开销大
- **移动构造**：高效但可能抛异常

标准库的选择策略**不是**普通的重载决议（"看谁匹配度高"），而是容器内部先做策略判断，再决定调用哪种构造。

---

## 二、选择策略（`std::move_if_noexcept` 逻辑）

标准库内部使用类似以下伪代码的逻辑：

```cpp
if (std::is_nothrow_move_constructible<T>::value ||
    !std::is_copy_constructible<T>::value) {
    // 用 T 的移动构造（相当于 std::move(x)）
} else {
    // 用 T 的拷贝构造（相当于按 const T& 传 x）
}
```

**决策表：**

| 移动构造 noexcept | 拷贝构造存在 | 选择 |
|:---:|:---:|:---|
| ✅ | ✅ | **移动构造** |
| ✅ | ❌ | **移动构造** |
| ❌ | ✅ | **拷贝构造**（异常安全优先） |
| ❌ | ❌ | **移动构造**（唯一选择） |

---

## 三、实验验证

### 3.1 代码

```cpp
#include <iostream>
#include <vector>

struct Obj1 {
    Obj1() { std::cout << "Obj1()" << std::endl; }
    Obj1(const Obj1&) { std::cout << "Obj1(Obj1&)" << std::endl; }
    Obj1(Obj1&&) { std::cout << "Obj1(Obj1&&)" << std::endl; }  // 无 noexcept
    Obj1& operator=(const Obj1&) = delete;
    Obj1& operator=(Obj1&&) = delete;
};

struct Obj2 {
    Obj2() { std::cout << "Obj2()" << std::endl; }
    Obj2(const Obj2&) { std::cout << "Obj2(Obj2&)" << std::endl; }
    Obj2(Obj2&&) noexcept { std::cout << "Obj2(Obj2&&)" << std::endl; }  // 有 noexcept
    Obj2& operator=(const Obj2&) = delete;
    Obj2& operator=(Obj2&&) = delete;
};

void testObj1() {
    std::vector<Obj1> v;
    v.reserve(2);
    v.emplace_back();
    v.emplace_back();
    v.emplace_back();  // 触发扩容
}

void testObj2() {
    std::vector<Obj2> v;
    v.reserve(2);
    v.emplace_back();
    v.emplace_back();
    v.emplace_back();  // 触发扩容
}

int main() {
    testObj1();
    testObj2();
}
```

### 3.2 输出

```
Obj1()
Obj1()
Obj1()
Obj1(Obj1&)       // ← 扩容时用拷贝构造
Obj1(Obj1&)       // ← 扩容时用拷贝构造
Obj2()
Obj2()
Obj2()
Obj2(Obj2&&)      // ← 扩容时用移动构造
Obj2(Obj2&&)      // ← 扩容时用移动构造
```

### 3.3 分析

- **Obj1**：移动构造没有 `noexcept`，`is_nothrow_move_constructible<Obj1> == false`，且拷贝构造存在 → vector 选择**拷贝构造**搬运旧元素
- **Obj2**：移动构造有 `noexcept`，`is_nothrow_move_constructible<Obj2> == true` → vector 选择**移动构造**搬运旧元素

---

## 四、为什么这样设计？

### 4.1 强异常安全保证

`std::vector` 扩容时，如果移动构造可能抛异常：
- 旧元素已经被部分移动，数据已损坏
- 无法回滚到扩容前的状态
- 违反强异常安全保证

因此，标准库宁可选择**拷贝构造**（即使更慢），也要保证异常安全。

### 4.2 noexcept 的作用

- **不加 `noexcept`**：vector 库认为"移动可能抛异常"，宁可多拷贝，不冒破坏旧数据的风险
- **加了 `noexcept`**：vector 库认为"移动绝不会抛异常"，放心大量用移动来扩容、重排

---

## 五、与普通重载决议的区别

```cpp
struct A {
    A() = default;
    A(const A&) { std::cout << "copy\n"; }
    A(A&&)      { std::cout << "move\n"; }
};

struct B {
    B(A&& a)       { std::cout << "B(A&&)\n"; }
    B(const A& a)  { std::cout << "B(const A&)\n"; }
};

int main() {
    A a;
    B b1(std::move(a));   // 明确右值引用，B(A&&) 被选中 → 普通重载决议

    const A ca;
    B b2(std::move(ca));  // std::move(const A&) 仍是 const，只能用 B(const A&)
}
```

**输出：**
```
B(A&&)
B(const A&)
```

这里 `B(A&&)` vs `B(const A&)` 是**普通重载决议**：看参数匹配度。

而 vector 扩容时的选择是**容器内部先判断策略，再决定用哪种构造**，两者机制完全不同。

---

## 六、`reserve` 注意事项

- `reserve(n)` 只保证 `capacity() >= n`，实际分配策略依赖实现
- 不同编译器/版本可能一次性分配更多空间
- 因此"第几次 `push_back/emplace_back` 触发扩容"在不同实现中可能不同

---

## 七、面试要点速查

| 问题 | 答案要点 |
|------|---------|
| vector 扩容时用移动还是拷贝？ | 看移动构造是否有 `noexcept`：有则优先移动，无则优先拷贝 |
| 为什么需要 `noexcept`？ | 强异常安全保证：移动抛异常会导致数据损坏无法回滚 |
| `std::move_if_noexcept` 是什么？ | 条件性移动：`noexcept` 移动构造 → 返回右值引用；否则 → 返回 const 左值引用 |
| 如何让自定义类型在 vector 扩容时高效？ | 给移动构造加上 `noexcept` |
| 这和普通重载决议一样吗？ | 不一样。vector 内部先判断策略再选择构造，不是看参数匹配度 |

[src: raw/ingested/2技术/cpp/C++基础语法手册-补充笔记：`std--vector`-扩容与-`noexcept`-移动构造选择.md]

## Related Pages
- [[STL容器与算法]]
- [[STL源码考点]]
- [[C++语言特性]]
- [[C++高频面试问题]]
