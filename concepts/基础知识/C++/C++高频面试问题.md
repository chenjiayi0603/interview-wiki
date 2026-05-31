# C++高频面试问题

See also: [[C++语言特性]], [[STL容器与算法]], [[智能指针]], [[C++多线程与并发]], [[vector扩容与noexcept移动构造]]

## 一、基础语法

### 1. static 关键字的作用？
- **局部静态变量**：函数内 static 变量只初始化一次，生命周期为整个程序
- **全局静态变量/函数**：限制作用域为当前文件（内部链接）
- **类静态成员**：属于类而非对象，所有对象共享
- **类静态成员函数**：只能访问静态成员，无 this 指针

### 2. 指针和引用的区别？
| 特性 | 指针 | 引用 |
|------|------|------|
| 初始化 | 可不初始化 | 必须初始化 |
| 重新绑定 | 可指向其他对象 | 不可重新绑定 |
| 空值 | 可为 nullptr | 不可为空 |
| sizeof | 指针本身大小（8/4 字节） | 引用对象的大小 |
| 多级 | 可有多级指针 | 无多级引用 |
| 底层实现 | 存储地址 | 本质是指针的语法糖 |

### 3. new/delete 与 malloc/free 的区别？
| 特性 | new/delete | malloc/free |
|------|-----------|-------------|
| 类型 | C++ 运算符 | C 库函数 |
| 调用构造函数/析构函数 | ✅ | ❌ |
| 返回类型 | 类型安全（返回具体类型指针） | void*，需强制转换 |
| 失败行为 | 抛 `std::bad_alloc` 异常 | 返回 NULL |
| 重载 | 可重载 | 不可重载 |
| 内存大小 | 编译器自动计算 | 需手动指定字节数 |

---

## 二、面向对象

### 4. 虚函数实现原理？
- 每个含虚函数的类有一个**虚函数表（vtable）**
- 每个对象有一个**虚函数指针（vptr）**指向 vtable
- 调用虚函数时通过 vptr → vtable 查找实际函数地址
- 虚函数表在编译期生成，存储在只读数据段

### 5. 虚析构函数为什么需要？
```cpp
Base* p = new Derived();
delete p;  // 若 ~Base() 非虚，只调用 ~Base()，Derived 部分泄漏
```
- 确保通过基类指针删除派生类对象时，正确调用派生类析构函数

### 6. 构造函数可以是虚函数吗？
- **不可以**。构造时 vptr 尚未初始化，无法动态绑定

### 7. 纯虚函数与抽象类？
- 纯虚函数：`virtual void func() = 0;`
- 含纯虚函数的类为抽象类，不能实例化
- 派生类必须实现所有纯虚函数才能实例化

---

## 三、内存管理

### 8. 内存泄漏常见原因与排查？
- **忘记 delete/delete[]**
- **循环引用**（shared_ptr 互相引用）
- **异常安全**：异常抛出导致 delete 未执行
- 排查工具：Valgrind、AddressSanitizer、Visual Studio 内存诊断

### 9. 智能指针循环引用如何解决？
```cpp
class B;
class A { std::shared_ptr<B> b_ptr; };
class B { std::shared_ptr<A> a_ptr; };  // 循环引用，永不释放

// 解决：将其中一个改为 weak_ptr
class B { std::weak_ptr<A> a_ptr; };
```

### 10. enable_shared_from_this 的作用？
- 在类内部安全获取自身的 `shared_ptr`
- 避免从 `this` 创建多个独立控制块

---

## 四、STL

### 11. vector 扩容机制？
- 当 `size() == capacity()` 时触发
- 新容量：GCC 1.5 倍，MSVC 2 倍
- 步骤：分配新内存 → 搬运旧元素 → 释放旧内存
- 搬运策略：移动构造有 `noexcept` 则优先移动，否则拷贝（详见 [[vector扩容与noexcept移动构造]]）

### 12. map 和 unordered_map 区别？
| 特性 | map | unordered_map |
|------|-----|---------------|
| 底层 | 红黑树 | 哈希表 |
| 有序性 | 有序 | 无序 |
| 查找 | O(log n) | O(1) 平均，O(n) 最坏 |
| 插入 | O(log n) | O(1) 平均 |
| 内存 | 较小 | 较大（桶+链表） |
| 迭代器失效 | 仅删除节点 | rehash 时全部失效 |

### 13. 迭代器失效场景？
- **vector**：扩容全部失效；insert/erase 使插入点及之后失效
- **deque**：两端插入不失效；中间插入全部失效
- **list/map/set**：仅被删除节点失效
- **unordered_map**：rehash 时全部失效

---

## 五、C++11/14/17/20

### 14. 移动语义与右值引用？
- 右值引用 `T&&`：绑定到临时对象
- 移动构造/赋值："窃取"资源，避免深拷贝
- `std::move`：将左值转为右值引用
- `std::forward`：完美转发，保持值类别

### 15. Lambda 表达式捕获方式？
- `[=]`：值捕获全部
- `[&]`：引用捕获全部
- `[x]`：值捕获 x
- `[&x]`：引用捕获 x
- `[this]`：捕获 this 指针
- `[*this]`（C++17）：捕获对象副本

### 16. 智能指针类型与选择？
- `unique_ptr`：独占所有权，不可拷贝，可移动
- `shared_ptr`：共享所有权，引用计数
- `weak_ptr`：不增加引用计数，用于打破循环引用

---

## 六、多线程

### 17. mutex 与 lock_guard/unique_lock 区别？
- `lock_guard`：构造加锁，析构解锁，不可手动控制
- `unique_lock`：可手动 lock/unlock，可延迟加锁，可配合条件变量

### 18. 原子操作与互斥锁的选择？
- 简单计数器/标志位 → `std::atomic`
- 复杂临界区 → `std::mutex`
- 原子操作无锁，性能更高，但功能有限

### 19. 死锁的四个必要条件与避免？
- 互斥、持有并等待、不可剥夺、循环等待
- 避免：固定加锁顺序、`std::lock` 同时加锁、`std::scoped_lock`（C++17）

---

## 七、编译与调试

### 20. 编译过程四个阶段？
1. **预处理**：宏展开、头文件包含、条件编译
2. **编译**：源码 → 汇编代码
3. **汇编**：汇编代码 → 目标文件（.o）
4. **链接**：目标文件 + 库 → 可执行文件

### 21. Core Dump 分析流程？
- `ulimit -c unlimited` 开启 core dump
- `gdb <executable> <core>` 加载分析
- `bt` 查看调用栈，`info threads` 查看所有线程
- `frame N` 切换栈帧，`print var` 查看变量

---

## 八、高频代码题

### 22. 实现线程安全单例（C++11）
```cpp
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance;  // C++11 保证线程安全
        return instance;
    }
private:
    Singleton() = default;
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};
```

### 23. 实现简单 string 类
```cpp
class MyString {
    char* data_;
    size_t size_;
public:
    MyString(const char* s) : size_(strlen(s)) {
        data_ = new char[size_ + 1];
        strcpy(data_, s);
    }
    MyString(const MyString& other) : size_(other.size_) {
        data_ = new char[size_ + 1];
        strcpy(data_, other.data_);
    }
    MyString(MyString&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    ~MyString() { delete[] data_; }
};
```

[src: raw/ingested/2技术/cpp/C++核心知识.md]

## Related Pages
- [[C++语言特性]]
- [[STL容器与算法]]
- [[智能指针]]
- [[C++多线程与并发]]
- [[vector扩容与noexcept移动构造]]
