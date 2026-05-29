# System V IPC

> 本文涵盖 Linux 下 System V IPC 三大机制：信号量、共享内存、消息队列。

See also: [[IPC进程间通信]], [[POSIX信号量]], [[POSIX共享内存]], [[POSIX消息队列]]

## 9.1 System V 信号量

```c
#include <sys/sem.h>

int semget(key_t key, int nsems, int semflg);
int semop(int semid, struct sembuf *sops, size_t nsops);
int semctl(int semid, int semnum, int cmd, ...);
int semtimedop(int semid, struct sembuf *sops, size_t nsops,
              const struct timespec *timeout);    // [2.6]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-九、IPC-进程间通信.md]

## 9.2 System V 共享内存

```c
#include <sys/shm.h>

int shmget(key_t key, size_t size, int shmflg);
void *shmat(int shmid, const void *shmaddr, int shmflg);
int shmdt(const void *shmaddr);
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-九、IPC-进程间通信.md]

## 9.3 System V 消息队列

```c
#include <sys/msg.h>

int msgget(key_t key, int msgflg);
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-九、IPC-进程间通信.md]

## 9.4 POSIX 消息队列

```c
#include <mqueue.h>

mqd_t mq_open(const char *name, int oflag, ...);
int mq_unlink(const char *name);
int mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len, unsigned int msg_prio);
int mq_timedsend(mqd_t mqdes, const char *msg_ptr, size_t msg_len,
                 unsigned int msg_prio, const struct timespec *abs_timeout);
ssize_t mq_receive(mqd_t mqdes, char *msg_ptr, size_t msg_len, unsigned int *msg_prio);
ssize_t mq_timedreceive(mqd_t mqdes, char *msg_ptr, size_t msg_len,
                       unsigned int *msg_prio, const struct timespec *abs_timeout);
int mq_notify(mqd_t mqdes, const struct sigevent *sevp);
int mq_getsetattr(mqd_t mqdes, const struct mq_attr *newattr, struct mq_attr *oldattr);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-九、IPC-进程间通信.md]

## 9.5 内核密钥环

```c
#include <keyutils.h>

key_serial_t add_key(const char *type, const char *description,
                     const void *payload, size_t plen, key_serial_t keyring);
key_serial_t request_key(const char *type, const char *description,
                         const char *callout_info, key_serial_t dest_keyring);
long keyctl(int cmd, ...);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-九、IPC-进程间通信.md]