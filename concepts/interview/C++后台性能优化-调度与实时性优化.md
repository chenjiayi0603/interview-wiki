# C++ 后台性能优化 - 调度与实时性优化

See also: [[Linux线程调度]], [[C++多线程与并发]], [[IPC进程间通信]]

## 5.1 实时调度策略

**原理**：使用实时调度策略保证关键线程的CPU时间。

```cpp
#include <pthread.h>
#include <sched.h>

// 设置线程为SCHED_FIFO实时调度策略
void set_realtime_scheduling(pthread_t thread, int priority) {
    struct sched_param param;
    param.sched_priority = priority;  // 1-99，数字越大优先级越高
    
    // 设置调度策略和优先级
    pthread_setschedparam(thread, SCHED_FIFO, &param);
    
    // 或者使用sched_setscheduler设置进程调度策略
    // sched_setscheduler(0, SCHED_FIFO, &param);
}

// 设置CPU亲和性（结合调度优化）
void set_cpu_affinity_and_scheduling(pthread_t thread, int cpu_id, int priority) {
    // 1. 设置CPU亲和性
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
    
    // 2. 设置实时调度
    struct sched_param param;
    param.sched_priority = priority;
    pthread_setschedparam(thread, SCHED_FIFO, &param);
}
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-5.-调度与实时性优化.md]

## 5.2 PREEMPT_RT实时内核

**优势**：将Linux内核转换为完全可抢占的实时内核。

**安装方法**：
```bash
# Ubuntu/Debian安装RT内核
sudo apt update
sudo apt install linux-image-rt-amd64

# Red Hat/CentOS安装RT内核
sudo yum install kernel-rt kernel-rt-devel

# 验证RT内核
uname -a  # 应该显示 PREEMPT_RT
```

**内核参数调优**：
```bash
# 实时调度参数
echo 1000000 > /proc/sys/kernel/sched_rt_period_us
echo -1 > /proc/sys/kernel/sched_rt_runtime_us  # 允许RT任务用满CPU

# 禁用透明大页（减少内存管理抖动）
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# CPU隔离（隔离核心给实时任务使用）
isolcpus=2,3  # 在GRUB启动参数中添加
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-5.-调度与实时性优化.md]

## 5.3 中断线程化与亲和性

**原理**：将硬件中断处理转换为可调度的内核线程。

```bash
# 查看中断亲和性
cat /proc/interrupts

# 设置IRQ亲和性（将中断绑定到特定CPU）
echo 4 > /proc/irq/IRQ_NUMBER/smp_affinity  # 绑定到CPU2（二进制100）

# 关闭irqbalance服务（防止自动迁移中断）
sudo systemctl stop irqbalance
sudo systemctl disable irqbalance
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-5.-调度与实时性优化.md]

## 5.4 内存锁定与预热

**原理**：避免页面错误导致的调度延迟。

```cpp
#include <sys/mman.h>
#include <unistd.h>

// 锁定进程内存（避免换出）
void lock_process_memory() {
    // 锁定当前和未来分配的内存
    mlockall(MCL_CURRENT | MCL_FUTURE);
}

// 预热内存（提前触发页面错误）
void warmup_memory(void* ptr, size_t size) {
    const size_t page_size = sysconf(_SC_PAGESIZE);
    char* p = (char*)ptr;
    
    // 按页访问内存，触发页面错误
    for (size_t i = 0; i < size; i += page_size) {
        p[i] = 0;  // 写入触发页面分配
    }
}

// 大页内存分配（减少TLB miss）
void* allocate_huge_pages(size_t size) {
    // 使用mmap分配大页内存
    void* ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
    if (ptr == MAP_FAILED) {
        // 回退到普通页面
        ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    }
    return ptr;
}
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-5.-调度与实时性优化.md]

## 5.5 调度优化 - GitHub项目实践案例

### 5.5.1 Linux内核PREEMPT_RT补丁
**GitHub**: https://github.com/linux-rt/linux-rt  
**Stars**: 1.2k+  
**简介**: Linux实时内核补丁

**应用场景**：工业控制、高频交易、实时系统

**技术说明**：PREEMPT_RT补丁将Linux内核转换为完全可抢占的实时内核，主要改进包括：
1. **自旋锁转换为互斥锁**：将不可抢占的自旋锁转换为可睡眠的实时互斥锁
2. **中断线程化**：将硬件中断处理转换为可调度的内核线程
3. **优先级继承**：防止优先级反转问题
4. **高精度定时器**：提供微秒级定时精度

**效果**：最大调度延迟从毫秒级降低到微秒级（通常<100μs）。

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-5.-调度与实时性优化.md]

### 5.5.2 cyclictest - 实时性测试工具
**GitHub**: https://github.com/linux-rt/rt-tests  
**Stars**: 500+  
**简介**: Linux实时性测试工具集

**应用场景**：测量系统调度延迟和实时性

**使用示例**:
```bash
# 测量调度延迟
sudo cyclictest -m -p 80 -n -i 1000 -l 10000

# 参数说明：
# -m: 锁定内存（mlockall）
# -p 80: 设置实时优先级80
# -n: 使用clock_nanosleep
# -i 1000: 间隔1000微秒
# -l 10000: 循环10000次

# 输出示例：
# T: 0 ( 1000) P:80 I:1000 C: 10000 Min:      2 Act:    3 Avg:    4 Max:      23
# Min: 最小延迟(μs), Act: 当前延迟, Avg: 平均延迟, Max: 最大延迟
```

**技术说明**：cyclictest创建高优先级实时线程，定期唤醒并测量实际唤醒时间与预期时间的差异。通过统计延迟分布（P50/P90/P99/P999），评估系统实时性。

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-5.-调度与实时性优化.md]

### 5.5.3 DPDK - 用户态调度优化
**GitHub**: https://github.com/DPDK/dpdk  
**Stars**: 2.4k+  
**简介**: 数据平面开发工具包

**应用场景**：高性能网络数据包处理

**调度优化技术**：
1. **CPU亲和性绑定**：每个worker线程绑定到专用CPU核心
2. **轮询模式驱动**：避免中断，减少上下文切换
3. **无锁数据结构**：减少锁竞争导致的调度延迟
4. **大页内存**：减少TLB miss，提高内存访问性能

**关键配置**:
```bash
# 启动DPDK应用时指定CPU亲和性
./dpdk_app --lcores='0@0,1@1,2@2'  # 线程0绑定CPU0，线程1绑定CPU1...

# 使用大页内存
--huge-dir=/mnt/huge  # 指定大页内存挂载点
```

[src: raw/ingested/2技术/性能优化/C++后台性能优化最佳实践总结-5.-调度与实时性优化.md]