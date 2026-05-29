# STL 容器与算法

See also: [[C++高频面试问题]], [[STL算法高效用法]]

## STL 概述与六大组件

### 六大组件

| 组件 | 说明 | 典型代表 |
|------|------|----------|
| **容器** | 存储数据 | vector、map、list 等 |
| **算法** | 操作容器 | sort、find、transform 等 |
| **迭代器** | 连接容器和算法 | begin、end、iterator |
| **仿函数** | 重载 `operator()` 的类 | less、greater |
| **适配器** | 修饰接口 | stack、queue、priority_queue |
| **空间配置器** | 内存管理 | 一级/二级配置器 |

### 组件关系图

```
容器 ←→ 迭代器 ←→ 算法
  ↓        ↓        ↓
空间配置器  仿函数   适配器
```

### STL 主流实现

- **SGI STL**：早期经典实现（GCC 2.x/3.x），allocator 用内存池+自由链表
- **GNU libstdc++**：GCC 4.x 后默认，allocator 改为 new/delete
- **MSVC STL（Dinkumware）**：微软实现
- **LLVM libc++**：Clang/Apple 平台
- **EASTL**：游戏专用高性能实现

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## 容器分类

### 序列式容器

| 类型 | 容器 | 底层实现 | 时间复杂度 | 特点 |
|------|------|----------|------------|------|
| **序列** | vector | 动态数组 | 尾部 O(1) 均摊，随机访问 O(1) | 连续内存，尾部高效 |
| | deque | 分段连续存储 | 头尾 O(1)，随机访问 O(1) | 双端队列 |
| | list | 双向链表 | 插入删除 O(1)，访问 O(n) | 任意位置高效 |
| | forward_list | 单向链表 | 插入删除 O(1)，访问 O(n) | C++11，低内存 |
| | array | 固定数组 | 随机访问 O(1) | C++11，定长 |

### 关联式容器
- set / multiset
- map / multimap

### 无序容器
- unordered_set / unordered_map
- unordered_multiset / unordered_multimap

### 容器适配器
- stack
- queue
- priority_queue

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## 容器实现原理

### vector

**核心特性**：动态数组，2 倍扩容，连续内存。

**关键考点**：

1. **扩容机制**：`size() == capacity()` 时触发，常见实现为 2 倍扩容
2. **emplace_back vs push_back**：emplace 接受构造参数，直接构造，避免临时对象
3. **迭代器失效**：扩容后全部失效；删除后需用 `erase` 返回值

```cpp
// 预分配优化
vector<int> vec;
vec.reserve(1000000);

// emplace_back 直接构造
vec.emplace_back(2, 3);  // 最优
```

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

### deque

**核心特性**：分段连续存储，中控器（map）管理多个缓冲区。

**底层结构**：
- 中控器：指针数组，指向各缓冲区
- 缓冲区：固定大小数组（如 512 字节）
- 迭代器含：`_M_cur`、`_M_first`、`_M_last`、`_M_node`

**push_back / push_front 伪代码**：
```cpp
void push_back(const T& val) {
    if (end_offset == chunk_size) {
        if (end_chunk == map_size - 1) expand_map();
        map[++end_chunk] = alloc_chunk();
        end_offset = 0;
    }
    map[end_chunk][end_offset++] = val;
}
void push_front(const T& val) {
    if (start_offset == 0) {
        if (start_chunk == 0) expand_map();
        map[--start_chunk] = alloc_chunk();
        start_offset = chunk_size;
    }
    map[start_chunk][--start_offset] = val;
}
```

**deque vs vector**：deque 头尾 O(1)，vector 头部 O(n)；deque 内存分段，vector 连续。

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

### list

**核心特性**：双向循环链表。

**list::size() 复杂度**：
- GCC C++11 前：O(n)，用 `distance(begin(), end())`
- GCC C++11 后、MSVC：O(1)，用成员变量 `_M_size`

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

### array（C++11）

**核心特性**：固定大小数组包装类，支持迭代器、`at()` 边界检查，不可动态扩容。

```cpp
array<int, 5> arr = {1, 2, 3, 4, 5};
arr.at(1) = 20;  // 带边界检查
```

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

### map/set

