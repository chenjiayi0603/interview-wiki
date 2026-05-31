# top/htop（实时监控）

## 基本使用

**top 命令**：
```bash
# 查看进程内存
top -p <pid>

# 按内存排序
top -o %MEM

# 查看线程
top -H -p <pid>
```

**htop 命令**（更友好）：
```bash
# 安装 htop
sudo apt-get install htop

# 使用 htop
htop -p <pid>
```

**关键字段**：
- **VIRT**：虚拟内存大小（VSZ）
- **RES**：物理内存使用（RSS）
- **SHR**：共享内存
- **%MEM**：内存使用百分比

**完整使用示例**：
```bash
# 启动上面的 heap_demo 后，在另一终端：
top -p $PID
# 或按内存排序找占用最高的进程
top -o %MEM

# 若已安装 htop，可看单进程
htop -p $PID
# 在 htop 中按 F5 可看树状图，便于看线程
```

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-3.-系统工具详解.md]