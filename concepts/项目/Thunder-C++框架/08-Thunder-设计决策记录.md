# Thunder 设计决策记录

> 关键设计决策、方案对比、取舍原因。

---

## 一、协程选型：Stackless vs Stackful

| 维度 | Stackless (C++20) | Stackful (Boost.Context) |
|------|-------------------|--------------------------|
| 栈大小 | 2KB-8KB | 64KB-1MB |
| 切换开销 | 50ns | 200ns |
| 实现方式 | 编译器变换 | 汇编手动切换 |
| 调试 | 正常 | 需特殊工具 |
| 适用场景 | 浅调用链 (< 10 层) | 深调用链 (递归解析) |

**决策**：选择 Stackless
- **原因**：Thunder 的 RPC 请求深度 < 10，不需要大栈
- **收益**：百万并发时内存省 30 倍（2KB vs 64KB/连接）

## 二、IPC 选型：共享内存 vs Unix Socket vs TCP

| 维度 | 共享内存 | Unix Domain Socket | TCP 回环 |
|------|---------|-------------------|----------|
| QPS | 1,500,000 | 500,000 | 300,000 |
| 延迟 | 0.15ms | 0.3ms | 0.5ms |
| syscall | 1 (sem_wait) | 2 (write+read) | 4+ |
| 零拷贝 | ✅ (mmap) | ❌ | ❌ |
| 复杂度 | 高 | 低 | 低 |

**决策**：Manager-Worker 间用共享内存，其他场景用 UDS
- **原因**：Manager-Worker 是高频通信路径，延迟敏感
- **收益**：3.3 倍延迟提升，150 万 QPS

## 三、RPC 协议选型

| 方案 | 头部大小 | 序列化速度 | 生态 |
|------|---------|-----------|------|
| 自研 binary | 22B | 250ns/1KB | 无 |
| Protobuf + HTTP/2 | ~50B | 500ns/1KB | 丰富 |
| msgpack | 可变 | 300ns/1KB | 中等 |

**决策**：自研 binary 协议
- **原因**：头部最小、序列化最快、无需 protoc 编译
- **代价**：需要额外 IDL 和版本管理

## 四、序列化方案对比

| 方案 | Size int32 | 序列化 1KB | 反序列化 1KB | 版本兼容 |
|------|-----------|------------|-------------|---------|
| Thunder binary | 4B | 250ns | 200ns | 手动 |
| Protobuf | 10B (varint) | 500ns | 400ns | 自动 |
| Flatbuffers | 4B | 0 (zero-copy) | 200ns | 自动 |
| Cap'n Proto | 4B | 0 (zero-copy) | 200ns | 自动 |

**决策**：自研 binary（初期）→ 考虑迁移 Flatbuffers（后期）
- **原因**：初期自研开发最快，后期需要更完善的兼容性时切换

## 五、Raft 优化点取舍

| 优化 | 效果 | 复杂度 | 是否采用 |
|------|------|--------|---------|
| 批量日志复制 | QPS 100x | 低 | ✅ |
| 预投票 Pre-Vote | 防止分裂 | 低 | ✅ |
| ReadIndex 优化 | 线性一致性读 | 中 | ✅ |
| 流水线复制 | 延迟降低 30% | 高 | ❌ |
| 联合共识 (Joint Consensus) | 安全成员变更 | 高 | ❌ |
| 快照分流 | 大快照不阻塞 | 中 | ✅ |

**不采用原因**：
- **流水线复制**：批量复制已足够，流水线增加复杂性
- **联合共识**：单节点变更足够安全，实现更简单

## 六、连接模型选型

| 模型 | 说明 | 适用 |
|------|------|------|
| 1 线程 1 epoll | 多线程各自 epoll | Thunder 设计 |
| 多路复用 (reactor) | 单 epoll + 多 Worker | Redis |
| 每个连接 1 线程 | 连接数受限 | Apache |

**决策**：每 Worker 绑定 CPU + 独立 epoll
- **原因**：避免锁竞争，利用 CPU 亲和性
- **收益**：50 万连接时 CPU 使用率仅 15%（空闲）

