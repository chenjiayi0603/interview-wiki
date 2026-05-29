# STL 算法高效用法

> 本文涵盖 STL 算法的高效使用细节与示例，包括查找、排序、去重、归约、分区、并行算法等。

See also: [[STL容器与算法]], [[STL源码考点]], [[C++语言特性]]

---

## 7.1 有序区间查找：lower_bound / upper_bound

**lower_bound**：第一个 >= 目标；**upper_bound**：第一个 > 目标；**equal_range**：两者组成的区间。O(log n)。

```cpp
std::vector<int> v = {1, 2, 2, 2, 3, 4};
auto lo = std::lower_bound(v.begin(), v.end(), 2);  // 第一个 2
auto hi = std::upper_bound(v.begin(), v.end(), 2);   // 第一个 3
// [lo, hi) 即所有 2
auto [l, r] = std::equal_range(v.begin(), v.end(), 2);
```

## 7.2 仅需第 K 大/小：nth_element

**nth_element** 将第 n 小的元素放到第 n 位，左边 <= 它，右边 >= 它，**不保证左右内部有序**。平均 O(n)，比全排序省。

```cpp
std::vector<int> v = {5, 2, 8, 1, 9, 3};
std::nth_element(v.begin(), v.begin() + 2, v.end());
// v[2] 是第 3 小的数，左边 <= v[2]，右边 >= v[2]
```

## 7.3 前 K 个有序：partial_sort

前 K 个最小且有序，其余无序。O(n log K)。

```cpp
std::vector<int> v = {5, 2, 8, 1, 9, 3};
std::partial_sort(v.begin(), v.begin() + 3, v.end());
// v 前 3 个为 {1, 2, 3} 且有序
// 1 2 3 8 9 5 
```

## 7.4 删除满足条件的元素：remove-erase 惯用法

**remove / remove_if** 只把"要保留的"移到前部，返回新的逻辑 end，**不改变 size**；真正删除用 **erase**。

```cpp
std::vector<int> v = {1, 2, 3, 2, 4, 2};
v.erase(std::remove(v.begin(), v.end(), 2), v.end());  // {1, 3, 4}

std::vector<int> v2 = {1, 2, 3, 4, 5};
v2.erase(std::remove_if(v2.begin(), v2.end(), [](int x){ return x % 2 == 0; }), v2.end());
// {1, 3, 5}
```

## 7.5 去重：sort + unique + erase

**unique** 只去除**相邻**重复，故先 **sort** 再 unique，再 **erase**。

```cpp
std::vector<int> v = {3, 1, 2, 2, 1, 3};
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());  // {1, 2, 3}
```

## 7.6 二分：binary_search / lower_bound

**binary_search** 只返回是否存在；要位置用 **lower_bound**。

```cpp
bool exists = std::binary_search(v.begin(), v.end(), 42);
auto it = std::lower_bound(v.begin(), v.end(), 42);
if (it != v.end() && *it == 42) { /* 找到了 */ }
```

## 7.7 求和 / 归约：accumulate / reduce

**accumulate** 顺序求和（或自定义二元 op）；**reduce**（C++17）可并行，但顺序未指定。

```cpp
#include <numeric>
std::vector<int> v = {1, 2, 3, 4, 5};
int sum = std::accumulate(v.begin(), v.end(), 0);
int product = std::accumulate(v.begin(), v.end(), 1, std::multiplies<int>());
```

## 7.8 填充与生成：fill / iota / generate

```cpp
std::vector<int> v(10);
std::fill(v.begin(), v.end(), 42);
std::iota(v.begin(), v.end(), 0);   // 0, 1, 2, ...
int x = 0;
std::generate(v.begin(), v.end(), [&x]{ return x++; });
```

## 7.9 分区：partition / stable_partition

**partition** 将满足谓词的元素放到前面，不保证相对顺序；**stable_partition** 保持相对顺序。

```cpp
std::vector<int> v = {1, 2, 3, 4, 5, 6};
auto mid = std::partition(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
// 前半为偶数，后半为奇数
```

## 7.10 并行算法（C++17）

指定 **execution::par** 或 **par_unseq** 可并行执行（实现支持时）。

```cpp
#include <execution>
std::vector<int> v = { ... };
std::sort(std::execution::par, v.begin(), v.end());
std::for_each(std::execution::par, v.begin(), v.end(), [](int& x){ x *= 2; });
```

