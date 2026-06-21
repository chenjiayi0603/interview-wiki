# 面试考点速查

> C++ 高频面试题、常见陷阱、项目深挖问题汇总。

---

## 一、高频面试题

### Q1: static 关键字的作用？

1. **静态局部变量**：函数内声明，只初始化一次，程序结束时销毁
2. **静态全局变量/函数**：限制作用域在本文件（internal linkage）
3. **类的静态成员**：所有对象共享，需类外初始化
4. **类的静态方法**：无 this 指针，只能访问静态成员

### Q2: 指针和引用的区别？

| 维度 | 指针 | 引用 |
|------|------|------|
| 可为空 | ✅ 可 nullptr | ❌ 必须绑定 |
| 重赋值 | ✅ | ❌ |
| 访问 | 需解引用 | 直接使用 |
| sizeof | 8 字节 | 对象大小 |
| 多级 | ✅ | ❌ |

### Q3: new/delete vs malloc/free？

- new/delete 是**运算符**，malloc/free 是**函数**
- new 调用构造函数，malloc 只分配内存
- delete 调用析构函数，free 只释放
- new 返回正确类型指针，malloc 返回 `void*`
- new 失败抛 `std::bad_alloc`，malloc 返回 NULL

### Q4: 虚函数实现原理？

- 每个有虚函数的类有一张**虚表（vtable）**——函数指针数组
- 每个对象有一个**虚表指针（vptr）**指向类的虚表
- 虚函数调用 = `vptr[vtable_index]`
- 多继承可能有多个 vptr
- 虚表编译时确定，vptr 在构造函数的初始化列表中设置
- **能否 inline**：虚函数可通过对象调用时 inline，通过指针/引用调用时不能（运行期多态）

### Q5: vector 扩容机制？

- `size() == capacity()` 时触发，常见 2 倍扩容
- 申请新内存 → 搬运元素（优先 noexcept 移动，否则拷贝）→ 销毁旧元素
- 迭代器**全部失效**
- amortized O(1) 插入
- 可用 `reserve(n)` 预分配避免扩容

### Q6: map vs unordered_map？

| 维度 | map | unordered_map |
|------|-----|--------------|
| 底层 | 红黑树 | 哈希表（链地址法） |
| 有序性 | key 有序 | 无序 |
| 复杂度 | O(log n) | 平均 O(1)，最坏 O(n) |
| 迭代器 | 双向 | 前向 |
| 自定义 key | 需 `operator<` 或 Compare | 需 Hash + KeyEqual |

### Q7: shared_ptr 线程安全性？

- **控制块引用计数**是原子的，多个线程同时修改 shared_ptr 时安全
- 多个线程同时**读写同一个 shared_ptr 对象**需要外部加锁
- 建议使用 `make_shared`（一次分配控制块+对象）

### Q8: emplace_back vs push_back？

- `push_back(T{args...})`：先构造临时对象，再拷贝/移动到容器
- `emplace_back(args...)`：直接在容器内构造，避免临时对象
- 对多参数构造或昂贵拷贝类型，emplace 更优

### Q9: C++20 协程是什么？

- `co_await`（等待）、`co_yield`（生成）、`co_return`（返回）
- 需实现 `promise_type` 和 `coroutine_handle`
- **无栈协程**（stackless），仅顶层可挂起
- 适用：异步 I/O、生成器、惰性序列
- vs libgo：C++20 是语言原语，libgo 是完整的 M:N 调度框架

### Q10: 什么是 RAII？为什么重要？

RAII（资源获取即初始化）：构造函数获取资源，析构函数释放资源。
- 异常安全：栈展开时自动调用析构函数
- 无泄漏：资源生命周期与对象绑定
- 典型应用：`unique_ptr`、`lock_guard`、`fstream`

### Q11: 如何避免死锁？

1. 固定加锁顺序（所有线程以相同顺序获取锁）
2. 使用 `std::scoped_lock` 同时锁多个 mutex
3. 使用 `std::lock()` 一次性锁多个 mutex
4. 尽量缩小临界区范围
5. 使用超时锁 `try_lock_for`

### Q12: C++ 内存对齐规则？

