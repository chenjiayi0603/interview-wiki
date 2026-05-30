# C++ 性能优化大厂考点

## 8. 大厂常见考点

### 8.1 性能分析类问题

**Q1: 如何定位性能瓶颈？**
```
1. 使用 profiler（perf、gprof）找出热点函数
2. 分析 CPU 使用率，找出 CPU 密集型函数
3. 检查内存分配，找出频繁分配/释放的地方
4. 分析缓存命中率，找出缓存不友好的代码
5. 查看 I/O 等待时间，找出 I/O 瓶颈
```

**Q2: 如何分析程序性能？**
```
1. 建立基准测试，记录关键指标
2. 使用性能分析工具（perf、valgrind）
3. 生成火焰图，可视化热点
4. 分析算法复杂度
5. 检查内存访问模式
6. 测量缓存命中率
```

**Q3: 如何优化慢查询？**
```
1. 分析查询路径，找出耗时操作
2. 优化算法复杂度（O(n²) -> O(n log n)）
3. 使用更高效的数据结构
4. 减少不必要的计算
5. 缓存计算结果
6. 并行化处理
```

### 8.2 内存优化类问题

**Q4: 如何减少内存分配？**
```
1. 使用对象池/内存池
2. 预分配容器容量（reserve）
3. 复用内存（clear 而不是重新分配）
4. 使用栈分配代替堆分配（小对象）
5. 使用移动语义减少拷贝
```

**Q5: 如何优化内存访问？**
```
1. 提高缓存局部性（顺序访问）
2. 减少指针间接访问
3. 优化数据结构布局（减少 padding）
4. 使用缓存行对齐避免 false sharing
5. 预取数据到缓存
```

**Q6: 如何避免内存泄漏？**
```
1. 使用 RAII（智能指针）
2. 使用 AddressSanitizer 检测
3. 确保 new/delete 配对
4. 检查循环引用（shared_ptr）
5. 使用 valgrind 定期检查
```

### 8.3 CPU优化类问题

**Q7: 如何优化分支预测？**
```
1. 减少分支数量（使用查找表）
2. 使用 likely/unlikely 提示
3. 将常见分支放在前面
4. 使用位运算代替简单分支
5. 帮助编译器生成条件移动指令
```

**Q8: 如何使用SIMD优化？**
```
1. 帮助编译器自动向量化（循环优化）
2. 使用 #pragma omp simd
3. 手动使用 SIMD 指令（AVX/AVX2）
4. 数据对齐到 16/32 字节
5. 处理连续数据
```

**Q9: 如何提高缓存命中率？**
```
1. 顺序访问数据（行优先遍历）
2. 提高数据局部性
3. 减少随机访问
4. 使用紧凑的数据结构
5. 预取数据
```

### 8.4 多线程优化类问题

**Q10: 如何减少锁竞争？**
```
1. 使用细粒度锁
2. 使用读写锁（读多写少）
3. 使用无锁数据结构（原子操作）
4. 减少锁持有时间
5. 使用线程局部存储
```

**Q11: 如何避免False Sharing？**
```
1. 将频繁访问的数据对齐到缓存行
2. 使用 padding 填充
3. 将不同线程的数据分开存储
4. 使用 thread_local
```

**Q12: 如何优化多线程性能？**
```
1. 合理设置线程数（CPU核心数）
2. 负载均衡（工作窃取）
3. 减少同步开销
4. 使用无锁编程
5. 避免过度并行化（小任务）
```

### 8.5 编译器优化类问题

**Q13: 编译器优化级别如何选择？**
```
-O0: 调试阶段，无优化
-O1: 基本优化，开发阶段
-O2: 完整优化，生产环境推荐
-O3: 激进优化，性能关键路径
-Os: 代码大小优化，嵌入式系统
```

**Q14: 链接时优化（LTO）的作用？**
```
1. 跨文件内联
2. 跨文件死代码消除
3. 全局优化
4. 减少代码大小
注意：会增加编译时间和内存占用
```

**Q15: 如何帮助编译器优化？**
```
1. 使用 const/constexpr
2. 使用 inline 提示
3. 使用 __restrict__ 指针
4. 避免指针别名
5. 使用 #pragma 指令
```

### 8.6 实际优化案例

**案例1: 字符串拼接优化**
```cpp
// ❌ 低效：多次分配
std::string result;
for (const auto& str : strings) {
    result += str;  // 可能触发多次重分配
}

// ✅ 高效：预分配
std::string result;
size_t total_size = 0;
for (const auto& str : strings) {
    total_size += str.size();
}
result.reserve(total_size);
for (const auto& str : strings) {
    result += str;  // 无重分配
}
```

**案例2: 查找优化**
```cpp
// ❌ O(n) 线性查找
bool contains(const std::vector<int>& vec, int value) {
    return std::find(vec.begin(), vec.end(), value) != vec.end();
}

// ✅ O(log n) 二分查找（如果有序）
bool contains(const std::vector<int>& vec, int value) {
    return std::binary_search(vec.begin(), vec.end(), value);
}

// ✅ O(1) 哈希查找
bool contains(const std::unordered_set<int>& set, int value) {
    return set.count(value) > 0;
}
```

**案例3: 循环优化**
```cpp
// ❌ 低效：每次调用 size()
for (int i = 0; i < vec.size(); ++i) {
    process(vec[i]);
}

// ✅ 高效：缓存 size()
for (size_t i = 0, n = vec.size(); i < n; ++i) {
    process(vec[i]);
}

// ✅ 更高效：使用迭代器
for (const auto& item : vec) {
    process(item);
}
```

### 8.7 性能优化原则

1. **测量优先**：先测量，再优化，验证优化效果
2. **80/20原则**：优化20%的热点代码，获得80%的性能提升
3. **避免过早优化**：先保证正确性，再优化性能
4. **权衡取舍**：性能 vs 可读性 vs 维护性
5. **持续监控**：建立性能监控，及时发现性能回退

[src: raw/ingested/2技术/性能优化/瓶颈-C++性能优化大厂考点-8.-大厂常见考点.md]