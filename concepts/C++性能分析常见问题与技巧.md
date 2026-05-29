# C++ 性能分析常见问题与技巧

> 本文总结 C++ 性能分析中的常见问题与实用技巧，涵盖 perf、strace、time 等工具的使用与优化方法。

See also: [[C++多线程与并发]], [[C++并发性能优化]], [[TBB最佳实践]], [[TBB性能优化技巧]]

## 6.1 常见问题

### Q1: perf 报告显示符号为 `[unknown]`，如何解决？

**原因**：缺少调试信息或符号表。

**解决方法**：
```bash
# 1. 编译时添加调试信息
g++ -g -O2 -o program program.cpp

# 2. 安装调试符号
# Ubuntu/Debian
sudo apt-get install linux-image-$(uname -r)-dbgsym

# 3. 使用 perf report 时指定符号路径
perf report --symfs=/usr/lib/debug
```

---

### Q2: strace 跟踪导致程序变慢，如何减少开销？

**解决方法**：
```bash
# 1. 只统计不显示详细信息（开销较小）
strace -c ./program

# 2. 只跟踪特定系统调用（减少跟踪量）
strace -e trace=open,read,write ./program

# 3. 限制跟踪时间
timeout 10 strace ./program
```

---

### Q3: 如何分析多线程程序的性能？

**解决方法**：
```bash
# 1. perf 支持多线程分析
perf record -g -p <pid>  # 自动跟踪所有线程

# 2. 按线程显示热点
perf top -p <pid> -H

# 3. 在 perf report 中按线程过滤
perf report --sort comm,pid,tid
```

---

### Q4: 如何比较优化前后的性能？

**解决方法**：
```bash
# 1. 使用 perf diff
perf record -g ./program_before
mv perf.data perf.data.before
perf record -g ./program_after
perf diff perf.data.before perf.data

# 2. 使用 perf stat 对比指标
perf stat ./program_before > before.txt
perf stat ./program_after > after.txt
diff before.txt after.txt
```

---

## 6.2 实用技巧

### 技巧 1：快速定位瓶颈的命令组合

```bash
# 一键分析：统计 + 热点定位
perf stat ./program && perf record -g ./program && perf report
```

// 例子：perf 性能分析一键组合命令演示

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <chrono>

// 简单热点：求数组和
long long compute_sum(const std::vector<int>& v) {
    long long s = 0;
    for (size_t i = 0; i < v.size(); ++i) {
        s += v[i];
    }
    return s;
}

