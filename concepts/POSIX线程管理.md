# POSIX 线程管理 (pthread)

See also: [[C++多线程与并发]], [[Linux线程调度]], [[POSIX信号量]]

## 一、线程创建和终止

```c
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                  void *(*start_routine)(void *), void *arg);  // [POSIX]
void pthread_exit(void *retval);                                // [POSIX]
int pthread_join(pthread_t thread, void **retval);              // [POSIX]
int pthread_detach(pthread_t thread);                           // [POSIX]
pthread_t pthread_self(void);                                   // [POSIX]
int pthread_equal(pthread_t t1, pthread_t t2);                  // [POSIX]
int pthread_cancel(pthread_t thread);                           // [POSIX]
int pthread_once(pthread_once_t *once_control, void (*init_routine)(void)); // [POSIX]
```

### 核心函数说明

- **`pthread_create`**：创建一个新线程，执行 `start_routine` 函数，传入 `arg` 参数。可通过 `attr` 设置线程属性（如分离状态、调度策略等），传 `NULL` 使用默认属性。
- **`pthread_exit`**：终止调用线程，返回 `retval` 给等待该线程的其他线程。
- **`pthread_join`**：等待指定线程终止，获取其返回值。类似于进程的 `waitpid`。
- **`pthread_detach`**：将线程设置为分离状态，线程终止时资源自动回收，无需其他线程 `join`。
- **`pthread_self`**：获取当前线程的线程 ID。
- **`pthread_equal`**：比较两个线程 ID 是否相等。
- **`pthread_cancel`**：向指定线程发送取消请求。
- **`pthread_once`**：保证 `init_routine` 在整个进程生命周期中仅执行一次，常用于初始化全局资源。

## 二、线程属性

```c
int pthread_attr_init(pthread_attr_t *attr);                                   // [POSIX]
int pthread_attr_destroy(pthread_attr_t *attr);                                // [POSIX]
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);        // [POSIX]
int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate); // [POSIX]
int pthread_attr_setscope(pthread_attr_t *attr, int contentionscope);          // [POSIX]
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);         // [POSIX]
int pthread_attr_getstacksize(const pthread_attr_t *attr, size_t *stacksize);  // [POSIX]
int pthread_attr_setstack(pthread_attr_t *attr, void *stackaddr, size_t stacksize); // [POSIX]
```

### 核心属性说明

- **分离状态（detachstate）**：`PTHREAD_CREATE_JOINABLE`（默认，需要 `pthread_join` 回收）或 `PTHREAD_CREATE_DETACHED`（自动回收）。
- **竞争范围（scope）**：`PTHREAD_SCOPE_SYSTEM`（与系统级线程竞争 CPU）或 `PTHREAD_SCOPE_PROCESS`（进程内竞争）。
- **栈大小（stacksize）**：设置线程栈的最小大小（字节）。
- **栈地址（stack）**：同时设置栈的起始地址和大小，用于精确控制线程栈内存位置。

## 三、互斥锁 (Mutex)

```c
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr); // [POSIX]
int pthread_mutex_destroy(pthread_mutex_t *mutex);     // [POSIX]
int pthread_mutex_lock(pthread_mutex_t *mutex);        // [POSIX]
int pthread_mutex_unlock(pthread_mutex_t *mutex);      // [POSIX]
int pthread_mutex_trylock(pthread_mutex_t *mutex);     // [POSIX]
int pthread_mutex_timedlock(pthread_mutex_t *mutex,
                            const struct timespec *abstime); // [POSIX]

int pthread_mutexattr_init(pthread_mutexattr_t *attr);       // [POSIX]
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr);    // [POSIX]
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type); // [POSIX]
```

### 互斥锁类型

通过 `pthread_mutexattr_settype` 设置：
- **`PTHREAD_MUTEX_NORMAL`**：普通锁，不检测死锁，重复加锁导致死锁。
- **`PTHREAD_MUTEX_ERRORCHECK`**：错误检查锁，重复加锁返回错误。
- **`PTHREAD_MUTEX_RECURSIVE`**：递归锁，同一线程可多次加锁，需对应次数解锁。
- **`PTHREAD_MUTEX_DEFAULT`**：默认类型，行为由实现定义。

## 四、条件变量 (Condition Variable)

```c
int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *attr); // [POSIX]
int pthread_cond_destroy(pthread_cond_t *cond);      // [POSIX]
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);    // [POSIX]
int pthread_cond_signal(pthread_cond_t *cond);       // [POSIX]
int pthread_cond_broadcast(pthread_cond_t *cond);    // [POSIX]
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex,
                           const struct timespec *abstime); // [POSIX]
```

