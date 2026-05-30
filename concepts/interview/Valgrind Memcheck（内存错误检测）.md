# Valgrind Memcheck（内存错误检测）

## 1.1 基本使用

### 1.1.1 安装与验证

**安装 Valgrind**：
```bash
# Ubuntu/Debian
sudo apt-get install valgrind

# CentOS/RHEL
sudo yum install valgrind

# macOS (使用 Homebrew)
brew install valgrind

# 验证安装
valgrind --version
```

### 1.1.2 基本命令

```bash
# 基本检测（默认使用 Memcheck）
valgrind ./program

# 完整内存泄漏检测
valgrind --leak-check=full --show-leak-kinds=all ./program

# 跟踪未初始化内存的来源
valgrind --track-origins=yes ./program

# 生成详细报告到文件
valgrind --leak-check=full --log-file=valgrind.log ./program

# 显示所有错误（不限制数量）
valgrind --error-limit=no ./program

# 详细输出
valgrind --verbose ./program
```

**编译要求**：
```bash
# 必须使用 -g 生成调试信息，-O0 避免优化影响检测
g++ -g -O0 program.cpp -o program
```

---

## 1.2 检测内存泄漏

### 1.2.1 示例代码

```cpp
// memory_leak.cpp
#include <iostream>

void memory_leak_example() {
    int* p = new int(42);
    // 忘记 delete p;
    std::cout << *p << std::endl;
}

int main() {
    memory_leak_example();
    return 0;
}
```

### 1.2.2 编译和检测

```bash
# 编译（注意：使用 -g 生成调试信息，-O0 避免优化影响检测）
g++ -g -O0 memory_leak.cpp -o memory_leak

# 使用 Valgrind 检测
valgrind --leak-check=full --show-leak-kinds=all ./memory_leak
```

### 1.2.3 输出解读

```
==12345== HEAP SUMMARY:
==12345==     in use at exit: 4 bytes in 1 blocks
==12345==   total heap usage: 1 allocs, 0 frees, 4 bytes allocated
==12345== 
==12345== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==12345==    at 0x4C2AB80: operator new(unsigned long) (vg_replace_malloc.c:334)
==12345==    by 0x4005F6: memory_leak_example() (memory_leak.cpp:5)
==12345==    by 0x40060B: main (memory_leak.cpp:9)
```

**关键信息**：
- `in use at exit`：程序退出时仍在使用的内存
- `definitely lost`：确定的内存泄漏
- 调用栈：显示内存分配的位置

### 1.2.4 泄漏类型详解

| 泄漏类型 | 说明 | 示例场景 |
|---------|------|---------|
| **definitely lost** | 确定的内存泄漏，程序结束时没有任何指针指向该内存 | `new` 后忘记 `delete` |
| **indirectly lost** | 间接泄漏，通常由其他泄漏导致 | 容器中的对象泄漏，如 `vector<Object*>` 中对象未释放 |
| **possibly lost** | 可能泄漏，指针可能指向内存中间位置 | 指针算术后丢失起始地址，如 `p = p + 10` 后无法找到原始地址 |
| **still reachable** | 仍然可达，程序结束时仍有指针指向，但可能不是真正的泄漏 | 全局变量、静态变量指向的内存 |

---

## 1.3 检测使用已释放内存（Use After Free）

### 1.3.1 示例代码

```cpp
// use_after_free.cpp
#include <iostream>

int main() {
    int* p = new int(42);
    delete p;
    *p = 100;  // 错误：使用已释放的内存
    std::cout << *p << std::endl;
    return 0;
}
```

### 1.3.2 检测命令

```bash
g++ -g -O0 use_after_free.cpp -o use_after_free
valgrind ./use_after_free
```

### 1.3.3 输出示例

```
==12345== Invalid write of size 4
==12345==    at 0x4005F6: main (use_after_free.cpp:6)
==12345==  Address 0x5203040 is 0 bytes inside a block of size 4 free'd
==12345==    at 0x4C2A0E0: operator delete(void*) (vg_replace_malloc.c:576)
==12345==    by 0x4005F1: main (use_after_free.cpp:5)
==12345== 
==12345== Invalid read of size 4
==12345==    at 0x40060A: main (use_after_free.cpp:7)
==12345==  Address 0x5203040 is 0 bytes inside a block of size 4 free'd
```

