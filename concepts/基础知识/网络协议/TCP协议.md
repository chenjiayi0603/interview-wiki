# TCP 协议

> 面向面试的系统化 TCP 协议笔记，整合协议结构、状态机、可靠性机制、内核调优与抓包要点。

See also: [[分布式IM消息系统-雷漫]]

## 一、协议概述

| 特性 | 说明 |
|------|------|
| **面向连接** | 一对一连接，传输前需三次握手建立连接 |
| **可靠传输** | 数据无差错、不丢失、不重复、按序到达 |
| **字节流** | 无边界、有序，对重复报文自动丢弃 |
| **全双工** | 双向传输，双方均可同时收发 |

### TCP 与 UDP 区别

| 对比项 | TCP | UDP |
|--------|-----|-----|
| 连接 | 面向连接 | 无连接 |
| 可靠性 | 可靠交付 | 尽最大努力交付 |
| 服务对象 | 一对一 | 一对一、一对多、多对多 |
| 拥塞/流量控制 | 有 | 无 |
| 首部开销 | 20 字节起（含选项可更长） | 固定 8 字节 |
| 传输方式 | 流式，无边界但有序 | 有边界，可能丢包、乱序 |
| 分片 | 传输层按 MSS 分片 | IP 层按 MTU 分片 |
| 典型应用 | HTTP/HTTPS、FTP | DNS、SNMP、视频流、广播 |

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 二、TCP 协议头结构

### 2.1 固定头部（20 字节）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |C|E|U|A|P|R|S|F|                             |
| Offset| Reserved  |W|C|R|C|S|S|Y|I|            Window           |
|       |           |R|E|G|K|H|T|N|N|                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| 源端口 / 目的端口 | 各 16 位 | 标识应用 |
| 序号 Sequence Number | 32 位 | 本报文段第一个字节的序号 |
| 确认号 Acknowledgment Number | 32 位 | 期望收到的下一个字节序号 |
| 数据偏移 Data Offset | 4 位 | 头部长度（以 4 字节为单位），最小 5，最大 15 |
| 标志位 | 6 位 | URG/ACK/PSH/RST/SYN/FIN |
| 窗口 Window | 16 位 | 接收窗口大小，用于流量控制 |
| 校验和 Checksum | 16 位 | 头部 + 数据 |
| 紧急指针 Urgent Pointer | 16 位 | 仅在 URG=1 时有效 |

### 2.2 常见标志位

| 标志 | 含义 |
|------|------|
| SYN | 建立连接 |
| ACK | 确认 |
| FIN | 关闭连接 |
| RST | 重置连接 |
| PSH | 推送，希望立即交付 |
| URG | 紧急指针有效 |

### 2.3 报文长度与 MTU/MSS

| 概念 | 层次 | 典型值 | 说明 |
|------|------|--------|------|
| MTU | 数据链路层 | 1500 字节（以太网） | 最大传输单元 |
| MSS | 传输层 | 1460 字节（TCP） | MSS = MTU - 20(IP) - 20(TCP) |
| UDP MSS | 传输层 | 1472 字节 | 1500 - 20 - 8 = 1472 |

- MSS 在三次握手 SYN 中协商，取双方最小值
- 带 TCP timestamp 时，实际数据约 1448 字节（1460 - 12）

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 三、TCP 选项（Options）

TCP 选项最大 40 字节，常见选项如下：

| 选项 | Kind | 长度 | 说明 |
|------|------|------|------|
| 结束 EOL | 0 | 1 | 选项结束 |
| 无操作 NOP | 1 | 1 | 填充对齐 |
| **MSS** | 2 | 4 | 最大报文段长度，仅在 SYN 中 |
| **SACK-Permitted** | 4 | 2 | 是否支持 SACK，仅在 SYN 中 |
| **SACK** | 5 | 可变 | 选择性确认，报告已收到的非连续块 |
| **Timestamp** | 8 | 10 | 时间戳，用于 RTT 计算、防序号回绕 |

