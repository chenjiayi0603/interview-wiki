# Linux 系统 IO 调优

See also: [[Linux线程调度]], [[内存管理]], [[C++网络编程]], [[性能优化]]

## 一、内核参数优化

**关键参数**（`/etc/sysctl.conf`）：

```bash
# 网络优化
net.core.rmem_max = 134217728      # 接收缓冲区最大值
net.core.wmem_max = 134217728      # 发送缓冲区最大值
net.core.netdev_max_backlog = 5000 # 表示内核网络层已经收到但还没交由协议栈处理的数据包队列长度上限。当网络包到达很快而处理速度赶不上时，未处理的数据包会先暂存在这个队列中，队列满时新包会被丢弃。
net.ipv4.tcp_fastopen = 3          # 启用 TCP Fast Open

# 内存优化
vm.swappiness = 0                  # 禁用 swap（低延迟系统）
vm.overcommit_memory = 1           # 允许内存超分配，Linux 内核可以让程序分配超过物理内存+swap的内存（不立即分配实际物理页），适合需要突发大内存但低概率OOM的低延迟系统。

# 调度优化
kernel.sched_rt_runtime_us = -1     # 设置为-1表示对实时进程的CPU运行时间不做限制（允许无限制的实时调度）
kernel.sched_rt_period_us = 1000000 # 设置实时调度时间周期为1秒（1000000微秒），决定了实时任务最大可用的周期窗口。例如配合 kernel.sched_rt_runtime_us=-1 时，实时任务不会被周期限制，适合低延迟需求。

# kernel.sched_rt_period_us 和 kernel.sched_rt_runtime_us 区别说明：
#
# kernel.sched_rt_period_us 指定"评估实时进程 CPU 时间分配"的周期窗口，单位为微秒，通常设为1000000（即1秒）。即每1秒为一轮评估。
# kernel.sched_rt_runtime_us 指定在每个周期内所有实时进程（SCHED_FIFO/SCHED_RR）总共能用的 CPU 时间，单位同样为微秒。
#
# 关系举例：
#   - 如果 period_us 为 1000000，runtime_us 为 950000，表示每1秒最多950ms可用于实时进程，剩下时间保证普通进程有机会运行。
#   - 如果 runtime_us 为 -1，则对实时进程的CPU时间无限制（完全实时），只有实时线程自己阻塞才会让出CPU，非常适合极致低延迟业务，但普通进程可能被"饿死"。

# 应用参数
sysctl -p  # 使配置生效
```

[src: raw/ingested/2技术/性能优化/IO优化-系统io调优.md]

## 二、中断绑定（IRQ Affinity）

**原理**：将网络中断绑定到特定 CPU，避免中断处理影响业务线程。

为什么要将中断处理（IRQ）绑定到特定 CPU？

1. **减少抖动（Jitter）和延迟不确定性**：对于低延迟 C++ 系统，业务线程往往被固定在特定 CPU 上运行。如果网络等外部中断随机到来并在同一 CPU 上被处理，就可能"抢占"业务线程，带来无法预期的中断延迟（比如业务主循环被中断打断，产生延迟波动）。

2. **提升缓存命中率**：中断绑定后，网卡收发包通常固定在目标 CPU "本地"处理，数据更容易命中该 CPU 的缓存，减少跨 CPU 数据搬迁，提升数据包处理速度。
这里的"中断"包括网络中断（如网卡收到数据包时触发的硬件中断）、存储中断等所有会唤醒 CPU 响应的硬件中断，并不仅限于网络。例如高性能 C++ 系统中最主要关注的是网卡收发相关中断。绑定中断到特定 CPU 后，网卡收发包通常会在同一个目标 CPU "本地"被处理。这样优点在于，网络数据和协议栈上下文会更频繁地命中该 CPU 的缓存，减少由于中断/线程跨 CPU 导致的数据迁移、缓存失效，从而提升数据包处理和业务线程的速度与延迟确定性。此外，合理的中断绑定可减少同核业务线程被频繁抢占对调度的负面影响，使高优先级业务流程更加可控。

