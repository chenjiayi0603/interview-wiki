# poll / select / pselect / ppoll

See also: [[epoll]], [[C++网络编程]]

## API 说明

```c
#include <poll.h>
#include <sys/select.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
int ppoll(struct pollfd *fds, nfds_t nfds, const struct timespec *tmo_p,
          const sigset_t *sigmask);                              // [2.6.16]
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
int pselect6(int nfds, fd_set *readfds, fd_set *writefds,
             fd_set *exceptfds, const struct timespec *timeout,
             const sigset_t *sigmask);                           // [2.6.16]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-二、I-O-多路复用与事件驱动.md]

## Related Pages
- [[epoll]]
- [[C++网络编程]]
