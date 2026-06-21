# Thunder 分布式异步集群框架 总览

> 自研高性能 C++20 分布式框架，RPC + 协程 + Raft 共识 + 动态插件。

---

## 一、文件地图

| 文件 | 内容 |
|------|------|
| **01-Thunder-架构设计.md** | 系统架构、协程模型、Raft 实现、插件系统 |
| **02-Thunder-性能数据.md** | 连接能力、RPC 吞吐、Raft 性能、对比数据 |
| **03-Thunder-面试考点速查.md** | STAR 故事、高频 Q&A |
| **04-Thunder-RPC协议设计.md** | RPC 协议格式、序列化、路由、服务发现 |
| **05-Thunder-协程调度器设计.md** | 调度器架构、工作窃取、IO 集成、定时器 |
| **06-Thunder-Raft实现详解.md** | Raft 状态机、选举优化、日志复制、快照 |
| **07-Thunder-部署与配置.md** | 编译构建、配置说明、部署架构、监控 |
| **08-Thunder-设计决策记录.md** | 关键设计决策、方案对比、取舍原因 |
| **09-Thunder-io_uring集成示例.md** | io_uring 协程化完整实现、使用示例、性能对比 |
| **10-Thunder-K8s部署.md** | Kubernetes 容器化部署、ConfigMap/Deployment/Service 配置 |

## 二、项目速览

| 项目 | 数值 |
|------|------|
| 语言 | C++20（原生协程） |
| 单进程连接 | 500,000（16 核 32GB） |
| 空请求 RPC QPS | 2,000,000 |
| Raft 选主时间 | 45ms（3 节点，1ms 网络） |
| 协程切换开销 | 50ns（stackless） |
| 对比 gRPC | QPS 高 25 倍，内存省 60% |

## 三、核心特性

- C++20 stackless 协程调度器（50ns 切换）
- RPC 框架（共享内存 IPC 150 万 QPS）
- Raft 共识算法（批量日志复制 10 万 QPS）
- 动态库插件热加载
- epoll/io_uring 事件驱动
- 多进程 Worker 架构 + 共享内存 IPC