### 3.1 SACK（Selective Acknowledgment）

- **作用**：告知发送方已收到的非连续数据块，减少不必要重传
- **SACK-Permitted**：在 SYN/SYN+ACK 中协商，双方都支持才启用
- **SACK 格式**：`[Left Edge, Right Edge)` 左闭右开区间，最多 4 组（受 40 字节限制）

### 3.2 D-SACK

- 扩展 SACK，第一个块表示**重复接收**的报文
- 用于判断：丢包、ACK 丢失、网络乱序、重传过度等

### 3.3 Socket 选项

| 选项 | 说明 |
|------|------|
| SO_REUSEADDR | 复用 TIME_WAIT 端口 |
| SO_RCVBUF / SO_SNDBUF | 接收/发送缓冲区大小 |
| SO_RCVLOWAT / SO_SNDLOWAT | 低水位，用于 I/O 复用判断可读/可写 |
| TCP_NODELAY | 关闭 Nagle 算法 |
| SO_REUSEPORT | 多进程监听同一端口，内核负载均衡 |

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 四、TCP 状态机

### 4.1 状态一览

| 状态 | 说明 |
|------|------|
| **CLOSED** | 初始状态 |
| **LISTEN** | 服务端监听，可接受连接 |
| **SYN_SENT** | 客户端已发 SYN，等待 SYN+ACK |
| **SYN_RCVD** | 服务端已收到 SYN 并回复 SYN+ACK，等待 ACK |
| **ESTABLISHED** | 连接已建立 |
| **FIN_WAIT_1** | 主动关闭方已发 FIN，等待 ACK |
| **FIN_WAIT_2** | 已收到 ACK，等待对方 FIN |
| **TIME_WAIT** | 已发 ACK，等待 2MSL 后进入 CLOSED |
| **CLOSE_WAIT** | 被动关闭方收到 FIN 并回复 ACK，等待本端关闭 |
| **CLOSING** | 双方同时关闭，均发 FIN 未收到 ACK |
| **LAST_ACK** | 被动关闭方已发 FIN，等待最终 ACK |

### 4.2 状态转换图（简化）

```
                    CLOSED
                       |
      +--- connect() --+-- listen() --+
      |                               v
      v                            LISTEN
  SYN_SENT                            |
      |          SYN+ACK             | SYN
      +----------------------------->+<-------------+
      |                              v              |
      |                         SYN_RCVD            |
      |                              |   ACK        |
      |                              +------------->|
      v                              v              |
  ESTABLISHED <----------------> ESTABLISHED <-----+
      |                              |
      | FIN                          | FIN
      v                              v
  FIN_WAIT_1                      CLOSE_WAIT
      | ACK                            | close()
      v                               v
  FIN_WAIT_2                      LAST_ACK
      | FIN                            | ACK
      v                               v
  TIME_WAIT --------------------> CLOSED
      | 2MSL
      v
  CLOSED
```

### 4.3 关键状态说明

- **SYN_RCVD**：短暂，一般不易看到；若客户端不发最后一个 ACK，会保持此状态
- **FIN_WAIT_1**：少见，对方通常很快回 ACK
- **TIME_WAIT**：主动关闭方等待 2MSL，确保旧报文消失、最后 ACK 可重传
- **CLOSING**：双方几乎同时 close，两端都发 FIN 但未收到 ACK

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 五、三次握手与四次挥手

### 5.1 三次握手

```
   Client                          Server
      |                                |
      |---------- SYN (seq=x) -------->|   SYN_SENT
      |                                |   SYN_RCVD
      |<------- SYN+ACK (seq=y,ack=x+1)|   (半连接队列)
      |                                |
      |---------- ACK (ack=y+1) ------>|   ESTABLISHED
      |   ESTABLISHED                  |   (全连接队列)
```

