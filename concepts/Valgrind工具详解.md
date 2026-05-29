# Valgrind（内存和性能分析）

**用途**：内存泄漏检测、函数级性能分析、缓存命中分析。

**常用命令**：
```bash
# 内存泄漏与分析
valgrind --leak-check=full ./program

# Callgrind函数级性能分析
valgrind --tool=callgrind ./program
kcachegrind callgrind.out.*    # 可视化评估

# Cache命中分析
valgrind --tool=cachegrind ./program
```

**输出示例**：
```
==12345== Memcheck, a memory error detector
==12345== LEAK SUMMARY:
==12345==    definitely lost: 8 bytes in 1 blocks
==12345==    indirectly lost: 0 bytes in 0 blocks
==12345==      possibly lost: 0 bytes in 0 blocks
==12345==    still reachable: 72,704 bytes in 1 blocks
==12345==         suppressed: 0 bytes in 0 blocks
...
```

**特点**：
- callgrind/cachegrind输出可在 kcachegrind 中可视化
- cachegrind 输出 cache miss 统计
- 相比perf，功能更全面但性能开销更大

[src: raw/ingested/2技术/性能优化/内存优化-perf-附录：其他性能分析工具.md]