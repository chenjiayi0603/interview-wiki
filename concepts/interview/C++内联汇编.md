# C++内联汇编

See also: [[C++语言特性]], [[C++进阶知识点]], [[性能优化]]

## 概述

在C++程序中使用汇编语言主要有以下几种方式：
- **内联汇编（Inline Assembly）**：在C++代码中直接嵌入汇编指令
- **独立汇编文件**：编写独立的`.s`或`.asm`文件，通过链接器链接
- **编译器内联函数**：使用编译器提供的内置函数（intrinsics）

### 为什么使用汇编？
1. **性能优化**：对关键路径进行极致优化
2. **硬件访问**：直接访问CPU特殊寄存器或指令
3. **系统调用**：实现底层系统功能
4. **学习研究**：理解编译器生成的代码

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## GCC内联汇编

### 基本语法

```cpp
asm [volatile] (
    "汇编指令模板"
    : 输出操作数列表
    : 输入操作数列表
    : 破坏寄存器列表
);
```

### 扩展内联汇编（Extended Inline Assembly）

#### 语法结构
```cpp
asm [volatile] (
    "汇编代码模板"
    : [输出操作数] "约束"(变量)
    : [输入操作数] "约束"(变量)
    : [破坏寄存器列表]
    : [标签列表]
);
```

#### 操作数约束

**输出约束：**
- `=`：只写操作数
- `+`：读写操作数
- `&`：早期破坏操作数

**输入约束：**
- `r`：通用寄存器
- `m`：内存操作数
- `i`：立即数
- `n`：编译时已知的立即数
- `g`：通用约束（寄存器、内存或立即数）

**寄存器约束：**
- `a`：只用 eax/ax/al（累加器）
- `b`：只用 ebx/bx/bl（基址寄存器）
- `c`：只用 ecx/cx/cl（计数寄存器）
- `d`：只用 edx/dx/dl（数据寄存器）
- `S`：只用 esi/si（源索引寄存器）
- `D`：只用 edi/di（目的索引寄存器）

#### 示例1：简单加法
```cpp
int add(int a, int b) {
    int result;
    asm(
        "addl %2, %1\n\t"
        "movl %1, %0"
        : "=r"(result)      // 输出：result
        : "r"(a), "r"(b)    // 输入：a, b
        : "cc"              // 破坏：条件码寄存器
    );
    return result;
}
```

#### 示例2：使用特定寄存器
```cpp
int multiply_by_10(int x) {
    asm(
        "leal (%0,%0,4), %0\n\t"  // x = x + 4*x = 5*x
        "addl %0, %0"              // x = 2*x = 10*x
        : "+r"(x)                  // 读写操作数
        :
        : "cc"
    );
    return x;
}
```

#### 示例3：内存操作
```cpp
void set_memory(int* ptr, int value) {
    asm(
        "movl %1, (%0)"
        :
        : "r"(ptr), "r"(value)
        : "memory"
    );
}
```

#### 示例4：读取CPU特性
```cpp
void get_cpu_info() {
    unsigned int eax, ebx, ecx, edx;
    asm(
        "cpuid"
        : "=a"(eax), "=b"(ebx), "=c"(ecx), "=d"(edx)
        : "a"(1)
        :
    );
    printf("CPU Features: %08x\n", edx);
}
```

#### 示例5：原子操作
```cpp
int atomic_increment(int* value) {
    int result;
    asm volatile(
        "lock incl %0"
        : "+m"(*value)
        :
        : "cc", "memory"
    );
    return *value;
}
```

#### 示例6：64位乘法
```cpp
long long multiply_64(int a, int b) {
    long long result;
    asm(
        "imull %2"
        : "=A"(result)  // A约束表示edx:eax
        : "a"(a), "r"(b)
        : "cc"
    );
    return result;
}
```

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## MSVC内联汇编

### 基本语法

```cpp
__asm {
    mov eax, 5      // 把5放入eax
    mov ebx, 10     // 把10放入ebx
    add eax, ebx    // eax = eax + ebx
}
```

### 示例1：简单计算
```cpp
int add_msvc(int a, int b) {
    int result;
    __asm {
        mov eax, a
        add eax, b
        mov result, eax
    }
    return result;
}
```

### 示例2：访问C++变量
```cpp
void swap_msvc(int& a, int& b) {
    __asm {
        mov eax, a
        mov ebx, b
        mov a, ebx
        mov b, eax
    }
}
```

