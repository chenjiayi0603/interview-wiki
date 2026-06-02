# STL 标准库

> 容器、迭代器、算法、空间配置器 —— 面试与工程核心知识。

---

## 一、六大组件

| 组件 | 说明 | 典型代表 |
|------|------|----------|
| **容器** | 存储数据 | vector、map、list |
| **算法** | 操作容器 | sort、find、transform |
| **迭代器** | 连接容器和算法 | begin/end、iterator_traits |
| **仿函数** | 重载 `operator()` | less、greater |
| **适配器** | 修饰接口 | stack、queue、priority_queue |
| **空间配置器** | 内存管理 | std::allocator |

---

## 二、容器

### 2.1 容器分类与底层

| 类型 | 容器 | 底层实现 | 关键复杂度 |
|------|------|----------|-----------|
| **序列** | vector | 动态数组 | 尾插 O(1) 均摊，随机访问 O(1) |
| | deque | 分段连续数组 | 头尾 O(1)，随机访问 O(1) |
| | list | 双向链表 | 插入删除 O(1)，访问 O(n) |
| | forward_list | 单向链表 | C++11，极低内存 |
| | array | 固定数组 | C++11，定长 |
| **关联** | set/map | 红黑树 | 插入/查找/删除 O(log n) |
| | multiset/multimap | 红黑树 | 允许重复 key |
| **无序** | unordered_set/map | 哈希表 | 平均 O(1)，最坏 O(n) |
| **适配器** | stack/queue | 默认 deque | LIFO/FIFO |
| | priority_queue | vector + 堆 | push/pop O(log n) |

### 2.2 vector 深度解析

**三指针布局**：`_M_start`、`_M_finish`、`_M_end_of_storage`

```cpp
size()     = _M_finish - _M_start
capacity() = _M_end_of_storage - _M_start
```

**扩容机制**：`size() == capacity()` 时触发，常见 2 倍扩容。
- 申请新内存 → 搬运旧元素（优先 `noexcept` 移动，否则拷贝） → 销毁旧元素
- **迭代器全部失效**
- 预分配用 `reserve(n)` 避免多次扩容

**emplace_back vs push_back**：
- `push_back(T{args...})`：先构造临时对象，再拷贝/移动到容器
- `emplace_back(args...)`：在容器内直接构造，避免临时对象

```cpp
struct Heavy {
    Heavy(int a, double b, std::string c) { /* 昂贵的构造 */ }
};

std::vector<Heavy> v;
v.push_back(Heavy(1, 2.0, "test"));   // 构造 Heavy → 移动进 vector → 析构临时对象
v.emplace_back(1, 2.0, "test");       // 直接在 vector 内构造，零拷贝
// ↑ emplace_back 更高效，尤其对于有多个参数的非平凡类型
```

**erase 的正确用法 —— 循环删除时注意迭代器失效**：

```cpp
// ❌ 错误：erase 后迭代器失效，但 for 循环仍在使用
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it % 2 == 0) v.erase(it);     // 未定义行为！
}

// ✅ 正确：erase 返回下一迭代器
auto it = v.begin();
while (it != v.end()) {
    if (*it % 2 == 0) it = v.erase(it); // 删除后 it 指向下一个
    else ++it;
}

// ✅ 更简洁：remove-erase 惯用法
v.erase(std::remove_if(v.begin(), v.end(), [](int x){ return x % 2 == 0; }), v.end());
```

**remove-erase 惯用法详解**：
`std::remove` 不删除元素，而是将"保留"的元素移到前面，返回新逻辑末尾。`erase` 再删除后面的无用元素。两步合起来才是真正的删除。

```cpp
// remove 原理：把要保留的元素往前挪
// 输入: [1, 2, 3, 4, 5]  删除所有偶数
// remove 后: [1, 3, 5, ?, ?]  返回指向第一个 ? 的迭代器
// erase 后: [1, 3, 5]

v.erase(std::remove(v.begin(), v.end(), val), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());  // 先去重再删除
```