- **底层**：红黑树
- **特点**：set 唯一有序，multiset 可重复有序；map 键唯一有序，multimap 键可重复
- **复杂度**：插入/查找/删除 O(log n)
- **注意**：迭代器为 `const_iterator`，不可修改元素
- **节点类型**：`pair<const Key, Value>`，key 不可修改

**map 的 [] 操作符**：不存在时插入并初始化为默认值，查找建议用 `find()` 或 `at()`。

**map vs multimap**：
- map：key 不重复，后写覆盖
- multimap：key 可重复，用 `equal_range(key)` 获取同 key 所有 value

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

### unordered_map/set

- **底层**：哈希表（链地址法）
- **复杂度**：平均 O(1)，最坏 O(n)
- **机制**：哈希函数、桶、负载因子、rehash

**自定义哈希**：需提供 `hash` 特化或哈希函数对象，以及 `operator==`。

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

### 容器适配器

#### stack / queue

- **默认底层**：deque
- **stack**：LIFO，push/pop/top
- **queue**：FIFO，push/pop/front/back
- **实现 FIFO**：推荐 `std::queue`（默认 deque），一般不推荐 list

#### priority_queue

- **底层**：vector + 堆算法（make_heap、push_heap、pop_heap）
- **默认**：大堆（less），小堆用 `greater<int>`
- **操作**：push O(log n)，pop O(log n)，top O(1)

**自定义比较器**：仿函数或 lambda + `decltype`。

```cpp
priority_queue<int, vector<int>, greater<int>> min_pq;  // 小堆
```

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## 迭代器

### 迭代器分类

| 类型 | 支持操作 | 典型容器 |
|------|----------|----------|
| 输入迭代器 | 只读、向前 | istream_iterator |
| 输出迭代器 | 只写、向前 | ostream_iterator |
| 前向迭代器 | 读写、向前 | forward_list |
| 双向迭代器 | 读写、双向 | list、set、map |
| 随机访问迭代器 | 读写、随机访问 | vector、deque、array |

### 迭代器失效

| 容器 | 失效场景 |
|------|----------|
| vector | 插入/删除（可能扩容或移动） |
| deque | 中间插入/删除 |
| list / map / set | 仅被删除的迭代器失效 |

**正确用法**：`it = vec.erase(it);` 使用返回值。

### 迭代器适配器

- 反向迭代器：`rbegin`、`rend`
- 插入迭代器：`back_inserter`、`front_inserter`
- 流迭代器：`ostream_iterator`

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## STL 算法

### 算法分类

**非变序算法**：count、count_if、find、find_if、search、binary_search、lower_bound、upper_bound、equal、mismatch

**变序算法**：for_each、transform、copy、remove、remove_if、unique、replace、sort、stable_sort、partial_sort、partition、stable_partition、fill、generate

### 查找算法

- **find / find_if**：O(n)，线性查找
- **binary_search**：O(log n)，要求已排序，返回 bool
- **lower_bound**：第一个 >= 目标
- **upper_bound**：第一个 > 目标

**注意**：按 key 查找 map 用 `m.find(key)`，不要用 `std::find` 遍历。

### 排序算法

- **sort**：O(n log n)，默认升序
- **stable_sort**：稳定排序
- **partial_sort**：部分排序，前 k 个有序
- **nth_element**：O(n) 平均，确定第 n 大/小元素位置

### 删除算法

- **remove / remove_if**：不真正删除，只移动，需配合 `erase`
- **unique**：去相邻重复，需先排序

```cpp
vec.erase(remove(vec.begin(), vec.end(), 2), vec.end());
sort(vec.begin(), vec.end());
vec.erase(unique(vec.begin(), vec.end()), vec.end());
```

### 算法复杂度速查

| 算法 | 复杂度 |
|------|--------|
| sort | O(n log n) |
| find / count | O(n) |
| binary_search / lower_bound | O(log n) |
| transform | O(n) |
| remove / unique | O(n) |
| nth_element | O(n) 平均 |

### 算法优化技巧

- **nth_element**：O(n) 找第 K 大/小，无需全排序
- **sort + unique + erase**：高效去重
- **accumulate**：替代手写循环求和
- **unordered_set/map**：O(1) 查找、去重
- **vector 优于 list**：随机访问、缓存友好
- **swap 释放 vector 空间**：`vector<int>().swap(v);`

> 更多算法高效用法详见 [[STL算法高效用法]]。

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## 仿函数与函数对象

