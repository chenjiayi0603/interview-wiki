# Linux 线程调度

See also: [[C++多线程与并发]], [[C++语言特性]]

## 一、Linux 线程调度策略

> 参考：https://blog.csdn.net/gogo0707/article/details/124977592

Linux 系统调度执行的最小单位是线程。线程的调度策略有以下三种：

三种策略分别面向不同场景：

- `SCHED_FIFO`：实时策略，先到先服务（同优先级下），更适合强实时、低延迟任务
- `SCHED_RR`：实时策略，在 FIFO 基础上加入时间片轮转，避免同优先级线程长期独占 CPU
- `SCHED_OTHER`：普通策略（默认），按动态优先级调度，强调系统整体吞吐与公平性

| 策略 | 类型 | 优先级特点 | 抢占与运行规则 | 典型场景 |
|---|---|---|---|---|
| `SCHED_FIFO` | 实时 | 静态优先级 `1~99` | 高优先级可抢占低优先级；同优先级基本按队列顺序执行，通常不时间片轮转 | 控制环、音视频实时处理 |
| `SCHED_RR` | 实时 | 静态优先级 `1~99` | 与 FIFO 抢占规则一致，但同优先级线程按时间片轮转 | 多个同级实时任务并发 |
| `SCHED_OTHER` | 非实时（默认） | 静态优先级固定为 `0`，再结合 nice 值形成动态优先级 | 调度器综合公平性与响应性分配 CPU | 大多数普通业务线程 |


IM（即时通讯）绝大多数场景用 SCHED_OTHER（默认）就对了。

前台消息收发、UI、网络 I/O：SCHED_OTHER
后台同步、日志、上传下载：SCHED_OTHER + 合理线程池/优先级管理
不建议用 SCHED_FIFO/RR：实时策略容易抢占过度，处理不好会影响系统整体响应，且通常需要更高权限
只有极少数"硬实时"子模块（比如严格低时延音频链路）才可能单独考虑 SCHED_RR/FIFO，并且要非常谨慎控制 CPU 占用与降级策略。

SCHED_FIFO 和 SCHED_RR 主要用于对任务响应时延、调度确定性有高要求的"软/硬实时"应用。常见场景包括：

- 工业控制（如 PLC 控制循环）：要求任务在严格时间窗口内执行，否则可能导致生产事故。
- 低时延音视频处理：如实时语音通话、专业音频工作站、直播间等，需要保证流畅无杂音，调度抖动极低。
- 航空航天、汽车电子控制系统：如自动驾驶、飞控任务的周期性调度。
- 高频交易系统：为缩短订单响应时延，可能将关键线程设为实时策略。
- 机器人控制：驱动控制信号、采集反馈回路需准确按周期执行。

**注意：** 使用 SCHED_FIFO/SCHED_RR 时需谨慎控制线程数量与运行时间，避免因高优先级线程长时间占用 CPU 影响系统正常调度。如一般应用（如 Web 服务器、数据库、普通网络通信等），无需启用实时策略。


直播业务主流程（推流控制、IM、业务逻辑）：通常还是 SCHED_OTHER
直播里的音频实时链路（采集/编码/播放关键线程）：在严格低时延需求下，可能考虑 SCHED_RR/FIFO

### SCHED_FIFO/SCHED_RR 实践代码示例

如下代码演示如何把当前线程调度策略设为 SCHED_FIFO 或 SCHED_RR，并设置优先级。需要 root 权限运行。

