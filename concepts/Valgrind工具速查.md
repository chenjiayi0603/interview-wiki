# Valgrind 工具速查

## 0. 核心要点快速总结

### 0.1 Valgrind 工具速查表

| 工具 | 主要用途 | 核心命令 | 适用场景 | 性能影响 |
|------|---------|---------|---------|---------|
| **Memcheck** | 内存错误检测（泄漏、越界、UAF） | `valgrind --leak-check=full ./program` | 深度调试，全面检测 | 慢（10-50倍） |
| **Massif** | 堆内存使用分析 | `valgrind --tool=massif ./program` | 内存占用热点分析 | 慢（10-50倍） |
| **Callgrind** | 函数调用分析 | `valgrind --tool=callgrind ./program` | 性能分析，调用关系 | 慢（10-50倍） |
| **Helgrind** | 线程错误检测 | `valgrind --tool=helgrind ./program` | 多线程调试，数据竞争 | 慢（10-50倍） |
| **Cachegrind** | 缓存性能分析 | `valgrind --tool=cachegrind ./program` | 缓存命中率分析 | 慢（10-50倍） |

**记忆口诀**：
「Memcheck 查内存，Massif 看堆用，Callgrind 看调用，Helgrind 查线程，Cachegrind 看缓存。」

**一行简答**：
「Valgrind 是 Linux 下的内存和性能分析工具套件，主要包括 Memcheck（内存错误检测）、Massif（堆内存分析）、Callgrind（调用分析）、Helgrind（线程检测）等工具，虽然运行慢但检测全面准确。」

---

### 0.2 内存问题类型速查

| 问题类型 | Memcheck 检测 | 典型场景 | 严重程度 | 输出关键词 |
|---------|-------------|---------|---------|-----------|
| **内存泄漏** | ✅ | `new` 后忘记 `delete` | 高 | `definitely lost`, `indirectly lost` |
| **使用已释放内存（UAF）** | ✅ | `delete` 后继续使用指针 | 严重 | `Invalid read/write`, `free'd` |
| **数组越界** | ✅ | 访问 `arr[10]` 但数组大小为 10 | 严重 | `Invalid read/write`, `after a block` |
| **未初始化内存** | ✅ | 使用未初始化的局部变量 | 高 | `Use of uninitialised value` |
| **重复释放** | ✅ | 对同一指针多次 `delete` | 严重 | `Invalid free()`, `free'd` |
| **内存对齐问题** | ✅ | 未对齐的内存访问 | 中 | `Invalid read/write` |

---

### 0.3 泄漏类型速查

| 泄漏类型 | 含义 | 严重程度 | 处理优先级 |
|---------|------|---------|-----------|
| **definitely lost** | 确定的内存泄漏，程序结束时没有任何指针指向该内存 | 高 | 必须修复 |
| **indirectly lost** | 间接泄漏，通常由其他泄漏导致（如容器中的对象泄漏） | 高 | 必须修复 |
| **possibly lost** | 可能泄漏，指针可能指向内存中间位置（如指针算术后丢失起始地址） | 中 | 需要检查 |
| **still reachable** | 仍然可达，程序结束时仍有指针指向，但可能不是真正的泄漏（如全局变量） | 低 | 视情况而定 |

[src: raw/ingested/2技术/性能优化/内存优化-valgrind-0.-核心要点快速总结.md]