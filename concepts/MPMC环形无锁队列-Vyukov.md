# MPMC 环形无锁队列（Vyukov）

See also: [[C++多线程与并发]], [[C++手写代码模板]], [[C++内存模型]]

## 核心原理

每个槽位增加序号（seq）辅助判断可写/可读状态，push/pop 都用 CAS 循环尝试。

### 槽位状态
- 可写入：`seq == pos`
- 可读取：`seq == pos + 1`
- 消费后下一轮可写：`seq == pos + Size`（防 ABA）

### seq 与 pos 关系
| 条件 | 状态 |
|------|------|
| `seq == pos` | 可写 |
| `seq == pos + 1` | 可读 |
| `seq == pos + Size` | 消费后下一轮可写 |

## 完整实现

```cpp
#include <atomic>
#include <cstddef>
#include <array>

template<typename T, size_t Size>
class MPMCQueue {
    static_assert((Size & (Size - 1)) == 0, "Size must be a power of 2");
    struct Cell {
        std::atomic<size_t> seq;
        T data;
    };
    std::array<Cell, Size> buffer;
    std::atomic<size_t> head{0};
    std::atomic<size_t> tail{0};
public:
    MPMCQueue() {
        for (size_t i = 0; i < Size; ++i) {
            buffer[i].seq.store(i, std::memory_order_relaxed);
        }
    }

    bool push(const T& data) {
        size_t pos = tail.load(std::memory_order_relaxed);
        for (;;) {
            Cell& cell = buffer[pos & (Size - 1)];
            size_t seq = cell.seq.load(std::memory_order_acquire);
            intptr_t dif = (intptr_t)seq - (intptr_t)pos;  // dif=seq-pos
            if (dif == 0) {       // 可写
                if (tail.compare_exchange_weak(pos, pos + 1, std::memory_order_acq_rel, std::memory_order_acquire)) {
                    cell.data = data;
                    cell.seq.store(pos + 1, std::memory_order_release);  // 写后 seq=pos+1
                    return true;
                }
            } else if (dif < 0) {  // 队列满
                return false;
            } else {               // CAS 竞争失败，重试
                pos = tail.load(std::memory_order_acquire);
            }
        }
    }

    bool pop(T& data) {
        size_t pos = head.load(std::memory_order_acquire);
        for (;;) {
            Cell& cell = buffer[pos & (Size - 1)];
            size_t seq = cell.seq.load(std::memory_order_acquire);
            intptr_t dif = (intptr_t)seq - (intptr_t)(pos + 1);  // dif=seq-(pos+1)
            if (dif == 0) {       // 可读
                if (head.compare_exchange_weak(pos, pos + 1, std::memory_order_acq_rel, std::memory_order_acquire)) {
                    data = cell.data;
                    cell.seq.store(pos + Size, std::memory_order_release);  // +Size 防 ABA，跳到下一轮
                    return true;
                }
            } else if (dif < 0) {  // 队列空
                return false;
            } else {                // 槽已被其他消费者处理，重试
                pos = head.load(std::memory_order_acquire);
            }
        }
    }
};
```

## 使用示例

```cpp
// 2 生产者 2 消费者
MPMCQueue<int, 256> queue;
// push/pop 返回 false 表示满/空，需重试或 yield
```

## 参考
- https://www.1024cores.net/articles/queues/bounded-mpmc-queue

[src: raw/ingested/2技术/cpp/C++多线程完整手册-11.1-MPMC-环形无锁队列（Vyukov）.md]

## Related Pages
- [[C++多线程与并发]]
- [[C++手写代码模板]]
- [[C++内存模型]]
