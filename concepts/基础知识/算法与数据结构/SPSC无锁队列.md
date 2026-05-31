# SPSC 无锁队列（std::atomic，单生产者单消费者）

> SPSC = Single Producer Single Consumer。  
> head 只由消费者推进，tail 只由生产者推进——无竞争写，不需要 CAS，只需 acquire/release 保证跨进程可见性。

[src: raw/ingested/2技术/cpp/C++多进程完整手册-九、SPSC-无锁共享内存队列（std--atomic）.md]

## 代码示例

```cpp
#include <atomic>
#include <cstring>
#include <cstdio>
#include <sys/mman.h>
#include <unistd.h>
#include <sys/wait.h>

static const int CAP  = 8;    // 槽位数
static const int MSGSZ = 64;  // 每条消息最大字节

struct SpscRing {
    std::atomic<int> rd{0};        // 消费者游标（只消费者写）
    std::atomic<int> wr{0};        // 生产者游标（只生产者写）
    char slot[CAP][MSGSZ];
};

// 生产者调用
bool enqueue(SpscRing* r, const char* msg) {
    int w    = r->wr.load(std::memory_order_relaxed);
    int next = (w + 1) % CAP;
    if (next == r->rd.load(std::memory_order_acquire))
        return false;                                  // 满
    strncpy(r->slot[w], msg, MSGSZ - 1);
    r->slot[w][MSGSZ - 1] = '\0';
    r->wr.store(next, std::memory_order_release);      // 发布：数据已就绪
    return true;
}

// 消费者调用
bool dequeue(SpscRing* r, char* out) {
    int rd = r->rd.load(std::memory_order_relaxed);
    if (rd == r->wr.load(std::memory_order_acquire))
        return false;                                  // 空
    strncpy(out, r->slot[rd], MSGSZ);
    r->rd.store((rd + 1) % CAP, std::memory_order_release); // 发布：槽位已释放
    return true;
}

int main() {
    void* mem = mmap(NULL, sizeof(SpscRing),
                     PROT_READ | PROT_WRITE,
                     MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    SpscRing* ring = new(mem) SpscRing{};   // placement new 初始化 atomic

    if (fork() == 0) {          // 子进程：生产者
        enqueue(ring, "hello");
        enqueue(ring, "world");
        return 0;
    }
    wait(nullptr);
    char buf[MSGSZ];
    while (dequeue(ring, buf))
        printf("recv: %s\n", buf);  // recv: hello / recv: world

    ring->~SpscRing();
    munmap(mem, sizeof(SpscRing));
    return 0;
}
```

## 内存序说明

| 操作 | 内存序 | 原因 |
|------|--------|------|
| 读对方游标（检查空/满） | `acquire` | 看到对方最新写入 |
| 写自己游标（推进） | `release` | 确保数据写完再发布游标 |
| 读自己游标 | `relaxed` | 只有自己修改，无竞争 |

## 与 mutex 版对比

| 维度 | mutex 版（3.4节） | atomic SPSC（本节） |
|------|-----------------|-------------------|
| 同步方式 | `pthread_mutex_t` + `PTHREAD_PROCESS_SHARED` | `std::atomic` + placement new |
| 性能 | 有系统调用（futex） | 无系统调用，纯用户态 |
| 适用场景 | 多生产者或多消费者 | 严格 1 生产者 + 1 消费者 |
| std::mutex 能用吗 | ❌ 无法设 PROCESS_SHARED | — |

## 相关页面
- [[SPMC无锁队列]]
- [[MPMC环形无锁队列-Vyukov]]
- [[IPC进程间通信]]
- [[C++多线程与并发]]