# TCP 调优与内核参数

> TCP 内核参数调优、连接队列管理、TIME_WAIT 优化、TLS 握手优化、Socket 选项等实战配置大全。

---

## 一、连接队列管理

### 1.1 半连接队列（SYN Queue）vs 全连接队列（Accept Queue）

```
客户端                         服务端
  |                                |
  |--------- SYN ----------------->|   → 放入半连接队列（SYN_RCVD）
  |<-------- SYN+ACK --------------|
  |--------- ACK ----------------->|   → 移入全连接队列（ESTABLISHED）
  |                                |   → accept() 取走
```

| 队列 | 状态 | 存储内容 | 触发时机 |
|------|------|----------|----------|
| **半连接队列** | SYN_RCVD | request_sock（轻量级） | 收到 SYN，回复 SYN+ACK 后 |
| **全连接队列** | ESTABLISHED | 完整 sock | 收到第三次 ACK 后 |

### 1.2 队列大小计算

**全连接队列**：
```
全连接队列上限 = min(backlog, somaxconn)
```

**半连接队列**（简化计算）：
```
半连接队列上限 = roundup_pow_of_two(min(tcp_max_syn_backlog, backlog, somaxconn) + 1)
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `backlog`（listen 函数参数） | 取决于应用 | 如 Java 默认 50，Nginx 默认 511 |
| `net.core.somaxconn` | 128 | 全连接队列硬上限 |
| `net.ipv4.tcp_max_syn_backlog` | 1024 | 半连接队列硬上限 |

### 1.3 队列满的处理

**半连接队列满**：
- `tcp_syncookies = 1`（推荐）：启用 SYN Cookie 继续处理，不丢连接
- `tcp_syncookies = 0`：直接丢弃新 SYN

**全连接队列满**：
- `tcp_abort_on_overflow = 0`（默认）：默默丢弃第三次 ACK，客户端重试
- `tcp_abort_on_overflow = 1`：回 RST，客户端立即感知连接失败

**查看队列状态**：
```bash
ss -lnt
# Recv-Q: accept 队列排队的连接数
# Send-Q: 全连接队列总大小
```

### 1.4 调优建议

```bash
# 高并发服务建议
listen(fd, 2048)                    # 应用层设置
echo 2048 > /proc/sys/net/core/somaxconn
echo 2048 > /proc/sys/net/ipv4/tcp_max_syn_backlog
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
# echo 1 > /proc/sys/net/ipv4/tcp_abort_on_overflow  # 生产环境可选
```

---

## 二、惊群与 SO_REUSEPORT

### 2.1 惊群问题

多进程/线程在同一个 listen fd 上 accept，新连接到来时所有进程被唤醒，但只有一个能成功，其余做无用功。

### 2.2 SO_REUSEPORT 方案

**特性**（Linux 3.9+）：
- 允许多个 socket bind 相同 IP:PORT
- 内核在多个 listen socket 间做负载均衡（基于四元组 hash）
- 同一用户下的 socket 才能绑定同端口

**优势**：
- 消除惊群（每个进程有自己的 listen socket）
- 内核级负载均衡
- 支持不停服更新（启动新实例→关闭旧实例）

**局限性**：
- listen socket 数量变化时可能导致握手包路由不一致
- 短期负载可能不均衡

**Go 示例**：
```go
import "golang.org/x/sys/unix"
unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
```

---

## 三、TIME_WAIT 优化

### 3.1 TIME_WAIT 状态

- 主动关闭方最后发送 ACK 后进入 TIME_WAIT
- 持续 2MSL（Linux 默认 60 秒）
- 目的：保证最后 ACK 可重传；让旧报文从网络中消失

### 3.2 优化参数

| 参数 | 默认值 | 优化建议 | 说明 |
|------|--------|----------|------|
| `tcp_tw_reuse` | 0 | **1**（服务端） | 复用 TIME_WAIT socket 建新连接（出站连接） |
| `tcp_tw_recycle` | 0 | **0**（不开启） | 快速回收，NAT 环境会出错，**已从 Linux 4.12 移除** |
| `tcp_fin_timeout` | 60 | 15~30 | FIN_WAIT_2 超时时间 |
| `tcp_max_tw_buckets` | 180000 | 500000 或适当调大 | 最大 TIME_WAIT 数量，超限时清除并告警 |

**注意**：`tcp_tw_reuse` 适用于客户端/出站连接复用，通过 `setsockopt` 的 `SO_REUSEADDR` 也可复用服务端 TIME_WAIT 端口。

---

## 四、TCP Keepalive

### 4.1 内核参数

| 参数 | 默认值 | 优化建议 | 说明 |
|------|--------|----------|------|
| `tcp_keepalive_time` | 7200（2小时） | 60~300 | 空闲多久后开始发送探测包 |
| `tcp_keepalive_intvl` | 75 | 10~20 | 探测间隔（秒） |
| `tcp_keepalive_probes` | 9 | 3 | 探测次数，超过后判定连接断开 |

### 4.2 Keepalive 原理

- 完全由操作系统内核协议栈处理，应用层无感知
- 对端协议栈自动回 ACK（即使应用已假死）
- 超时 N 次未响应后关闭 socket，应用层 next read/write 返回错误

**最坏情况探测时间**：keepalive_time + keepalive_intvl × keepalive_probes
- 默认：7200 + 75×9 = 7875 秒 ≈ 2.2 小时
- 优化后：60 + 10×3 = 90 秒

### 4.3 应用层心跳 vs TCP Keepalive

| 场景 | TCP Keepalive | 应用层心跳 |
|------|:------------:|:-----------:|
| 物理断网/宕机 | ✅（慢） | ✅（快） |
| 应用假死/阻塞 | ❌ | ✅ |
| 路由黑洞 | 取决于链路 | ✅ |
| 检测速度 | 分钟级~小时级 | 秒级（可配置） |

**结论**：TCP Keepalive 只能检测物理链路断开，无法发现应用级故障。强在线业务（IM、推送）**必须**使用应用层心跳。

### 4.4 代码配置

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
int keepalive = 1;
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &keepalive, sizeof(keepalive));

int keepidle = 120;    // tcp_keepalive_time
int keepintvl = 10;    // tcp_keepalive_intvl  
int keepcnt = 3;       // tcp_keepalive_probes
setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &keepidle, sizeof(keepidle));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &keepintvl, sizeof(keepintvl));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &keepcnt, sizeof(keepcnt));
```

