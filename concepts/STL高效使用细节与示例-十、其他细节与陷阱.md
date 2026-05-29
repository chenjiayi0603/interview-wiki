# STL 高效使用细节与示例 - 十、其他细节与陷阱

## 10.1 map[] 与不存在的 key

**operator[]** 在 key 不存在时会**插入**并值初始化，可能不符合“只读”预期；只读用 **find** 或 **at**。

## 10.2 vector 与 list 的选择

- 需要**随机访问、缓存友好、尾插为主**：**vector**。
- 需要**中间频繁插入删除、不需随机访问**：**list**。
- 多数场景 **vector** 更合适，list 的指针跳转对缓存不友好。

## 10.3 用 empty() 判断是否为空

统一用 **empty()** 而非 `size() == 0`，部分容器 **size()** 可能是 O(n)（如旧版 list）。

## 10.4 比较器与严格弱序

**sort / set / map** 等要求的比较必须满足**严格弱序**（如 `<`），否则未定义行为。例如不能用 `<=` 做“小于”比较。

## 10.5 迭代器失效小结

| 容器           | 插入/删除后失效情况 |
|----------------|---------------------|
| vector/deque   | 插入/删除可能使所有迭代器失效（扩容或移动） |
| list/map/set  | 仅被删除元素对应迭代器失效 |
| unordered_*   | rehash 时可能全部失效 |

## 10.6 用 std::array 替代裸数组

定长时用 **std::array**，支持迭代器、**at()** 边界检查、可拷贝、可作容器接口。

```cpp
std::array<int, 5> a = {1, 2, 3, 4, 5};
std::sort(a.begin(), a.end());
```

## 10.7 用 std::span（C++20）传递连续区间

非拥有视图，可接受数组、vector、string 等，避免重载多种指针+长度接口。

```cpp
void process(std::span<const int> s) {
    for (int x : s) use(x);
}
std::vector<int> v = {1, 2, 3};
process(v);
int arr[] = {4, 5, 6};
process(arr);
```

## 10.8 移动整个容器

**swap** 或 **移动赋值** 可 O(1) 转移容器内容，避免拷贝。

```cpp
std::vector<int> a = {1, 2, 3}, b;
b = std::move(a);  // a 变为未指定状态，通常为空
```

## 10.9 map 的 key 是 const

map 中元素类型为 **pair<const Key, Value>**，迭代器不能修改 key，只能修改 value。

```cpp
std::map<int, std::string> m = {{1, "a"}};
auto it = m.begin();
// it->first = 2;   // 错误：key 只读
it->second = "b";   // 正确
```

## 10.10 用 erase 接受 key（关联容器 C++11）

关联容器 **erase(key)** 返回删除个数（map/set 为 0 或 1），可避免先 find 再 erase。

```cpp
std::map<int, int> m = {{1, 10}, {2, 20}};
size_t n = m.erase(1);  // n == 1，无需先 find
```

## 10.11 用 erase 接受迭代器范围（C++11）

**erase(begin, end)** 一次删除一段，比循环 erase 单元素更高效（尤其 list）。

```cpp
std::list<int> lst = {1, 2, 3, 4, 5};
lst.erase(std::next(lst.begin(), 2), lst.end());  // 删除第 3 个到末尾
```

## 10.12 避免在循环里用 size() 做边界（vector 除外）

部分容器 **size()** 可能是 O(n)（如旧版 **std::list**），循环中应缓存或用 **empty()** 判断结束。

```cpp
// 不推荐对 list 在每次循环中调用 size()
for (auto it = lst.begin(); it != lst.end(); ++it) { }  // 用 end() 比较
```

## 10.13 reserve 对 list/deque 无效

**reserve** 仅对 **vector / string / unordered_*** 等有“容量”概念的容器有效，**list / deque / map / set** 没有 reserve。

## 10.14 从容器中取出元素：extract（C++17）

set/map/unordered 的 **extract** 可取出节点，修改 key 再 insert，避免整元素拷贝。

```cpp
std::set<int> s = {1, 2, 3};
auto node = s.extract(2);
node.value() = 20;
s.insert(std::move(node));  // s 中有 20，无 2
```

## 10.15 合并容器：merge（list/set 等）

**list::merge** 将另一个有序 list 合并进当前 list（另一 list 会变空）；**set** 无 merge，可用 **insert(range)**。

```cpp
std::list<int> a = {1, 3, 5}, b = {2, 4, 6};
a.merge(b);  // a == {1,2,3,4,5,6}, b 为空
```

## 10.16 string 的 data() 与 c_str()

**data()**（C++17 起）与 **c_str()** 对 std::string 返回相同指针；**data()** 可写（非 const 版本），且不保证结尾有 `'\0'`（C++11/14）；C++17 起 **data()** 返回指向连续存储的指针，与 **c_str()** 一致。需要 C API 时常用 **c_str()** 保证 `'\0'`。

## 10.17 用 empty() 而非 size() == 0

对所有容器统一用 **empty()**，语义清晰且对 list 等可能更高效。

## 10.18 自定义类型做 set/map key

必须可**比较**（提供 **operator<** 或自定义 **Compare**）；做 **unordered** key 必须提供 **Hash** 和 **Equal**。

## 10.19 用 make_unique / make_shared 放容器

容器中放智能指针时，优先 **make_unique / make_shared**，异常安全且减少一次分配（shared）。

```cpp
std::vector<std::shared_ptr<Widget>> v;
v.push_back(std::make_shared<Widget>(args...));
```

## 10.20 算法与 Ranges（C++20）

**std::ranges::sort(v)** 等可省略 begin/end，且支持 **views::filter/transform** 等惰性视图，减少中间容器。

```cpp
#include <ranges>
std::vector<int> v = {1, 2, 3, 4, 5};
auto even = v | std::views::filter([](int x){ return x % 2 == 0; });
auto squared = even | std::views::transform([](int x){ return x * x; });
```

[src: raw/ingested/2技术/cpp/STL高效使用细节与示例-十、其他细节与陷阱.md]