### 仿函数概念

重载 `operator()` 的类，可像函数一样调用。

### 标准库仿函数

```cpp
plus<int> add;    // add(3, 4) == 7
greater<int> gt;   // gt(5, 3) == true
less<int> lt;      // lt(2, 4) == true
```

### std::function（C++11）

可包装：普通函数、函数指针、lambda、仿函数、成员函数（配合 bind）。

```cpp
function<int(int, int)> f = [](int a, int b) { return a * b; };
```

### 仿函数 vs 函数指针

| 特性 | 仿函数 | 函数指针 |
|------|--------|----------|
| 效率 | 可内联 | 难内联 |
| 状态 | 可有成员 | 无 |
| 类型安全 | 是 | 否 |

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## 空间配置器

### 一级 vs 二级配置器

| 类型 | 处理范围 | 实现方式 |
|------|----------|----------|
| 一级 | > 128 字节 | 直接 malloc/free |
| 二级 | ≤ 128 字节 | 内存池 + 16 个自由链表 |

### 二级配置器原理

- **16 个自由链表**：管理 8、16、24、…、128 字节
- **8 字节对齐**：最小块需能存指针（32 位 4 字节，64 位 8 字节）
- **分配流程**：提升到 8 的倍数 → 查对应链表 → 有则分配，无则从内存池取

### 现代 STL allocator

- **libstdc++ / MSVC / libc++**：默认 `std::allocator` 直接 new/delete，无池化
- **SGI STL**：小对象用内存池+自由链表（GCC 2.x/3.x）
- **不用池化的原因**：通用性、线程安全、易维护、符合标准

### 池化 vs 非池化对比

| 项目 | 非池化（默认） | 池化（SGI） |
|------|----------------|-------------|
| 分配 | new/delete | 内存池+free list |
| 线程安全 | 较好 | 需额外设计 |
| 碎片 | 易产生 | 小对象复用 |
| 泄漏风险 | 低 | 管理不当易泄漏 |

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## 容器选择指南与速查表

### 容器选择决策树

```
需要随机访问？
├─ 是 → vector 或 deque（需头尾操作用 deque）
└─ 否 → 需要任意位置插入删除？→ list

需要有序？
├─ 是 → set/map（红黑树）
└─ 否 → unordered_set/unordered_map（哈希表）

需要优先队列？→ priority_queue
```

### 算法选择

```
排序：sort / stable_sort（需稳定）/ partial_sort（部分）
查找：有序用 binary_search/lower_bound，无序用 find
删除：remove + erase，去重用 sort + unique + erase
```

### 容器底层结构速查

| 容器 | 底层 |
|------|------|
| vector | 动态数组 |
| deque | 分段数组 |
| list | 双向链表 |
| set/map | 红黑树 |
| unordered_* | 哈希表 |
| priority_queue | 堆 + vector |
| stack/queue | 默认 deque |

### sizeof 速查（数组与容器）

| 类型 | sizeof 含义 |
|------|-------------|
| `int arr[5]` | 5×sizeof(int)，元素总大小 |
| `int* p` | 指针大小（8 字节） |
| `vector<int> v` | 仅管理结构（约 24 字节），不含元素 |
| `array<int,5> a` | 5×sizeof(int)，与内置数组一致 |

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## 大厂面试高频考点

### TOP 10 考点

1. **vector 扩容**：2 倍扩容、迭代器失效、reserve 优化
2. **map vs unordered_map**：红黑树 vs 哈希表，有序 vs 无序
3. **迭代器失效**：各容器失效场景
4. **emplace_back vs push_back**：直接构造 vs 拷贝/移动
5. **STL 算法复杂度**：sort、find、binary_search 等
6. **空间配置器**：一级/二级、内存池、自由链表
7. **priority_queue 底层**：堆 + vector
8. **deque 底层**：分段连续、中控器
9. **list::size()**：GCC 旧版 O(n)，新版 O(1)
10. **算法分类**：变序 vs 非变序

[src: raw/ingested/2技术/cpp/STL完整复习手册.md]

## Related Pages
- [[C++高频面试问题]]
- [[STL源码考点]]
- [[C++语言特性]]
- [[智能指针]]
- [[设计模式]]
- [[C++17并行算法]]
- [[STL算法高效用法]]