- **第一次**：客户端发 SYN，进入 SYN_SENT
- **第二次**：服务端回 SYN+ACK，进入 SYN_RCVD，将连接放入半连接队列
- **第三次**：客户端回 ACK，双方 ESTABLISHED，服务端将连接移入全连接队列

### 5.2 四次挥手

```
   Client                          Server
      |                                |
      |---------- FIN (seq=a) -------->|   FIN_WAIT_1
      |                                |   CLOSE_WAIT
      |<------- ACK (ack=a+1) ---------|   (可能与 FIN 合并)
      |   FIN_WAIT_2                   |
      |<------- FIN (seq=b) -----------|
      |   TIME_WAIT                    |   LAST_ACK
      |---------- ACK (ack=b+1) ------>|
      |   (等待 2MSL)                  |   CLOSED
      v
   CLOSED
```

- **延迟确认**：服务端 ACK 常与 FIN 合并，抓包可见"三次挥手"
- **TIME_WAIT 原因**：① 保证最后的 ACK 可重传 ② 使本连接产生的报文从网络中消失

### 5.3 半连接队列与全连接队列

| 队列 | 状态 | 触发时机 | 大小相关参数 |
|------|------|----------|--------------|
| **半连接队列** | SYN_RCVD | 收到 SYN，回复 SYN+ACK 后 | `min(tcp_max_syn_backlog, backlog, somaxconn)` 相关 |
| **全连接队列** | ESTABLISHED | 收到第三次 ACK 后 | `min(backlog, somaxconn)` |

- **半连接队列满**：可启用 SYN Cookie 抵御 SYN Flood
- **全连接队列满**：丢弃 ACK；`tcp_abort_on_overflow=1` 时回 RST

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 六、流量控制与拥塞控制

### 6.1 流量控制

- **目的**：控制发送速率，保证接收方能处理
- **机制**：滑动窗口，接收方通过 ACK 中的窗口字段通告
- **发送窗口** = min(接收窗口, 拥塞窗口)

### 6.2 滑动窗口

- 以字节为单位，发送端在未收到确认前可连续发送窗口内数据
- 只有收到确认、窗口前移后，发送窗口才能前移
- 窗口为 0 时，发送方暂停发送

### 6.3 拥塞控制详解

TCP 拥塞控制通过调节拥塞窗口（cwnd）实现，核心目标是根据网络状况自适应地控制发送速率，尽量避免造成网络拥塞和丢包。其主要包括以下四个阶段算法：

#### 1. 慢启动（Slow Start）
- 初始时 cwnd 设置为较小值（如 1 MSS）。
- 每收到一个 ACK，cwnd 增加 1 MSS（指数级增长）。
- 当 cwnd 达到慢启动门限（ssthresh）时，进入"拥塞避免"阶段。

#### 2. 拥塞避免（Congestion Avoidance）
- 当 cwnd ≥ ssthresh 时，每收到一个 ACK，cwnd 按 1/cwnd 递增（即每轮窗口大小后增长 1 MSS，线性增长）。
- 目的是缓慢探测网络最大带宽，防止过快增加导致拥塞。

#### 3. 快速重传（Fast Retransmit）
- 发送方如果连续收到 3 个重复 ACK，判断可能丢包，不等超时即重传出错数据包，加快数据恢复速度。

#### 4. 快速恢复（Fast Recovery）
- 响应快速重传，cwnd 不回到 1，而是直接减半（cwnd = ssthresh + 3 MSS），随后以拥塞避免方式增长。

#### 5. 超时重传
- 若发生超时，说明拥塞严重：ssthresh = cwnd/2, cwnd 重置为 1 MSS，重新"慢启动"探测。

**注意：**
- 拥塞窗口（cwnd）和接收方窗口共同决定实际发送窗口：发送窗口 = min(接收窗口, 拥塞窗口)
- 拥塞窗口是 TCP 发送方根据网络状况自适应调整的变量。

