# ShmRingQueue 集成方案（实际实现）

## 4.1 tagWorkerAttr 扩展

**文件**: `code/Net/include/labor/types/WorkerAttr.hpp`

```cpp
struct tagWorkerAttr
{
    // ... 现有字段（iWorkerIndex, iControlFd, iDataFd, iLoad 等）保持不变 ...

    // 新增共享内存队列
    ShmRingQueue* pMgrToWorkerQueue = nullptr;   // Manager → Worker 消息队列
    ShmRingQueue* pWorkerToMgrQueue = nullptr;   // Worker → Manager 消息队列
    int           iMgrToWorkerEventFd = -1;       // 通知 Worker 的 eventfd
    int           iWorkerToMgrEventFd = -1;       // 通知 Manager 的 eventfd
};
```

## 4.2 Worker 类扩展

**文件**: `code/Net/include/labor/Worker.hpp`

Worker 类新增：
- 构造函数增加 shm 参数（队列指针 + eventfd）
- `static void ShmReadCallback(struct ev_loop*, struct ev_io*, int revents)` — libev 回调
- 新增 private 成员：`m_pMgrToWorkerQueue`, `m_pWorkerToMgrQueue`, `m_iMgrToWorkerEfd`, `m_iWorkerToMgrEfd`, `m_pShmReadWatcher`
- 前置声明 `struct ShmRingQueue;` 在 `namespace net` 外部（避免命名空间冲突）

## 4.3 进程创建流程（Manager::CreateWorker）

**文件**: `code/Net/src/labor/Manager.cpp`

```
Manager::CreateWorker()
  │
  ├── 1. pMgrToWorker = ShmRingQueue::Create(128, 4096)   // mmap MAP_SHARED|MAP_ANON
  ├── 2. pWorkerToMgr = ShmRingQueue::Create(128, 4096)
  ├── 3. iMgrToWorkerEfd = eventfd(0, EFD_NONBLOCK|EFD_SEMAPHORE)
  ├── 4. iWorkerToMgrEfd = eventfd(0, EFD_NONBLOCK|EFD_SEMAPHORE)
  ├── 5. socketpair(...) iControlFds     // 保留，降级通道
  ├── 6. socketpair(...) iDataFds        // 保留，fd 传递
  │
  ├── 7. fork()
  │     │
  │     ├── 子进程(Worker):
  │     │     Worker(path, iControlFd, iDataFd, index, conf,
  │     │            pMgrToWorker, pWorkerToMgr,
  │     │            iMgrToWorkerEfd, iWorkerToMgrEfd)
  │     │     关闭 Manager 端的 eventfd
  │     │
  │     └── 父进程(Manager):
  │           关闭 Worker 端的 eventfd
  │           保存 pMgrToWorker, pWorkerToMgr, efd 到 tagWorkerAttr
```

## 4.4 Manager 侧集成

### 发送（Manager → Worker）— Shm 优先，自动降级

```cpp
bool Manager::SendToWorker(const MsgHead& oMsgHead, const MsgBody& oMsgBody)
{
    for (auto& [pid, attr] : m_mapWorker)
    {
        // 优先 shm
        if (attr.pMgrToWorkerQueue &&
            attr.pMgrToWorkerQueue->TryEnqueue(
                oMsgHead.cmd(), oMsgHead.seq(),
                oMsgBody.body().data(), oMsgBody.body().size()))
        {
            ShmRingQueue::NotifyEventFd(attr.iMgrToWorkerEventFd);
            continue;  // shm 写入成功，跳过 socket
        }
        // 降级走原有 socket，打 WARN 日志
        LOG4_WARN("shm queue full, fallback to socket for worker %d", attr.iWorkerIndex);
        auto iter = m_mapFdAttr.find(attr.iControlFd);
        if (iter != m_mapFdAttr.end())
        {
            SendTo(iter->second.get(), oMsgHead, oMsgBody);
        }
    }
    return true;
}
```

### 接收（Worker → Manager）— 批量 drain

在 `Manager::CheckWorker()` 中为每个 Worker 的 `iWorkerToMgrEventFd` 批量消费：

```cpp
// 读取 eventfd 计数器值
uint64_t count;
ssize_t n = read(attr.iWorkerToMgrEventFd, &count, sizeof(count));
// 批量 dequeue，构造 MsgHead/MsgBody，调用 DisposeDataFromWorker()
while (attr.pWorkerToMgrQueue->TryDequeue(cmd, seq, body_buf, body_len))
{
    MsgHead oMsgHead;
    MsgBody oMsgBody;
    oMsgHead.set_cmd(cmd);
    oMsgHead.set_seq(seq);
    oMsgBody.set_body(body_buf, body_len);
    DisposeDataFromWorker(attr, oMsgHead, oMsgBody);
}
```

## 4.5 Worker 侧集成

**文件**: `code/Net/src/labor/Worker.cpp`

### 构造函数

