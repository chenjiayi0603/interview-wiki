# valgrind 性能分析

valgrind 是 Linux 下的性能分析工具套件，包含 cachegrind（缓存分析）和 callgrind（调用图分析）等工具。

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-vtune-简介与使用-vtune-简介与使用.md]

## valgrind cachegrind（缓存分析）

```bash
valgrind --tool=cachegrind ./hotspot_lock_test 4 1000000
```

Cachegrind 输出包括：
- I refs: 指令引用次数
- D refs: 数据引用次数
- I1/D1 misses: 一级缓存未命中次数
- LL misses: 最后一级缓存未命中次数
- Miss rate: 未命中率

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-vtune-简介与使用-vtune-简介与使用.md]

## valgrind callgrind（调用图分析）

```bash
valgrind --tool=callgrind ./hotspot_lock_test 4 1000000
```

Callgrind 通过动态插桩运行程序，统计函数级的调用次数(Call)、指令读取次数(Ir)、内存读取(Mr)、写入(Dw)等详细信息，并可产生完整的调用图谱（call graph）与详尽的函数热点分布。

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-vtune-简介与使用-vtune-简介与使用.md]

## strace（系统调用跟踪）

```bash
strace -c ./hotspot_lock_test 4 1000000
```

strace 可以跟踪程序执行过程中的系统调用，统计各系统调用的耗时和调用次数。

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-vtune-简介与使用-vtune-简介与使用.md]