```cpp
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <unistd.h>

void* thread_func(void* arg) {
    int policy;
    struct sched_param param;
    pthread_getschedparam(pthread_self(), &policy, &param);

    if(policy == SCHED_FIFO) {
        printf("当前策略: SCHED_FIFO, 优先级: %d\n", param.sched_priority);
    } else if(policy == SCHED_RR) {
        printf("当前策略: SCHED_RR, 优先级: %d\n", param.sched_priority);
    } else {
        printf("当前策略: SCHED_OTHER\n");
    }

    // 占用 CPU，观察调度效果
    for(int i=0; i<5; ++i) {
        printf("线程运行: %d\n", i);
        sleep(1);
    }
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_attr_t attr;
    struct sched_param param;

    pthread_attr_init(&attr);

    // 切换 SCHED_FIFO / SCHED_RR 测试
    int policy = SCHED_FIFO;
    // int policy = SCHED_RR;
    pthread_attr_setschedpolicy(&attr, policy);

    // 设置优先级（SCHED_FIFO/RR 为 1~99，数值越大优先级越高）
    param.sched_priority = 50;
    pthread_attr_setschedparam(&attr, &param);
    pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

    pthread_create(&tid, &attr, thread_func, NULL);
    pthread_join(tid, NULL);

    pthread_attr_destroy(&attr);
    return 0;
}
```

**说明：**
- 需要用 `sudo` 或 root 权限运行，否则设置 SCHED_FIFO/SCHED_RR 以及较高优先级会失败。
- 可以通过调整 `policy`（SCHED_FIFO/SCHED_RR）以及 `sched_priority` 体验不同实时策略下线程的竞争调度效果。
- 实际项目中慎用实时策略！要防止线程因高优先级导致系统响应卡死等风险。

---


##### 1. SCHED_FIFO（静态优先级队列）

- 静态优先级必须设置为 **1~99**
- 一旦线程处于就绪态，就能立即抢占任何静态优先级为 0 的普通线程
- 规则：
  - 处于就绪态时，放入其所在优先级队列的**队尾**
  - 被更高优先级线程抢占后，放入其所在优先级队列的**队头**；当所有更高优先级线程不再运行后恢复
  - 调用 `sched_yield()` 后，放入其所在优先级队列的**队尾**
- 总结：会一直运行直到发生 I/O 请求、被更高优先级线程抢占，或主动调用 `sched_yield()` 让出 CPU

##### 2. SCHED_RR（时间片轮转）

- 与 SCHED_FIFO 类似，区别是每个 SCHED_RR 线程都会被分配一个**时间片**
- 时间片耗尽时，会被放入其所在优先级队列的队尾
- 可用 `sched_rr_get_interval()` 获取时间片数值

// 示例：创建两个不同优先级、不同策略的线程演示调度效果
```cpp
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <sched.h>

void* thread_func_high(void* arg) {
    int cnt = 0;
    while (cnt < 5) {
        printf("High priority thread running (cnt=%d)\n", cnt++);
        usleep(200 * 1000);
    }
    return NULL;
}

void* thread_func_low(void* arg) {
    int cnt = 0;
    while (cnt < 5) {
        printf("Low priority thread running (cnt=%d)\n", cnt++);
        usleep(200 * 1000);
    }
    return NULL;
}

int main() {
    pthread_t tid_high, tid_low;
    pthread_attr_t attr_high, attr_low;
    struct sched_param param_high, param_low;

    pthread_attr_init(&attr_high);
    pthread_attr_init(&attr_low);

    // 设置高优先级 SCHED_FIFO 线程
    pthread_attr_setschedpolicy(&attr_high, SCHED_FIFO);
    param_high.sched_priority = 80; // 需用 sudo/root
    pthread_attr_setschedparam(&attr_high, &param_high);
    pthread_attr_setinheritsched(&attr_high, PTHREAD_EXPLICIT_SCHED);

    // 设置低优先级 SCHED_RR 线程
    pthread_attr_setschedpolicy(&attr_low, SCHED_RR);
    param_low.sched_priority = 30;
    pthread_attr_setschedparam(&attr_low, &param_low);
    // 设置线程属性，让新线程显式使用我们设置的调度策略和优先级
    pthread_attr_setinheritsched(&attr_low, PTHREAD_EXPLICIT_SCHED);

    pthread_create(&tid_high, &attr_high, thread_func_high, NULL);
    pthread_create(&tid_low, &attr_low, thread_func_low, NULL);

    pthread_join(tid_high, NULL);
    pthread_join(tid_low, NULL);

    pthread_attr_destroy(&attr_high);
    pthread_attr_destroy(&attr_low);

    return 0;
}
```


