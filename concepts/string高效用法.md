# string 高效用法

## 2.1 reserve 减少扩容

与 vector 类似，已知长度时先 `reserve`。

```cpp
std::string s;
s.reserve(10000);
for (int i = 0; i < 10000; ++i)
    s += 'x';
```

## 2.2 避免临时 string：传 const string& 或 string_view

**函数参数**：若不修改且不持有，用 `const std::string&` 或 `std::string_view`（C++17）避免临时 string 和拷贝。

```cpp
void process(const std::string& str);
void process_sv(std::string_view str);  // C++17，可接受字面量、string、子串

process("hello");   // 会生成临时 string
process_sv("hello"); // 无临时 string，仅视图
```

## 2.3 移动而非拷贝

接收“可被拿走”的 string 时用**按值 + 移动**或**右值引用**，避免拷贝。

```cpp
void take(std::string s) {
    // 调用者传 std::move(s) 或临时对象时，只移动
}
std::string big = "very long ...";
take(std::move(big));  // 移动，big 变为空
```

## 2.4 拼接：operator+= 与 reserve

多次拼接时先 `reserve(预估总长)` 再 `+=`，减少扩容。

```cpp
std::string a = "hello", b = " ", c = "world";
std::string result;
result.reserve(a.size() + b.size() + c.size());
result += a;
result += b;
result += c;
```

## 2.5 查找：find / find_first_of / 等

- **find**：找子串或字符的首次出现。
- **find_first_of**：在当前串中找“任意一个给定字符”首次出现，适合“分隔符集合”。

```cpp
std::string s = "ab,cde;fg";
// 找第一个 ',' 或 ';'
size_t pos = s.find_first_of(",;");  // 3
// 找子串
size_t p2 = s.find("de");           // 4
```

## 2.6 子串：substr 产生新 string

需要子串时会分配新 string；若只读，用 `string_view` 可避免分配（C++17）。

```cpp
std::string s = "hello world";
std::string sub = s.substr(0, 5);  // 新分配 "hello"

// C++17：仅视图，不分配
std::string_view sv(s);
std::string_view sub_sv = sv.substr(0, 5);
```

[src: raw/ingested/2技术/cpp/STL高效使用细节与示例-二、string-高效用法.md]