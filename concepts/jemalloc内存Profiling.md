# jemalloc 内存 Profiling

jemalloc 内置强大的内存分析功能，可以追踪内存分配热点和检测内存泄漏。

## 启用 Profiling

```bash
# 通过环境变量启用 profiling
export MALLOC_CONF="prof:true,prof_prefix:jeprof.out"
./your_program

# 更详细的配置
export MALLOC_CONF="prof:true,prof_active:true,lg_prof_sample:17,prof_prefix:./heap"
./your_program
```

**配置参数说明**：
| 参数 | 说明 |
|------|------|
| `prof:true` | 启用 profiling |
| `prof_active:true` | 程序启动时立即开始采样 |
| `lg_prof_sample:N` | 采样间隔（2^N 字节），默认 19（512KB） |
| `prof_prefix:path` | 输出文件前缀 |
| `prof_leak:true` | 启用泄漏检测 |
| `prof_final:true` | 程序退出时生成最终报告 |

## 生成和分析 Heap Profile

```bash
# 1. 运行程序（启用 profiling）
export MALLOC_CONF="prof:true,prof_leak:true,prof_final:true,prof_prefix:./heap"
./your_program

# 2. 使用 jeprof 分析（类似 pprof）
# 查看文本报告
jeprof --text ./your_program ./heap.*.heap

# 生成调用图（PDF）
jeprof --pdf ./your_program ./heap.*.heap > heap_profile.pdf

# 生成火焰图格式
jeprof --collapsed ./your_program ./heap.*.heap > heap.collapsed
flamegraph.pl heap.collapsed > heap_flame.svg
```

## 代码中控制 Profiling

```cpp
#include <jemalloc/jemalloc.h>

void dump_heap_profile(const char* filename) {
    // 手动触发 heap dump
    mallctl("prof.dump", NULL, NULL, &filename, sizeof(filename));
}

void toggle_profiling(bool enable) {
    // 动态开启/关闭 profiling
    mallctl("prof.active", NULL, NULL, &enable, sizeof(enable));
}

int main() {
    // 启动时 profiling 可能是关闭的
    toggle_profiling(true);
    
    // ... 执行需要分析的代码 ...
    
    // 手动 dump 当前堆状态
    dump_heap_profile("manual_dump.heap");
    
    toggle_profiling(false);
    return 0;
}
```

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-3.-内存-Profiling-功能.md]