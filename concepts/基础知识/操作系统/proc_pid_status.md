# /proc/pid/status（进程状态）

## 基本使用

**查看进程内存状态**：
```bash
# 查看进程状态
cat /proc/<pid>/status

# 只看内存相关
cat /proc/<pid>/status | grep -i mem
```

**输出示例**：
```
VmPeak:     123456 kB    # 峰值虚拟内存
VmSize:     100000 kB    # 当前虚拟内存
VmLck:           0 kB    # 锁定的内存
VmPin:           0 kB    # 固定的内存
VmHWM:      50000 kB    # 峰值物理内存（RSS）
VmRSS:      45000 kB    # 当前物理内存（RSS）
VmData:     30000 kB    # 数据段大小
VmStk:        132 kB    # 栈大小
VmExe:          4 kB    # 代码段大小
VmLib:       5000 kB    # 共享库大小
VmPTE:        100 kB    # 页表大小
VmSwap:         0 kB    # 交换分区使用
```

**关键指标**：
- **VmRSS**：实际物理内存使用（最重要）
- **VmSize**：虚拟内存大小
- **VmHWM**：峰值物理内存
- **VmData**：堆内存大小

**完整示例**：
```bash
# 查看某进程内存相关状态
cat /proc/$PID/status | grep -E '^Vm|^Name|^Pid'

# 输出示例：
# Name:   heap_demo
# Pid:    12345
# VmPeak:   XXX kB
# VmSize:   XXX kB
# VmRSS:    XXX kB
# VmData:   XXX kB
```

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-3.-系统工具详解.md]