### 注意事项
- MSVC的`__asm`块在x64模式下**不支持**
- x64模式下需要使用编译器内置函数（intrinsics）或独立汇编文件

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## Clang内联汇编

Clang基本兼容GCC的内联汇编语法，但有一些差异：

- **GCC（GNU Compiler Collection）** 是历史最悠久、最广泛使用的开源C/C++编译器套件。
- **Clang** 是由LLVM社区主导开发的现代C/C++/Objective-C编译器前端，与LLVM后端配合使用，以模块化、易维护和扩展性强著称。

二者都支持C/C++的绝大多数标准。Clang 的内联汇编基本兼容 GCC（尤其是 x86 平台下），都是类Unix主流平台（Linux、*BSD、macOS）的C/C++编译器首选之一，可以互相替换多数场景。

不同点在于：GCC项目独立开发，有自己专有后端和优化链路；Clang是LLVM项目一部分，前端与LLVM后端/优化器协同工作。Clang在诊断和错误提示、AST工具链等方面表现更优，并注重可扩展性和代码质量分析。

```cpp
// Clang支持GCC语法
int add_clang(int a, int b) {
    int result;
    __asm__(
        "addl %2, %1\n\t"
        "movl %1, %0"
        : "=r"(result)
        : "r"(a), "r"(b)
        : "cc"
    );
    return result;
}
```

### Clang特有特性
- 更好的错误检查
- 支持更多架构（ARM、AArch64等）
- 更好的优化集成

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## 常见使用场景

### 1. 性能关键路径优化

```cpp
// 快速字符串复制（使用rep movsb）
void fast_memcpy(void* dest, const void* src, size_t n) {
    asm volatile(
        "rep movsb"
        : "+D"(dest), "+S"(src), "+c"(n)
        :
        : "memory"
    );
}
```

### 2. 位操作优化

```cpp
// 计算位1的个数（popcount）
int popcount_asm(unsigned int x) {
    int count;
    asm(
        "popcnt %1, %0"
        : "=r"(count)
        : "r"(x)
    );
    return count;
}
```

### 3. 浮点运算优化

```cpp
// 快速平方根倒数（Quake III算法）
float fast_inv_sqrt(float x) {
    float result;
    asm(
        "rsqrtss %1, %0"
        : "=x"(result)
        : "x"(x)
    );
    return result;
}
```

### 4. 系统调用

```cpp
// Linux系统调用示例
long syscall_asm(long number, long arg1, long arg2, long arg3) {
    long result;
    asm volatile(
        "syscall"
        : "=a"(result)
        : "a"(number), "D"(arg1), "S"(arg2), "d"(arg3)
        : "rcx", "r11", "memory"
    );
    return result;
}
```

### 5. 内存屏障

```cpp
// CPU内存屏障
asm volatile("mfence" ::: "memory");

// 编译器内存屏障
asm volatile("" ::: "memory");
```

**mfence 与编译器屏障的区别：**
- `mfence`：CPU指令级内存屏障，强制CPU在屏障前的所有读写操作完成后才执行屏障后的操作，防止CPU乱序执行带来的可见性问题。适用于多处理器共享内存场景。
- 编译器屏障（`asm volatile("" ::: "memory")`）：只阻止编译器优化时的重排序，不生成真正的机器指令，不影响CPU执行顺序。
- 二者结合使用：`asm volatile("mfence" ::: "memory")` 既防CPU也防编译器对内存操作的乱序。

### 6. CPU特性检测

```cpp
// 检测 CPU 是否支持 SSE4.2 指令集
bool has_sse42() {
    unsigned int eax, ebx, ecx, edx;
    asm volatile("cpuid"
                 : "=a"(eax), "=b"(ebx), "=c"(ecx), "=d"(edx)
                 : "a"(1)
                 : "cc");
    // ECX 寄存器的第 20 位（bit 20）表示 SSE4.2 支持
    return (ecx & (1 << 20)) != 0;
}
```

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## 性能优化示例

### 示例1：快速数组求和（SIMD）

