# perf - Linux 性能分析工具

> perf 是 Linux 内核提供的性能分析工具，基于 `perf_event` 子系统，可以直接访问硬件性能计数器。

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-perf---Linux-性能分析工具-perf---Linux-性能分析工具.md]

## 核心功能

- **热点函数定位**：找出 CPU 消耗最多的函数
- **调用栈分析**：分析函数调用关系和上下文
- **硬件事件统计**：统计 cycles、instructions、cache-misses 等
- **实时监控**：实时查看系统性能热点
- **火焰图生成**：可视化性能瓶颈

## 安装方法

```bash
# Ubuntu/Debian
sudo apt-get install linux-perf

# CentOS/RHEL
sudo yum install perf

# 验证安装
perf --version
```

## 核心命令

### perf record - 采集性能数据

**用途**：采集程序的硬件性能事件和采样数据，为后续分析提供详细信息。

**核心参数**：
- `-g`：记录调用栈信息，用于生成火焰图
- `-F <频率>`：设置采样频率（如 `-F 99` 表示 99Hz）
- `-e <事件>`：指定要采样的事件（如 `cache-misses`）
- `-p <pid>`：采样指定进程
- `-a`：采样所有 CPU（系统级）

**命令示例**：
```bash
# 基础用法：采集程序运行性能数据
perf record -g ./your_program

# 指定采样频率
perf record -F 99 -g ./your_program

# 采样指定进程
perf record -p <pid> -g -- sleep 30

# 采样特定硬件事件
perf record -e cache-misses -g ./your_program
```

### perf report - 查看热点报告

**用途**：分析 `perf record` 采集到的数据，以交互式界面显示热点函数、占用比例、调用栈等。

**核心参数**：
- `-g`：显示调用栈
- `-i <文件>`：指定输入文件（默认 perf.data）
- `--stdio`：以文本方式输出（非交互式）

**命令示例**：
```bash
# 交互式查看报告（默认）
perf report

# 文本方式输出
perf report --stdio

# 指定输入文件
perf report -i perf.data.old
```

### perf stat - 统计性能指标

**用途**：统计程序运行的整体性能指标，包括 cycles、instructions、cache-misses 等，并计算 IPC。

**核心参数**：
- `-e <事件>`：指定要统计的事件（如 `cache-misses`）
- `-p <pid>`：统计指定进程
- `-I <间隔>`：每 N 毫秒输出一次统计信息
- `-a`：统计所有 CPU

**命令示例**：
```bash
# 基础用法：统计整体性能指标
perf stat ./your_program

# 统计特定事件
perf stat -e cache-references,cache-misses ./your_program

# 统计指定进程
perf stat -p <pid>

# 定时输出统计信息
perf stat -I 1000 ./your_program  # 每1秒输出一次
```

### perf top - 实时监控热点

**用途**：实时显示系统或进程当前的 CPU 消耗热点函数。

**核心参数**：
- `-p <pid>`：监控指定进程
- `-H`：按线程显示热点
- `-e <事件>`：指定监控的事件

**命令示例**：
```bash
# 实时查看全局热点
perf top

# 实时关注指定进程热点
perf top -p <pid>

# 实时按线程显示热点
perf top -p <pid> -H
```

## 性能指标详解

### CPU 周期数（cycles）

**说明**：表示 CPU 总共消耗的时钟周期数，越少代表越高效。

**采集方式**：
```bash
perf stat -e cycles ./program
```

**优化建议**：减少无用指令，提高并发度，优化算法复杂度。

### 指令总数（instructions）

**说明**：程序总共执行的机器指令条数。

**采集方式**：
```bash
perf stat -e instructions ./program
```

**优化建议**：采用高效数据结构、算法减少不必要的操作。

### IPC（每周期指令数）

**说明**：每个 CPU 周期平均完成的指令数，是效率的重要指标。

**正常范围**：>1.0 表示效率高，<1.0 表示存在流水线停顿。

**优化建议**：减少分支、优化数据访问模式，提升指令并行性。

### 缓存未命中率（cache-misses）

**说明**：CPU 访问缓存时未命中的次数/比例，反映内存访问友好性。

**采集方式**：
```bash
perf stat -e cache-references,cache-misses ./program
```

**正常范围**：<10% 为佳，>25% 需要优化。

**优化建议**：调整数据布局、顺序访存、减少随机访问。

### 分支预测失败率（branch-misses）

**说明**：分支预测失败的次数/比例，失败比例高则性能受损。

**采集方式**：
```bash
perf stat -e branches,branch-misses ./program
```

**正常范围**：<5% 为佳，>10% 需要优化。

**优化建议**：减少 if-else 分支、用查找表、增加分支可预测性。

## 火焰图生成

### 什么是火焰图？

**定义**：一种性能分析可视化工具，用堆叠柱状条展示函数调用关系和各自消耗的 CPU 时间/采样次数。

**特点**：
- **横轴宽度** = 时间或采样比例（宽->更耗时）
- **纵轴高度** = 调用深度
- **颜色**：无语义，仅便于区分
- 支持多语言：C/C++、Java、Go 等

### 生成流程

1. **安装 Flamegraph 工具**
   ```bash
   git clone https://github.com/brendangregg/Flamegraph
   cd Flamegraph
   ```

2. **采集调用栈采样**
   ```bash
   perf record -F 99 -a -g -- sleep 30
   perf script > out.perf
   ```

3. **折叠原始样本**
   ```bash
   ./stackcollapse-perf.pl out.perf > out.folded
   ```

4. **生成 SVG 火焰图**
   ```bash
   ./flamegraph.pl out.folded > flame.svg
   ```

## 其他常用命令

- **perf script**：将 perf.data 转换为文本格式，用于生成火焰图
- **perf annotate**：查看热点函数的汇编代码和源码映射
- **perf list**：列出所有可用的性能事件
- **perf diff**：比较两次 perf report 的结果

## 可分析的内容范围

除了CPU性能，perf等工具还可以分析以下内容：

- **内存（Memory）**：分析内存分配、访问模式、缓存命中与未命中情况
- **I/O 性能（Disk/Network I/O）**：分析磁盘/网络等I/O操作的性能
- **上下文切换与调度**：监控系统的软/硬中断、上下文切换和调度延迟
- **锁竞争（Lock Contention）**：定位多线程程序中的锁竞争和同步热点
- **分支预测与指令流水**：分析分支预测失败、流水线停顿等底层CPU微结构事件
- **系统调用（Syscalls）**：分析程序中频繁触发的系统调用
- **能耗/功耗（Power Consumption）**：分析程序消耗的能量和功耗分布

## 相关页面

- [[C++并发性能优化]]
- [[C++多线程与并发]]
- [[内存管理]]