## 七、IO 引擎：epoll vs io_uring

| 维度 | epoll (EvIoBackend) | io_uring (AsioUringIoBackend) |
|------|---------------------|------------------------------|
| 内核版本 | 2.6+ | 5.1+（推荐 5.10+） |
| 实现方式 | 每个 fd 一个 ev_io watcher | ring_fd 一个 ev_io watcher |
| 写操作 | 异步，注册 EV_WRITE | 同步，直接 WriteFD（小包收益低） |
| EAGAIN 处理 | epoll 自动重触发 | 需显式重新 SubmitRead |

**实测数据（WSL2, 12 vCPUs, wrk 4 threads, 2026-05-11）**：

| 场景 | ev (epoll) | asio_uring | 差距 |
|------|-----------|------------|------|
| 小包 37B, c100 | 160,674 RPS / 705μs | 164,358 RPS / 2.49ms | RPS ≈持平 |
| 小包 37B, c500 | 187,832 RPS / 3.66ms | 164,086 RPS / 1.75ms | ev 吞吐略高 |
| 4KB, c100 | 73,137 RPS / 1.51ms | 68,677 RPS / 0.99ms | 延迟低 **34%** |
| 4KB, c500 | 60,106 RPS / 32.0ms | 68,679 RPS / 17.2ms | RPS 高 **14%** |
| 64KB, c100 | 6,207 RPS / 16.78ms | 6,675 RPS / 2.32ms | 延迟低 **86%** |
| 64KB, c500 | 6,370 RPS / 76.42ms | 5,896 RPS / 42.87ms | 尾延迟 1/50 |

**决策**：保留 ev + asio_uring 两档，移除手写 UringIoBackend
- **小包场景**：ev 和 asio_uring 吞吐持平，ev 略高 1.8%（在 WSL2 噪声范围内）
- **大包/超包场景**：io_uring 全面占优——4KB 延迟低 34%，64KB 延迟低 86%，尾延迟降至 1/50
- **架构选择**：主线程直驱（ev_prepare/ev_check/ring_fd 三路驱动）优于独立线程桥接，消除跨线程开销
- **代价**：依赖 Linux 5.1+，低版本内核自动降级到 epoll

## 八、为什么不用 gRPC？

| 需求 | gRPC | Thunder |
|------|------|---------|
| QPS 要求 > 100 万 | 8 万 | 200 万 |
| 内存 < 30KB/连接 | 50KB | 20KB |
| Raft 内置 | ❌ | ✅ |
| 动态插件 | ❌ | ✅ |
| C++20 协程 | ❌ | ✅ |

**结论**：gRPC 在性能、资源、功能三个维度均不满足需求。

## 九、为什么不用 brpc？

| 维度 | brpc | Thunder |
|------|------|---------|
| Raft 支持 | braft（独立库） | 内置 |
| 协程 | bthread（stackful） | C++20 stackless |
| 动态插件 | ❌ | ✅ |
| 文档 | 完善 | 待完善 |
| 成熟度 | 工业级 | 学习项目 |

**结论**：brpc 成熟度更高，但 Thunder 在协程和插件上更灵活。Thunder 更适合作为 C++20 协程实践和分布式系统学习的载体。

## 十、时序图：一次完整 RPC 调用

```
Client                Worker               Manager              Server
  │                     │                    │                   │
  │── TCP connect ─────→│                    │                   │
  │←── accept ──────────│                    │                   │
  │                     │── shm write ──────→│                   │
  │                     │                    │── RPC forward ───→│
  │                     │                    │←── process ──────│
  │                     │←── shm read ───────│                   │
  │←── response ────────│                    │                   │
```

## 十一、未来优化方向

1. **io_uring 全面替换 epoll**：实现零 syscall IO
2. **支持 RDMA**：数据中心内超低延迟 RPC
3. **动态负载均衡**：自适应请求转发策略
4. **WASM 插件支持**：更安全的动态扩展
5. **多语言客户端**：Python / Go / Rust SDK
