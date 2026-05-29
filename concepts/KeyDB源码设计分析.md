# KeyDB 源码层面的关键设计分析

> 本文基于 KeyDB 源码分析，涵盖网络层、连接管理、FastLock 实现、事务处理与 MVCC 快照机制。

See also: [[KeyDB存算分离项目]], [[RocksDB文件体系]], [[C++多线程与并发]], [[原子操作与内存模型]]

---

## 6.1 网络层：SO_REUSEPORT + 每线程独立事件循环

```c
// 每个 worker 线程都独立 bind + listen 同一端口
int fd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &yes, sizeof(yes));
bind(fd, addr, addrlen);
listen(fd, backlog);

// 内核自动在多个 listen socket 间负载均衡新连接
// 避免了多线程 accept 竞争
```

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-六、源码层面的关键设计分析.md]

## 6.2 连接管理：连接与线程绑定

```
连接生命周期：
  accept() → 分配给当前 Worker 线程
  → 该连接的所有后续操作都在同一线程内完成
  → 包括 read/parse/execute/write
  → 连接关闭也在同一线程

优势：
  - 避免跨线程传递连接的同步开销
  - 每个线程维护自己的客户端链表
  - 连接数据的局部性好，缓存友好
```

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-六、源码层面的关键设计分析.md]

## 6.3 FastLock 实现（x86-64 汇编）

FastLock 的核心是一个 **ticket lock** 变体：

```
结构体：
  active:  当前正在服务的票号
  avail:   下一个可发放的票号

获取锁：
  1. 原子递增 avail，获得自己的票号 my_ticket
  2. 自旋等待直到 active == my_ticket
  3. （自旋等待期间可执行 async rehash）

释放锁：
  1. 原子递增 active
  2. 下一个等待者的票号匹配，获得锁

特性：
  - 公平性：FIFO 顺序，避免饥饿
  - 递归支持：m_depth 记录同一线程的重入次数
  - 无系统调用：纯用户态操作
```

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-六、源码层面的关键设计分析.md]

## 6.4 事务（MULTI/EXEC）的处理

事务在 EXEC 时持有锁直到所有命令执行完毕，保证原子性：

```
MULTI  → 命令入队（不需要锁）
EXEC   → 获取 FastLock
       → 依次执行队列中所有命令
       → 释放 FastLock
```

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-六、源码层面的关键设计分析.md]

## 6.5 MVCC 快照机制

```
写操作流程：
  1. 获取 FastLock
  2. 创建数据的新版本
  3. 更新版本链
  4. 释放 FastLock

读操作流程（async mode）：
  1. 获取当前快照引用（无锁，原子操作）
  2. 在快照上执行读操作
  3. 释放快照引用

无需获取全局锁即可完成读操作。
```

[src: raw/ingested/2技术/rocksdb/keydb的性能分析-六、源码层面的关键设计分析.md]