**vector 性能对比**：

| 操作 | vector | list | deque |
|:----:|:------:|:----:|:-----:|
| 尾插 | O(1) 均摊 | O(1) | O(1) |
| 头插 | O(n) | O(1) | O(1) |
| 随机访问 | O(1) | O(n) | O(1) |
| 中间插入 | O(n) | O(1) | O(n) |
| 缓存友好 | ✅ 极高 | ❌ 低 | ✅ 高 |

**实战建议**：**默认用 vector**，除非特殊场景（频繁头插用 deque，频繁中间插入用 list）。

### 2.3 deque 内部结构

- **中控器（map）**：`T**` 指针数组，每个元素指向一个缓冲区
- **缓冲区**：固定大小数组（如 512 字节）
- 迭代器含：`_M_cur`、`_M_first`、`_M_last`、`_M_node`
- 随机访问：`(offset / buffer_size)` 定位缓冲区，`(offset % buffer_size)` 定位元素

### 2.4 map/set 红黑树

- **红黑树特性**：根黑、叶黑、红的孩子必黑、每条路径黑高相同
- map 的 key 是 `const` 的（`pair<const Key, Value>`），防止破坏有序性
- `operator[]`：不存在时**插入默认值**，只读用 `find()` 或 `at()`
- 插入返回 `pair<iterator, bool>`

```cpp
// map 基本用法
std::map<std::string, int> scores;
scores["Alice"] = 95;              // 插入：键不存在时创建默认值再赋值
scores["Bob"] = 87;

// operator[] vs find —— 重要的性能差异
int v1 = scores["Charlie"];        // ❌ 如果 Charlie 不存在，会插入默认值 0！
if (auto it = scores.find("Charlie"); it != scores.end()) {
    int v2 = it->second;           // ✅ find 不修改容器
}

// 插入
auto [it, inserted] = scores.insert({"David", 92});
if (inserted) {
    std::cout << "inserted: " << it->first << std::endl;
}

// 遍历（有序——按 key 的字典序）
for (const auto& [name, score] : scores) {
    std::cout << name << ": " << score << std::endl;
}

// set —— 不重复元素集合
std::set<int> s = {3, 1, 4, 1, 5, 9};
s.insert(2);                       // 重复值自动忽略
// s = {1, 2, 3, 4, 5, 9}
```

**map vs unordered_map 选型**：

| 场景 | 推荐 | 原因 |
|:----|:----|:-----|
| 需要有序遍历 | `map` (红黑树) | 按 key 排序输出 |
| 大量查找（>1000 元素） | `unordered_map` | O(1) 平均 vs O(log n) |
| 小集合（<50 元素） | `vector` 线性查找 | 缓存比哈希查找更快 |
| 自定义 key | `map` | 只需 `operator<`，不需写哈希函数 |

### 2.5 unordered_map 哈希表

- **链地址法**：桶数组 + 链表
- **负载因子**：`load_factor = size / bucket_count`，超过 `max_load_factor` 触发 rehash
- 自定义 key 需提供 `Hash` 和 `KeyEqual`（或 `operator==`）

```cpp
// 基本用法
std::unordered_map<std::string, int> umap;
umap.reserve(10000);               // ✅ 预分配桶，避免多次 rehash

// 自定义 hash —— 用于自定义类型
struct Person { std::string name; int age; };
struct PersonHash {
    size_t operator()(const Person& p) const {
        return std::hash<std::string>{}(p.name) ^ (std::hash<int>{}(p.age) << 1);
    }
};
std::unordered_map<Person, int, PersonHash> people;

// 遍历（无序）
for (const auto& [key, value] : umap) {
    std::cout << key << ": " << value << std::endl;
}

// rehash 性能影响
// 插入大量元素时，rehash 会重新分配桶并重新计算所有 key 的哈希值
// 如果是 O(n)，所以已知数量时提前 reserve 非常重要
```

**性能对比**：

