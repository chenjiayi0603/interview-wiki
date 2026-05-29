# jemalloc 统计信息与监控

> 本文介绍 jemalloc 内存分配器的统计信息获取和关键指标解读，帮助开发者监控内存使用、计算碎片率。

See also: [[内存管理]], [[C++STL内存管理]], [[TBB最佳实践]]

## 获取内存统计

```cpp
#include <jemalloc/jemalloc.h>
#include <iostream>
#include <cstring>

void print_jemalloc_stats() {
    // 方法1：打印详细统计到 stderr
    malloc_stats_print(NULL, NULL, NULL);
    
    // 方法2：获取特定指标
    size_t allocated, active, resident, mapped;
    size_t sz = sizeof(size_t);
    
    mallctl("stats.allocated", &allocated, &sz, NULL, 0);
    mallctl("stats.active", &active, &sz, NULL, 0);
    mallctl("stats.resident", &resident, &sz, NULL, 0);
    mallctl("stats.mapped", &mapped, &sz, NULL, 0);
    
    std::cout << "已分配内存: " << allocated / (1024.0 * 1024) << " MB\n";
    std::cout << "活跃内存:   " << active / (1024.0 * 1024) << " MB\n";
    std::cout << "驻留内存:   " << resident / (1024.0 * 1024) << " MB\n";
    std::cout << "映射内存:   " << mapped / (1024.0 * 1024) << " MB\n";
}

int main() {
    // 分配一些内存
    std::vector<void*> ptrs;
    for (int i = 0; i < 1000; ++i) {
        ptrs.push_back(malloc(1024 * 10));  // 10KB each
    }
    
    print_jemalloc_stats();
    
    // 释放内存
    for (auto p : ptrs) free(p);
    
    return 0;
}
```

**输出示例**：
```
已分配内存: 9.77 MB
活跃内存:   10.24 MB
驻留内存:   12.00 MB
映射内存:   16.00 MB
```

## 关键统计指标解读

| 指标 | 含义 |
|------|------|
| `stats.allocated` | 应用程序已分配的字节数 |
| `stats.active` | 分配器活跃使用的字节数（含元数据） |
| `stats.resident` | 实际占用的物理内存 |
| `stats.mapped` | 映射的虚拟内存总量 |
| `stats.retained` | 保留但未归还给操作系统的内存 |

**内存碎片率计算**：
```cpp
double fragmentation = (double)active / allocated - 1.0;
std::cout << "碎片率: " << fragmentation * 100 << "%" << std::endl;
```

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-4.-统计信息与监控.md]