/*
编译：
    gcc -o sched_demo sched_demo.c -lpthread
运行（需 root 权限）：
    sudo ./sched_demo

可修改调度策略和优先级进行深入体验。
*/

##### 3. SCHED_OTHER（动态优先级队列）

- 静态优先级必须设置为 **0**，是 Linux 的默认调度策略
- 按**动态优先级**（nice 值）调度
- 当线程已就绪但未被调度时，动态优先级会自动增加，以保证竞争公平性

---

### 静态优先级与动态优先级

| 类型     | 范围      | 说明                                                                 |
|----------|-----------|----------------------------------------------------------------------|
| 非实时   | 静态优先 0 | 普通线程                                                             |
| 实时     | 静态优先 1~99 | 高优先级线程                                                         |
| 静态优先级 |           | 不随执行改变，决定实时线程的基本调度次序                             |
| 动态优先级 |           | 仅用于非实时线程，会随运行表现改变                                   |

**动态优先级机制：**

- **CPU 消耗型**（如视频解码）：长时间占用 CPU，动态优先级会**降低**
- **I/O 消耗型**（如编辑器）：大部分时间睡眠，动态优先级会**提高**，获得更好响应

---

### SCHED_FIFO 与 SCHED_RR 的区别与选择

#### 区别

- **SCHED_FIFO（先进先出）**
  - 线程被分配静态优先级（1~99），一旦获得 CPU 就会一直运行，直到主动让出（sleep/yield/block），或者被更高优先级线程抢占。
  - 同优先级线程按照加入队列的顺序调度，不轮转时间片，即某线程只要不阻塞、无需被剥夺，就可独占 CPU。
  - 适用于对时延极端敏感的周期性、连续计算任务。

- **SCHED_RR（时间片轮转）**
  - 同样分配静态优先级（1~99）。
  - 与 SCHED_FIFO 原则相同，但同优先级线程会按照固定时间片（time slice）轮流运行，不会被单个线程长期独占 CPU。
  - 适合有多个高优先级任务并发、希望公平轮转、不希望单线程饿死其他线程的场景。

#### 选择建议

- **SCHED_FIFO**
  - 适用于**优先级绝对分明、单任务独占型、对延迟极度敏感**的场合；
  - 适合线程短时、高频率响应、且可以明确控制运行时长的循环调度，如**机器人控制主循环、工业 PLC**。

- **SCHED_RR**
  - 适于**多个实时任务需要公平处理，也很关注响应时延，但希望防止饿死同优先级线程**；
  - 常见于**高频交易系统核心撮合/风控线程、音频/视频流处理、周期性任务切换**等。

#### 举例

- **高频交易系统**
    - 场景：订单撮合、风控核心逻辑线程，需要尽可能低的时延响应，有时还需多个线程并发。
    - 用法：可以将关键撮合线程设为 SCHED_RR，优先级设高。若有极致低延迟线程只有一个，也可用 SCHED_FIFO。
    - 例子：
        - 多个风控线程 → SCHED_RR（优先级高，防止互相饿死）
        - 单一行情处理线程 → SCHED_FIFO（绝对优先）

- **机器人控制**
    - 场景：电机控制、实时传感器采集等，很多为硬件控制循环，需确保每个控制周期内必定执行操作，不容许延误。
    - 用法：主实时控制线程通常采用 SCHED_FIFO（确保无其他同级打断），传感器采集、辅助诊断可用 SCHED_RR 保证定期轮转。
    - 例子：
        - 主运动控制环（动力驱动）→ SCHED_FIFO（优先处理，强实时）
        - 传感器采集线程、监控线程 → SCHED_RR（均需周期轮转但可适度让步）