| 操作 | map (红黑树) | unordered_map (哈希) |
|:----:|:-----------:|:-------------------:|
| 查找 | O(log n) | O(1) 平均，O(n) 最坏 |
| 插入 | O(log n) | O(1) 平均，O(n) 最坏 |
| 内存 | 小（树节点） | 大（桶数组 + 链表） |
| 顺序 | 有序 | 无序 |
| 适用 | 需要排序、范围查询 | 大量快速查找 |

### 2.6 迭代器失效

| 容器 | 插入/删除失效情况 |
|------|------------------|
| vector | 可能全部失效（扩容或移动） |
| deque | 中间插入/删除全部失效 |
| list/map/set | 仅被删除元素迭代器失效 |
| unordered_* | rehash 时全部失效 |

---

## 三、算法

### 3.1 重要算法

| 算法 | 复杂度 | 说明 |
|------|--------|------|
| `sort` | O(n log n) | 内省排序（快排+堆排+插入排序混合） |
| `stable_sort` | O(n log n) | 稳定排序 |
| `nth_element` | O(n) 均摊 | 第 K 大/小，不做全排序 |
| `partial_sort` | O(n log k) | 前 K 个有序 |
| `lower_bound` / `upper_bound` | O(log n) | 有序区间二分查找 |
| `binary_search` | O(log n) | 返回 bool |
| `merge` | O(n+m) | 合并有序区间 |
| `remove` / `unique` | O(n) | 不删除元素，返回新逻辑末尾 |

### 3.2 完整算法实战

```cpp
#include <algorithm>
#include <numeric>

std::vector<int> v = {5, 2, 8, 1, 9, 3, 7, 4, 6, 2, 8};

// ------ 排序相关 ------
std::sort(v.begin(), v.end());                       // 升序 [1,2,2,3,4,5,6,7,8,8,9]
std::sort(v.begin(), v.end(), std::greater<>());     // 降序
std::stable_sort(v.begin(), v.end());                // 稳定排序（相等元素保持原序）

// 部分排序 —— 只需要前 K 个有序时比全排序快
std::partial_sort(v.begin(), v.begin() + 3, v.end());  // 前 3 个有序，后面无序

// 第 K 大元素 —— O(n)，不排序
std::nth_element(v.begin(), v.begin() + 4, v.end());   // 第 5 小的元素归位
int kth_smallest = v[4];                                // 左边都比它小，右边都比它大

// ------ 查找相关 ------
// 二分查找（有序区间）
auto it = std::lower_bound(v.begin(), v.end(), 5);  // 第一个 >= 5 的位置
if (it != v.end() && *it == 5) { /* 找到 */ }

auto range = std::equal_range(v.begin(), v.end(), 5);  // 值为 5 的区间
// range.first = lower_bound, range.second = upper_bound

// 线性查找（无序区间）
auto found = std::find(v.begin(), v.end(), 42);
if (found != v.end()) { /* 找到 */ }

// ------ 修改相关 ------
// 删除指定值（remove-erase 惯用法）
v.erase(std::remove(v.begin(), v.end(), 2), v.end());  // 删除所有 2

// 去重（先排序）
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());

// 转换
std::transform(v.begin(), v.end(), v.begin(), [](int x) { return x * 2; });

// 填充
std::fill(v.begin(), v.end(), 0);
std::generate(v.begin(), v.end(), []() { return rand() % 100; });

// ------ 数值算法 ------
int sum = std::accumulate(v.begin(), v.end(), 0);       // 求和
int product = std::accumulate(v.begin(), v.end(), 1, std::multiplies<>());  // 求积
int prefix_sum = 0;
std::partial_sum(v.begin(), v.end(), v.begin());        // 前缀和

// ------ 分区 ------
auto mid = std::partition(v.begin(), v.end(), [](int x) { return x % 2 == 0; });  // 偶奇分区
// v = [偶数...|奇数...]，mid 指向第一个奇数
auto mid2 = std::stable_partition(v.begin(), v.end(), pred);  // 稳定分区（保持相对顺序）

// ------ 集合操作（需要有序区间） ------
std::vector<int> a = {1,2,3,4,5}, b = {3,4,5,6,7};
std::vector<int> c;
std::set_intersection(a.begin(), a.end(), b.begin(), b.end(), std::back_inserter(c));
// c = {3,4,5}
std::set_union(a.begin(), a.end(), b.begin(), b.end(), std::back_inserter(c));
// c = {1,2,3,4,5,6,7}
```

