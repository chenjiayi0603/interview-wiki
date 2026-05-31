# TCP 连接队列

## 概述

TCP 服务端在调用 `listen` 后，内核会创建两个队列来管理连接：
- **半连接队列（Incomplete connection queue / SYN Queue）**：存储收到 SYN 但未完成三次握手的连接，状态为 `SYN_RCVD`。
- **全连接队列（Completed connection queue / Accept Queue）**：存储已完成三次握手但未被应用调用 `accept` 取走的连接，状态为 `ESTABLISHED`。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 半连接队列

### 作用

为了防止 SYN flood 攻击，内核在三次握手成功之前只分配占用内存极小的 `request_sock`，而不是完整的 `sock`。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

### 队列长度

半连接队列的长度由以下三个参数的最小值决定：
- `listen()` 的 `backlog` 参数
- `/proc/sys/net/ipv4/tcp_max_syn_backlog`
- `/proc/sys/net/core/somaxconn`

实际计算方式为：
```
roundup_pow_of_two(min(tcp_max_syn_backlog, backlog, somaxconn) + 1)
```
即三个参数取最小值后 +1，再向上取离它最近的 2 次幂整数。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

### 队列满时的行为

- **关闭 syncookies**（`net.ipv4.tcp_syncookies = 0`）：当队列满时，不再接受新的连接。
- **开启 syncookies**（`net.ipv4.tcp_syncookies = 1`）：当队列满时，不受影响，内核会发送 cookie 校验。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 全连接队列

### 作用

全连接队列保存已完成三次握手的连接，等待应用调用 `accept()` 取走。可以看作生产者（内核）和消费者（应用）模型。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

### 队列长度

全连接队列的长度为：
```
min(backlog, somaxconn)
```
其中 `backlog` 为 `listen()` 函数的第二个参数，`somaxconn` 为 `/proc/sys/net/core/somaxconn` 的值（默认 128）。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

### 队列满时的行为

当全连接队列满时，内核收到客户端的 ACK 包后的行为由 `tcp_abort_on_overflow` 决定：
- **`tcp_abort_on_overflow = 0`**（默认）：内核只丢弃客户端的 ACK 包，客户端无法感知，直到第一笔调用时才会知道连接被丢弃。
- **`tcp_abort_on_overflow = 1`**：内核返回 RST 包，重置 TCP 连接，客户端可以感知。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

### 查看队列状态

使用 `ss -lnt` 命令可以查看全连接队列的状态：
- `Recv-Q`：当前 accept 队列排队的连接个数
- `Send-Q`：全连接队列的总大小

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 相关参数调优

```bash
# 调整全连接队列大小
listen(fd, 2048)
echo '2048' > /proc/sys/net/core/somaxconn

# 调整半连接队列大小
echo '2048' > /proc/sys/net/ipv4/tcp_max_syn_backlog
```

[src: raw/ingested/2技术/网络协议/tcp优化-tcp连接队列.md]

## 相关页面

- [[TCP协议]]
- [[网络优化技术]]
- [[C++网络编程]]
