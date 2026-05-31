# STL源码考点

> STL 源码级面试考点：vector/deque/list/map/set/unordered_*/priority_queue 内部实现、迭代器 traits、空间配置器、erase-remove 惯用法。

See also: [[STL容器与算法]], [[C++语言特性]], [[vector扩容与noexcept移动构造]]

---

## 一、vector

### 1.1 内部实现
- **底层结构**：连续内存数组，三指针模型（`_M_start`, `_M_finish`, `_M_end_of_storage`）
- **扩容机制**：
  - 当 `size() == capacity()` 时触发扩容
  - 新容量通常是旧容量的 **1.5 倍（GCC）或 2 倍（MSVC）**
  - 扩容步骤：分配新内存 → 搬运旧元素 → 释放旧内存
  - 搬运策略：参见 [[vector扩容与noexcept移动构造]]
- **时间复杂度**：随机访问 O(1)，尾部插入均摊 O(1)，中间插入/删除 O(n)

### 1.2 迭代器失效
| 操作 | 失效情况 |
|------|---------|
| `push_back` / `emplace_back` | 若触发扩容，所有迭代器失效 |
| `insert` / `erase` | 插入点/删除点及之后的所有迭代器失效 |
| `clear` / `pop_back` | 被删除元素的迭代器失效 |

### 1.3 面试高频
- vector 扩容因子为什么是 1.5 或 2？
- `reserve` vs `resize` 区别？
- `emplace_back` vs `push_back` 区别？
- 如何释放 vector 多余容量？（`shrink_to_fit` 或 swap 惯用法）

---

## 二、deque

### 2.1 内部实现
- **底层结构**：分段连续内存（中控器 + 多个缓冲区）
- 中控器是一个指针数组，每个指针指向一块固定大小的缓冲区
- 支持两端 O(1) 插入删除，随机访问 O(1)（但比 vector 稍慢，需两次指针跳转）

### 2.2 迭代器失效
- 两端插入：不影响已有元素，迭代器不失效（但指针/引用仍有效）
- 中间插入/删除：所有迭代器失效

---

## 三、list

### 3.1 内部实现
- **底层结构**：双向循环链表
- 每个节点包含：前驱指针、后继指针、数据
- GCC 中 `std::list` 的 `_List_node` 结构包含 `_M_next`, `_M_prev`, `_M_data`

### 3.2 迭代器失效
- 插入：不失效
- 删除：仅被删除节点的迭代器失效

### 3.3 面试高频
- list 的 `size()` 是 O(1) 还是 O(n)？（C++11 起要求 O(1)，但某些旧实现可能是 O(n)）
- `splice` 操作原理？

---

## 四、map / set

### 4.1 内部实现
- **底层结构**：红黑树（自平衡二叉搜索树）
- 节点包含：左/右/父指针、颜色标记、键值对
- 中序遍历得到有序序列

### 4.2 迭代器失效
- 插入：不失效
- 删除：仅被删除节点的迭代器失效

### 4.3 面试高频
- 红黑树 vs AVL 树？
- `lower_bound` / `upper_bound` / `equal_range` 含义？
- `operator[]` vs `insert` / `emplace` 区别？

---

## 五、unordered_map / unordered_set

### 5.1 内部实现
- **底层结构**：哈希表（开链法，桶 + 链表）
- 负载因子超过 `max_load_factor()` 时触发 rehash
- 桶数量通常是质数

### 5.2 迭代器失效
- 插入：若触发 rehash，所有迭代器失效；否则不失效
- 删除：仅被删除节点的迭代器失效

### 5.3 面试高频
- 如何处理哈希冲突？
- `load_factor` 和 `max_load_factor` 含义？
- 为什么桶数量用质数？

---

## 六、priority_queue

### 6.1 内部实现
- **底层结构**：默认用 `std::vector` 作为容器，`std::make_heap` / `push_heap` / `pop_heap` 维护堆性质
- 默认是大顶堆（`std::less`）

### 6.2 面试高频
- 如何实现小顶堆？（`std::greater` 或自定义比较器）
- 如何自定义比较器？
- `priority_queue` 为什么不支持迭代器？

---

## 七、迭代器 traits

### 7.1 五种迭代器类型
```cpp
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : input_iterator_tag {};
struct bidirectional_iterator_tag : forward_iterator_tag {};
struct random_access_iterator_tag : bidirectional_iterator_tag {};
```

### 7.2 iterator_traits
```cpp
template <typename Iterator>
struct iterator_traits {
    using value_type        = typename Iterator::value_type;
    using difference_type   = typename Iterator::difference_type;
    using pointer           = typename Iterator::pointer;
    using reference         = typename Iterator::reference;
    using iterator_category = typename Iterator::iterator_category;
};

// 对原生指针的特化
template <typename T>
struct iterator_traits<T*> {
    using value_type        = T;
    using difference_type   = ptrdiff_t;
    using pointer           = T*;
    using reference         = T&;
    using iterator_category = random_access_iterator_tag;
};
```

---

## 八、空间配置器（Allocator）

### 8.1 标准分配器
- `std::allocator<T>`：默认分配器，封装 `::operator new` / `::operator delete`
- 提供 `allocate` / `deallocate` / `construct` / `destroy` 接口

### 8.2 SGI STL 二级配置器（面试经典）
- **一级配置器**：直接封装 `malloc` / `free`，处理内存不足时的 `new_handler`
- **二级配置器**：
  - 维护 16 个自由链表，分别管理 8/16/24/.../128 字节的小内存块
  - 大于 128 字节的请求交给一级配置器
  - 内存池机制：一次性向系统申请大块内存，切分后挂到自由链表

---

## 九、erase-remove 惯用法

### 9.1 问题
```cpp
std::vector<int> v = {1, 2, 3, 2, 4, 2, 5};
// 想删除所有值为 2 的元素
```

### 9.2 错误做法
```cpp
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 2)
        it = v.erase(it);  // 正确但 O(n²)
    else
        ++it;
}
```

### 9.3 正确做法（erase-remove）
```cpp
v.erase(std::remove(v.begin(), v.end(), 2), v.end());
// std::remove 将非 2 元素移到前面，返回新逻辑结尾的迭代器
// erase 删除尾部多余元素
// 时间复杂度 O(n)
```

### 9.4 原理
- `std::remove` 不真正删除元素，只是把不需要的元素"覆盖"掉
- 返回的迭代器指向"新结尾"之后的位置
- 配合 `erase` 完成真正的删除

---

## 十、面试高频问题汇总

| 问题 | 答案要点 |
|------|---------|
| vector 扩容机制 | 1.5/2 倍增长，分配-搬运-释放，noexcept 移动优先 |
| vector vs list | 连续内存 vs 链表；随机访问 vs 插入删除；迭代器失效规则不同 |
| map vs unordered_map | 红黑树 vs 哈希表；有序 vs O(1) 平均；内存占用不同 |
| 迭代器失效场景 | vector 扩容全失效，deque 两端插入不失效，list 仅删除节点失效 |
| 空间配置器 | 二级配置器、自由链表、内存池、128 字节分界 |
| erase-remove | O(n) 删除，remove 不真删只移动，配合 erase 完成 |

[src: raw/ingested/2技术/cpp/STL源码常见考点归纳.md]

## Related Pages
- [[STL容器与算法]]
- [[C++语言特性]]
- [[vector扩容与noexcept移动构造]]