### 3.3 并行算法（C++17）

**解决的问题**：单线程排序/遍历在多核 CPU 上浪费算力。

```cpp
#include <execution>

// 三种执行策略
std::sort(std::execution::seq, v.begin(), v.end());        // 顺序执行（默认）
std::sort(std::execution::par, v.begin(), v.end());        // 多线程并行
std::for_each(std::execution::par_unseq, v.begin(), v.end(),  // 并行+向量化
              [](int& x) { x *= 2; });

// 适用场景
// ✅ 大数据量计算（>10000 元素）
// ✅ 元素之间无依赖（如 transform、for_each）
// ❌ 小数据量（多线程调度开销 > 加速收益）
// ❌ 有数据竞争（需要外部加锁）
```

### 3.4 算法复杂度速查

| 算法 | 复杂度 | 说明 |
|:----|:------:|:-----|
| `sort` | O(n log n) | 内省排序，**平均最快** |
| `stable_sort` | O(n log n) | 归并排序，稳定 |
| `nth_element` | O(n) 均摊 | 部分排序，**面试常考** |
| `partial_sort` | O(n log k) | 前 k 个有序 |
| `lower_bound` | O(log n) | 有序区间二分 |
| `binary_search` | O(log n) | 返回 bool |
| `merge` | O(n+m) | 合并有序区间 |
| `remove` / `unique` | O(n) | **不删除元素**，返回新逻辑末尾 |
| `set_intersection` | O(n+m) | 有序集合交集 |
| `accumulate` | O(n) | 数值累加 |

---

## 四、空间配置器（allocator）

### 4.1 标准接口

```cpp
pointer allocate(size_type n);        // 分配内存
void deallocate(pointer p, size_type n); // 释放
void construct(pointer p, Args&&...); // 构造对象（C++20 弃用）
void destroy(pointer p);              // 析构对象（C++20 弃用）
```

### 4.2 SGI 二级配置器（历史经典）

| 级别 | 大小 | 实现 |
|------|------|------|
| 一级 | > 128 字节 | 直接 malloc/free |
| 二级 | ≤ 128 字节 | 内存池 + 16 个自由链表（8 字节对齐） |

现代 STL 实现（libstdc++/libc++/MSVC）默认 `std::allocator` 直接封装 `::operator new` / `delete`，不再使用池化。

---

## 五、string 高效用法

```cpp
// 预分配减少扩容
std::string s;
s.reserve(10000);

// 传参用 string_view 避免拷贝（C++17）
void process(std::string_view sv);

// 多次拼接先 reserve 预估总长
result.reserve(a.size() + b.size() + c.size());
result += a += b += c;

// 查找分隔符
size_t pos = s.find_first_of(",;");  // 找任意一个
```

---

## 六、迭代器分类

| 迭代器 | 支持操作 | 典型容器 |
|--------|----------|----------|
| 输入迭代器 | 只读、向前 | istream_iterator |
| 输出迭代器 | 只写、向前 | ostream_iterator |
| 前向迭代器 | 读写、向前 | forward_list |
| 双向迭代器 | 读写、双向 | list、set、map |
| 随机访问迭代器 | 读写、随机访问 | vector、deque、array |

```cpp
// iterator_traits 标签分发实现不同复杂度
template<class InputIt, class Distance>
void advance(InputIt& it, Distance n, input_iterator_tag) {
    while (n--) ++it;   // O(n)
}
template<class RandomIt, class Distance>
void advance(RandomIt& it, Distance n, random_access_iterator_tag) {
    it += n;            // O(1)
}
```
