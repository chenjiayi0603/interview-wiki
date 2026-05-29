# 零拷贝与异步 I/O

> 本文涵盖 Linux 下零拷贝数据传输与异步 I/O 相关的系统调用 API：splice、tee、vmsplice、sendfile、copy_file_range、传统 Linux AIO 以及 io_uring。

See also: [[C++POSIX文件操作]], [[Linux文件系统与路径操作]], [[IPC进程间通信]], [[C++网络编程]]

---

## 一、零拷贝数据传输

### 1.1 splice / tee / vmsplice / sendfile / copy_file_range

**API说明：**
```c
#include <fcntl.h>
#include <sys/sendfile.h>

ssize_t splice(int fd_in, loff_t *off_in, int fd_out, loff_t *off_out,
               size_t len, unsigned int flags);   // [Linux 2.6.17+]
// flags: SPLICE_F_MOVE | SPLICE_F_NONBLOCK | SPLICE_F_MORE

ssize_t tee(int fd_in, int fd_out, size_t len,
            unsigned int flags);                  // [Linux 2.6.17+]

ssize_t vmsplice(int fd, const struct iovec *iov,
                 unsigned long nr_segs, unsigned int flags); // [Linux 2.6.17+]

ssize_t sendfile(int out_fd, int in_fd,
                 off_t *offset, size_t count);    // [Linux 2.2+]

ssize_t copy_file_range(int fd_in, loff_t *off_in,
                        int fd_out, loff_t *off_out,
                        size_t len, unsigned int flags); // [Linux 4.5+]
```

**示例：sendfile零拷贝文件传输**
```c
#include <sys/sendfile.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/stat.h>

void sendfile_example(const char *src, const char *dst) {
    int in_fd = open(src, O_RDONLY);
    int out_fd = open(dst, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (in_fd == -1 || out_fd == -1) { perror("open"); return; }

    struct stat st;
    fstat(in_fd, &st);

    off_t offset = 0;
    ssize_t sent = sendfile(out_fd, in_fd, &offset, st.st_size);
    printf("sendfile: transferred %zd bytes\n", sent);

    close(in_fd);
    close(out_fd);
}
```

**示例：splice管道中转**
```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

void splice_example(const char *src, const char *dst) {
    int in_fd = open(src, O_RDONLY);
    int out_fd = open(dst, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    int pipefd[2];
    pipe(pipefd);

    // in_fd -> pipe -> out_fd（零拷贝）
    ssize_t n;
    while ((n = splice(in_fd, NULL, pipefd[1], NULL, 65536,
                       SPLICE_F_MOVE | SPLICE_F_MORE)) > 0) {
        splice(pipefd[0], NULL, out_fd, NULL, n, SPLICE_F_MOVE | SPLICE_F_MORE);
    }

    close(pipefd[0]); close(pipefd[1]);
    close(in_fd); close(out_fd);
    printf("splice copy done\n");
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-四、零拷贝与异步-I-O.md]

---

## 二、传统 AIO (Linux AIO) `[Linux 2.6]`

```c
#include <linux/aio_abi.h>

int io_setup(unsigned nr_events, aio_context_t *ctx_id);
int io_destroy(aio_context_t ctx_id);
int io_submit(aio_context_t ctx_id, long nr, struct iocb **iocbpp);
int io_getevents(aio_context_t ctx_id, long min_nr, long nr,
                 struct io_event *events, struct timespec *timeout);
int io_cancel(aio_context_t ctx_id, struct iocb *iocb, struct io_event *result);
int io_pgetevents(aio_context_t ctx_id, long min_nr, long nr,
                  struct io_event *events, struct timespec *timeout,
                  const struct __aio_sigset *sig);  // [4.18]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-四、零拷贝与异步-I-O.md]

---

## 三、io_uring — 高性能异步 I/O `[Linux 5.1+]`

**API说明：**
```c
#include <linux/io_uring.h>
#include <sys/syscall.h>

// 底层系统调用                                    // 全部 [Linux 5.1+]
int io_uring_setup(unsigned entries, struct io_uring_params *p);
int io_uring_enter(int fd, unsigned to_submit, unsigned min_complete,
                   unsigned flags, sigset_t *sig);
int io_uring_register(int fd, unsigned opcode, void *arg, unsigned nr_args);

// 通常使用 liburing 封装库：
#include <liburing.h>

int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags);
void io_uring_queue_exit(struct io_uring *ring);
struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring);
void io_uring_prep_readv(struct io_uring_sqe *sqe, int fd,
                         const struct iovec *iovecs, unsigned nr_vecs, off_t offset);
void io_uring_prep_writev(struct io_uring_sqe *sqe, int fd,
                          const struct iovec *iovecs, unsigned nr_vecs, off_t offset);
void io_uring_prep_read(struct io_uring_sqe *sqe, int fd, void *buf,
                        unsigned nbytes, off_t offset);
void io_uring_prep_write(struct io_uring_sqe *sqe, int fd, const void *buf,
                         unsigned nbytes, off_t offset);
void io_uring_prep_accept(struct io_uring_sqe *sqe, int fd,
                          struct sockaddr *addr, socklen_t *addrlen, int flags);
void io_uring_prep_connect(struct io_uring_sqe *sqe, int fd,
                           const struct sockaddr *addr, socklen_t addrlen);
void io_uring_sqe_set_data(struct io_uring_sqe *sqe, void *data);
int io_uring_submit(struct io_uring *ring);
int io_uring_wait_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);
int io_uring_peek_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);
void io_uring_cqe_seen(struct io_uring *ring, struct io_uring_cqe *cqe);
```

**示例：使用liburing读取文件**
```c
#include <liburing.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void io_uring_read_example() {
    struct io_uring ring;
    io_uring_queue_init(32, &ring, 0);

    int fd = open("/etc/hostname", O_RDONLY);
    char buf[256] = {0};

    // 提交读请求
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, sizeof(buf) - 1, 0);
    io_uring_sqe_set_data(sqe, buf);
    io_uring_submit(&ring);

    // 等待完成
    struct io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);
    if (cqe->res > 0) {
        printf("io_uring read %d bytes: %s\n", cqe->res, buf);
    }
    io_uring_cqe_seen(&ring, cqe);

    close(fd);
    io_uring_queue_exit(&ring);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-四、零拷贝与异步-I-O.md]

## Related Pages
- [[C++POSIX文件操作]]
- [[Linux文件系统与路径操作]]
- [[IPC进程间通信]]
- [[C++网络编程]]
- [[内存管理]]