3. **避免线程/中断迁移引发 perf 失稳**：如果中断每次被不同 CPU 处理，内核需要频繁迁移软中断上下文和数据，带来额外的调度抖动。

4. **支撑高并发与 NUMA 优化**：多核服务器常用多队列网卡，并将不同队列的中断分别绑定到不同核，既提升收包并行处理能力，也便于将网卡、CPU、内存三者都 numactl 绑定到同一节点，本地化流量、降低远程延迟。

5. **避免 irqbalance 意外行为**：Linux 的 irqbalance 服务默认会自动均衡中断到不同 CPU，但其策略未必适合低延迟或业务场景，手动指定能获得可控性和稳定效果。

**结论**：手动绑定中断到专用 CPU，可显著减少高性能系统关键线程/核心被"打扰"，获得延迟可预期性，是低延迟系统收发包/实时事件调度常用利器。

建议：对于极致延迟敏感系统：
  - 业务主线程与网络队列/中断一对一绑定同核 & NUMA 匹配。
  - 采用"专核专职"原则：有的核仅跑业务，有的核仅跑驱动和协议栈中断。

**中断绑定与CPU绑定的区别及影响**

- **中断绑定（IRQ Affinity）**：
  - 是指将某一硬件产生的"中断信号"的处理，指定由某一个或某几个特定CPU核心来负责。
  - 典型用法是将关键的网卡收发中断绑定到业务未占用的CPU或与业务同核，以减少线程被打断或提升缓存命中率。
  - 设置方式：通过 `/proc/irq/<irq号>/smp_affinity` 来指定哪些CPU可以接收、响应该IRQ。
  - 作用对象是**中断处理程序**（如内核的网络收包、驱动部分），不是线程或进程本身。

- **CPU绑定（进程/线程绑定，CPU Affinity/Pinning）**：
  - 指把某个用户空间的进程或线程**固定分配**到特定的CPU核心上运行，避免操作系统随意迁移。
  - 典型用法是高性能C++服务，将主业务线程、事件循环线程绑定到独占CPU防止被调度抖动。
  - 设置方式：`taskset` 工具，或 `sched_setaffinity` 代码接口，或C++线程库API等。
  - 作用对象是**进程/线程调度**，即控制用户进程在哪些CPU上运行。

**两者关系与区别总结**：

- **对象不同**：中断绑定是"硬件中断->CPU"，CPU绑定是"线程/进程->CPU"。
- **配合使用效果最佳**。极致低延迟系统常常"中断+线程都绑同核/配对核"，以最大化同核/L3缓存命中，减少打断和跨核通信。
- **错误配置风险**：如果业务线程和高频中断绑到同一个CPU，可能会互相抢占，导致性能抖动，需合理规划分配。

**影响区别举例**：

1. 只"CPU绑定"而无"中断绑定"：
    - 业务线程虽定在某CPU，但该核有可能还要"接收/响应"所有其它核或全网卡中断，业务线程容易被内核中断抢占（延迟不可控）。

2. 只"中断绑定"而不绑业务线程：
    - 某核虽然只负责中断，但业务线程还可能被系统调度到此核运行，仍难杜绝争用，且业务线程可能频繁迁移，缓存命中变差。

3. "中断/线程合理配合"：
    
    - 将网络队列 1 绑定到 CPU0（IRQ 1 → CPU0），并将 1 号业务线程绑在 CPU0，做到收包、中断、协议栈、业务处理全部局限在同个核（且此核不用处理其它高负载任务），这样延迟达最可控，抖动极低。
    
    - 网络队列（如网卡RX/TX队列）和中断是一一对应的关系：每个网络队列都会对应一个IRQ号（中断号），即每当该网络队列中有新数据到来时，会由网卡/驱动触发一个专门的硬件中断，由内核分配到一个指定CPU处理。通过将特定网卡队列的中断（IRQ）绑定到某个CPU（如队列1对应的IRQ绑定到CPU0），并将负责该流量/功能的业务线程也绑在同一CPU，这样收发包、中断、协议栈与业务处理都严格局限于同一个核，消除跨核中断和迁移带来的抖动，实现极致可控和稳定的低延迟。即：网络队列 ↔ IRQ（中断号） ↔ 目标CPU，网络收发"路径最短化"。

