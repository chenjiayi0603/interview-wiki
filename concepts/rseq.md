# rseq — 可重启序列

See also: [[原子操作与内存模型]], [[C++多线程与并发]]

## API 说明

```c
#include <linux/rseq.h>

int rseq(struct rseq *rseq, uint32_t rseq_len, int flags, uint32_t sig);
// 用于用户空间与内核协作的无锁算法
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十七、同步与-CPU-调度.md]