int main(int argc, char** argv) {
    size_t N = 10000000;
    if (argc > 1) N = std::stoull(argv[1]);
    std::vector<int> v(N, 1);

    auto beg = std::chrono::high_resolution_clock::now();
    long long result = compute_sum(v);
    auto end = std::chrono::high_resolution_clock::now();

    std::chrono::duration<double> d = end - beg;
    std::cout << "sum=" << result << std::endl;
    std::cout << "elapsed=" << d.count() << " s" << std::endl;
    return 0;
}
```

// 编译：g++ -O2 -g perf_example.cpp -o program

// 一键分析命令（统计+热点定位+报告）
/*
perf stat ./program && perf record -g ./program && perf report
*/

// 实际过程说明：
// 1. perf stat ./program               # 输出整体性能统计（IPC，耗时，cache行为等）
// 2. perf record -g ./program          # 采集带调用关系的性能热点数据
// 3. perf report                       # 基于记录的数据生成热点分布报告
//
// perf stat ./program && perf record -g ./program && perf report 的输出举例如下：
//
// 1. perf stat ./program 输出整体性能统计
//
// $ perf stat ./program
// sum=10000000
// elapsed=0.00753184 s
//
//  Performance counter stats for './program':
//
//         7,614.57 msec task-clock                #    1.008 CPUs utilized
//                 0      context-switches        #    0.000 K/sec
//                 0      cpu-migrations          #    0.000 K/sec
//               113      page-faults             #    0.015 M/sec
//        25,542,372      cycles                  #    3.355 GHz
//        19,999,985      instructions            #    0.78  insn per cycle
//         3,836,222      branches                #  504.019 M/sec
//            11,028      branch-misses           #    0.29% of all branches
//
//      0.007559157 seconds time elapsed
//
// 上述信息可看到程序耗时、IPC（每周期指令数）、cache 行为等关键指标。
//
// 2. perf record -g ./program 采集带调用栈信息的性能采样数据
//
// $ perf record -g ./program
// sum=10000000
// elapsed=0.00748912 s
// [ perf record: Woken up 1 times to write data ]
// [ perf record: Captured and wrote 0.024 MB perf.data (120 samples) ]
//
// 3. perf report 展示热点分布和函数调用关系
//
// $ perf report
//
// # Samples: 120
// # Event: cycles
// #
// # Overhead  Command    Shared Object        Symbol
// # ........  .........  ..................  ................................
//     65.00%  program    program             [.] compute_sum
//     25.00%  program    libc.so.6           [.] __libc_start_main
//      8.00%  program    program             [.] main
//      2.00%  program    [kernel.kallsyms]  [k] __do_user_fault
//
// 还支持用回车进入调用图界面，查看某个热点函数的详细调用栈信息（以 compute_sum 为例）：
// └─compute_sum
//     └─main
//         └─__libc_start_main
//
// 分析：
//
// - perf stat 汇总了程序的运行时间、CPU 周期、指令数量、IPC、分支预测等信息。结合这些数据可以快速判断程序是 CPU/内存密集，是否存在分支预测失败等。
// - perf record 和 perf report 结合可以定位最消耗 CPU 时间的热点函数（如本例 compute_sum），明确最该优化的代码片段或算法结构。
// - 通过采样记录的调用关系（-g），可以进一步追踪热点背后的调用路径，实现按需优化。


/usr/bin/time -v ./program 
sum=10000000                    // 输出最终的和，验证程序正确性
elapsed=0.00272772 s            // 程序运行所需的总时间（秒），反映计算耗时
        Command being timed: "./program"   // 被 time 工具监控的实际命令
        User time (seconds): 0.00          // 为什么这里是0？因为程序规模很小/计算极快，实际运行时间不到0.005秒，四舍五入显示为0——并非真的0秒。大部分耗时都极短，只有在任务量较大或系统计时分辨率较高的场景下，这里才会显示为非零。
        System time (seconds): 0.01        // 系统态消耗的 CPU 时间（主要是系统调用、内核操作）
        // 本进程获得的 CPU 利用率百分比（越高越说明 CPU 占有充分）
        Percent of CPU this job got: 87%   // 这是整体 CPU 占有率，包括用户态和系统态（即 user+system time 总和），表示进程占用 CPU 的百分比，而不是区分仅用户态还是仅系统态。
        Elapsed (wall clock) time (h:mm:ss or m:ss): 0:00.01   // 实际挂钟耗时，含所有等待
        Average shared text size (kbytes): 0        // 平均共享文本段大小（极少关注）
        Average unshared data size (kbytes): 0      // 平均非共享数据段大小（通常为0）
        Average stack size (kbytes): 0              // 平均栈空间大小（一般很小）
        Average total size (kbytes): 0              // 平均进程总空间（通常为0，仅部分系统下有效）
        Maximum resident set size (kbytes): 42560   // 峰值驻留内存（RAM）使用量，反映最大内存消耗
        Average resident set size (kbytes): 0       // 平均驻留内存
        Major (requiring I/O) page faults: 1        // 需要 I/O 的主缺页异常数量（通常应很低）
        Minor (reclaiming a frame) page faults: 9901 // 可从缓存获取的缺页异常数量（较高说明内存访问频繁但无需磁盘I/O）
        Voluntary context switches: 1               // 主动让出CPU的上下文切换次数
        Involuntary context switches: 0             // 被抢占导致的上下文切换次数（系统调度）
        Swaps: 0                                   // 进程被换出到交换区的次数（理想为0）
        File system inputs: 96                      // 文件系统读取字节数（输入）
        File system outputs: 0                      // 文件系统写入字节数（输出）
        Socket messages sent: 0                     // 发送的套接字消息数（没有网络通信即为0）
        Socket messages received: 0                 // 接收的套接字消息数
        Signals delivered: 0                        // 收到的信号数
        Page size (bytes): 4096                     // 系统的内存页大小（一般为4096字节）
        Exit status: 0                              // 程序退出状态码，0代表正常退出
        
// 分析 time：
// /usr/bin/time -v 输出详细记录了程序的运行资源消耗情况，主要关注以下指标：
// 1. User time (seconds) 和 System time (seconds)：反映程序在用户态和内核态分别消耗的 CPU 时间。本例中绝大多数时间在 user time，说明为纯 CPU 运算型任务；system time 极低，几乎无系统调用和内核开销。
// 2. Elapsed (wall clock) time（实际耗时）和 Percent of CPU this job got（CPU 使用率）：实际耗时与 CPU 时间之比给出了并发利用率或资源调度效率。若百分比接近 100%，说明是在独占 CPU，否则可能有调度等待或 I/O。
// 3. Maximum resident set size：最大驻留集内存（RAM）占用。数值较小，说明本例并无高内存压力。
// 4. Context switches、Page faults：上下文切换和缺页数量。都很低，说明程序运行平稳、未受系统资源竞争或异常影响。
// 5. 其余如文件 I/O、socket、信号等指标几乎为零，符合典型的算法内存计算型程序，没有外部阻塞因素。
// 总结：time 工具能快速判定程序是 CPU 密集、内存密集、I/O 密集还是系统调用瓶颈。本例输出显示完全为 CPU 密集型，性能主要受限于计算本身，进一步优化应从算法和指令层面着手（如向量化、并行化、缓存优化等），而非关注系统或 I/O 层面。

分析上面的 /usr/bin/time -v 工具输出，可以快速判断程序的性能瓶颈类型：

- **CPU 密集型**：如果 User time（用户态 CPU 时间）远高于 System time（内核态 CPU 时间），且 Elapsed time 和 CPU 利用率接近 100%，则说明大部分时间都在做纯计算（CPU 密集）。如本例属于典型的 CPU 密集型任务。
- **内存密集型**：若 Maximum resident set size（最大驻留集内存）非常大，Minor/Major page faults 很多，说明大量数据驻留/交换于内存，发生大量缺页异常，可能为内存密集型。此例内存占用不高，并非瓶颈。
- **I/O 密集型**：若 File system inputs/outputs、Socket messages 等明显增长，且 CPU 利用率低，表明程序运行过程中被磁盘、网络等 I/O 设备拖慢。此处 I/O 指标接近零，不是 I/O 密集。
- **系统调用瓶颈**：System time 较高，频繁的上下文切换（context switches）、信号（signals）、swap 等，表示程序被内核层操作耗时拖慢。此例 system time 极低，没有系统级瓶颈。

**总结**：虽然 time -v 输出的 User time (seconds): 0.00，看起来用户态消耗为 0，但这主要是因为程序运行非常快、时间极短，以至于数值被四舍五入为 0。实际上，绝大部分时间花在用户态代码计算上，只有极少数资源用于 system time（如 0.01 秒），说明该程序以计算为主、没有大量系统调用（比如 I/O 操作、内存频繁分配释放等）。配合其他 time -v 输出（如 I/O、缺页、内存分配、上下文切换次数都极低），可以判断该程序本质上是**CPU 密集型**工作负载，主要花时间在实际计算而非系统层开销。因此，后续优化应以算法、指令执行效率和并行计算为重点。



### 技巧 2：生成完整的性能分析报告

```bash
# 生成包含所有信息的报告
{
    echo "=== 整体性能统计 ==="
    perf stat ./program
    echo ""
    echo "=== 热点函数分析 ==="
    perf record -g ./program
    perf report --stdio
    echo ""
    echo "=== 系统调用统计 ==="
    strace -c ./program
} > performance_report.txt
```

perf stat 输出示例：
perf stat 主要用于查看总体的性能统计情况，从输出结果可以看到程序运行期间使用的 CPU 时间、上下文切换次数、CPU 迁移次数、缺页异常数量、指令数、每周期指令数（insn per cycle）、分支预测相关数据等。比如 IPC（每周期指令数）为 0.50，说明指令流水线还有较大优化空间；branch-misses 比例越低越好，cache 行为和分支预测会影响整体性能。

perf stat 可以帮助我们判断指令流水线的利用率。若输出中的 IPC（每周期指令数，insn per cycle）较低（如 0.50），代表指令流水线未被充分利用，说明存在优化空间。优化指令流水线可以从以下几个角度入手：

1. **减少流水线停顿**：尽量避免分支预测失败（branch-misses），可以通过简化分支、使用条件移动指令（如 C++17 的 if constexpr）或减少分支数量。
2. **提升缓存命中率**：优化数据结构和访问模式，减少 cache miss，确保数据局部性（如数组遍历的顺序访问）。
3. **减少数据依赖**：避免连续依赖性强的语句，提升指令间并行度。
4. **消除内存瓶颈**：减少频繁内存分配和释放、尽量批量处理数据，强化数据预取（software prefetch）。
5. **利用编译器优化**：开启高级别优化（如 -O3）、分析汇编指令，结合 SIMD 或矢量化指令并行处理数据。

整体思路是让 CPU 能“流水线化”地高效执行指令，避免因分支、依赖、cache miss 等问题阻塞流水线，提高每周期指令数，让程序运行更快。


```
       1,023.45 msec task-clock                #    1.001 CPUs utilized          
             100 context-switches              #    0.098 K/sec                  
               2 CPU-migrations                #    0.002 K/sec                  
         151,234 page-faults                   #    0.148 M/sec                  
   3,784,567,890 cycles                        #    3.701 GHz                    
   1,902,345,678 instructions                  #    0.50  insn per cycle    # 平均每个时钟周期执行 0.5 条指令（insn/instruction），流水线利用率较低，可进一步优化
      234,567,890 branches                     #  229.240 M/sec                  
        12,345,678 branch-misses               #    5.26% of all branches 
  1.022 sec elapsed time