**关键信息**：
- `Invalid write/read`：非法写入/读取
- `free'd`：内存已被释放
- 显示释放位置和使用位置

---

## 1.4 检测数组越界

### 1.4.1 示例代码

```cpp
// buffer_overflow.cpp
#include <iostream>

int main() {
    int arr[10];
    arr[10] = 42;  // 错误：越界访问（有效索引是 0-9）
    std::cout << arr[10] << std::endl;
    return 0;
}
```

### 1.4.2 检测命令

```bash
g++ -g -O0 buffer_overflow.cpp -o buffer_overflow
valgrind ./buffer_overflow
```

### 1.4.3 输出示例

```
==12345== Invalid write of size 4
==12345==    at 0x4005F6: main (buffer_overflow.cpp:5)
==12345==  Address 0x1fff00028 is 0 bytes after a block of size 40 alloc'd
==12345==    at 0x4005E0: main (buffer_overflow.cpp:4)
==12345== 
==12345== Invalid read of size 4
==12345==    at 0x40060A: main (buffer_overflow.cpp:6)
==12345==  Address 0x1fff00028 is 0 bytes after a block of size 40 alloc'd
```

**关键信息**：
- `after a block`：访问了分配块之后的内存
- `size 40`：数组大小为 40 字节（10 个 int）

---

## 1.5 检测未初始化内存

### 1.5.1 示例代码

```cpp
// uninitialized.cpp
#include <iostream>

int main() {
    int x;  // 未初始化
    std::cout << x << std::endl;  // 错误：使用未初始化的变量
    return 0;
}
```

### 1.5.2 检测命令

```bash
g++ -g -O0 uninitialized.cpp -o uninitialized
# 必须使用 --track-origins=yes 才能看到未初始化内存的来源
valgrind --track-origins=yes ./uninitialized
```

### 1.5.3 输出示例

```
==12345== Use of uninitialised value of size 4
==12345==    at 0x4005F6: main (uninitialized.cpp:5)
==12345==  Uninitialised value was created by a stack allocation
==12345==    at 0x4005E0: main (uninitialized.cpp:4)
```

**关键信息**：
- `Use of uninitialised value`：使用了未初始化的值
- `stack allocation`：未初始化值来自栈分配

---

## 1.6 检测重复释放（Double Free）

### 1.6.1 示例代码

```cpp
// double_free.cpp
#include <iostream>

int main() {
    int* p = new int(42);
    delete p;
    delete p;  // 错误：重复释放
    return 0;
}
```

### 1.6.2 检测命令

```bash
g++ -g -O0 double_free.cpp -o double_free
valgrind ./double_free
```

### 1.6.3 输出示例

```
==12345== Invalid free() / delete / delete[] / realloc()
==12345==    at 0x4C2A0E0: operator delete(void*) (vg_replace_malloc.c:576)
==12345==    by 0x4005F6: main (double_free.cpp:6)
==12345==  Address 0x5203040 is 0 bytes inside a block of size 4 free'd
==12345==    at 0x4C2A0E0: operator delete(void*) (vg_replace_malloc.c:576)
==12345==    by 0x4005F1: main (double_free.cpp:5)
```

**关键信息**：
- `Invalid free()`：非法的释放操作
- `free'd`：内存已经被释放过

---

## 1.7 Memcheck 常用选项

