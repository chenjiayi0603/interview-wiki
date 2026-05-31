# perf 核心命令详解

> 本文详细介绍了 perf 性能分析工具的核心命令，包括 perf record、perf report、perf stat、perf top 等命令的用途、参数、示例和输出说明。

[src: raw/ingested/2技术/性能优化/内存优化-perf-2.-核心命令详解.md]

## 2.1 perf record - 采集性能数据

**用途**：采集程序的硬件性能事件和采样数据，为后续分析提供详细的信息，支持抓取调用栈。

**典型场景**：识别程序运行期间的热点代码和性能瓶颈。

**核心参数**：
- `-g`：记录调用栈信息，用于生成火焰图
- `-F <频率>`：设置采样频率（如 `-F 99` 表示99Hz）
- `-e <事件>`：指定要采样的事件（如 `cache-misses`）
- `-p <pid>`：采样指定进程
- `-a`：采样所有CPU（系统级）

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

# 全系统采样
perf record -a -g -- sleep 30
```

**输出说明**：
- 执行后会生成 `perf.data` 文件，保存详细的采样信息
- 输出示例：
```
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.975 MB perf.data (xxxx samples) ]
```

[src: raw/ingested/2技术/性能优化/内存优化-perf-2.-核心命令详解.md]

---

## 2.2 perf report - 查看热点报告

**用途**：分析`perf record`采集到的数据（perf.data），以交互式界面显示热点函数、占用比例、调用栈等。

**典型场景**：开发者定位性能开销最大的函数，分析性能瓶颈位置。

**核心参数**：
- `-g`：显示调用栈
- `-i <文件>`：指定输入文件（默认perf.data）
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

**输出示例**：
```
Samples: 2K of event 'cycles', Event count (approx.): 1000000000
-  50.00%  your_program  your_program      [.] foo
   30.00%  your_program  your_program      [.] bar
   20.00%  your_program  your_program      [.] main
```

**特点**：
- 交互式界面，可展开查看调用关系
- 支持按照函数、调用栈、源码、汇编行查看热点
- 可配合 `perf annotate` 查看汇编级热点

[src: raw/ingested/2技术/性能优化/内存优化-perf-2.-核心命令详解.md]

---

## 2.3 perf stat - 统计性能指标

**用途**：统计程序运行的整体性能指标，包括 cycles、instructions、cache-misses 等，并计算 IPC（每周期指令数）。

**典型场景**：快速评估程序整体性能，识别CPU利用率、缓存效率、分支预测等问题。

**核心参数**：
- `-e <事件>`：指定要统计的事件（如 `cache-misses`）
- `-p <pid>`：统计指定进程
- `-I <间隔>`：每N毫秒输出一次统计信息
- `-a`：统计所有CPU

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

**输出示例**（对应指令）：
```bash
$ perf stat ./your_program
```
```
 Performance counter stats for './your_program':

    1,002.13 msec task-clock          # 0.998 CPUs utilized
    3,143,230      cycles             # 3.137 GHz
    1,234,567      instructions       # 0.39  insn per cycle
       21,431      cache-references
        5,432      cache-misses       # 25.35% of all cache refs
          134      page-faults
          567      branch-misses      # 2.15% of all branches

    1.004781086 seconds time elapsed
```

**关键指标说明**：
- **task-clock**：程序消耗的CPU时钟时间，反映CPU利用率
- **cycles**：CPU周期数，反映CPU时间消耗
- **instructions**：执行的CPU指令总数
- **insn per cycle (IPC)**：每周期执行的指令数
  - **含义**：衡量CPU流水线效率，>1.0表示效率高，<1.0表示存在流水线停顿
  - **示例**：0.39 IPC 表示平均每个周期执行0.39条指令，效率较低，可能存在内存等待或分支预测失败
- **cache-references**：缓存访问次数
- **cache-misses**：缓存未命中次数及比例
- **branch-misses**：分支预测失败次数及比例

[src: raw/ingested/2技术/性能优化/内存优化-perf-2.-核心命令详解.md]

---

## 2.4 perf top - 实时监控热点

**用途**：实时显示系统或进程当前的CPU消耗热点函数，类似`top`命令但聚焦于函数级别。

**典型场景**：排查运行中CPU资源消耗异常，快速锁定扰动点。

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

**输出示例**：
```
Samples: 10K of event 'cycles', Event count (approx.): 1000000000
Overhead  Command       Shared Object     Symbol
  34.56%  your_program  your_program      [.] foo
  22.34%  your_program  your_program      [.] bar
  12.45%  your_program  libc.so.6         [.] malloc
```

**特点**：
- 实时自刷新，聚焦于当前CPU占用热点
- 无需前期采集，立刻展示现状
- 适合线上快速定位CPU消耗点

[src: raw/ingested/2技术/性能优化/内存优化-perf-2.-核心命令详解.md]

---

## 2.5 三者对比速览

| 命令             | 用途             | 场景/特点                     |
|------------------|------------------|-------------------------------|
| `perf record`    | 采集采样数据     | 离线分析、可抓调用栈和详细事件 |
| `perf report`    | 分析查看报告     | 交互式热点分析，挖掘瓶颈      |
| `perf top`       | 实时监控热点     | 在线实时刷新，快速定位        |

[src: raw/ingested/2技术/性能优化/内存优化-perf-2.-核心命令详解.md]

---

## 2.6 其他常用命令

**perf script**：将perf.data转换为文本格式，用于生成火焰图
```bash
perf script > out.perf
```

**perf annotate**：查看热点函数的汇编代码和源码映射
```bash
perf annotate
```

**perf list**：列出所有可用的性能事件
```bash
perf list
```

**perf diff**：比较两次perf report的结果
```bash
perf diff perf.data.old perf.data.new
```

[src: raw/ingested/2技术/性能优化/内存优化-perf-2.-核心命令详解.md]