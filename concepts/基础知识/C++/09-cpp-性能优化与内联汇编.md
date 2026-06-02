# 性能优化与内联汇编

> SIMD 指令、内联汇编、内存优化、零拷贝、性能分析工具。

---

## 一、SIMD 指令优化

### 1.1 Intel Intrinsics

```cpp
#include <immintrin.h>

// SSE：128 位
__m128 a = _mm_load_ps(ptr);           // 对齐加载
__m128 sum = _mm_add_ps(a, b);          // 单精度加法

// AVX：256 位
__m256 c = _mm256_load_ps(ptr);        // 对齐加载
__m256 result = _mm256_mul_ps(c, d);    // 单精度乘法
_mm256_store_ps(ptr, result);           // 对齐存储

// AVX-512：512 位
__m512 e = _mm512_load_ps(ptr);
```

**适用场景**：数组运算、图像处理、矩阵运算、加密算法。

**关键注意**：
- 数据需要内存对齐（AVX 需要 32 字节对齐）
- 循环展开配合 SIMD 效果更好
- 编译器自动向量化有限，手动 SIMD 可获极致性能

### 1.2 编译器自动向量化

```cpp
// 编译器能自动向量化的模式
for (int i = 0; i < N; ++i) {
    a[i] = b[i] * c[i];  // 独立迭代，连续内存
}

// 阻碍自动向量化的因素
// 1. 非连续内存访问
// 2. 函数调用
// 3. 数据依赖（如 a[i] = a[i-1] * 2）
```

---

## 二、内联汇编

### 2.1 GCC 内联汇编格式

```cpp
asm [volatile] (
    "汇编指令模板"
    : 输出操作数列表
    : 输入操作数列表
    : 破坏寄存器列表
);
```

### 2.2 示例

```cpp
// 原子加法（x86）
int atomic_add(int* ptr, int val) {
    int prev;
    asm volatile(
        "lock xaddl %0, %1"
        : "=r"(prev), "+m"(*ptr)
        : "0"(val)
        : "memory"
    );
    return prev + val;
}

// CPUID 查询
void cpuid(int code, int* a, int* b, int* c, int* d) {
    asm volatile("cpuid"
        : "=a"(*a), "=b"(*b), "=c"(*c), "=d"(*d)
        : "a"(code));
}
```

**注意事项**：
- 破坏寄存器列表必须完整，否则编译器生成错误代码
- `volatile` 防止编译器优化掉内联汇编
- MSVC 使用 `__asm { ... }` 语法

---

## 三、内存优化

### 3.1 内存对齐

```cpp
// 控制对齐
struct alignas(64) CacheLine {
    int data[16];
};

// 动态对齐分配
void* ptr = std::aligned_alloc(64, 1024);  // C++17

// 编译器指令
#pragma pack(push, 1)  // 1 字节对齐
struct Packed { char a; int b; };
#pragma pack(pop)
```

**对齐规则**：结构体大小为最大成员对齐的整数倍，成员按声明顺序布局。

### 3.2 伪共享（False Sharing）

多线程访问同一缓存行（64 字节）的不同变量，导致缓存行无效化频繁。

```cpp
// ❌ 伪共享：a 和 b 可能在同缓存行
struct alignas(64) Counter {
    std::atomic<int> a;
    char padding[60];  // 填充到 64 字节
    std::atomic<int> b;
};
```

### 3.3 NUMA 感知

```cpp
// NUMA 架构下，访问本地内存比远程快
// numactl 控制绑定
numactl --cpubind=0 --membind=0 ./program

// libnuma 编程接口
#include <numa.h>
void* ptr = numa_alloc_onnode(4096, 0);  // 在指定节点分配
```

---

## 四、零拷贝技术

| 方式 | API | 场景 |
|------|-----|------|
| `sendfile` | 文件→socket，内核态直接传输 | 文件服务器 |
| `splice` | 两个文件描述符间零拷贝 | 代理转发 |
| `mmap` | 文件映射到内存，省略 read | 大文件读写 |
| `writev` | 聚集写，减少系统调用 | 多缓冲拼接 |

```cpp
// sendfile 示例：文件到 socket
int file_fd = open("file.txt", O_RDONLY);
int sock_fd = ...;
off_t offset = 0;
sendfile(sock_fd, file_fd, &offset, file_size);
// 全程无需在用户态拷贝数据
```

---

## 五、性能分析工具

| 工具 | 用途 |
|------|------|
| `perf` | CPU 性能计数器、采样分析 |
| `gprof` | 函数级性能分析（需 `-pg` 编译） |
| `Valgrind` | 内存错误检测、缓存分析 |
| `callgrind` | 函数调用关系和耗时 |
| `FlameGraph` | 火焰图可视化热点 |
| `top / htop` | 实时进程/线程资源 |
| `/proc/<pid>/` | 进程级内核统计信息 |

---

## 六、实战建议与面试追问

### Q1: 什么时候需要手写 SIMD 而不是依赖编译器自动向量化？
- 自动向量化要求：连续内存、独立迭代、无函数调用、简单循环体
- 编译器的自动向量化能力有限（尤其对复杂数据访问模式）
- **建议**：先用 `-O3 -march=native` 让编译器尝试，perf 验证未向量化后再手动 SIMD

### Q2: 内联汇编的替代方案？
- 编译器 builtin：`__builtin_expect`、`__sync_fetch_and_add` 等
- Intel Intrinsics：`_mm256_add_ps` 等，不用写汇编但能生成 SIMD 指令
- `std::atomic`：原子操作跨平台，比手写 `lock xaddl` 更安全
- **建议**：能用 Intrinsics 就不用内联汇编（可移植性更好）

### Q3: perf 使用要点？
```bash
# 采样分析热点
perf record -g ./program           # -g 记录调用栈
perf report                        # 查看热点函数

# 统计缓存未命中
perf stat -e cache-misses,branch-misses ./program

# 生成火焰图
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > flame.svg
```

### Q4: 零拷贝技术选型速查？

| 场景 | 推荐 | 原因 |
|:----|:-----|:-----|
| 静态文件发送 | `sendfile` | 一行代码，内核完成 |
| 网络代理/转发 | `splice` | 两个 fd 间零拷贝 |
| 大文件随机读写 | `mmap` | 按需缺页，类似内存访问 |
| 小文件/频繁读写 | `read/write` | 延迟更可预测，缺页开销可控 |
| 多个缓冲区拼接 | `writev` / `sendmsg` | 一次系统调用合并发送 |