- 结构体对齐：最大成员对齐的整数倍
- 可用 `#pragma pack(n)` 改变对齐
- `alignas(n)` / `alignof`（C++11）
- 伪共享：缓存行对齐（64 字节）

### Q13: 迭代器失效场景？

| 容器 | 插入失效 | 删除失效 |
|------|----------|----------|
| vector | 扩容后全部失效 | erase 后之后的迭代器失效 |
| deque | 全失效 | 中间删除全失效 |
| list/map/set | 不失效 | 仅被删元素的迭代器失效 |
| unordered_* | rehash 后全失效 | 仅被删元素失效 |

---

## 二、常见陷阱

### 1. map 的 operator[] 插入默认值

```cpp
std::map<int, std::string> m;
std::cout << m[42];  // 不存在时插入空 string，size 变为 1
// 用 find() 或 at() 替代
```

### 2. 同一原始指针管理多个 shared_ptr

```cpp
int* p = new int(42);
std::shared_ptr<int> sp1(p);
std::shared_ptr<int> sp2(p);  // 双 delete！独立控制块
```

### 3. remove 不删除元素

```cpp
std::vector<int> v = {1, 2, 3, 2};
std::remove(v.begin(), v.end(), 2);  // v.size() 还是 4！
// 必须配合 erase：v.erase(remove(...), v.end());
```

### 4. 循环引用导致内存泄漏

```cpp
struct A { std::shared_ptr<B> b; };
struct B { std::shared_ptr<A> a; };
// 一方改用 weak_ptr 解决
```

### 5. noexcept 移动构造对 vector 扩容的影响

vector 扩容时，若移动构造不是 noexcept，可能会改用拷贝构造（异常安全）。

### 6. 空 vector 的 &v[0] 是未定义行为

```cpp
std::vector<int> v;
int* p = &v[0];         // UB!
int* p = v.data();      // OK，C++11 起 data() 可为 nullptr
```

### 7. string 的 data() 不保证以 '\0' 结尾（C++11/14）

```cpp
std::string s = "hello";
// C++11: data() 保证连续但不一定 '\0' 结尾
// C++17起: data() 与 c_str() 一致
```

### 8. exit 在信号处理中的问题

```cpp
exit();   // 执行 atexit 回调、刷新流，可能死锁
_exit();  // 直接内核终止，安全
```

---

## 三、项目深挖问题

### Q1: 华为云 KeyDB 存算分离中如何保证数据一致性？

- Write-Ahead Log (WAL) 策略
- 计算层与存储层分离的挑战
- 数据校验机制
- 故障恢复流程
- gRPC 长连接保活

### Q2: 高并发 IM 系统如何设计线程模型？

- Reactor 模式（libev/epoll）
- 多 IO 线程 + 多工作线程
- 连接池管理
- 心跳检测
- 内存池优化减少分配开销

### Q3: 为什么选择用 C++ 实现微服务？

- 性能优势：低延迟、低内存占用
- 资源效率：高并发场景成本低
- 控制力：精细内存管理和网络调优
- 生态：gRPC、Envoy 等成熟框架
- 团队技术栈匹配

---

## 四、STL 源码级面试

| 问题 | 一句话答案 |
|------|-----------|
| vector 三指针含义？ | `_M_start`（起始）、`_M_finish`（尾后）、`_M_end_of_storage`（容量尾后） |
| deque 如何 O(1) 随机访问？ | 中控器（map）+ 缓冲区，通过块号+块内偏移定位 |
| list::size() 复杂度？ | 旧版 O(n)，现代 O(1)（成员变量维护） |
| map 的 key 为什么是 const？ | `pair<const Key, T>` 防止修改 key 破坏红黑树有序性 |
| remove 为什么不删除元素？ | 算法只操作迭代器区间，无法修改容器大小 |
| advance 如何实现不同复杂度？ | iterator_traits 标签分发（input_iterator_tag vs random_access_iterator_tag） |
| 空间配置器 8 字节对齐原因？ | 最小块需能存指针（free list 的 next） |
| priority_queue 底层？ | vector + make_heap/push_heap/pop_heap |
| unordered_map rehash 复杂度？ | O(n)，但均摊到每次插入为 O(1) |
| SGI 二级配置器何时用 free list？ | ≤ 128 字节，16 个 free list，8 字节递增 |
