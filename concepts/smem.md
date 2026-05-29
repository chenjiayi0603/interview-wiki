# smem（内存使用统计）

## 基本使用

```bash
# 安装（Linux）
sudo apt-get install smem

# 按 RSS 排序查看进程内存
smem -p -s rss

# 按用户统计
smem -u

# 只看指定进程（结合上面 heap_demo 的 PID）
smem -p -s rss | grep -E "PID|$PID"
```

**输出示例（片段）**：
```
  PID User     Command                         Swap      USS      PSS      RSS
12345 user     ./heap_demo                       0     2048     2100     2200
```
- USS：进程独占内存；PSS：按共享比例分摊；RSS：常驻集大小。

[src: raw/ingested/2技术/性能优化/内存优化-C++内存分析工具分析-3.-系统工具详解.md]