> 总结：  
> - 对"只有一个独占线程"的严格实时环，SCHED_FIFO 最可靠。
> - 对"多个同等级别任务均需有响应机会"的实时系统，SCHED_RR 更公平、安全。


## 二、Linux 进程/线程绑定到 CPU

> 参考：https://blog.csdn.net/zz460833359/article/details/122145838  
> 参考：https://blog.51cto.com/u_12870633/5077578

将进程或线程绑定到特定 CPU 核，可减少调度开销、保护关键进程/线程。

### 1. 进程绑定到 CPU

```c
#include <sched.h>

int sched_setaffinity(pid_t pid, size_t cpusetsize, const cpu_set_t *mask);
int sched_getaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);
```

**参数：**

- `pid`：进程 ID，为 0 表示当前进程
- `cpusetsize`：mask 大小
- `mask`：运行该进程的 CPU 集合

**mask 操作宏：**

```c
#define CPU_SET(cpu, cpusetp)    // 设置 cpu
#define CPU_CLR(cpu, cpusetp)    // 删除 cpu
#define CPU_ISSET(cpu, cpusetp)  // 判断 cpu
#define CPU_ZERO(cpusetp)        // 初始化为空
```

### 2. 线程绑定到 CPU

```c
#include <pthread.h>

int pthread_setaffinity_np(pthread_t thread, size_t cpusetsize, const cpu_set_t *cpuset);
int pthread_getaffinity_np(pthread_t thread, size_t cpusetsize, cpu_set_t *cpuset);
```

用法与进程接口类似。进程绑定后，线程仍可绑定到其他 CPU，互不冲突。

### 3. C++ 实现要点

- **必须使用 `pthread.h`**：`std::thread` 无法做 CPU 核绑定
- **获取 CPU 核数**：`sysconf(_SC_NPROCESSORS_CONF)`
- 线程内可用 `sched_setaffinity(0, ...)` 绑定当前线程（pid=0 表示当前进程/主线程）

### 4. 示例：进程在多个 CPU 间切换

```c
#include <stdio.h>
#include <unistd.h>
#include <sched.h>

void WasteTime() {
    int abc = 10000000;
    while (abc--) {
        int tmp = 10000 * 10000;
    }
    sleep(1);
}

int main(int argc, char **argv) {
    cpu_set_t mask;
    while (1) {
        CPU_ZERO(&mask);
        CPU_SET(0, &mask);
        if (sched_setaffinity(0, sizeof(mask), &mask) < 0)
            perror("sched_setaffinity");
        WasteTime();

        CPU_ZERO(&mask);
        CPU_SET(1, &mask);
        if (sched_setaffinity(0, sizeof(mask), &mask) < 0)
            perror("sched_setaffinity");
        WasteTime();
        // ... 可继续绑定 CPU 2、3 等
    }
}
```

### 5. 示例：线程分别绑定不同 CPU

```c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sched.h>

void *thread_func(void *param) {
    cpu_set_t mask;
    while (1) {
        CPU_ZERO(&mask);
        CPU_SET(1, &mask);
        if (pthread_setaffinity_np(pthread_self(), sizeof(mask), &mask) < 0)
            perror("pthread_setaffinity_np");
        // ... 工作 ...

        CPU_ZERO(&mask);
        CPU_SET(2, &mask);
        if (pthread_setaffinity_np(pthread_self(), sizeof(mask), &mask) < 0)
            perror("pthread_setaffinity_np");
        // ... 工作 ...
    }
    return NULL;
}

int main(int argc, char *argv[]) {
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(0, &mask);  // 主线程绑定 CPU 0
    sched_setaffinity(0, sizeof(mask), &mask);

    pthread_t th;
    pthread_create(&th, NULL, thread_func, NULL);
    // ...
}
```

### 6. 验证方式

```bash
top -p <进程id>   # 按 f -> 找到 P -> d 或空格勾选 -> 回车 查看 CPU affinity
top -d 2          # 每 2 秒刷新，查看线程和 CPU 状态
# 按 h、1 查看更详细的每核使用情况
```