```

perf report --stdio 展示各函数（或代码段）在性能采样中占据的比例。比如 compute_sum 占 34.56%，是主要的热点消耗；malloc 频繁出现通常表示内存分配管理是瓶颈之一。热点函数一般是优化的优先目标。

可以通过 perf report --stdio 的输出结果看出 compute_sum 的占比。在 perf report --stdio 的结果列表中，会显示每个函数（如 compute_sum）在性能采样中所占百分比。例如：

perf report --stdio
  Overhead  Command          Shared Object     Symbol
     34.56%  program          program          [.] compute_sum
     21.45%  program          libc.so.6        [.] malloc
     14.00%  program          program          [.] do_work
      7.89%  program          libstdc++.so.6   [.] std::vector<int>::push_back
      5.22%  program          program          [.] handle_request
      4.00%  program          libpthread.so.0  [.] pthread_mutex_lock
      3.00%  program          program          [.] process_data

这里 “34.56%” 就表示 compute_sum 占了程序运行期间 34.56% 的 CPU 采样，是主要热点函数。如果没有在输出中看到 compute_sum，则说明它不是当前的热点。

strace -c 用于统计系统调用的分布和耗时，便于判别程序运行是否受到 I/O、文件操作、网络通信等的影响。比如 read、write 占比很高时，表示系统调用层面的 I/O 是耗时大头；若 close 等较多，可能有频繁的资源回收、连接关闭等行为。

perf report --stdio 输出部分：

```
  Overhead  Command          Shared Object     Symbol
     34.56%  program          program          [.] compute_sum
     21.45%  program          libc.so.6        [.] malloc
     14.00%  program          program          [.] do_work