```cpp
#include <immintrin.h>

float sum_array_simd(const float* array, size_t size) {
    __m128 sum = _mm_setzero_ps();
    size_t i = 0;

    for (; i + 4 <= size; i += 4) {
        __m128 v = _mm_loadu_ps(&array[i]);
        sum = _mm_add_ps(sum, v);
    }

    float temp[4];
    _mm_storeu_ps(temp, sum);
    float total = temp[0] + temp[1] + temp[2] + temp[3];

    for (; i < size; ++i) {
        total += array[i];
    }
    return total;
}
```

**SIMD（Single Instruction, Multiple Data，单指令多数据）** 是一种并行计算技术，允许处理器在一条指令下同时对多个数据执行相同的操作。常用于向量处理、大数据量的循环等场景，显著提升数据处理性能。

### 示例2：原子操作实现

```cpp
// 原子比较并交换（CAS）
bool compare_and_swap(int* ptr, int expected, int desired) {
    int old;
    asm volatile(
        "lock cmpxchgl %2, %1"
        : "=a"(old), "+m"(*ptr)
        : "r"(desired), "a"(expected)
        : "cc", "memory"
    );
    return old == expected;
}
```

### 示例3：循环展开优化

```cpp
void process_array_asm(int* arr, size_t size) {
    size_t i = 0;
    for (; i + 4 <= size; i += 4) {
        asm volatile(
            "addl $1, %0\n\t"
            "addl $1, %1\n\t"
            "addl $1, %2\n\t"
            "addl $1, %3"
            : "+m"(arr[i]), "+m"(arr[i+1]),
              "+m"(arr[i+2]), "+m"(arr[i+3])
            :
            : "cc"
        );
    }
    for (; i < size; ++i) {
        arr[i] += 1;
    }
}
```

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## 最佳实践

### 1. 使用volatile关键字
```cpp
asm volatile(
    "addl $1, %0"
    : "+m"(arr[i])
    :
    : "cc"
);
```

### 2. 正确声明破坏的寄存器
```cpp
asm volatile (
    "movl (%0), %%eax;"
    "xchgl (%1), %%eax;"
    "movl %%eax, (%0);"
    :
    : "r"(a), "r"(b)
    : "eax", "memory", "cc"
);
```

### 3. 避免不必要的内联汇编
- 优先使用编译器内置函数（intrinsics）
- 让编译器进行优化
- 只在确实需要时使用汇编

**编译器内置函数（intrinsics）** 是指编译器直接提供的一些特殊函数，这些函数通常会直接映射到某条CPU指令或一组高效的指令。例如：
- `__builtin_popcount(x)`：统计一个整数中1的个数，等价于x86的`POPCNT`指令
- `__builtin_expect(cond, 1)`：提示编译器分支预测
- `__mm_prefetch(const char* p, int i)`：预取内存到缓存

### 4. 跨平台考虑
```cpp
#ifdef __GNUC__
    // GCC/Clang内联汇编
    asm(...);
#elif defined(_MSC_VER)
    // MSVC内联汇编（仅x86）
    __asm {...}
#else
    // 回退到C++实现
#endif
```

### 5. 文档和注释
- 使用内联汇编实现快速位计数
- 使用x86-64的POPCNT指令
- 要求CPU支持SSE4.2

### 6. 性能测试
- 使用基准测试验证性能提升
- 比较汇编版本和C++版本的性能
- 考虑可读性和维护性

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## 调试技巧

### 1. 查看生成的汇编代码

#### GCC/Clang
```bash
g++ -S -o output.s source.cpp
```

#### MSVC
```bash
cl /FA source.cpp
```

### 2. 使用objdump反汇编
```bash
objdump -d my_program > asm_output.txt
```

### 3. 使用GDB调试
```bash
(gdb) disassemble function_name
(gdb) stepi  # 单步执行汇编指令
(gdb) info registers  # 查看寄存器值
```

### 4. 编译器优化级别影响
- `-O0`：保持内联汇编原样
- `-O2/-O3`：可能重新排序或优化周围的代码
- 使用volatile防止过度优化

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## 实际应用案例

### 案例1：高性能哈希函数

```cpp
// MurmurHash3的汇编优化版本
uint32_t murmur_hash3_asm(const void* key, size_t len, uint32_t seed) {
    const uint8_t* data = (const uint8_t*)key;
    uint32_t h = seed;
    uint32_t k;
    
    while (len >= 4) {
        asm(
            "movl %1, %0\n\t"
            "imull $0xcc9e2d51, %0\n\t"
            "roll $15, %0\n\t"
            "imull $0x1b873593, %0"
            : "=r"(k)
            : "m"(*(uint32_t*)data)
            : "cc"
        );
        h ^= k;
        h = (h << 13) | (h >> 19);
        h = h * 5 + 0xe6546b64;
        data += 4;
        len -= 4;
    }
    return h;
}
```