**操作步骤：**

1. `top -p <pid>`：进入后按 `f` → 用方向键找到 P (Last used CPU) → 按 `d` 或空格勾选 → 回车或 `q` 退出
2. 按 `H`（大写）：切换为线程视图，可看到各线程的 P 列（所在核号）
3. 按 `1`：显示每核 CPU 使用率

**示例输出：**

进程视图（P 列表示当前/最后使用的 CPU 核）：
```
  PID USER      PR  NI    VIRT    RES  SHR S  %CPU  %MEM     P  TIME+ COMMAND
12345 user      20   0   12345   1024  512 R  99.0   0.1     2  00:15.22 my_prog
```

线程视图（多线程分别显示，P 列可验证各线程绑定核）：
```
  PID USER      PR  NI    VIRT    RES  SHR S  %CPU  %MEM     P  TIME+ COMMAND
12345 user      20   0  123456   2048 1024 R  50.0   0.1     0  00:05.10 my_prog
12346 user      20   0  123456   2048 1024 R  50.0   0.1     1  00:05.08 my_prog
12347 user      20   0  123456   2048 1024 R  49.0   0.1     2  00:05.12 my_prog
```

**说明**：线程视图中「PID」列实际显示的是 **TID（线程 ID）**。Linux 下主线程的 TID 等于进程 PID（12345）；子线程有独立 TID（12346、12347）。可通过 `ls /proc/<pid>/task/` 查看该进程下所有线程的 TID。

按 `1` 后的每核负载（高负载核对应绑定的线程）：
```
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni, 100.0 id
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni, 100.0 id
%Cpu2  : 98.0 us,  1.0 sy,  0.0 ni,   1.0 id
%Cpu3  :  0.0 us,  0.0 sy,  0.0 ni, 100.0 id
```

---

## 三、线程状态

> 参考：https://zhuanlan.zhihu.com/p/175943809

### 线程生命周期中的四种状态

| 状态   | 说明                                                                 |
|--------|----------------------------------------------------------------------|
| **准备** | 等待可用 CPU，其余条件已就绪。创建或从阻塞中唤醒后进入               |
| **运行** | 已获得 CPU 并执行，多核下可有多个线程同时运行                       |
| **阻塞** | 暂停等待某个条件，例如：I/O、条件变量、对已锁互斥量加锁、sigwait 等 |
| **终止** | 从线程函数返回、`pthread_exit` 或被强制终止                         |

### 示例：线程创建与退出

```c
int main(int argc, char *argv[]) {
    pthread_t tid;
    int err;
    Person s;
    memcpy(s.name, "Peter", 15);
    s.age = 25;
    err = pthread_create(&tid, NULL, say_hello, (void *)&s);
    if (err != 0) {
        printf("创建线程失败\n");
        return 0;
    }
    printf("主线程运行中!!\n");
    int *retval;
    pthread_exit(retval);
}
```

---

## 四、进程状态

> 参考：https://blog.csdn.net/Zx13170918986/article/details/125768407

### Linux 进程状态

| 状态                       | 说明                                                                 |
|----------------------------|----------------------------------------------------------------------|
| **TASK_RUNNING**           | 正在运行或在运行队列中等待调度                                       |
| **TASK_INTERRUPTIBLE**     | 阻塞（睡眠），等待事件或资源；可被信号或 `wake_up` 唤醒               |
| **TASK_UNINTERRUPTIBLE**   | 不可中断阻塞，不处理信号；仅在等待事件发生时被显式唤醒                |
| **TASK_KILLABLE**          | 类似不可中断，但可响应致命信号（Linux 2.6.25+）                      |
| **TASK_STOPPED**           | 收到 SIGSTOP、SIGTSTP、SIGTTIN、SIGTTOU 等信号后暂停                 |
| **TASK_TRACED**            | 被调试器暂停，如 ptrace 监控                                         |
| **EXIT_ZOMBIE**            | 已结束，父进程尚未 wait 回收                                         |
| **EXIT_DEAD**              | 父进程 wait 后，进程被系统彻底删除                                   |

