# SPMC 无锁队列（std::atomic CAS，单生产者多消费者）

> 1 个生产者直接 store wr，多个消费者用 **CAS 抢占 rd**，抢到后再读数据。

[src: raw/ingested/2技术/cpp/C++多进程完整手册-九、SPSC-无锁共享内存队列（std--atomic）.md]

## 代码示例

```cpp
#include <atomic>
#include <cstring>
#include <cstdio>
#include <sys/mman.h>
#include <unistd.h>
#include <sys/wait.h>

static const int CAP   = 16;
static const int MSGSZ = 64;

struct SpmcRing {
    std::atomic<int> wr{0};   // 只生产者推进，直接 store
    std::atomic<int> rd{0};   // 多消费者用 CAS 抢占
    char slot[CAP][MSGSZ];
};

// 单生产者：与 SPSC 相同，直接 store
bool enqueue(SpmcRing* r, const char* msg) {
    int w    = r->wr.load(std::memory_order_relaxed);
    int next = (w + 1) % CAP;
    if (next == r->rd.load(std::memory_order_acquire))
        return false;  // 满
    strncpy(r->slot[w], msg, MSGSZ - 1);
    r->slot[w][MSGSZ - 1] = '\0';
    r->wr.store(next, std::memory_order_release);
    return true;
}

// 多消费者：多个进程同时看到同一个 h，只有一个能 CAS 成功拿到 slot[h]
bool dequeue(SpmcRing* r, char* out) {
    int h, next;
    do {
        h = r->rd.load(std::memory_order_relaxed);
        if (h == r->wr.load(std::memory_order_acquire))
            return false;  // 空
        next = (h + 1) % CAP;
        // 多个消费者并发执行到这里，rd 都是 h
        // CAS 把 rd 从 h 改成 next：只有一个能成功，其余失败重新 load
    } while (!r->rd.compare_exchange_weak(
                 h, next,
                 std::memory_order_acquire,   // 成功：我独占 slot[h]，可以读
                 std::memory_order_relaxed));  // 失败：h 被别人抢走，重试拿新的 h
    // 走到这里：CAS 成功，slot[h] 只属于我，其他消费者会去抢 h+1
    strncpy(out, r->slot[h], MSGSZ);
    return true;
}

int main() {
    void* mem = mmap(NULL, sizeof(SpmcRing),
                     PROT_READ | PROT_WRITE,
                     MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    SpmcRing* ring = new(mem) SpmcRing{};

    // fork 3 个消费者子进程
    for (int i = 0; i < 3; ++i) {
        if (fork() == 0) {
            char out[MSGSZ];
            // 轮询直到消费到一条消息
            while (!dequeue(ring, out))
                ;
            printf("consumer %d got: %s\n", getpid(), out);
            return 0;
        }
    }

    // 父进程：生产者，写 3 条消息
    for (int i = 0; i < 3; ++i) {
        char buf[MSGSZ];
        snprintf(buf, MSGSZ, "msg-%d", i);
        while (!enqueue(ring, buf))  // 满则自旋
            ;
    }
    for (int i = 0; i < 3; ++i) wait(nullptr);

    ring->~SpmcRing();
    munmap(mem, sizeof(SpmcRing));
    return 0;
}
```

## CAS 核心逻辑（消费者）

```
h = rd.load()               // 读当前读游标
h == wr → 空，返回 false
next = (h+1) % CAP
CAS(rd, h, next)            // 只有一个消费者能把 rd 从 h 改成 next
  成功 → 拥有槽位 h，读 slot[h]
  失败 → h 已被别的消费者抢走，重新 load 再试
```

## SPSC vs SPMC 核心区别：rd 游标有没有竞争

| | SPSC | SPMC |
|--|------|------|
| wr 推进 | 生产者直接 store | 生产者直接 store（一样）|
| rd 推进 | 消费者直接 store | **多消费者 CAS 抢** |
| 为什么 SPSC 不用 CAS | rd 只有一个人改，无竞争 | — |
| 为什么 SPMC 要 CAS | 多个消费者同时看到同一个 h，都想推进 rd，必须有且仅有一个抢到 slot[h] | |

## 具体场景

- **SPSC**：消费者看到 `rd=3, wr=5`，直接读 `slot[3]`，store `rd=4`，没人跟它竞争。
- **SPMC**：3 个消费者同时看到 `rd=3, wr=5`，都想读 `slot[3]`——如果都直接 store `rd=4`，slot[3] 会被读 3 次，slot[4] 只被读 1 次，数据乱。CAS 保证只有 1 个消费者能把 rd 从 3 改成 4，其他两个失败后重新 load `rd=4` 再去抢 `slot[4]`。

> **一句话**：生产者只有一个所以 wr 不用 CAS；消费者多了就要 CAS 保证每个槽只被一个人消费。

## 三种模式对比

| 模式 | 生产者 | 消费者 | CAS 位置 |
|------|--------|--------|---------|
| SPSC | store wr | store rd | 无 CAS |
| SPMC（本节） | store wr | **CAS rd** | 消费者抢 rd |
| MPSC | **CAS wr** + ready 标志 | store rd | 生产者抢 wr |

## 相关页面
- [[SPSC无锁队列]]
- [[MPMC环形无锁队列-Vyukov]]
- [[IPC进程间通信]]
- [[C++多线程与并发]]