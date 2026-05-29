# OpenMP 概述

## OpenMP 简介

**OpenMP**（Open Multi-Processing）是一个支持多平台共享内存并行编程的 API，采用指令式并行编程模型。

**背景说明**：
- **标准化与维护**：规范由 **OpenMP ARB**（Architecture Review Board，架构审查委员会）发布与演进；成员涵盖处理器、编译器与 HPC 厂商，保证跨实现的一致性。OpenMP 不并入 C/C++/Fortran 语言标准本身，而是作为这些语言上可移植的**并行扩展约定**存在。
- **发展脉络**：首个公开版本 **OpenMP 1.0** 于 **1997 年**推出，统一了此前各厂商互不兼容的共享内存并行指令实践。此后历经 2.x、3.x、4.x、**5.x** 等版本，逐步加入任务并行、SIMD、加速器/卸载（offload）等能力，以适配多核 CPU、众核与异构计算趋势。
- **技术定位**：面向 **单进程、共享地址空间** 的并行（多线程），典型部署在 **多核/多路 SMP 单机或单节点**；与 **MPI**（多进程、分布式内存、跨节点）互补，常见组合为「节点内 OpenMP + 节点间 MPI」。相对直接使用 **Pthreads** 等底层线程 API，OpenMP 更强调通过编译指令对循环与代码块做**增量式并行化**，降低改造串行代码的成本。

**核心特点**：
- **指令式编程**：通过编译指令（pragma）控制并行
- **共享内存模型**：所有线程共享同一内存空间
- **增量并行化**：可以逐步将串行代码改为并行
- **可移植性**：支持 C/C++/Fortran，跨平台
- **简单易用**：相比 MPI 等消息传递模型更简单

## 适用边界与限制

**为什么在部分领域看起来“不够流行”**：
- **场景差异**：OpenMP 擅长解决单机多核 CPU 的计算并行；而很多业务系统以网络 I/O、异步编排、分布式扩展为主，关注点不同。
- **生态差异**：在通用应用开发中，常见并发方案是语言/框架自带模型或任务运行时（如线程池、协程、任务调度框架），OpenMP 的可见度相对较低。
- **GPU 关注度更高**：异构计算场景下，很多团队优先投入 CUDA/SYCL 等生态，导致 OpenMP 在舆论层面的“存在感”弱于其实际使用量。

**工程约束与常见限制**：
- **共享内存边界**：OpenMP 主要面向同一地址空间内的线程并行；跨节点分布式并行通常需要与 MPI 等技术结合。
- **并行收益依赖负载特征**：任务粒度过小、同步过多、缓存争用或 false sharing 会显著降低加速比，甚至慢于串行版本。
- **可移植性与版本差异**：不同编译器对 OpenMP 版本和特性的支持程度不同，实际工程中需关注兼容性与降级策略。
- **调试复杂度更高**：数据竞争、共享/私有变量划分错误、归约配置不当等问题，往往比串行代码更难复现与定位。

## OpenMP 执行模型

```
主线程（Master Thread）
    ↓
遇到 #pragma omp parallel
    ↓
创建 N 个线程（Thread Team）
    ↓
所有线程执行并行区域代码
    ↓
并行区域结束，线程合并
    ↓
继续串行执行
```

**关键概念**：
- **Fork-Join 模型**：主线程遇到并行区域时 fork 出多个线程，执行完后 join 回主线程
- **线程团队**：并行区域内的所有线程组成一个团队
- **线程 ID**：每个线程有唯一 ID（0 到 N-1）

## 编译与运行

**编译选项**：
```bash
# GCC
gcc -fopenmp program.c -o program

# Clang
clang -fopenmp program.c -o program

# MSVC (Windows)
cl /openmp program.c

# Intel Compiler
icc -qopenmp program.c -o program
```

**运行时环境变量**：
```bash
# 设置线程数
export OMP_NUM_THREADS=4

# 设置线程绑定
export OMP_PROC_BIND=true

# 设置线程调度策略
export OMP_SCHEDULE=dynamic,4

# 显示线程信息
export OMP_DISPLAY_ENV=true
```

[src: raw/ingested/2技术/cpp/并行库-C++_openmp并行编程-概述.md]