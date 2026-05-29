# /proc/pid/statm（内存页统计）

## 基本使用

```bash
# 查看进程内存页统计（页大小通常 4KB）
cat /proc/<pid>/statm
```

**输出说明**（一行 7 个数字，单位均为页）：
```
12345 10000 5000 100 0 2000 0
```
- 第 1 列：总虚拟大小（size）
- 第 2 列：常驻大小（resident）
- 第 3 列：共享页数（shared）
- 第 4 列：文本/代码（trs）
- 第 5 列：库（lrs）
- 第 6 列：数据/栈（drs）
- 第 7 列：脏页（dt）

**完整示例**：
```bash
# 结合 heap_demo 的 PID
cat /proc/$PID/statm
# 换算为 KB：第二列 * 4 约等于 RSS(KB)
echo "RSS(KB) ≈ $(($(cat /proc/$PID/statm | cut -d' ' -f2) * 4))"
```

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-3.-系统工具详解.md]