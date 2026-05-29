# userfaultfd — 用户空间缺页处理

> `userfaultfd` 是 Linux 4.3+ 引入的系统调用，允许用户空间程序处理缺页异常（page fault），实现自定义的内存管理策略。

See also: [[内存管理]], [[Linux内核与系统调用]], [[进程内存区域与资源限制]]

---

## API 说明

```c
#include <linux/userfaultfd.h>
#include <sys/ioctl.h>

int userfaultfd(int flags); // flags: O_CLOEXEC | O_NONBLOCK
// 通过 ioctl 注册监控范围：UFFDIO_REGISTER, UFFDIO_COPY, UFFDIO_ZEROPAGE 等
// read返回 struct uffd_msg 描述缺页事件
```

## 核心流程

1. 调用 `userfaultfd()` 创建一个 userfaultfd 文件描述符
2. 通过 `ioctl(uffd, UFFDIO_API, ...)` 初始化 API 版本
3. 使用 `mmap` 分配一块匿名内存区域
4. 通过 `ioctl(uffd, UFFDIO_REGISTER, ...)` 注册监控该内存区域，指定模式（如 `UFFDIO_REGISTER_MODE_MISSING` 监控缺页）
5. 在另一个线程中通过 `read(uffd, ...)` 阻塞等待缺页事件（`struct uffd_msg`）
6. 收到 `UFFD_EVENT_PAGEFAULT` 事件后，用户空间决定如何处理：
   - `UFFDIO_COPY`：从用户空间分配的内存拷贝数据到缺页地址
   - `UFFDIO_ZEROPAGE`：映射零页面
7. 处理完成后，触发缺页的线程继续执行

## 示例代码

```c
#define _GNU_SOURCE
#include <sys/syscall.h>
#include <linux/userfaultfd.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <pthread.h>
#include <poll.h>

static void *fault_handler_thread(void *arg) {
    int uffd = (int)(long)arg;
    long page_size = sysconf(_SC_PAGESIZE);

    for (;;) {
        struct pollfd pfd = { .fd = uffd, .events = POLLIN };
        poll(&pfd, 1, -1);

        struct uffd_msg msg;
        read(uffd, &msg, sizeof(msg));

        if (msg.event != UFFD_EVENT_PAGEFAULT) continue;

        printf("Page fault at %p\n", (void *)msg.arg.pagefault.address);

        // 用零页面填充
        char *page = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
                         MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
        memset(page, 'A', page_size);

        struct uffdio_copy copy = {
            .src = (unsigned long)page,
            .dst = msg.arg.pagefault.address & ~(page_size - 1),
            .len = page_size,
            .mode = 0
        };
        ioctl(uffd, UFFDIO_COPY, &copy);
        munmap(page, page_size);
    }
    return NULL;
}

void userfaultfd_example() {
    long page_size = sysconf(_SC_PAGESIZE);
    int uffd = syscall(SYS_userfaultfd, O_CLOEXEC | O_NONBLOCK);

    struct uffdio_api api = { .api = UFFD_API };
    ioctl(uffd, UFFDIO_API, &api);

    void *region = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

    struct uffdio_register reg = {
        .range = { .start = (unsigned long)region, .len = page_size },
        .mode = UFFDIO_REGISTER_MODE_MISSING
    };
    ioctl(uffd, UFFDIO_REGISTER, &reg);

    pthread_t handler;
    pthread_create(&handler, NULL, fault_handler_thread, (void *)(long)uffd);

    // 访问时触发缺页，由用户空间处理
    printf("First byte: %c\n", ((char *)region)[0]);

    munmap(region, page_size);
    close(uffd);
}
```

## 典型应用场景

- **虚拟机实时迁移（Live Migration）**：QEMU/KVM 使用 userfaultfd 实现 post-copy 迁移，按需从源端拉取内存页
- **快照/检查点恢复**：进程恢复时延迟加载内存页
- **分布式共享内存**：远程节点按需拉取内存页
- **内存去重/压缩**：用户空间自定义页面内容管理

## 相关系统调用

- `UFFDIO_REGISTER`：注册监控的内存范围
- `UFFDIO_UNREGISTER`：取消注册
- `UFFDIO_COPY`：将用户空间页面内容复制到缺页地址
- `UFFDIO_ZEROPAGE`：将缺页地址映射为零页面
- `UFFDIO_WAKE`：唤醒等待该范围事件的线程
- `UFFDIO_WRITEPROTECT`：设置/清除写保护（Linux 5.7+）

[src: raw/ingested/2技术/cpp/Linux c系统调用-十八、高级内存管理.md]

## Related Pages
- [[内存管理]]
- [[Linux内核与系统调用]]
- [[进程内存区域与资源限制]]
- [[mmap]]
