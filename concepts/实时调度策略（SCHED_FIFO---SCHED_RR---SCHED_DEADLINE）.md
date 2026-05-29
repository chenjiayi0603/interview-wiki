# 实时调度策略（SCHED_FIFO / SCHED_RR / SCHED_DEADLINE）

See also: [[Linux线程调度]], [[低延迟C++实时性保障]], [[低延迟Linux实时系统调试验证面试要点]]

## 3.1 调度类与优先级

Linux 调度类分层次：

- **实时类**：`SCHED_FIFO` / `SCHED_RR` / `SCHED_DEADLINE`  
- **普通类**：`SCHED_OTHER` (CFS) + nice 值

优先级含义：

- **实时优先级 1–99，数字越大优先级越高**。  
- 所有实时线程优先于 CFS 线程，被调度时会抢占普通任务。
- 普通 nice 优先级（100–139）只在所有实时 runnable 队列为空时才起作用。

## 3.2 SCHED_FIFO 与 SCHED_RR

**共性：**

- 都是 **实时调度策略**，使用 `pthread_setschedparam` 或 `sched_setscheduler` 设置。
- 都要求 root 或具有 `CAP_SYS_NICE` 能力。

**SCHED\_FIFO：**

- 同优先级队列内严格 FIFO，无时间片；线程一旦获得 CPU 会一直运行，直到：
  - 主动阻塞（I/O、锁等待等）；
  - 主动让出 CPU（如 `sched_yield`）；
  - 退出。
- 适合 **单/少量极关键线程**，追求极致低延迟，不在意多线程公平性。

**SCHED\_RR：**

- 与 SCHED\_FIFO 一样按照实时优先级调度，但同级线程有 **时间片轮转**。
- 适合 **多个同等级实时线程** 需要周期性获得 CPU、避免互相饿死的场景。

**从两个文档综合出的实践建议：**

- 单个极度关键环（如高频撮合主循环、DPDK 主 poll 线程）：  
  → **SCHED\_FIFO + 高优先级（80+）**。
- 多个协同实时线程（多路行情解码、风控、控制环路）：  
  → **SCHED\_RR + 合理时间片**，防止有线程饥饿。
- 非关键后台线程：保持 **SCHED\_OTHER + 合理 nice**，避免打断实时线程。

## 3.3 SCHED_DEADLINE（简述）

文档中简单提到：

- 按 **运行时间 / 周期 / 截止时间** 预留 CPU 带宽（CBS 模型）。
- 适用于周期性或偶发硬实时任务（如固定 1ms 控制回路）。
- 需要内核打开 `CONFIG_SCHED_DEADLINE`，通过 `sched_setattr()` 设置。

调度优化角度：  
若业务是“严格周期 + 有 deadline 上界”的控制环，可以考虑 `SCHED_DEADLINE`，通过内核保证 CPU 上的带宽预留和截止时间满足，而不是手工调度优先级。

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-三、实时调度策略（SCHED_FIFO---SCHED_RR---SCHED_DEADLINE）.md]