# C++可变参数函数

## 三种实现方式对比

| 分类 | 方法 | 主要优点 | 主要缺点 | 适用场景 |
| --- | --- | --- | --- | --- |
| 1️⃣ | `initializer_list` | 简洁、安全、类型统一 | 仅支持相同类型参数 | 同类型参数，如求和、统计等 |
| 2️⃣ | 可变参数宏 (`va_list`) | 与 C 兼容、运行时灵活 | 不安全、类型需自控 | C 接口、系统函数如 printf |
| 3️⃣ | 可变参数模板 (`template<typename...>`) | 类型安全、通用性强 | 实现复杂、编译开销大 | 泛型库、现代 C++ 框架开发 |

## initializer_list（C++11）

```cpp
int sum(initializer_list<int> il) {
    int sum = 0;
    for(auto p = il.begin(); p != il.end(); ++p)
        sum += *p;
    return sum;
}
cout << sum({1,2,3});  // 输出 6
```

## 可变参数宏（va_list）

```cpp
int sum(int count, ...) {
    va_list ap;
    va_start(ap, count);
    int s = 0;
    for(int i=0; i<count; ++i)
        s += va_arg(ap, int);
    va_end(ap);
    return s;
}
cout << sum(3,1,2,3);  // 输出 6
```

## 可变参数模板

```cpp
template<typename T>
void print(const T& t) { cout << t << endl; }

template<typename T, typename... Args>
void print(const T& t, const Args&... rest) {
    cout << t << " ";
    print(rest...);
}
print(1, "abc", 3.14);  // 输出: 1 abc 3.14
```

[src: raw/ingested/2技术/cpp/C++基础语法手册-第八章-可变参数函数.md]

## Related Pages
- [[C++语言特性]]
- [[C++进阶知识点]]
- [[静多态与动态多态]]