接收并存储 shm 队列指针和 eventfd：

```cpp
Worker::Worker(const std::string& strWorkPath, int iControlFd, int iDataFd,
               int iWorkerIndex, util::CJsonObject& oJsonConf,
               ShmRingQueue* pMgrToWorker, ShmRingQueue* pWorkerToMgr,
               int iMgrToWorkerEfd, int iWorkerToMgrEfd)
    : m_pMgrToWorkerQueue(pMgrToWorker)
    , m_pWorkerToMgrQueue(pWorkerToMgr)
    , m_iMgrToWorkerEfd(iMgrToWorkerEfd)
    , m_iWorkerToMgrEfd(iWorkerToMgrEfd)
    , m_pShmReadWatcher(nullptr)
{ ... }
```

### 发送给 Manager — Shm 优先

```cpp
bool Worker::SendToParent(const MsgHead& oMsgHead, const MsgBody& oMsgBody)
{
    // 优先 shm
    if (m_pWorkerToMgrQueue &&
        m_pWorkerToMgrQueue->TryEnqueue(
            oMsgHead.cmd(), oMsgHead.seq(),
            oMsgBody.body().data(), oMsgBody.body().size()))
    {
        ShmRingQueue::NotifyEventFd(m_iWorkerToMgrEfd);
        return true;
    }
    // 降级走原有 socket，打 WARN 日志
    LOG4_WARN("shm worker->mgr queue full, fallback to socket");
    // ... 现有 socket 代码 ...
}
```

### libev 回调处理 shm 消息

`ShmReadCallback` 注册在 `CreateEvents()` 中：

```cpp
bool Worker::CreateEvents()
{
    // ... 现有事件注册 ...

    // 注册 shm eventfd 读事件
    if (m_iMgrToWorkerEfd >= 0)
    {
        m_pShmReadWatcher = new ev_io;
        ev_io_init(m_pShmReadWatcher, ShmReadCallback, m_iMgrToWorkerEfd, EV_READ);
        ev_io_start(m_loop, m_pShmReadWatcher);
        LOG4_INFO("shm read watcher registered for worker %d", GetWorkerIndex());
    }
    return true;
}
```

```cpp
void Worker::ShmReadCallback(struct ev_loop* loop, struct ev_io* watcher, int revents)
{
    // 消费 eventfd 计数器（EFD_SEMAPHORE 模式每次 read 返回 1）
    uint64_t val;
    ssize_t n = read(watcher->fd, &val, sizeof(val));

    Worker* pWorker = ...; // 从 watcher data 获取
    // 批量 dequeue
    uint32_t cmd, seq, body_len;
    char body_buf[4096];
    while (pWorker->m_pMgrToWorkerQueue->TryDequeue(cmd, seq, body_buf, body_len))
    {
        // 构造 MsgHead/MsgBody 并调用 Dispose()
        MsgHead oMsgHead;
        MsgBody oMsgBody;
        oMsgHead.set_cmd(cmd);
        oMsgHead.set_seq(seq);
        oMsgBody.set_body(body_buf, body_len);
        pWorker->Dispose(..., oMsgHead, oMsgBody, ...);
    }
}
```

### 销毁

```cpp
void Worker::Destroy()
{
    // 停止并删除 shm ev_io watcher
    if (m_pShmReadWatcher)
    {
        ev_io_stop(m_loop, m_pShmReadWatcher);
        delete m_pShmReadWatcher;
        m_pShmReadWatcher = nullptr;
    }
    // 关闭 Worker 侧 eventfd（Manager 侧由 Manager::Destroy() 关闭）
    ShmRingQueue::CloseEventFd(m_iMgrToWorkerEfd);
    ShmRingQueue::CloseEventFd(m_iWorkerToMgrEfd);
    // ... 现有清理 ...
}
```

## 4.6 Worker 重启处理

`Manager::RestartWorker()` 中：
1. 销毁旧 shm 队列：`ShmRingQueue::Destroy(attr.pMgrToWorkerQueue, 128, 4096)`
2. 关闭旧 eventfd
3. 创建新 shm 队列和 eventfd
4. fork 新 Worker，传入新队列指针

## 4.7 Manager::Destroy 清理

遍历所有 Worker，销毁 shm 队列和 eventfd：

```cpp
void Manager::Destroy()
{
    for (auto& [pid, attr] : m_mapWorker)
    {
        ShmRingQueue::Destroy(attr.pMgrToWorkerQueue, 128, 4096);
        ShmRingQueue::Destroy(attr.pWorkerToMgrQueue, 128, 4096);
        ShmRingQueue::CloseEventFd(attr.iMgrToWorkerEventFd);
        ShmRingQueue::CloseEventFd(attr.iWorkerToMgrEventFd);
    }
    // ... 现有清理 ...
}
```

[src: raw/ingested/3项目/分布式IM-雷漫/源码-plan_shm_queue-四、集成方案（实际实现）.md]