```

strace -c 输出部分：

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 50.00    0.001234         123        10           read
 25.00    0.000617          61        10           write
 25.00    0.000617          61        10           close
------ ----------- ----------- --------- --------- ----------------
100.00    0.002468                     30           total
```

perf diff 输出示例：

```
    40.00%  program_before  [.] foo
    30.00%  program_before  [.] bar
    20.00%  program_after   [.] foo   (-20.00%)   # cpu时间减少了20%
    50.00%  program_after   [.] bar   (+20.00%)
```

perf diff 比较优化前后二进制在相同 workload 下的热点变化。可以直观看到如 foo 函数优化后占用由 40% 降为 20%，而 bar 由 30% 增至 50%，这种变化有助于判定本次优化动作的真实效果，是否引入了新的瓶颈或优化偏移等问题。


具体优化举例

假设有如下性能瓶颈代码：

```cpp
#include <vector>
#include <numeric>

int main() {
    std::vector<int> data(10000000);
    // 错误写法：不断 push_back，存在频繁扩容的低效
    std::vector<int> v;
    for (int i = 0; i < data.size(); ++i) {
        v.push_back(i);
    }
    // 计算总和
    auto sum = std::accumulate(v.begin(), v.end(), 0);
}
```

通过 `perf report` 发现 `std::vector<int>::push_back` 耗时高。针对这个问题的优化示例：

