# AddressSanitizer - 内存错误检测

**用途**：检测内存越界、使用未初始化内存、内存泄漏等问题。

## 使用方法

```bash
# 编译时启用 AddressSanitizer
g++ -fsanitize=address -g -O1 program.cpp -o program

# 运行程序
./program
```

## 输出示例

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x604000000014 at pc 0x000000400967 bp 0x7ffde11bcdf0 sp 0x7ffde11bcde8
WRITE of size 4 at 0x604000000014 thread T0
    #0 0x400966 in main example.cpp:5
    #1 0x7f66eb9a8830 in __libc_start_main (/lib64/libc.so.6+0x20830)
    #2 0x4007c9 in _start (a.out+0x4007c9)

0x604000000014 is located 0 bytes to the right of 20-byte region [0x604000000000,0x604000000014)
allocated by thread T0 here:
    #0 0x7f976dfcd110 in operator new[](unsigned long) (/usr/lib/x86_64-linux-gnu/libasan.so.4+0xde110)
    #1 0x400896 in main example.cpp:3
```

## 常见问题定位

- 会输出详细的错误类型、地址、调用栈及分配位置
- 非常适合定位缓冲区溢出、Use-after-free、内存越界等问题

## 示例代码

```cpp
#include <iostream>

int main() {
    int* arr = new int[5];
    for (int i = 0; i <= 5; ++i) { // 越界写，i 最多只能到 4
        arr[i] = i;
    }
    delete[] arr;
    return 0;
}
```

编译并运行（带 AddressSanitizer）后，会输出内存越界错误信息，定位出错位置。

## 特点

- 运行时检测，无需额外工具
- 性能开销约 2-3 倍
- 可以精确定位内存错误位置

## 注意事项

- AddressSanitizer 只有在程序实际运行到发生越界访问（如非法写/读）时，才会检测到并立即输出错误信息。
- 编译指令（推荐带调试信息和较低优化等级，方便调试）：`g++ -fsanitize=address -g -O1 example.cpp -o example`
- 也可用 clang++ 替换 g++，命令格式类似：`clang++ -fsanitize=address -g -O1 example.cpp -o example`
- 使用 `-fsanitize=address` 编译的程序，AddressSanitizer 检查到内存错误时，相关错误报告会自动打印到控制台（通常是标准错误 stderr）。

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-其他性能分析工具.md]