### 使用要点

- **`pthread_cond_wait`**：原子地释放 `mutex` 并阻塞等待条件变量；被唤醒后重新获取 `mutex` 再返回。必须与 `while` 循环配合检查条件，防止虚假唤醒。
- **`pthread_cond_signal`**：唤醒至少一个等待该条件变量的线程。
- **`pthread_cond_broadcast`**：唤醒所有等待该条件变量的线程。
- **`pthread_cond_timedwait`**：带超时的条件等待，超时返回 `ETIMEDOUT`。

## 五、读写锁 (Read-Write Lock)

```c
int pthread_rwlock_init(pthread_rwlock_t *rwlock, const pthread_rwlockattr_t *attr); // [POSIX]
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);   // [POSIX]
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);    // [POSIX]
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);    // [POSIX]
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);    // [POSIX]
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock); // [POSIX]
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock); // [POSIX]
int pthread_rwlock_timedrdlock(pthread_rwlock_t *rwlock,
                               const struct timespec *abstime); // [POSIX]
int pthread_rwlock_timedwrlock(pthread_rwlock_t *rwlock,
                               const struct timespec *abstime); // [POSIX]
```

### 读写锁规则

- **读锁（rdlock）**：多个线程可同时持有读锁（共享）。
- **写锁（wrlock）**：独占锁，写锁持有期间不允许其他读锁或写锁。
- **写优先 vs 读优先**：实现相关，通常写锁请求会阻塞后续读锁请求，防止写者饥饿。
- **try 系列**：非阻塞版本，无法获取锁时立即返回 `EBUSY`。
- **timed 系列**：带超时的加锁，超时返回 `ETIMEDOUT`。

## 六、自旋锁 (Spin Lock)

```c
int pthread_spin_init(pthread_spinlock_t *lock, int pshared);  // [POSIX]
int pthread_spin_destroy(pthread_spinlock_t *lock);            // [POSIX]
int pthread_spin_lock(pthread_spinlock_t *lock);               // [POSIX]
int pthread_spin_trylock(pthread_spinlock_t *lock);            // [POSIX]
int pthread_spin_unlock(pthread_spinlock_t *lock);             // [POSIX]
```

### 自旋锁特点

- 与互斥锁不同，自旋锁在等待时**忙等（busy-wait）**，不会让出 CPU。
- 适用于**临界区极短**的场景（如仅更新几个变量），避免上下文切换开销。
- **`pshared`** 参数：`PTHREAD_PROCESS_PRIVATE`（进程内共享）或 `PTHREAD_PROCESS_SHARED`（进程间共享，需配合共享内存）。
- 不适合临界区较长的场景，会浪费 CPU 资源。

## 七、线程局部存储 (TLS)

```c
int pthread_key_create(pthread_key_t *key, void (*destructor)(void *)); // [POSIX]
int pthread_key_delete(pthread_key_t key);            // [POSIX]
int pthread_setspecific(pthread_key_t key, const void *value); // [POSIX]
void *pthread_getspecific(pthread_key_t key);         // [POSIX]
```

### TLS 使用模式

- **`pthread_key_create`**：创建一个线程局部存储键，可指定 `destructor` 函数在线程退出时自动清理该键对应的值。
- **`pthread_setspecific`**：为当前线程设置键对应的值。
- **`pthread_getspecific`**：获取当前线程键对应的值。
- **`pthread_key_delete`**：删除键（不调用 destructor）。
- 与 C++11 `thread_local` 关键字相比，POSIX TLS 更底层，需手动管理生命周期。

## 八、线程屏障 (Barrier)

```c
int pthread_barrier_init(pthread_barrier_t *barrier,
                        const pthread_barrierattr_t *attr, unsigned int count); // [POSIX]
int pthread_barrier_destroy(pthread_barrier_t *barrier);  // [POSIX]
int pthread_barrier_wait(pthread_barrier_t *barrier);     // [POSIX]
```

### 屏障使用场景

- **`count`** 参数指定需要等待的线程数量。
- **`pthread_barrier_wait`**：调用线程阻塞，直到 `count` 个线程都调用了 `wait`，然后所有线程同时被释放继续执行。
- 适用于**分阶段并行计算**：每个阶段完成后所有线程在屏障处同步，再进入下一阶段。
- 屏障可重复使用：所有线程被释放后，屏障自动重置，可再次使用。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-4.-线程管理-(pthread).md]

## Related Pages
- [[C++多线程与并发]]
- [[Linux线程调度]]
- [[POSIX信号量]]
- [[POSIX进程控制]]
- [[C++手写代码模板]]