## 7.11 合并有序区间：merge / inplace_merge

**merge** 将两个有序区间合并到第三个；**inplace_merge** 原地合并同一容器内两段连续有序区间。

```cpp
std::vector<int> a = {1, 3, 5}, b = {2, 4, 6}, out;
std::merge(a.begin(), a.end(), b.begin(), b.end(), std::back_inserter(out));
// out == {1, 2, 3, 4, 5, 6}

std::vector<int> v = {1, 3, 5, 2, 4, 6};  // [0,3) 与 [3,6) 各自有序
std::inplace_merge(v.begin(), v.begin() + 3, v.end());
// v == {1, 2, 3, 4, 5, 6}
```

## 7.12 集合操作：set_union / set_intersection / set_difference

有序区间上的集合运算，结果有序，输出到插入迭代器。

```cpp
std::vector<int> a = {1, 2, 3}, b = {2, 3, 4}, u, i, d;
std::set_union(a.begin(), a.end(), b.begin(), b.end(), std::back_inserter(u));   // 并
std::set_intersection(a.begin(), a.end(), b.begin(), b.end(), std::back_inserter(i)); // 交
std::set_difference(a.begin(), a.end(), b.begin(), b.end(), std::back_inserter(d));   // 差
// u == {1,2,3,4}, i == {2,3}, d == {1}
```

## 7.13 最值与位置：min_element / max_element / minmax_element

返回迭代器，O(n)。

```cpp
auto it = std::min_element(v.begin(), v.end());
auto [min_it, max_it] = std::minmax_element(v.begin(), v.end());
```

## 7.14 判断是否有序：is_sorted / is_sorted_until

**is_sorted** 返回 bool；**is_sorted_until** 返回第一个破坏有序的迭代器。

```cpp
bool ok = std::is_sorted(v.begin(), v.end());
auto pos = std::is_sorted_until(v.begin(), v.end());
```

## 7.15 堆操作：make_heap / push_heap / pop_heap

手动维护堆时用堆算法，**priority_queue** 内部即如此实现。

```cpp
std::vector<int> v = {3, 1, 4, 1, 5};
std::make_heap(v.begin(), v.end());
v.push_back(9);
std::push_heap(v.begin(), v.end());
std::pop_heap(v.begin(), v.end());
v.pop_back();
```

## 7.16 复制与移动：copy / move

**std::move** 算法对区间内元素做移动赋值（从源到目标），源区间变为"未指定状态"。

```cpp
std::vector<std::string> src = {"a", "b"}, dst(2);
std::move(src.begin(), src.end(), dst.begin());
// dst 得到字符串，src 元素变为空等未指定状态
```

## 7.17 查找首次/末次不满足条件：find_if_not / find_if

**find_if** 找第一个满足谓词的；**find_if_not** 找第一个不满足的。

```cpp
auto it = std::find_if(v.begin(), v.end(), [](int x){ return x > 5; });
auto it2 = std::find_if_not(v.begin(), v.end(), [](int x){ return x < 0; });
```

## 7.18 条件计数与判断：count_if / all_of / any_of / none_of

```cpp
int n = std::count_if(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
bool all_pos = std::all_of(v.begin(), v.end(), [](int x){ return x > 0; });
bool any_even = std::any_of(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
bool none_neg = std::none_of(v.begin(), v.end(), [](int x){ return x < 0; });
```

## 7.19 transform 与生成新序列

**transform** 一元或二元，输出到另一区间。

```cpp
std::vector<int> a = {1, 2, 3}, b;
std::transform(a.begin(), a.end(), std::back_inserter(b), [](int x){ return x * 2; });
// b == {2, 4, 6}
```

## 7.20 交换两段区间：swap_ranges

```cpp
std::vector<int> a = {1, 2, 3}, b = {4, 5, 6};
std::swap_ranges(a.begin(), a.end(), b.begin());
// a == {4, 5, 6}, b == {1, 2, 3}
```

[src: raw/ingested/2技术/cpp/STL高效使用细节与示例-七、算法高效用法.md]

## Related Pages
- [[STL容器与算法]]
- [[STL源码考点]]
- [[vector高效用法]]
- [[C++语言特性]]
- [[C++17并行算法]]
