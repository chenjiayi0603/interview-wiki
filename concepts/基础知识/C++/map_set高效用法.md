# map / set 高效用法

> 本文涵盖 C++ map/set 高效使用细节：emplace、try_emplace、insert_or_assign、extract 等。

See also: [[STL容器与算法]], [[vector高效用法]]

## 3.1 插入：emplace 与 insert

**emplace** 在容器内就地构造元素，避免临时 `pair` 或 `Key` 的构造。

```cpp
#include <map>
#include <string>

std::map<int, std::string> m;

// ❌ insert(pair) 会构造临时 pair
m.insert(std::make_pair(1, "one"));

// ✅ emplace：直接构造 pair<const int, string>
m.emplace(1, "one");
m.emplace(2, "two");
```

## 3.2 避免重复查找：operator[] 与 insert 返回值

**operator[]**：若 key 不存在会**先默认构造 value 再插入**，适合“存在则用，不存在则建”且默认值可接受时。

若想“存在则用，不存在才插入”且**不想默认构造**，用 **insert** 或 **try_emplace**（C++17）。

```cpp
std::map<int, std::string> m;

// 仅当 key 不存在时插入
auto [it, inserted] = m.insert({1, "one"});
if (!inserted)
    it->second = "updated";  // 已存在则更新
```

## 3.3 try_emplace（C++17）

key 已存在时**不移动/不拷贝** value，避免临时 string 等；仅 key 不存在时构造 value。

```cpp
std::map<int, std::string> m;
std::string value = "large string ...";

// ❌ emplace：若 key 已存在，value 可能仍会参与临时对象
// ✅ try_emplace：key 已存在则什么都不做，不碰 value
m.try_emplace(1, value);       // 1 不存在时用 value 移动构造
m.try_emplace(1, "another");   // 1 已存在，不构造 "another" 的 string
```

## 3.4 insert_or_assign（C++17）

“有则覆盖，无则插入”，语义清晰，且可避免一次查找后再赋值的重复逻辑。

```cpp
std::map<int, std::string> m;
m.insert_or_assign(1, "one");   // 插入
m.insert_or_assign(1, "ONE");  // 更新为 "ONE"
```

## 3.5 查找用 find / at，慎用 operator[]

若**只读**，用 `find()` 或 `at()`，避免 operator[] 在不存在时插入。

```cpp
auto it = m.find(42);
if (it != m.end())
    use(it->second);

// 或：存在才访问，否则抛异常
try {
    std::string s = m.at(42);
} catch (const std::out_of_range&) {}
```

## 3.6 范围提取（C++17）：extract

**extract** 从 set/map 中“取出”节点而不析构元素，可修改 key 再插回，或移到另一容器，避免拷贝/移动 value。

```cpp
std::map<int, std::string> m = {{1, "a"}, {2, "b"}};
auto node = m.extract(1);  // 取出节点
if (!node.empty()) {
    node.key() = 10;       // 改 key
    m.insert(std::move(node));  // 插回
}
```

[src: raw/ingested/2技术/cpp/STL高效使用细节与示例-三、map---set-高效用法.md]