---

## 五、完整 sysctl 参数参考

### 5.1 连接建立

| 参数 | 默认值 | 建议值 | 说明 |
|------|--------|--------|------|
| `tcp_syncookies` | 1 | 1 | SYN Cookie 防止 SYN Flood |
| `tcp_max_syn_backlog` | 1024 | 8192 | 半连接队列长度 |
| `tcp_syn_retries` | 5 | 2 | 主动建连 SYN 重试次数 |
| `tcp_synack_retries` | 5 | 2 | 被动建连 SYN+ACK 重试次数 |
| `net.core.somaxconn` | 128 | 2048 | 全连接队列上限 |
| `tcp_abort_on_overflow` | 0 | 0/1 | 全连接队列满是否回 RST |

### 5.2 连接断开

| 参数 | 默认值 | 建议值 | 说明 |
|------|--------|--------|------|
| `tcp_fin_timeout` | 60 | 15~30 | FIN_WAIT_2 超时 |
| `tcp_tw_reuse` | 0 | 1 | 复用 TIME_WAIT |
| `tcp_max_tw_buckets` | 180000 | 适当调大 | TIME_WAIT 上限 |
| `tcp_retries1` | 3 | 3 | 未建立连接重试次数 |
| `tcp_retries2` | 15 | 5~8 | 已建立连接重试次数 |
| `tcp_orphan_retries` | 7 | 3 | 孤儿连接重试次数 |