**核心参数（Linux 内核名）：**
| 参数             | 说明                 |
|------------------|----------------------|
| tcp_congestion_control | 拥塞控制算法选择（如 cubic/reno） |
| tcp_moderate_rcvbuf    | 自动调节接收缓存      |
| ssthresh               | 慢启动门限          |
| cwnd                   | 拥塞窗口大小        |

**常见算法演变：**
- 经典: Reno、NewReno
- 现代: CUBIC（Linux 默认）、BBR

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 七、可靠性机制

### 7.1 超时重传（RTO）

- 发送后未在 RTO 内收到 ACK 则重传
- RTO 基于 RTT 估算（如 RTT 采样、指数加权移动平均）

### 7.2 快速重传

- 接收方收到乱序包时，重复发送对"缺口"前一个序号的 ACK
- 发送方连续收到 3 个相同 ACK 即重传，不等超时

### 7.3 延迟确认

- 收到数据后不立即回 ACK，通常延迟约 200ms（定时器周期性检查）
- 目的：合并 ACK；若有数据要发，可捎带 ACK
- 四次挥手常变"三次"，即因 ACK 与 FIN 捎带

### 7.4 Nagle 算法

- 任意时刻最多一个未被确认的小段（< MSS）
- 与延迟确认合用可能带来延迟；可设 `TCP_NODELAY` 关闭

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 八、内核参数与调优

### 8.1 连接建立相关

| 参数 | 默认值 | 优化建议值 | 说明 |
|------|--------|-----------|------|
| tcp_syncookies | 0 | 1 | 半连接队列溢出时用 Cookie 防御 SYN Flood |
| tcp_max_syn_backlog | 1024 | 8192 | 半连接队列最大长度，提升高并发承载 |
| tcp_syn_retries | 5 | 2 | 主动建连 SYN 重试次数，降低失败等待时间 |
| tcp_synack_retries | 5 | 2 | 被动建连 SYN+ACK 重试次数，防御无效连接消耗 |
| somaxconn | 128 | 1024/2048 | 全连接队列上限，与 listen backlog 取小、业务高并发可适当调高 |

### 8.2 TIME_WAIT 相关

| 参数 | 默认值 | 优化建议值 | 说明 |
|------|--------|-----------|------|
| tcp_tw_reuse | 0 | 1 | 复用 TIME_WAIT 套接字建新连接（服务器推荐） |
| tcp_tw_recycle | 0 | 0 | 快速回收 TIME_WAIT（不推荐/NAT 环境会出错） |
| tcp_fin_timeout | 60 | 15~30 | FIN_WAIT_2 超时时间（秒），加速关闭无用连接 |
| tcp_max_tw_buckets | 180000 | 500000 | 最大 TIME_WAIT 数量，应对高并发避免拒绝新连接 |

### 8.3 重传与保活

| 参数 | 默认值 | 优化建议值 | 说明 |
|------|--------|-----------|------|
| tcp_retries1 | 3 | 3 | 未建立连接前的重试次数，适度保守 |
| tcp_retries2 | 15 | 5~8 | 已建立连接的重试次数，及时释放僵尸连接 |
| tcp_keepalive_time | 7200 | 60~300 | 空闲多久后发 keepalive 探测，短连接压测建议 60~120 |
| tcp_keepalive_probes | 9 | 3 | keepalive 探测次数，减少探测延迟 |
| tcp_keepalive_intvl | 75 | 10~20 | 探测间隔（秒），降低掉线探测时延 |

### 8.4 其他常用参数

| 参数 | 优化建议值及意义 |
|------|-----------------|
| tcp_abort_on_overflow | 1（生产环境建议开启，队列满时回 RST 及时反馈客户端） |
| tcp_sack / tcp_dsack | 1（保持默认开启，有效提升丢包恢复效率） |
| tcp_timestamps | 1（默认开启，建议不改动） |
| ip_local_port_range | 10000 65000（高并发建议扩大，如 10000 65535，提高端口可用范围） |

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 九、抓包分析要点（Wireshark）

### 9.1 常用过滤

