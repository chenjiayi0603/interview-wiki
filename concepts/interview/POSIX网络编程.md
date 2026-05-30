# POSIX 网络编程 (Socket)

> 本文涵盖 POSIX 标准中网络编程相关的 C API：地址信息获取、scatter/gather I/O、socket 选项、字节序转换、地址转换等。

See also: [[C++网络编程]], [[Boost.Asio]], [[TCP协议]], [[C++POSIX文件操作]], [[POSIX进程派生]]

---

## 一、地址信息 (Address Info)

```c
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char *node, const char *service,
                const struct addrinfo *hints,
                struct addrinfo **res);           // [POSIX]
void freeaddrinfo(struct addrinfo *res);          // [POSIX]
const char *gai_strerror(int errcode);            // [POSIX]
int getnameinfo(const struct sockaddr *sa, socklen_t salen,
                char *host, socklen_t hostlen,
                char *serv, socklen_t servlen, int flags); // [POSIX]
```

**核心函数**：
- `getaddrinfo`：将主机名/服务名解析为 `addrinfo` 链表，支持 IPv4/IPv6，是协议无关的地址解析方式。
- `freeaddrinfo`：释放 `getaddrinfo` 返回的链表。
- `gai_strerror`：将 `getaddrinfo` / `getnameinfo` 的错误码转为可读字符串。
- `getnameinfo`：反向解析，将 socket 地址转为主机名/服务名。

**`getaddrinfo` 参数说明**：
- `node`：主机名（如 "www.example.com"）或 IP 地址字符串（如 "192.168.1.1"），可为 NULL。
- `service`：服务名（如 "http"）或端口号字符串（如 "80"），可为 NULL。
- `hints`：期望的地址类型提示（如 `AF_INET`、`SOCK_STREAM`），可为 NULL。
- `res`：输出参数，返回 `addrinfo` 链表头指针。

**与旧 API 对比**：
- 旧 API：`gethostbyname` / `gethostbyaddr`（仅 IPv4，非线程安全，已废弃）。
- 新 API：`getaddrinfo` / `getnameinfo`（协议无关，线程安全，推荐使用）。

---

## 二、Scatter/Gather Socket I/O

```c
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);    // [POSIX]
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags); // [POSIX]
```

**功能**：
- `recvmsg` / `sendmsg` 支持 scatter/gather I/O，一次系统调用可操作多个缓冲区（`iovec` 数组）。
- 同时可传递辅助数据（ancillary data），如通过 `SCM_RIGHTS` 在进程间传递文件描述符。

**`struct msghdr` 关键字段**：
- `msg_name` / `msg_namelen`：目标/源地址（用于无连接协议如 UDP）。
- `msg_iov` / `msg_iovlen`：I/O 向量数组（scatter/gather 缓冲区）。
- `msg_control` / `msg_controllen`：辅助数据（控制信息）。
- `msg_flags`：接收时的标志（如 `MSG_TRUNC`、`MSG_EOR`）。

---

## 三、Socket 选项与控制

### 3.1 socketpair

```c
int socketpair(int domain, int type, int protocol, int sv[2]); // [POSIX]
```

- 创建一对已连接的匿名 socket（全双工通信通道）。
- `sv[0]` 和 `sv[1]` 分别返回两个 socket 描述符，可双向通信。
- 常用于父子进程间通信（`fork` 后各持一端）。

### 3.2 shutdown

```c
int shutdown(int sockfd, int how);                 // [POSIX]
```

- 关闭 socket 的发送/接收通道，但不释放文件描述符。
- `how` 参数：
  - `SHUT_RD`：关闭读通道。
  - `SHUT_WR`：关闭写通道（发送 FIN）。
  - `SHUT_RDWR`：同时关闭读写。
- 与 `close` 的区别：`close` 释放描述符，仅当引用计数为 0 时才真正关闭；`shutdown` 直接关闭通道，无视引用计数。

### 3.3 getsockopt / setsockopt

```c
int getsockopt(int sockfd, int level, int optname,
               void *optval, socklen_t *optlen);   // [POSIX]
int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen); // [POSIX]
```

