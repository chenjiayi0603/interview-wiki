# vector 高效用法

> 本文涵盖 `std::vector` 的高效使用技巧，包括预分配容量、emplace_back、noexcept 移动构造、容量释放、批量赋值、删除元素、避免中间插入、对接 C API 等。

See also: [[STL容器与算法]], [[STL源码考点]], [[C++语言特性]]

---

## 1. 预分配容量：reserve

**避免多次扩容**：已知元素个数时先 `reserve`，可避免 2 倍扩容带来的多次分配与元素搬运。

```cpp
#include <vector>

// ❌ 低效：可能触发多次扩容
std::vector<int> bad;
for (int i = 0; i < 1000000; ++i)
    bad.push_back(i);

// ✅ 高效：一次分配
std::vector<int> good;
good.reserve(1000000);
for (int i = 0; i < 1000000; ++i)
    good.push_back(i);
```

**注意**：`reserve(n)` 只保证 `capacity() >= n`，具体实现可能分配更大（如 2 的幂），所以"第几次 push 触发扩容"依赖实现。

## 2. emplace_back 优于 push_back

**emplace_back** 在容器尾部**就地构造**元素，接受构造参数，避免临时对象和一次拷贝/移动。

```cpp
#include <vector>
#include <string>

struct Item {
    int id;
    std::string name;
    Item(int i, std::string s) : id(i), name(std::move(s)) {}
};

std::vector<Item> v;

// ❌ push_back：先构造临时 Item，再拷贝/移动到容器
v.push_back(Item(1, "a"));

// ✅ emplace_back：直接在容器内构造
v.emplace_back(1, "a");
v.emplace_back(2, "b");
```

对**可移动且移动成本低**的类型，`push_back(T)` 与 `emplace_back(T)` 差异不大；对**多参数构造**或**昂贵拷贝**的类型，emplace 更优。

## 3. noexcept 移动构造与 vector 扩容

vector 扩容时需要**搬运**旧元素。标准库用 `std::move_if_noexcept` 逻辑选择：
- 若 `is_nothrow_move_constructible_v<T>` 为 true，优先用**移动构造**；
- 否则为异常安全，可能用**拷贝构造**。

因此：为元素类型提供 **noexcept 移动构造**能让 vector 扩容时少拷贝、多移动。

```cpp
struct Fast {
    Fast() = default;
    Fast(const Fast&) { /* 拷贝 */ }
    Fast(Fast&&) noexcept { /* 移动 */ }  // 关键：noexcept
};
std::vector<Fast> v;
v.reserve(2);
v.emplace_back();
v.emplace_back();
v.emplace_back();  // 扩容时走移动，不会走拷贝
```

## 4. 释放多余容量：shrink_to_fit / swap 技巧

**shrink_to_fit()**（C++11）：请求实现释放多余容量，**非强制**。

```cpp
std::vector<int> v;
v.reserve(1000);
for (int i = 0; i < 10; ++i) v.push_back(i);
// capacity() 可能仍为 1000
v.shrink_to_fit();  // 请求缩小到 size()，实现可能配合
```

**强制释放容量**：与临时 vector 交换，再丢弃临时对象。

```cpp
std::vector<int> v;
v.reserve(10000);
// ... 使用后 v.size() 很小但 capacity() 很大
std::vector<int>().swap(v);  // v 变为空且 capacity 为 0
```

## 5. 批量赋值与 assign

需要**整体替换**内容时，用 `assign` 可避免先 clear 再多次 push_back 的多次扩容。

```cpp
std::vector<int> v = {1, 2, 3};

// 用迭代器范围赋值
std::vector<int> from = {10, 20, 30, 40};
v.assign(from.begin(), from.end());  // v == {10, 20, 30, 40}

// 用 n 个相同值赋值
v.assign(5, 42);  // v == {42, 42, 42, 42, 42}

// 用初始化列表
v.assign({7, 8, 9});  // v == {7, 8, 9}
```

## 6. 删除元素与迭代器失效

**erase 返回下一有效迭代器**，循环中删除时用返回值，避免未定义行为。

```cpp
std::vector<int> v = {1, 2, 2, 3, 2, 4};
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 2)
        it = v.erase(it);  // 使用返回值
    else
        ++it;
}
// v == {1, 3, 4}
```

**remove + erase 惯用法**：先逻辑"移除"，再一次性擦除尾部。

```cpp
std::vector<int> v = {1, 2, 2, 3, 2, 4};
v.erase(std::remove(v.begin(), v.end(), 2), v.end());
// v == {1, 3, 4}
```

## 7. 避免在循环中多次 insert(pos, x)

在 vector **中间**反复 `insert` 会导致后面元素整体后移，O(n) 每次。若需在中间大量插入，考虑：
- 先收集要插入的内容，再一次性插入；或
- 换用 `list`/`deque` 等结构。

```cpp
// ❌ 低效：每次 insert 都可能移动后面所有元素
std::vector<int> v = {1, 2, 3, 4, 5};
for (int i = 0; i < 1000; ++i)
    v.insert(v.begin() + 2, i);

// ✅ 若必须用 vector：先 reserve，再在合适位置批量处理
std::vector<int> extra(1000);
std::iota(extra.begin(), extra.end(), 0);
v.reserve(v.size() + extra.size());
v.insert(v.begin() + 2, extra.begin(), extra.end());
```

## 8. 用 data() 与 size() 对接 C API

需要连续内存指针时，用 `data()`（C++11）和 `size()`，避免手写 `&v[0]` 对空 vector 的未定义行为。

```cpp
std::vector<int> v = {1, 2, 3};
void c_api(int* p, size_t n);
c_api(v.data(), v.size());  // 空 vector 时 data() 可为 nullptr(C++11 起)
```

[src: raw/ingested/2技术/cpp/STL高效使用细节与示例-一、vector-高效用法.md]

## Related Pages
- [[STL容器与算法]]
- [[STL源码考点]]
- [[C++语言特性]]
- [[C++高频面试问题]]