| 过滤表达式 | 说明 |
|------------|------|
| `tcp.port == 80` | 指定端口 |
| `tcp.flags.syn == 1` | SYN 包 |
| `tcp.flags.fin == 1` | FIN 包 |
| `tcp.analysis.retransmission` | 重传包 |
| `tcp.analysis.ack_rtt > 0.2 and tcp.len==0` | 延迟超过 200ms 的纯 ACK |

### 9.2 关键字段观察

- **Seq / Ack**：序号与确认号是否连续、有无乱序
- **Win**：窗口大小变化，判断流量控制
- **Len**： payload 长度
- **Options**：MSS、SACK、Timestamp 等

### 9.3 常见异常

| 现象 | 可能原因 |
|------|----------|
| 大量 SYN 无 ACK | SYN Flood 或服务不可达 |
| 大量重传 | 丢包、拥塞、RTT 大 |
| 只有 SYN/SYN+ACK | 客户端未发第三次 ACK 或 ACK 丢失 |
| RST | 连接被重置、全连接队列满（tcp_abort_on_overflow） |

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 十、常见面试题速记

1. **为什么三次握手？**  
   防止已失效的连接请求突然到达服务端，造成资源浪费；两次无法确认双方收发能力，四次多余。

2. **为什么四次挥手？**  
   TCP 全双工，关闭需要双方各自发 FIN；被动方可能还有数据要发， ACK 与 FIN 分开发送。

3. **TIME_WAIT 为什么是 2MSL？**  
   保证最后的 ACK 可重传；让本连接产生的报文从网络中消失，避免影响新连接。

4. **流量控制与拥塞控制的区别？**  
   流量控制针对接收能力；拥塞控制针对网络负载，是全局性的。

5. **TCP 如何保证可靠？**  
   序号确认、超时重传、快速重传、SACK、校验和、流量控制、拥塞控制。

6. **半连接队列满会怎样？**  
   开启 syncookies 时用 Cookie 继续；否则丢弃新 SYN。

7. **全连接队列满会怎样？**  
   丢弃第三次握手的 ACK；tcp_abort_on_overflow=1 时回 RST。

[src: raw/ingested/2技术/网络协议/tcp协议分析.md]

## 十一、D-SACK（Duplicate SACK）

D-SACK 是 RFC2883 对 SACK 的扩展，使得扩展后的 SACK 具有通知发送端哪些数据被重复接收了。

### 11.1 引入 D-SACK 的目的

引入 D-SACK 的目的是使 TCP 进行更好的流控，具体来说有以下几个好处：

1. 让发送方知道，是发送的包丢了，还是返回的 ACK 包丢了；
2. 网络上是否出现了包失序；
3. 数据包是否被网络上的路由器复制并转发了；
4. 是不是自己的 timeout 太小了，导致重传。

通过 D-SACK 这种方法，发送方可以更仔细判断出当前网络的传输情况，可以发现数据段被网络复制、错误重传、ACK 丢失引起的重传、重传超时等异常的网络状况。

### 11.2 D-SACK 的判断与规则

D-SACK 使用了 SACK 的第一个段来做标志，如何判断 D-SACK：

1. 如果 SACK 的第一个段的范围被 ACK 所覆盖，那么就是 D-SACK；
2. 如果 SACK 的第一个段的范围被 SACK 的第二个段覆盖，那么就是 D-SACK。

D-SACK 的规则如下：

1. 第一个 block 将包含重复收到的报文段的序号；
2. 跟在 D-SACK 之后的 SACK 将按照 SACK 的方式工作；
3. 如果有多个被重复接收的报文段，则 D-SACK 只包含其中第一个。

[src: raw/ingested/2技术/网络协议/tcp优化-tcp算法-TCP-的那些事---D-SACK.md]

## Related Pages
- [[分布式IM消息系统-雷漫]]
- [[C++网络编程]]
- [[Linux_IO]]
- [[服务网格平台-即构]]
- [[Go网络编程]]
- [[Redis连接问题]]