- 获取/设置 socket 选项。
- `level`：选项级别（如 `SOL_SOCKET`、`IPPROTO_TCP`）。
- 常用选项：
  - `SO_REUSEADDR`：允许重用本地地址（解决 TIME_WAIT 问题）。
  - `SO_KEEPALIVE`：启用 TCP keep-alive。
  - `SO_RCVBUF` / `SO_SNDBUF`：接收/发送缓冲区大小。
  - `TCP_NODELAY`：禁用 Nagle 算法。

### 3.4 getsockname / getpeername

```c
int getsockname(int sockfd, struct sockaddr *addr,
                socklen_t *addrlen);               // [POSIX]
int getpeername(int sockfd, struct sockaddr *addr,
                socklen_t *addrlen);               // [POSIX]
```

- `getsockname`：获取 socket 本地地址。
- `getpeername`：获取 socket 对端地址。

---

## 四、字节序转换

```c
#include <arpa/inet.h>

uint16_t htons(uint16_t hostshort);  // [POSIX] Host to Network Short
uint16_t ntohs(uint16_t netshort);   // [POSIX] Network to Host Short
uint32_t htonl(uint32_t hostlong);   // [POSIX] Host to Network Long
uint32_t ntohl(uint32_t netlong);   // [POSIX] Network to Host Long
```

**命名规则**：
- `h` = host（主机字节序，通常为小端 x86）
- `n` = network（网络字节序，大端）
- `s` = short（16 位，用于端口号）
- `l` = long（32 位，用于 IPv4 地址）

**使用场景**：
- 设置端口号：`addr.sin_port = htons(8080);`
- 读取端口号：`int port = ntohs(addr.sin_port);`

---

## 五、地址转换

```c
#include <arpa/inet.h>

int inet_pton(int af, const char *src, void *dst);             // [POSIX]
const char *inet_ntop(int af, const void *src,
                      char *dst, socklen_t size);              // [POSIX]
```

**功能**：
- `inet_pton`（presentation to network）：将 IP 地址字符串（如 "192.168.1.1" 或 "::1"）转为网络字节序的二进制格式。
- `inet_ntop`（network to presentation）：反向转换，二进制 → 字符串。
- `af`：地址族（`AF_INET` 或 `AF_INET6`）。

**与旧 API 对比**：
- 旧 API：`inet_addr` / `inet_ntoa`（仅 IPv4，非线程安全，已废弃）。
- 新 API：`inet_pton` / `inet_ntop`（支持 IPv4/IPv6，线程安全，推荐使用）。

---

## 六、面试高频问题

### Q1: `getaddrinfo` 相比 `gethostbyname` 的优势？
- **协议无关**：支持 IPv4 和 IPv6，无需修改代码。
- **线程安全**：`gethostbyname` 返回静态缓冲区指针，多线程不安全；`getaddrinfo` 动态分配内存。
- **更灵活**：通过 `hints` 参数精确控制返回的地址类型。

### Q2: `shutdown` 和 `close` 的区别？
- `close`：减少文件描述符引用计数，仅当计数为 0 时才关闭 socket。
- `shutdown`：直接关闭读/写通道，无视引用计数。
- 典型场景：`fork` 后父子进程共享 socket，`close` 只减少引用计数，`shutdown` 才能真正关闭通道。

### Q3: 为什么需要字节序转换？
- 网络协议规定使用**大端字节序**（网络字节序）。
- x86 架构使用**小端字节序**（主机字节序）。
- 在设置端口号、IP 地址时必须转换，否则跨平台通信会出错。

### Q4: `sendmsg` / `recvmsg` 的典型用途？
- **Scatter/Gather I/O**：一次系统调用读写多个不连续缓冲区，减少系统调用次数。
- **传递文件描述符**：通过 `SCM_RIGHTS` 辅助数据在进程间传递 fd（如 Nginx 的 worker 进程间传递连接）。
- **获取目标地址**：UDP 场景下 `recvmsg` 可获取数据包的目标 IP（用于多宿主主机）。

### Q5: `SO_REUSEADDR` 的作用？
- 允许重用处于 `TIME_WAIT` 状态的本地地址和端口。
- 服务器重启时可立即绑定同一端口，避免 "Address already in use" 错误。

[src: raw/ingested/2技术/cpp/C++ POSIX API参考手册-18.-网络编程-(Socket).md]

## Related Pages
- [[C++网络编程]]
- [[Boost.Asio]]
- [[TCP协议]]
- [[C++POSIX文件操作]]
- [[POSIX进程派生]]
- [[POSIX用户组与环境]]