**结论**：
- "中断绑定"与"CPU（线程/进程）绑定"互为补充，两者都合理配置，结合 NUMA 亲和性，是高性能、低延迟 C++系统工程优化的关键基础。
- 推荐：业务线程与关键中断在同核配对，非业务核心收集其它杂项中断和调度，最大化决定性和缓存本地化。


```bash
# 查看中断分布
cat /proc/interrupts

# 绑定中断到 CPU 0（比如网卡的一个收发队列对应的中断号为24，网卡每个硬件队列会分配独立中断号）
echo 1 > /proc/irq/24/smp_affinity  # 这里的 24 通常就是对应网卡某个 RX/TX 队列产生的中断号

# 补充说明："硬件队列"是指网卡在硬件层面上划分出的用于收发网络数据的独立排队单元。多数高端万兆/千兆网卡支持多队列功能，即一个物理网卡会包含多个RX（收包）/TX（发包）硬件队列（如 RX0、RX1、TX0、TX1 等），每个队列可以独立处理一部分网络流量，对应不同的CPU内核进行分流，实现网卡多核多队列负载均衡。在 Linux 下，这些硬件队列通常会映射为 /proc/interrupts 里的多个独立中断号（IRQ），每个中断号就对应着一个 RX 或 TX 队列。

# 这样就能把不同网络队列的中断分配给不同 CPU（比如 RX0→CPU0，RX1→CPU1），具体队列和 IRQ 的具体关系可用如下命令查看：
cat /proc/interrupts | grep eth  # eth 可换成你的网卡名，结果中的每一行就是一个队列的中断号

# 总结：每个网卡队列 <-> 一个中断号 <-> 可分别指定目标 CPU，高性能收包/发包路径更短、可控。

# 或者使用 irqbalance 工具
systemctl stop irqbalance


# ==== 查看网卡硬件队列与中断绑定关系通用方法 ====

# 1. 查看所有中断及其分配设备（推荐直接 grep 网卡名/队列名）：
cat /proc/interrupts | grep eth    # eth为你的网卡名，可替换enp|ens等

# 示例输出（每一行为一个中断，含有网卡队列名）
#  24:  150230   0   0   0  PCI-MSI 524288-edge      eth0-TxRx-0
#  25:  120145   0   0   0  PCI-MSI 524289-edge      eth0-TxRx-1

# 2. 查看对应网卡实际的所有硬件队列及分配关系（以Intel网卡为例）：
ethtool -l eth0         # 查看eth0支持多少个队列/通道
ethtool -S eth0 | grep queue   # 查看详细统计、判断队列活跃性
ethtool -i eth0         # 查看驱动信息

# 3. 批量显示所有以eth0命名的接口的多队列和中断分配：
for irqfile in /proc/irq/*/affinity_hint; do
    [ -f "$irqfile" ] || continue
    irq=$(basename "$(dirname "$irqfile")")
    desc=$(cat /proc/interrupts | grep -w "^ *$irq:" | awk '{print $NF}')
    if echo "$desc" | grep -q "eth0"; then
        echo "IRQ $irq ($desc): affinity_hint=$(cat $irqfile), smp_affinity=$(cat /proc/irq/$irq/smp_affinity)"
    fi
done

# 4. 如需详细分析通配网卡名（如所有ens*多队列）：
cat /proc/interrupts | egrep '(eth|ens|enp)'

# 5. 查看哪些CPU正处理这些队列的中断
# /proc/irq/<IRQ>/smp_affinity 以及 smp_affinity_list 都可查看绑定

# 6. 如果你的驱动支持 xps/rps，可以进一步查看/调整软中断分配：
cat /sys/class/net/eth0/queues/rx-*/rps_cpus
cat /sys/class/net/eth0/queues/tx-*/xps_cpus

# 7. 列举所有网卡的多队列名称
ls -1 /sys/class/net/ | xargs -I{} bash -c 'echo "==== {} ===="; ls /sys/class/net/{}/queues'

# 补充：部分新式万兆/多队列网卡队列名为enpXsY-TxRx-Z，此法同样适用



# 手动绑定
```
    // ==== 6.2 中断绑定完整测试例子 ====

    // 以下是一个演示如何在 Linux 系统中手动将网卡中断（IRQ）绑定到指定 CPU 的完整测试/脚本流程。
    // 实际步骤（需 root 权限）：
    // 1. 查看所有中断及对应设备（如 eth0 网卡）
    // 2. 找到网卡中断号
    // 3. 将该中断 affinity 设为固定位于 CPU 0,1,2... 或其它目标 CPU
    // 4. 验证设置效果

    # 1. 查看当前中断分布（含网卡名，比如 eth0）
    cat /proc/interrupts | grep eth

    # 示例输出（行首数字为中断号）：
    #  24:   5123    0   0   0   PCI-MSI 524288-edge      eth0

    # 2. 查看当前 affinity 设置
    cat /proc/irq/24/smp_affinity

    # 3. 固定中断到 CPU0（十六进制，每一位代表一个 CPU，1=CPU0, 2=CPU1, 4=CPU2, 8=CPU3, ...）
    echo 1 > /proc/irq/24/smp_affinity
    # 如要绑定 CPU1 则用 2，绑定 CPU0 和 CPU1 用 3，等等。

    # 4. 多队列网卡可绑定多个中断到不同 CPU：
    # 假如 eth0 支持4队列，分别是24、25、26、27，可对应映射到 CPU0~CPU3：
    echo 1 > /proc/irq/24/smp_affinity  # CPU0
    echo 2 > /proc/irq/25/smp_affinity  # CPU1
    echo 4 > /proc/irq/26/smp_affinity  # CPU2
    echo 8 > /proc/irq/27/smp_affinity  # CPU3

    # 5. 验证 affinity 成功
    cat /proc/irq/24/smp_affinity_list  # 显示已绑定 CPU 编号

    # 6. 关闭 irqbalance 服务，避免自动修改 affinity
    systemctl stop irqbalance

    # 7. 编写自动化脚本（推荐，适合生产一键绑定所有网卡队列示例）
    # 绑定 eth0 所有队列到 CPU0~CPU3（假定有4队列，中断号自动检索）：

    IFACE="eth0"
    CPU_MAX=3   # 假设有4个CPU

    # 获取该网卡所有 IRQ 号
    IRQS=$(cat /proc/interrupts | grep "$IFACE" | awk '{print $1}' | sed 's/://')
    i=0
    for irq in $IRQS; do
        cpu_mask=$((1 << (i % (CPU_MAX + 1))))
        printf "绑定中断号 %-5s 到 CPU%-2d (%x)\n" "$irq" "$((i % (CPU_MAX + 1)))" "$cpu_mask"
        printf "%x" "$cpu_mask" > /proc/irq/$irq/smp_affinity
        ((i++))
    done

    # 本地内存的形成：在 NUMA（Non-Uniform Memory Access，非一致性内存访问架构）服务器中，通常有多个 CPU 物理插槽，每个 CPU（也称为节点）都会直接连接一部分物理内存条。这部分直接连接的物理内存就称为该 CPU 的"本地内存"（local memory）。当线程在某个 CPU 上运行并申请内存时，操作系统（比如 Linux）会按优先级将内存从当前 CPU 所属节点的本地内存上分配；只有本地内存不足时，才会从其他节点（即"远程内存"）分配。这样形成的"本地内存"，具备该 CPU 访问延迟最低、带宽最高的特点，能最大程度减少跨节点访问所带来的延迟惩罚。因此，合理将线程和中断绑定在同一 NUMA 节点，并利用节点本地内存分配，是高性能和低延迟 C++ 系统设计的关键。

