# futex — 快速用户空间互斥

See also: [[C++多线程与并发]], [[Linux线程调度]], [[线程同步机制]]

## API 说明

```c
#include <linux/futex.h>
#include <sys/syscall.h>

long futex(uint32_t *uaddr, int futex_op, uint32_t val,
           const struct timespec *timeout, uint32_t *uaddr2, uint32_t val3);

// futex_op:
//   FUTEX_WAIT     — 如果*uaddr==val则休眠
//   FUTEX_WAKE     — 唤醒最多val个等待者
//   FUTEX_WAIT_BITSET / FUTEX_WAKE_BITSET
//   FUTEX_REQUEUE  / FUTEX_CMP_REQUEUE
//   FUTEX_LOCK_PI  / FUTEX_UNLOCK_PI  — 优先级继承锁
```

## 示例：基于futex的简单锁

```c
#define _GNU_SOURCE
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <stdint.h>
#include <stdatomic.h>
#include <stdio.h>
#include <pthread.h>

static atomic_uint futex_val = 0;

static long futex_wait(uint32_t *addr, uint32_t expected) {
    return syscall(SYS_futex, addr, FUTEX_WAIT, expected, NULL, NULL, 0);
}

static long futex_wake(uint32_t *addr, uint32_t count) {
    return syscall(SYS_futex, addr, FUTEX_WAKE, count, NULL, NULL, 0);
}

void futex_lock() {
    while (atomic_exchange(&futex_val, 1) != 0) {
        futex_wait((uint32_t *)&futex_val, 1);
    }
}

void futex_unlock() {
    atomic_store(&futex_val, 0);
    futex_wake((uint32_t *)&futex_val, 1);
}

int shared_counter = 0;

void *futex_worker(void *arg) {
    for (int i = 0; i < 100000; i++) {
        futex_lock();
        shared_counter++;
        futex_unlock();
    }
    return NULL;
}

void futex_example() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, futex_worker, NULL);
    pthread_create(&t2, NULL, futex_worker, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Futex counter: %d (expected 200000)\n", shared_counter);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十七、同步与-CPU-调度.md]