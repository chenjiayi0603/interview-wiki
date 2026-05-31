# /proc/pid/maps（进程地址空间）

## 基本使用

**/proc/pid/maps 用途**：
- 查看进程的详细地址空间布局
- 分析内存映射详情
- 定位内存泄漏区域

**查看命令**：
```bash
# 查看进程地址空间
cat /proc/<pid>/maps

# 实时监控
watch -n 1 'cat /proc/<pid>/maps'

# 查找特定地址
cat /proc/<pid>/maps | grep <address>
```

**输出示例**：
```
00400000-00401000 r-xp 00000000 08:01 123456 /path/to/program
00600000-00601000 rw-p 00000000 00:00 0      [heap]
7f1234567000-7f1234568000 rw-p 00000000 00:00 0      [heap]
7f1234568000-7f1234569000 r-xp 00000000 08:01 789012 /lib/x86_64-linux-gnu/libc-2.23.so
...
```

**字段说明**：
- **地址范围**：起始地址-结束地址
- **权限**：r=读，w=写，x=执行，p=私有，s=共享
- **偏移量**：文件中的偏移
- **设备**：主设备号:次设备号
- **inode**：文件 inode 号
- **路径名**：映射的文件路径

## 分析技巧

**查找内存泄漏**：
```bash
# 查找匿名映射（可能是泄漏）
cat /proc/<pid>/maps | grep -v "\.so\|\.exe" | grep "rw-p"

# 查找堆内存增长
watch -n 1 'cat /proc/<pid>/maps | grep heap | wc -l'
```

**完整示例**（沿用上面 `heap_demo` 的 PID）：
```bash
# 查看完整地址空间
cat /proc/$PID/maps

# 只看堆与匿名可写区
cat /proc/$PID/maps | grep -E 'heap|anon'
```

**/proc/pid/maps 输出示例（片段）**：
```
00400000-00401000 r-xp 00000000 08:01 123456  /path/to/heap_demo
00600000-00601000 rw-p 00000000 00:00 0       [heap]
01a2a000-01c2b000 rw-p 00000000 00:00 0       [heap]   # 约 2MB
7f1234567000-7f1234588000 r-xp 00000000 08:01 789012  /lib/x86_64-linux-gnu/libc-2.31.so
```

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-3.-系统工具详解.md]