// 中断绑定对性能的影响有多少

// **影响分析：**
// 中断绑定（IRQ Affinity/IRQ Pinning）对高性能、低延迟 C++ 系统至关重要，尤其是在多核和 NUMA 架构下。具体体现在：
// 1. **缓存局部性提升**：将网络等设备的中断固定在与业务线程相同的 CPU 核心和 NUMA 节点上，可以最大程度利用 L1/L2/L3 缓存和本地内存带宽，减少跨 NUMA 的内存访问、缓存失效。
// 2. **降低上下文切换与抢占**：独占核/明确分配中断，避免"中断随意漂移导致 CPU/cache 被无关业务打断"，可极大降低抖动和尾延迟（P99, P999）。
// 3. **避免负载失衡与拥塞**：合理分布多队列中断到不同物理核心，能平衡流量压力，防止单核瓶颈。
// 4. **提升高峰稳定性**：在极端高负载（例如 10G/40G/100G 网络包洪水、百万消息/秒等）下，绑定中断可让延迟抖动收敛于可预测范围，并防止极端"抖峰"。

// **实际收益举例**：  
// - 高频交易、DPDK、软中断受限业务，CPU亲和+中断绑定常带来 10-40%+ 性能提升，P99尾延迟可下降1-2数量级。
// - 若中断、线程错绑定（比如都"亲和"在 NUMA-1，但中断却在 NUMA-0），延迟可比最佳方案高 30-100% 甚至更多，高峰期偶发抖动、缓存Miss暴涨。
// - 在单核/单网卡极端压力下，不绑定时某些业务可能被其它核"抢跑"，导致丢包、延迟长尾飙升。绑定能让性能曲线极大变稳。