| 选项 | 说明 | 示例 | 使用场景 |
|------|------|------|---------|
| `--leak-check=<no|summary|full>` | 泄漏检测级别 | `--leak-check=full` | 需要详细泄漏信息时 |
| `--show-leak-kinds=<kind>` | 显示泄漏类型 | `--show-leak-kinds=all` | 显示所有类型的泄漏 |
| `--track-origins=yes` | 跟踪未初始化内存来源 | `--track-origins=yes` | 调试未初始化内存问题时 |
| `--log-file=<file>` | 输出到文件 | `--log-file=valgrind.log` | 需要保存报告时 |
| `--suppressions=<file>` | 使用抑制文件 | `--suppressions=suppress.supp` | 过滤误报时 |
| `--verbose` | 详细输出 | `--verbose` | 需要详细信息时 |
| `--error-limit=no` | 不限制错误数量 | `--error-limit=no` | 错误很多时 |
| `--show-reachable=yes` | 显示可达内存 | `--show-reachable=yes` | 分析 still reachable 时 |
| `--leak-resolution=high` | 高分辨率泄漏检测 | `--leak-resolution=high` | 需要精确泄漏信息时 |

[src: raw/ingested/2技术/性能优化/内存优化-valgrind-1.-Valgrind-Memcheck（内存错误检测）.md]

---

## 6. 常见问题与解决方案

### 6.1 Valgrind 误报问题

**问题**：Valgrind 报告大量误报，如何过滤？

**解决方案**：
```bash
# 使用抑制文件
valgrind --suppressions=suppress.supp ./program

# 创建抑制文件示例
cat > suppress.supp << EOF
{
   ignore_libc_leaks
   Memcheck:Leak
   match-leak-kinds: reachable
   ...
   fun:malloc
}
EOF
```

**常见误报来源**：
- 系统库的内存分配
- 第三方库的已知问题
- 编译器生成代码的内存操作

---

### 6.2 无法定位问题位置

**问题**：Valgrind 报告了错误但无法定位代码位置

**解决方案**：
1. **确保使用 `-g` 编译选项**：
   ```bash
   g++ -g -O0 program.cpp -o program
   ```

2. **使用 `-fno-omit-frame-pointer`**：
   ```bash
   g++ -g -O0 -fno-omit-frame-pointer program.cpp -o program
   ```

3. **检查符号表**：
   ```bash
   nm program | grep main
   objdump -t program | grep main
   ```

4. **使用 `addr2line` 转换地址**：
   ```bash
   addr2line -e program 0x4005f6
   ```

---

### 6.3 工具运行太慢

**问题**：Valgrind 运行太慢，如何加速？

**解决方案**：
1. **减少检测范围**：
   ```bash
   # 只检测泄漏，不检测其他问题
   valgrind --leak-check=full --track-origins=no ./program
   ```

2. **使用 AddressSanitizer 替代**（更快）：
   ```bash
   g++ -fsanitize=address -g -O1 program.cpp -o program
   ```

3. **减少检测的代码范围**：
   - 只检测特定函数
   - 使用 `--suppressions` 过滤已知问题

4. **使用系统工具监控**（几乎无性能影响）：
   ```bash
   pmap -x <pid>
   top -p <pid>
   ```

---

### 6.4 检测不到某些问题

**问题**：Valgrind 检测不到某些内存问题

**解决方案**：
1. **确保编译选项正确**：
   - 使用 `-g -O0`
   - 不要使用 `-O2` 或更高优化级别

2. **检查是否在检测范围内**：
   - Valgrind 只能检测堆内存和栈内存
   - 无法检测静态内存的问题

3. **使用其他工具补充**：
   - AddressSanitizer：快速检测内存错误
   - 静态分析工具：检测代码问题

4. **检查抑制文件**：
   - 确保没有意外抑制了相关错误

---

### 6.5 Massif 输出文件找不到

**问题**：运行 Massif 后找不到输出文件

**解决方案**：
1. **明确指定输出文件**：
   ```bash
   valgrind --tool=massif --massif-out-file=massif.out ./program
   ```

2. **查找输出文件**：
   ```bash
   # 输出文件通常在当前目录
   ls -la massif.out.*

   # 或查找所有 massif 文件
   find . -name "massif.out.*"
   ```

3. **检查权限**：
   - 确保有写入权限
   - 检查磁盘空间

[src: raw/ingested/2技术/性能优化/内存优化-valgrind-6.-常见问题与解决方案.md]