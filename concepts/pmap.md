# pmap（进程内存映射）

## 基本使用

**pmap 用途**：
- 查看进程的内存映射
- 分析内存使用分布
- 定位内存占用热点

**基本命令**：
```bash
# 查看进程内存映射
pmap -x <pid>

# 显示详细信息
pmap -d <pid>

# 显示所有线程的内存映射
pmap -x <pid> -p

# 持续监控
watch -n 1 'pmap -x <pid>'
```

**输出示例**：
```
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000       4       4       0 r-x--  program
0000000000600000       4       4       4 rw---  program
00007f1234567000    1024     512     512 rw---  [heap]
00007f1234568000     132     132       0 r-x--  libc-2.23.so
...
```

**字段说明**：
- **Address**：内存地址范围
- **Kbytes**：虚拟内存大小（KB）
- **RSS**：实际物理内存使用（KB）
- **Dirty**：脏页大小（KB）
- **Mode**：内存权限（r=读，w=写，x=执行）
- **Mapping**：内存映射类型（程序、库、堆、栈等）

## 分析内存使用

**查找内存占用大的区域**：
```bash
# 按 RSS 排序
pmap -x <pid> | sort -k3 -n -r

# 查看堆内存
pmap -x <pid> | grep heap

# 查看共享库内存
pmap -x <pid> | grep "\.so"
```

## 完整实战示例：从程序到 pmap 分析

**示例程序**（`heap_demo.cpp`，分配堆内存并持续约 10 秒便于观察）：
```cpp
// heap_demo.cpp
#include <iostream>
#include <cstring>
#include <unistd.h>

int main() {
    const size_t size = 2 * 1024 * 1024;  // 2MB
    char* p = new char[size];
    memset(p, 0xAB, size);
    std::cout << "PID=" << getpid() << ", 已分配 2MB 堆，10 秒后退出\n";
    sleep(10);
    delete[] p;
    return 0;
}
```

**完整操作流程**：
```bash
# 1. 编译
g++ -O0 -o heap_demo heap_demo.cpp

# 2. 后台运行并记下 PID
./heap_demo &
PID=$!
echo "PID=$PID"

# 3. 查看该进程内存映射（在程序 sleep 期间执行）
pmap -x $PID

# 4. 按 RSS 排序找大块
pmap -x $PID | sort -k3 -n -r | head -20

# 5. 只看堆
pmap -x $PID | grep heap

# 6. 等待进程结束
wait
```

**pmap -x 输出示例（片段）**：
```
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000       4       4       0 r-x--  heap_demo
0000000000600000       4       4       4 rw---  heap_demo
0000000001a2a000   2052    2048    2048 rw---  [heap]   <-- 约 2MB 堆
00007f1234567000     132     132       0 r-xp  libc-2.31.so
...
total kB           -----   -----   -----
```

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-3.-系统工具详解.md]