// **小结：**
// "中断+线程亲和/绑核"不是可有可无的微调，而是低延迟 C++ 系统的硬性基础设施优化——对 TPS、P99 延迟、带宽极限都有不可替代拉动作用。


[src: raw/ingested/2技术/性能优化/低延迟-低延迟c++系统分析-六、系统调优-六、系统调优.md]

## 三、透明大页（Transparent Huge Pages）禁用

透明大页（Transparent Huge Pages, THP）是一项将物理内存动态聚合为较大内存页（通常为2MB甚至更大）的技术，旨在提升某些工作负载下的内存管理效率、减少TLB（Translation Lookaside Buffer）miss。但在高度低延迟和确定性要求的C++系统中，THP 往往会引入不确定的延迟：

- **分配/回收时长抖动**：THP 的自动分配和合并、碎片整理等操作是异步且不可控的，可能在业务关键路径上抢占CPU，造成不可预测的暂停。
- **内存碎片风险**：频繁的大页分配容易导致物理内存碎片，后续再分配大页时开销飙升。
- **内核行为难以干预**：THP 相关内核线程运行时机不可控，导致抖动溯源困难。

**结论**：对于追求极致确定性响应和微秒级别稳定延迟的应用场景，必须禁用 THP，以消除其带来的潜在延迟波动。绝非"提升性能"的通用优化，反而可能害大于利。


**问题**：THP 可能导致延迟抖动。

```bash
# 禁用透明大页
echo never > /sys/kernel/mm/transparent_hugepage/enabled    # 禁用透明大页功能
echo never > /sys/kernel/mm/transparent_hugepage/defrag     # 禁用THP的自动碎片整理（避免分配时抖动）
```

[src: raw/ingested/2技术/性能优化/IO优化-系统io调优.md]

## 四、禁用 CPU 节能特性

```bash
# 设置 CPU 为性能模式
echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 禁用 C-states（深度睡眠）
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo  # Intel
# 或使用 cpupower
cpupower set -b 0
```

[src: raw/ingested/2技术/性能优化/IO优化-系统io调优.md]

## Related Pages
- [[Linux线程调度]]
- [[内存管理]]
- [[C++网络编程]]
- [[性能优化]]
- [[TCP协议]]