### 5.3 保活

| 参数 | 默认值 | 建议值 | 说明 |
|------|--------|--------|------|
| `tcp_keepalive_time` | 7200 | 60~300 | 空闲多久后开始保活探测 |
| `tcp_keepalive_probes` | 9 | 3 | 保活探测次数 |
| `tcp_keepalive_intvl` | 75 | 10~20 | 探测间隔（秒） |

### 5.4 性能相关

| 参数 | 默认值 | 建议值 | 说明 |
|------|--------|--------|------|
| `tcp_sack` | 1 | 1 | 选择性确认 |
| `tcp_dsack` | 1 | 1 | D-SACK |
| `tcp_timestamps` | 1 | 1 | 时间戳选项 |
| `tcp_window_scaling` | 1 | 1 | 窗口扩大因子 |
| `tcp_moderate_rcvbuf` | 1 | 1 | 自动调节接收缓存 |
| `ip_local_port_range` | 32768 60999 | 10000 65000 | 本地端口范围 |
| `tcp_congestion_control` | cubic | cubic/bbr | 拥塞控制算法 |

---

## 六、Socket 选项

### 6.1 常用选项

| 选项 | 说明 |
|------|------|
| `SO_REUSEADDR` | 复用处于 TIME_WAIT 的地址 |
| `SO_REUSEPORT` | 多进程监听同一端口（内核负载均衡） |
| `SO_RCVBUF` / `SO_SNDBUF` | 接收/发送缓冲区大小（系统会加倍） |
| `SO_RCVLOWAT` / `SO_SNDLOWAT` | 低水位标记（I/O 复用判断可读/可写） |
| `TCP_NODELAY` | 关闭 Nagle 算法 |
| `TCP_CORK` | 攒包发送（与 Nagle 不同，对端可能延迟） |
| `TCP_QUICKACK` | 关闭延迟 ACK（需每次设置） |
| `TCP_DEFER_ACCEPT` | 延迟 accept 直到有数据到达 |
| `SO_KEEPALIVE` | 开启 TCP 保活探测 |

### 6.2 缓冲区说明

- `setsockopt` 设置 SO_RCVBUF 时，内核会取 `设置值 × 2`（需 ≥ 256 字节）
- SO_SNDBUF 最小值 2048 字节
- 可通过 `tcp_moderate_rcvbuf` 自动调节接收缓存

---

## 七、TLS 握手优化

### 7.1 TLS 握手与 TCP 的关系

HTTPS = HTTP over TLS over TCP

| 场景 | TCP | TLS | 总 RTT |
|------|-----|-----|--------|
| TLS 1.2 首次连接 | 1 | 2 | **3 RTT** |
| TLS 1.3 首次连接 | 1 | 1 | **2 RTT** |
| TLS 1.3 0-RTT 重连 | 0（TFO） | 0 | **0 RTT** |

### 7.2 TLS 1.2 vs 1.3

| 指标 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| 握手 RTT | 2 | **1** |
| 密钥交换 | RSA / ECDHE | 仅 ECDHE（前向安全） |
| 0-RTT 恢复 | 不支持 | **支持（PSK）** |
| 总延迟（含 TCP） | 3 RTT | 2 RTT |

### 7.3 TCP Fast Open (TFO) + TLS 1.3

- TFO 允许在 SYN 中携带数据 → TCP 减少 1 RTT
- TLS 1.3 0-RTT 允许在 ClientHello 中携带 HTTP 数据
- 重连场景理论可达 **0 RTT**！

```bash
# 开启 TFO（客户端 + 服务端）
sysctl -w net.ipv4.tcp_fastopen=3
```

**注意事项**：
- 0-RTT 有重放攻击风险，仅适合幂等 GET 请求
- 服务端需开启 TLS Session Ticket / PSK 以支持 0-RTT
