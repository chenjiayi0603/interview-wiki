# perf性能分析工具大厂考点复习文档

## 0. 核心要点速览 ✅

### 0.1 大厂面试必考核心点

**perf工具定位**：
- perf 是 Linux 下最常用的性能分析工具，支持统计函数调用、采样热点、硬件事件、内存与缓存命中等
- 常见采样命令：`perf record -g ./your_program`，可分析调用栈和热点分布

**记忆口诀**：  
「热点定位 perf record，详细指标 perf stat，函数占比 perf report，实时监控 perf top，硬件瓶颈一网打。」

**一行简答**：「perf可采集程序指令/周期/缓存/热点函数等性能指标，是性能分析的标准工具，掌握 record/stat/report/top 用法。」

---

### 0.2 perf核心命令速查表

| 命令 | 用途 | 核心参数 | 典型场景 |
|------|------|---------|---------|
| **perf record** | 采集性能数据 | `-g`(调用栈), `-F`(频率), `-e`(事件), `-p`(进程) | 离线分析、定位瓶颈 |
| **perf report** | 查看热点报告 | `-g`, `-i`, `--stdio` | 热点函数分析 |
| **perf stat** | 统计性能指标 | `-e`(事件), `-p`(进程), `-I`(间隔) | 性能基准/瓶颈分析 |
| **perf top** | 实时监控热点 | `-p`(进程), `-H`(线程) | 线上排查 |
| **perf script** | 转文本调用栈 | `-i`(输入文件) | 火焰图生成 |
| **perf annotate** | 汇编级分析 | `-i`, `--stdio` | 指令/流水线优化 |

---

### 0.3 核心性能指标速查

| 指标 | 含义 | 正常范围 | 优化方向 |
|------|------|---------|---------|
| **cycles** | CPU周期数 | - | 减少指令数、提高IPC |
| **instructions** | 指令总数 | - | 优化算法、减少冗余计算 |
| **IPC** | 每周期指令数 | >1.0 | 减少内存等待、优化分支 |
| **cache-misses** | 缓存未命中率 | <10% | 优化数据布局、顺序访问 |
| **branch-misses** | 分支预测失败率 | <5% | 减少分支、使用查找表 |

---

### 0.4 大厂面试高频考点

**Q1: 如何使用perf定位CPU热点函数？**
- 使用 `perf record -g ./program` 采集性能数据
- 使用 `perf report` 查看热点函数分布
- 使用 `perf top` 实时监控CPU热点

**Q2: perf stat能统计哪些关键性能指标？**
- cycles（CPU周期数）
- instructions（指令总数）
- IPC（每周期指令数，>1.0表示效率高）
- cache-misses（缓存未命中率，<10%为佳）
- branch-misses（分支预测失败率，<5%为佳）

**Q3: 如何识别缓存未命中和分支预测失败？**
- 使用 `perf stat -e cache-references,cache-misses ./program` 统计缓存事件
- 使用 `perf stat -e branches,branch-misses ./program` 统计分支事件
- 使用 `perf record -e cache-misses -g ./program` 采样缓存未命中

**Q4: 如何生成火焰图？**
- `perf record -F 99 -g ./program` 采集调用栈
- `perf script > out.perf` 导出数据
- `stackcollapse-perf.pl out.perf > out.folded` 折叠调用栈
- `flamegraph.pl out.folded > flame.svg` 生成SVG火焰图

**Q5: perf record、perf report、perf top的区别？**
- **perf record**：采集性能数据，生成perf.data文件，支持离线分析
- **perf report**：分析perf.data，以交互式界面显示热点函数和调用栈
- **perf top**：实时显示系统或进程当前的CPU消耗热点，无需前期采集

[src: raw/ingested/2技术/性能优化/内存优化-perf-0.-核心要点速览-✅.md]