# HTTPS / TLS 握手与 TCP 优化

## 1. TLS 握手与 TCP 的关系

HTTPS = HTTP over TLS over TCP。TLS 握手发生在 TCP 三次握手之后，因此：

- TCP 三次握手：1 RTT
- TLS 1.2 完整握手：2 RTT（Certificate + Key Exchange）
- 总计：**最少 3 RTT** 才能发送第一个 HTTP 请求

这是 HTTPS 延迟的主要来源，也是 TCP 优化参数与 TLS 优化紧密耦合的原因。

## 2. TLS 1.2 完整握手流程 (https.png)

```
Client                                    Server
  |                                         |
  |--- TCP SYN --------------------------->|  (0.5 RTT)
  |<-- TCP SYN+ACK ------------------------|
  |--- TCP ACK --------------------------->|  (1 RTT TCP 完成)
  |                                         |
  |--- ClientHello ----------------------->|  (RTT 1: TLS)
  |<-- ServerHello + Certificate           |
  |    + ServerKeyExchange                 |
  |    + ServerHelloDone ------------------|
  |                                         |
  |--- ClientKeyExchange                   |
  |    + ChangeCipherSpec                  |  (RTT 2: 密钥协商)
  |    + Finished ------------------------>|
  |<-- ChangeCipherSpec                    |
  |    + Finished -------------------------|
  |                                         |
  |--- HTTP Request (加密) --------------->|  (应用数据开始)
  |<-- HTTP Response (加密) --------------|
```

**总计**：3 RTT 后才发送首个 HTTP 请求。

## 3. TLS 1.3 优化 (https2.png)

TLS 1.3 大幅减少握手 RTT：

| 指标 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| 握手 RTT | 2 | **1** |
| 总 RTT (含 TCP) | 3 | **2** |
| 密钥交换 | RSA/ECDHE | 仅 ECDHE (前向安全) |
| 0-RTT 恢复 | 不支持 | **支持 (PSK)** |

**TLS 1.3 1-RTT 握手**：

```
Client                                    Server
  |--- ClientHello + KeyShare ------------>|  (生成的密钥)
  |<-- ServerHello + KeyShare              |
  |    + EncryptedExtensions               |
  |    + Certificate (加密)                |  (1 RTT)
  |    + Finished -------------------------|
  |--- Finished -------------------------->|
  |--- HTTP Request (加密) --------------->|
```

**TLS 1.3 0-RTT 恢复**（重连场景）：
- Client 发送 PSK + 早期应用数据（0-RTT data）
- 首个 HTTP 请求随 ClientHello 一起发出
- 总延迟 = 仅 TCP 1 RTT（2 RTT 含 TCP 握手从零开始）

## 4. 与 TCP 优化参数的关系

### TCP Fast Open (TFO) + TLS 1.3
- TFO 允许在 SYN 中携带数据 → TCP 减少 1 RTT
- TLS 1.3 0-RTT 允许在 ClientHello 中携带 HTTP 数据
- 两者结合：**重连场景可大幅缩短延迟**

### 参数配置要点
- `tcp_fastopen` = 3（客户端+服务端启用 TFO）
- 服务端 TLS Session Ticket / PSK 需开启以支持 0-RTT
- 注意 0-RTT 重放攻击风险：幂等请求可用 0-RTT，非幂等需禁用

### 延迟预算
| 场景 | TCP | TLS | 总 RTT |
|------|-----|-----|--------|
| 首次连接 (TLS 1.2) | 1 | 2 | **3** |
| 首次连接 (TLS 1.3) | 1 | 1 | **2** |
| 重连 (TFO + TLS 1.3 0-RTT) | 0 | 0 | **0** |
| 重连 (TFO + TLS 1.2 Session ID) | 0 | 1 | **1** |

## 5. 面试要点

1. **为什么 HTTPS 慢？** — TCP 三握 + TLS 握手，首次最少 2-3 RTT
2. **TLS 1.3 如何优化？** — 1-RTT 握手 + 0-RTT 恢复，取消静态 RSA 密钥交换
3. **TFO 如何与 TLS 配合？** — 重连场景可做到理论 0 RTT
4. **0-RTT 的风险？** — 重放攻击，仅适用于幂等 GET 请求
5. **Session ID vs Session Ticket vs PSK** — Session ID 需服务端缓存；Ticket 服务端无状态；PSK (TLS 1.3) 是首选

[src: raw/ingested/2技术/网络协议/tcp优化-https_tls握手优化.md]