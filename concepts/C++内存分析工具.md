# C++ 内存分析工具

See also: [[内存泄漏]], [[内存管理]], [[C++调试]]

## 0. 核心要点快速总结

### 0.1 内存分析工具速查表

| 工具类别 | 工具名称 | 主要用途 | 核心命令 | 适用场景 |
|---------|---------|---------|---------|---------|
| **Valgrind** | Memcheck | 内存错误检测（泄漏、越界、UAF） | `valgrind --leak-check=full ./program` | 深度调试，全面检测 |
| **Valgrind** | Massif | 堆内存使用分析 | `valgrind --tool=massif ./program` | 内存占用热点分析 |
| **Sanitizer** | AddressSanitizer | 内存越界/非法访问检测 | `g++ -fsanitize=address -g -O1` | 快速检测，日常开发 |
| **Sanitizer** | LeakSanitizer | 内存泄漏检测 | `g++ -fsanitize=leak -g` | 快速泄漏检测 |
| **Sanitizer** | MemorySanitizer | 未初始化内存检测 | `g++ -fsanitize=memory -g` | 未初始化内存问题 |
| **系统工具** | pmap | 进程内存映射查看 | `pmap -x <pid>` | 实时内存映射分析 |
| **系统工具** | /proc/pid/maps | 进程地址空间查看 | `cat /proc/<pid>/maps` | 详细内存布局分析 |
| **系统工具** | top/htop | 进程内存监控 | `top -p <pid>` | 实时内存使用监控 |

**记忆口诀**：
「Valgrind 全面但慢，Sanitizer 快速集成，系统工具实时监控，三者结合定位问题。」

**一行简答**：
「C++ 内存分析工具主要包括 Valgrind（全面检测但慢）、Sanitizer 系列（快速集成）、系统工具（实时监控），掌握三者用法可高效定位内存问题。」

---

### 0.2 内存问题类型速查

| 问题类型 | 检测工具 | 典型场景 | 严重程度 |
|---------|---------|---------|---------|
| **内存泄漏** | Memcheck, LSan | `new` 后忘记 `delete` | 高 |
| **使用已释放内存（UAF）** | Memcheck, ASan | `delete` 后继续使用指针 | 严重 |
| **数组越界** | Memcheck, ASan | 访问 `arr[10]` 但数组大小为 10 | 严重 |
| **未初始化内存** | Memcheck, MSan | 使用未初始化的局部变量 | 高 |
| **重复释放** | Memcheck, ASan | 对同一指针多次 `delete` | 严重 |
| **内存对齐问题** | Memcheck | 未对齐的内存访问 | 中 |

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-0.-核心要点快速总结.md]