```cpp
#include <vector>
#include <numeric>

int main() {
    std::vector<int> data(10000000);
    // 优化：提前分配空间，避免反复扩容
    std::vector<int> v;
    v.reserve(data.size());
    for (int i = 0; i < data.size(); ++i) {
        v.push_back(i);
    }
    // 计算总和
    auto sum = std::accumulate(v.begin(), v.end(), 0);
}
```

优化后，再执行性能分析，通常会看到 `push_back` 的占用率大幅下降，总体运行时间缩短。

**总结优化过程：**
1. 通过 `perf record` + `perf report` 发现热点函数；
2. 分析代码确认 `push_back` 导致性能瓶颈（多次扩容）；
3. 代码层面优化（如 `reserve` 预分配）；
4. 再次用 `perf` 验证效果，观察瓶颈是否移除。

这种“定位 -> 优化 -> 验证”的流程是 C++ 性能分析和优化的通用范式。



### 技巧 3：持续监控性能

```bash
# 每 10 秒输出一次性能统计
while true; do
    perf stat -p <pid> -- sleep 10
done
```

### 技巧 4：分析特定时间段的性能

```bash
# 分析程序运行后 5-10 秒的性能
perf record -p <pid> -g -- sleep 5  # 等待 5 秒
perf record -p <pid> -g -- sleep 5  # 采集 5 秒数据
perf report
```

---

## 6.3 性能分析最佳实践

1. **先整体后局部**：先用 `perf stat` 了解整体性能，再用 `perf record` 定位热点
2. **多次测量取平均**：性能数据可能有波动，多次测量取平均值更准确
3. **对比优化前后**：使用 `perf diff` 对比优化效果
4. **结合多种工具**：perf + strace + 代码分析，全面了解性能问题
5. **关注关键指标**：IPC、cache-misses、branch-misses 是核心指标
6. **生产环境谨慎**：strace 和 perf 都有性能开销，生产环境谨慎使用

[src: raw/ingested/2技术/性能优化/内存优化-c++性能分析-常见问题与技巧-常见问题与技巧.md]