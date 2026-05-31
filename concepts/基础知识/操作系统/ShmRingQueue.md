# ShmRingQueue

> 基于共享内存的环形无锁队列，用于进程间高性能通信。

See also: [[IPC进程间通信]], [[POSIX共享内存]], [[MPMC环形无锁队列-Vyukov]], [[分布式IM消息系统-雷漫]]

## 概述

`ShmRingQueue` 是雷漫分布式 IM 系统中用于进程间通信的核心数据结构，基于 `mmap(MAP_SHARED|MAP_ANONYMOUS)` 实现共享内存环形队列，支持多生产者/多消费者无锁操作。

## 实现细节

### 文件位置

- 头文件：`code/Net/include/labor/types/ShmRingQueue.hpp`（约 230 行）
- 单元测试：`code/test/labor/test_shm_queue.cpp`（约 300 行）

### 核心方法

| 方法 | 说明 |
|------|------|
| `Create()` / `Destroy()` | 工厂方法，基于 `mmap(MAP_SHARED|MAP_ANONYMOUS)` 创建/销毁共享内存队列 |
| `TryEnqueue()` / `TryDequeue()` | 核心入队/出队方法，使用原子操作 + release/acquire 语义保证无锁安全 |
| `NotifyPending()` | 通知待处理事件 |
| `MaxBodySize()` | 返回最大消息体大小 |
| `Count()` | 返回当前队列中元素数量 |
| `CreateEventFd()` / `NotifyEventFd()` / `CloseEventFd()` | eventfd 辅助方法，用于事件通知 |
| `GetSlotData()` | 内存偏移计算，获取指定槽位数据指针 |

### 测试结果

- 7 个单元测试 + 3 个端到端测试 = 10/10 全部通过

## 集成到 Worker 架构

### Phase 2：双写双读（已完成）

- `tagWorkerAttr` 新增 4 个字段（2 队列指针 + 2 eventfd）
- `Worker.hpp` 新增 shm 成员变量 + `ShmReadCallback` 声明
- `Manager::CreateWorker()` 中创建 shm + eventfd
- `Manager::RestartWorker()` 中重建 shm + eventfd
- `Manager::SendToWorker()` 双写（shm 优先，socket 降级 + WARN）
- `Manager::CheckWorker()` 批量 drain Worker→Manager 队列
- `Worker` 构造函数接收 shm/efd 参数
- `Worker::SendToParent()` 双写（shm 优先，socket 降级）
- `Worker::CreateEvents()` 注册 ev_io watcher 监听 eventfd
- `Worker::ShmReadCallback()` libev 回调，批量 dequeue + 分发
- `Worker::Destroy()` 中清理 shm/efd 资源
- `Manager::Destroy()` 中清理所有 Worker 的 shm/efd 资源
- 编译通过 `cmake --build build -j1`
- 安装到 deploy/ `cmake --install build`
- 生产节点重启，10 个进程全部健康运行

### Phase 3：eventfd 通知优化（未开始）

- [ ] 统计降级次数，加到上报指标中
- [ ] 性能压测对比

### Phase 4：清理（可选，未开始）

- [ ] 如果 shm 路径稳定运行足够长时间无降级，可评估移除 socketpair 控制通道
- [ ] **注意**：`iDataFd` socketpair 永远保留，用于 `send_fd_with_attr` 传递客户端 fd

[src: raw/ingested/3项目/分布式IM-雷漫/源码-plan_shm_queue-六、实施完成情况.md]