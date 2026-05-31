# gprof（GNU性能分析器）

**用途**：函数级性能分析，统计程序中每个函数的调用次数与耗时。

**使用方法**：
```bash
# 1. 编译时加 -pg
g++ -pg -O2 -o program program.cpp

# 2. 执行程序，生成gmon.out文件
./program

# 3. 查看性能分析报告
gprof program gmon.out > report.txt
```

**输出示例**：
```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 30.00      0.03     0.03     1000     0.03     0.04  foo
 25.00      0.05     0.02     1000     0.02     0.03  bar
...
```

**特点**：
- 主要用于分析各函数耗时分布和调用关系
- 帮助定位热点和性能瓶颈
- 相比perf，功能较简单，但使用方便

[src: raw/ingested/2技术/性能优化/内存优化-perf-附录：其他性能分析工具.md]