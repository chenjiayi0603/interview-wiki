# membarrier — 内存屏障

See also: [[原子操作与内存模型]], [[C++多线程与并发]]

## API 说明

```c
#include <linux/membarrier.h>
#include <sys/syscall.h>

int membarrier(int cmd, unsigned int flags, int cpu_id);
// cmd:
//   MEMBARRIER_CMD_QUERY
//   MEMBARRIER_CMD_GLOBAL                  — 全局屏障
//   MEMBARRIER_CMD_GLOBAL_EXPEDITED        — 加速全局屏障
//   MEMBARRIER_CMD_REGISTER_GLOBAL_EXPEDITED
//   MEMBARRIER_CMD_PRIVATE_EXPEDITED       — 进程内加速屏障
//   MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED
//   MEMBARRIER_CMD_PRIVATE_EXPEDITED_RSEQ  — Linux 5.10+
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十七、同步与-CPU-调度.md]