### 案例2：快速排序关键部分

```cpp
void quicksort_partition_asm(int* arr, int low, int high) {
    int pivot = arr[high];
    int i = low - 1;

    for (int j = low; j < high; j++) {
        bool less;
        asm volatile(
            "cmpl %2, %1\n\t"
            "setl %0"
            : "=r"(less)
            : "r"(arr[j]), "r"(pivot)
            : "cc"
        );
        if (less) {
            i++;
            asm volatile(
                "movl %0, %%eax\n\t"
                "movl %1, %%ebx\n\t"
                "movl %%ebx, %0\n\t"
                "movl %%eax, %1\n\t"
                : "+m"(arr[i]), "+m"(arr[j])
                :
                : "eax", "ebx", "memory"
            );
        }
    }
    asm volatile(
        "movl %0, %%eax\n\t"
        "movl %1, %%ebx\n\t"
        "movl %%ebx, %0\n\t"
        "movl %%eax, %1\n\t"
        : "+m"(arr[i+1]), "+m"(arr[high])
        :
        : "eax", "ebx", "memory"
    );
}
```

### 案例3：加密算法优化

```cpp
// AES加密的S-box查找优化
uint8_t aes_sbox_lookup(uint8_t value) {
    uint8_t result;
    asm(
        "movzbl %1, %0\n\t"
        "movb sbox_table(,%0,1), %0"
        : "=r"(result)
        : "r"(value)
        : "memory"
    );
    return result;
}
```

### 案例4：时间戳获取

```cpp
// 使用RDTSC指令获取高精度时间戳
uint64_t rdtsc() {
    uint32_t lo, hi;
    asm volatile(
        "rdtsc"
        : "=a"(lo), "=d"(hi)
        :
        : "cc"
    );
    return ((uint64_t)hi << 32) | lo;
}
```

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## 常见问题和陷阱

### 1. 寄存器破坏未声明
```cpp
// 错误示例
int bad_example(int x) {
    int result;
    asm(
        "movl %1, %%eax\n\t"
        "addl $10, %%eax\n\t"
        "movl %%eax, %0"
        : "=r"(result)
        : "r"(x)
        // 缺少: "eax" 声明
    );
    return result;
}
```

### 2. 内存屏障缺失
```cpp
// 多线程环境下需要内存屏障
void unsafe_atomic_op(int* ptr) {
     asm("incl %0" : "+m"(*ptr));  // 缺少lock前缀和内存屏障
}
```

### 3. 平台相关代码未保护
```cpp
#ifdef __x86_64__
    // x86-64特定代码
#elif defined(__aarch64__)
    // ARM64特定代码
#else
    // 通用实现
#endif
```

**x86-64与ARM64汇编的区别：**
- x86-64：使用如rax、rbx、rcx、rdx等寄存器，指令如mov、add、sub、mul等
- ARM64（AArch64）：使用x0~x30通用寄存器，指令如add、ldr、str等
- 汇编代码不能直接通用，需注意语法、寄存器、指令差异

### 4. 编译器优化冲突
```cpp
// 使用volatile防止优化
asm volatile("nop" ::: "memory");
```

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## 总结

### 使用汇编的时机
1. ✅ 性能关键路径，编译器优化不足
2. ✅ 需要访问特殊CPU指令或寄存器
3. ✅ 实现原子操作或内存屏障
4. ✅ 系统级编程需求

### 避免使用汇编的时机
1. ❌ 可以用C++和编译器优化达到相同效果
2. ❌ 代码可读性和维护性更重要
3. ❌ 跨平台兼容性是主要考虑
4. ❌ 团队缺乏汇编经验

### 关键要点
- 优先使用编译器内置函数（intrinsics）
- 充分测试和验证汇编代码
- 提供清晰的文档和注释
- 考虑可移植性和维护成本
- 使用适当的工具进行调试和性能分析

[src: raw/ingested/2技术/cpp/c++嵌入汇编的使用分析.md]

## Related Pages
- [[C++语言特性]]
- [[C++进阶知识点]]
- [[性能优化]]
- [[SIMD指令优化]]
