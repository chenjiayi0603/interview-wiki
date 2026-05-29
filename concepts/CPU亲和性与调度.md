# CPU 亲和性与调度 (sched)

See also: [[Linux线程调度]], [[C++多线程与并发]], [[futex]]

## API 说明

```c
#define _GNU_SOURCE
#include <sched.h>

int sched_setaffinity(pid_t pid, size_t cpusetsize,
                      const cpu_set_t *mask);    // [Linux]
int sched_getaffinity(pid_t pid, size_t cpusetsize,
                      cpu_set_t *mask);          // [Linux]

int sched_setscheduler(pid_t pid, int policy,
                       const struct sched_param *param);  // [POSIX]
int sched_getscheduler(pid_t pid);               // [POSIX]
int sched_setparam(pid_t pid,
                   const struct sched_param *param);      // [POSIX]
int sched_getparam(pid_t pid, struct sched_param *param); // [POSIX]
int sched_get_priority_max(int policy);          // [POSIX]
int sched_get_priority_min(int policy);           // [POSIX]
int sched_yield(void);                           // [POSIX]
int sched_rr_get_interval(pid_t pid, struct timespec *tp); // [POSIX]

// policy: SCHED_OTHER [POSIX] | SCHED_FIFO [POSIX] | SCHED_RR [POSIX]
//         SCHED_BATCH [Linux] | SCHED_IDLE [Linux] | SCHED_DEADLINE [Linux]

// CPU集合操作宏                                   // 全部 [GNU/Linux]
void CPU_ZERO(cpu_set_t *set);
void CPU_SET(int cpu, cpu_set_t *set);
void CPU_CLR(int cpu, cpu_set_t *set);
int CPU_ISSET(int cpu, cpu_set_t *set);
int CPU_COUNT(cpu_set_t *set);
```

## 示例：设置CPU亲和性

```c
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>

void cpu_affinity_example() {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);

    // 获取当前CPU亲和性
    sched_getaffinity(0, sizeof(cpuset), &cpuset);
    int num_cpus = sysconf(_SC_NPROCESSORS_ONLN);
    printf("Available CPUs: %d\n", num_cpus);

    for (int i = 0; i < num_cpus; i++) {
        if (CPU_ISSET(i, &cpuset))
            printf("  CPU %d: available\n", i);
    }

    // 绑定到CPU 0
    CPU_ZERO(&cpuset);
    CPU_SET(0, &cpuset);
    if (sched_setaffinity(0, sizeof(cpuset), &cpuset) == 0) {
        printf("Bound to CPU 0\n");
    }

    // 线程绑定
    pthread_t self = pthread_self();
    pthread_setaffinity_np(self, sizeof(cpuset), &cpuset);
    printf("Thread bound to CPU 0\n");

    sched_yield(); // 主动让出CPU
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十七、同步与-CPU-调度.md]