### 如何查看进程状态

**ps / top 的 STAT 列：**

| 字符 | 含义           | 对应内核状态           |
|------|----------------|------------------------|
| R    | 运行/可运行    | TASK_RUNNING           |
| S    | 可中断睡眠     | TASK_INTERRUPTIBLE     |
| D    | 不可中断睡眠   | TASK_UNINTERRUPTIBLE   |
| T    | 停止           | TASK_STOPPED           |
| t    | 调试停止       | TASK_TRACED            |
| Z    | 僵尸           | EXIT_ZOMBIE            |

**常用命令：**

```bash
ps -eo pid,state,comm              # 查看所有进程状态
ps -p <pid> -o pid,state,cmd       # 查看指定进程
cat /proc/<pid>/status | grep State  # 从 proc 读取
top                               # S 列同 STAT
```

**举例验证：**

| 操作 | 预期状态 | 验证 |
|------|----------|------|
| `sleep 60 &` | S | `ps -p <pid> -o state` → S |
| `kill -STOP <pid>` | T | `ps -p <pid> -o state` → T |
| 子 exit 父不 wait | Z | `ps aux \| grep 'Z'` 看到 defunct |
| 磁盘/NFS 故障 | D | `ps -eo pid,state,cmd \| grep ' D'` |

### 进程状态示例代码

**TASK_RUNNING**：运行或就绪
```c
while (1) { volatile long x = 0; for (int i = 0; i < 1000000; i++) x++; }  // 纯 CPU 运算
```

**TASK_INTERRUPTIBLE**：可被信号唤醒的睡眠
```c
sleep(3600);                    // 睡眠，可用 kill 或 Ctrl+C 唤醒
read(fd, buf, size);            // 阻塞读，信号可中断
pthread_cond_wait(&cond, &m);   // 条件变量等待
```

**TASK_STOPPED**：收到 SIGSTOP 后暂停
```bash
# 终端 1：运行程序
./my_prog
# 终端 2：发送 SIGSTOP
kill -STOP <pid>
# 此时 ps 显示 T（stopped），恢复用 kill -CONT <pid>
```

**EXIT_ZOMBIE**：子进程退出但父进程未 wait
```c
// 父进程
if (fork() == 0) { exit(0); }   // 子进程退出，变为僵尸
sleep(60);                      // 父进程不调用 wait()，子进程保持 Z 状态
```

**EXIT_DEAD**：父进程 wait 后子进程彻底消失
```c
if (fork() == 0) { exit(0); }
wait(NULL);  // 父进程回收后，子进程从 Z 变为彻底删除
```

**说明**：TASK_UNINTERRUPTIBLE 通常由内核在 I/O 等待（如磁盘/网络故障）时设置，用户态难以精确构造；TASK_TRACED 在 gdb attach 时产生；TASK_KILLABLE 多为内核内部使用。

### D 状态（TASK_UNINTERRUPTIBLE）原理

1. 内核周期性收集进程资源请求，将发起请求的进程先放入 parking 队列。
2. 收集结束后，将可运行进程放入 runnable 队列等待调度。
3. **D 状态**出现在"获取资源"阶段：内核为进程拿资源（如读磁盘）时，若驱动无响应或故障，无法立刻拿到数据，则将进程置为 D 状态。
4. D 状态进程**无法被 kill**，只能等待资源就绪或重启系统。

**说明**：内核可用 `set_task_state`、`set_current_state` 修改进程状态。

[src: raw/ingested/2技术/cpp/Linux c线程调度.md]

## Related Pages
- [[C++多线程与并发]]
- [[C++语言特性]]
- [[线程同步]]
- [[进程同步]]
- [[Linux_IO]]
- [[性能优化]]
